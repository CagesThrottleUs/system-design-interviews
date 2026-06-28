# Problem 03 — Notification System at Scale

**Difficulty:** Intermediate  
**Category:** Async Messaging, Fan-out, Multi-channel Delivery  
**Appears at:** Amazon, Meta, Google, Apple, Uber, Slack, Airbnb, LinkedIn  
**Typical slot:** 45 minutes

---

## Problem Statement

You are a senior engineer at a social marketplace called **Kora** — think Instagram meets Etsy. The product team has identified that timely, relevant notifications are the single highest-leverage growth lever: users who receive a well-timed "someone liked your listing" or "your order shipped" notification return to the app 3× more often than users who don't. At the same time, the support queue has a weekly spike every Monday from users complaining about notification spam. Three users this month have publicly tweeted that they deleted the app because of it.

The VP of Product says: "We need to send notifications at scale without spamming people into churning." The VP of Engineering says: "The current system is a synchronous loop in the order-service that calls APNs directly. It falls over every Black Friday."

**Your job:** Design a notification system that is reliable, multi-channel, user-preference-aware, and capable of handling 100 million registered users across mobile push, email, and SMS — without the delivery pipeline becoming a single point of failure in the order or feed services.

---

## Actors

| Actor | Description |
|-------|-------------|
| **End User** | Registered Kora user. Has devices (iOS/Android), an email address, and optionally a phone number. Can configure per-channel opt-in/opt-out and DND hours. |
| **Producer Service** | Any internal service that triggers a notification: order-service, feed-service, marketing-service, payment-service. |
| **Notification Service** | The system you are designing. Accepts notification requests, applies user preferences, fans out to appropriate channels. |
| **Channel Provider** | External services: APNs (Apple Push Notification service), FCM (Firebase Cloud Messaging), an email provider (SES/SendGrid), an SMS provider (Twilio/SNS). |
| **Admin / Ops** | Internal team that monitors delivery rates, DLQ depths, and can manually replay failed batches. |

---

## Functional Requirements

### Core (FR1–FR6)

| ID | Requirement |
|----|-------------|
| **FR1** | The system MUST accept notification requests from internal producer services via an async API. |
| **FR2** | The system MUST support at least three delivery channels: mobile push (iOS via APNs, Android via FCM), email, and SMS. |
| **FR3** | The system MUST respect per-user, per-channel opt-in/opt-out preferences. A user who has opted out of marketing SMS must never receive a marketing SMS, regardless of what the producer service requests. |
| **FR4** | The system MUST deliver notifications with at-least-once semantics — a notification must not be silently dropped on transient provider failure. |
| **FR5** | The system MUST support notification priority tiers: **transactional** (payment confirmed, order shipped — highest priority, SLA-bound) vs **marketing** (promo, re-engagement — lower priority, can be delayed or rate-limited). |
| **FR6** | The system MUST track delivery status per notification per channel (queued, sent, delivered, failed, bounced) and expose that status to the producer service. |

### Secondary

| ID | Requirement |
|----|-------------|
| **FR7** | The system SHOULD support DND (Do Not Disturb) hours per user — marketing notifications queued during DND are held and delivered after the DND window ends. Transactional notifications bypass DND. |
| **FR8** | The system SHOULD deduplicate notifications — if the same logical event fires twice (e.g., order webhook delivered twice), the user should receive only one notification. |
| **FR9** | The system SHOULD support templating — producers supply a template key + variables, the notification service renders the final message body. This decouples producers from copy changes. |
| **FR10** | The system SHOULD maintain a device registry — mapping user IDs to device tokens — and handle token rotation automatically (expired APNs tokens, FCM token refreshes). |

---

## Non-Functional Requirements

| Attribute | Target |
|-----------|--------|
| **Throughput** | Sustain 10,000 notification events/second at peak (e.g., flash sale triggers). |
| **P99 transactional latency** | Transactional notification enqueued → sent to provider within 5 seconds. |
| **P99 marketing latency** | Marketing notification enqueued → sent to provider within 5 minutes (respects DND, batching). |
| **Delivery reliability** | ≥99.9% of transactional notifications eventually delivered (no silent drops). |
| **Availability** | Notification service API: 99.9% uptime. Provider-side failures must not surface as API errors to producers. |
| **Scalability** | Must support 100M registered users, 10M DAU, without re-architecture. |
| **Observability** | DLQ depth, delivery success rate per channel, provider error rate — all must be alertable. |
| **Compliance** | System must support GDPR right-to-erasure (purge user notification history) and CAN-SPAM opt-out within 10 business days. |

---

## Capacity Estimation Hints

Use these assumptions to anchor your napkin math:

| Parameter | Value |
|-----------|-------|
| Registered users | 100M |
| DAU | 10M |
| Avg notifications per DAU per day | 3 |
| Peak multiplier (flash sale) | 10× |
| Channel split | 70% mobile push, 20% email, 10% SMS |
| Avg notification payload | 1 KB |
| Device tokens per user (avg) | 1.5 (users have multiple devices) |
| Notification record retention | 30 days |

**Starter math:**
- Baseline throughput: 10M DAU × 3 notifications/day ÷ 86,400 s/day ≈ **350 notifications/second**
- Peak throughput: 350 × 10 = **3,500/second** (conservative); design for **10,000/second** headroom
- Push volume: 3,500 × 0.70 ≈ 2,450 push/second → **~1.7B push/day** during peak
- Daily data written: 30M total notifications × 1 KB ≈ **30 GB/day** (notification records only)

---

## Clarifying Questions

A strong candidate asks most of these before drawing a single box.

### Channels & Providers
1. Which channels must be supported at launch? Are web push and in-app notifications in scope?
2. Do we own the email/SMS infrastructure or use third-party providers (SES, Twilio)? What are their rate limits?
3. Should the system support multiple providers per channel for failover (e.g., Twilio primary → SNS fallback for SMS)?

### Delivery Semantics
4. What delivery guarantee is required — at-least-once or exactly-once? (Exactly-once is expensive; clarify if idempotent consumption is acceptable.)
5. If a push token is invalid (device wiped, app uninstalled), should we fall back to email? Or silently drop?
6. What is the retry policy on provider failure? How many retries before we move to DLQ?

### Scale
7. What is the fan-out pattern? Are most notifications 1-to-1 (order confirmation) or 1-to-many (admin broadcast to all users)?
8. What is the expected peak? Is it predictable (scheduled campaigns) or unpredictable (viral event)?

### User Preferences
9. Is per-notification-type granularity required, or just per-channel? (e.g., "opt out of marketing push but keep transactional push")
10. How quickly must an opt-out take effect? Real-time (next send) or best-effort (within a minute)?

---

## Key Components

<details>
<summary>Expand (don't read until after you attempt the design)</summary>

- **Notification API** — accepts produce requests, validates, enqueues
- **User Preference Store** — per-user, per-channel, per-notification-type opt-in flags + DND hours
- **Device Registry** — user → [device token, platform, last seen, token status]
- **Message Queue** — async fan-out buffer (Kafka or SQS); priority-segmented queues
- **Channel Workers** — pool of workers per channel: PushWorker (calls APNs/FCM), EmailWorker, SMSWorker
- **Provider Adapters** — thin wrappers per external provider (handle auth, rate limiting, error translation)
- **Delivery Log** — per-notification delivery status (queued → sent → delivered/failed)
- **Dead Letter Queue (DLQ)** — undeliverable notifications for operator replay or root cause
- **Template Engine** — renders notification body from template key + variables
- **Token Rotation Handler** — processes APNs/FCM feedback to remove stale tokens

</details>
