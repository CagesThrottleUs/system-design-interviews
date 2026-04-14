# /learn — System Design Learning Skill

## Purpose
Deliver structured system design lessons to Laksh, tracking progress across a 3-layer curriculum. Uses web research for current content. Always writes key terms to chat before speaking.

## On Invocation

### Step 1 — Read Progress
Read `curriculum/progress.md` to determine:
- Current layer and next lesson
- Identified knowledge gaps
- Lesson format preferences

Read `curriculum/00_overview.md` to understand the full curriculum map.

### Step 2 — Determine Mode
The user may invoke `/learn` in three ways:
1. **No args** → deliver the next lesson per progress.md
2. **Topic specified** (e.g., `/learn APNs`) → teach that specific topic
3. **Review** (e.g., `/learn review`) → quiz on the last completed lesson

### Step 3 — Research (if needed)
For any lesson, use WebSearch to fetch:
- Current official documentation (Apple docs, Kafka docs, etc.)
- Real engineering blog posts from companies who've built this at scale
- Recent updates or changes to technology behavior

Do not rely solely on training knowledge for technical specifics — verify with search.

### Step 4 — Write Key Terms to Chat FIRST
Before any voice explanation, write a text block to chat containing:
- Topic name and lesson code (e.g., L1-01)
- 5–10 key terms with one-line definitions
- Critical numbers/data points (e.g., "port 5223", "HTTP/2 for APNs", "64KB max payload")
- One "mental model" sentence that captures the core concept

Format:
```
---
**Lesson [CODE]: [TITLE]**

Key terms:
• [term]: [one-line definition]
• ...

Critical numbers:
• [metric/number]: [what it means]

Mental model: [one sentence that makes it click]
---
```

### Step 5 — Deliver Lesson via Voice
Use `mcp__voicemode__converse` to explain the topic conversationally. Structure:
1. **Why it matters** — connect to a real problem or the gap from sessions
2. **How it actually works** — first principles, not just buzzwords
3. **The key tradeoff** — what you give up by choosing this approach
4. **When to use it in interviews** — which question archetypes this shows up in
5. **Checkpoint question** — ask one question to verify understanding

Keep voice explanations focused. Write the detailed notes to the lesson file simultaneously or after.

### Step 6 — Write Full Lesson Notes
Save the lesson to `curriculum/layer[N]-[name]/L[N]-[NN]_[topic].md` with:
- Topic overview
- How it works (diagrams in ASCII if helpful)
- Key numbers and data points
- Tradeoffs
- When to use in interviews
- Common mistakes to avoid
- Sources

### Step 7 — Update Progress
Update `curriculum/progress.md`:
- Mark lesson complete
- Note any gaps that came up during the lesson
- Set next lesson

---

## Lesson Content Guide

### Layer 1 Foundation Lessons

**L1-01: Networking**
Research: TCP 3-way handshake, TCP vs UDP, HTTP/1.1 keep-alive vs HTTP/2 multiplexing, WebSockets upgrade process, Server-Sent Events, long polling vs short polling
Key numbers: TCP port range 1-65535, HTTP/2 default port 443, WebSocket upgrade via HTTP 101

**L1-02: Push Delivery (APNs & FCM)**
Research: Apple APNs documentation, FCM architecture, device token lifecycle
Key insight: Device is CLIENT. OS maintains ONE persistent connection to APNs (port 5223). Your backend sends to APNs via HTTP/2 POST. APNs delivers via persistent connection to device.
Key numbers: APNs port 5223 (device), APNs port 443 (provider), max payload 4KB (APNs), store-and-forward window varies by notification type

**L1-03: DNS & CDN**
Research: DNS resolution chain, CDN PoP architecture, cache-control headers, CDN cache invalidation strategies

**L1-04: Load Balancing**
Research: L4 vs L7 differences, consistent hashing algorithm, session affinity, health check patterns

**L1-05: Databases — Relational**
Research: B-tree vs LSM-tree indexes, WAL (write-ahead log), MVCC, primary-replica lag, read replica use cases

**L1-06: Databases — NoSQL**
Research: Cassandra ring architecture, DynamoDB partition keys, Redis data structures, when columnar vs document vs KV

**L1-07: Caching**
Research: Redis sentinel vs cluster, cache-aside pattern implementation, cache stampede / thundering herd, hot key problem

**L1-08: Message Queues & Streaming**
Research: Kafka partition, offset, consumer group internals; SQS visibility timeout; exactly-once semantics

**L1-09: Object Storage**
Research: S3 multipart upload, presigned URLs, eventual consistency model, storage classes

**L1-10: Distributed Systems**
Research: CAP theorem proof intuition, Raft leader election, vector clocks, CRDTs

**L1-11: API Design**
Research: REST idempotency, gRPC streaming modes, GraphQL N+1 problem, API gateway patterns

**L1-12: Security**
Research: TLS handshake, JWT structure and verification, OAuth2 flows, API rate limiting at gateway

**L1-13: Observability & Monitoring**
Research: The three pillars (metrics, distributed traces, logs), Prometheus metrics types (counter/gauge/histogram), OpenTelemetry, SLO vs SLI vs SLA definitions, alerting on burn rate vs threshold
Key insight: "How do you know it's working?" — every interviewer probes this. Observability is not optional in 2025.
Key numbers: p50/p95/p99 latency percentiles, error budget = 1 - availability target, trace sampling rates

**L1-14: AI/ML Infrastructure Basics**
Research: Feature store purpose and architecture (Feast, Tecton), vector databases (Pinecone, pgvector, Weaviate), embedding generation pipeline, online vs offline inference, cold start problem in recommendations, model serving latency targets
Key insight: AI/ML system design is now a baseline expectation at senior level in 2025. You need to know how a recommendation pipeline or similarity search system works at a high level even if not specializing in ML.
Key numbers: typical embedding dimensions (768, 1536), ANN search (approximate nearest neighbor) vs exact KNN tradeoff, p99 inference latency targets (< 100ms online serving)

### Layer 2 Pattern Lessons

**L2-11: Outbox Pattern**
Research: Transactional outbox pattern (save event to DB in same transaction as state change, separate poller publishes), inbox pattern for idempotent consumers, dual-write problem and why it causes data loss, Debezium CDC as alternative
Key insight: Solves the hardest distributed systems problem — "I updated the DB and also need to publish an event, but what if one fails?" Without outbox, you get silent data loss.

**L2-12: Backpressure & Flow Control**
Research: Consumer lag in Kafka, backpressure in reactive streams, load shedding strategies (drop newest vs oldest), flow control in gRPC, queue depth as the critical metric, producer throttling patterns
Key insight: Systems fail not from load spikes but from not knowing they're overloaded. Backpressure is the mechanism that propagates "slow down" signals upstream.

**L2-13: Data Pipeline Patterns**
Research: Lambda architecture (batch layer + speed layer + serving layer), Kappa architecture (streaming only, reprocessing via log replay), when each makes sense, Apache Flink vs Spark Streaming, reconciliation challenges
Key insight: Lambda was the dominant pattern 2010–2018. Kappa is the 2025 default for new systems because log-based storage (Kafka) enables reprocessing without a separate batch path.

### Layer 3 Archetype Lessons

**L3-13: Ticketing/Booking System (Ticketmaster)**
Research: Seat reservation with optimistic locking vs pessimistic locking, distributed lock with Redis (SETNX + TTL), seat hold timeout, waiting room pattern for demand spikes, exactly-once ticket issuance
Key challenges: Thundering herd on popular events, preventing double-booking under concurrent requests, fairness vs latency tradeoff in queue

**L3-14: Recommendation System (Netflix/Spotify)**
Research: Collaborative filtering vs content-based filtering, matrix factorization, two-tower neural network model, offline training pipeline, online serving with feature store, cold start problem (new user / new item), A/B testing for model evaluation
Foundations: L1-06 (NoSQL), L1-07 (Caching), L1-14 (AI/ML Infrastructure), L2-03 (CQRS), L2-13 (Data Pipelines)

**L3-15: Distributed Task Scheduler (Airflow at scale)**
Research: Task dependency DAG, distributed cron (leader election for single-fire), at-least-once vs exactly-once execution semantics, worker pool management, task state machine (pending/running/success/failed/retry), distributed lock for preventing duplicate runs
Foundations: L1-08 (Message Queues), L1-10 (Distributed Systems), L2-02 (Event-Driven), L2-08 (Reliability)

**L3-16: Location-Based Service (Yelp/Google Places)**
Research: Geospatial indexing approaches — quad tree vs geohash vs S2 cells, range query vs radius query, PostGIS, bounding box pre-filter + haversine refinement, read-heavy optimization with Redis geospatial commands
Key numbers: Geohash precision levels (1 char = 5000km, 6 chars = 1.2km, 9 chars = 5m), typical search radius (5–50km), POI update frequency

**L3-17: Leaderboard System**
Research: Redis sorted sets (ZADD/ZRANK/ZRANGE), global vs game-scoped leaderboards, around-me queries (rank ± N), leaderboard snapshot for historical rankings, partitioned leaderboards for scale, approximate rankings vs exact
Key insight: Redis sorted sets solve this problem almost perfectly for a single region. The interview is really about what happens at global scale where a single sorted set is too large.

**L3-18: Collaborative Editing (Google Docs)**
Research: Operational Transformation (OT) — the original approach, CRDTs (Conflict-free Replicated Data Types) — the modern approach, why eventual consistency works here, WebSocket connection management, presence indicators, document version vectors
Key insight: OT requires a central server to order operations. CRDTs are distributed by design — convergence without coordination. Google Docs uses OT; Figma uses CRDTs.

**L3-19: Live Streaming Platform (Twitch)**
Research: RTMP ingest (broadcaster → origin), HLS/DASH adaptive bitrate delivery, WebRTC for sub-second latency vs HLS tradeoff (10–30s vs <1s), CDN edge cache for stream segments, transcoding pipeline (multiple bitrate ladders), live vs VOD storage separation
Key numbers: HLS default segment size (2–6s), typical stream latency HLS (10–30s), WebRTC (<500ms), transcoding time must be < segment duration

---

## Interviewer Mode vs Teacher Mode

This skill runs in **Teacher Mode**. When in teacher mode:
- Explaining gaps is allowed and encouraged
- Hints are free — this is learning, not scoring
- Connect every concept back to "how does this change a system design?"
- After teaching a concept, always say: "Now where would you use this in the notification system we designed?"

When the user invokes `/interview`, switch to Interviewer Mode (strict, no hints, scored).

---

## Quality Bar

A lesson is complete when the user can:
1. Explain the concept in one sentence without jargon
2. State the main tradeoff
3. Name one system design question where it's relevant
4. Identify the mistake a candidate makes when they don't know it
