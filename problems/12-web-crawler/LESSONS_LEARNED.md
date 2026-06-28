# Lessons Learned — Web Crawler

---

## Real Interview Stories

### Story 1 — Google, L5 (2024)

The candidate described a URL queue backed by Redis. The interviewer asked: "You have 5 billion URLs. How do you ensure you never send more than one request per second to any single domain?" The candidate proposed a distributed rate limiter with a shared Redis counter per domain. The interviewer pushed: "You have 10,000 pages/sec distributed across 200 worker machines. Every worker now does a Redis call per URL to check the domain counter. What happens to Redis?"

The candidate calculated: 10,000 requests/sec × 1 Redis write = Redis becomes the bottleneck and the single point of failure for the entire crawler. They then proposed local in-memory rate limiters per worker but couldn't explain how to coordinate them without either a shared state (same Redis problem) or allowing politeness violations.

The interviewer explained the two-tier queue: domain-to-worker affinity means each domain is owned by exactly one worker, so that worker's local in-memory rate limiter is authoritative — no coordination needed. The candidate had re-derived the problem that the two-tier frontier solves, but didn't know the solution. **The lesson: understand the two-tier URL frontier and why domain affinity is the key insight.**

---

### Story 2 — Microsoft/Bing, Senior Engineer (2023)

The candidate designed a crawler with a simple Bloom filter for URL deduplication. The interviewer asked: "What's the false positive rate of your Bloom filter, and what happens when you get a false positive?" The candidate correctly answered that a false positive means treating a new URL as already-seen, so the URL gets skipped.

The interviewer then asked: "What's the false negative rate?" The candidate said "about the same — a few percent." This was wrong. Bloom filters have no false negatives by construction: if a URL was inserted, the filter always returns "possibly present." The error was misunderstanding the fundamental property of the data structure. The interviewer pushed: "So if your Bloom filter says 'seen,' is it possible the URL is actually new?" Candidate: "Yes." Interviewer: "And if it says 'not seen,' is it possible the URL was already crawled?" The candidate realized they had gotten it backwards.

**The lesson: Bloom filters have false positives (says "seen" when it's new — a real URL gets skipped) but zero false negatives (never says "not seen" for a URL that was inserted). Know this cold.**

---

### Story 3 — Amazon, L6 (2023)

The candidate gave a solid crawl architecture but proposed a simple BFS from seed URLs with no re-crawl mechanism. The interviewer asked: "Wikipedia updates 10,000 articles per day. How does your crawler discover those updates?"

The candidate's design would discover them only during the next full crawl cycle — 7 days later. The interviewer pushed: "A competitor search engine indexes Wikipedia changes within 4 hours. How would you achieve that?" The candidate added a "re-crawl service" but couldn't explain how it prioritized which URLs to re-crawl without crawling everything frequently (defeating the purpose of a 7-day crawl cycle).

The interviewer explained: track content_hash per URL. On each crawl, compare new hash to stored hash. If the hash changed, increment a "change_frequency" counter. URLs with high change frequency get scheduled for frequent re-crawls (every few hours); URLs that never change (old blog posts, static pages) get re-crawled infrequently (30–90 days). This is essentially a scheduling problem with a dynamic priority queue.

**The lesson: the re-crawl scheduler is a first-class component, not an afterthought. Initial crawl and re-crawl are different problems: initial crawl is a graph traversal; re-crawl is a scheduling problem driven by observed change frequency.**

---

## General Pattern Mistakes

**Mistake 1: Proposing a single global queue for all URLs.** A global FIFO queue has no priority mechanism and no per-domain rate limiting. To enforce politeness, you'd need to scan the queue for URLs from non-rate-limited domains — O(n) scan at scale. The correct answer is a two-tier structure: priority queues at the front, per-domain back queues enforce politeness naturally.

**Mistake 2: DNS as an afterthought.** DNS is mentioned last, or not at all. At 10,000 pages/sec, DNS is on the critical path of every single request. 50–120 ms cold DNS lookup × 10,000 requests/sec = you'd need 500–1,200 concurrent DNS requests. In-process async DNS with TTL-respecting LRU cache is mandatory. The Mercator crawler (1999) identified DNS as one of the five core pipeline bottlenecks — this is a 25-year-old known problem.

**Mistake 3: Bloom filter misuse.** Two failure modes: (1) Claiming both false positives and false negatives — Bloom filters only have false positives. (2) Over-fetishizing Bloom filters as always better than a DB index. For a focused crawler with fewer than 100M URLs, a DB index on URL hash is simpler, doesn't need a rebuild on restart, and has 0% false positive rate. Know when each is appropriate.

**Mistake 4: No crawler trap defense.** The web contains an unbounded number of URLs: calendar pages, session IDs, infinite redirect loops, dynamically generated content. Without defenses, the URL frontier grows without bound, the crawler gets stuck in traps, and storage fills. Address this: URL normalization, max crawl depth (15 hops), per-domain URL cap, content-hash deduplication.

**Mistake 5: Single-machine deduplication bottleneck.** All 200 worker machines routing every new URL through a single central Bloom filter service for dedup check. At 10,000 pages/sec × 15 extracted URLs per page = 150,000 dedup lookups/sec through one service. With domain-hash partitioning, each worker owns its domain's dedup — no cross-worker coordination needed.

---

## Self-Assessment Checklist

**Fundamentals:**
- [ ] Correctly identified the core tension: politeness vs. throughput, before starting design
- [ ] Proposed a two-tier URL frontier (front queues for priority + back queues for politeness)
- [ ] Named BFS as the traversal strategy and justified why (discovers high-in-degree pages early)

**Components:**
- [ ] Identified DNS caching as mandatory, with an estimate of cold vs. cached latency
- [ ] Proposed Bloom filter for deduplication with correct false-positive / zero-false-negative properties
- [ ] Described robots.txt cache with 24-hour TTL and behavior on 404 vs 503

**Scale:**
- [ ] Derived target QPS (5B pages ÷ 7 days = ~10,000 pages/sec)
- [ ] Calculated Bloom filter size for 1B URLs (~1.2 GB at 1% FP rate)
- [ ] Named domain-hash partitioning as the distribution strategy and explained DNS cache benefit

**Failures:**
- [ ] Addressed lease-based URL frontier (URLs re-queued on worker crash)
- [ ] Addressed crawler traps (max depth, per-domain cap, URL normalization)
- [ ] Addressed re-crawl scheduling (change frequency detection, priority queue)

---

## Remediation Targets

**If you didn't know the two-tier frontier:** Study the Mercator and UbiCrawler papers. The two-tier structure (priority front queues + politeness back queues) is the canonical academic answer to this problem and is expected at Google/Microsoft.

**If you mixed up Bloom filter false positive/negative:** Study Bloom filter mechanics. Bit array, k hash functions. Insert: set k bits. Query: if any bit is 0, definitely not present. If all k bits are 1, probably present (but might be a collision = false positive). False negative is structurally impossible.

**If you skipped re-crawl scheduling:** Think about it as a scheduling problem, not a crawl problem. You have a budget (can crawl N URLs/sec). You want to allocate that budget to maximize freshness across all URLs. URLs that change frequently get more budget. URLs that never change get almost none. This is a priority queue problem with dynamic priorities.

**If you ignored DNS:** Run the numbers. 10,000 pages/sec × 80ms average DNS (mix of cached/cold) = 800 seconds of DNS overhead per second = impossible without parallelism. Now understand why async DNS with caching is a fundamental design requirement, not an optimization.
