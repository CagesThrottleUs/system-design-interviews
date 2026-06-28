# Solution Guide — Ad Click Aggregation

Read after your attempt. If you haven't attempted yet, close this file.

---

## Component Map

| Component | Role | Technology Choice | Why |
|-----------|------|-------------------|-----|
| Click Ingestor | Receive, validate, deduplicate raw click events | Stateless HTTP servers | High write throughput; dedup check via Redis |
| Message Queue | Durable, replayable event log | Apache Kafka | Horizontal scale, replayable for reprocessing, handles 100K+ events/sec |
| Stream Processor | Real-time windowed aggregations | Apache Flink | Event-time windowing, stateful operators, exactly-once with checkpointing |
| Batch Processor | Accurate historical counts for billing | Apache Spark | Large-scale historical reprocessing; used nightly for billing reconciliation |
| Aggregation Store | Pre-computed window results, sub-500ms query | ClickHouse or Apache Druid | Columnar OLAP, billion-row range queries in <100ms |
| Raw Event Store | Archive of all raw events for replay | S3 + Parquet (partitioned by date) | Cheap cold storage; enables reprocessing when bugs found |
| Serving Layer | Combine pre-aggregated buckets for arbitrary range queries | Stateless query service | Stitches per-minute aggregates into requested time windows |
| Dedup Store | In-window click event deduplication | Redis with TTL | 30-minute dedup window, sub-millisecond lookup |

---

## Architecture Diagram

```
INGEST PATH (100K+ events/sec)
────────────────────────────────────────────────────────────────────

  Ad Server / Mobile App
    │  POST /click { ad_id, user_id, click_id, timestamp, ... }
    ▼
┌───────────────────┐
│  Click Ingestor   │
│  (stateless)      │
│                   │  1. Validate schema
│                   │  2. Check dedup: Redis SETNX click_id TTL=30min
│                   │     (if exists: discard duplicate)
│                   │  3. Produce to Kafka: partition key = ad_id (salted)
└───────────────────┘


KAFKA PARTITIONING (hot partition solution)
────────────────────────────────────────────

  ad_id: "ad_viral_123"  →  ad_viral_123:0, ad_viral_123:1, ..., ad_viral_123:7
  (salted key = ad_id + random suffix 0-7)

  Kafka partition 0:  [ad_viral_123:0, ad_small_456, ad_rare_789, ...]
  Kafka partition 1:  [ad_viral_123:1, ad_other_001, ...]
  ...
  Kafka partition 7:  [ad_viral_123:7, ...]
  → viral ad spread across 8 partitions; aggregation sums the 8 partial counts


REAL-TIME PATH (latency < 30 seconds)
────────────────────────────────────────────────────────────────────

  Kafka
    │
    ▼
┌───────────────────────────────────────────────────────────────────┐
│  Flink Stream Processor                                           │
│                                                                   │
│  1. Parse event                                                   │
│  2. Assign event time (use event timestamp, not processing time)  │
│  3. Apply tumbling window: 1 minute                               │
│     Window closes when watermark passes window_end + 5s grace     │
│  4. Group by (ad_id, window_start_minute)                        │
│  5. Emit: { ad_id, window_start, click_count }                    │
│                                                                   │
│  Late events (arrive > 5s after watermark): allowed up to 5 min  │
│  After 5 min grace: late events go to side output (counted later) │
└───────────────────────────────────────────────────────────────────┘
    │
    ▼
┌───────────────────┐     ┌──────────────────────────────────────┐
│  ClickHouse       │     │  Raw Event Store (S3/Parquet)         │
│  (aggregation     │     │  Partitioned by: date/hour/ad_id      │
│   store)          │     │  Retention: 30 days                   │
└───────────────────┘     └──────────────────────────────────────┘
    │ (sub-500ms queries)              │
    ▼                                  ▼
Campaign Dashboard            Batch processor for billing
                              (Spark job, runs daily)


BATCH PATH (billing accuracy, runs nightly)
────────────────────────────────────────────────────────────────────

  S3 raw events (yesterday's partition)
    │
    ▼
┌───────────────────────────────┐
│  Spark Batch Job              │
│                               │
│  1. Read all raw events       │  exact counts, no approximations
│  2. Global dedup by click_id  │  longer window than real-time
│  3. Aggregate by ad_id / day  │
│  4. Compare with Flink output │
│  5. Write billing totals to   │
│     Billing DB (PostgreSQL)   │
└───────────────────────────────┘


SERVING LAYER (arbitrary range queries)
────────────────────────────────────────

  Client: "clicks for ad_123 from Jan 1 to Jan 15"
    │
    ▼
┌──────────────────┐
│  Query Service   │
│                  │  1. Decompose [Jan 1, Jan 15] into pre-agg buckets
│                  │  2. Query ClickHouse for per-day agg (Jan 1-14)
│                  │  3. Query ClickHouse for per-hour agg (Jan 15 hours)
│                  │  4. Query ClickHouse for per-minute agg (current partial)
│                  │  5. Sum the buckets → return total
└──────────────────┘
```

---

## Capacity Math

**Click volume:**
- 100 million clicks/day ÷ 86,400 = **1,157 clicks/sec average**
- Peak (10×): **11,570 clicks/sec** — well within Kafka's capacity (millions/sec)
- Round to **100,000 events/sec** for headroom (campaigns can spike 100× briefly)

**Kafka sizing:**
- 100,000 events/sec × 500 bytes = **50 MB/sec** throughput
- With 3× replication: **150 MB/sec** to Kafka brokers
- At 100K events/sec, 100 partitions provides 1,000 events/sec per partition
- Each partition = one Flink subtask → **100 Flink workers** for real-time path

**Hot partition math:**
- 10M active ads. Top 1% (100,000 ads) generate 80% of clicks.
- Worst case: one viral ad = 10,000 clicks/sec. One Kafka partition can handle ~100K events/sec.
- Not a crisis today, but with key salting (8 suffixes), viral ad is spread across 8 partitions → 8× more headroom.

**Storage:**
- Raw events: 100M events/day × 500 bytes = **50 GB/day** compressed (~10 GB at 5:1 ratio)
- 30-day retention: **300 GB** on S3 — cheap
- Aggregated per-minute: 10M ads × 1,440 minutes/day × 20 bytes per aggregate = **288 GB/day**
- Per-hour rollup (60× reduction): **4.8 GB/day** — 3 years = **5.2 TB**
- Per-day rollup (1,440× reduction from per-minute): **200 MB/day**

**Flink checkpoint interval:**
- 1-minute checkpointing: max 1 minute of re-processing on failure
- Checkpoint state: 10M ads × (1 minute window state) × 8 bytes per count = **80 MB per checkpoint** — trivial

---

## API Design

**Ingest click event:**
```
POST /v1/clicks
Body: {
  "click_id": "clk_abc123",    // client-generated UUID for deduplication
  "ad_id": "ad_xyz",
  "user_id": "usr_hash_789",   // hashed for privacy
  "placement_id": "pub_001",
  "device_type": "MOBILE",
  "geo": "US",
  "timestamp": "2025-01-15T14:23:45.123Z"   // event time, not server time
}
Response 200: { "status": "accepted" }
Response 409: { "status": "duplicate" }     // click_id already seen
```

**Query click count (arbitrary time range):**
```
GET /v1/ads/{ad_id}/clicks?start=2025-01-15T00:00:00Z&end=2025-01-15T23:59:59Z&granularity=HOUR
Response 200: {
  "ad_id": "ad_xyz",
  "total_clicks": 48293,
  "granularity": "HOUR",
  "data_points": [
    { "window_start": "2025-01-15T00:00:00Z", "clicks": 1204 },
    ...
  ]
}
```

**Query unique user count (HyperLogLog):**
```
GET /v1/ads/{ad_id}/unique-users?start=2025-01-15T00:00:00Z&end=2025-01-15T23:59:59Z
Response 200: {
  "ad_id": "ad_xyz",
  "unique_users_approx": 38201,
  "error_rate": "~2%"
}
```

---

## Data Model

**click_aggregates table (ClickHouse, the hot query path):**
```sql
CREATE TABLE click_aggregates (
  ad_id           String,
  window_start    DateTime,
  granularity     Enum('MINUTE', 'HOUR', 'DAY'),
  click_count     UInt64,
  unique_users_hll AggregateFunction(uniq, UInt64),  -- HyperLogLog for unique users
  device_mobile   UInt32,
  device_desktop  UInt32,
  geo_us          UInt32,
  geo_other       UInt32
) ENGINE = AggregatingMergeTree()
PARTITION BY toYYYYMM(window_start)
ORDER BY (ad_id, granularity, window_start);
-- ClickHouse merges partial aggregates from multiple inserts automatically
```

**billing_totals table (PostgreSQL — exact counts for invoicing):**
```sql
CREATE TABLE billing_totals (
  ad_id           VARCHAR(64) NOT NULL,
  billing_date    DATE NOT NULL,
  click_count     BIGINT NOT NULL,      -- exact, from Spark batch job
  billable_amount BIGINT NOT NULL,      -- in microcents to avoid floating point
  processed_at    TIMESTAMP NOT NULL,
  PRIMARY KEY (ad_id, billing_date)
);
```

**dedup_keys table (Redis, hot path):**
```
Key:   dedup:{click_id}
Value: "1"
TTL:   1800 seconds (30 minutes)
Type:  SETNX — atomic, one winner
```

---

## Key Design Decisions

### Decision 1: Dual Pipeline (Lambda-Like) for Real-Time + Billing Accuracy

This is the core architectural choice. The two consumers of click data have irreconcilable requirements:

| Consumer | Latency Need | Accuracy Need |
|----------|-------------|---------------|
| Campaign dashboard | < 30 seconds | Approximate (within 1%) acceptable |
| Billing | < 24 hours | < 0.01% error required |

| Architecture | Speed path | Accuracy path | Operational burden |
|-------------|-----------|---------------|-------------------|
| Lambda | Flink (real-time) | Spark batch | Two separate codebases; must stay in sync |
| Kappa | Flink only | Flink reprocessing from Kafka | Single codebase; Flink handles both |
| Dual consumer (recommended) | Flink consuming Kafka | Spark consuming S3 (written from Kafka) | Two consumers of same Kafka topic — separate concerns |

**Choice: Dual consumer.** Same Kafka topic, two consumers. Flink consumes for real-time. Spark runs nightly over S3 (where all events are also written) for billing accuracy. This is conceptually closer to Lambda than Kappa, but avoids the "dual codebase" problem — each consumer serves one purpose and is independently maintainable.

**Trade-off accepted:** Flink counts may differ slightly from Spark counts (different dedup windows, late arrival handling). Campaign managers see approximate real-time data; finance uses the authoritative Spark-computed number. This is Google's Photon + Mesa model: real-time approximate for display, batch accurate for billing.

---

### Decision 2: HyperLogLog for Unique Users, Not Exact Count

**The problem:** 10 million active ads × 100 million unique users. Exact unique-user count per ad per day requires storing a set of user IDs. For a viral ad: set size = 10 million user IDs × 8 bytes = 80 MB per ad per day. For 10 million active ads: 800 TB per day. Impossible.

| Approach | Memory per ad per day | Error rate |
|----------|----------------------|------------|
| Exact (hash set) | up to 80 MB | 0% |
| HyperLogLog (Redis PFADD) | ~1.5 KB fixed | ~2% standard error |
| Count-Min Sketch | Few KB | Not applicable (CMS counts frequency, not cardinality) |

**Choice: HyperLogLog for unique user counts.** HLL uses ~1.5 KB regardless of cardinality — 80 bytes for 100 unique users or 100 million. Error rate is ~2%, meaning the reported "38 million unique users" may be anywhere from 37.2M to 38.8M. For campaign performance reporting, this is acceptable. For billing (CPM = cost-per-mille impressions), use exact impression count from the event count (not cardinality).

**Critical interview distinction:**
- **HyperLogLog (HLL):** "How many DISTINCT users saw this ad?" — cardinality estimation
- **Count-Min Sketch (CMS):** "How many TIMES was this ad clicked?" — frequency estimation
- Using HLL to count total clicks is WRONG — that's frequency, not cardinality
- Using CMS to count unique users is WRONG — that's cardinality, not frequency

---

### Decision 3: Tumbling Windows for Aggregation, Not Sliding

| Window Type | Use Case | Memory Cost | Storage Cost |
|------------|----------|-------------|--------------|
| **Tumbling (1-min buckets)** | "Clicks per minute for ad X" | Low — each event in exactly one window | Low — one row per (ad, minute) |
| **Sliding (1-hour, 10-min hop)** | "Clicks in rolling last hour" | High — each event in 6 windows | 6× more rows |
| **Session** | "Clicks in user session" | Variable | Not relevant for aggregation |

**Choice: Tumbling windows as the storage primitive; sliding windows computed at query time.** Store per-minute tumbling buckets. When a campaign manager asks "clicks in the last hour," the serving layer sums 60 one-minute buckets. This is much more storage-efficient than pre-materializing sliding windows. For common query patterns (last 1 hour, last 24 hours), the serving layer's summation is sub-millisecond.

**Trade-off accepted:** Slightly more query-time computation than pre-materialized sliding windows. At the serving layer, summing 60 or 1,440 pre-aggregated buckets takes microseconds in ClickHouse. The storage savings (60× reduction in stored rows) make this the clear winner.

---

## Deep Dive: The Hot Partition Problem

A viral ad receives 50,000 clicks/sec at peak. You've partitioned Kafka by `ad_id`. All 50,000 events/sec go to one partition, one broker, one Flink consumer. That consumer falls behind; its consumer lag grows; the dashboard for that ad stops updating. Meanwhile, other partitions process 100 events/sec each.

**Why this matters:** The Pareto principle in ad traffic — roughly 5% of ads generate 80% of clicks. One campaign launch can instantly create a hot partition.

**Solutions:**

**1. Key salting:**
Before producing to Kafka, append a random suffix 0–(N-1) to the ad_id:
- Instead of key = `ad_viral_123`
- Use key = `ad_viral_123:0`, `ad_viral_123:1`, ..., `ad_viral_123:7`
- Events spread across 8 partitions → 8 consumers → 8× throughput

The catch: now the count for `ad_viral_123` is split across 8 consumers. The Flink aggregation step must sum the 8 partial counts:
```
Flink subtask 0: count_ad_viral_123_0 = 6,234
Flink subtask 1: count_ad_viral_123_1 = 6,189
...
Flink subtask 7: count_ad_viral_123_7 = 6,302
→ Total: sum(6,234 + 6,189 + ... + 6,302) = 50,041
```

This is a two-stage aggregation: local pre-aggregation per salt → global aggregation at serving layer.

**2. Two-level aggregation in Flink:**
Local pre-aggregation (per-consumer, every 5 seconds): reduce 50,000 raw events to 1 pre-aggregated count entry. Send the pre-aggregated entry downstream to the global aggregation window.

This reduces hot-partition throughput on the downstream global aggregation step by 10,000× (one count per 5 seconds instead of 50,000 raw events per 5 seconds).

**3. Detection and dynamic repartitioning:**
Monitor Kafka consumer lag per partition. If one partition has > 5× average lag, flag it as hot. Use Kafka's `kafka-reassign-partitions` to move it to a dedicated set of brokers. Or: assign additional consumers to that partition group (one partition can only be consumed by one consumer per group — so you'd need to increase partition count first).

---

## Failure Modes & Mitigations

| Failure | Impact | Detection | Mitigation |
|---------|--------|-----------|------------|
| Flink checkpoint fails | Counts drift; duplicate processing on restart | Checkpoint timeout alert | At-least-once delivery; Flink replays from last successful checkpoint; accept brief double-counting in dashboard (reconciled by batch) |
| Kafka broker failure | Partitions on that broker unavailable | Consumer lag spike | Kafka replication (3×): leader election < 30s; consumers automatically reconnect to new leader |
| ClickHouse node failure | Dashboard queries fail | Health check | ClickHouse Replicated engine: automatic failover to replica |
| Redis dedup store failure | Duplicate clicks flow through | Write failure rate spike | Accept brief duplicate window; batch dedup catches them; charge advertisers conservatively |
| Late-arriving mobile events | Under-count in real-time window | Events arrive after window closes | Flink watermark with 5-minute grace period; side output for stragglers; batch job includes all events |
| Hot partition (viral ad) | One Flink consumer overwhelmed | Consumer lag spike | Key salting (8 suffixes per ad); two-level aggregation pre-reduces event volume |
| Billing count mismatch (Flink vs Spark) | Advertiser disputed charge | Nightly comparison: Flink aggregate vs Spark aggregate | Trust Spark as the authoritative billing number; Flink is "good enough" for display; difference documented for advertisers |

---

## What Strong Candidates Do

1. **Name the dual pipeline architecture explicitly** and justify why real-time and billing accuracy cannot share the same pipeline. The key insight: billing requires exact counts, but exact counts in a stream processor require exactly-once semantics, which adds latency and complexity. Separating the concerns lets each pipeline optimize for its requirement.

2. **Correctly choose HyperLogLog for cardinality, Count-Min Sketch for frequency** — and state clearly which one answers which question. Confusing the two is a common error that signals shallow knowledge.

3. **Address hot partitions proactively** — viral ads are not an edge case in ad systems. Key salting is the standard solution. Know that salting requires two-stage aggregation at the consumer.

4. **Explain late-arriving event handling** — mobile devices batch events and send when connectivity resumes, potentially 30+ minutes later. Flink watermarks with a grace period (5 minutes is typical for real-time; batch job catches the rest) is the right answer.

5. **Separate the deduplication window** from the watermark grace period. Dedup window (30 minutes) is for preventing double-counting of retried events. Watermark grace period (5 minutes) is for handling late-arriving events that just haven't reached Kafka yet due to network delay.

6. **Know what exactly-once really costs** — Flink's exactly-once with external sinks requires two-phase commit, which adds checkpoint interval latency (typically 1–5 minutes) to end-to-end latency. For a 30-second real-time requirement, this is unacceptable. The correct choice: at-least-once for the stream processor + idempotent sink writes + batch reconciliation for billing.

---

## What Average Candidates Miss

1. **Proposing exact counting for real-time with no latency penalty.** Stream processors that guarantee exactly-once semantics (Flink with two-phase commit sinks) add the checkpoint interval to end-to-end latency. With 1-minute checkpoints, real-time data is at minimum 1 minute stale. For a 30-second latency requirement, this is unacceptable. The answer: approximate counts in real-time, exact counts in batch.

2. **Using HyperLogLog to count total clicks.** HLL counts distinct values. If you want to count how many times an ad was clicked (including multiple clicks by the same user), you want a simple counter, not HLL. HLL is specifically for "how many distinct users clicked this ad."

3. **Not addressing late events.** A mobile user clicks an ad on an airplane. The event is buffered. They land, turn on WiFi, and the event arrives 45 minutes later. Without a late-event policy, the click is assigned to the correct 1-minute time window but that window has already closed and been materialized. The click is silently dropped. This is both an accuracy problem and a billing problem.

4. **No mention of the serving layer.** Candidates describe a pipeline that produces per-minute aggregates but stop there. How does an advertiser query "clicks in the last 30 days"? The serving layer stitches together pre-aggregated buckets (per-day for older data, per-hour for recent data, per-minute for the live window) to answer arbitrary range queries. Without this, the data pipeline is complete but unusable.

5. **Ignoring click fraud detection's relationship to billing.** Google's approach: real-time detection filters obviously fraudulent clicks immediately; a second pass (post-processing) catches sophisticated fraud and issues retroactive credits. This "credit system" is why a batch reconciliation layer is necessary — even if you don't do fraud detection in your design, mentioning that invalid clicks must be retroactively credited signals business awareness.
