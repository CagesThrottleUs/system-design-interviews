# Rate Limiting Algorithms

## Intent

Bound the request rate any single client, tenant, or endpoint can generate so that one actor's traffic cannot exhaust shared infrastructure capacity.

---

## The Problem

Without rate limiting, a single client sending 10,000 requests per second can overwhelm a service designed for 1,000 req/sec across all clients — the equivalent of a self-inflicted DDoS. The problem compounds in distributed deployments. With 50 API server instances sitting behind a load balancer, each server sees roughly 1/50 of total traffic. A client that round-robins requests across all 50 servers can send 50× the intended limit while every individual server's local counter stays well below its threshold. In-process counters are therefore useless for enforcement in any horizontally-scaled system. You need a shared, atomic counter that all API nodes consult — and designing that counter well is the core engineering challenge.

A second, subtler problem: rate limiting is almost always applied at multiple granularities simultaneously. You may want to limit a specific user to 1,000 req/hour, limit a specific endpoint to 10,000 req/sec globally, and limit an IP address to 100 req/min on the login endpoint. These three policies can coexist on the same request. Understanding each algorithm's memory and latency characteristics is what allows you to choose the right one for each policy.

---

## The Five Algorithms

### 1. Token Bucket

A bucket holds tokens up to a maximum capacity B. Tokens are added at a constant rate R tokens per second. Each incoming request consumes one token. If the bucket is empty when a request arrives, that request is rejected (or queued, depending on policy). The critical property is that the bucket allows instantaneous bursts of up to B requests — you can drain the entire bucket in a single millisecond — while enforcing the average rate R over any sufficiently long window.

Implementation in Redis is straightforward: store two values per client — a token count and a last-refill timestamp. On each request, compute elapsed time since last refill, add `elapsed * R` tokens (capped at B), then attempt to decrement. The entire read-modify-write must execute atomically, which means using a Lua script or Redis's `EVAL` to avoid race conditions between the read and write.

Token bucket is the right choice for APIs where short bursts are acceptable and expected. Mobile applications are a canonical case: when a user opens an app after being offline, it might legitimately send a burst of 20 API calls in the first second to refresh its state, then settle to an average of 2 calls per second. A token bucket with B=20 and R=2 handles this gracefully. A strict rate limiter would reject 18 of those 20 bursting calls and break the user experience.

The trade-off: token bucket is generous about bursting, which means it is also the easiest to abuse. A client that knows the bucket capacity can intentionally burst at the limit repeatedly. If you need to protect a fragile downstream system from any instantaneous spike, the leaky bucket is a better fit.

### 2. Leaky Bucket

Requests enter a FIFO queue of capacity Q (the "bucket"). A worker processes requests from the front of the queue at a fixed rate R. If the queue is full when a new request arrives, that request is dropped. Unlike the token bucket, there is no concept of stored capacity — the output rate is always exactly R, regardless of how bursty the input is. The bucket "leaks" at a constant rate.

The implementation reflects this: a fixed-size queue per client, a background processing thread or event loop that drains the queue at rate R, and a size check on enqueue. In practice, pure leaky bucket implementations at the application layer are less common than token bucket, because the queue introduces latency (your request might sit in the queue for up to Q/R seconds before being processed), which is often unacceptable for synchronous APIs.

The leaky bucket's strength is its absolute output smoothness. It is the right choice when you need to protect a downstream component from any bursty input — for example, smoothing writes to a database that can handle exactly 500 writes/second but will fall over at 600. A queue in front of it that drains at 500/second gives the database a perfectly steady write rate regardless of how spiky the upstream traffic is. Message queue systems like SQS consumer groups naturally implement leaky bucket semantics at the consumer side.

The trade-off: the smoothness is also a limitation. Leaky bucket cannot accommodate any burst, even when the system has spare capacity. A client that has been quiet for 10 seconds cannot spend that accumulated "goodwill" on a short burst — there is no concept of accumulated tokens.

### 3. Fixed Window Counter

Divide time into fixed, non-overlapping windows of duration W (e.g., one minute from :00 to :59). Maintain a single counter per client per window. Each request increments the counter; if the counter exceeds the limit, reject the request. At the start of each new window, reset the counter to zero.

The implementation is extremely simple: a Redis key like `ratelimit:{user_id}:{window_number}` with `INCR` and an expiry set to `2W` (to allow for clock skew). Memory cost is O(1) per client. Latency is a single Redis round-trip. This simplicity makes fixed window counters attractive for high-scale systems where you can tolerate the one known flaw.

That flaw is the window boundary burst. Consider a limit of 100 requests per minute. A client sends 100 requests between 11:59:45 and 11:59:59 — all are accepted, because the counter in the 11:xx window is at 100. At 12:00:00, the window resets. The client immediately sends another 100 requests, all of which are accepted because the new counter starts at zero. In the 30-second span from 11:59:45 to 12:00:14, the client has sent 200 requests — exactly double the stated limit. A downstream service that expected at most 100 requests per minute just received 200 in 30 seconds.

For many use cases, this edge case is acceptable. If you are rate-limiting non-critical API access and the burst is brief, a fixed window counter's simplicity and O(1) cost may outweigh the accuracy concern. But for protecting login endpoints from brute-force attacks or enforcing strict SLA commitments, the boundary burst is a real vulnerability.

### 4. Sliding Window Log

Store the exact timestamp of every request from a client in a sorted set (e.g., Redis `ZSET` keyed by user_id). On each request, remove all timestamps older than `now - window_size`, count the remaining entries, and if that count is below the limit, add the new timestamp and allow the request.

This approach is perfectly accurate. There is no window boundary — the window truly slides. A client cannot exploit the 11:59/12:00 boundary because the algorithm always looks at the last 60 seconds regardless of wall-clock minute boundaries.

The cost is memory proportional to actual request volume. If you allow 1,000 requests per hour per user and you have 1 million active users all at the maximum rate, you are storing up to 1 billion timestamps in Redis. Each Redis sorted set entry is approximately 50–100 bytes, which means up to 100 GB of RAM for that single rate-limiting policy. In practice, users are not all at their limit simultaneously, but this worst-case analysis makes sliding window log suitable only for low-traffic scenarios or very low limits. It works well for APIs with a limit of 10 req/minute per user where accuracy is critical — say, a payment submission endpoint.

The trade-off: accuracy at the cost of memory proportional to traffic. The algorithm also requires a `ZREMRANGEBYSCORE` (to expire old entries) plus `ZCARD` plus `ZADD` on every request, which is three Redis operations versus one for a fixed window counter.

### 5. Sliding Window Counter

This is the hybrid that most production systems actually deploy. It approximates the accuracy of a sliding window log with the O(1) memory of a fixed window counter.

The algorithm keeps two fixed window counts: the count for the previous completed window (`prev`) and the count for the current in-progress window (`curr`). It also tracks `elapsed` — how far through the current window we are as a fraction from 0.0 to 1.0. The estimated count for the true sliding window is:

```
estimated = prev * (1 - elapsed) + curr
```

The intuition: if we are 30% through the current minute, the last 60 seconds consists of 70% of the previous minute plus 30% of the current minute. We weight each window's count by its contribution to the sliding window.

The error bound on this approximation is small. Cloudflare analyzed the approach and found the maximum error is approximately 0.003% of the window size under adversarial traffic patterns. For a limit of 1,000 requests per minute, this means the algorithm might occasionally allow 1,000.03 requests — effectively zero real-world impact.

Memory cost: two counters per client per policy. At 1 million users, this is 2 million integers — trivially storable. This combination of accuracy and efficiency is why sliding window counter is the default choice for production rate limiting systems, including the implementations used by major API providers.

---

## Distributed Rate Limiting

The problem with local counters is fundamental, not incidental. With 50 API servers, a client that distributes its traffic evenly sees only 2% of total requests on each server. If the rate limit is 100 req/sec and each server enforces independently, the client can actually send 5,000 req/sec total before any server rejects anything. The limit is worthless.

The standard solution is Redis as a centralized atomic counter. All 50 API servers talk to the same Redis instance (or Redis cluster) for rate limit lookups. The critical requirement is atomicity: the read-check-increment must be a single atomic operation. Without atomicity, consider this race:

- Server A reads count = 99 (limit is 100)
- Server B reads count = 99
- Server A increments to 100 — request allowed
- Server B increments to 100 — request allowed

Both requests went through, but together they caused the 101st request in the window to succeed. With high concurrency, this race can allow significantly more traffic than the limit.

Redis's `INCR` command is atomic by design — Redis is single-threaded for command execution, so INCR is guaranteed to be a read-modify-write with no interleaving. A standard pattern is a Lua script that combines INCR with EXPIRE (to set the key TTL atomically on first creation):

```lua
local count = redis.call('INCR', KEYS[1])
if count == 1 then
  redis.call('EXPIRE', KEYS[1], ARGV[1])
end
return count
```

For the sliding window counter, two keys are needed (current and previous window). Both operations still execute atomically in the Lua script.

Redis replication lag introduces a secondary concern in multi-region deployments. If you replicate a Redis primary to a secondary in another region and enforce rate limits against the local secondary, requests in two regions can proceed simultaneously against stale data. Stripe handles this by using regional rate limit enforcement with separate per-region quotas rather than a single global counter — a write to the US doesn't block a write to the EU, and the combined global limit is enforced approximately, with reconciliation in the background.

---

## Rate Limit Response Headers

When a request is rate-limited or approaching the limit, the response must include headers that allow clients to self-throttle and implement correct retry logic. These headers are standardized across the industry:

- `X-RateLimit-Limit` — the total number of requests allowed in the window (e.g., `1000`)
- `X-RateLimit-Remaining` — requests remaining in the current window (e.g., `743`)
- `X-RateLimit-Reset` — Unix timestamp at which the window resets and the counter clears (e.g., `1719532800`)
- `Retry-After` — present only on 429 responses; seconds the client must wait before retrying (e.g., `42`)

The `Retry-After` header is especially important. Without it, clients will typically implement exponential backoff starting from a short interval, which generates thundering-herd retries when a window resets and every throttled client simultaneously becomes eligible. A precise `Retry-After` tells the client exactly when to retry, distributing retry traffic more evenly across the new window.

Interviewers at infrastructure-focused companies will specifically ask about 429 handling. A well-designed API also returns a `Retry-After` header on 503 responses from overloaded services, not just rate-limit rejections — the client retry semantics are the same regardless of whether the delay is enforced or recommended.

---

## Trigger Signals

Mention rate limiting when the problem involves:
- Any public-facing API (someone will ask "how do you prevent abuse?")
- Login or authentication endpoints (brute-force prevention is a rate limiting problem)
- A design with a downstream service that has a known throughput limit and needs protection
- Any mention of "fair usage" or "tenant isolation" in multi-tenant systems
- Webhook delivery systems where you need to throttle delivery per subscriber
- Notification systems where you need to limit messages per user per time period
- Any design where the interviewer mentions traffic spikes or DDoS scenarios

---

## Problems Where It Applies

**Rate Limiter (standalone)** — The obvious one; the entire problem is selecting and implementing the algorithm.

**Design Uber API** — Each driver app sends location updates every 4 seconds. At 1M active drivers, this is 250,000 writes/sec globally. Rate limiting per driver (maximum 1 update/3 seconds) prevents a misbehaving driver app from hammering the location service with hundreds of updates per second and starving other drivers' updates.

**Design a Public API** — Any REST API for third parties needs tiered rate limiting: free tier at 60 req/hour, paid tier at 5,000 req/hour, enterprise tier at 100,000 req/hour. The sliding window counter at per-API-key granularity with Redis is the standard implementation.

**Login Endpoint Protection** — Anti-brute-force: limit 5 failed login attempts per account per 15-minute window. Fixed window counter is sufficient here because the consequence of the boundary burst (an extra 5 attempts in 30 seconds) is acceptable relative to the implementation simplicity.

---

## Trade-offs

**Accuracy vs. memory vs. latency:** Sliding window log is most accurate but uses O(requests) memory. Fixed window counter is least accurate but O(1). Sliding window counter approximates accuracy at O(1) cost. Token and leaky buckets are O(1) but model different flow properties (bursty vs. smooth). Choose sliding window counter as the default; switch to sliding window log only when you can bound request volume and need forensic accuracy.

**Granularity — per user vs. per IP vs. per API key vs. per tenant:** These serve different threat models. Per-IP limits protect against unauthenticated abuse (scraping, credential stuffing before authentication) but break in NAT environments where thousands of corporate users share one IP. Per-API-key limits enforce developer quotas but don't protect against a compromised key that's being used maliciously. Per-tenant limits enforce business SLA guarantees. In practice, you layer all three: a loose per-IP limit at the edge (Cloudflare / nginx), a strict per-API-key limit at the API gateway, and a per-tenant limit in the business logic layer.

**Global rate limit vs. per-endpoint limit:** A single global limit of 1,000 req/hour is simpler but allows a client to exhaust their quota entirely on one expensive endpoint (e.g., a search endpoint that scans 10M records) and then fail on cheap endpoints (a profile read). Per-endpoint limits — say, 100 req/hour for search and 900 req/hour for read operations — protect specific expensive operations independently. GitHub uses per-endpoint limits for their GraphQL API, billing different query complexities differently, which is the logical extension of this idea.

**Soft limit vs. hard limit:** Hard limits reject immediately with 429. Soft limits throttle — either delay the request (adding artificial latency to smooth traffic) or add the request to a queue. Soft limits are kinder to clients but add latency and complexity. Twilio uses a soft limit approach for SMS sending: requests above the rate limit are queued for a short period rather than rejected, because an SMS that arrives 500ms late is far preferable to an SMS that is dropped entirely.

**Synchronous enforcement vs. asynchronous enforcement:** For maximum accuracy, check the rate limit before processing the request (synchronous). For maximum throughput, process the request optimistically and count after the fact, rejecting if over-limit on the next request (asynchronous). Asynchronous enforcement has systematic over-admission by roughly one request per concurrent server under high load, but avoids a Redis round-trip on the critical path. Stripe uses synchronous enforcement for financial operations; less critical APIs can use asynchronous.

---

## Real-World Usage

**Stripe** documented their rate limiting architecture in their engineering blog. As of approximately 2023, the Stripe API applies rate limits per API key in live mode at roughly 100 write requests per second per API key, with separate, often lower limits per specific endpoint. Stripe uses Redis-backed token bucket counters with per-endpoint customization, allowing their payments endpoints to have independent limits from their metadata and reporting endpoints. They return precise `Retry-After` headers and recommend exponential backoff with jitter.

**GitHub** applies rate limiting at two levels as of 2024. Unauthenticated requests are limited to 60 requests per hour per originating IP. Authenticated requests with a personal access token are limited to 5,000 requests per hour per token. The REST API uses a fixed window counter resetting on the hour; the GraphQL API uses a point-based system where each query is assigned a cost between 1 and 5,000 points, and the per-hour limit is 5,000 points — a query that returns 100 nodes costs more than a query returning 10. This is the production-grade evolution of simple request counting: charging for resource consumption rather than request count.

**Cloudflare** runs rate limiting at the edge across their global network without a central Redis. Their distributed rate limiting system uses local counters per PoP (point of presence) with periodic synchronization, accepting eventual consistency in exchange for sub-millisecond enforcement latency at edge nodes in 2024. They describe this as "approximate rate limiting" — the actual behavior is that a request might be allowed even if the global counter is slightly over the limit, but the overage is bounded. For Cloudflare's use case (blocking DDoS traffic at layer 7), approximate limiting at wire speed is the right trade-off: a 5% over-admission during a coordinated attack is irrelevant if you're stopping 99.9% of the attack traffic.

---

## Common Mistakes

**1. Using a fixed window counter on the login endpoint without understanding the boundary burst.** A limit of "5 failed attempts per minute" enforced with a fixed window counter allows 10 failed attempts in the 2-second window straddling a minute boundary. For brute-force protection on a 4-digit PIN, 10 attempts instead of 5 doubles the attack surface. Use a sliding window counter or sliding window log for security-critical endpoints.

**2. Implementing rate limiting in application code without atomicity.** The check-then-increment pattern — read count, compare to limit, if under limit then increment and allow — has a TOCTOU (time-of-check to time-of-use) race condition. Two concurrent requests both read "99", both compare against 100, both increment to 100, and both are allowed. Under high concurrency, the actual admission rate can be 2–3× the stated limit. Always use Redis INCR (atomic) or a Lua script that combines the check and increment.

**3. Storing timestamps in milliseconds for high-traffic APIs in the sliding window log.** If you store a 13-digit millisecond Unix timestamp in a Redis sorted set for a client that sends 1,000 requests per second, you accumulate 1,000 entries per second per client. For a 1-hour window limit across 10,000 concurrent clients, the sorted set storage is 36 billion entries. The sliding window log requires bounding the concurrent client count or the maximum requests per window to be practical.

**4. Returning a 429 without a `Retry-After` header.** Without `Retry-After`, well-written clients implement exponential backoff from a short base interval. Every throttled client that backed off for (say) 1 second will retry simultaneously when their backoff expires, creating a thundering herd at the rate-limit reset boundary. This can cause the rate limiter to reject the thundering herd again, extending the outage. `Retry-After` staggers retries naturally.

**5. Rate limiting at a single layer.** Limiting only at the API gateway and not at individual microservices means that a single microservice can still be overwhelmed if the API gateway limit is set too loosely or if the microservice is called from multiple upstream paths. Defense-in-depth means: a coarse limit at the edge (per IP, per API key), a medium limit at the API gateway, and a fine-grained limit at service-to-service calls using something like a circuit breaker combined with per-caller rate limiting. Forgetting to mention the service-mesh layer is a common HLD omission in interviews.
