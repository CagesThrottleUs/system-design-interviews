# Mock Interview Script — Ad Click Aggregation

**For the interviewer only. Do not share with the candidate.**

---

## Pre-Session Setup

**Target level:** Senior engineer (L5/ICT3 equivalent)
**Expected duration:** 45 minutes
**This is an Advanced problem.** Stream processing depth expected.
**Red flags:** Single pipeline for real-time + billing, HyperLogLog misuse, no late event handling, no hot partition awareness, exactly-once without knowing the latency cost
**Green flags:** Dual pipeline separation, HLL vs CMS distinction, key salting + two-level aggregation, watermark grace period, serving layer for range queries

---

## Opening

"I'd like you to design a real-time ad click aggregation system. Advertisers buy ads, users click them, and we need to: (1) show campaign managers how many clicks their ads are getting in near-real-time, and (2) bill advertisers accurately at month-end based on total clicks. The platform processes 100 million clicks per day."

*Let the candidate ask clarifying questions before starting.*

---

## Phase 1: Requirements (0–8 min)

**Expected clarifying questions:**
- Real-time latency requirement? (30 seconds — influences pipeline design significantly)
- Billing accuracy requirement? (< 0.01% — means approximate real-time, exact batch)
- Unique users vs. total clicks? (Drives HLL vs counter choice)
- Late-arriving events? (Mobile offline retry)
- Deduplication window?

**Probe if candidate doesn't ask about accuracy:** "Your dashboard shows 48,293 clicks for an ad today. Your billing invoice says 48,301. Which number is right, and why might they differ?"
Expected: real-time path is approximate (at-least-once + potential duplicates); billing path is exact (batch dedup over full window). The difference is normal and expected.

**Gate:** Candidate must have identified the tension between real-time latency and billing accuracy before proceeding.

---

## Phase 2: Estimation (8–13 min)

**Guide candidate to key numbers:**
- 100M clicks/day ÷ 86,400 = **1,157 clicks/sec average**
- Peak (10×): **11,570 clicks/sec** — modest throughput for Kafka
- Per-minute aggregates: **10M ads × 1,440 min = 14.4B rows/day** — too many to store individually
- Per-day aggregates: **10M ads × 1 row = 10M rows/day** — very manageable
- Implication: need a rollup hierarchy (per-minute → per-hour → per-day)

**Probe:** "Your ClickHouse table stores per-minute aggregates for all 10 million ads. How many rows per day?" Answer: 14.4 billion. "At 20 bytes per row, that's 288 GB/day. Is that OK for 30 days?" Answer: 8.6 TB. Then: "But for months-old data, do we need per-minute granularity?" Drive toward rollup hierarchy.

---

## Phase 3: HLD (13–28 min)

**Expected components:**
1. Kafka (partitioned by ad_id with salting)
2. Flink (real-time windowed aggregation)
3. Spark (nightly batch for billing)
4. ClickHouse (aggregation store for queries)
5. S3 (raw event archive for reprocessing)
6. Serving layer (range query composition)

**Critical probe — single pipeline:** "Can you use the same Flink job for both the real-time dashboard and billing accuracy?"
Expected: No. Exactly-once for billing → adds checkpoint interval latency → violates 30s requirement. Two separate consumers.

**Critical probe — hot partition:** "A major brand launches a viral campaign. One ad receives 80,000 clicks per second for 30 minutes. Walk me through what happens in your Kafka cluster."
Expected: hot partition; consumer falls behind; dashboard shows stale data for that ad. Solution: key salting (random 0–7 suffix) → spread across 8 partitions → two-level aggregation.

**Critical probe — HLL:** "A campaign manager wants to know how many unique users clicked their ad today. Walk me through how you'd compute that."
Expected: HyperLogLog. PFADD user_id per ad per day. PFCOUNT returns approximate unique count with ~2% error. Contrast with exact count (requires storing full user_id set — too expensive at scale).

---

## Phase 4: Deep Dive (28–40 min)

Choose the weakest area:

**Option A — Late Events:**
"A user clicks an ad while on a flight with no WiFi. Their phone is in airplane mode for 4 hours. When they land, the click event is sent to your system. Walk me through what happens: which window does this event belong to? Is it counted? How do you handle it in both the real-time and billing paths?"
Expected: 4-hour-late event is beyond Flink's watermark grace period (5 min) → goes to side output. Real-time path: not counted (acceptable). Billing path: Spark batch job reads ALL raw events from S3, including this one, so it IS counted in billing.

**Option B — Serving Layer:**
"A campaign manager opens their dashboard and requests: 'show me hourly click counts for ad X for the last 30 days.' Walk me through how your serving layer answers this query efficiently."
Expected: decompose range into buckets → older days use per-day aggregates in ClickHouse → recent days use per-hour aggregates → current partial hour uses per-minute aggregates → sum the buckets. Point out this is all pre-computed, so summation is sub-millisecond per bucket.

**Option C — Exactly-Once vs At-Least-Once:**
"Your billing depends on accurate click counts. You propose at-least-once delivery with idempotent sinks. Walk me through exactly how duplicates can occur, and how idempotent sinks prevent double-counting."
Expected: producer retries on timeout → same event sent twice → Kafka may have two copies of same click_id. Flink produces count of 2 for that click. Idempotent sink: ClickHouse INSERT with click_id dedup (or Flink dedup operator). Alternatively: Spark batch does global dedup over all click_ids for the billing number.

---

## Phase 5: Failure & Scaling (40–44 min)

**Probe:** "Your Flink checkpoint fails and the job restarts. What happens to your click counts for the last 5 minutes?"
Expected: Flink replays from last successful checkpoint (5 minutes ago). Those 5 minutes of events are reprocessed. At-least-once: events may be counted twice in ClickHouse → idempotent writes (INSERT with dedup by event window + ad_id) prevent double-count at ClickHouse level.

**Probe:** "You've grown to 10 billion clicks per day. What breaks first?"
Expected: Kafka throughput (10B/day ÷ 86,400 = 116K/sec — still fine with 200+ partitions). ClickHouse write throughput for per-minute aggregates (14.4B rows/day → 167K inserts/sec → need to batch insert). Flink state size grows linearly with number of active ads.

---

## Closing

"Two final questions:
1. A bug was discovered in your click deduplication logic — for the last 7 days, some clicks were counted twice. How do you correct the historical counts in ClickHouse without taking the system offline?
2. An advertiser claims they were overbilled by $50,000 last month. Walk me through your investigation."

Strong answers: (1) Replay from S3 raw events with corrected dedup logic into a new ClickHouse table, then swap the serving layer to point to the corrected table. (2) Pull billing_totals from PostgreSQL for that advertiser + date range; compare with Spark batch logs; compare with PSP settlement; identify which click events were erroneously counted; cross-reference with dedup store to find false negatives.

---

## Scoring Checklist

| Area | Weight | 1 (Poor) | 3 (OK) | 5 (Strong) |
|------|--------|----------|--------|------------|
| Requirements clarification | 10% | Didn't ask about accuracy/latency split | Asked about latency; missed billing accuracy | Identified real-time vs billing tension, asked about dedup window, late events |
| HLD completeness | 30% | Single pipeline; no serving layer | Dual pipeline present; ClickHouse mentioned | Dual consumer (Flink + Spark), S3 archive, serving layer, rollup hierarchy |
| LLD depth | 30% | No HLL, no hot partition discussion | HLL mentioned; key salting mentioned | HLL vs CMS distinguished, key salting + two-level aggregation explained, watermark grace period |
| Tradeoff reasoning | 20% | No justification | Approximate vs exact tradeoff named | Exactly-once latency cost, HLL error rate vs memory, tumbling vs sliding window storage |
| Communication clarity | 10% | Disorganized | Logical flow | Clear separation of ingest/process/serve, explicit failure narration |
