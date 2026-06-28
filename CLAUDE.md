# CLAUDE.md — HLD Interview Practice Harness

Context for AI agents operating in this repository.

---

## Repository Purpose

Problem-solving-based learning system for High-Level Design (HLD) / system design interviews. Two goals that reinforce each other:

1. **Interview prep** — score 15+/21 on the evaluation rubric consistently
2. **Engineering growth** — develop intuition for system shapes, trade-offs, and failure modes that transfers to real engineering work

The harness is built around the principle that **you learn systems by solving problems, not by studying patterns**. Every pattern reference, every rubric dimension, every drill exists to support active problem-solving — not passive reading.

AI agents operate in one of six modes: Mock Interviewer, Evaluator, Solution Explainer, Gap Coach, Problem Creator, or Deep Dive Teacher. See `AGENTS.md` for full workflow definitions.

---

## Directory Structure

```
README.md                    — philosophy, 15-problem bank, usage modes
PHILOSOPHY.md                — 5 mental models, engineering growth mindset
EVALUATION_RUBRIC.md         — 7-dimension scoring system (authoritative)
GAP_REMEDIATION.md           — targeted drills per dimension gap
CLAUDE.md                    — this file (AI agent context)
AGENTS.md                    — agent role definitions and 6 workflows

problems/
  _template/                 — blank template for new problems (5 files)
  NN-problem-name/
    PROBLEM.md               — problem statement (share with candidate)
    MOCK_INTERVIEW.md        — interviewer script (do NOT share)
    PROPOSED_SOLUTION.md     — candidate's design (gitignored; AI fills during session)
    SOLUTION_GUIDE.md        — reference solution (reveal after attempt)
    LESSONS_LEARNED.md       — real interview stories + general pattern mistakes

mock-interview/
  HOW_TO_RUN.md              — all 4 session modes
  SESSION_SCORECARD.md       — fill-in scorecard template

patterns/
  CONSISTENT_HASHING.md
  CACHING_STRATEGIES.md
  FAN_OUT_PATTERNS.md
  SAGA_PATTERN.md
  RATE_LIMITING_ALGORITHMS.md
  GEOSPATIAL_INDEXING.md
  DATABASE_SHARDING.md
  MESSAGE_QUEUE_PATTERNS.md

progress/
  TRACKER.md                 — session log and dimension trend tracker
```

---

## The PROPOSED_SOLUTION.md Workflow

This file is the persistent record of what a candidate actually designed. It is gitignored — it lives only on the candidate's machine and changes with every attempt.

| Phase | Who acts | What happens |
|-------|----------|-------------|
| During mock interview | AI Interviewer | Fills PROPOSED_SOLUTION.md incrementally per phase |
| Solo practice | Candidate | Fills PROPOSED_SOLUTION.md manually |
| Post-attempt | AI Evaluator | Reads PROPOSED_SOLUTION.md; scores against rubric; writes evaluation into the file |
| Solution reveal | AI Solution Explainer | Compares PROPOSED_SOLUTION.md with SOLUTION_GUIDE.md; explains diffs |
| Deep dive | AI Teacher | References PROPOSED_SOLUTION.md to show what the candidate did and why the design works |

**Critical rule:** Never reveal SOLUTION_GUIDE.md until PROPOSED_SOLUTION.md has been filled.

---

## Evaluation System

7 dimensions, each scored 0–3. Source of truth: `EVALUATION_RUBRIC.md`.

| # | Dimension | What it measures |
|---|-----------|-----------------|
| 1 | Requirements Scoping | Did candidate define functional + NFRs, out-of-scope, and explicit assumptions? |
| 2 | Capacity Estimation | Did candidate derive QPS/storage/bandwidth from DAU? Does estimation drive design? |
| 3 | API Design | Correct resource modeling, HTTP semantics, auth, pagination, idempotency? |
| 4 | Data Model & Storage | Right DB type with justification, schema, indexing aligned to access patterns? |
| 5 | High-Level Architecture | Correct components, data flow clear, no unnecessary SPOFs? |
| 6 | Deep Dive & Tradeoffs | Picked the right 1-2 components, named alternatives explicitly, stated trade-offs? |
| 7 | Failure Modes & Scaling | Identified what breaks first, described graceful degradation, explained scaling path? |

Max: 21. Interview-ready threshold: 15+.

---

## Problem Bank (15 problems)

| # | Problem | Tier | Key Concepts |
|---|---------|------|-------------|
| 01 | URL Shortener | Foundation | Hashing, read-heavy caching, 301 vs 302 |
| 02 | Rate Limiter | Foundation | Token bucket, sliding window, distributed counters |
| 03 | Notification System | Foundation–Intermediate | Fan-out, APNs/FCM, at-least-once delivery |
| 04 | Social Feed | Intermediate | Fan-out on write vs read, celebrity problem |
| 05 | Video Streaming | Intermediate | Transcoding pipeline, ABR/DASH, CDN |
| 06 | Search Autocomplete | Intermediate | Trie + top-K precompute, async update pipeline |
| 07 | Chat System | Intermediate | WebSocket routing, message ordering, group fan-out |
| 08 | Ride Sharing | Intermediate | Geospatial indexing, driver location at scale |
| 09 | Collaborative Editor | Advanced | OT vs CRDT, immutable character IDs, snapshot/compaction |
| 10 | Distributed Cache | Intermediate | Consistent hashing, LRU, thundering herd |
| 11 | Object Storage | Intermediate | Erasure coding, metadata/data separation, multipart upload |
| 12 | Web Crawler | Intermediate | Two-tier URL frontier, Bloom filter, politeness |
| 13 | Payment System | Advanced | Idempotency, double-entry ledger, SAGA, transactional outbox |
| 14 | Booking System | Intermediate–Advanced | Inventory locking, hold-with-TTL, overbooking |
| 15 | Ad Click Aggregation | Advanced | Dual pipeline, HyperLogLog, hot partition, late events |

---

## Agent Behavioral Rules

1. Fill `PROPOSED_SOLUTION.md` incrementally during mock interview — not after
2. Never reveal `SOLUTION_GUIDE.md` until `PROPOSED_SOLUTION.md` has content
3. `MOCK_INTERVIEW.md` is for the AI interviewer only — never share with candidate
4. Scoring always uses `EVALUATION_RUBRIC.md` — do not invent criteria
5. Write AI evaluation output into the **AI Evaluation Output** section of `PROPOSED_SOLUTION.md`
6. Remediation always links to `GAP_REMEDIATION.md` sections — not ad-hoc advice
7. Update `progress/TRACKER.md` after every evaluated session
8. After Solution Explainer, offer Deep Learning Mode — say "Type 'deep dive [pattern/concept]' to understand why"
9. Always prefer web search before writing problem statements or adding new problems
10. Real numbers must cite company + approximate year — never invent statistics

---

## Adding New Problems

1. Copy `problems/_template/` to `problems/NN-problem-name/`
2. Run web searches for: companies that ask this, real engineering numbers, common mistakes
3. Fill in all 5 files using real data from step 2
4. Add to README.md problem bank table
5. Commit: `docs(problems): add [name] — [key concept] problem`

**Quality bar per problem:**
- 5+ functional requirements with NFR table
- Capacity estimation hints (DAU, ratios, sizes)
- 8+ clarifying questions grouped by category
- 3 explicit Key Design Decisions (choice / alternative / why / trade-off)
- 3+ Real Interview Stories in LESSONS_LEARNED.md
- 4+ General Pattern Mistakes in LESSONS_LEARNED.md
- Full interviewer script in MOCK_INTERVIEW.md with per-phase hints

---

## HLD Patterns — Quick Lookup

Full details in `patterns/`. These are the most commonly needed in HLD interviews:

| Pattern | Trigger signal | Core mechanism |
|---------|---------------|----------------|
| Consistent Hashing | "Add/remove cache/DB nodes without resharding all keys" | Hash ring with virtual nodes |
| Cache-Aside (Lazy) | "DB is read-heavy; reduce DB load" | App checks cache first, populates on miss |
| Write-Through | "Need strong read-after-write consistency" | Write to cache and DB synchronously |
| Fan-out on Write | "Users need fast feed reads; write volume manageable" | Pre-materialize feed on post creation |
| Fan-out on Read | "Celebrity accounts; write amplification unacceptable" | Merge at read time from following list |
| Token Bucket | "Allow bursts; smooth average rate" | Tokens replenish at rate R, bucket cap B |
| Sliding Window | "Strict no-burst rate limit" | Count requests in rolling window |
| SAGA | "Distributed transaction across services" | Compensating transactions, no 2PC |
| Geohash | "Find entities near a location" | Encode lat/lng as prefix-comparable string |
| Database Sharding | "Single DB can't handle write throughput" | Partition by hash/range key |
