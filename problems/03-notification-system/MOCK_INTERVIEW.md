# Mock Interview Script — Notification System at Scale

**Format:** 45 minutes, Apple ICT3 / Meta E5 / Amazon SDE2 level  
**Interviewer role:** Read every line aloud. Do not skip probes. Mark the scorecard during the session.

---

## Pre-Interview Setup (2 min, not timed)

Before starting the clock, confirm:
- Candidate has a whiteboard or shared drawing tool
- Candidate understands the session will have four phases
- Candidate knows they can ask clarifying questions at any point

Say to candidate:
> "We're going to work through a system design problem. I'll give you a prompt, and I'd like you to walk me through your design in real time — thinking out loud as you go. We have about 45 minutes. Ready?"

---

## Phase 1 — Problem Statement & Requirements (0:00–5:00)

### Deliver the prompt

> "Design a notification system for a social marketplace. The platform has 100 million registered users, 10 million DAU. Users interact with listings, orders, and social content. The product team wants to send users timely notifications — order updates, social activity, marketing promotions — across mobile push, email, and SMS. The current implementation sends notifications synchronously from the order service. It falls over on high-traffic events. Your job is to design a scalable, reliable, multi-channel notification system."

*Start the clock.*

---

### Requirements Clarification: What to listen for

A strong candidate will ask clarifying questions before touching the whiteboard. Score against these:

**Channels (must ask at least one)**
- "Which channels must be supported at launch — push, email, SMS? In-app? Web push?"
- "Do we own the provider integrations or are we on top of APNs, FCM, SES, Twilio?"

**Delivery semantics (must ask at least one)**
- "What delivery guarantee is required — at-least-once or exactly-once?"
- "If push fails, should we fall back to email automatically?"

**Scale (must ask at least one)**
- "What's the peak QPS? Is the 10× flash sale scenario real?"
- "What's the fan-out shape — mostly 1-to-1 (order confirmation) or do we have broadcast campaigns?"

**User preferences (should ask)**
- "Do users need granular opt-out — per-channel? Per-notification-type?"
- "Are DND hours a requirement?"

**If candidate jumps to architecture without asking ANY clarifying questions:**
> "Before you start drawing, what questions do you have about the requirements?"

If they still don't ask after that prompt, note it on the scorecard.

---

### Feed these answers when asked (or when candidate starts designing)

| Question | Answer |
|----------|--------|
| Channels | Push (iOS + Android), email, SMS. In-app is out of scope for today. |
| Providers | Use APNs, FCM, SES or SendGrid, Twilio. |
| Delivery guarantee | At-least-once for transactional. Best-effort for marketing. |
| Fallback across channels | Not required; optional stretch goal. |
| Fan-out shape | Mostly 1-to-1. Marketing campaigns up to 10M users. |
| Peak QPS | Design for 10,000 notifications/second at peak. |
| User opt-out | Per-channel opt-out required. Per-notification-type is a stretch goal. |
| DND hours | Yes, per-user DND required. Marketing only — transactional bypasses DND. |
| Priority | Two tiers: transactional (P99 ≤ 5s to provider) and marketing (≤5 min, can be delayed). |
| Delivery tracking | Yes — producers need to check notification delivery status. |

---

## Phase 2 — High-Level Design (5:00–25:00)

### What to expect from a strong candidate

Within 5 minutes, the candidate should have:
1. Identified the core problem: synchronous delivery inside producer services = tight coupling + fragility
2. Proposed async queue-based decoupling as the primary architectural decision
3. Named the major components: notification API, queue, fan-out workers, channel workers, providers

**Do NOT proceed with this list** — let the candidate discover it. Only prompt if stuck.

---

### Probing questions (use as candidate draws; pick what's relevant)

**On the queue:**
> "You've drawn a queue between the notification service and the providers. What specific technology are you using, and why?"

*Listen for:* Kafka vs SQS tradeoffs. Kafka = higher throughput, ordering per partition, log retention for replay. SQS = simpler, managed, no log replay. Netflix RENO uses SQS. Either is defensible; listen for the reasoning.

> "How do you ensure that a marketing blast to 10 million users doesn't delay a 'payment declined' transactional notification that arrives while the campaign is running?"

*Listen for:* Priority separation. Not a "priority field" — physically separate Kafka partitions or separate topics with separate consumer groups. If candidate says "I'd just sort by priority," push back:
> "With a single queue and a priority field, consumers process in insertion order unless you build priority ordering into the consumer. How would you actually implement that?"

**On the user preference layer:**
> "Walk me through what happens when a user clicks 'Unsubscribe' in a marketing email."

*Listen for:* Preference store update, propagation to all providers' suppression lists, the preference check happening before every delivery attempt. If they say "SendGrid handles that":
> "What happens when the marketing-service sends the next campaign batch directly to SendGrid using its own user list? Would that user be protected?"

If candidate has no preference store in the design, escalate:
> "Are you familiar with CAN-SPAM? What's the legal obligation on opt-out processing time?"

**On device tokens:**
> "You've drawn a box for APNs. Walk me through what happens when APNs returns a 410 status code."

*Listen for:* 410 Unregistered = token permanently invalid. Mark INVALID in device registry. Never retry. This is the non-obvious answer. If candidate says "retry":
> "The 410 means the device token no longer exists — the app was uninstalled or the device was wiped. Why would retrying help?"

> "What HTTP version does APNs require, and why does that matter for your worker's connection model?"

*Listen for:* HTTP/2, persistent connections, connection pool, up to 1,000 concurrent streams per connection. If candidate doesn't know:
> "APNs dropped HTTP/1.1 support in 2021. HTTP/2 uses multiplexed streams over a persistent connection. How does that change how you'd design your APNs worker?"

**On delivery reliability:**
> "Your push worker calls APNs and APNs returns a 503 — service unavailable. Walk me through exactly what happens."

*Listen for:* Retry logic, exponential backoff with jitter, classification of retriable vs. non-retriable errors, eventually writing to DLQ after max retries. If candidate says "log and move on":
> "So the user never gets the notification? Is that acceptable for a 'payment failed' notification?"

> "After 5 failed retry attempts, what happens to the notification?"

*Listen for:* DLQ. Ops visibility. Ability to replay. If candidate has no DLQ:
> "How does the ops team know when the system is silently dropping transactional notifications?"

**On fan-out at scale:**
> "The marketing team wants to send a flash sale notification to all 10 million active users. Walk me through how that fan-out works and how long it takes."

*Listen for:* Napkin math. 10M users / 10K/s = 1,000 seconds = ~17 minutes. Is 17 minutes acceptable for a marketing campaign? (Usually yes.) Worker pool sizing. Rate limiting per provider (FCM 600K/minute quota). If candidate handwaves:
> "Roughly how long would it take to fan out to 10M users at the throughput we established earlier?"

---

### HLD Boundary Enforcement

If the candidate starts naming database columns, specific algorithm implementations, or internal data structures:
> "That sounds like detail we'd cover in the low-level design. For now, let's stay at the component level — what are the major services and how do they communicate? We'll drill into internals in the next phase."

---

### Transition to LLD (at 20–22 minutes)

> "Good. I have a solid picture of the architecture. Let's go deeper on a few specific areas. I want to focus on: the data model, delivery reliability, and the device registry. Which would you like to start with?"

*Let candidate choose — their choice reveals what they're confident about.*

---

## Phase 3 — Low-Level Design (25:00–40:00)

### Data model probe

> "Sketch out the data model for a notification record and the device registry. What are the key fields, what's the primary key, and what indexes do you need?"

*Listen for:*
- Notification record: id, idempotency_key (unique index for dedup), user_id, priority, template_key, template_vars, status, created_at, expires_at
- Device registry: user_id (partition key), device_id (clustering key), platform, token, token_status, last_seen_at
- Candidate should explain why device registry is in Cassandra (write-heavy on token updates) vs. notification records potentially in PostgreSQL

> "You have a unique index on idempotency_key. What's your deduplication strategy if the order-service sends the same 'order shipped' event twice because of a webhook retry?"

*Listen for:* The unique constraint causes the second INSERT to fail or return the existing record. Return the existing notification_id to the producer. No second enqueue. No duplicate notification to user. At-least-once in the queue layer only fires once because the idempotency_key gates the queue publish.

### Delivery reliability deep dive

> "Walk me through the full lifecycle of a transactional notification — from POST to the notification API to the user's phone lighting up — including what happens if APNs fails on the first attempt."

This is a verbal walkthrough test. Listen for completeness:
1. POST /notifications → API validates → checks idempotency key → writes notification record (status=QUEUED) → publishes to Kafka notifications.raw transactional partition → returns 202
2. Fan-out worker: reads from Kafka, reads user preferences from Redis cache, resolves device tokens from device registry, publishes to chan.push.transactional
3. Push worker: reads from chan.push.transactional, renders template, calls APNs via HTTP/2
4. APNs returns 503: worker classifies as retriable, schedules retry at T+5s (attempt 2), eventually T+30s (attempt 3), T+2m (attempt 4), T+10m (attempt 5)
5. If APNs returns 200 on attempt 3: worker updates delivery_log status=SENT, commits Kafka offset
6. If all 5 attempts fail: worker writes to DLQ topic, updates delivery_log status=FAILED
7. APNs may return new token in apns-device-token header: worker updates device_registry

If the candidate skips any step, probe: "What happens to the Kafka offset after the APNs call?"

### Concurrency and scaling probe

> "Your push worker pods are processing from the chan.push.transactional Kafka topic. How many pods do you need to sustain 3,500 iOS push/second? Walk me through the math."

*Expected reasoning:*
- APNs HTTP/2: assume 100ms round-trip per request, 1 stream per request → 10 requests/second per stream
- 1,000 streams per HTTP/2 connection → 10,000 requests/second per connection
- But HTTP/2 streams have setup overhead; practical throughput with pipelining is lower
- Assume 1,000 effective requests/second per connection (conservative)
- 3,500 iOS push/second → 3–4 APNs connections needed
- 1 pod manages N connections → 1–2 pods for iOS at this scale; run 3 for redundancy

*Key point the candidate should make:* Connection reuse is the critical insight. Opening a new TCP+TLS connection per notification would be catastrophically slow — each takes ~100–200ms. HTTP/2 connection pooling reduces amortized overhead to near zero per notification.

### Preference check timing probe

> "You check user preferences in the fan-out worker. User A changes their preference from 'push enabled' to 'push disabled' at the exact moment a notification for User A is mid-flight in the fan-out worker. What happens?"

*Listen for:* This is a race condition. The fan-out worker may have already read the old preference from Redis cache before the update propagated. The user may receive one more notification after disabling push. This is acceptable — the system is eventual consistency, not linearizable. The fix is not to make preference reads linearizable (too expensive) but to make the cache TTL short enough (e.g., 60 seconds) and to handle user complaints via support. CAN-SPAM's 10-business-day window provides legal cover for this window.

---

## Phase 4 — Wrap-Up & Failure Modes (40:00–45:00)

### Failure mode round

Pick two from this list based on what the candidate has not already covered:

> "What happens if the Redis preference cache is completely unavailable?"

Expected: Fall back to reading preferences from PostgreSQL directly. Slower (10ms vs. 0.1ms) but functionally correct. No notifications dropped. Cache miss path must exist and be tested.

> "FCM is having a major outage. It's returning 503s for every request. Walk me through the next 30 minutes in your system."

Expected: Attempt 1 fails → exponential backoff → attempts 2–5 fail → DLQ. DLQ depth spikes → alert fires → ops paged. Kafka consumer lag on chan.push.android grows. When FCM recovers, workers resume. Ops replays DLQ. Describe how long the DLQ replay takes and whether you'd rate-limit the replay to avoid immediately re-triggering MessageRateExceeded errors on FCM recovery.

> "What's the weakest part of your design?"

*This is the self-awareness test.* Good answers: "The fan-out worker is a potential hotspot if the Redis cache is cold — I haven't designed a circuit breaker there." Or: "The device registry in Cassandra — if schema changes are needed, Cassandra migrations are painful." Or: "My DLQ is a Kafka topic, but I haven't designed the operator console to replay it safely — a bad replay could re-deliver millions of stale notifications."

> "What would you do differently with more time?"

Good answers: Exactly-once delivery for transactional (Kafka transactions + idempotent producers), per-notification-type preference granularity, delivery receipt callbacks from APNs (APNs supports delivery receipts in some configurations), ML-based notification ranking to reduce volume while increasing click-through (Meta deployed this in Oct 2022 and reduced notification volume while increasing CTR).

---

## Scoring Checklist

Score each area 1–5.

### Requirements Clarification (weight: 10%)

| Check | Score |
|-------|-------|
| Asked about channels before drawing | 0 or 1 |
| Asked about delivery semantics | 0 or 1 |
| Asked about priority tiers | 0 or 1 |
| Asked about user preferences/opt-out | 0 or 1 |
| Asked about scale/QPS | 0 or 1 |

Score: sum / 5 → convert to 1–5 scale (0 = 1, 5 = 5).

### HLD Completeness (weight: 30%)

| Check | Score |
|-------|-------|
| Decoupled producers from delivery via async queue | 0 or 1 |
| Correct fan-out architecture (API → queue → worker → channel workers → providers) | 0 or 1 |
| Priority separation at queue/partition level | 0 or 1 |
| User preference layer present and checked before every delivery | 0 or 1 |
| Device registry present | 0 or 1 |
| Channel abstraction layer (producers don't know APNs vs FCM) | 0 or 1 |
| Delivery tracking / status system | 0 or 1 |

Score: sum / 7 → convert to 1–5 scale.

### LLD Depth (weight: 30%)

| Check | Score |
|-------|-------|
| Notification data model: idempotency_key, priority, status, expires_at | 0 or 1 |
| Device registry: partition by user_id, token_status field, last_seen_at | 0 or 1 |
| Retry state machine with explicit backoff schedule | 0 or 1 |
| Non-retriable error classification (410 Unregistered → mark INVALID, never retry) | 0 or 1 |
| DLQ with ops visibility | 0 or 1 |
| Deduplication via idempotency key | 0 or 1 |
| APNs HTTP/2 connection pool (not new connection per notification) | 0 or 1 |
| Token rotation handling (read apns-device-token header, update registry) | 0 or 1 |

Score: sum / 8 → convert to 1–5 scale.

### Tradeoff Reasoning (weight: 20%)

| Check | Score |
|-------|-------|
| Explicitly rejected synchronous delivery and named the reason | 0 or 1 |
| Named at-least-once vs. exactly-once tradeoff and chose deliberately | 0 or 1 |
| Named Kafka vs. SQS with reasoning | 0 or 1 |
| Explained priority queue design (separate partitions, not just a field) | 0 or 1 |
| Named a weakest part of their design | 0 or 1 |

Score: sum / 5 → convert to 1–5 scale.

### Communication Clarity (weight: 10%)

| Check | Score |
|-------|-------|
| Named components before detailing them | 0 or 1 |
| Did napkin math on at least one sizing question | 0 or 1 |
| Structured the design top-down (API → queue → workers → providers) | 0 or 1 |
| Did not get lost in implementation details during HLD phase | 0 or 1 |
| Admitted uncertainty rather than bluffing | 0 or 1 |

Score: sum / 5 → convert to 1–5 scale.

---

## Final Score Calculation

```
Requirements (10%):   ___ / 5  × 0.10 = ___
HLD Completeness (30%): ___ / 5 × 0.30 = ___
LLD Depth (30%):        ___ / 5 × 0.30 = ___
Tradeoff Reasoning (20%): ___ / 5 × 0.20 = ___
Communication (10%):    ___ / 5 × 0.10 = ___
                                  Total: ___ / 5
```

**Hire bar (approximate):**
- < 2.5: No hire
- 2.5–3.4: Borderline / needs more prep
- 3.5–4.2: Hire (SDE2 / E4 level)
- > 4.2: Strong hire (Senior / E5 / ICT3 level)

---

## Debrief Prompts (post-session, 5 min)

> "What do you think went well in your design?"

> "If you were to redo this in a second attempt, what would you change?"

> "Were there any areas where you felt you didn't have enough depth?"

After candidate answers, share gaps from the scoring checklist — specifically any item they missed that is in the solution guide. Do not share the solution guide in full.
