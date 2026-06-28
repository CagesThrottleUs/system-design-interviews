# Proposed Solution — [Problem Name]

Fill in every section during and immediately after your session. Score yourself on the final page **before** opening SOLUTION_GUIDE.md. Honest self-scoring is the only way this exercise builds calibration.

---

## Header

| Field | Value |
|-------|-------|
| Date | |
| Problem | |
| Mode | Solo / Partner / AI |
| Time Spent | min |
| Hints Used | (each hint costs 0.5 points from Deep Dive score) |

---

## Requirements Captured

### Functional — Core MVP

What are the 3–5 things this system must do to be considered working?

1.
2.
3.
4.
5.

### Functional — Secondary (V2)

What are the real-but-deferrable requirements you identified?

1.
2.
3.

### Non-Functional Requirements

| Dimension | Your Estimate or Requirement |
|-----------|------------------------------|
| DAU | |
| Peak Write QPS | |
| Peak Read QPS | |
| Latency SLA (p99) | |
| Availability target | |
| Consistency model | |
| Data retention | |

### Explicitly Out of Scope

What did you call out as out of scope for this session?

1.
2.
3.

### Clarifying Assumptions Made

What did you assume rather than confirm? List every assumption — assumptions you didn't name during the session are not credit.

1.
2.
3.

---

## Capacity Estimates

Show your full derivation chain. Do not just write the final number.

**DAU:** _______________

**Notifications per user per day:** _______________

**Total notifications per day:** _______________  
*Calculation: DAU × per-user rate = ___*

**Average write QPS:** _______________  
*Calculation: total/day ÷ 86,400 = ___*

**Peak write QPS (burst multiplier applied):** _______________  
*Calculation: avg QPS × burst factor = ___*

**Devices per user:** _______________

**Per-device delivery tasks per second (avg / peak):** _______________  
*Calculation: ___*

**Storage — payload (per day):** _______________  
*Calculation: total notifications/day × avg payload size = ___*

**Storage — payload (retention period):** _______________  
*Calculation: per-day × retention days = ___*

**Storage — delivery status records (per day):** _______________  
*Calculation: per-device delivery tasks/day × record size = ___*

**Storage — delivery status (retention period):** _______________  
*Calculation: per-day × retention days = ___*

**Bandwidth estimate:** _______________  
*Calculation: peak delivery tasks/sec × avg payload size = ___*

**Bottleneck identified:**  
Which component in your design handles the highest throughput, and what is that throughput?

*Answer:*

---

## API Design

Write your API endpoints here. Use REST format: METHOD /path. Include request body fields and key response fields. Mark any endpoint that needs an idempotency mechanism.

### Endpoint 1

```
METHOD /path

Request:
{
}

Response (success):
{
}

Response (error):
{
}

Idempotency: Y / N — [if Y, describe mechanism]
```

### Endpoint 2

```
METHOD /path

Request:
{
}

Response (success):
{
}

Response (error):
{
}

Idempotency: Y / N — [if Y, describe mechanism]
```

### Endpoint 3

```
METHOD /path

Request:
{
}

Response (success):
{
}

Idempotency: Y / N — [if Y, describe mechanism]
```

---

## Data Model

Write your schema here. For SQL, use CREATE TABLE statements with column types and constraints. For NoSQL, describe the document or row structure with key design. Always name your indexes and explain why each exists.

### [Table / Collection 1 Name]

```sql
-- describe the table and its role in one line above the DDL
```

Index rationale:

### [Table / Collection 2 Name]

```sql
-- describe the table and its role in one line above the DDL
```

Index rationale:

### [Table / Collection 3 Name]

```sql
-- describe the table and its role in one line above the DDL
```

Index rationale:

**Database technology choices:**

| Store | Technology Chosen | Primary Access Pattern | Why This Technology |
|-------|------------------|----------------------|-------------------|
| | | | |
| | | | |
| | | | |

---

## High-Level Architecture

Draw your architecture diagram here as ASCII art, or describe the components and connections in text if drawing is not possible in your current mode. Label every arrow with a protocol or data type.

```
[Your architecture diagram here]
```

**Component inventory** (fill in after drawing):

| Component | Type | Responsibility |
|-----------|------|----------------|
| | | |
| | | |
| | | |
| | | |
| | | |

---

## Deep Dive: [Component You Chose]

**Why did you choose this component for the deep dive?**  
What makes it the hardest part of the system, and why is getting it wrong more costly than getting other components wrong?

*Answer:*

**What is the specific hard problem within this component?**  
Not a generic description of the component — the specific challenge that doesn't exist at small scale or that has counterintuitive solutions.

*Answer:*

**Your solution:**  
Be specific. Name the data structure, protocol, algorithm, or topology. Vague answers ("use a cache") do not count.

*Answer:*

**Tradeoff in your solution:**  
What does your solution give up? Every engineering choice has a cost. Name it.

*Answer:*

**Alternative considered:**  
Name one alternative approach you evaluated.

*Alternative:*

**Why rejected:**  
What specific property of the alternative made it worse for this problem? One sentence, be specific.

*Answer:*

---

## Failure Modes Discussed

| Failure Scenario | Impact on Users | Detection Method | Mitigation | Residual Risk |
|-----------------|-----------------|-----------------|------------|---------------|
| | | | | |
| | | | | |
| | | | | |

**Single Point of Failure (SPOF) identified:**  
Which component in your design has no redundancy, and what is the user-facing impact if it fails?

*Answer:*

**Cascade failure scenario:**  
Describe a scenario where one component failing causes a second component to fail.

*Answer:*

---

## Extension Points

What 2–3 ways does your design naturally extend if the requirements expand?

1.

2.

3.

---

## Post-Session Self-Score

**Fill in BEFORE reading SOLUTION_GUIDE.md.** Score based on what you actually said and drew during the session, not what you wish you had said.

| Dimension | Weight | Score (0–3) | Evidence — what specifically did I say/draw? | Anti-patterns I hit |
|-----------|--------|-------------|---------------------------------------------|-------------------|
| Requirements Clarification | 10% | | | |
| Capacity Estimation | 10% | | | |
| HLD Completeness | 20% | | | |
| API Design | 10% | | | |
| Data Model | 15% | | | |
| Deep Dive Quality | 25% | | | |
| Failure Modes | 10% | | | |
| **Total** | **100%** | **/ 21** | | |

**Interview Readiness: ___ %** (score / 21 × 100)

**Hint deduction:** ___ hints × 0.5 = ___ points deducted  
**Adjusted score:** ___ / 21

---

## Post-Solution-Guide Reflection

*Fill in AFTER reading SOLUTION_GUIDE.md.*

**Biggest gap between my design and the solution:**


**Thing I got right that I wasn't sure about:**


**Three things I didn't know before this session:**

1.
2.
3.

**What I'll focus on in the next session:**


---

## AI Evaluation Output

*Leave blank — Claude fills this section after an `/interview` session.*

**Session ID:**  
**Evaluated at:**  
**Interviewer notes:**  

| Dimension | Score Awarded | Key Evidence Observed | Top Anti-Pattern Noted |
|-----------|--------------|----------------------|----------------------|
| Requirements Clarification | | | |
| Capacity Estimation | | | |
| HLD Completeness | | | |
| API Design | | | |
| Data Model | | | |
| Deep Dive Quality | | | |
| Failure Modes | | | |
| **Total** | | | |

**Hints used:** ___  
**Adjusted total:** ___ / 21  
**Readiness band:** Not Ready / Approaching Ready / Ready / Strong  

**Top gap identified:**  
**Recommended drill:**  
**Narrative feedback:**  
