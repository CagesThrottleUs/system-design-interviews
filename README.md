# System Design Interview Practice

A structured practice environment for senior-level system design interviews, built around engineering judgment over knowledge recall.

---

## Philosophy

System design interviews exist because software engineering at scale is fundamentally about making consequential decisions under uncertainty. When a company asks you to design YouTube in 45 minutes, they are not testing whether you have memorized how YouTube works. They are watching how you think: how you scope ambiguity, which constraints you interrogate first, which tradeoffs you reach for instinctively, and whether you treat failure as an afterthought or as a first-class concern from the first minute of the conversation. The interview is a proxy for thousands of architectural decisions you will make over the course of a career.

What separates strong candidates from average ones at the senior level is not breadth of knowledge — it is the habit of problem-first thinking. Average candidates jump to solutions: "I would use Kafka for this." Strong candidates ask why before how: "What are the delivery semantics here? Is exactly-once required, or is at-least-once acceptable with idempotent consumers?" That question reveals whether an engineer understands that the choice of message queue is downstream of the consistency requirement, not upstream of it. Senior engineers have internalized that every component in a distributed system exists to solve a specific failure mode, and naming the component before naming the failure is reasoning backwards.

Forty-five minutes is enough time to reveal deep patterns. It shows whether you instinctively size a system before designing it, or whether capacity estimation feels like an interruption to the "real" work. It shows whether you drive toward bottlenecks proactively or wait to be asked. It shows whether your API design reflects the data model, or whether the two exist independently in your mind. Most importantly, it reveals whether tradeoffs feel like genuine tension to you — things that require real reasoning — or whether they feel like a box to check on the way to finishing the answer. Interviewers at the senior level are specifically looking for that tension. They want to hear you push back on your own design.

The practices in this repository are not designed to make you a better interview performer in isolation. They are designed to make you a better engineer who also performs well in interviews. When you practice debriefing a design against an evaluation rubric, you are building the same discipline you need when reviewing an architecture document at work. When you practice articulating failure modes before a probing interviewer, you are building the same instinct that stops you from shipping a distributed system without a dead-letter queue. The interview is synthetic; the thinking patterns it develops are not.

---

## Modes

**Solo Practice** is the default mode and should be your first pass on every problem. Set a 45-minute timer, open a scratch file or get out paper, and attempt the design before reading any guide. The attempt-first discipline is non-negotiable: reading the solution before attempting it converts a design exercise into a reading exercise, which trains recall instead of judgment. After the timer fires, compare your design against the SOLUTION_GUIDE for that problem. Note the gaps. Write them down. Do not move on until you can articulate specifically what you would have needed to know or think to close each gap.

**AI-assisted Interview Mode** invokes `/interview` inside Claude Code to start a live session with an experienced interviewer persona. The interviewer does not validate vague answers, does not hint unprompted, and will push back directly when something is technically wrong or underspecified. This mode replicates real interview conditions more closely than any static guide. It is intentionally uncomfortable. Use it after your solo attempt, not instead of it — the goal is to stress-test a design you already believe in, not to be walked through one you have not attempted.

---

## Problem Bank

Problems are tiered by the depth of tradeoff reasoning they require. Tier 1 problems are table-stakes for any senior engineering interview: if these are rough, the rest of the preparation plan should not advance. Tier 2 problems require solid data model thinking, multi-component coordination, and informed storage choices. Tier 3 problems require all of the above plus genuine treatment of failure at scale, operational complexity awareness, and the ability to reason about consistency models and distributed systems tradeoffs under time pressure.

| Problem | Tier | Key Concepts | Known Asked At |
|---------|------|-------------|----------------|
| URL Shortener | 1 | Hashing, KV store, redirect | Meta, Google, Microsoft |
| Rate Limiter | 1 | Token bucket, sliding window, Redis | Stripe, Uber, Amazon |
| Notification System | 1 | Fan-out, message queues, APNs/FCM | Meta, Apple, Airbnb |
| Design Twitter Feed | 2 | Fan-out on write/read, caching, ranking | Twitter/X, Meta, LinkedIn |
| Design YouTube | 2 | Video transcoding, CDN, blob storage | Google, Netflix, TikTok |
| Web Crawler | 2 | BFS/DFS, dedup, distributed queue | Google, Microsoft, Amazon |
| Design Typeahead/Autocomplete | 2 | Trie, prefix search, caching | Google, Amazon, Uber |
| Design Uber/Lyft | 3 | Geospatial indexing, matching, real-time | Uber, Lyft, DoorDash |
| Design WhatsApp/Messenger | 3 | WebSocket, fan-out, message ordering | Meta, Apple, Discord |
| Design Google Maps | 3 | Graph routing, tile serving, geospatial | Google, Apple, Lyft |
| Design TikTok/Reels | 3 | Recommendation ML, CDN, video pipeline | TikTok, Meta, YouTube |
| Design Distributed Cache | 2 | Consistent hashing, eviction, replication | Redis, Amazon, Cloudflare |
| Design Payment System | 3 | Idempotency, saga, double-entry ledger | Stripe, PayPal, Coinbase |
| Design Search Engine | 3 | Inverted index, crawling, ranking | Google, Elasticsearch, Amazon |
| Design Live Streaming | 3 | HLS/RTMP, CDN, latency vs quality | Twitch, YouTube, Amazon |

---

## How to Use

Solo practice works best when you treat it as a real constraint. Set a 45-minute timer, open a blank file or get out paper, and attempt the entire design without referring to notes or guides. The attempt does not need to be correct — it needs to be honest. The goal of the first pass is to surface the gaps in your own reasoning: the places where you glossed over a tradeoff, where you named a component without knowing why it belongs there, where you skipped capacity estimation because it felt like overhead. Write those gaps down when you debrief. They are the data that drives the next week of practice.

AI Interview Mode should be invoked with `/interview` inside Claude Code after your solo attempt. The session opens with requirements clarification and follows the same three-phase structure as a real interview: high-level design, low-level design, and wrap-up. The interviewer will not validate hand-wavy answers, will call out technically wrong statements directly, and will probe for tradeoffs after every design decision. Come with a design you have already thought through. The value of this mode is the pushback — if the session feels easy, you are not being pushed hard enough. Ask the interviewer to challenge your weakest component explicitly.

After completing both a solo attempt and an AI interview session on a problem, read the SOLUTION_GUIDE in detail. Pay particular attention to the sections that diverged from your design. Reading with a specific gap in mind is far more effective than reading passively. For each divergence, ask whether it was a knowledge gap (you did not know the component or technique) or a judgment gap (you knew it but did not reach for it at the right moment). Knowledge gaps are closed by reading and exposure. Judgment gaps are closed only by more attempts and deliberate debriefs — they require repeated activation, not new information.

After scoring any session, review your lowest-scoring dimensions and open GAP_REMEDIATION.md. Each dimension in the rubric has a corresponding drill: if your data model scores are consistently low, there are targeted schema design exercises that isolate that component. If failure modes score low, there are drills that force you to enumerate failure scenarios before any solution components appear. Work the specific drill for your two weakest dimensions before your next attempt. Do not try to improve everything at once — compounding two focused improvements per cycle adds up faster than spreading effort thinly.

---

## Scoring

Each interview attempt is scored across seven dimensions. Scores are 0–3 per dimension, weighted as follows:

| Dimension | Weight | Max Points |
|-----------|--------|------------|
| Requirements Scoping | 10% | 3 |
| Capacity Estimation | 15% | 3 |
| API Design | 10% | 3 |
| Data Model & Storage | 15% | 3 |
| High-Level Architecture | 20% | 3 |
| Deep Dive & Tradeoffs | 20% | 3 |
| Failure Modes & Scaling | 10% | 3 |
| **Total** | **100%** | **21** |

Interview-ready threshold: 15/21 (weighted ~71%). Consistent 18+ is senior-level performance.

---

## Practice Progression

Week one should be spent entirely on Tier 1 problems, and the goal for that week is narrow: requirements scoping and capacity estimation, nothing else. That sounds limited, but most engineers underestimate how much discipline those two phases require. A requirements clarification that fails to surface delivery semantics, offline behavior, or ordering guarantees will produce a high-level design that solves the wrong problem. A capacity estimation skipped entirely leaves you without intuition for when a single database is sufficient, when you need sharding, and when replication starts mattering. Do all three Tier 1 problems at least once during week one, debrief against the rubric for only those two dimensions, and resist the temptation to evaluate anything else.

In week two, revisit the Tier 1 problems with fresh attempts before moving to your first Tier 2 attempts. The revisit is deliberate: you want to verify that the scoping and estimation improvements from week one have stuck before layering in the next focus area, which is data model and storage. For each problem, push yourself to specify the schema before you name the database, and to justify the database choice against the access patterns the schema implies. Begin the first two Tier 2 problems (Twitter Feed and YouTube are good starting points) as solo attempts; do not evaluate them against the full rubric yet — just observe where your reasoning breaks down under more complexity.

Week three is the full Tier 2 set evaluated against all seven rubric dimensions. Every attempt should be followed by a scoring pass across the complete rubric, and every problem should end with at least one `/interview` session to stress-test the design under pushback. By the end of week three, you should have scores across multiple attempts for all seven dimensions. Patterns will emerge: consistent low scores in one or two areas tell you where your engineering judgment has real gaps that solo study has not closed. Note these patterns explicitly — they drive week four.

Week four is Tier 3 problems plus one live mock interview session per day. Tier 3 problems will expose weaknesses that Tier 2 obscured, particularly around consistency models, operational complexity, and tradeoffs that have no clean answer. After each session, compare your score against your week-three baseline. Identify the two dimensions where your scores are lowest or most volatile, and do the corresponding drills from GAP_REMEDIATION.md before the next attempt. Do not try to run all seven drills — two focused dimensions per cycle compounds faster than diluted effort across everything.

---

## See Also

- [PHILOSOPHY.md](PHILOSOPHY.md)
- [EVALUATION_RUBRIC.md](EVALUATION_RUBRIC.md)
- [GAP_REMEDIATION.md](GAP_REMEDIATION.md)
