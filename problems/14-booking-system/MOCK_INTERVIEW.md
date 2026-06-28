# Mock Interview Script — Hotel / Flight Booking System

**For the interviewer only. Do not share with the candidate.**

---

## Pre-Session Setup

**Target level:** Senior engineer (L5/ICT3 equivalent)
**Expected duration:** 45 minutes
**This is an Advanced problem.** Correctness and failure scenarios matter more than scale.
**Red flags:** No hold mechanism, 2PC for charge+inventory, no hold expiry worker, claims only airlines overbook
**Green flags:** Separated search/booking paths, Redis SETNX hold with TTL, DB-level CHECK constraint, SAGA with named compensating transactions, hold expiry race addressed

---

## Opening

"I'd like you to design a hotel and flight booking system — think Booking.com or Expedia. Users search for rooms or flights, select a specific room or seat, enter payment details, and confirm the booking. Design the system with correctness as the primary requirement — I don't want any double-bookings."

*Let the candidate clarify requirements. Don't volunteer that it's hotels AND flights until asked — see if they ask.*

---

## Phase 1: Requirements (0–8 min)

**Expected clarifying questions:**
- Hotels, flights, or both? (Important — flights require per-seat locking, hotels are per-room-type)
- Overbooking policy? (Probe their knowledge — both hotels and airlines overbook)
- Hold during checkout? When does a hold get created? How long does it last?
- Consistency model? (Should say CP for booking)

**Probe if overbooking not asked:** "Airlines sell more seats than the plane has. How do you want to handle overbooking policy in your design?"

**Probe if hold not mentioned:** "The user selects a room at 2:00 PM and takes 4 minutes to enter their payment details. Is the room visible as available to other searchers during those 4 minutes? What if they abandon checkout?"

---

## Phase 2: Estimation (8–13 min)

**Guide candidate to key numbers:**
- 500K bookings/day ÷ 86,400 = **5.8 writes/sec** (booking is NOT a throughput problem — it's a correctness problem)
- 10M searches/day ÷ 86,400 = **116 searches/sec** (100× more reads than writes)
- Peak concurrent holds: **~10,000** (trivial for Redis)
- Hotel inventory table: **73 million rows** (5,000 hotels × 20 types × 730 days)

**Key probe:** "Given that bookings are only 5.8 writes/sec, what's the hardest engineering challenge in this problem?" 
Expected: concurrency on the SAME inventory item, not throughput. Many users competing for the last room simultaneously.

---

## Phase 3: HLD (13–28 min)

**Expected path:**
1. Search Service → Redis/Elasticsearch cache (eventual consistency OK)
2. Hold Service → Redis SETNX with TTL
3. Payment Service → PSP adapter with idempotency key
4. Booking Service / Inventory commit → DB atomic UPDATE + CHECK constraint
5. Hold Expiry Worker → background daemon

**Critical probe — no hold mentioned:** "User selects room, spends 5 minutes filling out payment form. Another user selects same room, also spends 5 minutes filling out payment form. Both submit at the same time. What happens?"
Expected: without a hold, both reach payment simultaneously → one succeeds, one fails → need the hold to prevent this UX failure

**Critical probe — hold design:** "Walk me through your hold implementation in Redis. What's the key? What's the value? What happens if two users try to hold the same room simultaneously?"
Expected: SETNX with key=`hold:{hotel}:{room_type}:{date}`, value=user_id, TTL=300s. SETNX is atomic — only one writer wins. Second writer gets 0 and must return INVENTORY_UNAVAILABLE.

**Critical probe — DB invariant:** "Your Redis hold expires after 5 minutes. What prevents two users from successfully committing bookings to the same room in the database?"
Expected: DB-level `UPDATE room_type_inventory SET total_reserved = total_reserved + 1 WHERE (total_inventory - total_reserved) >= 1` — atomic, and DB CHECK constraint is last resort

---

## Phase 4: Deep Dive (28–40 min)

**Option A — Hold Expiry Race:**
"User holds seat 14A. TTL is 300 seconds. User's 3D Secure authentication takes 280 seconds. The user's payment succeeds at second 305 — 5 seconds after the hold expired. Another user grabbed the hold at second 301. Walk me through exactly what happens at each step."
Expected: hold expiry at 300s; second user holds at 301s; first user's payment succeeds at 305s; booking service checks hold ownership before committing — hold belongs to second user, not first → issue refund to first user, return HOLD_EXPIRED

**Option B — Hold Expiry Worker:**
"Redis TTL fires. The hold key is deleted. What needs to happen in your inventory database? Who does it, and how do they know which inventory row to update?"
Expected: hold value contained hotel_id, room_type_id, date; keyspace notification triggers expiry worker; worker decrements total_reserved; booking service must not have committed inventory (if it did, the booking committed first and the hold delete was correct)

**Option C — Flight Seat Map:**
"You've added flight booking. I click on flight UA456 and need to see a seat map showing which seats are available, held, or booked in real-time. 1,000 users are viewing the same seat map simultaneously. Walk me through the architecture."
Expected: seat state in Redis hash (flight_id → {seat_id → AVAILABLE/HELD/BOOKED}); updated on hold create/release/booking commit; read QPS = 1,000 × (however many map refreshes/sec) served from Redis; durability in PostgreSQL

---

## Phase 5: Failure & Scaling (40–44 min)

**Probe:** "Your Redis cluster goes down. What happens to the booking flow?"
Expected: new holds cannot be created (booking path unavailable); in-progress holds with payment submitted can still commit to DB if booking service has hold_id cached; existing bookings unaffected; Redis recovery restores hold state from DB (or accept brief period of no-holds = no new checkouts)

**Probe:** "It's New Year's Eve booking surge — 10× normal volume. Which component fails first?"
Expected: search path serves from cache — no issue; booking path is 58 writes/sec — still no issue; the bottleneck is Redis for holds (10,000 concurrent holds × 10× = 100,000 hold keys — still trivial). The real bottleneck would be PSP (Stripe's rate limits) not your own infrastructure.

---

## Closing

"Two questions:
1. What's the one scenario where a user gets charged but ends up with no booking?
2. How would you detect and automatically recover from that?"

Strong answer: Hold expires during payment, second user holds, first payment succeeds, booking service rejects (HOLD_EXPIRED), first user gets charged and immediately refunded. Detection: payment succeeded + HOLD_EXPIRED error = automatic refund trigger, logged as "orphaned charge" event for ops review. Recovery: reconciliation job compares PSP charges against bookings table nightly.

---

## Scoring Checklist

| Area | Weight | 1 (Poor) | 3 (OK) | 5 (Strong) |
|------|--------|----------|--------|------------|
| Requirements clarification | 10% | Didn't ask about hold or overbooking | Asked about hold; missed overbooking nuance | Asked about hold TTL, overbooking policy (both hotel+airline), locking preference |
| HLD completeness | 30% | No hold mechanism; search + booking same path | Hold present; search cache mentioned | Two-tier (search=cache, booking=DB), SETNX hold with TTL, expiry worker, DB constraint |
| LLD depth | 30% | No concurrency mechanism at DB level | DB UPDATE for inventory decrement | Atomic UPDATE with CHECK constraint, SETNX Lua script, hold expiry race addressed |
| Tradeoff reasoning | 20% | No justification for locking choices | Optimistic vs pessimistic mentioned | SAGA vs 2PC justified with PSP argument, locking choice driven by QPS, hold TTL trade-off |
| Communication clarity | 10% | Disorganized | Logical flow | Clear separation of search/hold/booking paths, explicit failure narration |
