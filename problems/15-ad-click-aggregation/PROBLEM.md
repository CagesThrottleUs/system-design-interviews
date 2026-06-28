# Ad Click Aggregation

**Difficulty:** Advanced
**Category:** Stream processing, Real-time analytics
**Time Box:** 45 min
**Key Concepts:** Lambda vs Kappa architecture, HyperLogLog vs Count-Min Sketch, windowing (tumbling/sliding/session), hot partition problem, exactly-once semantics
**Asked at:** Meta, Google, Amazon, ByteDance/TikTok, Twitter/X

---

## Problem Statement

Your company runs a digital advertising platform. Advertisers pay to display ads to users. Every time a user clicks an ad, the platform records the event and eventually bills the advertiser. The business depends on two things being simultaneously true: campaign managers need to see near-real-time click metrics to adjust bids and budgets (latency of seconds to minutes is acceptable), and finance needs accurate billing figures that reconcile with bank statements to the cent (latency of hours is acceptable, but accuracy is non-negotiable).

These two requirements — fast and approximate vs. slow and exact — are the core tension of this problem. Every design decision flows from navigating this tension.

For reference scale: Google processes tens of billions of ad impressions per day on its advertising platform, which generated $82.3 billion in revenue in Q4 2025 alone. Google's Photon system processes geographically distributed ad event streams in real-time and deliberately chose an at-most-once-with-reconciliation model for live reporting, with a separate accurate pipeline for billing. Alibaba's Flink cluster processed 1 trillion events per day as of 2019 during Singles Day (Double 11), peaking at 472 million transactions per second. LinkedIn's Kafka cluster carries 7 trillion messages per day across 100+ clusters.

---

## Actors

| Actor | Description | Primary Action |
|-------|-------------|----------------|
| **User** | Browses and clicks on ads | Generates click events |
| **Ad Server** | Renders ads and tracks impressions | Produces click events to the pipeline |
| **Advertiser** | Pays for ad placements | Views campaign dashboards, receives invoices |
| **Campaign Manager** | Internal role at advertiser | Monitors real-time metrics, adjusts bids |
| **Finance / Billing** | Internal platform team | Runs monthly billing reconciliation |
| **Fraud Detection Team** | Internal | Investigates and credits invalid clicks |

---

## Functional Requirements

**Core (MVP):**
- FR1: Ingest click events in real-time (ad_id, user_id, timestamp, placement_id, device_type)
- FR2: Provide per-minute click counts per ad_id to campaign managers within 30 seconds of the click
- FR3: Provide accurate total click counts per ad_id per day for billing (accuracy within 0.01%)
- FR4: Support querying click counts over arbitrary time ranges (last hour, last 7 days, last month)
- FR5: Deduplicate click events — the same click must not be counted twice

**Secondary:**
- FR6: Count unique users who clicked an ad (cardinality, not frequency)
- FR7: Aggregate by multiple dimensions: ad_id, campaign_id, advertiser_id, device_type, geo
- FR8: Detect click fraud patterns in near-real-time (anomalous click rates per publisher)
- FR9: Support data replay and reprocessing when bugs are found in aggregation logic
- FR10: Filter out known invalid clicks before counting

---

## Non-Functional Requirements

| Requirement | Target | Rationale |
|-------------|--------|-----------|
| Real-time dashboard latency | < 30 seconds end-to-end | Campaign managers need to react to performance changes |
| Billing accuracy | < 0.01% error rate | Billing discrepancy = advertiser disputes, revenue loss |
| Throughput | > 100,000 click events/sec peak | Major ad platforms see spikes during viral content |
| Storage retention | Raw events: 30 days; aggregates: 3 years | Reprocessing window + billing audit trail |
| Duplicate rate after deduplication | < 0.001% | Mobile apps retry on timeout; dedup is mandatory |
| Query latency for dashboards | < 500 ms for pre-aggregated windows | Campaign dashboards must feel responsive |
| Fault tolerance | No click event loss (at-least-once delivery) | Every click may represent billable ad spend |

---

## Capacity Estimation Hints

Use these to derive your own numbers before the solution guide:

- **Click volume:** 100 million clicks/day (roughly 1% CTR on 10 billion impressions)
- **Peak/average ratio:** ~10× (viral content, flash ad campaigns)
- **Average click event size:** 500 bytes (ad_id, user_id, timestamp, placement, device, geo)
- **Number of active ads:** 10 million ads at any time
- **Billing period:** Monthly; each ad may have millions of clicks
- **Cardinality problem:** 10 million ads × 100 million unique users

Derive: peak events/sec, Kafka partition requirements, storage for raw events, storage for aggregated data.

---

## Good Clarifying Questions to Ask

### Semantics
1. Is click count approximate or exact for the real-time dashboard? (Drives Lambda vs Kappa vs exact counting)
2. Is click count exact for billing? Or approximate within a tolerance? (< 0.01% error is tight — drives billing pipeline design)
3. What is the event deduplication window? (A mobile app might retry a click up to 30 minutes later — drives dedup storage)

### Scale
4. How many unique active ads? (10M × per-minute aggregates × 30-day retention = large storage)
5. Do we need per-ad aggregates only, or also per-campaign, per-advertiser, per-device? (Multi-dimension aggregation multiplies storage)

### Latency
6. What freshness do campaign managers need? (30 seconds vs. 5 minutes changes the pipeline design significantly)
7. Can we serve slightly stale data (e.g., 60 seconds behind) for the real-time dashboard in exchange for much simpler architecture?

### Windowing
8. Do we need sliding windows (e.g., "last 1 hour updated every minute") or tumbling windows (e.g., "clicks in each 1-minute bucket")? (Sliding windows require storing more intermediate state)
9. How do we handle late-arriving events? (Mobile devices batch events offline and send when connectivity resumes — late arrivals can be 30+ minutes)

### Reprocessing
10. If we discover a bug in our click-counting logic, can we reprocess historical data? (Requires raw events to be retained; drives retention policy)

---

## Key Components

<details>
<summary>Reveal components</summary>

- **Click Ingestor** — receives raw click events from ad servers via HTTP; validates, deduplicates at ingest, and produces to Kafka
- **Kafka Cluster** — durable, replayable event log; partitioned by ad_id (with hot partition mitigation)
- **Stream Processor (Flink)** — consumes from Kafka, applies windowed aggregations, produces per-minute counts
- **Batch Processor (Spark)** — daily reconciliation job consuming raw events for billing-accurate counts
- **Aggregation Store (ClickHouse / Druid)** — OLAP database for pre-aggregated window results; sub-500ms query latency
- **Raw Event Store (S3)** — long-term cold storage of all raw click events for reprocessing and audit
- **Serving Layer** — API that queries aggregation store, combines pre-aggregated buckets for arbitrary range queries
- **Deduplication Store (Redis)** — holds click_id set for dedup within a 30-minute window

</details>
