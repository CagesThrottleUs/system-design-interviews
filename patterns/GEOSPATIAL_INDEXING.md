# Geospatial Indexing

## Intent

Efficiently answer proximity queries ("find all X within Y km of point P") over millions of live, frequently-updated coordinate records without scanning every row.

---

## The Problem

"Find all drivers within 2km of a passenger." The naive solution: fetch all active driver locations from the database and compute the Haversine distance for each driver in application code. At 10 million active drivers globally, this is 10 million floating-point distance computations per passenger request. Uber processes tens of thousands of matching requests per second. 10M computations × 10K req/sec = 100 billion distance calculations per second. No single machine does that, and adding machines linearly to do distance math is not a data architecture — it's a budget problem.

The less naive solution: put latitude and longitude in a relational database and add a B-tree index on each column. Then issue a range query: `WHERE lat BETWEEN 12.0 AND 12.02 AND lon BETWEEN 77.5 AND 77.52`. This works, and it's better than full table scans, but it still has a fundamental flaw. A B-tree index sorts data along one dimension. An index on latitude puts all rows with lat=12.01 near each other on disk, regardless of their longitude. The query optimizer can use the latitude index to narrow the result set, but it cannot use both the latitude index and the longitude index simultaneously in a way that narrows both dimensions at once. You get a O(rows matching latitude range) scan of the latitude-filtered results to then apply the longitude filter. In a dense city like Mumbai, thousands of rows might match the latitude range for any 2km band — and you still have to read all of them to apply the longitude filter.

The real fix is to reduce a 2D problem to a 1D problem. If you can convert (lat, lon) pairs to a single value that preserves proximity — nearby points get nearby values — then a standard B-tree index on that single value gives you an efficient range query. This is the insight behind every geospatial indexing algorithm.

---

## Geohash

Geohash works by interleaving the bits of latitude and longitude to produce a single string that encodes a rectangular cell. The encoding works in alternating bisections: the first bit indicates which half of the longitude range the point falls in, the second bit indicates which half of the latitude range, the third bit subdivides longitude again, and so on. After encoding as bits, the result is converted to a base-32 string using a custom alphabet.

The resulting string has a crucial property: the longer the shared prefix between two geohash strings, the closer together the corresponding points are. A geohash prefix search in a database — `WHERE geohash LIKE 'tdr1y%'` — retrieves all points within the cell encoded by that prefix. A standard B-tree index on the geohash string makes this a fast index range scan rather than a table scan.

Real cell sizes at each precision level (memorize these for interviews):
- Length 1: ~5,000 km × 5,000 km (roughly a continent)
- Length 4: ~40 km × 20 km (roughly a large city)
- Length 5: ~5 km × 2.5 km (a few neighborhoods)
- Length 6: ~1.2 km × 0.6 km (several city blocks)
- Length 7: ~150 m × 150 m (a city block)
- Length 8: ~40 m × 20 m (a building footprint)

For a "find drivers within 2km" query, length 5 gives cells of roughly the right size. You compute the passenger's geohash at length 5, and search for drivers with matching prefixes.

The prefix search has an important limitation that every candidate gets wrong in interviews. Two points that are geographically adjacent but fall on opposite sides of a geohash cell boundary will have different prefixes, even if they are only a few meters apart. Consider two drivers at the northeastern corner of cell `tdr1x` and the northwestern corner of cell `tdr1y` — they might be 50 meters apart but their geohash strings share only the first 4 characters, not 5. If you search only within the passenger's cell, you miss drivers in adjacent cells who are actually closer than some drivers within the passenger's cell.

The fix is always to search the target cell plus all 8 neighbors. Given a geohash at length 5, you compute the 8 surrounding cells (north, northeast, east, southeast, south, southwest, west, northwest) and query for drivers in any of the 9 cells. Most geohash libraries provide a `neighbors()` function. This guarantees no nearby driver is missed at the cost of at most 9× the search space — and in practice, most cells are sparse, so the extra cells add few false positives.

In a Redis implementation, drivers publish their location by calling `GEOADD drivers {lon} {lat} {driver_id}`, which stores points in a sorted set using geohash encoding internally. A proximity query uses `GEORADIUS drivers {lon} {lat} 2 km ASC COUNT 20`, which performs the neighbor-cell search internally and returns the closest N drivers. Redis's GEORADIUS command was built precisely for this use case and is used in production by services like Grab (Southeast Asian ride-sharing, serving 187 million users as of 2023 per Grab's annual report).

---

## Quadtree

A quadtree solves the same proximity problem with a different data structure. Start with a bounding rectangle covering the entire map. If that rectangle contains more than K points (a configurable threshold, typically 50–200), divide it into exactly four equal quadrants and recursively apply the same rule to each quadrant. Continue subdividing until every leaf node contains at most K points. A search for drivers within radius R of a point P traverses the tree from the root, descending only into quadrants that intersect the search circle, and collects all points from qualifying leaf nodes.

The quadtree's critical advantage over geohash is adaptive density handling. In Manhattan (10,000 drivers per square mile), the quadtree creates very fine cells to keep each leaf under the threshold. In rural Montana (0.1 drivers per thousand square miles), the quadtree creates large cells. The cell resolution automatically matches point density. A geohash at a fixed precision level creates the same cell everywhere — a length-7 geohash creates 150m × 150m cells in both Manhattan and rural Montana, which means the Manhattan cells might contain hundreds of drivers while the Montana cells contain zero. The quadtree eliminates this imbalance by construction.

The quadtree's disadvantage is complexity of implementation and the difficulty of distributing it across machines. A geohash is a string — you can shard a database by geohash prefix, store geohash strings in any key-value store, and add a B-tree index with zero framework support. A quadtree is a tree data structure that requires either in-memory storage (fast but requires all data to fit on one machine) or a custom distributed implementation (complex). For this reason, quadtrees are common in game engines and mapping applications where the full dataset fits in memory, while geohashes dominate distributed database systems.

The update story also differs. Updating a driver's location in a geohash-indexed system is a single key-value update: compute the new geohash and overwrite the old value. Updating a location in a quadtree might require rebalancing — if the driver crossed a quadrant boundary, you need to remove the point from one leaf node and add it to another, and if that node was at the threshold, you might trigger a merge of empty sibling nodes. In practice, most quadtree implementations avoid dynamic rebalancing by rebuilding the tree periodically or by using a loose threshold that tolerates temporary overflows.

---

## S2 Geometry and H3

Geohash and quadtrees both work on a flat projection of the Earth. For global queries, this introduces distortion: near the poles, latitude-longitude grid cells cover much smaller actual areas than near the equator, and distance computations require the Haversine formula to correct for spherical geometry. At a system design interview level, these distortions rarely matter. But two production systems were built specifically to address them.

**S2 Geometry** was developed by Google and is what powers Google Maps, Foursquare, and several other mapping services. S2's key insight is to map the Earth's sphere onto the six faces of a cube (using a projection that minimizes area distortion), then to index each face using a Hilbert curve. A Hilbert curve is a space-filling curve — it visits every point in a 2D grid exactly once, and crucially, it has strong locality preservation: points that are nearby on the curve are nearby in 2D space. S2 cells at the same level cover approximately equal areas on the sphere, regardless of latitude. A level-12 S2 cell covers about 3.3 km² everywhere on Earth. This uniformity makes capacity planning and sharding straightforward in a way that latitude-longitude-based systems are not.

**H3** is Uber's hexagonal hierarchical geospatial indexing system, open-sourced by Uber in 2018. Before H3, Uber used geohash for driver-passenger matching. They switched because hexagons have a geometric property that squares lack: in a hexagonal grid, all six neighbors of any cell are the same distance from the cell's center. In a square grid, the four edge neighbors are at distance 1 and the four corner neighbors are at distance √2 ≈ 1.41. This asymmetry means that when Uber's routing algorithms compute supply/demand heatmaps and diffuse demand across neighboring cells, square cells create directional artifacts (demand spreads faster diagonally than cardinally). Hexagonal cells give isotropic diffusion, which produces more accurate surge pricing models and smoother ETA estimates.

Uber's 2018 engineering blog post on H3 describes how they use it not just for driver-passenger matching but for building surge pricing heatmaps, analyzing market health by region, and planning driver incentive zones. H3 was designed to be hierarchical — a level-8 H3 cell can be subdivided into exactly 7 level-9 cells — which allows Uber to aggregate statistics at coarse granularity (city level) and then drill down to fine granularity (neighborhood level) using the same index. As of 2024, H3 has been adopted by Foursquare, Strava, and several urban planning tools outside of Uber.

---

## Trigger Signals

Mention geospatial indexing when the problem involves:
- Any location-aware matching (ride-sharing, food delivery, dating apps with proximity)
- "Find nearby X" queries over large datasets (nearby restaurants, nearby users, nearby stores)
- A design where the database stores latitude/longitude coordinates and the interviewer asks about query performance
- Delivery zone assignment or geofencing (is this delivery address inside our service area?)
- Heatmap or density computation over geographic regions
- Any logistics problem involving routing or coverage analysis

---

## Problems Where It Applies

**Design Uber/Lyft** — The driver-passenger matching problem is the canonical geospatial indexing question. At 1M active drivers, you need to answer "find 5 closest drivers to this passenger" in under 100ms. The standard answer is geohash (or H3, if you want to signal awareness of Uber's actual architecture) stored in Redis, with a GEORADIUS query on the driver's current cell and its 8 neighbors.

**Design Google Maps / Nearby Search** — The "find restaurants near me" feature queries a geospatial index over tens of millions of business locations. The read-heavy, rarely-updated nature of POI (point of interest) data makes this a good candidate for geohash with a PostGIS-backed database or a pre-computed S2 cell index, rather than a Redis in-memory approach.

**Design Yelp** — Same as nearby search. Yelp's engineering team has published about using PostGIS (PostgreSQL's geospatial extension) for proximity queries, with geohash-based partitioning for sharding their restaurant database across multiple PostgreSQL instances.

**Design a Delivery System** — Geofencing (is this order address within our delivery zone?) is a geospatial containment query rather than a proximity query, but the same indexing structures apply. A delivery zone can be represented as a polygon of geohash cells, and checking whether a delivery address falls within the zone becomes a lookup in a set of geohash prefixes.

---

## Trade-offs Table

| Property | Geohash | Quadtree | S2 | H3 |
|---|---|---|---|---|
| **Representation** | Base-32 string | Tree of bounding boxes | 64-bit integer cell ID | 64-bit integer cell ID |
| **Storage** | Simple string, any KV store | In-memory tree or custom DB | Integer column, B-tree index | Integer column, B-tree index |
| **Update cost** | O(1) — recompute string, update single key | O(log n) — may require node split/merge | O(1) — recompute cell ID | O(1) — recompute cell ID |
| **Handles density variation?** | No — fixed cell size at each precision | Yes — adaptive subdivision | Approximately — cells nearly equal area | No — fixed cell size at each resolution |
| **Edge/boundary problem** | Yes — straddle-boundary miss; fix: search 9 cells | No — tree traversal handles boundaries natively | Minimal — Hilbert curve minimizes boundary effects | Yes — search 7 neighbors (center + 6) |
| **Neighbor shape** | Square | Rectangle (variable size) | Approximately square on sphere | Hexagon (isotropic) |
| **Best use case** | Distributed systems, simple sharding, Redis | Dense in-memory datasets, game engines, variable density | Global-scale mapping, equal-area analysis | Routing, heatmaps, demand diffusion |

---

## Real-World Usage

**Uber (H3, 2018):** Uber published a detailed engineering blog post in 2018 explaining why they migrated from geohash to H3 for their global marketplace. The core reason was isotropic neighbor distances for their surge pricing diffusion algorithm. As of 2024, Uber uses H3 at resolution levels 8–10 for demand forecasting (level 8 cells are ~0.74 km², level 10 cells are ~0.015 km²). H3 is open-source at github.com/uber/h3 and has over 4,000 stars and production adoption outside Uber.

**Google Maps (S2, ~2017):** Google Maps uses S2 geometry internally for proximity search, region indexing, and coverage analysis. Foursquare published a blog post in 2016 describing their migration to S2 for venue proximity queries, citing the equal-area cell property as critical for their global venue database of over 100 million places. The S2 library is open-source at github.com/golang/geo and has been ported to multiple languages.

**Yelp (PostGIS, ~2015–2024):** Yelp's engineering blog has documented their use of PostgreSQL with the PostGIS extension for geospatial queries. PostGIS adds a `GEOGRAPHY` type that stores latitude/longitude and supports spatial indexes (R-trees under the hood, not geohash), with functions like `ST_DWithin(location, target_location, radius_meters)` that query using the spatial index efficiently. Yelp's search pipeline uses Elasticsearch's geo_distance query for the full-text + proximity combined search case, with geohash-based shard routing.

---

## Common Mistakes

**1. Forgetting the neighbor cell search.** The single most common geohash mistake in interviews. If you search only the cell containing the query point, you will miss nearby points that fall in adjacent cells just over the boundary. The correction — search the target cell and all 8 surrounding cells — is a one-liner in code and a mandatory detail in any interview answer involving geohash. A candidate who proposes geohash without mentioning neighbor search is proposing a broken system.

**2. Using Euclidean distance instead of Haversine for distance filtering.** After retrieving candidate drivers from the geohash cell search, you still need to compute the actual distance and filter to those within the radius. The Euclidean formula `sqrt((Δlat)² + (Δlon)²)` is wrong for geographic coordinates because one degree of longitude at 45° latitude corresponds to ~79km, while one degree of latitude is always ~111km. The Haversine formula accounts for the Earth's spherical geometry. At small scales (a few km in a mid-latitude city), the Euclidean error is small (~0.5%), but for a geographically-aware system design, using Haversine signals the right level of rigor.

**3. Treating geospatial indexing as the only problem when designing Uber.** Driver-passenger matching is not just a proximity query. At Uber's scale, the system also has to handle driver availability state, passenger request deduplication, optimistic locking when multiple passengers try to claim the same driver, and cancellation race conditions. Geospatial indexing answers "which drivers are nearby" — it does not answer "which of those nearby drivers can I safely lock for this passenger." Candidates who spend the entire HLD on geohash indexing without addressing the matching transaction and driver state machine are solving only half the problem.

**4. Proposing a relational database with two separate B-tree indexes on lat and lon columns as the solution.** As described in the problem section, a B-tree index on latitude and a separate B-tree index on longitude cannot be used simultaneously in a way that filters both dimensions efficiently. The optimizer will use one index to narrow the result set and then apply the other as a filter over the narrowed results — not as a combined spatial index. The correct solutions are: PostGIS R-tree spatial index (for relational databases), geohash strings with a B-tree index, Redis GEORADIUS, or a dedicated geospatial search engine like Elasticsearch geo_distance. Proposing naive dual B-tree indexing in a system design interview signals unfamiliarity with how database indexes work.
