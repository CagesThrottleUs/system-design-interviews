# Progress Tracker

This document is the single source of truth for your mock interview preparation. Update it after every session. Never retroactively inflate scores.

---

## 1. Session Log

| # | Date | Problem | Mode | Score (/21) | Readiness % | Top Gap | Notes |
|---|------|---------|------|-------------|-------------|---------|-------|
| 1 | — | — | — | — | — | — | — |
| 2 | — | — | — | — | — | — | — |
| 3 | — | — | — | — | — | — | — |
| 4 | | | | | | | |
| 5 | | | | | | | |
| 6 | | | | | | | |
| 7 | | | | | | | |
| 8 | | | | | | | |
| 9 | | | | | | | |
| 10 | | | | | | | |

---

## 2. Dimension Progress

Track your rolling average across all sessions. Update after each session by recalculating the mean.

| Dimension | Weight | Sessions Tracked | Avg Score (/3) | Trend | Target |
|-----------|--------|-----------------|----------------|-------|--------|
| Requirements Clarification | 10% | — | — | — | 2.5 |
| Capacity Estimation | 10% | — | — | — | 2.5 |
| HLD Completeness | 20% | — | — | — | 2.5 |
| API Design | 10% | — | — | — | 2.0 |
| Data Model | 15% | — | — | — | 2.0 |
| Deep Dive Quality | 25% | — | — | — | 2.5 |
| Failure Modes | 10% | — | — | — | 2.0 |

**Trend key:** ↑ improving over last 3 sessions | → stable | ↓ declining | — insufficient data

---

## 3. Problem Completion Status

| # | Problem | Tier | Category | Attempted | Best Score (/21) | Last Attempted | Notes |
|---|---------|------|----------|-----------|-----------------|----------------|-------|
| 1 | Notification System | 1 | Messaging / Async | N | — | — | Gate 1 unlock |
| 2 | URL Shortener | 1 | Storage / Hashing | N | — | — | Gate 2 |
| 3 | Rate Limiter | 1 | Middleware / Algorithms | N | — | — | Gate 2 |
| 4 | News Feed | 2 | Social / Fan-out | N | — | — | Gate 3 |
| 5 | Web Crawler | 2 | Distributed / Crawling | N | — | — | Gate 3 |
| 6 | Chat System | 2 | Real-time / WebSocket | N | — | — | Gate 3 |
| 7 | Search Autocomplete | 2 | Search / Indexing | N | — | — | Gate 3 |
| 8 | Object Storage (S3-like) | 2 | Storage / Distributed FS | N | — | — | Gate 4 |
| 9 | Video Upload & Streaming | 2 | Media / CDN | N | — | — | Gate 4 |
| 10 | Distributed Job Scheduler | 3 | Scheduling / Reliability | N | — | — | Gate 4 |
| 11 | Ad Click Aggregator | 3 | Analytics / Streaming | N | — | — | Gate 5 |
| 12 | Key-Value Store | 3 | Storage / Consistency | N | — | — | Gate 5 |
| 13 | Payment Processing System | 3 | Finance / Exactly-Once | N | — | — | Gate 5 |
| 14 | Live Location Tracking | 3 | Real-time / Geo | N | — | — | Gate 6 |
| 15 | Distributed Tracing System | 3 | Observability / Sampling | N | — | — | Gate 6 |

**Tier definitions:** 1 = foundational, expected at every level | 2 = demonstrates breadth | 3 = differentiates senior candidates

---

## 4. Anti-Pattern Recurrence Tracker

Track every time you hit one of these patterns. When you've gone three consecutive sessions without hitting a pattern, mark it Resolved.

| Anti-Pattern | Times Seen | Last Seen (Session #) | Consecutive Sessions Clean | Resolved? |
|-------------|-----------|----------------------|--------------------------|-----------|
| Skipped capacity estimation entirely | 0 | — | 0 | N |
| Used SQL for everything without justification | 0 | — | 0 | N |
| No failure modes discussed | 0 | — | 0 | N |
| API design missing idempotency | 0 | — | 0 | N |
| Chose wrong deep dive component | 0 | — | 0 | N |
| Cannot name alternatives to chosen approach | 0 | — | 0 | N |
| Requirements not clarified before drawing | 0 | — | 0 | N |
| Went into LLD detail during HLD phase | 0 | — | 0 | N |
| Presented capacity numbers without derivation | 0 | — | 0 | N |
| No end-to-end data flow in HLD | 0 | — | 0 | N |

---

## 5. Interview Readiness Gate

Complete this checklist before scheduling your target interview. All items must be checked.

**Quantitative Gates**

- [ ] Scored 15+/21 on at least 3 different problems
- [ ] Attempted all 3 Tier 1 problems
- [ ] Attempted at least 3 Tier 2 problems
- [ ] No dimension averaging below 1.5/3 across all sessions
- [ ] Can complete capacity estimation in under 5 minutes (self-reported from last 3 sessions)

**Qualitative Gates**

- [ ] Anti-pattern recurrence tracker shows at least 4 patterns marked Resolved
- [ ] Have attempted the same problem twice (second attempt with ≥3 point improvement)
- [ ] Completed at least one AI interview session
- [ ] Can articulate the tradeoff between strong and eventual consistency without prompting
- [ ] Can name and explain CAP theorem implications for at least 3 different problem types

**Readiness Summary**

Gates checked: ___ / 10

Status: NOT READY / APPROACHING / READY (circle when applicable)

Target interview date: _______________
