# Lessons Learned — Hotel / Flight Booking System

---

## Real Interview Stories

### Story 1 — Airbnb, E5 (2024)

The candidate proposed a design where search queries hit the live PostgreSQL inventory database directly. The interviewer asked: "You have 1,000 searches/sec at peak. Walk me through the database query for each search." The candidate described: join hotels table + room_type_inventory table + filter by date range + group by room type.

The interviewer asked how many rows that touches. The candidate calculated: 5,000 hotels × 20 room types × N dates in range ≈ millions of rows per query. At 1,000 searches/sec, that's billions of row-reads per second on a database that also handles writes. The interviewer said: "Airbnb's actual search doesn't query the inventory DB directly. How would you fix this?"

The candidate didn't know the answer. The correct answer is a read cache layer (Redis or Elasticsearch) that holds pre-computed availability snapshots, refreshed on inventory changes. Search is eventually consistent — a room shown as available may be booked by the time the user clicks it. This is acceptable because the booking step has strong consistency.

**The lesson: search and booking are fundamentally different consistency requirements. Read the rule: search serves approximate data at low latency; booking enforces exact correctness. Separate the paths.**

---

### Story 2 — Booking.com, Senior Engineer (2023)

The interviewer asked: "Two users click 'Book Now' on the last available room at the same millisecond. Walk me through exactly what happens in your system." The candidate said: "The first one to complete payment gets the room."

The interviewer probed: "They both start payment at the same time. They both complete payment at approximately the same time. How does your system ensure exactly one booking succeeds?"

The candidate said: "We check availability before charging." The interviewer: "You check, it says available. You both charge. You both update the database. Who wins?" The candidate realized there was a race condition but couldn't explain how to fix it.

The correct answer: the DB-level transaction with `UPDATE room_type_inventory SET total_reserved = total_reserved + 1 WHERE (total_inventory - total_reserved) >= 1` is atomic. The CHECK constraint `total_inventory - total_reserved >= 0` prevents the count from going negative. One UPDATE will find the check passes and commit; the other will find the check fails (rowcount=0) and rollback. The database serializes concurrent updates to the same row.

**The lesson: understand that the final line of defense against double-booking is the DB-level atomic UPDATE + CHECK constraint — not application-level availability checks. Application checks can race; DB transactions cannot.**

---

### Story 3 — Expedia, L5 (2024)

The candidate's design included a "2-phase commit between the payment service and inventory service" to ensure atomicity. The interviewer asked: "Walk me through what happens when Stripe is the payment processor. Is Stripe a participant in your 2PC?"

The candidate said they'd coordinate via Stripe's API. The interviewer: "Does Stripe's API support the 2PC prepare phase? Can Stripe hold a transaction in 'prepared' state waiting for your coordinator to send commit?" The candidate looked it up mentally — Stripe's API has authorize (pre-auth) and capture (finalize), but there's no 2PC protocol. They can't participate in a distributed 2PC.

The candidate then proposed a custom 2PC coordinator. The interviewer asked what happens when the coordinator crashes between "prepare" and "commit." The candidate acknowledged this is the exact problem 2PC is known for — coordinator crash = all participants blocked indefinitely.

The interviewer guided the candidate toward SAGA: charge the card (can be compensated by refund), then update inventory (can be compensated by incrementing the counter back). If inventory update fails, issue PSP refund. This is SAGA's compensating transaction pattern.

**The lesson: 2PC requires all participants to implement the protocol. External services (PSP, hotel management systems) will never implement your proprietary 2PC. SAGA with compensating transactions is the correct pattern for cross-service atomicity involving external systems.**

---

## General Pattern Mistakes

**Mistake 1: No hold mechanism.** Proposing "check availability → charge card → confirm booking" without any hold between check and charge. In the minutes between checking availability and completing payment, the room can be booked by someone else. The hold is the mechanism that bridges this gap. Without it, every booking attempt is a race condition.

**Mistake 2: Hold without expiry worker.** Redis TTL auto-deletes the hold key, but the inventory table still shows the room as reserved. Without a hold expiry worker that decrements `total_reserved` when a hold expires without a booking, inventory is permanently lost. Over days, all inventory becomes "reserved" even though no bookings exist.

**Mistake 3: Claiming only airlines overbook.** Both hotels and airlines practice overbooking as a deliberate economic strategy. Unsold inventory on a perishable asset (tonight's hotel room, this flight's seat) is pure loss on a fixed-cost asset. Hotels "walk" guests to comparable properties; airlines bump passengers with compensation. The handling mechanism differs, but the economic motivation and the overbooking practice are the same. Saying "hotels don't overbook" is factually wrong and signals incomplete research.

**Mistake 4: Conflating hold-time with booking-time consistency.** Search can return approximate availability (a room shown as available may have been booked 30 seconds ago — the cache hasn't refreshed). Hold creation requires strong consistency (SETNX must be atomic). Booking commit requires strong consistency (DB UPDATE must be atomic). These three moments have different consistency requirements. A single consistency model applied everywhere is either too strict for search (kills performance) or too loose for booking (causes double-bookings).

**Mistake 5: Pessimistic locking for hotels without justification.** `SELECT ... FOR UPDATE` holds a row lock from selection to transaction commit. For hotel booking at 5.8 writes/sec, this is fine. But if a candidate applies pessimistic locking to flight seat selection with thousands of concurrent users selecting seats, the DB is serializing all seat selections through row-level locks — throughput collapses. Know the QPS-driven choice: low QPS = optimistic locking preferred; high contention = Redis-based hold to pre-filter, then DB atomic update.

---

## Self-Assessment Checklist

**Fundamentals:**
- [ ] Separated search path (cache, eventual consistency) from booking path (DB, strong consistency)
- [ ] Described the hold as a Redis SETNX with TTL before asking the candidate to implement it
- [ ] Named the hold TTL as a design parameter (default: 5–10 minutes) and justified it

**Concurrency:**
- [ ] Described SETNX atomicity (only one winner per hold key)
- [ ] Explained the DB-level atomic UPDATE as the correctness backstop
- [ ] Named the CHECK constraint `total_inventory - total_reserved >= 0` as last-resort guard

**SAGA:**
- [ ] Listed all 4 SAGA steps for booking (hold → charge → commit inventory → release hold/notify)
- [ ] Described compensating transaction for payment failure (release hold)
- [ ] Described compensating transaction for inventory commit failure (issue refund)

**Failure modes:**
- [ ] Addressed the hold expiry during payment race condition
- [ ] Addressed abandoned checkout (Redis TTL auto-release + expiry worker)
- [ ] Addressed server crash mid-SAGA

**Advanced:**
- [ ] Addressed overbooking policy (airlines yes, hotels usually, but can be configured)
- [ ] Justified locking choice (optimistic for hotels / Redis hold + atomic UPDATE for flights)
- [ ] Described seat map as separate sub-problem with per-seat state

---

## Remediation Targets

**If you didn't design the hold mechanism:** Think about the checkout flow. The user selects a room, is shown a price, and enters payment details — this takes 2–5 minutes. The hold is the answer to "how do you prevent the room from being booked by someone else during those 2–5 minutes?" Study Ticketmaster's seat hold design (Redis TTL = 300 seconds, Lua script for atomic SETNX).

**If you proposed 2PC:** Study the fundamental limitation of 2PC with external services. Stripe, Adyen, and Braintree have no 2PC API. The correct alternative is SAGA with compensating transactions. Know the compensation for each step.

**If you couldn't describe the hold expiry race:** Walk through the scenario: hold expires at second 300, payment completes at second 305. Two events happen in close succession. The booking service must check if the hold still exists at commit time. If not, it must refund and reject.

**If you missed the search/booking separation:** The rule is: whenever reads >> writes and freshness tolerance exists, separate read and write paths. Search reads can be stale by 60 seconds. Booking writes must be exact. This is the CQRS pattern applied to availability: writes go to the inventory DB; reads come from a pre-computed cache.
