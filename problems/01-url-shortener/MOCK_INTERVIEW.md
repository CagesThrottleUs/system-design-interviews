# Mock Interview Script — URL Shortener

**Target level:** Apple ICT3 / Google L5 / Amazon SDE II (Senior)
**Total time:** 45 minutes
**Format:** Read the interviewer lines aloud or silently. Fill in candidate responses with your actual answers. Score yourself at the end using the rubric.

---

## Interviewer Persona

You are a Staff Engineer who has built URL shorteners and high-throughput redirect systems in production. You do not spoon-feed. You push back on vague answers. You ask "how" and "why" after every high-level claim. You call out technical errors directly rather than hinting.

---

## Phase 1 — Problem Introduction (0:00–2:00)

**Interviewer:**
"Let's design a URL shortener — something like TinyURL or Bitly. Users can paste a long URL and get back a short one. Anyone who clicks the short link gets sent to the original URL. That's the basic idea. You have 45 minutes. Where do you want to start?"

*[Wait for candidate to respond. A strong candidate starts with requirements clarification. An average candidate starts drawing boxes.]*

**If candidate starts drawing immediately, interrupt:**
"Before we get into the design, I'd like to make sure we understand the problem. What do you want to know about the scope?"

---

## Phase 2 — Requirements Clarification (2:00–7:00)

*[Let the candidate ask questions. Use the responses below. Do not volunteer information unprompted.]*

**If asked about scale:**
"Think of something like Bitly. Hundreds of millions of links created per month. Many more redirects — assume a 100-to-1 read-to-write ratio."

**If asked about analytics:**
"Yes, link creators need to see click counts. Time-series breakdown by day would be useful. We don't need real-time; a few minutes of lag is fine."

**If asked about custom aliases:**
"Nice to have. Not MVP, but the design shouldn't prevent it."

**If asked about link expiry:**
"Yes, links should support an expiry date set at creation."

**If asked about authentication:**
"Assume link creators are authenticated users. Redirects are public — no auth required to click a link."

**If asked about global vs. regional:**
"Global product. Users everywhere."

**If candidate doesn't ask about redirect caching or 301 vs 302:**
*[Do not prompt. Note this gap. Score it after the session.]*

**If candidate doesn't ask about analytics before designing:**
*[Note this gap. It will surface when they propose synchronous click counting.]*

**After 4–5 minutes of requirements, if candidate hasn't moved on:**
"Sounds like you have a good picture. What's your capacity estimate before we design?"

---

## Phase 3 — Capacity Estimation (7:00–12:00)

*[Listen for the candidate doing the math out loud. Push for numbers.]*

**If candidate skips estimation or says "we can assume it scales":**
"Walk me through the numbers. How many redirects per second do we need to handle?"

**Expected responses (accept within ~2× of these):**
- Write QPS: ~38–100 writes/sec
- Read QPS: ~3,000–4,000 avg, ~100K peak
- 5-year storage: ~3–15 TB
- Read:write ratio: ~100:1

**If candidate gets the read:write ratio:**
"Good. What does that ratio tell you about where you need to invest in the design?"

*[Looking for: "Cache is load-bearing, not optional." If they say "we should add a cache," but don't state it's mandatory, push:]*
"Is the cache optional? What's the p99 redirect latency without it?"

**If candidate doesn't do napkin math:**
"What's your peak redirect QPS? I want a number."

*[If they can't produce one: "Let me help you anchor this. 10 billion clicks per month divided by 30 days divided by 86,400 seconds. What do you get?"]*

---

## Phase 4 — High-Level Design (12:00–30:00)

*[Candidate should be drawing the system. Ask probing questions throughout.]*

### On short code generation:

**After candidate mentions short codes:**
"How do you generate the short code? Walk me through the algorithm."

**If candidate says MD5 / hash:**
"What happens when two long URLs produce the same 8-character hash prefix? How often does that happen at 6 billion links?"

*[If they can't answer: "Calculate it. 16^8 is about 4 billion possible values. You have 6 billion links. What does that tell you?"]*

"Now your generation algorithm requires checking the DB before every insert. What's the latency impact under high write load?"

**If candidate proposes counter + Base62:**
"Good. How do you make the counter highly available? If that single counter goes down, what happens to link creation?"

### On redirect:

**After candidate proposes the redirect endpoint:**
"What HTTP status code do you return on redirect, and why?"

**If candidate says 301:**
"The PM says we need click analytics. Walk me through what happens on the second click from the same browser after a 301."

*[Wait for candidate to work through the browser caching implication. Do not explain it.]*

"So does your analytics system ever see that second click?"

**If candidate corrects to 302:**
"Good. Any performance implications of 302 vs 301?"

*[Looking for: "Every click hits the server. That's why the cache is critical — we handle it in Redis, not the DB."]*

### On the cache:

**After candidate mentions Redis:**
"How big is your cache? What's your eviction policy? How do you handle a viral link that suddenly gets 100K QPS from cold?"

"What happens when Redis is down? Does the redirect still work?"

*[Looking for: "Fall through to DB. Redis is a performance layer, not a hard dependency. But at 100K QPS, DB alone can't sustain the latency SLA — need circuit breaker and auto-scaling buffer."]*

### On analytics:

**After candidate mentions click counting:**
"Walk me through what happens on the hot path when a user clicks. Is the analytics write synchronous with the 302 response?"

**If candidate proposes synchronous DB write per click:**
"10 billion clicks per month. That's about 4,000 writes per second on average to your analytics store, synchronized with every redirect. What's the latency impact on the redirect path?"

*[If they don't see it: "Your redirect path now has two serial writes: one to the URL read (cache/DB) and one to the analytics store. Under 100K QPS, what happens to your p99?"]*

**If candidate proposes async Kafka pipeline:**
"Good. What's the delivery guarantee? If a Kafka broker goes down mid-click, do we lose that event? Is that acceptable?"

*[Looking for: acknowledgment of at-least-once vs exactly-once, and statement that approximate analytics is acceptable per NFR.]*

### On the ID generator:

"You mentioned a distributed counter for ID generation. Walk me through the HA design for that component. What happens during failover?"

*[Looking for: Snowflake-style (multiple nodes, machine ID in the bit layout), or range-based pre-allocation to app servers.]*

---

## Phase 5 — Low-Level Design (30:00–40:00)

*[Shift to data model and specific algorithms.]*

**Data model:**

"Show me your primary data table. What are the fields, the partition key, and why that DB type?"

*[Looking for: shortCode as partition key, longUrl, userId, expiresAt, isActive. DB choice justified by KV access pattern — no joins.]*

"Where does your analytics data live? Same table or separate?"

*[Looking for: separate store, time-series optimized, ClickHouse or Cassandra.]*

**Short code namespace:**

"How many unique short codes can you generate with 7 Base62 characters? Is that enough for 5 years of link creation at your stated volume?"

*[62^7 ≈ 3.5 trillion. 6 billion links over 5 years. 3.5T >> 6B — plenty of headroom.]*

**Custom alias:**

"A user wants the alias /apple instead of a generated code. What changes in your create flow?"

*[Looking for: attempt DB/cache lookup for custom alias, return 409 Conflict if taken, otherwise write it like any link.]*

**Expiry:**

"A link expires at midnight. A user clicks it at 12:00:01 AM. Your cache still has it. What happens?"

*[Looking for: Cache TTL ≤ link expiry time. After expiry, cache miss falls to DB which checks expiresAt. Return 410 Gone. Brief window where cache still serves expired link — acceptable or add explicit invalidation.]*

---

## Phase 6 — Failure Modes (40:00–43:00)

"Let's do a quick failure mode sweep. For each component you've drawn, what's the failure mode and your mitigation?"

*[Prompt through: API servers, Redis, DynamoDB, Kafka, ID generator, analytics store.]*

*[A strong candidate answers without needing to be prompted for each one.]*

---

## Phase 7 — Wrap-up (43:00–45:00)

"Two minutes left. What's the weakest part of your design?"

*[Listen for honest self-assessment. Weak candidates claim the design is solid. Strong candidates name: viral link cold-cache stampede, analytics exactly-once delivery, ID generator HA under rapid-failover.]*

"If you had another 30 minutes, what would you add?"

*[Looking for: CDN caching for the hottest links, pre-warming cache for scheduled campaigns, URL scanning pipeline for phishing/malware, rate limiting on creation endpoint.]*

---

## Scoring Rubric

Score each category 1–5 after the session. 1 = not addressed, 3 = addressed with gaps, 5 = addressed proactively and correctly.

### Requirements Clarification (10% of total)

| Criterion | Score (1–5) |
|-----------|-------------|
| Asked about scale / QPS | |
| Asked about analytics (count vs time-series vs nothing) | |
| Asked about custom aliases | |
| Confirmed 302 vs 301 implications (or raised proactively) | |
| Stated MVP scope before designing | |

**Category score: ___/5**

---

### High-Level Design Completeness (30% of total)

| Criterion | Score (1–5) |
|-----------|-------------|
| Stateless API servers behind load balancer | |
| Collision-free short code generation (not MD5-only) | |
| KV-optimized database (not just "a database") | |
| Redis cache as load-bearing (not optional) | |
| 302 redirect (not 301), with explanation | |
| Async analytics pipeline (not synchronous per-click write) | |
| ID generator HA (not single Redis INCR) | |

**Category score: ___/5**

---

### LLD Depth (30% of total)

| Criterion | Score (1–5) |
|-----------|-------------|
| Primary table fields, types, and partition key | |
| DB choice justified by access pattern (write-once, read-many, no joins) | |
| Analytics store separate from URL table | |
| 7-char Base62 namespace math (62^7 ≈ 3.5T) | |
| Expiry cache-invalidation race condition addressed | |
| Custom alias conflict path | |

**Category score: ___/5**

---

### Tradeoff Reasoning (20% of total)

| Criterion | Score (1–5) |
|-----------|-------------|
| Named alternative to chosen short code strategy and why it was rejected | |
| Named 301 consequence on analytics explicitly | |
| Cache sizing — gave a number, not just "add Redis" | |
| Named failure impact of each critical component | |
| Tradeoff between analytics delivery guarantee and redirect latency | |

**Category score: ___/5**

---

### Communication Clarity (10% of total)

| Criterion | Score (1–5) |
|-----------|-------------|
| Followed estimation → HLD → LLD sequence | |
| Named each component before explaining it | |
| Proactively stated tradeoffs without prompting | |
| Corrected own errors gracefully when challenged | |
| Stayed within time allocations per phase | |

**Category score: ___/5**

---

### Final Score

```
Requirements (10%):    ___/5 × 0.10 = ___
HLD (30%):             ___/5 × 0.30 = ___
LLD (30%):             ___/5 × 0.30 = ___
Tradeoffs (20%):       ___/5 × 0.20 = ___
Communication (10%):   ___/5 × 0.10 = ___

TOTAL: ___/5.0
```

**Interpretation:**
- 4.5–5.0: Strong hire at target level. Likely promoted above level.
- 3.5–4.4: Hire at target level. Standard.
- 2.5–3.4: Possible hire at one level below. Gaps to address before retrying.
- 1.5–2.4: No hire. Significant gaps in fundamental reasoning.
- Below 1.5: No hire. Return to curriculum foundations.

---

## Interviewer Notes Template

```
Date: ___________
Candidate level attempt: ___________

Strong moments:
-
-
-

Gaps observed:
-
-
-

Single most important thing to fix before next attempt:


Overall: ___/5.0
```
