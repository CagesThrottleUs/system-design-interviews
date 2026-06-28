# Lessons Learned: Distributed Rate Limiter

Real interview stories and recurring failure patterns. Read these before any mock.

---

## Story 1: The Amazon Interview — "Redis Goes Down. Now What?"

A senior engineer candidate (5 years experience) was interviewing for an Amazon SDE II position. They had clearly prepared — they correctly chose token bucket, explained the refill rate formula, and drew a clean architecture diagram with Redis and a downstream service. The interview was going well.

Then the interviewer asked: "Your rate limiter depends on Redis. What happens to your system when Redis goes down?"

The candidate paused. Then said: "We'd handle the error and return a 500."

The interviewer pressed: "A 500 to every API request? That takes the API down completely."

Candidate: "We'd... retry the Redis call?"

Interviewer: "Retrying adds latency on every request. If Redis is down, retries will all fail too. You'd add latency and still have no answer."

The candidate couldn't recover. They hadn't thought about the dependency failure. In their mental model, Redis was a reliable black box.

**The specific gap:** The candidate designed a system where the rate limiter was in the *critical path* of every request and had a single point of failure — but never thought about what happens when that dependency fails. They hadn't asked the question: "What breaks first?"

**What they should have said:** "This is the most important question in the design. The rate limiter sits in front of every request. If it fails, we have two choices: fail-open (allow all traffic to the backend) or fail-closed (block all traffic with 429). The tradeoff is availability vs protection. For an external API, I'd default to fail-open with a local in-process fallback — each rate limiter node maintains a local counter that enforces `global_limit / N` per node. Under Redis outage, a determined attacker can hit all N nodes and get N× the rate, but that's an acceptable risk window compared to taking the entire API offline."

**The mental model to internalize:** Every component you add to the critical path becomes a new availability constraint. Your system's availability is at most the product of each component's availability. A 99.99% API gateway + a 99.9% Redis = ~99.89% combined — you just burned your entire error budget on Redis alone.

---

## Story 2: The Google Interview — Algorithm Expert, Distributed Novice

A candidate interviewing for a Google L5 position had clearly read Alex Xu's chapter and Stripe's engineering blog. They knew every algorithm. Their explanation of sliding window counter (the hybrid approach that approximates the sliding window using two fixed window counters weighted by time overlap) was textbook-perfect.

Then the interviewer asked: "You have 50 API gateway nodes, all sharing Redis. Walk me through what happens at the Redis level when 1,000 concurrent requests for the same user arrive at the same time."

The candidate described the correct algorithm — they said each server would call `INCR` on the Redis key.

Interviewer: "Show me the code. Specifically the read-increment-check sequence."

The candidate wrote:

```python
count = redis.get(key)
if count < limit:
    redis.incr(key)
    forward_request()
```

The interviewer circled `redis.get` and `redis.incr`. "How many Redis roundtrips is this?"

"Two."

"And between those two roundtrips, what can happen?"

The candidate finally saw it. Between the GET and the INCR, 999 other goroutines could read the same stale count and all pass the check. The system would allow far more requests than the limit.

**The specific gap:** The candidate knew the *theory* of the algorithm but missed the *implementation* reality of distributed concurrent increment. Rate limiting only works if the check and the increment are atomic — one indivisible operation. A Lua script on Redis or a `SETNX`-style approach is required.

**What they should have said:** "We can't do a two-step read-then-write — that's a classic TOCTOU race condition. All 1,000 concurrent requests would read the same count, all pass the check, and all increment. We need atomicity. The Redis Lua script runs atomically on the Redis server — no other command can interleave. We EVAL the script once per request: increment, check against the limit, return allowed/denied. One roundtrip, no race."

**The mental model to internalize:** "Distributed" and "concurrent" are not adjectives you add to a system design — they are *constraints* that invalidate naïve implementations. Every multi-step operation in a distributed system needs an answer to: "what happens if another actor modifies this state between my two steps?"

---

## Story 3: The LLM Gateway Interview — New Question Shape, Same Core Principles (2024–2026)

By 2025, "design a rate limiter" had evolved into "design a rate limiter for an LLM inference API" at companies like Anthropic, OpenAI-adjacent teams, and any company with a GenAI platform. This version has additional complexity that tripped up candidates who'd prepared only for the classic formulation.

A candidate at a well-known AI company was asked to design the rate limiting layer for an LLM gateway. They described standard token-bucket rate limiting per API key and seemed confident.

The interviewer asked: "Your rate limiter counts *requests*. But GPT-4 calls vary from 10 tokens to 128,000 tokens. Should you rate limit by request count or by token count?"

The candidate had no prepared answer. They said "request count, I think?"

Interviewer: "One user sends 10 requests of 100 tokens each — 1,000 tokens total. Another sends 1 request of 100,000 tokens. Same request count, 100× different GPU cost and latency. How does your rate limiter handle that?"

The candidate couldn't answer.

**The specific gap:** The candidate had memorized the standard rate limiter design without understanding the *unit of cost* being limited. For LLM APIs, the natural limiting unit is tokens (cost-proportional), not requests (cost-agnostic). The architecture is the same — Redis, Lua script, token bucket — but the "cost per request" is not 1; it's the token count of the request. For streaming completions, you may not know the full token count until the response is finished, requiring post-request accounting or pre-request estimation with correction.

**What they should have said:** "For a standard API, requests are the natural unit — each request is roughly the same cost. But for LLMs, cost scales with tokens. We should use token-weighted rate limiting: each request costs `prompt_tokens + completion_tokens` tokens from the bucket. For streaming responses where we don't know completion length upfront, we deduct a pessimistic estimate (e.g., `prompt_tokens × 2`) at request start and correct after completion. This matches how OpenAI's tier pricing actually works — limits are per minute in tokens, not per minute in requests."

**The mental model to internalize:** Understand the *unit of cost* for the system you're designing. Rate limiting is fundamentally about controlling resource consumption. The unit of measurement must map to the resource being protected. Requests, tokens, bytes, CPU-seconds, dollars — the choice changes the implementation.

---

## General Pattern Mistakes

### Mistake 1: Designing Only the Happy Path

Candidates draw a beautiful architecture with Redis and a rate limiter middleware, walk through how a request is allowed, and stop. They never ask: "What component fails first? What does the system do?" Every real production system fails. The rate limiter is no exception. Before ending your design, ask yourself: "What happens if each box in my diagram disappears?" If you can't answer for every box, you haven't finished.

### Mistake 2: One Redis for Everything

A single Redis instance serving both counters and rules is a common design. It seems simple but it conflates two access patterns that have opposite characteristics: counters are write-heavy, require atomic INCR, have short TTLs, and can tolerate slightly stale reads. Rules are read-heavy, require transactional multi-field updates, have no TTL, and must be consistent. Mixing them means your counter operations contend with rule reads. More practically: if you want to do a rolling restart of your counter Redis, you can't — everything depends on it. Keep them separate.

### Mistake 3: No Response Contract

After correctly designing the rejection logic, candidates say "return HTTP 429" and move on. They never mention `X-RateLimit-Remaining`, `X-RateLimit-Reset`, or `Retry-After`. These headers are not a nice-to-have — they are the contract that allows well-behaved clients to implement exponential backoff, display meaningful error messages to their users, and avoid hammering the API in a retry storm. A rate limiter without proper response headers makes the problem *worse* by creating a feedback loop: clients see 429, immediately retry, get another 429, retry faster. With `Retry-After: 47`, a well-behaved client waits 47 seconds. This is the difference between a rate limiter that solves the problem and one that amplifies it.

### Mistake 4: Ignoring Multi-Granularity Limits

Real-world rate limiters almost always enforce multiple granularities simultaneously. Stripe enforces per-second, per-minute, and per-hour limits on the same API key. The per-second limit prevents burst spikes. The per-minute limit enforces steady-state. The per-hour limit enforces daily budgets. Candidates who design only one granularity are missing how production systems actually work. The implementation handles this by evaluating multiple rules and rejecting if *any* is exceeded — each rule has its own Redis key and window.

### Mistake 5: Forgetting That Rules Must Update Without a Deploy

Most candidates design a rules store but don't explain how new rules take effect. If rules are compiled into the binary or loaded at startup, adding a new limit (say, adding a rate limit for a new endpoint you just launched) requires a full service restart — which means brief downtime or a rolling restart. A production rate limiter must support dynamic rule updates. The solution: a background poller on each rate limiter node that fetches updated rules from Postgres every 30 seconds and swaps them into the in-memory cache atomically. New rule = propagated within 30 seconds, no restart required.

---

## Self-Assessment Checklist

Run through this after every mock or study session on rate limiting.

**Requirements Phase**
- [ ] Did I ask about fail-open vs fail-closed before designing?
- [ ] Did I clarify the identity model (per-user vs per-IP vs per-key)?
- [ ] Did I ask about multiple tiers (free/pro/enterprise) with different limits?
- [ ] Did I ask about burst allowances vs strict per-window limits?
- [ ] Did I clarify latency budget for the rate limiter component?

**Design Phase**
- [ ] Did I separate counter storage from rules storage and explain why?
- [ ] Did I describe the atomic operation (Lua script or equivalent)?
- [ ] Did I mention the race condition that a non-atomic approach would cause?
- [ ] Did I include rate limit response headers (Remaining, Reset, Retry-After)?
- [ ] Did I address what happens when Redis goes down?
- [ ] Did I explain how rule changes propagate without a restart?
- [ ] Did I mention clock skew and Redis TIME as reference clock?

**Deep Dive Phase**
- [ ] Can I explain fixed window's boundary spike problem with a concrete example?
- [ ] Can I explain the token bucket refill formula?
- [ ] Can I draw the Redis data model for at least two algorithms?
- [ ] Can I explain why consistent hashing routes counters to deterministic shards?
- [ ] Can I discuss multi-granularity limits (per-second + per-minute + per-hour)?

**Failure Modes**
- [ ] Redis node down → what happens?
- [ ] Redis partition (split brain) → what happens?
- [ ] Rules store down → what happens?
- [ ] Rate limiter node crash → what happens?
- [ ] Clock skew between nodes → what happens?

**Observability**
- [ ] Did I mention metrics (allowed/rejected counts, per-user headroom)?
- [ ] Did I describe how ops would know if rate limiter is misconfigured (false positives)?

---

## Remediation Targets

| Weak area | Remediation |
|-----------|-------------|
| Race condition + atomic ops | Write the Lua script from scratch, run it against a local Redis, verify behavior under concurrent load |
| Fail-open vs fail-closed reasoning | For 5 different systems (payment API, social feed, internal admin, LLM API, authentication service), argue which you'd choose and why |
| Algorithm tradeoffs | Draw fixed window, sliding window log, sliding window counter, and token bucket side-by-side — label the weakness of each |
| Response headers | Memorize the 4 standard headers and write a sample HTTP 429 response from scratch |
| Capacity math | Redo the capacity section of the Solution Guide without looking — derive Redis node count from DAU |
| Failure modes | For each box in the architecture diagram, write one sentence describing what happens when it fails and what the system does |
