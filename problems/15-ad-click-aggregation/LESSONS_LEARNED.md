# Lessons Learned — Ad Click Aggregation

---

## Real Interview Stories

### Story 1 — Meta, E6 (2024)

The candidate proposed a Flink pipeline that guaranteed exactly-once semantics for all click events, reasoning that billing accuracy required it. The interviewer from a former Meta staff engineer asked: "What's the latency impact of exactly-once with Flink's two-phase commit sink?"

The candidate didn't know. The interviewer explained: exactly-once with an external sink requires Flink to checkpoint before committing to the sink. Checkpoint intervals are typically 1–5 minutes in production. With a 1-minute checkpoint interval, real-time data is at minimum 1 minute stale. The candidate's requirement was "< 30 seconds latency."

The interviewer guided the candidate toward the correct answer: at-least-once delivery for the real-time path (low latency, approximate), with idempotent writes to ClickHouse (dedup by click_id on insert). For billing, use the nightly Spark job over raw events in S3, which can do global deduplication over the full 24-hour window with exactly-once semantics.

**The lesson: exactly-once semantics in stream processing are not free. They add latency equal to the checkpoint interval. Know this trade-off. For real-time analytics where approximate is acceptable, at-least-once + idempotent sink is the industry standard.**

---

### Story 2 — Google, L5 (2023)

The candidate designed a system and, when asked "how do you count unique users who saw an ad?", proposed storing a set of user IDs per ad per day in Redis. The interviewer asked: "How big is that set for a viral ad with 50 million unique viewers?"

The candidate calculated: 50 million × 8 bytes (user ID) = 400 MB per ad per day. The interviewer: "We have 10 million active ads. What's the total storage?" Candidate: 10M × 400 MB = 4 PB per day. Obviously impossible.

The interviewer asked if the candidate knew of a more efficient data structure for counting distinct elements. The candidate said "maybe a Bloom filter?" The interviewer: "A Bloom filter tells you if an element was seen before, but it can't give you the count. What you need is something that estimates the count of distinct elements."

The candidate didn't know HyperLogLog. The interviewer explained it: ~1.5 KB fixed memory regardless of cardinality, ~2% error rate. At 10M active ads: 10M × 1.5 KB = 15 GB — completely manageable.

**The lesson: HyperLogLog is a named data structure for cardinality estimation. Know it, understand when it applies (distinct counts), understand its error rate (~2%), and contrast it with Count-Min Sketch (frequency estimation). HLL is a required vocabulary item for senior ad-tech interviews.**

---

### Story 3 — ByteDance/TikTok, Senior Engineer (2024)

The candidate proposed partitioning the Kafka topic by `ad_id`. The interviewer asked: "TikTok runs a new ad for a major movie release. The ad gets 100,000 clicks per second for 5 minutes during the premiere. What happens in your Kafka cluster?"

The candidate realized: 100,000 events/sec going to one partition, one broker, one consumer. Consumer lag grows. Real-time dashboard for that ad shows data from 5+ minutes ago. Other campaigns unaffected — their partitions are fine.

The interviewer: "How do you fix this?" The candidate proposed increasing the number of partitions. The interviewer: "You already have 100 partitions. The movie ad is in one of them. Adding more partitions doesn't help unless you can move the hot key to the new partitions. How do you route a specific ad to multiple partitions?"

The candidate arrived at key salting (append a random 0–7 suffix to the ad_id before partitioning) but couldn't explain how to reconstruct the total count from 8 partial counts. The interviewer explained two-level aggregation: each of 8 Flink consumers maintains its own partial count; a downstream reduce step sums the 8 partials.

**The lesson: hot partition is not an edge case in ad systems — it's the normal case for viral content. Key salting + two-level aggregation is the standard solution. Know it well enough to explain the two-stage reduction step.**

---

## General Pattern Mistakes

**Mistake 1: Single pipeline for real-time and billing.** Proposing one Flink job that serves both campaign dashboards and generates billing totals. The problem: optimizing for billing accuracy (exactly-once, possibly higher latency) conflicts with optimizing for real-time display (low latency, approximate acceptable). Two consumers of the same Kafka topic — one for real-time (Flink), one for billing (Spark over S3) — serve each purpose optimally.

**Mistake 2: HyperLogLog confusion.** Using HLL to count total clicks (wrong — HLL counts distinct values, not total frequency). Or using Count-Min Sketch to count unique users (wrong — CMS estimates frequency for a given key, not cardinality of a set). Or claiming HLL has zero false positives (wrong — ~2% standard error, tunable). These confusions signal that the candidate knows the names but not the mechanics.

**Mistake 3: No mention of late-arriving events.** Mobile devices are offline, airplanes lose connectivity, user agents retry on timeout. Ad click events frequently arrive minutes or even hours after they occurred. A Flink pipeline that doesn't handle late events will silently drop them (after the window closes). For a billing system, dropped events = lost revenue. The answer: watermark with grace period (5–30 min), plus batch job as backstop for very late events.

**Mistake 4: No serving layer.** The pipeline produces per-minute aggregates. The candidate stops there. But campaign managers ask "how many clicks in the last 24 hours?" The serving layer exists to stitch together pre-aggregated buckets to answer arbitrary range queries. Without it, the aggregation pipeline produces data nobody can easily query.

**Mistake 5: Over-indexing on exactly-once without knowing the cost.** Many candidates say "we need exactly-once" without knowing that Flink's exactly-once with two-phase commit sinks adds the checkpoint interval to end-to-end latency. This is a fundamental trade-off: stronger delivery guarantee → higher latency. For ad click dashboards with a 30-second freshness requirement, exactly-once is incompatible with the SLA.

---

## Self-Assessment Checklist

**Fundamentals:**
- [ ] Identified the dual pipeline requirement before designing (real-time vs. billing accuracy have different needs)
- [ ] Chose tumbling windows as the storage primitive and explained range query composition
- [ ] Addressed at-least-once vs. exactly-once and their latency implications

**Data Structures:**
- [ ] Named HyperLogLog for unique user count, with error rate (~2%) and memory (~1.5 KB)
- [ ] Named Count-Min Sketch for click frequency estimation and distinguished it from HLL
- [ ] Correctly assigned each data structure to the right use case

**Scale:**
- [ ] Derived peak events/sec from capacity hints
- [ ] Identified hot partition problem for viral ads
- [ ] Described key salting + two-level aggregation as the solution

**Failures:**
- [ ] Addressed late-arriving mobile events (watermark + grace period + batch backstop)
- [ ] Addressed Flink failure (checkpoint + replay from Kafka)
- [ ] Addressed deduplication window separately from watermark grace period

**Advanced:**
- [ ] Explained why billing uses Spark batch over S3 instead of trusting Flink stream counts
- [ ] Described the serving layer's bucket-stitching for arbitrary range queries
- [ ] Mentioned click fraud credit as a reason the batch reconciliation layer exists

---

## Remediation Targets

**If you didn't know HyperLogLog:** Study the HLL data structure. Key properties: ~1.5 KB fixed memory regardless of cardinality; ~2% standard error (configurable); supported natively in Redis (PFADD/PFCOUNT) and ClickHouse (uniq function). It answers "how many distinct X?" never "how many times did X occur?"

**If you proposed exactly-once without knowing the cost:** Study Flink's checkpointing and two-phase commit sinks. The checkpoint interval is the key variable. With a 1-minute checkpoint interval, your real-time data is at best 1 minute stale. For a 30-second requirement, exactly-once is incompatible. The production answer is: at-least-once + idempotent sink for real-time; batch for accuracy.

**If you missed the hot partition problem:** Run the numbers for a viral ad. 100K clicks/sec for one ad. One Kafka partition handles ~1M events/sec in theory, so a single ad isn't a crisis today — but at 1M events/sec for a major viral campaign, it is. Know that key salting is the solution and understand the two-level aggregation it requires.

**If you didn't address late events:** Think about the mobile use case. A user clicks an ad on a subway with no signal. The event is queued locally. 20 minutes later, they emerge with signal — the event is sent. Your 1-minute window for that event has long since closed. What happens? The answer is a Flink side output for late events (beyond grace period) + batch job that processes the full 24-hour event log including very late arrivals.
