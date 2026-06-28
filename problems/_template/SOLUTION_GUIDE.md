# Solution Guide — Notification System

> **Do not read this until after you have completed your session and scored yourself on PROPOSED_SOLUTION.md.**

---

## Component Map

| Component | Type | Responsibility | Why This, Not the Alternative |
|-----------|------|----------------|-------------------------------|
| API Gateway | Managed gateway (Kong / AWS API GW) | TLS termination, auth, rate limiting on ingress | Separates cross-cutting concerns from the notification service logic; publisher teams get a stable interface even as internal components change |
| Notification Service | Stateless application service | Validates payload, generates idempotency key, writes to Kafka topic | Stateless design allows horizontal scaling; putting idempotency key generation here rather than at the publisher means publishers don't need to coordinate |
| Kafka (Notification Events topic) | Distributed log | Durable, ordered, replayable ingestion queue | Kafka's replication factor of 3 gives durability without a synchronous DB write on the hot path; replayability lets us reprocess events during incidents; chosen over SQS because ordering within a partition lets us deduplicate by user |
| Fan-out Service | Kafka consumer group | Reads notification events, resolves device tokens, writes per-device delivery records | The fan-out step is where the 1-to-N amplification happens; keeping it separate from ingestion means a slow device token lookup doesn't back-pressure the ingestion path |
| Device Token Store | Cassandra cluster | Stores userId → [deviceToken, platform, createdAt, lastSeenAt] | Cassandra's wide-row model maps perfectly to the query "give me all tokens for user X"; write throughput is high (token refreshes on every app launch), and Cassandra handles that without write hotspots |
| APNs / FCM / Web Push Workers | Consumer group per platform | Reads per-device delivery tasks, formats platform-specific payload, calls external API | Separate worker pools per platform allow independent scaling and rate limiting; APNs and FCM have very different retry semantics and authentication models |
| Delivery Status Store | PostgreSQL (sharded by notification ID) | Tracks per-device delivery state machine | Delivery status is queried by notification ID + device ID, a pattern that shards well; PostgreSQL chosen over Cassandra here because status updates involve conditional writes (only update to "delivered" if current state is "enqueued") |
| Dead Letter Queue | Kafka DLQ topic | Holds notifications that exhausted retry budget | Using Kafka as the DLQ instead of a separate system lets us replay failed messages through the same consumer infrastructure without special tooling |

---

## Architecture Diagram

```
Publishers (Feed Svc / Messaging Svc / etc.)
         │
         │  POST /v1/notifications
         ▼
┌─────────────────────┐
│    API Gateway      │  ← TLS, auth, rate limit
└─────────────────────┘
         │
         │  validated event + idempotency key
         ▼
┌─────────────────────┐
│  Notification Svc   │  ← stateless, horizontally scaled
└─────────────────────┘
         │
         │  produce to Kafka
         ▼
┌─────────────────────────────────────────┐
│     Kafka: notification-events topic    │  ← 3 replicas, partitioned by userId
└─────────────────────────────────────────┘
         │
         │  consume (fan-out consumer group)
         ▼
┌─────────────────────┐        ┌──────────────────────────┐
│   Fan-out Service   │───────▶│  Device Token Store      │
└─────────────────────┘        │  (Cassandra)             │
         │                     └──────────────────────────┘
         │  per-device delivery tasks (one per device token)
         ▼
┌──────────────────────────────────────────────────┐
│    Kafka: per-platform delivery topics           │
│    (notification-ios / notification-android /    │
│     notification-web)                            │
└──────────────────────────────────────────────────┘
    │              │              │
    ▼              ▼              ▼
┌───────┐    ┌──────────┐   ┌──────────┐
│ APNs  │    │  FCM     │   │  Web     │
│Worker │    │  Worker  │   │  Push    │
│ Pool  │    │  Pool    │   │  Worker  │
└───────┘    └──────────┘   └──────────┘
    │              │              │
    └──────────────┴──────────────┘
                   │
       ┌───────────┴────────────────┐
       │                            │
       ▼                            ▼
┌──────────────┐            ┌──────────────┐
│ Delivery     │            │  Dead Letter │
│ Status Store │            │  Queue       │
│ (PostgreSQL) │            │  (Kafka DLQ) │
└──────────────┘            └──────────────┘
```

Arrow legend: `→` synchronous HTTP/gRPC | `──▶` produce to Kafka | `▼` consume from Kafka

---

## Key Design Decisions

### Decision 1 — Fan-out happens at consumption time, not at publication time

**Decision:** The notification event published by a service contains only a logical target (user ID and notification type). The fan-out to individual device tokens happens inside the fan-out service, which runs as a Kafka consumer group.

**Why:** The publishing service should not need to know how many devices a user has, what platforms they use, or what the current state of their tokens is. Coupling publishers to device count creates a burst problem at the publisher: a post that gets 100,000 likes in a minute would require each publisher to fan out to potentially 2.5 × 100,000 = 250,000 device-level delivery requests. By moving fan-out to a consumer, we absorb that burst in the consumer group's lag rather than in the publisher's latency.

**Alternative considered:** Fan-out at publish time — the publisher service resolves tokens and writes directly to per-platform queues.

**Why rejected:** This couples publisher throughput to device token store read latency. It also means the publisher service needs APNs and FCM integration, which is a violation of single-responsibility. And it makes re-processing during incidents much harder because you cannot replay a Kafka topic to re-fan-out — you'd need to recontact the publisher.

---

### Decision 2 — Separate delivery worker pools per platform

**Decision:** APNs, FCM, and Web Push each have their own Kafka consumer group and worker pool.

**Why:** APNs uses HTTP/2 with certificate-based authentication and a specific JSON payload format. FCM uses OAuth 2.0 service account tokens that expire and need rotation. Web Push uses VAPID keys and a different content-encoding scheme. The retry semantics also differ: APNs returns synchronous success/failure; FCM has a batch response format; Web Push uses HTTP 410 to signal permanent subscription removal. Mixing these in a single worker means a bug in one platform's authentication renewal can cause delivery failures across all platforms.

**Alternative considered:** A single universal delivery worker that branches on platform type.

**Why rejected:** The branch-on-platform approach works at low scale but becomes hard to operate: you cannot independently scale APNs capacity vs FCM capacity, you cannot separately rate-limit each platform's outbound connection pool, and a deployment that changes APNs integration necessarily touches FCM code, increasing blast radius.

---

### Decision 3 — Idempotency key generated at the Notification Service, not by the publisher

**Decision:** The Notification Service generates a deterministic idempotency key by hashing (notificationType + userId + contentId + publisherServiceId + time-window-bucket). The key is stored in the delivery status record. Before enqueuing, the service checks for an existing record with the same key.

**Why:** Publishers are distributed services that may retry a request after a timeout. If each publisher generates its own idempotency key, we cannot prevent cross-service duplicates (e.g., the feed service and the notifications service both deciding to send a "you were mentioned" notification for the same event). Centralizing key generation at the notification service gives us a single deduplication boundary.

**Alternative considered:** Require publishers to include their own idempotency key in the request.

**Why rejected:** This is a contract burden on every publisher team and requires them to understand our deduplication semantics. It also doesn't prevent two different publisher services from independently deciding to notify about the same event.

---

## Capacity Math

Starting numbers from the problem statement:

```
DAU = 100,000,000
Notifications per user per day = 3
Total notifications per day = 300,000,000

Average write QPS = 300,000,000 / 86,400 ≈ 3,472/sec
Peak write QPS = 3,472 × 5 = 17,360/sec

Devices per user = 2.5
Total per-device delivery records per day = 300,000,000 × 2.5 = 750,000,000

Payload storage per day = 300,000,000 × 512 bytes = 153.6 GB/day
7-day payload retention = 153.6 × 7 = 1.07 TB

Status record per device-notification = 200 bytes
Status storage per day = 750,000,000 × 200 bytes = 150 GB/day
30-day status retention = 150 × 30 = 4.5 TB

Device token storage:
  Total active devices = 100,000,000 × 2.5 = 250,000,000
  Average token size = (0.70 × 32) + (0.25 × 150) + (0.05 × 100) bytes ≈ 65 bytes per token
  Total token storage = 250,000,000 × 65 bytes ≈ 16 GB (fits in Cassandra easily)
```

**Bottleneck analysis:** The delivery path (fan-out service → platform workers) handles 750M per-device delivery tasks per day = ~8,700 delivery tasks/sec on average, ~43,500/sec at peak. This is the bottleneck. The ingestion path at 17K writes/sec is well within Kafka's capacity. The storage tier at ~4.5 TB / 30 days is manageable with a single sharded PostgreSQL cluster or a managed database service.

---

## API Design

### POST /v1/notifications

Enqueue a notification for delivery. Called by publisher services.

```
POST /v1/notifications
Authorization: Bearer {service-account-token}
Content-Type: application/json

{
  "idempotency_key": "feed-svc:like:post_id=abc123:user_id=u789:bucket=2024060112",
  "recipient_user_id": "u789",
  "notification_type": "like",
  "priority": "normal",
  "ttl_seconds": 86400,
  "payload": {
    "title": "Alice liked your post",
    "body": "\"The one about distributed systems...\"",
    "deep_link": "myapp://post/abc123",
    "image_url": "https://cdn.example.com/avatars/alice.jpg"
  }
}

Response 202 Accepted:
{
  "notification_id": "notif_01HXK4BWPQ7MV",
  "status": "enqueued",
  "estimated_delivery_ms": 2000
}

Response 409 Conflict (idempotency key already seen):
{
  "notification_id": "notif_01HXK4BWoriginal",
  "status": "already_enqueued",
  "message": "Idempotency key already processed"
}
```

### GET /v1/notifications/{notification_id}/status

Check delivery status for a specific notification.

```
GET /v1/notifications/notif_01HXK4BWPQ7MV/status

Response 200 OK:
{
  "notification_id": "notif_01HXK4BWPQ7MV",
  "recipient_user_id": "u789",
  "created_at": "2024-06-01T12:34:56Z",
  "delivery_records": [
    {
      "device_token_id": "dt_iphone14_u789",
      "platform": "apns",
      "status": "delivered",
      "delivered_at": "2024-06-01T12:34:59Z",
      "attempts": 1
    },
    {
      "device_token_id": "dt_ipad_u789",
      "platform": "apns",
      "status": "failed",
      "last_attempt_at": "2024-06-01T12:35:10Z",
      "attempts": 3,
      "failure_reason": "token_invalid"
    }
  ]
}
```

### POST /v1/devices

Register or refresh a device token.

```
POST /v1/devices
Authorization: Bearer {user-session-token}
Content-Type: application/json

{
  "device_token": "abc123def456...",
  "platform": "apns",
  "app_version": "4.2.1",
  "os_version": "iOS 17.4"
}

Response 200 OK:
{
  "device_id": "dt_iphone14_u789",
  "registered_at": "2024-06-01T12:00:00Z"
}
```

### DELETE /v1/devices/{device_id}

Deregister a token on logout.

```
DELETE /v1/devices/dt_iphone14_u789
Authorization: Bearer {user-session-token}

Response 204 No Content
```

---

## Data Model

### Device Token Store (Cassandra)

```sql
-- Cassandra table: device_tokens
-- Partition key: user_id
-- Clustering key: device_token_id (for unique identification)

CREATE TABLE device_tokens (
    user_id         TEXT,
    device_token_id TEXT,       -- internal UUID
    platform        TEXT,       -- 'apns' | 'fcm' | 'web_push'
    token_value     TEXT,       -- the actual APNs/FCM token string
    app_version     TEXT,
    os_version      TEXT,
    is_active       BOOLEAN,
    registered_at   TIMESTAMP,
    last_seen_at    TIMESTAMP,
    PRIMARY KEY (user_id, device_token_id)
) WITH CLUSTERING ORDER BY (device_token_id ASC);

-- Secondary index for reverse lookup (token_value → user_id)
-- needed when APNs tells us a token is invalid
CREATE INDEX ON device_tokens (token_value);
```

**Why Cassandra:** The primary query is "give me all active tokens for user_id X" — a single partition read. Cassandra's wide-row model handles this with a single partition scan, regardless of how many devices a user has. The write pattern is high throughput and append-heavy (each app launch may refresh the last_seen_at timestamp). Cassandra handles that without write contention.

### Delivery Status Store (PostgreSQL)

```sql
CREATE TABLE notifications (
    notification_id     TEXT PRIMARY KEY,     -- ULID for time-sortable IDs
    recipient_user_id   TEXT NOT NULL,
    notification_type   TEXT NOT NULL,
    priority            TEXT NOT NULL DEFAULT 'normal',
    ttl_seconds         INTEGER NOT NULL,
    payload_json        JSONB NOT NULL,
    idempotency_key     TEXT NOT NULL UNIQUE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at          TIMESTAMPTZ NOT NULL
);

CREATE INDEX idx_notifications_recipient ON notifications (recipient_user_id, created_at DESC);
CREATE INDEX idx_notifications_idempotency ON notifications (idempotency_key);
CREATE INDEX idx_notifications_expires ON notifications (expires_at) WHERE expires_at > NOW();

CREATE TABLE delivery_records (
    delivery_id         TEXT PRIMARY KEY,
    notification_id     TEXT NOT NULL REFERENCES notifications(notification_id),
    device_token_id     TEXT NOT NULL,
    platform            TEXT NOT NULL,
    status              TEXT NOT NULL CHECK (status IN ('enqueued', 'delivered', 'failed', 'expired')),
    attempts            INTEGER NOT NULL DEFAULT 0,
    last_attempt_at     TIMESTAMPTZ,
    delivered_at        TIMESTAMPTZ,
    failure_reason      TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_delivery_notification ON delivery_records (notification_id);
CREATE INDEX idx_delivery_status_enqueued ON delivery_records (status, created_at)
    WHERE status = 'enqueued';   -- partial index for the retry worker scan
```

**Index rationale:** The partial index on `status = 'enqueued'` is the key optimization. The retry worker periodically scans for enqueued records that have been sitting too long without a delivery attempt — this query runs constantly and must be fast. The full-table scan without this index would degrade as delivery history accumulates.

---

## Deep Dive: Fan-Out at Scale

The fan-out problem sounds simple at first: read the notification event, find the user's device tokens, write a delivery record per token. At 100M users with 2.5 devices each, this is 750M delivery records per day — manageable with a well-partitioned database. The problem emerges when you consider the distribution of follower counts on a social platform.

Most users have tens or hundreds of followers. A small number of accounts — verified celebrities, major brands, viral content creators — have tens of millions of followers. When one of those high-follower accounts posts content that triggers a "you might like this" notification to all their followers, a single notification event generates 10 million fan-out tasks. At the fan-out service's average write rate of ~8,700 tasks/sec, processing 10M tasks for a single event takes nearly 19 minutes. During that 19 minutes, the fan-out consumer group's lag grows, creating a pipeline delay for all notifications — including high-priority DMs from ordinary users.

The standard solution is a two-tier fan-out strategy. For accounts below a follower threshold (say, 100,000 followers), use push fan-out: the fan-out service writes all delivery records synchronously as it processes the event. For accounts above the threshold, use a lazy fan-out: enqueue the celebrity notification event separately to a dedicated high-volume topic, and let a separate worker pool process it more slowly without impacting the main delivery pipeline. The critical design choice is that high-priority notifications (DMs, security alerts) are never subject to the celebrity throttle — they always go through the fast path.

A further optimization for the celebrity case is to not fan-out to every follower at all, but instead store the notification in a centralized "pending notification" store and have clients pull it on next app open. This is less real-time but dramatically reduces write amplification. The choice between push and pull depends on the latency SLA for that notification type — which is exactly why the per-type TTL and priority configuration exists.

---

## Failure Modes & Mitigations

| Failure | Impact | Detection | Mitigation | Residual Risk |
|---------|--------|-----------|------------|---------------|
| APNs returns 429 (rate limit) | Delivery delays for iOS users | Metrics: APNs 4xx error rate spike | Exponential backoff with jitter; per-account rate limiter on the APNs worker pool | Notifications may be delayed during sustained APNs rate limiting |
| Fan-out service consumer lag grows | All notification delivery delayed | Kafka consumer group lag metric; alert at 5-minute lag threshold | Auto-scale fan-out service pods; add partitions to Kafka topic | During scale-up event, lag continues growing for 2–3 minutes |
| Device token invalid (APNs 410) | Delivery silently fails for stale tokens | APNs HTTP 410 response | On 410 response, immediately mark token inactive in Device Token Store | Token store update is async — a small window where additional delivery attempts hit the same invalid token |
| PostgreSQL primary failover | Delivery status writes fail during failover (typically 10–30 sec) | Health check probe; replication lag alert | Delivery status writes are buffered in a local retry queue on workers; workers retry on reconnect | Status records written during the buffer window may be out of order |
| Kafka cluster unavailable | Ingestion path down; notifications silently dropped | Kafka broker health check; producer error rate alert | Notification Service returns 503 to publishers; publishers retry with backoff; producers accumulate in publisher-side buffer | If Kafka outage exceeds publisher retry budget, notifications are lost |

---

## Extension Points

**Priority queuing:** The current design treats all notifications equally in the Kafka topic. Adding priority requires either (a) separate Kafka topics per priority level with the fan-out service consuming the high-priority topic first, or (b) a single topic with a priority field and a consumer that processes high-priority messages first by maintaining two queues internally. Option (a) is operationally simpler; option (b) is more flexible. The data model already has a `priority` field, so no schema migration is required to add this.

**Notification batching:** When a user receives 200 "likes" on a post in a 10-minute window, batching them into "Alice and 199 others liked your post" requires a batching buffer layer between the fan-out service and the delivery workers. The buffer collects delivery tasks for the same (user, notification_type, content_id) tuple over a configurable window, then emits a single summarized delivery task. The tradeoff is delivery latency — the first notification in a batch window is held until the window expires.

**Scheduled notifications:** The ingestion API already supports a future `deliver_at` field (design omitted for brevity). Adding it requires a scheduled notification store (Redis Sorted Set keyed by `deliver_at` timestamp) and a dispatcher service that polls for notifications whose delivery time has arrived and enqueues them into the normal Kafka pipeline. Redis Sorted Set's `ZRANGEBYSCORE` makes the polling query O(log N + M) where M is the number of notifications due in the current poll window.

---

## What Strong Candidates Do Differently

Strong candidates state the fan-out problem before being asked. They identify that the 1-to-N amplification is the core engineering challenge of this system and explain how the architecture addresses it, rather than treating it as an obvious detail. This shows they've thought about the failure modes at scale rather than just describing components.

Strong candidates justify their database choices against access patterns. "I'll use Cassandra for the device token store because the primary query is a partition scan by user_id and Cassandra's wide-row model handles that natively" is a 3/3 answer. "I'll use a NoSQL database for tokens" is a 1/3 answer — it names a category without the reasoning.

Strong candidates design for APNs token invalidation. This is a real operational problem that trips up many candidates. When APNs returns a 410 for a device token, you have to remove that token from your store — otherwise every subsequent notification attempt for that user on that device will fail, burning retry budget. Strong candidates know this because they've thought about the lifecycle of a device token, not just its happy-path registration.

Strong candidates choose the right deep dive. The two hardest problems in this system are fan-out at scale and delivery reliability (retry + idempotency). A candidate who spends 15 minutes deep-diving the database schema for the notification payload has made a poor choice — that's not where the complexity is.

---

## What Average Candidates Miss

Average candidates define no TTL on individual notification types. They say "notifications expire after X days" without realizing that a password reset notification that arrives 48 hours late is not just useless — it's a security problem. And a weekly digest that expires after 24 hours is an ops problem. Per-type TTL is an interview signal.

Average candidates do not address idempotency at all. The retry problem is real: a publisher service that times out on an HTTP call to the notification service will retry. Without idempotency, the user gets two "Alice liked your post" notifications. Most candidates who miss this haven't thought about the network between the publisher and the notification service as a failure domain.

Average candidates describe fan-out but don't identify the celebrity problem. Every candidate knows that you read device tokens and write delivery records per device. Fewer realize that the uniform-fan-out model breaks for high-follower accounts and that you need a tiered strategy.

Average candidates propose retrying to APNs with a fixed interval. A fixed retry interval under APNs 429 load creates a thundering herd: every worker retries at the same time, APNs rate-limits again, and you've made the problem worse. Exponential backoff with jitter is the standard solution, and the reason for jitter is worth explaining.
