# Lessons Learned — Notification System at Scale

These are composite narratives drawn from real interview patterns at top companies. Names are anonymized. Use them to stress-test your design before the interview.

---

## Story 1: The Meta Round — The Opt-Out Trap

**Setting:** Senior engineer candidate, Meta E5 loop. The interviewer was a staff engineer on the notification infrastructure team.

**What happened:**

The candidate's design was technically solid — Kafka fan-out, channel workers, retry logic. The diagram was clean. Then the interviewer asked a single question: "Walk me through what happens when a user clicks 'Unsubscribe' in a marketing email."

The candidate said: "The email provider (SendGrid) handles that — they have an unsubscribe mechanism built in."

The interviewer paused, then said: "And what happens when the marketing-service sends the next campaign batch tomorrow?"

The candidate said: "Well, they'd pull the user list and... oh."

The system had no preference store. The opt-out existed only in SendGrid's suppression list. The marketing-service compiled its own user list from the product database, bypassed the notification service entirely, and sent directly to SendGrid — it would never see the suppression list. The next campaign blast would hit the opted-out user.

The interviewer then asked: "Are you familiar with CAN-SPAM?" The candidate was not.

**The escalation:** The interviewer asked what the legal obligation was on unsubscribe processing time. The candidate guessed "immediate." The correct answer is that CAN-SPAM requires honoring opt-out requests within 10 business days — but most companies honor it within minutes to avoid reputation damage and deliverability penalties. GDPR (EU) is stricter: the right to object under Article 21 requires "immediate" cessation pending business-grounds review.

**What the candidate should have done:**

Before drawing any boxes, establish the opt-out architecture as a first-class component:

1. User preferences stored in a central preference store (not in the email provider's suppression list).
2. All notification paths — including direct producer-to-provider integrations — must go through the preference check layer.
3. Opt-out propagates to all providers: notification service updates the preference store, which triggers a sync to all provider suppression lists (SendGrid, Twilio, APNs opt-out).
4. Audit log of opt-out events for compliance reporting.

**Key lesson:** The preference layer is not a nice-to-have feature — it is a legal requirement. Design it before you design delivery. Every path that delivers a notification must check preferences. If the design has any bypass path (producer calling provider directly), that bypass path is a compliance liability.

---

## Story 2: The Amazon Round — The "Fire and Forget" Disaster

**Setting:** SDE2 candidate, Amazon SDE2 loop. Interviewer was from the Alexa notifications team.

**What happened:**

The candidate's design:
- Order service calls notification service via HTTP POST.
- Notification service calls APNs/SES directly, returns 200 on success.

The interviewer probed: "What happens if APNs returns a 503 during a high-traffic event?"

Candidate: "The notification service catches the error and returns a 500 to the order service. The order service logs it."

Interviewer: "So the user never gets the notification?"

Candidate: "Right, but that's APNs' fault."

Interviewer: "Walk me through your retry strategy."

Candidate: "The order service could retry the POST to the notification service."

Interviewer: "The order service is in the middle of a checkout flow. How many times does it retry, for how long, with what backoff, and what does the checkout response to the customer look like while retrying?"

The candidate realized the design had conflated the notification delivery path with the checkout critical path. Retrying a slow provider inside a user-facing request is a latency disaster. The candidate had no queue, no async delivery, no retry state machine outside the order-service.

The interviewer escalated: "Describe your delivery guarantee. What percentage of users get the 'order confirmed' notification after a successful order placement?"

Honest answer: "Unknown — depends on provider availability at the moment the order completes."

**The real failure mode:** At Amazon Prime Day peak, APNs returns elevated 503 rates. Every order-service thread that processes a notification delivery is now blocked on a slow APNs call. Thread pool exhaustion. Order-service degradation. All because the notification path was synchronous and tightly coupled to the checkout path.

**What the candidate should have done:**

1. **Decouple delivery from the producer's critical path.** The order-service publishes an event to Kafka (fire-and-truly-forget for the order-service). Returns immediately. Notification delivery is async.
2. **Define delivery guarantees.** "We guarantee at-least-once delivery for transactional notifications, with retries up to 5 attempts over a 10-minute window. If all fail, the notification lands in the DLQ and ops is alerted within 60 seconds."
3. **Design the retry state machine explicitly.** Not "retry somewhere" but "attempt 1 immediate, attempt 2 at T+5s, attempt 3 at T+30s, attempt 4 at T+2m, attempt 5 at T+10m, then DLQ."
4. **Show the DLQ.** Ops needs a way to replay failed notifications after provider recovery. Without a DLQ, permanently failed notifications are silently lost.

**Key lesson:** "Fire and forget" with no retry is not a delivery guarantee — it's a delivery hope. The interviewer is testing whether you understand that reliability requires explicit retry state machines, async decoupling, and observable failure handling.

---

## Story 3: The Apple/Uber Round — The APNs Black Box

**Setting:** ICT3 candidate, Apple round. Interviewer was from the Apple Push Notification infrastructure team. (Yes, they will test you on APNs internals when interviewing at Apple.)

**What happened:**

Candidate described their push delivery component as: "A worker that calls the APNs REST API for each device token."

Interviewer: "What HTTP version does APNs require?"

Candidate: "HTTP/1.1? Standard REST?"

Interviewer: "APNs dropped HTTP/1.1 support in 2021. It requires HTTP/2. Why does that matter for your worker design?"

The candidate did not know. They did not know that:
- HTTP/2 uses multiplexed streams over a single persistent connection
- APNs allows up to 1,000 concurrent streams per connection
- A naive implementation that opens a new HTTP/1.1 connection per notification has enormous overhead: TCP handshake + TLS negotiation + HTTP request for every notification
- An HTTP/2 connection pool is required: open N connections (to maintain N×1,000 concurrent in-flight requests), reuse them, pipeline requests across streams

The interviewer then asked: "What happens when APNs returns a 410 for a device token?"

Candidate: "I'd log an error."

Interviewer: "And the next time you try to deliver to that user, what happens?"

Candidate: "I'd... try again?"

Correct answer: 410 Unregistered means the device token is permanently invalid (app uninstalled or device wiped). The worker must update the device_registry to mark that token INVALID, remove it from active delivery pool, and never attempt to use it again. Continuing to send to a 410 token wastes APNs quota and degrades your sender reputation score.

The interviewer then asked about the `apns-device-token` response header and what it means when APNs returns a new token value in this header even on a successful 200 response. The candidate had no idea this existed.

(APNs occasionally rotates tokens and returns the new token in the `apns-device-token` response header. If your worker doesn't read and store this new token, you'll eventually be sending to a stale token and receiving 410s.)

**What the candidate should have done:**

APNs is not a black box. Know:
1. HTTP/2 is required. Design a connection pool with persistent connections.
2. 1,000 concurrent streams per connection. For 3,500 iOS push/second, you need 3–4 persistent connections per worker pod (at typical ~100ms round-trip, each stream can sustain ~10 req/s, so 1000 streams × 10 req/s = 10,000/s per connection — but real-world latency variance means running 3–5 connections per pod is prudent).
3. 410 Unregistered = permanently dead token. Mark INVALID, never retry.
4. 429 TooManyRequests = per-device rate limit hit. Back off for this specific token; do not retry within the same campaign blast.
5. Read `apns-device-token` response header for token rotation.

Similarly for FCM:
- FCM maintains one persistent connection per Android device via Google Play Services, shared across all apps.
- `InvalidRegistration` = dead token. `canonicalRegistrationId` in response = new token. `MessageRateExceeded` = back off.

**Key lesson:** Push provider knowledge is a differentiator at companies that run push infrastructure (Apple, Meta, Uber). Know the connection model, the error taxonomy, and the token lifecycle. This is the difference between a candidate who has designed systems and one who has operated them.

---

## General Pattern Mistakes

### Mistake 1: Requirements in 60 seconds

Some candidates start drawing boxes before clarifying a single requirement. The interviewer at a senior level is watching whether you define the problem before solving it. You cannot design a notification system without knowing:
- Which channels? (Push alone is very different from push + email + SMS)
- Delivery guarantee? (at-least-once vs. exactly-once changes the entire queue design)
- Priority tiers? (Without this, you don't know you need separate queue partitions)
- Fan-out shape? (1-to-1 vs. 1-to-100M changes your worker sizing by orders of magnitude)

**Fix:** Spend 5 minutes on clarifying questions. Write down the answers visibly. Reference them during design ("based on our earlier agreement that transactional P99 is 5s, I'm partitioning the queue like this...").

### Mistake 2: Ignoring the fan-out math

"Send notification to all 10M users who liked a celebrity's post" is not "loop and call FCM." At 10,000 notifications/second, a 10M fan-out takes 1,000 seconds — 16.7 minutes. Is that acceptable for a time-sensitive notification? Do you batch at write time or read time? How do you handle users who follow the celebrity but have push disabled?

**Fix:** Do the napkin math. 100M users at 10K/s = 2.8 hours for a full blast. State this explicitly and discuss whether it's acceptable for marketing (yes) vs. transactional (no — transactional is 1-to-1 or 1-to-small-group, not broadcast).

### Mistake 3: Single queue, no priority separation

A single Kafka topic with a "priority" field is not priority separation — consumers process messages in order. A low-priority marketing campaign that generates 10M messages will delay high-priority transactional messages that arrive while the campaign is being processed.

**Fix:** Physically separate Kafka partitions (or separate topics) for TRANSACTIONAL vs. MARKETING, with separate consumer groups and separate worker pools. This is the Netflix RENO pattern.

### Mistake 4: No observability in the design

Completing a notification system design without mentioning metrics, alerting, or operational visibility is an automatic weakness flag for senior roles. The interviewer knows that systems fail in production, and they want to see that you understand the operational lifecycle, not just the happy path.

**Fix:** Add an observability section to your design: delivery success rate per channel (alert if <99%), DLQ depth (alert if >100), provider error rate by error code (alert on sustained FCM Unavailable), P99 fan-out latency per priority tier.

### Mistake 5: Confusing the template rendering location

Some candidates put a full template engine in the Notification API (synchronous, request path). This is wrong for two reasons:
1. Template rendering can be slow for complex templates (rich email HTML). Doing it synchronously on the POST path increases API latency.
2. Template rendering at fan-out time (in the worker) allows per-user personalization (localization, timezone-aware scheduling, personalized copy) using data available at delivery time, not at enqueue time.

**Fix:** Store `template_key + template_vars` in the notification record. Render at delivery time in the channel worker, using user locale/timezone from the user profile. This also means template changes take effect immediately without re-queuing old notifications.

---

## Self-Assessment Checklist

Run through this after every practice session. Be honest.

### Requirements Phase
- [ ] Asked about channels before drawing any boxes
- [ ] Clarified delivery semantics (at-least-once vs. exactly-once)
- [ ] Identified priority tiers
- [ ] Asked about user preferences and opt-out
- [ ] Asked about fan-out shape (1-to-1 vs. broadcast)
- [ ] Established scale targets (QPS, user count)

### Architecture Phase
- [ ] Decoupled delivery from producer's critical path (async queue)
- [ ] Priority separation at queue level (not just a field — physically separate partitions/queues)
- [ ] User preference layer checked before every delivery attempt
- [ ] Device registry with token rotation handling mentioned
- [ ] Channel abstraction layer (producer doesn't know APNs vs. FCM)

### Reliability Phase
- [ ] Retry state machine with explicit backoff schedule
- [ ] Non-retriable vs. retriable error classification
- [ ] Dead Letter Queue for permanently failed notifications
- [ ] DLQ depth alerting mentioned
- [ ] Deduplication via idempotency key

### Provider Knowledge
- [ ] APNs: HTTP/2, connection pools, 410 Unregistered handling, token rotation
- [ ] FCM: single persistent connection per device, InvalidRegistration handling, canonicalRegistrationId
- [ ] FCM quota: 600,000/minute — calculate if your design exceeds it
- [ ] APNs: 1,000 concurrent streams per HTTP/2 connection — size your connection pool

### Compliance
- [ ] GDPR right-to-erasure mentioned (purge notification history + suppress future sends)
- [ ] CAN-SPAM opt-out processing mentioned

### Observability
- [ ] Per-channel delivery success rate metric defined
- [ ] DLQ depth alert defined
- [ ] Provider error rate metric mentioned

---

## Remediation Targets

If you failed any section above, study these before your next attempt:

| Gap | Study Material |
|-----|----------------|
| APNs internals | Apple developer docs: "Sending notification requests to APNs", HTTP/2 provider API section |
| FCM internals | Firebase docs: "About FCM messages", connection model section |
| Kafka priority queues | Netflix RENO blog post (Feb 2022): "Ensuring prioritized notifications" |
| Fan-out at scale | Uber RAMEN engineering blog (2024): gRPC migration, 70K QPS, 600K concurrent connections |
| GDPR/CAN-SPAM | CAN-SPAM Act summary (FTC.gov); GDPR Article 17 (right to erasure) and Article 21 (right to object) |
| Retry patterns | AWS Architecture Blog: "Exponential backoff and jitter" |
| Exactly-once vs at-least-once | Martin Kleppmann "Designing Data-Intensive Applications" Chapter 11 |
