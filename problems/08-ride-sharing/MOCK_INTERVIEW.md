# Mock Interview Script — Ride Sharing
*For the interviewer. Do not share with candidate.*

## Pre-Session Setup
- Problem: Design a ride-sharing backend (Uber/Lyft scale)
- Target level: Apple ICT3 / Uber Senior / Lyft Senior
- Time: 45 minutes
- Key traps to watch: location update volume, double-booking, state machine, geospatial choice

## Opening (read aloud, verbatim)
"Design the backend for a ride-sharing application like Uber or Lyft. A rider requests a ride, gets matched with a nearby driver, and the driver completes the trip. You have 45 minutes. Start with any clarifying questions."

*Start timer.*

## Phase 1: Requirements (0–8 min)

**Answers if asked:**
| Question | Answer |
|----------|--------|
| Scale? | 500K active drivers at peak, 26M trips/day |
| Location update frequency? | Drivers update every 4 seconds |
| Real-time location sharing? | Yes — rider sees driver moving during trip |
| Surge pricing? | Mention as important; no need to deep dive |
| ETA: straight-line or road network? | Road network ideal, but simplification acceptable |
| Group rides? | Out of scope |
| Multiple cities? | Yes — treat each city as an independent geo context |
| Driver rejection / re-matching? | Yes — driver can reject; system matches to next candidate |

**Red flag:** Candidate starts designing before asking about location update frequency.

## Phase 2: Estimation (8–13 min)

**Critical trap question:** "How many writes per second does your system handle?"

**Expected:**
- Trip requests: 26M / 86,400 ≈ 300/sec (easy)
- Driver location updates: 500K drivers / 4 sec = 125K/sec (the actual bottleneck)

**If they miss location updates:** "What about driver location updates? How frequently do those happen? How many drivers do we have active at peak?"

**If they get both:** "Good. Which one drives your storage and write throughput design?"

## Phase 3: High-Level Design (13–28 min)

**Expected components:**
- Location Service + Redis GEOSET (or similar geospatial index)
- Dispatch Service with matching algorithm
- Trip Service with state machine (PostgreSQL, strong consistency)
- WebSocket for real-time location during trip
- Driver notification (WebSocket + APNs/FCM fallback)

**Critical probe at 18 min:** "Walk me through the entire flow from the moment a rider taps 'Request Ride' to the moment a driver receives the trip request on their app. Be specific — name every component and every data store."

**Critical probe at 23 min:** "Two riders 200m apart both request a ride simultaneously. Your GEORADIUS query returns the same driver as the best match for both. What happens?"

*If candidate doesn't address double-booking → mark down. Ask: "What prevents that driver from being committed to two trips?"*

**If they use PostgreSQL for location:** "You have 125,000 location updates per second going to PostgreSQL. How does that perform?"

## Phase 4: Deep Dive (28–40 min)

**Option A — Geospatial Index:**
"Go deeper on your location index. If I use Redis GEOSET, walk me through how GEORADIUS works internally. What data structure does Redis use? What's the time complexity?"

**Option B — Matching Algorithm:**
"Your matching algorithm returns the closest driver. I challenge you: what's wrong with always picking the closest driver? Walk me through three scenarios where closest-by-distance is not the best choice."

**Option C — Trip State Machine:**
"Walk me through your full trip state machine — every state and every transition, including failure paths. What happens if the driver accepts but never arrives? What happens if the rider cancels 5 minutes into the trip?"

**Option D — Real-Time Location During Trip:**
"During the trip, the rider's app shows the driver's position updating in real time. How does that work? How many concurrent connections? What's the write fanout per location update?"

## Phase 5: Failure & Scaling Probe (40–44 min)

Pick TWO:
1. "Your Redis location cluster goes down during evening rush. What is the blast radius? Can riders still request rides? What degrades?"
2. "PostgreSQL trip table is getting slow — 100ms writes instead of 2ms. The 3-second match latency SLA is at risk. What do you do?"
3. "A city has a major event (Super Bowl). Driver supply is constant but rider demand spikes 10×. Walk me through how surge pricing should work. What does your Surge Engine do?"
4. "A driver's app crashes mid-trip. They lose their WebSocket connection. How does the trip proceed? How does the rider's app handle this?"

## Closing (44–45 min)
"Final question: what is the weakest part of your design, and what would you tackle first with another hour?"

## Scoring Checklist

| Dimension | Evidence | Score (1-5) |
|-----------|----------|-------------|
| Requirements | Asked about location frequency, driver count, real-time sharing | |
| Estimation | Correctly identified 125K writes/sec from location updates | |
| Geospatial design | Redis GEOSET or equivalent; explained WHY not a relational DB | |
| Double-booking prevention | Distributed lock or DB-level transaction; specific mechanism | |
| Trip state machine | Named all states + failure transitions before component design | |
| Consistency separation | Different DBs for location (eventual) vs trip state (strong) | |
| Failure reasoning | Two or more failure scenarios with specific mitigations | |
