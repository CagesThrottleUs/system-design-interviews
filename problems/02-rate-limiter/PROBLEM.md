# Problem 02: Distributed Rate Limiter

**Difficulty:** Intermediate  
**Category:** Distributed Systems, Infrastructure  
**Frequently asked at:** Amazon, Google, Meta, Apple, Stripe, Cloudflare  
**Reference:** Alex Xu, *System Design Interview*, Chapter 4  
**Also prominent as:** LLM gateway rate limiter (2024–2026)

---

## Problem Statement

You are a platform engineer at a company that runs a public API platform — think AWS API Gateway, Stripe, or an LLM inference API like OpenAI. Your API is now serving tens of millions of requests per day across thousands of registered developers and enterprise customers. The business has three tiers: free (60 req/hour), pro (5,000 req/hour), and enterprise (15,000 req/hour).

Last quarter, a single misconfigured client in the free tier sent a burst of 50,000 requests in 30 seconds during a Black Friday event. The downstream microservices — inventory, payments, user auth — cascaded into failure. The incident lasted 47 minutes and cost $2.1M in lost revenue. Your incident postmortem identified no rate limiting as the root cause.

Your job is to design a **distributed rate limiter** that sits in the request path of every API call. It must enforce per-user, per-tier, and per-endpoint limits at scale, survive partial infrastructure failures without taking down legitimate traffic, and do so without adding more than 5ms of latency to every request. The system must scale to 10 million daily active users and handle burst traffic during peak events. The engineering director wants this to be the traffic cop that protects every backend service the company runs — current and future.

---

## Actors

| Actor | Description | Volume |
|-------|-------------|--------|
| API Client (Free) | Registered developers on free tier | ~8M users |
| API Client (Pro) | Paying developers and startups | ~1.8M users |
| API Client (Enterprise) | Large companies with SLA agreements | ~200K users |
| Unauthenticated Caller | Unregistered traffic, bots, scrapers | Unknown; assume 10–20% of traffic |
| Backend Service | Downstream microservices protected by the limiter | 50+ services |
| Platform Admin | Engineers who configure rate limit rules | ~50 internal users |

---

## Functional Requirements

### Core (must-have)

| ID | Requirement |
|----|-------------|
| FR1 | The system must enforce configurable rate limits per user/API key |
| FR2 | Limits must be enforced per endpoint and per user tier (free, pro, enterprise) |
| FR3 | Requests that exceed the limit must be rejected with HTTP 429 Too Many Requests |
| FR4 | Rejected responses must include standard rate limit headers: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`, `Retry-After` |
| FR5 | Admins must be able to update rate limit rules without restarting the service |
| FR6 | The system must support both hard limits (hard block) and soft limits (allow with degraded priority) |

### Secondary (nice-to-have, may be asked in deep dive)

| ID | Requirement |
|----|-------------|
| FR7 | Support per-IP rate limiting for unauthenticated traffic |
| FR8 | Support global rate limits across all users (circuit breaker for the entire platform) |
| FR9 | Emit metrics and alerts when a user is consistently hitting limits (not just per-request) |
| FR10 | Support allowlisting for internal services and trusted partners |

---

## Non-Functional Requirements

| Requirement | Target | Rationale |
|-------------|--------|-----------|
| Latency overhead | ≤5ms p99 added to every request | Anything higher noticeably degrades API SLAs and would be rejected by upstream product teams |
| Availability | 99.99% (52 min downtime/year) | Rate limiter failure = all API traffic blocked or all limits bypassed; both are catastrophic |
| Throughput | 8M req/sec peak (10M DAU, 5x burst) | Based on DAU estimate; see Capacity Estimation Hints |
| Consistency | Approximate: allow up to ~0.1% overage in distributed counters | Exact counting requires coordination that kills the latency budget; slight overage is acceptable |
| Storage | Fit active counters in memory (Redis) | Disk-based counters at sub-ms latency is impractical |
| Rule propagation | New rules propagate within 30 seconds | Admins expect changes to take effect quickly without a full deploy |
| Scalability | Linear scale from 1M to 100M DAU by adding nodes | No single-node bottlenecks; horizontal scaling required |

---

## Capacity Estimation Hints

*(Provide assumptions; work through the math yourself in your answer.)*

**Traffic:**
- Assume 10M DAU
- Each user makes ~10 API calls/minute on average
- Peak traffic is approximately 5× average
- Consider what percentage of requests are reads vs writes

**Storage:**
- Each rate limit counter is a user+endpoint+window key
- Assume a typical user hits ~5 distinct endpoints
- A key + counter takes roughly 100–200 bytes in Redis
- Consider how long counters need to live (TTL = window size)

**Infrastructure:**
- A single Redis node handles ~100K–200K INCR operations/sec
- Redis `INCR` latency: 1–3ms over local network
- Sliding window log: 5–15ms (reads past entries, more expensive)
- A single rate limiter API server can handle ~50K req/sec at 5ms budget

**Rule storage:**
- Rules change infrequently (maybe 100 writes/day)
- Rules are read on every request — critical hot path
- Rule set fits entirely in memory; reads should be local, not network

---

## Clarifying Questions

*(Group by category; ask 3–4 before designing, not all at once)*

### Scale & Load
1. What is the expected DAU and peak requests per second? Is peak traffic predictable (e.g., scheduled events) or bursty and unpredictable?
2. How many distinct rate limit rules exist? (Per-user? Per-endpoint? Per-tier? All three simultaneously?)

### Delivery Semantics & Correctness
3. If the rate limiter's counter store (Redis) goes down, should we fail open (allow all traffic) or fail closed (block all traffic)? What is the business priority — protect backends or maximize availability?
4. Do we need exact enforcement (e.g., billing-sensitive APIs where 1 extra request costs money) or is approximate enforcement acceptable (allow up to 0.1% overage to keep latency low)?

### Latency SLAs
5. What is the latency budget for the rate limiter itself? Is 5ms p99 acceptable, or do we have a tighter SLA?
6. Where in the request path does this sit — at the API gateway before auth, or after auth (so we know the user identity)?

### Ordering & Multi-Tenancy
7. Should limits be enforced per user, per IP, per API key, or all three depending on whether the caller is authenticated?
8. Are there burst allowances — for example, allow 10x the normal rate for 5 seconds before kicking in, or is it strictly per-window?

### Compliance & Operations
9. Do we need to log every rejected request for audit purposes (e.g., for billing disputes or abuse investigation)?
10. How quickly must admin-configured rule changes take effect — seconds, minutes, or is a 30-second propagation delay acceptable?

---

## Key Components

<details>
<summary>Hint: What are the main pieces?</summary>

**Rate Limiter Middleware**  
Sits in the request path (as a gateway plugin, sidecar, or library). Intercepts every incoming request, identifies the caller (user ID, IP, API key), looks up applicable rules, checks and atomically increments the counter, and either forwards the request or returns HTTP 429.

**Counter Store (Redis)**  
Holds the per-user, per-window, per-endpoint counters. Must support atomic increment + expiry in a single operation. Redis `INCR` + `EXPIRE` (or `SET ... EX ... NX`) is the primitive. Multiple Redis nodes (cluster or replica set) are needed at scale.

**Rules Store**  
Holds the rate limit configuration: "user tier X gets Y requests per Z seconds on endpoint W." Rules are read frequently but written rarely. Can be a relational database (source of truth) with an in-memory cache on each rate limiter node, refreshed every 30 seconds.

**Rate Limit Response Handler**  
Constructs the HTTP 429 response with proper headers: `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset` (Unix timestamp of window reset), and `Retry-After` (seconds until retry is safe).

**Metrics & Observability Pipeline**  
Emits per-user, per-endpoint rate limit hit counts. Feeds dashboards and alerts. Critical for distinguishing "rate limiter protecting us" from "rate limiter breaking legitimate users."

**Admin Config Service**  
API for platform admins to create, update, and delete rate limit rules. Writes to the Rules Store; change propagates to in-memory caches within 30 seconds.

</details>
