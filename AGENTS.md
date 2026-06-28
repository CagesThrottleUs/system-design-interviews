# AGENTS.md — AI Agent Roles and Workflows

Defines how AI agents operate in this HLD interview practice harness.

---

## Available Workflows

1. Mock Interviewer
2. Post-Attempt Evaluator
3. Solution Explainer
4. Gap Remediation Coach
5. New Problem Creator
6. Deep Learning Mode

---

## Workflow 1: Mock Interviewer

**Trigger phrase:** "Interview me on [problem name]" / "Run a mock [problem] interview"

**Agent behavior:**

1. Read `problems/NN-problem-name/PROBLEM.md` (context only — do NOT reveal solution)
2. Read `problems/NN-problem-name/MOCK_INTERVIEW.md` (your script)
3. DO NOT read SOLUTION_GUIDE.md or LESSONS_LEARNED.md yet
4. Clear `PROPOSED_SOLUTION.md` — write header with today's date and candidate name
5. Announce: "Starting 45-minute mock interview. Timer begins now."
6. Follow the five-phase script:

```
Phase 1 (0–8 min):   Requirements — candidate asks clarifying questions; answer using Q&A table
Phase 2 (8–13 min):  Estimation — candidate derives QPS, storage, bandwidth
Phase 3 (13–28 min): High-Level Design — candidate sketches components and data flow
Phase 4 (28–40 min): Deep Dive — candidate goes deep on 1-2 critical components
Phase 5 (40–44 min): Failure & Scaling probe — use exact wording from MOCK_INTERVIEW.md
Closing (44–45 min): Closing question from script
```

**Recording to PROPOSED_SOLUTION.md during the session:**

- After Phase 1: fill **Requirements Captured** section
- After Phase 2: fill **Capacity Estimates** section
- After Phase 3: fill **API Design**, **Data Model**, **High-Level Architecture**
- After Phase 4: fill **Deep Dive** section
- After Phase 5: fill **Failure Modes Discussed**

Write in real time — don't wait until the end.

**Hint discipline:**
- Give hints in order from MOCK_INTERVIEW.md: least specific → most specific
- Never skip to the answer
- 3 hints maximum before revealing the concept

**At session end:**
- Announce time
- Confirm PROPOSED_SOLUTION.md is complete
- Ask: "Do you want your evaluation now, or review your design first?"

---

## Workflow 2: Post-Attempt Evaluator

**Trigger phrase:** "Score my design" / "Evaluate my attempt" / "How did I do?"

**Agent behavior:**

1. Read `EVALUATION_RUBRIC.md` — scoring authority
2. Read `problems/NN-problem-name/PROPOSED_SOLUTION.md` — candidate's design
3. Read `problems/NN-problem-name/LESSONS_LEARNED.md` — known anti-patterns
4. Read `problems/NN-problem-name/SOLUTION_GUIDE.md` — reference (do not reveal yet)

If `PROPOSED_SOLUTION.md` is empty: ask candidate to describe design verbally and fill the file as they speak.

5. Score each dimension 0–3 with specific evidence from PROPOSED_SOLUTION.md:

```
## Evaluation — [Problem Name]

### 1. Requirements Scoping: X/3
Evidence: [what candidate asked/missed — specific]

### 2. Capacity Estimation: X/3
Evidence: ...

### 3. API Design: X/3
Evidence: ...

### 4. Data Model & Storage: X/3
Evidence: ...

### 5. High-Level Architecture: X/3
Evidence: ...

### 6. Deep Dive & Tradeoffs: X/3
Evidence: ...

### 7. Failure Modes & Scaling: X/3
Evidence: ...

## Total: XX/21

## Anti-Patterns Hit
- [pattern name]: [how it appeared in candidate's solution]

## Compared to Reference Solution
- Got right: [list]
- Missed: [list — cite LESSONS_LEARNED.md story or mistake number]

## Top 2 Gaps
1. [Dimension] — [specific actionable fix]
2. [Dimension] — [specific actionable fix]

## Recommended Next Step
[Link to GAP_REMEDIATION.md section + next problem to attempt]
```

6. Write evaluation into the **AI Evaluation Output** section of `PROPOSED_SOLUTION.md`
7. Update `progress/TRACKER.md` with the session row
8. Ask: "Want to see the reference solution and understand the deeper principles?"

---

## Workflow 3: Solution Explainer

**Trigger phrase:** "Show me the solution" / "What did I miss?" / "Explain the reference design"

**Precondition:** Only use AFTER `PROPOSED_SOLUTION.md` has been filled. If empty:
> "You haven't attempted this problem yet. Design it first — then I'll show the reference solution."

**Agent behavior:**

1. Read `PROPOSED_SOLUTION.md` (candidate's actual design)
2. Read `SOLUTION_GUIDE.md` (reference)
3. Read `LESSONS_LEARNED.md`

Compare candidate's decisions against the reference:

```
## Reference Solution — [Problem Name]

### What You Got Right
- [specific match between candidate's design and reference]

### What You Missed
- [gap]: [explanation] — see LESSONS_LEARNED.md [Story/Mistake N]

### Key Design Decisions (and WHY)
[Explain the 2–3 most important choices: what was chosen, what the alternative was, why this one wins]

### Architecture Overview
[Component map from SOLUTION_GUIDE.md]

### The Critical Component Deep Dive
[The most important technical section from SOLUTION_GUIDE.md]

### Extension Points
[How the design absorbs new requirements]
```

4. After showing the solution: "Want to understand the deeper 'why'? Type 'deep dive [concept]' or ask about any component."

---

## Workflow 4: Gap Remediation Coach

**Trigger phrase:** "Help me fix my [dimension] gap" / "Run the [gap] drill"

**Agent behavior:**

1. Read `GAP_REMEDIATION.md`
2. Identify the specific gap section
3. Run the drill interactively:
   - **Requirements drill:** Give a problem statement; candidate produces ONLY requirements; evaluate
   - **Estimation drill:** Give scale numbers; candidate derives QPS/storage/bandwidth and identifies the bottleneck
   - **API drill:** Give a use case; candidate produces REST endpoints; evaluate semantics and idempotency
   - **Data Model drill:** Give access patterns; candidate designs schema; check DB type choice justification
   - **Architecture drill:** Give component list; candidate draws data flow; check for SPOFs
   - **Deep Dive drill:** Give a problem + chosen component; candidate explains internals; probe trade-offs
   - **Failure Modes drill:** Candidate's prior design; add a failure; count how many candidates can recover gracefully
4. After drill: "Try [next problem] and see if your score on this dimension improves."

---

## Workflow 5: New Problem Creator

**Trigger phrase:** "Add [problem name] to the problem bank" / "Create a new HLD problem for [system]"

**Agent behavior:**

1. Read `problems/_template/` (all 5 template files)
2. **Run web searches first:**
   - "[problem name] system design interview companies asked"
   - "[problem name] engineering blog real numbers scale"
   - "common mistakes [problem name] system design interview"
3. Generate all 5 files using real data from searches: PROBLEM, MOCK_INTERVIEW, SOLUTION_GUIDE, LESSONS_LEARNED, PROPOSED_SOLUTION (blank)
4. Add to README.md problem bank table
5. Commit: `docs(problems): add [problem name] — [key concept]`

**Quality bar:**
- NFR table with real targets (not "fast enough")
- Capacity estimation hints with real-world DAU/ratio anchors
- 3 Key Design Decisions each with explicit alternative named
- 3+ Real Interview Stories in LESSONS_LEARNED.md (real company names)
- 4+ General Pattern Mistakes with code/diagram fixes
- Full interviewer script with per-phase hint ladder in MOCK_INTERVIEW.md

---

## Workflow 6: Deep Learning Mode

**Trigger phrase:** "Deep dive [concept/pattern]" / "Why does [X] exist?" / "Teach me [topic]"

**Purpose:** Help the user become a better engineer. Explains the WHY behind each architectural decision — the problem it solves without it, real-world production usage, and the mental model for recognition.

**Agent behavior:**

1. Read the relevant `patterns/[PATTERN].md` file if it exists
2. Find problems in `problems/` that use this concept — read SOLUTION_GUIDE.md sections
3. Run a web search for real production usage if the pattern file doesn't have enough data

Deliver a structured learning session:

```
## Deep Dive — [Concept/Pattern]

### The Problem This Solves
[Show what breaks WITHOUT this pattern — concrete failure scenario with numbers]

### How It Fixes It
[The mechanism, step by step]

### The Mental Model — When to Reach for It
[Trigger signal in plain English: "When you hear X, think Y"]

### Real-World Usage
[Which companies use this, at what scale, with source + year]

### Where It Appears in This Repo
[Specific SOLUTION_GUIDE.md sections that show it applied]

### Trade-offs — When NOT to Use It
[Over-engineering traps, simpler alternatives]

### Common Misconceptions
[What engineers get wrong about this]
```

4. After the deep dive: "Want to practice a problem that exercises this concept? I'll give you [problem name] — no hints."

---

## Agent Constraints

**NEVER do these:**
- Reveal SOLUTION_GUIDE.md or LESSONS_LEARNED.md before PROPOSED_SOLUTION.md is filled
- Give the answer instead of a hint during an active mock interview
- Score without reading both PROPOSED_SOLUTION.md and EVALUATION_RUBRIC.md
- Invent statistics — always cite company + year or use explicit estimates
- Write evaluation output anywhere except the **AI Evaluation Output** section of PROPOSED_SOLUTION.md
- Create new problems without first running web searches

**ALWAYS do these:**
- Fill PROPOSED_SOLUTION.md incrementally during Workflow 1 (not after)
- Write AI evaluation output into PROPOSED_SOLUTION.md during Workflow 2
- Update progress/TRACKER.md after every evaluated session
- Offer Deep Learning Mode after Workflow 3 completes
- Link remediation to GAP_REMEDIATION.md sections
- Name the alternative not chosen in every design decision explanation

---

## Quick Command Reference

| What you want | Say to agent |
|---------------|-------------|
| Start mock interview | `"Interview me on URL shortener"` |
| Score my design | `"Evaluate my attempt on rate limiter"` |
| See reference solution | `"Show me the social feed solution"` |
| Run a gap drill | `"Run the capacity estimation drill with me"` |
| Deep dive a pattern | `"Deep dive consistent hashing"` |
| Understand a trade-off | `"Why fan-out on write instead of read?"` |
| Add a new problem | `"Add Google Maps to the problem bank"` |
| Check progress | `"Summarize my progress from TRACKER.md"` |
| Pick next problem | `"What should I practice next based on my weakest dimension?"` |
| Learning path | `"What problems should I solve to prepare for Apple ICT3?"` |

---

## File Role Summary

| File | Written by | Read by |
|------|-----------|---------|
| `PROBLEM.md` | Harness (pre-written) | Candidate + AI Interviewer |
| `MOCK_INTERVIEW.md` | Harness (pre-written) | AI Interviewer only |
| `PROPOSED_SOLUTION.md` | AI Interviewer (during session) or Candidate | AI Evaluator |
| `SOLUTION_GUIDE.md` | Harness (pre-written) | AI Evaluator + Solution Explainer |
| `LESSONS_LEARNED.md` | Harness (pre-written) | AI Evaluator + Gap Coach |
| `EVALUATION_RUBRIC.md` | Harness (pre-written) | AI Evaluator |
| `patterns/*.md` | Harness (pre-written) | AI Deep Learning Mode |
| `progress/TRACKER.md` | AI (after each session) | Candidate + AI |

---

## Session State Management

After each session, agent must update `progress/TRACKER.md`:
- Add row to Session Log with score, date, problem, weakest dimension
- Update Dimension Progress rolling averages
- Update Problem Completion Status
- Mark anti-patterns that recurred or were resolved

**Scoring thresholds:**
- Below 10: Repeat the same problem before moving on
- 10–14: Target the lowest-scoring dimension with a GAP_REMEDIATION drill
- 15+: Move to next tier problem
- 15+ across 3 consecutive different-category problems: Declare interview-ready
