# Design a Notification System

**Metadata**

| Field | Value |
|-------|-------|
| Difficulty | Medium (Tier 1) |
| Category | Messaging / Asynchronous Delivery |
| Time Box | 45 minutes |
| Key Concepts | Fan-out, message queues, push vs pull, idempotent delivery, APNs / FCM |
| Known Asked At | Meta, Apple, Google, Amazon, Uber, LinkedIn |

---

## Problem Statement

You've been asked to design the notification infrastructure for a social platform with 100 million daily active users. Users receive push notifications for mentions, likes, direct messages, and system alerts. The system must reliably deliver notifications to iOS and Android devices and to web browsers, and it must continue working correctly when individual users are on intermittent mobile connections — delivering any queued notifications when they come back online. The business priority is delivery reliability; it is more acceptable to send a notification twice than to silently drop one.

---

## Actors

**Publisher** — Any backend service within the platform that wants to trigger a notification. This includes the feed service (when a post is liked), the messaging service (when a DM arrives), the mention detection pipeline (when a username is detected in content), and the alerting system (for account-level events like a password reset or a login from a new device). Publishers know nothing about device tokens, APNs certificates, or delivery infrastructure — they fire a logical event and the notification system handles the rest.

**Consumer — Mobile Client** — iOS or Android apps installed on user devices. These clients receive push notifications via Apple Push Notification Service (APNs) or Firebase Cloud Messaging (FCM) respectively. They register their device tokens with the platform on first launch and on token refresh. They may be offline for minutes, hours, or days; when they reconnect, they expect to receive any notifications they missed, subject to TTL expiry.

**Consumer — Web Client** — Browser-based clients that use the Web Push protocol (RFC 8030) to receive notifications via a service worker. Web push has different token management semantics from APNs and FCM but the same delivery model at the logical level.

**Admin** — Internal operators who can inspect delivery rates, view dead-letter queue contents, trigger redelivery, and configure per-notification-type TTLs and rate limits. The admin persona is out of scope for this problem but the data model must accommodate it.

---

## Functional Requirements — Core MVP

1. **Send a notification to a specific user.** Given a user ID and a notification payload (type, title, body, deep link), the system must deliver that notification to all active device tokens registered to that user across all supported platforms (iOS, Android, web).

2. **Register and manage device tokens.** Clients must be able to register a device token, refresh a stale token, and deregister a token when the user logs out. The system must handle APNs token invalidation responses (when APNs tells us a token is no longer valid, we must purge it).

3. **At-least-once delivery with deduplication.** A notification must be delivered at least once to each registered device. If delivery fails (APNs timeout, network error), the system must retry. If a notification is accidentally enqueued twice, the client should only display it once.

4. **Delivery status tracking.** The system must track the delivery state of each notification per device: enqueued, delivered (acknowledged by APNs/FCM), failed (unrecoverable), or expired (TTL elapsed without delivery).

5. **Support notification types with configurable TTL.** Different notification types have different urgency and TTL values. A "new DM" notification is urgent and expires after 24 hours. A "weekly digest" notification is low priority and expires after 7 days. A "password reset" notification must be delivered within 60 seconds or not at all.

---

## Functional Requirements — Secondary (V2)

1. **User notification preferences.** Users can opt out of specific notification types (e.g., "don't send me like notifications"), set quiet hours, and manage per-device preferences. The fan-out layer must check preferences before dispatching.

2. **Notification batching and digest.** For high-volume scenarios (a viral post receiving thousands of likes per minute), the system should be able to batch multiple notifications of the same type into a single summary ("Alice and 47 others liked your post") rather than flooding the user.

3. **Analytics and funnel tracking.** Track open rates, click-through rates, and delivery latency by notification type, platform, and user cohort. This data feeds the product experimentation loop.

4. **Priority queuing.** High-priority notifications (DMs, security alerts) jump ahead of low-priority ones (promotional, digest) in the delivery queue. Under load shedding, low-priority notifications are dropped first.

5. **Scheduled notifications.** Publishers can enqueue a notification with a future delivery time (e.g., a scheduled appointment reminder). The system must hold it until the delivery window and then dispatch.

---

## Non-Functional Requirements

| Dimension | Requirement |
|-----------|------------|
| Scale — DAU | 100 million |
| Scale — Notification volume | 300 million notifications per day (3 per user on average) |
| Write QPS (ingestion) | ~3,500 notifications/sec average; ~15,000/sec peak (5× burst) |
| Read QPS (status queries) | ~2× write QPS |
| Delivery latency (p99) | < 5 seconds from publisher event to APNs/FCM handoff |
| APNs / FCM round-trip | < 500ms p99 (not in our control, but we must handle their SLAs) |
| Availability | 99.9% uptime on the ingestion path; 99.5% on the delivery path |
| Durability | Zero notification loss after the event is accepted into the queue |
| Storage retention | Delivery status logs retained for 30 days; notification payloads retained for 7 days |
| Consistency | Eventual; devices may receive notifications out of order if the device was offline |

---

## Out of Scope

The following topics are real engineering concerns but are explicitly out of scope for a 45-minute session. You will not be penalized for not addressing them, but if you bring them up with a clear "I'd handle this in v2 by..." you earn credit for breadth.

1. **Notification rendering and deep linking.** How the client UI renders the notification, handles deep link routing, and tracks user interaction with the notification is owned by the client teams, not the notification infrastructure.

2. **The upstream event detection systems.** How the mention detection pipeline identifies @mentions in post content, how the like counter decides when to trigger a notification vs batch, and how the feed ranking system decides what to notify about — all of these are owned by other teams. Your system consumes a normalized notification event from a queue; what's upstream of that queue is not your problem.

3. **GDPR and notification data retention compliance.** Right-to-erasure workflows, data export, and cross-border data residency rules require significant legal and infrastructure work and will be handled as a dedicated compliance project.

---

## Capacity Estimation Hints

The following starting numbers are sufficient to derive all estimates you need. Do not look up answers — work through the math yourself.

- 100M DAU
- On average, each user receives 3 notifications per day
- Assume a typical user has 2.5 registered devices (some users have a phone, tablet, and laptop)
- Average notification payload size: 512 bytes (title, body, metadata, deep link URL)
- Delivery status record per notification per device: ~200 bytes
- APNs token size: 32 bytes. FCM token size: ~150 bytes.
- Assume 70% iOS, 25% Android, 5% web
- Peak traffic is 5× the average rate

From these numbers you can derive: total notifications per day → notifications per second at average and peak → total device-notification records per day → storage per day for payloads vs. for status records → total token storage. Which tier is the bottleneck?

---

## Good Clarifying Questions

> **STUDY ONLY — these are not revealed during interview mode. Use this section after your session to check whether you asked the right questions.**

A strong candidate would ask the following within the first five minutes:

1. **Delivery semantics:** "Does every notification need to be delivered, or is best-effort acceptable for certain types?" (The answer is that at-least-once is required for DMs and security alerts; best-effort is fine for likes.)

2. **Device token lifecycle:** "How do we handle a user who reinstalls the app? Their old token is invalid but we won't know until APNs rejects it." (This surfaces the token invalidation feedback loop as a real problem.)

3. **Offline behavior:** "If a user is offline for 72 hours, what happens to the notifications we tried to send? Do they all get delivered on reconnect, or just the most recent N?" (This surfaces TTL design as a core concern, not an afterthought.)

4. **Fan-out scale:** "Are there users with very large follower counts who post content that triggers notifications at massive scale?" (This surfaces the celebrity problem — a single event triggering 10M notifications — which requires a different fan-out architecture.)

5. **Rate limiting:** "If a single post gets 50,000 likes in a minute, do we send 50,000 individual 'Alice liked your post' notifications, or do we aggregate?" (This surfaces the batching/throttling requirement.)

6. **Notification priority:** "Are all notification types treated equally, or do DMs and security alerts need to be prioritized over likes?" (This surfaces priority queue design.)

7. **Idempotency:** "If the publisher service retries an event after a timeout, how do we prevent the user from receiving the same notification twice?" (This surfaces idempotency key design at the API layer.)

---

## Key Components

> **HINT — attempt to identify these yourself before reading. These are the components that appear in a strong HLD for this problem. Cover them first, then check.**

1. **Notification Ingestion API** — The entry point for publishers. Accepts a normalized notification event and enqueues it.
2. **Notification Queue** — A durable message queue (Kafka or SQS) that decouples ingestion from delivery.
3. **Fan-out Service** — Reads notification events from the queue, resolves the target user's device tokens, and writes per-device delivery tasks.
4. **Device Token Store** — A database mapping user IDs to their registered device tokens across platforms.
5. **Per-Platform Delivery Workers** — Separate worker pools for APNs, FCM, and Web Push. Each knows how to authenticate and format requests for its target platform.
6. **Delivery Status Store** — Tracks the state of each notification per device (enqueued, delivered, failed, expired).
7. **Dead Letter Queue** — Holds notifications that have failed all retry attempts for manual inspection and potential redelivery.
