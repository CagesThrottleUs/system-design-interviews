# System Design Curriculum — Master Overview

## The 3-Layer Hierarchy

Every system design question decomposes into three layers:

```
Layer 3: Question Archetypes
  ↓ built from
Layer 2: Patterns
  ↓ built from
Layer 1: Foundations (Base Concepts & Technologies)
```

You cannot reliably design any system without knowing the foundations.
You cannot reason about tradeoffs without knowing the patterns.
The archetypes are where you practice applying both.

---

## Layer 1 — Foundations

These are the concepts every interview assumes you already know.
If you don't know these, your HLD will have fundamental errors (e.g., device-as-server).

| # | Topic | Status | Key Concepts |
|---|-------|--------|--------------|
| L1-01 | Networking: TCP, UDP, HTTP, WebSockets | ✅ | persistent connection, port, handshake, HTTP/1 vs HTTP/2, polling vs push |
| L1-02 | Push Delivery: APNs & FCM | 🔄 | device token, persistent OS-level connection, store-and-forward, port 5223 |
| L1-03 | DNS & CDN | ☐ | TTL, edge caching, origin server, cache invalidation, anycast |
| L1-04 | Load Balancing | ☐ | L4 vs L7, round-robin, least-connections, consistent hashing, health checks |
| L1-05 | Databases — Relational | ☐ | ACID, B-tree index, primary-replica replication, read replica, sharding |
| L1-06 | Databases — NoSQL | ☐ | BASE, document/columnar/KV/graph types, Cassandra, DynamoDB, when to use |
| L1-07 | Caching | ☐ | write-through, write-back, cache-aside, LRU/LFU, Redis, cache stampede |
| L1-08 | Message Queues & Streaming | ☐ | Kafka (topics, partitions, consumer groups), SQS, at-least/exactly-once, offset |
| L1-09 | Storage: Blob & Object | ☐ | S3, block vs file vs object, presigned URLs, multipart upload |
| L1-10 | Distributed Systems Fundamentals | ☐ | CAP theorem, eventual consistency, strong consistency, vector clocks, Raft basics |
| L1-11 | API Design | ☐ | REST vs gRPC vs GraphQL, rate limiting, idempotency keys, versioning |
| L1-12 | Security Fundamentals | ☐ | TLS, JWT, OAuth2, API keys vs tokens, secret management |
| L1-13 | Observability & Monitoring | ☐ | metrics/traces/logs (three pillars), Prometheus, distributed tracing, alerting, SLO/SLI/SLA |
| L1-14 | AI/ML Infrastructure Basics | ☐ | feature stores, vector DBs, embeddings, model serving, inference latency, cold start |

---

## Layer 2 — Patterns

Design patterns that combine foundation concepts to solve classes of problems.

| # | Topic | Status | Key Concepts |
|---|-------|--------|--------------|
| L2-01 | Fan-out Strategies | ☐ | fan-out on write, fan-out on read, hybrid; when each scales |
| L2-02 | Event-Driven Architecture | ☐ | pub/sub, event sourcing, event bus, choreography vs orchestration |
| L2-03 | CQRS | ☐ | command/query separation, separate read/write models, eventual consistency |
| L2-04 | Saga Pattern | ☐ | distributed transactions without 2PC, compensating transactions |
| L2-05 | Sharding Strategies | ☐ | range, hash, directory-based; hotspots, resharding |
| L2-06 | Replication Patterns | ☐ | primary-replica, multi-primary, leaderless (Dynamo-style) |
| L2-07 | Rate Limiting Algorithms | ☐ | token bucket, leaky bucket, sliding window log, sliding window counter |
| L2-08 | Reliability Patterns | ☐ | circuit breaker, bulkhead, retry with jitter, timeout, graceful degradation |
| L2-09 | Notification Pipeline Pattern | ☐ | priority queue separation, per-tier workers, dead letter queues, store-and-forward |
| L2-10 | Search & Indexing Patterns | ☐ | inverted index, write-ahead log, elasticsearch architecture |
| L2-11 | Outbox Pattern | ☐ | transactional outbox, inbox pattern, exactly-once event publishing, dual-write problem |
| L2-12 | Backpressure & Flow Control | ☐ | consumer lag, flow control, shed load, queue depth monitoring, producer throttling |
| L2-13 | Data Pipeline Patterns | ☐ | Lambda architecture (batch + speed layer), Kappa architecture (streaming only), reconciliation |

---

## Layer 3 — Question Archetypes

The 12 most common system design question categories. Each session is a mock interview.

| # | Question | Foundations Used | Patterns Used | Status |
|---|----------|-----------------|---------------|--------|
| L3-01 | Notification System | L1-01,02,07,08 | L2-09, L2-08 | 🔄 In Progress |
| L3-02 | URL Shortener | L1-05, L1-04 | L2-05 | ☐ |
| L3-03 | Social News Feed | L1-07, L1-08 | L2-01, L2-03 | ☐ |
| L3-04 | Rate Limiter | L1-11 | L2-07 | ☐ |
| L3-05 | Web Crawler | L1-01, L1-08 | L2-02 | ☐ |
| L3-06 | Search Autocomplete | L1-05, L1-07 | L2-10 | ☐ |
| L3-07 | Distributed Cache (design Redis) | L1-07, L1-10 | L2-06, L2-05 | ☐ |
| L3-08 | File/Object Storage (Dropbox/S3) | L1-09, L1-10 | L2-06 | ☐ |
| L3-09 | Chat System (WhatsApp/Slack) | L1-01, L1-08 | L2-02, L2-09 | ☐ |
| L3-10 | Video Streaming (YouTube/Netflix) | L1-03, L1-09 | L2-06 | ☐ |
| L3-11 | Ride Sharing (Uber) | L1-01, L1-10 | L2-02, L2-08 | ☐ |
| L3-12 | Payment System | L1-10, L1-12 | L2-04, L2-07 | ☐ |
| L3-13 | Ticketing/Booking System (Ticketmaster) | L1-05, L1-07, L1-10 | L2-08, L2-11 | ☐ |
| L3-14 | Recommendation System (Netflix/Spotify) | L1-06, L1-07, L1-14 | L2-03, L2-13 | ☐ |
| L3-15 | Distributed Task Scheduler (Airflow) | L1-08, L1-10 | L2-02, L2-08 | ☐ |
| L3-16 | Location-Based Service (Yelp/Google Places) | L1-05, L1-06, L1-07 | L2-05, L2-10 | ☐ |
| L3-17 | Leaderboard System | L1-06, L1-07 | L2-01, L2-12 | ☐ |
| L3-18 | Collaborative Editing (Google Docs) | L1-01, L1-10 | L2-02, L2-06 | ☐ |
| L3-19 | Live Streaming Platform (Twitch) | L1-01, L1-03, L1-09 | L2-06, L2-12 | ☐ |

---

## Status Legend
- ☐ Not started
- 🔄 In progress
- ✅ Complete
- 🔁 Needs review

---

## Recommended Learning Order

1. L1-01 (Networking) → L1-02 (APNs/FCM) — fixes session_001 gap, foundational for everything
2. L1-08 (Message Queues) — needed for almost every archetype
3. L1-05 + L1-06 (Databases) — required for every LLD
4. L1-07 (Caching) — comes up in every single design
5. L1-10 (Distributed Systems) — enables tradeoff reasoning
6. L1-03 (DNS & CDN) → L1-04 (Load Balancing) — infrastructure fundamentals
7. L1-09 (Object Storage) — needed for file/video archetypes
8. L1-11 (API Design) → L1-12 (Security) — polish and depth
9. L1-13 (Observability) — critical in 2025; interviewers probe "how do you know it's working?"
10. L1-14 (AI/ML Infrastructure) — baseline expectation for senior roles in 2025
11. Layer 2 Patterns (L2-01 through L2-13) — cross-cutting design approaches
12. Layer 3 Archetypes (L3-01 through L3-19) — mock interviews applying all of the above

---

## When to Schedule a Mock Interview

Mock interviews are gated by which foundations and patterns you've covered. Attempting an archetype before its prerequisites are done produces bad signal — you'll get stuck on mechanism questions, not design questions.

### Gate 1 — Notification System Retry
**Unlock after:** L1-01, L1-02, L1-08
**Question:** L3-01 (Notification System)
**Why now:** These three lessons directly patch the session_001 gaps. You'll be able to explain the APNs/FCM relay model, async message pipeline, and persistent connection architecture. This is the fastest path to a meaningful improvement from your 9/25 baseline score.

### Gate 2 — URL Shortener, Rate Limiter
**Unlock after:** Gate 1 + L1-05, L1-06, L1-07, L1-11
**Questions:** L3-02 (URL Shortener), L3-04 (Rate Limiter)
**Why now:** Both questions are relatively self-contained and lean on databases and caching. Good confidence builders before tackling fan-out heavy designs.

### Gate 3 — Social Systems (Feed, Chat, Crawler)
**Unlock after:** Gate 2 + L1-03, L1-04, L1-10, L2-01, L2-02
**Questions:** L3-03 (Social News Feed), L3-05 (Web Crawler), L3-09 (Chat System)
**Why now:** Feed and chat require fan-out reasoning (L2-01) and event-driven thinking (L2-02). Chat specifically requires the WebSocket routing knowledge from L1-01.

### Gate 4 — Storage, Search, Distributed Cache
**Unlock after:** Gate 3 + L1-09, L2-05, L2-06, L2-10
**Questions:** L3-06 (Search Autocomplete), L3-07 (Distributed Cache), L3-08 (File Storage)
**Why now:** These require sharding and replication pattern reasoning that L2-05/06 cover. File storage needs object storage fundamentals from L1-09.

### Gate 5 — Advanced Systems (Payment, Ride Sharing, Streaming)
**Unlock after:** Gate 4 + L1-12, L1-13, L2-04, L2-08, L2-09, L2-11
**Questions:** L3-10 (Video Streaming), L3-11 (Ride Sharing), L3-12 (Payment System), L3-13 (Ticketing)
**Why now:** Payment and ticketing require saga pattern (L2-04) and outbox pattern (L2-11) for correctness. Ride sharing needs real-time and geospatial reasoning. Video streaming needs CDN + object storage. These are the hardest archetypes and only make sense after full Layer 1 + most of Layer 2.

### Gate 6 — Modern Systems (AI, Leaderboard, Collab, Live)
**Unlock after:** Gate 5 + L1-14, L2-12, L2-13
**Questions:** L3-14 (Recommendation), L3-15 (Task Scheduler), L3-16 (Location), L3-17 (Leaderboard), L3-18 (Collaborative Editing), L3-19 (Live Streaming)
**Why now:** AI/ML infrastructure (L1-14) is now a baseline expectation for senior roles. Leaderboard and collaborative editing require specialized data structure and conflict resolution knowledge.

### Interview Cadence Rule

Do not schedule two mocks in the same week. After each mock, read the session notes before starting the next lesson block. The diagnostic value of a mock is in understanding your specific gaps, not in raw repetition.
