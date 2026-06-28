# Solution Guide — Web Crawler

Read after your attempt. If you haven't attempted yet, close this file.

---

## Component Map

| Component | Role | Technology Choice | Why |
|-----------|------|-------------------|-----|
| URL Frontier | Priority queue for next URL to crawl | Front queues (priority) + Back queues (one per domain) | Decouples priority from politeness; allows per-domain rate limiting |
| DNS Resolver | Hostname → IP mapping | In-process LRU cache + async resolver | Cold DNS: 50–120 ms; cached: 1–5 ms; mandatory at 10K req/sec |
| Fetcher Worker | HTTP GET, follow redirects, decompress | Go/C++ worker processes | High concurrency, low memory footprint; Go goroutines excel here |
| HTML Parser | Extract href links, resolve relative URLs | libxml2 / cheerio | Battle-tested; handles malformed HTML (common on the web) |
| URL Normalizer | Canonical form for URL deduplication | Custom logic | Prevents crawling `http://example.com` and `HTTP://EXAMPLE.COM/` separately |
| Seen-URL Filter | Deduplication: have we queued this URL? | Bloom filter in Redis + periodic DB flush | 1.2 GB covers 1B URLs at 1% FP vs. 32 GB for hash set |
| robots.txt Cache | Per-domain crawl rules | Redis with 24-hour TTL | One robots.txt fetch per domain per day; avoids per-URL overhead |
| Content Store | Stores raw HTML and metadata | S3 (content) + PostgreSQL (metadata) | Large blobs → object storage; metadata → structured queries |
| Re-crawl Scheduler | Assigns next crawl time per URL | Priority queue keyed on predicted change time | High-change pages re-crawled frequently; static pages rarely |

---

## Architecture Diagram

```
                         ┌─────────────────────────────────────────┐
                         │            URL FRONTIER                  │
                         │                                          │
                         │  Front Queues (priority tiers)           │
                         │  [HIGH] ─────────────────────┐           │
                         │  [MED]  ──────────────────────┤  Selector│
                         │  [LOW]  ──────────────────────┤  Policy  │
                         │                               │          │
                         │  Back Queues (one per domain) │          │
                         │  [example.com] ◀──────────────┘          │
                         │  [github.com]                            │
                         │  [news.ycombinator.com]                  │
                         └────────────────┬────────────────────────┘
                                          │ next URL to crawl
                                          ▼
┌──────────────────────────────────────────────────────────────────────┐
│                        FETCHER WORKER POOL                           │
│                                                                      │
│  1. Check robots.txt cache ──▶ Redis (robots.txt TTL: 24h)          │
│         (miss) ──▶ fetch robots.txt from domain                     │
│                                                                      │
│  2. DNS resolution ──────────▶ In-process LRU cache                 │
│         (miss) ──▶ upstream DNS (50-120 ms, cached after)           │
│                                                                      │
│  3. HTTP GET ─────────────────────────────────────────────────────▶ Web
│         ← 200 HTML, 301/302 redirect, 429/503 (back off), timeout    │
│                                                                      │
│  4. HTML parse + link extract                                        │
│                                                                      │
│  5. For each extracted URL:                                          │
│        Normalize → Check Bloom filter → (new) enqueue to Frontier   │
└────────────────────────────────┬───────────────────────────────────┘
                                 │
                    ┌────────────┴────────────┐
                    ▼                         ▼
         ┌──────────────────┐      ┌─────────────────────┐
         │   Content Store  │      │   Bloom Filter       │
         │   S3 (raw HTML)  │      │   Redis (seen URLs)  │
         │   PG (metadata)  │      │   ~1.2 GB / 1B URLs  │
         └──────────────────┘      └─────────────────────┘
                    │
                    ▼
         ┌──────────────────┐
         │  Index Consumer  │
         │  (downstream)    │
         └──────────────────┘


DISTRIBUTED CRAWLING (multiple workers):

  URL frontier shards by domain hash (consistent hashing)
  Worker 1 owns: example.com, github.com, ...
  Worker 2 owns: nytimes.com, reddit.com, ...
  Worker 3 owns: wikipedia.org, amazon.com, ...

  Cross-domain links: Worker 1 extracts link to nytimes.com
                      → route to Worker 2's frontier queue
                      → Worker 2 checks its own Bloom filter
```

---

## Capacity Math

**Throughput target:**
- 5 billion pages, crawl cycle = 7 days
- 5B ÷ 7 ÷ 86,400 = **~8,267 pages/sec** — call it 10,000 pages/sec for headroom

**Worker count:**
- Each worker: one HTTP fetch at a time, avg fetch+parse time ~500 ms
- To achieve 10,000 pages/sec: 10,000 × 0.5 sec = **5,000 concurrent workers**
- In practice, workers are async (goroutines): 200 workers × 50 concurrent goroutines each = 10,000 concurrent fetches

**Bandwidth:**
- 10,000 pages/sec × 25 KB average compressed HTML = **250 MB/sec inbound**
- Per year: 250 MB/s × 86,400 × 365 ≈ **7.9 PB/year** of fetched content

**Storage:**
- Raw HTML content: 5B pages × 25 KB = **125 TB compressed** per crawl cycle
- With 5 historical crawl cycles retained: **625 TB**
- Metadata per URL: ~200 bytes (URL, last-crawled, hash, content-type, status)
- 5B URLs × 200 bytes = **1 TB** for URL metadata DB

**Bloom filter sizing:**
- 1B unique URLs at 1% false positive rate: **1.2 GB** (9.6 bits/URL)
- 1B unique URLs at 0.1% false positive rate: **1.8 GB** (14.4 bits/URL)
- Compare: hash set of 1B 32-byte hashes = **32 GB** — 17–26× more memory
- Fits in Redis (max 512 MB default, configurable). Use Redis BITSET backed Bloom filter.

**robots.txt overhead:**
- 1.5 billion unique domains on the web, but only ~100M with significant traffic
- robots.txt per domain: ~2 KB average
- 100M domains × 2 KB = **200 GB** for robots.txt cache
- TTL: 24 hours; refresh 100M / 86,400 ≈ **1,157 domain robots.txt fetches/sec** added to load

**DNS cache:**
- 100M active domains; average TTL ~1 hour
- DNS cache entries: 100M × (hostname + IP = ~50 bytes) = **5 GB** — trivially fits in memory

---

## API Design

The crawler is primarily an internal data pipeline, not a public API. Key internal interfaces:

**URL Frontier — Enqueue:**
```
POST /frontier/enqueue
Body: { "url": "https://...", "priority": 0-9, "source_url": "https://..." }
Response 200: { "queued": true } or { "queued": false, "reason": "seen" }
```

**URL Frontier — Dequeue (worker pulls next URL):**
```
GET /frontier/next?worker_id=w42&domain_affinity=example.com
Response 200: { "url": "https://...", "domain": "example.com", "lease_id": "l_xyz" }
Response 204: No URL available for this worker
```

**Worker — Acknowledge completion:**
```
POST /frontier/ack
Body: { "lease_id": "l_xyz", "status": "success|failed|skipped", "new_urls": [...] }
```

**robots.txt Cache — Lookup:**
```
GET /robots?domain=example.com
Response 200: { "rules": [{ "path": "/private/*", "allow": false }], "crawl_delay_ms": 1000 }
```

---

## Data Model

**url_metadata table (PostgreSQL, sharded by domain_hash):**
```sql
CREATE TABLE url_metadata (
  url_hash      CHAR(32) PRIMARY KEY,        -- MD5 of normalized URL
  url           TEXT NOT NULL,
  domain        VARCHAR(255) NOT NULL,
  last_crawled  TIMESTAMP,
  last_modified TIMESTAMP,
  content_hash  CHAR(64),                     -- SHA-256 of content, for change detection
  crawl_depth   SMALLINT DEFAULT 0,
  http_status   SMALLINT,
  priority      SMALLINT DEFAULT 5,           -- 0 (highest) to 9 (lowest)
  next_crawl    TIMESTAMP,                    -- re-crawl scheduler sets this
  INDEX idx_next_crawl (domain, next_crawl)
);
```

**domain_policy table (politeness rules cache):**
```sql
CREATE TABLE domain_policy (
  domain          VARCHAR(255) PRIMARY KEY,
  robots_txt      TEXT,
  crawl_delay_ms  INT DEFAULT 1000,
  last_fetched    TIMESTAMP NOT NULL,
  is_blocked      BOOLEAN DEFAULT FALSE,      -- domain in robots.txt Disallow /
  INDEX idx_last_fetched (last_fetched)       -- for refresh scheduler
);
```

**DB type reasoning:** PostgreSQL for metadata because URL lookups need range queries (all URLs in a domain due next crawl), the schema is structured, and write volume (10K writes/sec) is manageable with sharding. Content stored in S3 because raw HTML blobs are large and query-opaque — object storage is the right abstraction.

---

## Key Design Decisions

### Decision 1: Two-Tier URL Frontier (Front + Back Queues)

| Dimension | Single Priority Queue | Two-Tier (front + back) |
|-----------|----------------------|-------------------------|
| **Politeness** | Must skip URLs to same domain that arrive too soon — O(n) scan | Back queue per domain enforces rate naturally |
| **Priority** | Simple | Front queues set priority; selector picks front queue, then back queue |
| **Memory** | One queue | More queues but each is small and domain-bounded |
| **Worker assignment** | Any worker can get any URL — DNS cache cold | Domain-to-worker affinity maximizes DNS cache hit rate |

**Choice: Two-tier.** Front queues carry priority (0 = news/high-PageRank, 9 = rarely-changing static pages). The selector picks from front queues based on a biased random walk (front queue 0 selected 50% of the time, queue 1 25%, etc.). Selected URL goes to the back queue for that domain. The back queue dispatcher enforces per-domain politeness — it only releases a URL from a domain's queue when the crawl-delay has elapsed since the last request to that domain.

**Trade-off accepted:** More complex implementation, more memory for queue metadata. Worth it because without back queues, you either violate politeness or burn CPU scanning for non-violating URLs.

---

### Decision 2: Bloom Filter vs. DB Index for Seen-URL Deduplication

| Dimension | Bloom Filter | DB Index on URL hash |
|-----------|-------------|----------------------|
| **Memory at 1B URLs** | 1.2 GB (1% FP rate) | ~32 GB (32-byte hash per entry) |
| **Lookup speed** | O(k) hash functions, sub-millisecond | O(log n) B-tree or O(1) hash index |
| **False positive rate** | ~1% (tunable) — skips a real new URL occasionally | 0% |
| **False negative rate** | Impossible — never misidentifies a seen URL as new | Impossible |
| **Persistence** | Must be rebuilt on restart | Durable by default |

**Choice: Bloom filter for primary deduplication, DB as backup.** At 1% FP rate, 1% of genuinely new URLs are skipped — acceptable. The critical property is no false negatives: we never re-crawl a page thinking it's new. DB-backed deduplication for the long-term record; Bloom filter is the hot path check that avoids DB hits for 99% of URL checks.

**Trade-off accepted:** ~1% of new URLs are skipped per crawl cycle due to false positives. For a search index, missing 1% of newly discovered URLs is tolerable. If false positives are unacceptable (e.g., financial data crawl), use a DB index at higher memory cost.

**Important nuance:** HelloInterview and other prep resources note that Bloom filters are "overkill" for many designs. A DB index on URL hash is simpler and operationally easier — know when each is appropriate.

---

### Decision 3: Domain-Based URL Partitioning Across Workers

| Dimension | Round-Robin URL Assignment | Domain-Hash Partitioning |
|-----------|---------------------------|--------------------------|
| **Politeness enforcement** | Every worker must coordinate on per-domain rate limits | Each domain owned by one worker — no coordination |
| **DNS cache efficiency** | Worker A fetches example.com, then Worker B fetches example.com — both pay DNS | Worker A owns all example.com URLs — DNS cached locally |
| **Failure isolation** | One worker slowdown affects all domains | Slow domain slows one worker only |
| **Load balancing** | Even by URL count | Uneven — large domains (wikipedia.org) overwhelm one worker |

**Choice: Domain-hash partitioning with overflow handling.** Consistent hashing assigns domains to workers. For hot domains (Wikipedia, GitHub) that generate more load than one worker can handle, split by URL path prefix within the domain (e.g., `/en/` → worker A, `/fr/` → worker B).

**Trade-off accepted:** Load imbalance — large domains concentrate work. Mitigated by splitting hot domains across multiple workers by path prefix. This adds coordination complexity but prevents worker starvation.

---

## Deep Dive: Politeness Enforcement

Politeness is the most underappreciated component of a web crawler. Getting it wrong leads to one of two bad outcomes: being IP-blocked by web servers, or causing actual harm to small sites that can't handle aggressive crawl rates.

**The mechanism:**

robots.txt specifies crawl rules per User-Agent. The crawler fetches `https://domain.com/robots.txt` at the start of crawling any new domain, parses it, and caches the result for 24 hours. The critical fields are `Disallow` (don't fetch this path) and `Crawl-delay` (minimum seconds between requests).

The back queue enforces politeness structurally: each domain has its own back queue, and the queue dispatcher tracks `last_request_time` per domain. It only releases the next URL from a domain's queue when `now - last_request_time >= crawl_delay`. If `crawl_delay` is not specified in robots.txt, the default is 1 second (conservative).

**Behavioral differences across crawlers:**
- Googlebot does NOT respect `Crawl-delay` at all. It uses adaptive throttling based on server response times (if the server is slow to respond, Googlebot backs off). To reduce Googlebot's crawl rate, site owners must use Google Search Console.
- Bingbot honors `Crawl-delay` strictly.
- For a new search engine crawler, honoring `Crawl-delay` is the ethical default — you don't yet have the relationship with site owners that Google has built over 25 years.

**Crawler traps:** Pages designed to generate infinite URLs trap naive crawlers. Common patterns:
- Calendar pages: `?date=2025-01-01`, `?date=2025-01-02`, … → infinite
- Session ID URLs: `?sessionid=abc123` → different URL, same content
- Infinite redirect loops

Mitigations:
- URL normalization: strip session IDs from query params before enqueueing
- Max crawl depth: don't follow links more than 15 hops from seed URLs
- Max URLs per domain: cap at 10M to prevent one domain consuming the entire frontier
- Content hash deduplication: if fetched content matches a previously seen hash, mark as duplicate even if URL is new (SimHash for near-duplicate detection)

**Re-crawl scheduling:** Not all pages change at the same rate. A news article changes once (when published). A live sports score changes every 30 seconds. The re-crawl scheduler tracks `change_frequency` per URL: compare `content_hash` across successive crawls, record how often it changes. Assign `next_crawl_time` using exponential backoff for static pages (if unchanged for 7 days, re-crawl in 14 days; if unchanged for 14 days, re-crawl in 30 days) and aggressive re-crawl for high-change pages.

---

## Failure Modes & Mitigations

| Failure | Impact | Detection | Mitigation |
|---------|--------|-----------|------------|
| Worker crash mid-crawl | URLs in-flight are lost | Heartbeat timeout | Lease-based frontier: URL has a lease_id with TTL; if worker doesn't ack within TTL, URL re-queued automatically |
| DNS resolution failure | Can't fetch URLs for affected domains | Fetch attempt returns NXDOMAIN | Retry with exponential backoff; after 5 failures, mark domain as unavailable for 24 hours |
| robots.txt unavailable | Don't know crawl rules | 404, 503, timeout on robots.txt fetch | 404 → assume no restrictions (permissive). 503/timeout → wait 1 hour before retry. Never crawl until robots.txt resolved. |
| Bloom filter restart/corruption | Re-crawl already-seen URLs | Service restart, data loss | Rebuild Bloom filter from url_metadata DB at startup. Accept brief window of duplicate crawls during rebuild. |
| Hot domain dominates frontier | One domain starves all others | Back queue depth imbalance metric | Per-domain max queue depth (10M URLs). Shard hot domains across multiple workers by path prefix. |
| Crawler trap (infinite URLs) | Frontier grows unbounded | Frontier size metric, URL count per domain | Per-domain URL cap + max crawl depth (15 hops from seed). SimHash content deduplication rejects near-duplicate pages. |
| 429 / 503 from target server | Being rate-limited or blocked | HTTP 4xx/5xx response | Exponential backoff with jitter: 1s, 2s, 4s, 8s... up to 1 hour cap. Reduce per-domain concurrency. |

---

## What Strong Candidates Do

1. **Explain the two-tier frontier unprompted** — name "front queues" for priority and "back queues" for politeness, explain the selector policy. This shows understanding of the core architectural challenge.
2. **Quantify the Bloom filter trade-off** — state that 1.2 GB covers 1B URLs at 1% FP vs. 32 GB for a hash set, and explain that false positives are acceptable (a skipped new URL) but false negatives are not.
3. **Justify domain-hash partitioning** — explain that it maximizes DNS cache hit rate (one worker owns all URLs for a domain) and eliminates cross-worker coordination for politeness.
4. **Address crawler traps explicitly** — calendar URLs, session IDs, and infinite redirects are real problems. Mention URL normalization, max depth, and per-domain URL caps.
5. **Distinguish re-crawl policy from initial crawl** — initial crawl is BFS from seeds; re-crawl is priority-scheduled based on change frequency. These are different problems requiring different mechanisms.
6. **BFS over DFS with reasoning** — BFS discovers high-in-degree (important) pages early. DFS can spend weeks in deep unimportant site structures. This choice matters.

---

## What Average Candidates Miss

1. **Conflating URL frontier with a simple FIFO queue.** A FIFO queue with no priority cannot favor high-value pages. A FIFO queue with no per-domain tracking cannot enforce politeness without CPU-intensive scanning.

2. **Ignoring DNS caching.** At 10,000 pages/sec with 120 ms uncached DNS lookups, the crawler would need 1.2 million concurrent DNS requests just to keep up. In-process DNS caching with async resolution is not optional — it's load-bearing.

3. **Per-IP not per-domain rate limiting.** Thousands of domains share a single IP address (shared hosting, CDNs). If you rate limit by domain but don't check the IP, you may be sending 1,000 req/sec to the same IP from 1,000 different domains. Rate limit should apply to both domain AND resolved IP.

4. **No fault tolerance for in-flight URLs.** A worker crash loses all URLs it was processing unless there's a lease mechanism. Candidates who say "worker picks from queue, processes it" without discussing what happens if the worker dies before acknowledging completion miss a critical reliability gap.

5. **Treating robots.txt as a one-time lookup.** robots.txt changes over time. A site that allowed crawling in January may disallow it in March. The 24-hour TTL cache with re-fetch is the correct pattern — not a one-time parse at crawler startup.
