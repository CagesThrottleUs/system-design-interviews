# Ride Sharing (Uber / Lyft)
**Difficulty:** Advanced
**Category:** Geospatial / Real-Time / State Machine
**Time Box:** 45 min
**Key Concepts:** Geohash, Driver Location Updates, Trip State Machine, Matching Algorithm, Surge Pricing
**Asked at:** Uber, Lyft, DoorDash, Grab, Didi, Google, Meta

## Problem Statement

You are designing the core backend for a ride-sharing application. A rider opens the app, requests a ride from their location, and within seconds sees a list of nearby drivers with estimated arrival times. A driver accepts the request, drives to the rider, completes the trip, and gets paid. The system must handle millions of concurrent riders and drivers across hundreds of cities simultaneously.

The hardest parts of this problem are not the obvious ones. The real challenges are: tracking thousands of driver locations updating every few seconds (a write-heavy, high-frequency geospatial stream), matching riders to the optimal nearby driver within seconds (a real-time spatial query under load), and maintaining a consistent trip state machine that both rider and driver see the same version of (a distributed state coordination problem).

## Actors
| Actor | Description |
|-------|-------------|
| Rider | Opens the app, requests a ride, tracks driver location in real-time |
| Driver | Broadcasts location every 4 seconds, accepts/rejects trip requests |
| Dispatcher | Internal service that matches riders to available drivers |
| Trip Manager | Maintains authoritative trip state (REQUESTED → IN_PROGRESS → COMPLETED) |

## Functional Requirements
**Core (MVP):**
- FR1: Rider requests a ride from current location to destination
- FR2: System shows nearby available drivers with ETA
- FR3: System matches rider to optimal nearby driver
- FR4: Driver can accept or reject the match request
- FR5: Rider and driver see each other's real-time location during the trip
- FR6: Trip completes, fare is calculated, both parties are rated

**Secondary:**
- FR7: Surge pricing — fare multiplier when demand exceeds supply in an area
- FR8: ETA calculation using real road network (not straight-line distance)
- FR9: Driver earnings and trip history

## Non-Functional Requirements
| Requirement | Target | Rationale |
|-------------|--------|-----------|
| Driver location update latency | < 5 seconds visible to rider | Rider needs to see driver moving in real time |
| Match latency | < 3 seconds from request to driver notification | Beyond this, conversion rate drops sharply |
| Availability | 99.99% | Revenue-critical; downtime = lost rides |
| Driver location write throughput | 500K updates/sec (peak) | 500K active drivers × 1 update/4 sec × 4 cities peak |
| Trip state consistency | Strong (linearizable) | Rider and driver must see same trip state; double-booking is catastrophic |
| Location data freshness | < 4 seconds stale | Driver sends update every 4 seconds |

## Capacity Estimation Hints
- Active drivers at peak: 500,000 worldwide
- Active riders looking for rides: 100,000 at any moment
- Driver location update frequency: every 4 seconds
- Average trip duration: 15 minutes
- Trips per day: 25 million (Uber's 2023 figure: ~26M trips/day)
- Location payload size: 50 bytes (driver_id, lat, lon, heading, timestamp)
- Search radius for matching: 3 km (urban), 10 km (suburban)

## Good Clarifying Questions to Ask

**Scale:**
- How many concurrent active drivers? How many riders requesting rides simultaneously?
- How many cities? Does each city have an independent deployment?
- What's the expected peak QPS for driver location updates?

**Features:**
- Do we need real-time location sharing between rider and driver during the trip?
- Is ETA based on straight-line distance or real road network?
- Is surge pricing in scope?
- Do we need to handle driver rejection and re-matching?

**Consistency:**
- What happens if two riders are matched to the same driver simultaneously?
- Can a driver be double-booked?

**Constraints:**
- What is the matching radius — how far will the system search for drivers?
- Do we need to optimize for closest driver, or factor in heading and traffic?

## Key Components
<details><summary>Reveal components</summary>

- **Location Service**: Receives driver location updates (WebSocket/HTTP); writes to Redis geospatial index
- **Redis GEOSET**: Geospatial index per city; GEORADIUS queries for nearby drivers in O(N+log M) time
- **Dispatch Service**: Runs matching algorithm; sends trip request to selected driver; handles accept/reject
- **Trip Service**: Owns the authoritative trip state machine; persists to PostgreSQL with row-level locking
- **ETA Service**: Computes estimated arrival time using road network graph (pre-computed or third-party)
- **Surge Engine**: Computes demand/supply ratio per geohash cell; publishes multipliers
- **WebSocket Gateway**: Maintains long-lived connections for real-time rider/driver location sharing during trip
- **Notification Service**: Pushes trip requests to driver app via WebSocket or APNs/FCM
</details>
