# Lessons Learned — URL Shortener

This document captures patterns extracted from real interview accounts and common failure modes. Use it for post-attempt diagnosis, not pre-attempt priming.

---

## Real Interview Stories

### Story 1: Google — Senior SWE (L5) — 2024

A candidate with strong backend experience jumped straight into the architecture after two minutes of requirements. They proposed a sensible design: API servers, a PostgreSQL DB with a `urls` table, and a Redis cache. The interviewer let it run for fifteen minutes before asking: "How do you generate the short code?"

The candidate said: "I'll MD5 the long URL and take the first 8 characters."

The interviewer followed up: "What happens when two different long URLs produce the same 8-character prefix?"

The candidate hadn't thought about this. They proposed checking the DB before inserting and retrying with a different prefix. The interviewer pushed: "How often does that happen at scale?" The candidate couldn't answer — they hadn't done the math. Birthday paradox on a 16^8 = 4.3 billion space sounds large, but at 6 billion insertions, the collision probability is near 100%.

**Where it went wrong:** No capacity estimation before architecture. The collision problem is only visible when you compute the namespace size against the expected volume.

**Consequence:** The candidate spent the last 20 minutes recovering rather than discussing tradeoffs, analytics, or failure modes. They never reached the LLD phase. Final score: 2.5/5 overall.

**Fix:** Always do napkin math on the short code namespace before choosing a generation strategy. 62^7 = 3.5 trillion; that's the number to know. Counter + Base62 eliminates the collision problem entirely — derive the shortCode from a guaranteed-unique ID.

---

### Story 2: Amazon — SDE II (L5) — 2023

A candidate proposed a clean design: stateless API servers, DynamoDB for URL storage, and a Redis cache for redirects. Solid. The interviewer asked about redirect behavior.

"I'll return a 301 redirect," the candidate said. "It's a permanent redirect, so it's semantically correct."

The interviewer asked: "The product manager says we need click analytics. How does your design support that?"

The candidate said: "I'll log to a click_events table on every redirect."

The interviewer paused and asked: "If a user clicks the link twice from the same browser, does the second click reach your server?"

The candidate realized the problem mid-answer. A 301 response caches in the browser permanently. The second click never leaves the browser. The analytics DB never sees it. The campaign manager sees 1 click when the actual number was 10,000.

**Where it went wrong:** 301 vs 302 is a decision with explicit product requirements implications. The candidate treated it as a technical detail when it directly determines whether the core secondary requirement (analytics) is achievable.

**Consequence:** The interviewer noted the candidate didn't reason about system-level implications of protocol choices. Score: 3/5. Reached L4 offer, not L5.

**Fix:** Before choosing a redirect type, ask yourself: "Does the product need to observe every click?" If yes, 302. If the product explicitly doesn't need analytics and wants browser-cached redirects for performance, 301 with a `Cache-Control: max-age` is reasonable. State the tradeoff explicitly.

---

### Story 3: Meta — E5 — 2024

A strong candidate handled requirements, napkin math, and the core architecture confidently. They proposed counter-based shortCode generation, DynamoDB, Redis cache, and a Kafka pipeline for analytics. The interviewer was impressed and spent 10 minutes on the analytics pipeline — Kafka consumers, ClickHouse aggregations, the deduplication strategy for at-least-once delivery.

Then the interviewer asked: "Walk me through your ID generator design."

The candidate said: "A single Redis instance with INCR gives me atomic, monotonically increasing IDs."

Interviewer: "What happens when that Redis instance goes down?"

Candidate: "I'd use Redis Sentinel for automatic failover."

Interviewer: "Failover takes how long? What happens during that window?"

The candidate couldn't quantify. Redis Sentinel failover takes 30–180 seconds depending on configuration. During that window, no new links can be created. The candidate hadn't thought through the availability implications.

**Where it went wrong:** The candidate identified the ID generator as a component but didn't apply the same failure-mode analysis they'd applied to other components.

**Consequence:** The E5 bar requires thinking about availability across every critical component without prompting. The candidate passed with E4 signal on availability reasoning.

**Fix:** For every component, ask "what breaks if this goes down?" For the ID generator:
- Redis INCR + Sentinel: 30–180 sec failover window, during which link creation fails
- Snowflake-style (2+ nodes, each with a unique machine ID baked in): single node failure is transparent, other node continues
- Range-based pre-allocation (DB allocates 1000-ID blocks to each app server): app servers generate IDs locally, tolerate ID generator downtime for the duration of the block

---

## General Pattern Mistakes

### Mistake 1: Skipping requirements — assuming the feature set

A candidate builds an elegant URL shortener with link previews, QR code generation, A/B testing destinations, and team folders. The interviewer asked for: create short URL, redirect, basic click count. The candidate spent 35 of 45 minutes on features that weren't asked for and never got to failure modes or the LLD.

Overbuilding is as disqualifying as underbuilding. The first 5 minutes of requirements clarification determines whether you're solving the right problem.

**Fix:** Ask 3–4 clarifying questions before drawing a single box. Confirm the MVP scope. Confirm analytics requirements — this determines the entire async pipeline decision. Confirm whether custom aliases are needed — this changes code generation from pure counter to "check-if-taken" with a conflict path.

---

### Mistake 2: Optimizing the write path, ignoring the read path

The write path (100M links/month ≈ 38 QPS) is trivially simple. A single DynamoDB table handles it with capacity to spare. Candidates who focus heavily on write optimization — debating whether to use synchronous vs asynchronous writes, worrying about write throughput — are solving the wrong bottleneck.

The read path (10B clicks/month ≈ 3,858 QPS average, 100K QPS peak) is where the system fails if you get it wrong. Every second the cache is cold or unavailable, 100K users are hitting the database. At DynamoDB pricing, that's also an expensive mistake.

**Fix:** State the read:write ratio explicitly. "100:1 read-to-write. This means the system is fundamentally a read-heavy KV cache. Cache is load-bearing, not an optimization."

---

### Mistake 3: Not explaining why the DB choice matches the access pattern

"I'll use PostgreSQL" — fine at small scale. "I'll use DynamoDB" — fine without justification if the interviewer doesn't push. But strong candidates connect the DB choice to the access pattern: write-once, read-many, no joins, single lookup by primary key. The entire case for a KV store over a relational DB is in those four properties. If you can't articulate them, the DB choice looks arbitrary.

**Fix:** When naming a database, immediately follow it with: "because the access pattern is X, and this DB is optimized for that." If you choose relational, explain how you'd shard it at 6 billion rows.

---

### Mistake 4: Analytics as an afterthought

"I'll add a click counter column to the URL table and increment it on every redirect." This approach has two problems:
1. Incrementing a counter on the hot-path redirect forces a write on every read. At 100K QPS, that's 100K writes/sec to the URL table — an order of magnitude more writes than the original write path generated.
2. Counter-only analytics doesn't support time-series breakdown, referrer analysis, or geo data.

The analytics requirement fundamentally changes the architecture. It introduces an async pipeline (Kafka) and a separate analytics store (ClickHouse or Cassandra). Candidates who don't surface this requirement in the requirements phase miss the opportunity to architect it correctly.

**Fix:** Ask "what analytics are needed?" early. If the answer is "click counts are enough," a simple counter column is defensible. If the answer is "breakdown by time, referrer, geo," you need an event pipeline — and you should design that explicitly.

---

### Mistake 5: No plan for viral link load spikes

"The system handles 3,858 redirect QPS" — true at average load. A link goes viral on Twitter. 50 million impressions in one hour. 10% click rate = 5M clicks in one hour ≈ 1,388 redirect QPS sustained, but the actual burst is much higher in the first few minutes.

Candidates who plan for average load but not peak load haven't thought about the system's failure mode. "We'll auto-scale" is not an answer — auto-scaling has latency (EC2 instances take 2–5 minutes to become healthy). During that window, the origin DB absorbs all traffic.

**Fix:** Design the cache for peak, not average. Warm the cache before known events (sports finals, product launches). Design the DB connection pool for peak. Use CDN caching for the hottest links. Mention that viral events are the design point, not the average.

---

## Self-Assessment Checklist

After your attempt, score yourself on each item:

**Requirements (before drawing anything)**
- [ ] Asked about read:write ratio (or stated ~100:1 assumption)
- [ ] Confirmed analytics requirements (none vs count vs full pipeline)
- [ ] Asked about custom aliases
- [ ] Asked about link expiry
- [ ] Stated the MVP scope explicitly

**Estimation (before architecture)**
- [ ] Computed write QPS from monthly volume
- [ ] Computed read QPS and stated the read:write ratio
- [ ] Computed 5-year storage requirement
- [ ] Identified the redirect cache as the load-bearing bottleneck

**Core Architecture**
- [ ] Named the cache as load-bearing (not optional)
- [ ] Chose 302 (not 301) and explained why
- [ ] Proposed a collision-free short code generation strategy
- [ ] Identified the ID generator as a potential SPOF and mitigated it

**Data Model**
- [ ] Chose a KV store and justified the access pattern match
- [ ] Defined the primary table with the right partition key
- [ ] Explained where analytics events are stored (separate from URL table)

**Failure Modes**
- [ ] Described behavior when Redis is down
- [ ] Described behavior when the ID generator is down
- [ ] Described behavior when a link is deleted while cached
- [ ] Described the viral link spike scenario

**Analytics (if required)**
- [ ] Proposed async Kafka pipeline (not synchronous DB write per click)
- [ ] Named a time-series-appropriate analytics store
- [ ] Described at-least-once delivery and idempotency strategy

---

## Remediation Targets

| If you scored 0–1 on... | The gap is... | Go to |
|-------------------------|---------------|-------|
| Requirements phase | Jumping to solutions before understanding constraints | curriculum/layer1-foundations — practice the 5-category framework every attempt |
| Capacity estimation | Not grounding design in numbers | SOLUTION_GUIDE.md Capacity Math section; practice deriving QPS from scratch |
| 301 vs 302 | Protocol semantics not connected to product requirements | SOLUTION_GUIDE.md Decision 2 + L1-01 networking lesson on HTTP redirect types |
| Short code generation | Collision math not internalized | SOLUTION_GUIDE.md Decision 1; work through the birthday paradox calculation |
| Caching strategy | Cache treated as optional optimization | SOLUTION_GUIDE.md Decision 3; re-derive why 50 ms SLA requires cache |
| ID generator HA | SPOF identification — missing failure-mode scan | SOLUTION_GUIDE.md Failure Modes table; build habit of asking "what breaks if this goes down?" for every box |
| Analytics pipeline | Async vs sync not intuitive | curriculum/layer1-foundations/L1-08 (Message Queues) — prerequisites for pipeline design |
| DB selection | Access pattern → DB type matching not fluent | curriculum/layer1-foundations/L1-05 and L1-06 (Relational + NoSQL) |
