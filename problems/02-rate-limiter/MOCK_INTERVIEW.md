# Mock Interview Script: Distributed Rate Limiter

**Target level:** Apple ICT3 / Google L5 / Amazon SDE II  
**Total time:** 45 minutes  
**Interviewer:** Read every line in italics aloud, exactly as written. Improvise only when the candidate opens a door.

---

## Before You Start

Set up your scoring sheet. Track these categories live:

| Category | Max | Running Score |
|----------|-----|--------------|
| Requirements clarification | 3 | |
| Architecture completeness | 5 | |
| Atomic operations (race condition) | 4 | |
| Fail-open vs fail-closed | 3 | |
| Algorithm tradeoffs | 3 | |
| Observability & headers | 2 | |
| Communication | 1 | |
| **Total** | **21** | |

---

## Phase 1: Problem Setup & Requirements (0–8 min)

**Opening (read verbatim):**

*"I'd like you to design a distributed rate limiter. You're working at an API platform company — think something like AWS API Gateway or Stripe. You serve about 10 million daily active users across three tiers: free users get 60 requests per hour, pro users get 5,000 per hour, and enterprise users get 15,000 per hour. You need to build a rate limiter that enforces these limits at scale. You have about 45 minutes. I'm happy to answer questions. Please think out loud."*

---

**Wait for candidate to ask clarifying questions. Do not volunteer information. Score their questions:**

**Questions that MUST be asked for full score (award 1 point each):**

| Question | Score if asked | Response to give |
|----------|---------------|-----------------|
| Fail-open or fail-closed when rate limiter fails? | 1 | *"Good question. Let's say the business priority is protecting the backends. What would you recommend, and what are the tradeoffs?"* — let them answer before you agree or disagree |
| Per-user vs per-IP vs both? | 1 | *"Authenticated users should be limited by user ID. Unauthenticated requests — scrapers, bots — should be limited by IP. Assume you have both."* |
| Latency budget? | 1 | *"The overall API SLA is 100ms p99. Budget whatever you think is reasonable for the rate limiter, but defend it."* |

**Other reasonable questions (do not score, but do answer):**
- Exact vs approximate counting? → *"Approximate is acceptable. A 0.1% overage is fine."*
- Global limits (platform-wide circuit breaker)? → *"Nice-to-have, not required. Focus on per-user limits first."*
- How quickly must rule changes propagate? → *"Within 30 seconds is acceptable."*
- Multiple granularities (per-second + per-hour)? → *"Interesting — park that for the deep dive. Focus on per-hour for now."*

**If the candidate asks zero clarifying questions and jumps straight to design:**  
Interrupt once: *"Before you dive in — any questions about the requirements? I find the constraints matter a lot here."* If they still dive in: let them. Dock the requirements score to 0–1.

**Time check at 8 minutes.** If requirements are dragging past 10 minutes: *"Good — let's set those as our constraints and move into the design. You can refine as we go."*

---

## Phase 2: High-Level Design (8–30 min)

**Prompt (read if they don't start):**

*"Walk me through your high-level design. Start with the components and how a request flows through them."*

---

**Things to listen for. If not mentioned within 5 minutes, ask the corresponding probe:**

### Component: Counter Store

If they don't mention Redis or an equivalent in-memory store within 5 minutes:  
*"Where are you storing the request counters? Walk me through that decision."*

If they say "database" or "PostgreSQL" for counters:  
*"You're proposing a database write on every API request. At 8 million requests per second, what's your concern there?"*

*(Wait for them to arrive at "latency" and "write throughput." If they don't:)*  
*"At 8M req/sec, how long would each request have to wait for the database?"*

### Atomic operations — the core probe

When the candidate describes how the rate limiter checks and increments the counter, ask:

*"Show me the exact sequence of operations against your counter store for a single request. Write it out in pseudocode."*

**If they write a two-step GET-then-INCR:**  
*"Walk me through what happens if 1,000 requests for the same user arrive at the same time. Specifically — what does each of those 1,000 requests see when they call GET?"*

*(Wait. If they don't spot the race condition:)*  
*"They all call GET before any of them has called INCR. What do they all see?"*

*(Wait. They should say "the same stale value." If they don't:)*  
*"They all see count=0. They all pass the check. They all call INCR. How many requests did you just allow past a limit of 100?"*

**After they identify the race condition, ask:**  
*"How do you fix it?"*

**Award 3/4 if they arrive here with prompting. Award 4/4 only if they proactively identified the race condition before being asked.**

### Fail-open vs fail-closed — the production probe

*"Your rate limiter depends on Redis. Redis goes down. What does your system do?"*

**Score 0/3** if: blank, says "return 500 to all requests," says "retry and hope Redis recovers"  
**Score 1/3** if: correctly names fail-open (allow all traffic) and fail-closed (block all traffic), but doesn't name a tradeoff or recommendation  
**Score 2/3** if: names both options, explains the tradeoff (availability vs protection), picks one with justification  
**Score 3/3** if: names both options, explains the tradeoff, proposes the hybrid (local in-process fallback counters at limit/N per node), and connects it to the business priority stated in requirements

### Rules vs counters separation

If they use the same store for both rules and counters, ask:  
*"Your rules change maybe 100 times a day. Your counters change 8 million times a second. Why are you putting them in the same store?"*

*(Let them reason through it. If stuck:)*  
*"What are the access patterns of rules vs counters? How do those patterns affect your store choice?"*

### Response contract

If they haven't mentioned rate limit headers, ask at the 20-minute mark:  
*"You're rejecting a request with 429. What does the response body and headers look like? What does the client do with that response?"*

They should mention: `X-RateLimit-Remaining`, `X-RateLimit-Reset`, `Retry-After`. Award 1/2 for mentioning headers at all. Award 2/2 for explaining that `Retry-After` enables client-side backoff, preventing retry storms.

---

**Time check at 25 minutes.** If architecture is complete, move to deep dive. If still in HLD:  
*"Let's assume the high-level design is solid. I want to go deeper on a few areas."*

---

## Phase 3: Deep Dive (30–40 min)

Pick **two** of the following based on what the candidate missed in Phase 2:

---

### Deep Dive A: Algorithm Tradeoffs

*"You chose [algorithm they picked]. Walk me through its weaknesses. When would you NOT use it?"*

Then:  
*"A user is allowed 100 requests per minute. It's 11:59:58 PM. They send 100 requests. It's now 12:00:01 AM. They send 100 more. How many requests did they just send in 3 seconds? Is this a problem with your algorithm?"*

*(This is the fixed window boundary spike. They should identify it. If not:)*  
*"What does your counter look like at 12:00:00 AM? What about at 11:59:59 PM?"*

If they designed token bucket, ask:  
*"Walk me through the math. A user is allowed 100 req/hour with a burst of 200. It's been idle for 2 hours. How many requests can they send right now? Show me the calculation."*

*(They should say: min(200, 0 + 2 hours × 100/hour tokens) = min(200, 200) = 200 tokens. Bucket is capped at burst capacity, not 2× burst.)*

**Award algorithm tradeoffs score:**  
- 3/3: Proactively identifies weakness of chosen algorithm + explains boundary spike problem  
- 2/3: Explains weakness when asked + correctly answers the boundary spike question  
- 1/3: Can explain the chosen algorithm but struggles with comparison or boundary case  
- 0/3: Cannot explain tradeoffs between algorithms

---

### Deep Dive B: Distributed Coordination

*"You have 50 rate limiter nodes. Each one can handle 160K req/sec. They all share the same Redis cluster. How do you ensure a user's requests on Node 1 and Node 2 see the same counter?"*

*(They should say: Redis is the single source of truth. Consistent hashing routes a given user+key to a specific Redis shard. All limiter nodes hit the same shard for the same key.)*

Then:  
*"Your Redis cluster has 10 shards. Shard 3 goes down. What happens to the users whose keys were on Shard 3?"*

*(If fail-open: those users are temporarily unthrottled. If fail-closed: those users are temporarily blocked. This is the per-shard version of the fail-open/fail-closed problem. They should connect it.)*

Then:  
*"Node A uses its local system clock for window boundaries. Node B's clock is 200ms fast. User hits Node A at 11:59:59.900 — that's in the old window. Same user hits Node B at 12:00:00.050 — but Node B thinks it's 12:00:00.250, so it puts it in the new window. Is this a problem?"*

*(They should identify clock skew as a source of window-boundary inconsistency and propose Redis TIME command as the reference clock.)*

---

### Deep Dive C: Observability (use if candidate skipped it entirely)

*"Rate limiter is deployed. Three weeks later, your support team gets 50 tickets from enterprise customers saying they're getting 429s even though they're sure they're under their limit. How would you investigate?"*

*(Listen for: per-user metrics on 429 rate, per-endpoint counters, ability to inspect a specific user's current counter + rule in Redis, audit log of rule changes.)*

*"What metrics should the rate limiter emit so your on-call engineer can distinguish between 'rate limiter is working correctly and protecting us' vs 'rate limiter is misconfigured and blocking legitimate users'?"*

---

## Phase 4: Failure Probe (40–45 min)

*"Last question. Imagine it's 11 PM. You've deployed the rate limiter. An alert fires: 429 error rate just jumped from 0.01% to 12% across all users and tiers. Walk me through your incident response."*

**Look for:**
- Checking metrics: which users/tiers/endpoints are affected
- Checking Redis: is it healthy? latency normal?
- Checking rule store: did any rules change recently?
- Checking deployment: was there a deploy in the last 30 minutes?
- Rollback procedure: can they roll back a bad rule update without a deploy?

*"It's not Redis. You check the rules store. Someone updated the free tier limit from 60 req/hour to 60 req/second by mistake. You fix the rule. How long until customers stop seeing 429s?"*

*(They should say: up to 30 seconds for the cache refresh to propagate the corrected rule. This ties back to their dynamic rule propagation design.)*

---

## Closing (45 min)

*"Good. We're at time. I'll share some feedback."*

**Compute final score:**

| Category | Max | Score |
|----------|-----|-------|
| Requirements clarification (3 must-ask questions) | 3 | |
| Architecture completeness (all major components present, separated correctly) | 5 | |
| Atomic operations (race condition identified + Lua/atomic solution) | 4 | |
| Fail-open vs fail-closed (named tradeoff + recommendation + hybrid) | 3 | |
| Algorithm tradeoffs (weakness of chosen + boundary spike or token math) | 3 | |
| Observability & headers (429 headers + metrics discussion) | 2 | |
| Communication (clear, structured, thinks out loud) | 1 | |
| **Total** | **21** | |

**Scoring interpretation:**
- 18–21: Strong hire — proactive, failure-first thinking, production awareness
- 13–17: Lean hire — solid fundamentals, needs polish on failure modes and production depth
- 8–12: No hire — misses critical distributed systems properties (atomic ops, fail behavior)
- 0–7: Strong no hire — fundamental gaps in concurrent system design

**Verbal feedback structure:**
1. Name one thing they did well (specific, not generic)
2. Name the single most important gap (the one thing that would change the hire/no-hire)
3. Suggest one concrete remediation (read X, practice Y, study Z)
