# Session Scorecard

Fill in this document during and immediately after every practice session. Complete the self-score **before** reading SOLUTION_GUIDE.md.

---

## Session Header

| Field | Value |
|-------|-------|
| Date | |
| Problem | |
| Mode | Solo / Partner / AI / Deep Dive |
| Time Spent | min |
| Partner (if applicable) | |

---

## Dimension Scores

Score each dimension 0–3 using the rubric definitions below each table.

---

### Dimension 1 — Requirements Clarification (Weight: 10%)

| Field | Your Entry |
|-------|-----------|
| Score | ___ / 3 |
| Evidence — what specifically did I ask or clarify? | |
| Did I use the 5-category framework? (Scale / Delivery / Latency / Ordering / Compliance) | Y / N |
| Anti-pattern: Jumped to design before asking any clarifying questions | Y / N |
| Anti-pattern: Asked only about functional requirements, skipped NFRs | Y / N |
| Anti-pattern: Took requirements as given without probing edge cases | Y / N |

**Rubric:** 0 = Jumped straight to design with no clarification. 1 = Asked 1–2 surface questions. 2 = Covered scale and at least one of delivery/latency/consistency. 3 = Covered all 5 categories or demonstrated clear judgment about which to prioritize given the problem.

---

### Dimension 2 — Capacity Estimation (Weight: 10%)

| Field | Your Entry |
|-------|-----------|
| Score | ___ / 3 |
| Evidence — what estimates did I produce? | |
| Did I derive a bottleneck from the math? | Y / N |
| Anti-pattern: Skipped capacity estimation entirely | Y / N |
| Anti-pattern: Stated numbers without showing the derivation chain | Y / N |
| Anti-pattern: Did not use the capacity math to inform any architectural choice | Y / N |

**Rubric:** 0 = No estimation attempted. 1 = Stated some numbers without derivation. 2 = Derived QPS and storage with reasonable chain; identified which tier is under most load. 3 = Full derivation chain, identified bottleneck component, used math to justify at least one architectural choice.

---

### Dimension 3 — High-Level Design Completeness (Weight: 20%)

| Field | Your Entry |
|-------|-----------|
| Score | ___ / 3 |
| Evidence — which components did I name and how did I describe their responsibilities? | |
| Did the design cover end-to-end data flow? | Y / N |
| Anti-pattern: Named components without explaining responsibilities | Y / N |
| Anti-pattern: Missing a critical component (e.g., message queue, API gateway) | Y / N |
| Anti-pattern: Went into LLD detail during HLD phase | Y / N |
| Anti-pattern: Used SQL for everything without justification | Y / N |

**Rubric:** 0 = Fewer than 4 components named; no data flow. 1 = Main components present but missing 1–2 critical pieces or no protocol labels. 2 = Complete component list with labeled data flows and clear responsibilities. 3 = Complete design with protocol choices justified, data flow end-to-end, and tradeoffs at the component level noted.

---

### Dimension 4 — API Design (Weight: 10%)

| Field | Your Entry |
|-------|-----------|
| Score | ___ / 3 |
| Evidence — which endpoints did I define, and what did I specify? | |
| Did I include an idempotency mechanism? | Y / N |
| Anti-pattern: No API design produced | Y / N |
| Anti-pattern: Endpoints defined without request/response schemas | Y / N |
| Anti-pattern: No idempotency consideration for write operations | Y / N |
| Anti-pattern: Used GET for state-changing operations | Y / N |

**Rubric:** 0 = No API design. 1 = Endpoint names only, no schemas. 2 = Endpoints with request/response shape defined. 3 = Full API design with idempotency keys, pagination, error codes, and versioning consideration.

---

### Dimension 5 — Data Model (Weight: 15%)

| Field | Your Entry |
|-------|-----------|
| Score | ___ / 3 |
| Evidence — what schema did I produce, and what indexes did I define? | |
| Did I explain the rationale for my database choice? | Y / N |
| Anti-pattern: No schema or data model produced | Y / N |
| Anti-pattern: Schema defined without any indexes | Y / N |
| Anti-pattern: Wrong database type for the access pattern | Y / N |
| Anti-pattern: No foreign key or partition key rationale | Y / N |

**Rubric:** 0 = No data model. 1 = Rough schema with field names only. 2 = Schema with primary keys, foreign keys or partition keys, and at least one index justified. 3 = Full schema with index strategy, database choice justified against access patterns, and sharding or partitioning strategy noted.

---

### Dimension 6 — Deep Dive Quality (Weight: 25%)

| Field | Your Entry |
|-------|-----------|
| Score | ___ / 3 |
| Evidence — which component did I deep dive, and what specific design decisions did I make? | |
| Did I name an alternative and explain why I rejected it? | Y / N |
| Anti-pattern: Chose the wrong component for the deep dive (not the hardest part) | Y / N |
| Anti-pattern: Described the component at HLD level instead of going deep | Y / N |
| Anti-pattern: Could not name alternatives to chosen approach | Y / N |
| Anti-pattern: No failure scenario surfaced during deep dive | Y / N |

**Rubric:** 0 = No deep dive or deep dive indistinguishable from HLD. 1 = Went deeper on one component but didn't surface tradeoffs. 2 = Specific design decisions with one alternative named and rejected. 3 = Chose the highest-leverage component, surfaced the hard problem within it, proposed a solution with specific tradeoffs named, and named at least two alternatives with explicit rejection reasoning.

---

### Dimension 7 — Failure Modes & Operational Thinking (Weight: 10%)

| Field | Your Entry |
|-------|-----------|
| Score | ___ / 3 |
| Evidence — which failure modes did I name and what mitigations did I propose? | |
| Did I identify the SPOF in my design? | Y / N |
| Anti-pattern: No failure modes discussed | Y / N |
| Anti-pattern: Mentioned retries without specifying retry strategy | Y / N |
| Anti-pattern: No cascade failure scenario considered | Y / N |

**Rubric:** 0 = No failure modes mentioned. 1 = Named one failure mode without mitigation. 2 = 2–3 failure modes with concrete mitigations. 3 = Identified SPOF, named cascade failure scenario, proposed monitoring/alerting approach, and discussed residual risk.

---

## Score Summary

| Dimension | Weight | Score (0–3) | Weighted |
|-----------|--------|-------------|---------|
| Requirements Clarification | 10% | | |
| Capacity Estimation | 10% | | |
| HLD Completeness | 20% | | |
| API Design | 10% | | |
| Data Model | 15% | | |
| Deep Dive Quality | 25% | | |
| Failure Modes | 10% | | |
| **Total** | **100%** | **___ / 21** | |

**Interview Readiness: ___ %** (Total / 21 × 100)

| Band | Score Range | Meaning |
|------|------------|---------|
| Not Ready | 0–9 | Significant gaps; use learning mode |
| Approaching Ready | 10–14 | Practice more; address top gaps |
| Ready | 15–17 | Schedule the interview |
| Strong | 18–21 | You are likely to perform well |

---

## Post-Session Reflection

**Top gap identified this session:**


**Recommended drill from GAP_REMEDIATION.md:**


**One thing I will do differently next session:**


**AI Evaluation Notes** (filled by Claude after `/interview` session — leave blank for solo/partner sessions):


---

## Version

Scorecard v1.0 — matches EVALUATION_RUBRIC.md v1.0
