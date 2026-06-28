# Hotel / Flight Booking System

**Difficulty:** Advanced
**Category:** Consistency-critical, Inventory management
**Time Box:** 45 min
**Key Concepts:** Inventory management under concurrent writes, soft-lock (hold) pattern, pessimistic vs optimistic locking, SAGA vs 2PC, overbooking policy
**Asked at:** Airbnb, Booking.com, Expedia, Lyft (ride booking variant)

---

## Problem Statement

You are building a hotel and flight booking platform. Travelers search for available rooms or seats, select one, enter payment details, and complete a booking. The inventory — hotel rooms available on a specific date, airline seats on a specific flight — is finite and heavily contended. Two travelers looking at the same listing at the same time must not both successfully book the same room or seat.

The engineering challenge is the gap between search and booking. A traveler searches, sees a room is available, spends 5 minutes comparing prices, and then clicks "Book Now." In those 5 minutes, the room may have been booked by someone else. If the system says "room available" until payment completes, it may simultaneously tell 50 people the same room is available — and then only one can succeed, leaving 49 frustrated users with failed payments. If the system reserves the room the moment someone starts checkout, it may hold inventory hostage for users who never complete payment.

For real-world context: Booking.com lists over 28 million accommodation options globally. Expedia previously allowed only one concurrent API request per property to prevent overloading small hotels. Ticketmaster uses Redis-based seat holds with a 300-second (5-minute) TTL. ByteByteGo's reference design for hotel booking pre-populates `room_type_inventory` tables for up to 2 years ahead — 5,000 hotels × 20 room types × 730 days = 73 million rows in a single database.

---

## Actors

| Actor | Description | Primary Action |
|-------|-------------|----------------|
| **Traveler** | Person searching and booking travel | Search → Select → Checkout → Pay |
| **Hotel / Airline** | Inventory owner | Manages room/seat availability, accepts/rejects bookings |
| **Platform** | Booking.com, Expedia equivalent | Aggregates inventory, processes transactions |
| **Payment Service** | PSP (Stripe, Adyen) | Charges traveler's card |
| **OTA Partners** | Other online travel agencies | Supply external inventory feeds |

---

## Functional Requirements

**Core (MVP):**
- FR1: Search available rooms/seats by date/destination with real-time inventory
- FR2: Reserve (soft-lock) a specific room or seat for a traveler during checkout
- FR3: Complete a booking by charging payment and confirming the reservation
- FR4: Release a hold automatically if payment is not completed within N minutes
- FR5: Cancel a confirmed booking with appropriate refund based on cancellation policy

**Secondary:**
- FR6: Handle concurrent booking attempts for the same inventory — exactly one succeeds
- FR7: Support overbooking configuration per property (airlines may enable, hotels typically don't)
- FR8: Notify the traveler and property owner of booking confirmation via email/push
- FR9: Display seat map for flights with real-time per-seat availability
- FR10: Support multi-room / multi-leg bookings as a single transaction

---

## Non-Functional Requirements

| Requirement | Target | Rationale |
|-------------|--------|-----------|
| Booking correctness | Zero double-bookings | Two bookings for same room/night = both guests arrive to no room — severe customer failure |
| Search latency (p99) | < 500 ms | Travelers compare prices across tabs; slow search = abandoned session |
| Checkout latency (p99) | < 3 seconds | Acceptable wait during payment; higher latency = cart abandonment |
| Hold TTL | 5–10 minutes | Enough time to complete payment; short enough to prevent inventory stagnation |
| Consistency | CP (consistency over availability) during reservation | Prefer temporary unavailability over double-booking |
| Availability | 99.99% for search; 99.9% for booking | Search can tolerate cached/slightly-stale results; booking must work |
| Scale | 10M searches/day, 500K bookings/day | Reads far outnumber writes; search path can serve cached inventory |

---

## Capacity Estimation Hints

Use these to derive your own numbers:

- **Search volume:** 10 million searches/day; 95% never result in a booking
- **Booking volume:** 500,000 bookings/day
- **Concurrent checkouts (peak):** ~10,000 simultaneous users in checkout flow
- **Average hold duration:** 4 minutes (users who complete payment); 8 minutes (users who abandon)
- **Seat map requests:** flights generate 20× more inventory lookups than hotels (per-seat vs. per-room)
- **Inventory rows (hotel):** 5,000 hotels × 20 room types × 730 days = 73 million rows
- **Inventory rows (flight):** 10,000 flights/day × 200 seats = 2 million rows (per-day view)

Derive: booking write QPS, peak concurrent holds, storage for inventory tables, seat map read QPS.

---

## Good Clarifying Questions to Ask

### Inventory Semantics
1. Is this hotels only, flights only, or both? (Seat-level locking for flights is much more granular than room-type locking for hotels)
2. Do we support overbooking? For which inventory types? (Airlines intentionally overbook 5–10%; hotels usually don't; changes the inventory counter design)
3. When a user starts checkout, do we immediately soft-lock the room/seat? Or only upon payment confirmation? (Core design question — drives the hold architecture)

### Concurrency
4. What happens when two users try to book the same last room simultaneously? Do we prefer to let both fail and retry, or does one win atomically? (Optimistic vs. pessimistic locking choice)
5. What's the hold TTL? (Determines how long inventory is unavailable to other searchers during a stalled checkout)

### Scale
6. How many simultaneous searches vs. bookings? (Drives separation of search path from booking path)
7. Do we need real-time seat maps for flights, or is approximate availability sufficient for search? (Seat maps require per-seat state — much higher read amplification)

### Consistency
8. Is it acceptable for search to show slightly stale availability (e.g., a room that was just booked appears available for up to 60 seconds)? (Yes for search; No for the actual booking transaction)
9. Do we need cross-service atomicity — charge card AND reduce inventory must both succeed or both fail? (Core SAGA design question)

### Compliance
10. Are there regulatory requirements on booking confirmation timing or cancellation rights? (EU/UK passengers rights, hotel consumer protection laws)

---

## Key Components

<details>
<summary>Reveal components</summary>

- **Search Service** — handles search queries; serves from cached/pre-computed inventory; eventual consistency acceptable
- **Inventory Service** — manages room_type_inventory / seat inventory; strongly consistent; single source of truth for "is this available?"
- **Hold Service** — creates soft-lock holds with TTL in Redis; prevents double-booking during checkout window
- **Booking Service** — orchestrates the booking SAGA: hold → charge → confirm → release hold
- **Payment Service** — wraps PSP; idempotent charge with Idempotency-Key
- **Confirmation Service** — sends booking confirmation to traveler and property owner
- **Hold Expiry Worker** — background process that releases expired holds and returns inventory to available pool
- **Seat Map Service** — serves per-seat availability for flight seat maps; fine-grained locking per seat

</details>
