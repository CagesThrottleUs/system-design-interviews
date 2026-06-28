# Caching Strategies

## Intent

Store frequently read data closer to the application to reduce read latency and offload downstream systems.

---

## The Problem

Databases are optimized for durability and consistency, not raw read throughput. A PostgreSQL instance that is healthy at 1,000 queries per second will start showing p99 latency exceeding 100 milliseconds around 10,000 QPS as connection pools saturate, buffer pools thrash, and disk I/O becomes the bottleneck. A Redis instance serving the same key space at 10,000 QPS operates at sub-millisecond latency because it reads entirely from memory and avoids disk I/O almost entirely. The performance gap is not marginal — it is two to three orders of magnitude.

At internet scale this gap becomes existential. Netflix reported a CDN cache hit rate of approximately 95% as of 2022. This means 95% of all video segment requests are served directly from edge caches without ever touching an origin server. At Netflix's scale — hundreds of millions of daily active users, streaming peak loads of several terabits per second — the 5% miss rate still represents enormous traffic, but it is manageable. Without caching, the origin server fleet would need to be 20× larger to handle the same load. The hit rate is the key metric: the fraction of requests that are served from cache without incurring the cost of a cache miss (fetching from origin, incurring latency, loading the database).

A cache miss is not free. On a cold start — when a service deploys fresh with an empty cache — every request hits the database. If your service normally absorbs 99% of reads from cache and you deploy to a new region or restart after a cache flush, that initial wave of traffic lands entirely on the database. This is the cold start problem, and it is a production risk every caching design must address explicitly.

---

## The Four Strategies

**Cache-aside (lazy loading)** is the most common strategy. The application is responsible for cache management: on every read, it checks the cache first; if the key is present (a cache hit), it returns the cached value; if the key is absent (a cache miss), it fetches from the database, stores the result in the cache with a TTL, and returns the value to the caller. Cache-aside gives you fine-grained control and ensures you only cache data that is actually requested. The cost is the cache miss penalty: on cold start, or after a TTL expiry on a popular key, the first request for that key incurs the full database round-trip plus the overhead of writing back to the cache. In practice, cache-aside is the default choice for most read-heavy workloads because the miss penalty is acceptable and the implementation is simple.

**Write-through** ensures the cache and database are always synchronized by writing to both on every mutation. When a write comes in, the application writes to the cache and then (synchronously) writes to the database before acknowledging success to the caller. The guarantee this provides is strong: if a key is in the cache, it is guaranteed to reflect the latest committed value. The cost is write latency: what was a single database write is now two writes in sequence. For workloads that are heavily write-bound (e.g., a metrics ingestion pipeline at 50,000 writes per second), this double-write overhead can become prohibitive. Write-through is the right choice when your read-to-write ratio is high and stale data is unacceptable — financial account balances, session tokens, access control lists.

**Write-behind (write-back)** optimizes for write throughput by breaking the synchronous coupling between cache and database. Writes go to the cache immediately and succeed from the caller's perspective; the cache system then flushes dirty keys to the database asynchronously, typically in batches. The throughput gain is real: a write-behind cache can absorb bursts of writes far exceeding what the underlying database can sustain, smoothing the write curve. The cost is durability risk. If the cache node fails or is evicted before a dirty key is flushed, that data is permanently lost. Write-behind is appropriate when the data being cached is non-critical or reconstructable — user preference updates, aggregate counters, event click streams — and when the business can tolerate occasional data loss. It is never appropriate for financial transactions, authentication state, or any data where loss creates an inconsistency the user would notice.

**Read-through** places the cache transparently in front of the database from the application's perspective. The application issues a read against the cache using the same interface it would use against the database; on a miss, the cache layer itself fetches from the database, populates its store, and returns the result. The distinction from cache-aside is who is responsible for populating the cache on a miss: in cache-aside, it is the application code; in read-through, it is the cache layer itself. This makes read-through attractive when you want to enforce consistent caching logic across multiple service instances without duplicating that logic in every caller. Managed caching systems like Amazon ElastiCache's DAX (DynamoDB Accelerator) implement read-through semantics. The trade-off is reduced visibility: bugs in cache population are harder to diagnose when the application cannot observe the miss path directly.

---

## Cache Invalidation

Phil Karlton, a developer at Netscape in the early 1990s, is credited with the observation that "there are only two hard things in computer science: cache invalidation and naming things." The joke has endured because the problem has not been solved.

The simplest invalidation strategy is TTL-based expiry: every cached key is given a time-to-live, and when that TTL expires, the key is evicted and the next read triggers a miss. TTL is simple to implement and reason about, but it creates a staleness window. If a user updates their profile photo and the profile cache has a 60-second TTL, other users may see the old photo for up to a minute. Whether that is acceptable depends on the application's consistency requirements.

Event-driven invalidation removes the staleness window by explicitly deleting or updating cache entries when the underlying data changes. When the user updates their profile, the write path also sends an invalidation event to the cache. This is more precise but more complex: every write path must know which cache keys it invalidates, and in a distributed system with multiple writers, maintaining that mapping is non-trivial. A missed invalidation silently produces stale reads.

The thundering herd problem is the intersection of TTL-based invalidation and popularity. When a highly requested key expires — a trending news article, a popular product listing — many concurrent requests simultaneously find the cache empty and all race to the database to repopulate it. In extreme cases, this can produce a spike of hundreds or thousands of database queries in a single second for a single key. Three mitigations are commonly used together. First, jittered TTLs: instead of expiring all keys of a type at exactly `base_ttl` seconds, randomize the expiry in the range `base_ttl ± 15%`. This staggers mass expiry and smooths the miss curve. Second, the probabilistic early expiration (PER) algorithm: each request, before the key actually expires, has a small probability of proactively refreshing it based on how close the TTL is to expiry. This warms the cache before the key actually goes cold. Third, a mutex or distributed lock on the repopulation path: only one request is allowed to fetch from the database on a miss; all other concurrent requests either wait or serve slightly stale data until the first request completes.

---

## Eviction Policies

When a cache reaches its memory limit, it must evict existing entries to make room for new ones. The eviction policy determines which entries are sacrificed.

Least Recently Used (LRU) evicts the entry that has not been accessed for the longest time. It is effective when access patterns exhibit temporal locality — recently accessed data is likely to be accessed again soon. Most web caches and session caches fit this pattern. Redis implements an approximation of LRU: rather than maintaining a true sorted access list (which would require O(log n) operations on every read), it samples a configurable number of random keys (5 by default) and evicts the least recently used among the sample. This approximation produces results nearly indistinguishable from true LRU in practice, at a fraction of the memory overhead.

Least Frequently Used (LFU) evicts the entry that has been accessed the fewest total times. LFU is better suited to stable hot-key workloads where some keys are permanently popular — application configuration, country-to-timezone mappings, feature flags. Unlike LRU, LFU does not evict a key merely because it was last accessed an hour ago; it considers total access count. Redis added LFU support in version 4.0 (2017).

FIFO (First In First Out) evicts the oldest inserted key regardless of access frequency. It is rarely the right choice for a cache, because age of insertion has no bearing on future usefulness. Random eviction — evict a randomly selected key — is simple and sometimes competitively effective for uniform access distributions, as demonstrated in experiments comparing eviction policies against real-world traces.

---

## Trigger Signals

- Read traffic is dominated by a small, predictable working set (Pareto distribution on access patterns).
- Database read latency is degrading under load and the read set is largely stable.
- Interviewer introduces a URL shortener, feed timeline, or typeahead — these are canonical caching use cases.
- The system needs to serve identical computed results (e.g., rendered feed, search suggestions) to many users.
- The interviewer asks about scaling reads without adding more database replicas.
- Downstream systems (databases, third-party APIs) have rate limits or high per-call latency.

---

## Problems Where It Applies

**URL Shortener:** The redirect table — mapping a short code to its destination URL — is read-heavy and immutable after creation. Every redirect request benefits from cache-aside: check cache first, on miss load from the database and cache with a long TTL (URLs rarely change).

**Twitter Feed (Timeline Caching):** Twitter described in their 2013 engineering blog their "Timelines" service, which pre-computed and cached the timelines of active users in Redis. By 2013 they were caching timelines for hundreds of millions of users, with the most active user timelines held in RAM indefinitely. The write path (a new tweet) fanned out into all follower timeline caches immediately.

**Typeahead / Autocomplete:** Prefix-to-suggestion mappings are expensive to compute (trie traversal over millions of terms) but are requested millions of times per minute for common prefixes. Cache-aside with a read-through variant is standard: cache the top-k suggestions for each prefix, refreshing on a scheduled cycle rather than on demand.

**YouTube (Metadata Caching):** Video metadata — title, description, view count, thumbnail URL — is read millions of times per video per day but updated infrequently. YouTube caches video metadata aggressively in a tiered cache, with a short TTL on view counts (which change frequently) and a much longer TTL on static fields like title and description.

---

## Trade-offs

**Consistency.** Every caching strategy except write-through accepts a window of staleness. Whether that window is 30 seconds (TTL-based) or seconds (event-driven invalidation with propagation delay), reads may return data that does not reflect the latest write. In a distributed system where multiple application servers each have their own local cache layer, two simultaneous requests can return different values for the same key during an invalidation window. For applications where stale reads are user-visible and harmful (bank balance, inventory availability), caching requires either strong invalidation guarantees or must be scoped to data where staleness is acceptable.

**Memory cost.** Cache memory is finite. A cache cluster large enough to hold your full working set provides near-perfect hit rates; a cache that holds 10% of the working set forces 90% of requests through to the database. Right-sizing the cache requires knowing your working set size and key access distribution — data you often do not have until the system is running.

**Cold start.** An empty cache performs no better than no cache, and it is worse in one specific way: the first wave of requests all miss, producing a spike of database load that may exceed what the database was dimensioned to handle. Mitigations include cache warming scripts (pre-populating the cache before traffic arrives), gradual traffic rollout (blue-green or canary deployments), and circuit breakers that shed load when the database is overwhelmed.

**Cache stampede.** Distinct from the thundering herd (which is about a single key expiring), a cache stampede occurs when many keys expire simultaneously — for example, when a cache cluster restarts after a deployment and all cached keys expire at once. Jittered TTLs mitigate this at the key level; coordinated cache warm-up scripts address it at the cluster level.

---

## Real-World Usage

**Instagram (Tiered Memcached Cache, ~2019):** Instagram's engineering team documented their use of a tiered Memcached architecture in which a large, persistent Memcached cluster holds the canonical cached state, and each application server maintains a smaller local cache (L1) for ultra-hot keys. The L1 cache reduces latency for the most popular objects (celebrity profiles, trending content) to near-zero by avoiding a network hop to the shared Memcached cluster. Instagram reported in approximately 2019 that their Memcached deployment spans hundreds of servers and handles millions of requests per second at peak. Cache-aside is the primary strategy; invalidation is event-driven via a write path that publishes invalidation events to all cache tiers.

**Airbnb (Listing Cache):** Airbnb caches property listing metadata — price, availability, photos, host details — using a cache-aside strategy. A listing page involves aggregating data from multiple backing services (pricing engine, calendar service, review service). Each service has its own cache layer with TTLs tuned to the data's update frequency: availability data has a short TTL (seconds to minutes, since it changes on every booking), while host reputation scores have a TTL measured in hours.

**Twitter (Timeline Cache, ~2013):** Twitter's engineering blog post from 2013 describes their Timelines service storing pre-computed home timelines in Redis. Each timeline is a sorted list of tweet IDs (not full tweet objects). For most users, the timeline is cached entirely in RAM. The "big" users (celebrities, news accounts) whose tweets fan out to hundreds of millions of followers were handled separately — their tweets were not pushed into every follower's cache; instead, follower timelines were assembled on-the-fly by merging the cached timeline with a real-time lookup of the celebrity's recent tweets. At the time, Twitter's Timelines service was serving hundreds of millions of users with a p99 latency under 100ms.

---

## Common Mistakes

**1. Proposing caching without specifying the invalidation strategy.** A cache with no invalidation plan silently serves stale data indefinitely. Interviewers will always probe: "What happens when the underlying data changes?" You must specify: TTL-based, event-driven, or write-through, and defend why that invalidation window is acceptable for the specific data being cached.

**2. Caching mutable, user-specific data with a long TTL.** Caching a user's account balance or cart contents with a 5-minute TTL means a user who makes a purchase could see an incorrect balance for up to 5 minutes. Candidates often apply a single TTL strategy uniformly across all data types. The correct answer differentiates: long TTL for slow-changing public data (product descriptions), short TTL or event-driven invalidation for rapidly changing or user-specific data.

**3. Ignoring the cache cold start problem.** Most candidates describe a steady-state system. Interviewers will introduce a variant: "What happens when you deploy a new region?" or "What happens after a cache restart?" A system that depends on a warm cache for its latency SLAs must have a strategy for surviving cache cold starts without spiking the database.

**4. Treating the cache as the primary store.** Write-behind can make a cache behave like a primary store for a window of time, but it is not a database. Candidates sometimes describe write-behind as equivalent to durable storage. The distinction matters: if the interviewer introduces a node failure scenario, write-behind produces data loss that must be acknowledged as an explicit tradeoff.

**5. Proposing caching as the answer to a consistency problem.** Caching introduces eventual consistency; it does not solve consistency problems. If the interviewer introduces a scenario where two users must see the same inventory count before both can book the same item (a hotel room, a concert ticket), caching the inventory count with any TTL greater than zero creates a race condition where both users see "1 remaining" and both successfully book. This class of problem requires a transactional write path with pessimistic locking or compare-and-swap, not a cache.
