# Solution Guide — Chat System (WhatsApp / Messenger)

## Component Map
```
[Mobile Client] ←──WebSocket──→ [Chat Server A]
[Mobile Client] ←──WebSocket──→ [Chat Server B]
                                       ↓
                              [Message Router]
                              (user → server mapping)
                                       ↓
                    ┌──────────────────┼──────────────────┐
                    ↓                  ↓                  ↓
             [Message Store]   [Fan-Out Queue]    [Push Service]
             (Cassandra)       (Kafka, groups)    (APNs/FCM)
                    ↓                  ↓
             [History API]    [Group Delivery Workers]
```

## Architecture Diagram (ASCII)
```
1:1 ONLINE MESSAGE FLOW:
Alice (on Server A)                           Bob (on Server B)
    │                                               │
    │ "Hello" via WebSocket                         │
    ▼                                               │
[Chat Server A]                                     │
    │                                               │
    ├─ 1. Write to Cassandra (message_store)        │
    │     row: (conv_id, msg_id, "Hello", ts)       │
    │                                               │
    ├─ 2. Lookup Bob's server: Server B             │
    │     (via User Presence Service / Redis)       │
    │                                               │
    ├─ 3. Send to Server B via internal gRPC        │
    │                                         ┌─────┘
    │                                   [Chat Server B]
    │                                         │
    │                                         ├─ 4. Push to Bob's WebSocket
    │                                         │
    │◄────────── ack: delivered ──────────────┤
    │                                         │
    ▼                                         │
Alice sees ✓✓ (delivered)               Bob sees message
    │                                         │
    │                                   Bob opens message
    │◄────────── ack: read ───────────────────┤
Alice sees ✓✓ (blue = read)


1:1 OFFLINE MESSAGE FLOW:
[Chat Server A]
    ├─ 1. Write to Cassandra
    ├─ 2. Lookup Bob: OFFLINE (no server registered)
    └─ 3. Enqueue push notification via APNs/FCM
         → APNs/FCM delivers to Bob's device
         → Bob opens app → WebSocket established
         → App requests messages since last_ack
         → Server streams missed messages
         → App sends "delivered" ack


GROUP CHAT FAN-OUT (50 members):
[Chat Server A] receives group message from Alice
    ├─ 1. Write to Cassandra (group message)
    ├─ 2. Publish to Kafka topic: "group:{group_id}"
    │
[Fan-Out Workers] (consumer group)
    ├─ Read group member list from Group Service
    ├─ For each member:
    │    ├─ Online: send via their Chat Server
    │    └─ Offline: push notification via APNs/FCM
```

## Capacity Math

**Messages per second:**
- 500M DAU × 40 messages/day = 20B messages/day
- 20B / 86,400s = 231,481 messages/second
- Peak 3×: ~700,000 messages/second
- (Source: WhatsApp processes 100B+ messages/day as of 2022, ~1.16M/sec — this design is 1/100th that scale)

**WebSocket connections:**
- 50M concurrent users (10% of DAU active simultaneously)
- 50M WebSocket connections across Chat Servers
- Each Chat Server handles 10,000 connections (comfortable for Go/Erlang; Erlang handles 2M per node — WhatsApp's original architecture choice)
- Servers needed: 50M / 10,000 = 5,000 Chat Servers
- (Real: WhatsApp used ~1,000 Erlang servers for 2B users in 2014, each handling 2M connections — significantly more efficient)

**Storage:**
- 231,481 messages/sec × 100 bytes = 23 MB/sec = 2 TB/day
- 7-day retention: 14 TB
- With 3× replication (Cassandra): 42 TB
- Media storage: order of magnitude larger; use separate blob store (S3)

## API Design

### Send Message (via WebSocket)
```
// Client → Server (WebSocket JSON frame)
{
  "type": "send_message",
  "client_msg_id": "uuid-generated-by-client",  // for deduplication
  "conversation_id": "conv_abc123",
  "content": "Hello!",
  "content_type": "text",
  "timestamp_client": 1735000000000
}

// Server → Client (acknowledgment)
{
  "type": "ack",
  "client_msg_id": "uuid-generated-by-client",
  "server_msg_id": "msg_xyz789",
  "status": "sent"
}
```

### Delivery Receipt (Server → Sender via WebSocket)
```
{
  "type": "receipt",
  "msg_id": "msg_xyz789",
  "status": "delivered",  // or "read"
  "timestamp": 1735000001000
}
```

### Message History (REST, for initial load or scroll-back)
```
GET /v1/conversations/{conv_id}/messages?before={msg_id}&limit=50
Authorization: Bearer {token}

Response 200:
{
  "messages": [
    {
      "msg_id": "msg_xyz789",
      "sender_id": "user_alice",
      "content": "Hello!",
      "content_type": "text",
      "timestamp": 1735000000000,
      "status": "read"
    }
  ],
  "has_more": true
}
```

### WebSocket Connection
```
GET /v1/chat/connect
Upgrade: websocket
Authorization: Bearer {token}

// Server registers: user_id → server_id in User Presence Service
// Connection maintained with heartbeat every 30 seconds
```

## Data Model

### Message Store (Cassandra)
```
Table: messages
Partition key: conversation_id
Clustering key: message_id DESC (ULID — sortable by time)

┌─────────────────┬───────────────┬─────────────────────────────────────┐
│ conversation_id │ VARCHAR       │ Partition key — routes to shard     │
│ message_id      │ ULID          │ Clustering key — time-sorted within │
│ sender_id       │ VARCHAR       │ Who sent it                         │
│ content         │ TEXT          │ Message body (encrypted at rest)    │
│ content_type    │ ENUM          │ text / image / video / file         │
│ status          │ ENUM          │ sent / delivered / read             │
│ created_at      │ TIMESTAMP     │ Server-assigned timestamp           │
└─────────────────┴───────────────┴─────────────────────────────────────┘

Why Cassandra: Time-series append workload. Partition by conversation_id means
all messages in a conversation are co-located. Clustering by message_id gives
ordered reads without a sort. Write throughput of 100K+/sec per node.
TTL: 7 days (Cassandra TTL on rows; device-side storage is permanent)

Why ULID not UUID: ULIDs are time-sortable (48-bit timestamp prefix + 80-bit
random). This means message_id implicitly encodes time — no separate timestamp
needed for ordering. UUIDs are fully random and require a separate created_at
for ordering.
```

### User Presence (Redis)
```
Key:   "presence:{user_id}"
Value: JSON { "server_id": "chat-server-42", "last_seen": 1735000000 }
TTL:   60 seconds (heartbeat refreshes every 30s)

When TTL expires: user treated as offline → push notification path
```

### Group Membership (PostgreSQL)
```
Table: group_members
┌────────────┬───────────┬───────────┐
│ group_id   │ VARCHAR   │           │
│ user_id    │ VARCHAR   │           │
│ role       │ ENUM      │ admin/mem │
│ joined_at  │ TIMESTAMP │           │
└────────────┴───────────┴───────────┘
Index: (group_id) for fan-out member lookup
Why PostgreSQL: Group metadata is relational (membership, roles, settings).
Not time-series. Small number of writes (join/leave). Strong consistency wanted.
```

## Key Design Decisions

### Decision 1: WebSocket over HTTP Long Polling
**Choice made:** WebSocket persistent connection for message delivery.

**Alternative rejected:** HTTP long polling — client holds open an HTTP request; server responds when a message arrives, then client immediately opens another request.

**Why this:** WebSocket is a single persistent TCP connection with full-duplex communication. After the initial handshake, message delivery adds zero round-trip overhead. Long polling requires a new HTTP handshake for every message, adds ~50-150ms latency per message, and doubles the number of connections (one for polling + one for sending). At 50M concurrent users, long polling would require maintaining 2× the connection state.

**Trade-off accepted:** WebSocket connections require stateful servers — a user's connection is pinned to one Chat Server. This makes horizontal scaling more complex (must route messages to the correct server) and makes deploys more disruptive (rolling deploy disconnects users on that server). Mitigate with graceful shutdown: stop accepting new connections, drain existing ones over 30 seconds, then terminate.

---

### Decision 2: Write-to-Cassandra Before Fan-Out
**Choice made:** Persist the message to Cassandra atomically before attempting any delivery to recipients.

**Alternative rejected:** Deliver the message in memory (to online users) first, then persist asynchronously.

**Why this:** If we deliver in memory first and then Cassandra write fails, the recipient saw a message that doesn't exist in the database. On app restart or device switch, the message is gone. This violates the "zero message loss" guarantee. By writing to Cassandra first and only then routing to recipients, we ensure durability before any delivery attempt. This is the "write-ahead log" principle applied to messaging. WhatsApp's architecture explicitly documents this ordering (Facebook Engineering blog, 2020).

**Trade-off accepted:** Adds one Cassandra write to the hot path before the user sees the message delivered. Cassandra writes at SSD are ~1-2ms with quorum writes. This is acceptable within the 500ms latency budget.

---

### Decision 3: Push Fan-Out vs Pull-on-Receive for Group Chat
**Choice made:** Push fan-out via Kafka — when a group message arrives, fan it out to all members immediately via a queue.

**Alternative rejected:** Pull-on-receive — members request messages when they connect; no proactive fan-out.

**Why this:** Pull-on-receive works for small groups but breaks for active groups. Consider: Bob is in 20 active groups. Every time Bob connects, his client must poll all 20 groups for new messages — 20 reads per connection event. At 50M reconnections/day, this is 1 billion reads/day just for reconnect polling. With push fan-out, each message triggers one write per recipient (delivered to their server or queued for offline delivery). The fan-out is amortized over message send time, not concentrated at connection time.

**Trade-off accepted:** For large groups (500 members), fan-out at send time creates a write burst: 500 recipient lookups + 500 delivery operations per message. A popular group sending 100 messages/minute generates 50,000 operations/minute. This is the fundamental tension in large group chat. Mitigate by: (1) Kafka-based fan-out with parallel workers, (2) batching offline push notifications (don't ping every offline member immediately — batch for 5 seconds).

## Deep Dive: Message Ordering and the Client-Side Sequence Number Problem

The hardest problem in chat is not delivery — it's ordering. Consider:

- Alice sends message A and message B in quick succession.
- Message A is delayed in the network.
- Bob receives B before A.
- Bob's chat history shows B → A, which is wrong.

**The wrong solution:** Use server timestamps (created_at). Network latency makes this unreliable — two messages sent 50ms apart may arrive at the server 200ms apart, and server timestamps don't reflect the sender's intent.

**The correct solution (client-side sequence numbers):**

Each message includes:
1. `client_sequence_number`: monotonically increasing per sender per conversation
2. `server_msg_id` (ULID): assigned by the server, encodes server-receive time

The server stores both. The client renders messages ordered by `client_sequence_number` for messages from the same sender, and by `server_msg_id` (time-sortable ULID) across senders.

**Edge case: concurrent sends from multiple devices**
Alice is logged in on her phone and laptop. She sends from both simultaneously. Each device has its own sequence counter. Solution: sequence numbers are per-device, not per-user. The receiving client sorts cross-device messages by `server_msg_id`.

**Group chat ordering:**
For group chats, there is no total ordering that satisfies all members (CAP theorem for distributed systems). WhatsApp uses logical ordering: the server assigns a `server_sequence_number` per group conversation. All members see the same server-assigned order, even if the "natural" order was different from members' perspectives. This is a design choice — causal consistency, not causal real-order.

## Failure Modes & Mitigations

| Component | Failure | Mitigation | Trade-off |
|-----------|---------|------------|-----------|
| Chat Server crash | All connections on that server drop | Clients auto-reconnect; User Presence TTL expires; clients request missed messages via REST history API | Brief message gap; clients must implement reconnect logic |
| Cassandra write failure | Message not persisted | Sender receives error ack; client retries with same client_msg_id (idempotent); server deduplicates | User sees retry UI briefly; acceptable |
| User Presence Redis miss | Cannot find Bob's server | Fall back to offline path (push notification); false offline for < 60s | Bob gets a push notification even if online; minor UX issue |
| Kafka fan-out lag | Group message delayed | Recipients experience delay; message still delivered; monitor Kafka consumer lag | Real-time guarantee degrades to near-real-time for groups |
| APNs/FCM unavailable | Offline users not notified | Message still in Cassandra; user retrieves on next app open; no data loss | User doesn't know a message arrived until they open the app |
| Message deduplication failure | Duplicate messages appear | client_msg_id is a unique key in Cassandra (upsert semantics); duplicates collapsed | Requires client to generate truly unique IDs (UUID/ULID) |

## What Strong Candidates Do Differently
1. **Propose WebSocket from the start with a specific justification** — they say "WebSocket because we need sub-500ms latency and full-duplex communication; long polling would add 50-150ms per message and double the connection overhead."
2. **Distinguish 1:1 from group chat architectures** — they recognize these have fundamentally different fan-out costs and propose different delivery paths for each.
3. **Address message durability explicitly** — they say "write to Cassandra before any delivery attempt" and explain why.
4. **Tackle multi-device** — they recognize a user logged into phone + laptop needs message sync and propose a mechanism.
5. **Explain delivery receipts as a separate acknowledgment flow** — not a field update in the message table, but a separate event that flows back through the system.

## What Average Candidates Miss
- **Stateful vs stateless**: Candidates propose "stateless chat servers behind a load balancer" without realizing that WebSocket connections are inherently stateful — a message for Bob must reach the specific server where Bob's WebSocket is connected.
- **The routing problem**: When Alice's server needs to deliver to Bob, it doesn't know which of the 5,000 Chat Servers Bob is connected to. Missing the User Presence Service means missing the core routing mechanism.
- **Offline users**: Many candidates design only for the online case. "What if Bob is offline?" is often met with silence.
- **Group fan-out cost**: Candidates propose delivering group messages the same way as 1:1 messages without considering 500× the delivery operations. This design breaks at group scale.
- **Message ordering**: Missing client-side sequence numbers and explaining how out-of-order delivery is handled.
