# Session 001: Notification System

**Date:** 2026-04-15  
**Company:** Apple  
**Role:** ICT3 (Senior Engineer equivalent)  
**Duration:** In progress

---

## Question

> Design a notification system that handles push notifications, SMS, and email at scale (500M active users globally). It's a platform — multiple first-party and third-party apps plug into it.

---

## Requirements Clarification

### Candidate's Initial Questions
1. Scale — how many users? *(Good start)*
2. Multi-app or single-app? *(Good)*

### Missed by Candidate (hint requested — noted)
The following universal 5-category framework was explained verbally. These categories apply to any distributed system design question.

**1. Scale & Load Characteristics**
- Peak vs. average throughput — e.g., 500M pushes fired simultaneously after a sports event
- Read/delivery receipts required? (affects write amplification)

**2. Delivery Semantics**
- At-least-once, at-most-once, or exactly-once?
- What happens when device is offline — store-and-forward or drop?
- TTL per notification type?

**3. Latency SLAs by Notification Type**
- Transactional (OTP, security alerts): <1s — requires different pipeline than marketing
- Marketing/promotional: minutes acceptable
- These SLAs drive architectural separation of pipelines

**4. Ordering & Deduplication**
- Do notifications need to arrive in order per user?
- Idempotency required on retries?
- Channel fallback (push → SMS) — does that affect ordering?

**5. Compliance & Opt-Out**
- GDPR opt-out, unsubscribe handling
- Do-not-disturb windows (timezone-aware delivery)
- Multi-device targeting: all devices or most recent?

---

## Agreed Requirements (Interviewer-provided after requirements phase)

| Category | Requirement |
|----------|-------------|
| Scale | 500M active users, multi-app platform (first & third party) |
| Notification types | Transactional (OTP, payment, security alerts) / Event-driven (sports, breaking news, burst) / Marketing (promos, newsletters) |
| Latency SLAs | Transactional <1s · Event-driven <5s · Marketing = minutes acceptable |
| Delivery semantics | Transactional = exactly-once · Event-driven = at-least-once + TTL (30s–2min) · Marketing = best-effort fire-and-forget |
| Offline behavior | Transactional + event-driven: store-and-forward up to TTL · Marketing: drop if offline |
| Channel fallback | Push → SMS fallback for transactional only |
| Receipts | Delivery receipt (notification reached device) required · Open receipt optional, per-app config |
| Multi-device | Send to all active devices; most recent prioritized for transactional |
| Compliance | GDPR opt-out enforced · DND windows by user timezone (marketing only) · Explicit per-app consent required · Region-based data residency |
| Availability | 99.99% for transactional · 99.9% for event-driven and marketing |

---

## HLD / LLD Boundary — Framework (coaching note)

**HLD answers one question:** What are the major components and how does data flow between them?

**Stop when:** You can draw a diagram where each box = named service, each arrow = named protocol.

**The test:** If your next sentence names a field, column, port, retry interval, or algorithm → crossed into LLD. Say "I'll cover that in LLD" and keep moving.

| HLD ✓ | LLD ✗ (park it) |
|-------|----------------|
| "API gateway routes to a priority queue" | "The queue uses port 5672" |
| "Separate worker pools per notification tier" | "Worker retries 3x with exponential backoff" |
| "Delivery tracker records receipts" | "Receipts table has columns: id, user_id, status, ts" |
| "Push adapter calls APNs/FCM" | "APNs connection pool size is 100" |

**Rule:** HLD names services. LLD names internals. Breadth-first in HLD — every major concern gets a box (ingestion, routing, delivery, receipts, compliance, failure handling). Don't go inside any box until LLD.

**Red flag phrase:** "So inside this component..." or "The table would have..." → stop, you're in LLD.

---

## High-Level Design

*(In progress — candidate designing)*

---

## Interviewer Probes & Responses

| Probe | Response | Assessment |
|-------|----------|------------|
| "What other requirements are you missing?" | Asked for hints | Hint noted. Explained 5-category universal framework |
| "Drive requirements gathering using the framework" | Asked another meta question about framework correctness | Second hint noted. Confirmed framework, corrected compliance dismissal |
| "Requirements gathering — go" | Full attempt — see assessment below | See below |

### Requirements Attempt #1 — Assessment

**Covered well:**
- Notification type differentiation: marketing (fire-and-forget), sports events (burst load), payment (exactly-once) — correct delivery semantic mapping per type
- Compliance questions: per-user vs baseline platform consent, extensibility — sharp and real
- Budget/cost constraints alongside latency — good addition
- Idempotency and ordering — mentioned

**Critical issue:**
- Started designing mid-requirements: said "we need an ingestion system", "device discovery is a component" — these are solution decisions, not requirements questions. Requirements phase = questions only, no components.

**Missed:**
- Read receipts (did notification reach device?) vs. delivery receipts (did user open it?) — significant storage and throughput implications
- Offline behavior: store-and-forward vs. drop when device offline
- Channel fallback: if push fails, does system fall back to SMS?
- Multi-device targeting: send to all devices or most recent?

**Category Coverage:**
| Category | Covered? | Notes |
|----------|----------|-------|
| Functional requirements | Partial | Mixed with solution design |
| Scale & load | ✓ | Good burst scenario (sports event) |
| Latency SLAs | ✓ | Per notification type |
| Delivery semantics | ✓ | Good mapping per type |
| Ordering & idempotency | Partial | Mentioned but not explicitly asked |
| Compliance | ✓ | Strong — consent model question |

---

## HLD — Candidate's Design (summarized)

**Covered:**
- Device registration service — application installs register device at install time, storing device identifier and consent
- Device manager — handles registration, stores device-to-application mapping
- SMS and email as external specialized services plugged into device manager
- Global vs regional ingestion — FQDNs per region, global ingester fans out to regional ingesters
- Application registration service — one-time setup per app, cost model, global vs regional scope
- Write-heavy database consideration — Cassandra for global application data, regional transactional DB for device data

**Critical Error:**
- Device-as-server model: candidate said devices have IP addresses, open a port, and the notification service makes inbound HTTP calls to the device. **This is wrong.**
- Reality: devices are behind NAT/cellular, cannot receive inbound connections. Devices maintain an outbound persistent TCP connection to APNs (iOS) or FCM (Android). Backend sends to APNs/FCM with a device token; APNs/FCM delivers over the persistent connection.
- For an Apple ICT3 interview, not knowing APNs is a significant gap.

**Other gaps:**
- No priority queue separation for notification tiers (acknowledged but not addressed)
- No offline store-and-forward mechanism shown
- No delivery deduplication / exactly-once guarantee mechanism
- Global fan-out to all regions is inefficient — should route to user's home region only
- Some LLD details crept in again (Cassandra selection, HTTP specifics)

---

## Score

| Area | Score (1–5) | Notes |
|------|-------------|-------|
| Requirements clarification | 2/5 | Framework correct after coaching (2 hint requests). Started designing mid-requirements. Missed read receipts, offline behavior, channel fallback independently. |
| HLD completeness | 1/5 | Fundamental error: device-as-server delivery model. Missing priority queues, async workers, APNs/FCM integration. |
| LLD depth | N/A | Session ended before LLD |
| Tradeoff reasoning | 2/5 | Identified cost as a constraint (good). Noted marketing vs transactional difference. Did not apply tradeoffs to architectural decisions. |
| Communication clarity | 3/5 | Verbose but self-aware. Good meta-cognition. Self-corrected on depth multiple times. |
| **Weighted Total** | **~9/25** | |

## Overall Feedback

**Would Hire at ICT3:** No — not yet.

**Strengths:**
- Strong meta-cognition — self-corrected on HLD depth, requirements phase discipline, and framework thinking
- Correctly identified notification type differentiation and delivery semantic mapping (marketing = fire-and-forget, payment = exactly-once)
- Added cost/budget as a design constraint — most candidates miss this
- Compliance and consent model questions were sharp

**Critical gaps to close before next session:**
1. **APNs / FCM internals** — how push delivery actually works. Device tokens, persistent connections, why you can't hit a device by IP. 30 min of reading closes this completely.
2. **Async queue-based pipelines** — Kafka / SQS for high-throughput async processing. How ingestion hands off to workers without blocking.
3. **One real production blog post** — Uber, Airbnb, or Slack notification system engineering post. Anchors all the above in real architecture.

**Learning resources identified by candidate:**
- Hello Interview — system design patterns, question breakdowns (Yelp, Uber, YouTube, etc.)
- Jordan Has No Life — fundamentals (Kafka, Hadoop, database internals, event-driven systems)

**Note:** This was session 1 and primarily a diagnostic + coaching session, not a scored interview. The score reflects performance without coaching adjustments.
