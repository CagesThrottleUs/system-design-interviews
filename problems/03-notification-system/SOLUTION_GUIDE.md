# Solution Guide — Notification System at Scale

> This guide is for study and post-interview debrief. Do not read before your first attempt.

---

## Component Map

| Component | Role | Tech Options |
|-----------|------|--------------|
| Notification API | Accepts producer requests, validates, enqueues | REST/gRPC service |
| User Preference Store | Per-user opt-in/opt-out, DND hours | PostgreSQL + Redis cache |
| Device Registry | user_id → [device tokens, platform] | Cassandra or DynamoDB |
| Message Queue | Async fan-out, priority separation | Kafka (preferred) or AWS SQS |
| Fan-out Worker | Reads queue, applies preferences, routes to channel queues | Kafka consumer group |
| Channel Workers | PushWorker, EmailWorker, SMSWorker | Horizontally scaled pods |
| Provider Adapters | APNs, FCM, SES/SendGrid, Twilio | Thin HTTP/gRPC wrappers |
| Delivery Log | Per-notification per-channel status | Cassandra (write-heavy, time-series) |
| Dead Letter Queue | Undeliverable after retries | Kafka DLQ topic or SQS DLQ |
| Template Engine | Renders body from key + variables | In-process (Mustache/Jinja) |
| Token Rotation Handler | Processes provider feedback, removes stale tokens | Async consumer of provider callbacks |

---

## ASCII Architecture Diagram

```
  Producer Services
  (order-svc, feed-svc,
   marketing-svc, payment-svc)
         |
         | POST /notifications  (async enqueue)
         v
  +--------------------+
  |  Notification API  |  <-- validates schema, deduplication check,
  |  (REST/gRPC)       |      writes notification record (status=QUEUED)
  +--------------------+
         |
         | publish to topic: notifications.raw
         v
  +============================+
  ||   Kafka: notifications   ||   <-- priority partitioned:
  ||   .raw (input topic)     ||       partition 0-3  = TRANSACTIONAL
  +============================+       partition 4-7  = MARKETING
         |
         v
  +--------------------+
  |   Fan-out Worker   |  <-- reads user preferences from Redis cache
  |   (consumer group) |      checks DND window
  |                    |      resolves device tokens from registry
  +--------------------+
         |
    _____|_______________________________________
   |             |                |             |
   v             v                v             v
[Kafka]       [Kafka]         [Kafka]        [Kafka]
chan.push    chan.email       chan.sms      chan.inapp
   |             |                |
   v             v                v
+--------+  +----------+  +----------+
| Push   |  | Email    |  | SMS      |
| Worker |  | Worker   |  | Worker   |
+--------+  +----------+  +----------+
   |    \        |              |
   |     \       v              v
   |  [APNs]  [SES /        [Twilio /
   |           SendGrid]     SNS]
   v
 [FCM]

Each Channel Worker:
  - Retry loop with exponential backoff (max 5 attempts)
  - On success: update delivery_log status=SENT
  - On hard failure: publish to DLQ topic
  - Reads APNs/FCM feedback → Token Rotation Handler

  +---------------------+
  |  Delivery Log       |  <-- Cassandra, partition key = notification_id
  |  (Cassandra)        |      status updates from all workers
  +---------------------+

  +---------------------+
  |  Dead Letter Queue  |  <-- Ops dashboard reads DLQ depth
  |  (Kafka DLQ topic)  |      alert if depth > threshold
  +---------------------+
         |
         v
  [Operator Console: replay or discard]
```

---

## Full Capacity Math

### Baseline throughput

```
10M DAU × 3 notifications/day = 30M notifications/day
30M / 86,400 seconds           = 347/second baseline
Peak (10× for flash sales)     = 3,470/second
Design headroom                = 10,000/second
```

### Channel breakdown at peak (10,000/s)

```
Mobile push  (70%)  = 7,000 push/second
Email        (20%)  = 2,000 email/second
SMS          (10%)  = 1,000 SMS/second
```

### Provider rate limit check

```
FCM default quota: 600,000 messages/minute = 10,000/second
→ 7,000 push/second (Android portion ~50%) = 3,500 Android/second → within FCM quota for single project
→ Heavy campaigns require multiple FCM project IDs or quota increase request

APNs: up to 1,000 concurrent HTTP/2 streams per connection
→ Open 10+ parallel connections per APNs pod to sustain 3,500 iOS push/second
→ Each stream can pipeline requests; APNs returns 429 TooManyRequests per-device if burst is too high

Twilio SMS: ~100 SMS/second per long code, 1,000/second on toll-free number
→ 1,000 SMS/second at peak requires pool of numbers or short code
```

### Storage

```
Notification record: ~1 KB × 30M/day × 30 day retention = 900 GB
Device registry:     ~200 bytes × 100M users × 1.5 tokens = 30 GB
User preferences:    ~500 bytes × 100M users               = 50 GB
Delivery log:        ~500 bytes × 30M events/day × 30 days = 450 GB
Total storage:       ~1.4 TB (fits in Cassandra cluster, no sharding panic)
```

### Queue sizing

```
Kafka throughput needed: 10,000 messages/second × 1 KB = 10 MB/second
→ Well within Kafka capacity (Kafka can sustain 100s of MB/s per broker)
→ 6-partition notification.raw topic; 3 brokers; replication factor 3
→ Retention: 24 hours for replay capability
```

---

## API Design

### Producer-facing: Enqueue a notification

```
POST /v1/notifications
Content-Type: application/json

{
  "idempotency_key": "order-svc:order:8827:shipped",   // dedup key
  "user_id":         "usr_abc123",
  "priority":        "TRANSACTIONAL",                   // or MARKETING
  "channels":        ["PUSH", "EMAIL"],                 // requested channels; preferences applied on top
  "template_key":    "order.shipped.v2",
  "template_vars":   {
    "order_id":        "ORD-8827",
    "estimated_date":  "July 4"
  },
  "ttl_seconds":     86400                              // expire if not delivered within 24h
}

Response 202 Accepted:
{
  "notification_id": "ntf_xyz789",
  "status":          "QUEUED"
}
```

### Status check (producer polling / webhook)

```
GET /v1/notifications/{notification_id}/status

Response 200:
{
  "notification_id": "ntf_xyz789",
  "user_id":         "usr_abc123",
  "deliveries": [
    { "channel": "PUSH",  "status": "DELIVERED", "delivered_at": "2026-06-28T10:05:02Z" },
    { "channel": "EMAIL", "status": "SENT",       "sent_at":      "2026-06-28T10:05:01Z" }
  ]
}
```

### Preference management (user-facing, via user-service)

```
PUT /v1/users/{user_id}/notification-preferences
{
  "channels": {
    "PUSH":  { "marketing": false, "transactional": true },
    "EMAIL": { "marketing": true,  "transactional": true },
    "SMS":   { "marketing": false, "transactional": false }
  },
  "dnd": {
    "enabled": true,
    "start_hour": 22,    // 10pm local time
    "end_hour":   8,     // 8am local time
    "timezone":   "America/Los_Angeles"
  }
}
```

---

## Data Models

### notifications table (PostgreSQL or Cassandra)

```sql
CREATE TABLE notifications (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  idempotency_key  TEXT UNIQUE NOT NULL,      -- dedup; index on this
  user_id          UUID NOT NULL,
  priority         ENUM('TRANSACTIONAL', 'MARKETING') NOT NULL,
  template_key     TEXT NOT NULL,
  template_vars    JSONB,
  requested_channels TEXT[],                  -- ['PUSH', 'EMAIL']
  status           ENUM('QUEUED', 'IN_FLIGHT', 'PARTIAL', 'DONE', 'FAILED'),
  ttl_seconds      INT NOT NULL DEFAULT 86400,
  created_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  expires_at       TIMESTAMPTZ GENERATED ALWAYS AS (created_at + (ttl_seconds * interval '1 second')) STORED
);
```

### device_registry table (Cassandra — high write rate on token refresh)

```
device_registry
  user_id        UUID        (partition key)
  device_id      UUID        (clustering key)
  platform       TEXT        -- 'IOS' | 'ANDROID'
  token          TEXT        -- APNs device token or FCM registration token
  token_status   TEXT        -- 'ACTIVE' | 'STALE' | 'INVALID'
  app_version    TEXT
  registered_at  TIMESTAMP
  last_seen_at   TIMESTAMP
  updated_at     TIMESTAMP
```

### user_preferences table (PostgreSQL + Redis TTL cache)

```sql
CREATE TABLE user_notification_preferences (
  user_id          UUID NOT NULL,
  channel          TEXT NOT NULL,          -- 'PUSH' | 'EMAIL' | 'SMS'
  notification_type TEXT NOT NULL,         -- 'TRANSACTIONAL' | 'MARKETING' | '*'
  opted_in         BOOLEAN NOT NULL DEFAULT TRUE,
  updated_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  PRIMARY KEY (user_id, channel, notification_type)
);

CREATE TABLE user_dnd_settings (
  user_id     UUID PRIMARY KEY,
  enabled     BOOLEAN NOT NULL DEFAULT FALSE,
  start_hour  INT,        -- 0-23, local time
  end_hour    INT,        -- 0-23, local time
  timezone    TEXT
);
```

### delivery_log table (Cassandra — time-series, append-only)

```
delivery_log
  notification_id  UUID        (partition key)
  channel          TEXT        (clustering key part 1)
  attempt_number   INT         (clustering key part 2)
  status           TEXT        -- 'QUEUED' | 'SENT' | 'DELIVERED' | 'FAILED' | 'BOUNCED'
  provider         TEXT        -- 'APNS' | 'FCM' | 'SES' | 'TWILIO'
  provider_msg_id  TEXT        -- provider-assigned message ID for correlation
  error_code       TEXT        -- FCM: 'MessageRateExceeded', APNs: 'TooManyRequests', etc.
  attempted_at     TIMESTAMP
  completed_at     TIMESTAMP
```

---

## Key Design Decisions

### Decision 1: Queue-based fan-out vs. synchronous delivery

**Rejected:** Synchronous delivery (order-service directly calls APNs).
- Why rejected: A 200ms APNs call inside a checkout flow adds latency to a latency-sensitive path. APNs outages (they happen: ~3–4 times/year) would cause checkout failures. Black Friday 2023 scenario: APNs 503 → order-service 500 cascade.

**Chosen:** Queue-based fan-out via Kafka.
- Producer services publish a notification event and immediately return. Kafka absorbs the burst. Workers pull at a sustainable rate. Provider failures are isolated behind the queue — the order-service never sees them.
- Tradeoff accepted: notifications are not instantaneous (P99 ≤ 5s for transactional). This is acceptable for all known use cases on Kora.

### Decision 2: At-least-once vs. exactly-once delivery

**Rejected:** Exactly-once.
- Exactly-once end-to-end (from producer to APNs) is not achievable — APNs and FCM do not support transactional acknowledgment. The best we can achieve is exactly-once delivery *from our system to the provider's API*; what the provider does internally is opaque.

**Chosen:** At-least-once with idempotency key deduplication.
- Producer provides an `idempotency_key` (e.g., `order-svc:order:8827:shipped`). Notification API checks this key on receipt — duplicate returns 202 with the existing `notification_id`, no second enqueue. Workers are idempotent: if a worker crashes after calling APNs but before committing to the delivery log, re-processing the Kafka offset sends a duplicate push. This is acceptable — receiving "your order shipped" twice is far better than never receiving it.
- The idempotency_key unique index on the notifications table is the dedup fence.

### Decision 3: Channel abstraction layer

**Why it matters:** Without an abstraction, each producer service would need to know the difference between APNs HTTP/2 connection pooling and FCM's legacy XMPP vs. HTTP API. Every provider change requires touching every producer.

**Design:** Fan-out worker applies user preferences and resolves the *logical* channel (PUSH, EMAIL, SMS) to *physical* channel queues (chan.push.ios, chan.push.android, chan.email, chan.sms). Channel workers own the provider-specific implementation. Producers never know which provider was used.

### Decision 4: Priority tiers — separate Kafka partitions, separate consumer groups

**Problem:** A marketing blast to 10M users should not delay a "payment declined" transactional notification.

**Design:** 
- `notifications.raw` topic uses priority as a partition routing key: partition 0–3 = TRANSACTIONAL, partition 4–7 = MARKETING.
- Two separate consumer groups: `fanout-transactional` (8 workers, higher CPU allocation) and `fanout-marketing` (4 workers, lower CPU allocation, rate-throttled to 1,000/s).
- Same pattern at the channel level: `chan.push.transactional` vs. `chan.push.marketing` — separate queues, separate worker pools.

This is exactly what Netflix did with RENO (2022): priority-segmented SQS queues feeding priority-specific compute clusters. A "file upload complete" notification cannot delay a "account suspended" notification.

---

## Deep Dive: Delivery Reliability

### Retry Logic

Every channel worker implements the same retry state machine:

```
attempt 1: immediate
attempt 2: 5 seconds
attempt 3: 30 seconds
attempt 4: 2 minutes
attempt 5: 10 minutes
→ move to DLQ
```

Backoff is exponential with jitter: `sleep = base * (2^attempt) + random(0, base)`. Jitter prevents thundering herd when a provider recovers from an outage and all workers retry simultaneously.

### Provider Error Classification

Not all errors are retriable. Workers must classify:

| Error | Retriable? | Action |
|-------|-----------|--------|
| APNs 429 TooManyRequests (per-device) | No (for this notification) | Drop or reduce per-device rate |
| APNs 410 Unregistered (device token invalid) | No | Mark token INVALID in device_registry; try email fallback |
| FCM MessageRateExceeded | Yes (backoff) | Retry with longer backoff |
| FCM InvalidRegistration | No | Mark token INVALID in device_registry |
| FCM Unavailable (server error) | Yes | Retry |
| SES 4xx (bad recipient) | No | Mark email as bounced; remove from active list |
| SES 5xx (service error) | Yes | Retry |
| Twilio 429 (rate limit) | Yes (backoff) | Retry |
| Twilio 21211 (invalid number) | No | Mark phone as invalid |
| Network timeout | Yes | Retry |

### Dead Letter Queue

After 5 failed attempts, a delivery attempt is written to the DLQ Kafka topic with full context:
- notification_id, channel, all attempt records, last error code
- Ops dashboard shows DLQ depth per channel
- Alert fires if DLQ depth > 100 messages (signal of sustained provider outage)
- Ops can replay individual notifications or entire DLQ batches after provider recovery

### Deduplication at Push Layer

APNs and FCM have their own deduplication mechanisms:
- APNs: `apns-collapse-id` header — same collapse-id replaces an undelivered notification on the device. Useful for "you have 3 new messages" where only the latest matters.
- FCM: `collapse_key` — same semantics as APNs collapse-id.

Workers set these headers for notification types where only the latest matters (badge updates, unread counts). They do NOT set them for transactional notifications where every delivery is distinct.

### Token Rotation — The Hidden Complexity

This is where most candidates fail. Device tokens are not static:

**APNs:** Token changes when:
- User backs up and restores to a new device
- App is reinstalled
- iOS generates a new token after ~1 year (APNs Development Environment rotates more frequently)

APNs returns `410 Unregistered` when a token is stale. The worker must read the `apns-device-token` response header on successful delivers — APNs may return a new token even on success. Worker writes the new token to device_registry immediately.

**FCM:** FCM returns `registration_ids_canonicalized` in batch responses listing old → new token mappings. Worker processes these mappings and updates device_registry. FCM also returns `InvalidRegistration` for dead tokens — worker marks them INVALID.

**If you don't handle token rotation:** Within weeks, device_registry fills with dead tokens. Delivery rates decay silently. You fire millions of notifications to tokens that APNs returns 410 for, consuming quota and producing no deliveries. This is a silent, insidious failure mode.

---

## Failure Modes

| Component Fails | Impact | Mitigation |
|-----------------|--------|------------|
| Notification API pod crashes | Producer POST fails | Multiple pods behind load balancer; producer retries on 5xx |
| Kafka broker(s) fail | Fan-out stalls | Replication factor 3; Kafka persists to disk; lag accumulates but no data loss |
| Fan-out worker crashes mid-batch | Kafka offset not committed; messages reprocessed | At-least-once semantics; idempotency key prevents duplicate notifications to user |
| APNs outage | Push notifications queue up | Channel workers retry; DLQ if sustained; transactional falls back to email if configured |
| FCM outage | Android push fails | Same as APNs; consider SMS fallback for high-priority |
| Redis (preference cache) unavailable | Workers read from PostgreSQL directly | Cache miss path exists in all workers; slower but functional |
| Cassandra (delivery log) unavailable | Delivery status untracked | Notification delivery proceeds; log writes queued in local buffer (best-effort) |
| DLQ at capacity | Messages dropped after DLQ overflow | Alert on DLQ depth; auto-paging ops; DLQ is Kafka with high retention, very hard to overflow |

---

## What Strong Candidates Do Differently

1. **Name FCM's connection model.** FCM maintains a single persistent connection per Android device, shared across all apps via Google Play Services. This is why FCM can push to apps that aren't running. Candidates who know this signal real depth.

2. **Mention device token rotation proactively.** Most candidates treat APNs/FCM tokens as stable UUIDs. They are not. Candidates who describe the rotation cycle and the worker logic to handle `410 Unregistered` stand out at Apple and Uber rounds.

3. **Separate priority at every layer.** Weak designs have a single queue and a "priority" field. Strong designs have physically separate Kafka partitions, separate consumer groups, and separate channel worker pools — so a marketing blast never starves a transactional notification.

4. **Call out the fan-out problem for large user segments.** Sending a notification to all 100M users isn't "loop and send." It requires: worker pool sizing, per-provider rate limiting (FCM 600K/min quota), exponential backoff on MessageRateExceeded, estimated fan-out time (100M / 10K/s ≈ 2.8 hours for a full blast — is that acceptable for a marketing campaign?).

5. **Address GDPR/CAN-SPAM unprompted.** Opt-out layer is not just a UX nicety — it is a legal requirement. CAN-SPAM requires honor within 10 business days. GDPR right-to-erasure requires purging notification history. Candidates who name this without being prompted signal awareness of real-world constraints.

6. **Design observability.** Delivery success rate per channel, DLQ depth, provider error rate by error code — these must be alertable. A design without observability is incomplete for senior roles.

---

## What Average Candidates Miss

- Jumping to architecture in 60 seconds without clarifying channels, delivery semantics, or scale targets.
- Single-queue design — no priority separation. Marketing blasts delay transactional messages.
- No user opt-out / preference layer. Interviewer will escalate to GDPR compliance.
- Treating APNs/FCM as simple HTTP REST calls. Not knowing HTTP/2 requirement for APNs, not knowing FCM's connection model.
- No retry / exponential backoff. "Fire and forget" is not a delivery guarantee.
- No DLQ. Permanently failed notifications vanish silently.
- No deduplication. Webhook retries cause duplicate notifications.
- No token rotation handling. Delivery rates decay silently over weeks.
- No observability mentions. No way to know the system is failing until users complain.
- Over-engineering the first pass — jumping to exactly-once Kafka transactions when at-least-once + idempotency key is sufficient and simpler.
