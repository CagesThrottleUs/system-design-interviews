# Mock Interview Script — Web Crawler

**For the interviewer only. Do not share with the candidate.**

---

## Pre-Session Setup

**Target level:** Senior engineer (L5/ICT3 equivalent)
**Expected duration:** 45 minutes
**Red flags:** Single global URL queue, ignoring DNS, Bloom filter false-negative misunderstanding, no crawler trap defense, no re-crawl mechanism
**Green flags:** Two-tier frontier unprompted, BFS with justification, domain-hash partitioning for DNS efficiency, lease-based fault tolerance, re-crawl scheduling by change frequency

---

## Opening

"I'd like you to design a web crawler. The goal is to crawl the entire web — about 5 billion pages — to power a new search engine. You should crawl all 5 billion pages within roughly 7 days, and re-crawl high-priority pages within 24 hours of their changes."

*Let the candidate ask clarifying questions. Don't volunteer information.*

---

## Phase 1: Requirements (0–8 min)

**Expected clarifying questions:**
- Full web vs. focused domain? (Good — affects scope dramatically)
- JavaScript rendering? (Good — JS requires headless browser, 10–50× more expensive)
- Politeness requirements? (Critical — the core tension)
- What counts as a duplicate URL? (Good — drives dedup strategy)

**If candidate doesn't ask about politeness, prompt:** "One of the sites we're crawling is a small blog running on a shared hosting plan that can handle 2 requests per second. How does your design handle that?"

**If candidate doesn't ask about scale, prompt:** "What scale are you designing for? How did you arrive at that number?"

---

## Phase 2: Estimation (8–13 min)

**Let candidate work through math, then fill gaps:**

Expected numbers:
- 5B pages ÷ 7 days ÷ 86,400 sec = **~8,300 pages/sec** (round to 10,000)
- Average page: 100 KB uncompressed, 25 KB compressed
- Total content storage: 5B × 25 KB = **125 TB per cycle**
- Bloom filter for 1B URLs at 1% FP: **~1.2 GB** (worth deriving this)

**Key probe (if candidate proposes hash set):** "Your URL dedup table has 1 billion entries, each a 32-byte hash. What's the memory footprint?" Answer: 32 GB. Then: "Compare that to a Bloom filter at 1% false positive rate." Answer: 1.2 GB. "What's the trade-off you're accepting?"

---

## Phase 3: HLD (13–28 min)

**Expected components:**
1. URL Frontier (probe if it's a simple FIFO — it shouldn't be)
2. Fetcher workers (probe for async/concurrent design)
3. DNS resolver (probe if missing — it's often missed)
4. robots.txt cache
5. Content store + metadata DB

**Critical probe — URL frontier:** "Walk me through your URL frontier in detail. How does it enforce politeness? How does it prioritize news articles over static pages?"
- If candidate says "priority queue with per-domain rate limiter via Redis": probe the Redis bottleneck (200 workers × 10K req/sec = 10K Redis writes/sec just for rate limiting)
- Expected answer: two-tier design with back queues per domain

**Critical probe — DNS:** "Your system fetches 10,000 pages/sec. DNS resolution is 80ms uncached. Walk me through the DNS lookup path for a URL."
- Expected: in-process DNS cache, async resolver, TTL-respecting LRU

**Critical probe — distributed design:** "You have 200 worker machines. URL A on machine 1 links to a URL on example.com. URL B on machine 150 also links to a URL on example.com. How do you ensure the politeness limit for example.com is enforced across both machines?"
- Expected: domain-hash partitioning → all example.com URLs go to one worker → no cross-worker coordination needed

---

## Phase 4: Deep Dive (28–40 min)

Choose based on candidate's weaker area:

**Option A — Fault Tolerance:**
"Your worker is processing a URL when the process crashes. Walk me through exactly what happens: Is that URL lost? Does it get re-crawled? How do you prevent crawling it twice?"
Expected: lease-based frontier (URL has a TTL lease; worker acks on completion; lease expires on crash → URL re-queued)

**Option B — Crawler Traps:**
"Walk me through a site that has a calendar application at `/events?date=YYYY-MM-DD`. The page for any date links to the pages for the day before and the day after. How does your crawler handle this?"
Expected: URL normalization, max crawl depth, per-domain URL cap, SimHash for near-duplicate content

**Option C — Re-crawl Policy:**
"Wikipedia publishes 10,000 article updates per day. Your crawler completes a full cycle in 7 days. How does your crawler discover Wikipedia updates faster than once per week?"
Expected: content-hash comparison on each crawl, change frequency tracking, priority queue for re-crawl with high-change pages scheduled frequently

---

## Phase 5: Failure & Scaling (40–44 min)

**Probe:** "Your Bloom filter service crashes and you lose all state. What happens, and how do you recover?"
Expected: Rebuild from url_metadata DB at startup. Accept a brief period of duplicate crawl requests during rebuild. The Bloom filter is an optimization, not the source of truth.

**Probe:** "You've been asked to crawl GitHub.com specifically. GitHub has 100 million public repositories, each with potentially thousands of files. How does your per-domain URL cap policy need to change?"
Expected: acknowledgement that per-domain caps need to be configurable. High-value domains (GitHub, Wikipedia) warrant higher caps. The cap is a safety valve against traps, not a business rule.

---

## Closing

"Two questions:
1. What is the weakest assumption in your design?
2. If a competitor search engine was getting Wikipedia content 4 hours after publication, and you're getting it 7 days after, what specific component would you change first?"

---

## Scoring Checklist

| Area | Weight | 1 (Poor) | 3 (OK) | 5 (Strong) |
|------|--------|----------|--------|------------|
| Requirements clarification | 10% | Skipped straight to design | Asked about scale and politeness | Asked about JS rendering, re-crawl freshness, dedup semantics |
| HLD completeness | 30% | Single FIFO queue, no DNS | Priority queue, DNS mentioned | Two-tier frontier, domain-hash partitioning, DNS caching quantified |
| LLD depth | 30% | No Bloom filter details, no crawler trap defense | Bloom filter present, some trap awareness | Bloom filter sizing math, lease-based fault tolerance, re-crawl scheduling by change frequency |
| Tradeoff reasoning | 20% | Choices made without justification | Some tradeoffs named (BFS vs DFS) | Bloom filter vs hash set memory math, domain partitioning politeness coordination trade-off, BFS justified |
| Communication clarity | 10% | Rambling | Logical but verbose | Clear pipeline description, proactively raised failure modes |
