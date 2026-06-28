# Lessons Learned — Ride Sharing

## Real Interview Stories

**Story 1: Uber Senior Engineer — Missed the Write Bottleneck**
Company: Uber | Level: Senior Engineer | Round: Technical Design

What happened: Candidate opened the capacity estimation with "Let's calculate trips per second: 26M trips/day ÷ 86,400 = ~300 trips/second. Easy." Built the entire design around optimizing the 300 TPS trip request path. Used PostgreSQL for everything. Never mentioned driver location updates.

Where it went wrong: Interviewer asked: "How many location updates are you handling per second?" Candidate: "Location updates? ... drivers update their location... maybe 125,000 per second?" Interviewer: "Right. How does that change your design?" Candidate's PostgreSQL-for-everything design couldn't handle 125K writes/second.

Consequence: Candidate spent the remaining 15 minutes retrofitting Redis into a design that was fundamentally wrong. No time for deep dive on matching algorithm.

What to do instead: During estimation, identify ALL write streams. For ride sharing: trips (300/sec), driver location updates (125K-500K/sec). The location update stream is the bottleneck, not the trip stream. Design for it from the start.

---

**Story 2: Lyft Staff Engineer — The Double-Booking Blind Spot**
Company: Lyft | Level: Staff Engineer | Round: System Design

What happened: Candidate designed a solid dispatch system with Redis GEOSET for driver locations, GEORADIUS for nearby driver lookup, and a scoring algorithm. Interviewer asked: "Your GEORADIUS query returns 50 nearby drivers. You select the top-3 and send a request to driver #1. Meanwhile, another rider in a different part of the city sends their request. Your dispatch system returns the same driver #1 as their top candidate. You send a request to driver #1 from both riders simultaneously. What happens?"

Where it went wrong: Candidate said "only one of them will get a response first." This is wrong — both requests arrive at the driver simultaneously (or within milliseconds). Without coordination, both trip records can be created.

Consequence: Interviewer escalated: "Now you have driver #1 committed to two riders. What is your system's behavior?" Candidate had no answer. Marked down severely on distributed systems reasoning.

What to do instead: Explicitly address the race condition. When Dispatch Service selects a driver, it acquires a distributed lock using Redis SETNX: "SET driver:{id}:lock NX EX 15". Only one instance wins the lock. The other immediately retries with driver #2. The lock is released when the driver accepts or rejects (or after 15 seconds).

---

**Story 3: DoorDash Senior Engineer — Geospatial Depth**
Company: DoorDash | Level: Senior SWE | Round: System Design (delivery driver matching — same core problem)

What happened: Candidate said "I'll use a geospatial database like PostGIS to find nearby drivers with a spatial query." Interviewer asked to go deeper: "Walk me through what PostGIS does with that query. How does it find drivers within 3km?"

Where it went wrong: Candidate couldn't explain the underlying spatial index. They knew PostGIS existed but not that it uses an R-tree index, how R-trees work spatially, or why Redis GEOSET (using geohash-encoded sorted sets) achieves O(log N + M) rather than O(N).

Consequence: Interviewer concluded the candidate was using tools they didn't understand. Red flag at senior level.

What to do instead: Know your spatial index. Redis GEOSET encodes (lat, lon) as a 52-bit geohash stored as the score in a sorted set. GEORADIUS finds the geohash ranges that cover the search circle and does a range scan on the sorted set — O(log N + M). Geohash cells adjacent in hash space are adjacent geographically (with some edge cases). This is the key insight.

## General Pattern Mistakes

**Mistake 1: Ignoring Driver Location Update Volume**
What happens: Candidate designs for trip request throughput (300/sec) and uses RDBMS for everything.
Why wrong: Driver location updates run at 125K-500K writes/second — 400× the trip rate. PostgreSQL cannot handle this write volume without significant sharding. Redis GEOADD handles 100K+ writes/second per instance.
Fix: Identify location updates as the primary write bottleneck in estimation. Route to Redis GEOSET, not the trip database.

**Mistake 2: No Race Condition Protection on Driver Assignment**
What happens: Candidate selects a driver and assigns the trip without distributed coordination.
Why wrong: Multiple Dispatch Service instances can select the same driver simultaneously. Both create trip records. Driver gets two trip requests. Double-booking is catastrophic for trust.
Fix: Redis SETNX distributed lock on the driver before assignment. Only one instance wins. Others fall through to next candidate.

**Mistake 3: Omitting the Trip State Machine**
What happens: Candidate jumps to components without defining states and transitions.
Why wrong: Without a state machine, there's no answer for: "driver times out on accepting," "rider cancels mid-trip," "driver marks arrival but rider doesn't board." The state machine is the design spec for the Trip Service.
Fix: Always draw the state machine first (REQUESTED → DRIVER_ASSIGNED → IN_PROGRESS → COMPLETED). Name all failure transitions (FAILED, CANCELLED, DISPUTED). Then build components to implement these transitions.

**Mistake 4: Using the Same DB for Location and Trip State**
What happens: Candidate uses one database (usually PostgreSQL) for both driver locations and trip records.
Why wrong: These have opposite consistency requirements. Trip state needs strong consistency (ACID transactions, no double-booking). Location data needs high write throughput with eventual consistency (stale by a few seconds is fine). Using PostgreSQL for location writes at 125K/sec creates an unnecessary performance bottleneck.
Fix: Redis GEOSET for location (fast writes, expiry-based staleness detection). PostgreSQL for trip state (ACID transactions, row locking).

**Mistake 5: Euclidean Distance as ETA**
What happens: Candidate computes driver-to-rider distance as Euclidean (straight line), uses it as ETA proxy.
Why wrong: Urban driving routes are rarely straight-line. A 500m straight-line distance can be a 5-minute drive. Sending the "nearest" driver by Euclidean distance may result in longer actual wait than a driver 700m away in better road alignment.
Fix: Distinguish between distance and ETA explicitly. Mention that ETA requires road network graph traversal (Dijkstra/A* on OpenStreetMap or Google Maps API). For the interview, propose a simplification: use historical average speed in the city × straight-line distance × road_factor (e.g., 1.4 for urban grids). Full ETA is a separate microservice.

## Self-Assessment Checklist
- [ ] I identified driver location updates (not trip requests) as the primary write bottleneck
- [ ] I proposed Redis GEOSET (or equivalent geospatial index) for driver location, not PostgreSQL
- [ ] I drew the trip state machine with all states and failure transitions before building components
- [ ] I addressed double-booking prevention with distributed locking
- [ ] I separated location storage (Redis, eventual) from trip state (PostgreSQL, strong)
- [ ] I described the offline/push path for driver notification
- [ ] I described the real-time location sharing during trip (WebSocket)
- [ ] I addressed at least two failure scenarios without prompting

## Remediation Targets
- If you missed location volume: Practice "estimate first, then pick the right storage" as a habit. Always enumerate all write streams before designing storage.
- If you missed double-booking: Study Redis SETNX and distributed locking patterns. This is a core distributed systems skill.
- If you missed the state machine: Before any system with multi-step workflows, spend 3 minutes drawing the state machine. It becomes the design spec.
- If you struggled with geospatial indexing: Study Redis GEOSET commands. Understand that GEORADIUS is O(log N + M), not O(N).
