# Web Crawler

**Difficulty:** Intermediate
**Category:** Distributed systems, Data pipeline
**Time Box:** 45 min
**Key Concepts:** URL frontier with priority queues, politeness enforcement, Bloom filter deduplication, distributed URL partitioning, re-crawl scheduling
**Asked at:** Google, Amazon, Microsoft/Bing

---

## Problem Statement

Your team is building a web crawler to power a new search engine. The crawler must continuously discover, fetch, parse, and index web pages at web scale — not a few thousand pages, but billions. The internet contains an estimated 5 billion pages, new content is published every second, and existing pages change at unpredictable rates. The crawler must find the new content quickly, respect site owners' crawl preferences, and not overload any individual web server in the process.

The engineering challenge is not the happy path — fetching a page and extracting links is trivial. The challenge is scale, correctness, and the fundamental tension between two competing goals: **throughput** (crawl as many pages as possible) and **politeness** (never hammer a single server faster than it can handle). Every design decision in a crawler ultimately traces back to this tension.

For reference: Google's Caffeine indexing system, deployed in 2010, delivers results that are 50% fresher than the previous batch-based system. Caffeine processes hundreds of thousands of pages per second and stores roughly 100 petabytes of data. Common Crawl, the open non-profit crawler, downloads approximately 2.7 billion pages per monthly crawl, producing 420 terabytes of uncompressed data per run.

---

## Actors

| Actor | Description | Primary Action |
|-------|-------------|----------------|
| **Seed Provider** | Initial set of high-value URLs to bootstrap the crawl | Provides the starting URL list |
| **Crawler Worker** | Fetches a URL, parses HTML, extracts links | Consumes from URL frontier, produces new URLs |
| **URL Frontier** | Priority queue managing which URL to crawl next | Schedules URL delivery to workers |
| **Web Servers (external)** | Sites being crawled | Serve pages, publish robots.txt |
| **Index Consumer** | Downstream service that processes crawled content | Reads from crawl output store |

---

## Functional Requirements

**Core (MVP):**
- FR1: Given a seed list of URLs, crawl and download the HTML content of each reachable page
- FR2: Parse each crawled page to extract new URLs, add them to the crawl queue
- FR3: Never crawl the same URL twice in the same crawl cycle (deduplication)
- FR4: Respect robots.txt: honor Disallow directives and crawl-delay hints
- FR5: Never send more than N requests per second to a single domain

**Secondary:**
- FR6: Prioritize high-value pages (high PageRank, recently changed) over low-value ones
- FR7: Re-crawl pages periodically based on their observed change frequency
- FR8: Detect and skip near-duplicate content (same content, different URLs)
- FR9: Handle JavaScript-rendered pages (headless browser for JS-heavy sites)

---

## Non-Functional Requirements

| Requirement | Target | Rationale |
|-------------|--------|-----------|
| Crawl throughput | > 10,000 pages/sec | At 10K pages/sec, 5B pages take ~6 days — one full crawl cycle |
| Politeness | ≤ 1 req/sec per domain (default) | Respecting crawl-delay avoids being IP-banned and avoids harming sites |
| URL deduplication accuracy | < 0.01% false positive rate | Skipping a real new page is acceptable; crawling the same page 1,000 times is not |
| DNS lookup latency | < 5 ms (cached) | At 10K pages/sec, 120 ms DNS per request = impossible; caching is mandatory |
| Storage for crawled content | ~5 PB | 5B pages × 100 KB average compressed HTML |
| Freshness latency | < 24 hours for top-tier pages | Breaking news must be discoverable same-day |
| Fault tolerance | Worker failure must not lose URLs | URLs in flight must be re-queued on worker crash |

---

## Capacity Estimation Hints

Use these to derive your own numbers before reading the solution guide:

- **Web scale:** 5 billion pages to crawl
- **Average page size:** 100 KB uncompressed HTML (after compression: ~25 KB)
- **Average outlinks per page:** 15–20 URLs extracted per page
- **Crawl cycle target:** Complete a full crawl cycle in ~7 days
- **robots.txt cache TTL:** 24 hours (refetch daily per domain)
- **Domains seen:** approximately 1.5 billion unique registered domains in the global URL space

Derive: target QPS, worker count, bandwidth, storage for content, storage for URL dedup structure, number of URLs in frontier.

---

## Good Clarifying Questions to Ask

### Scope
1. Is this a focused crawler (specific domains/topics) or a full web crawler? (Dramatically different scale and architecture)
2. Do we need to crawl JavaScript-rendered content, or is raw HTML sufficient? (JS requires headless browsers — 10–50× more expensive per page)
3. What's our crawl frequency target? Crawl everything once, or continuously re-crawl changing pages? (Re-crawl policy changes the URL frontier design significantly)

### Scale
4. How many pages do we need to index total, and what's the target freshness for tier-1 pages? (Determines QPS target and freshness scheduling logic)
5. Do we need to handle ~50 languages and international character sets in URLs? (URL normalization becomes complex with IDNs)

### Politeness
6. How aggressively should we honor crawl-delay — should we follow it strictly, or should we adapt based on server response times? (Googlebot ignores crawl-delay entirely; Bingbot respects it)
7. What's our policy for sites that don't have a robots.txt? (Default: crawl with conservative rate limiting)

### Deduplication
8. If two URLs have identical content (canonical duplicates), do we want to store both or deduplicate? (Drives whether you need content hashing beyond URL hashing)
9. How much false-positive URL skipping is acceptable? (Determines Bloom filter size vs. DB-backed seen-set)

### Compliance
10. Do we need to handle robots.txt `noindex` directives (don't index, but still allowed to crawl), or is this purely a crawl decision tool? (Affects downstream pipeline, not just crawler)

---

## Key Components

<details>
<summary>Reveal components</summary>

- **URL Frontier** — the priority queue that determines what to crawl next; splits into front queues (priority-based) and back queues (one per domain, enforces politeness)
- **DNS Resolver** — in-process caching DNS resolver; resolving hostnames is 50–120 ms cold; must be cached
- **Fetcher** — makes HTTP GET requests; handles redirects, timeouts, gzip decompression, encoding detection
- **HTML Parser / Link Extractor** — parses fetched HTML, extracts anchor hrefs, resolves relative URLs to absolute
- **URL Normalizer** — canonicalizes URLs (lowercase hostname, remove default ports, sort query params, decode %XX)
- **Seen-URL Filter** — deduplication structure; Bloom filter for memory efficiency, or DB-backed for accuracy
- **robots.txt Cache** — per-domain cache of parsed Disallow rules; refresh every 24 hours
- **Content Store** — stores raw crawled HTML + metadata; blob storage (S3/GCS) for content, DB for metadata
- **Scheduler / Re-crawl Planner** — assigns re-crawl priority based on observed change frequency

</details>
