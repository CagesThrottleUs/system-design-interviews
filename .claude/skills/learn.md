# /learn — System Design Learning Skill

## Purpose
Deliver structured system design lessons, track progress, record knowledge gaps discovered during learning, automatically detect when mock interviews are unlocked, and schedule them. When all lessons are complete, add a new mock interview instead of stopping.

---

## On Invocation

### Step 1 — Read State

Read both files before doing anything else:

```
curriculum/progress.md   — current lesson, completed lessons, identified gaps
curriculum/00_overview.md — full curriculum map and interview gate definitions
```

### Step 2 — Determine Mode

| Invocation | Mode |
|------------|------|
| `/learn` (no args) | Deliver next lesson from progress.md |
| `/learn <topic>` (e.g., `/learn APNs`) | Teach that specific topic |
| `/learn review` | Quiz on the last completed lesson |
| `/learn gaps` | Show all recorded knowledge gaps and their status |
| `/learn schedule` | Show which interview gates are unlocked and suggest scheduling |

If the next lesson in progress.md is a **Layer 3 archetype** (L3-xx), do not teach it as a lesson — run it as a mock interview instead (strict mode, no hints, scored). Say: "L3 archetypes are mock interviews. Invoking `/interview` for L3-01 now." and invoke the interview skill.

If **all lessons are complete** (every L1, L2, and scheduled L3 entry in progress.md is marked ✅), skip to the **Curriculum Complete** section at the bottom of this skill.

---

### Step 3 — Research (Mandatory)

Spawn a research subagent before writing the lesson. The subagent should gather:
- Official documentation for the technology (e.g., Kafka docs, Apple APNs docs)
- Engineering blog posts from companies who've implemented this at scale (with numbers and dates)
- Current adoption stats or benchmark data from the last 2 years

Use the Lesson Content Guide at the bottom of this skill for what to search per lesson. Do not rely on training knowledge for statistics — verify with search.

---

### Step 4 — Deliver Lesson as LaTeX PDF

Follow the **Curriculum Writing Standard** in `CLAUDE.md` exactly. Key requirements:
- Write narrative prose — not bullets as primary content
- Explain WHY before WHAT
- Use `\thinkbox` for embedded curiosity prompts
- Use `\industrybox` for real company data with numbers and years
- Use `\trapbox` for common interview mistakes
- Curiosity Corner with ≥5 deep Q&A entries
- Self-quiz with 5 interviewer-probe questions and model answers

Compile with `pdflatex -interaction=nonstopmode` and open the PDF immediately after.

Save to: `curriculum/layer[N]-[name]/L[N]-[NN]-[topic].tex`

---

### Step 5 — Gap Recording (During and After Lesson)

As you write and deliver the lesson, track concepts the user struggled with or that revealed adjacent gaps. At the end of the lesson, explicitly ask:

> "Did anything in this lesson feel unclear or raise a question you couldn't answer? I'll record it as a gap to revisit."

Record all gaps — both ones identified during teaching and ones the user flags — in `curriculum/progress.md` under **Identified Knowledge Gaps** in this format:

```markdown
| [Gap description] | [Severity: Critical/High/Medium/Low] | [Covered By: lesson code] | ☐ |
```

**Severity guide:**
- **Critical** — caused a wrong answer in a past session, or would cause a fundamentally broken design
- **High** — would cause a significant design flaw at senior level
- **Medium** — polish and depth gap; noticeable but not fatal
- **Low** — nice-to-have; wouldn't affect score meaningfully

Also check: does this lesson reveal that a previously recorded gap is now closed? If the user demonstrates understanding of a gap's concept during this lesson, mark it ✅ in progress.md.

---

### Step 6 — Update Progress

Update `curriculum/progress.md`:
1. Move the just-completed lesson to the **Completed Lessons** table with today's date
2. Update **Next Lesson** to the following lesson in the recommended order
3. Add any new gaps discovered (Step 5)
4. Mark any gaps now closed as ✅

---

### Step 7 — Gate Check (Run After Every Lesson)

After updating progress, check whether any interview gate has just been unlocked.

Read the completed lessons list from progress.md. Cross-reference against these gates:

**Gate 1 — Notification System Retry**
Prerequisites: L1-01 ✅, L1-02 ✅, L1-08 ✅
Unlocks: L3-01 (Notification System)

**Gate 2 — URL Shortener, Rate Limiter**
Prerequisites: Gate 1 complete + L1-05 ✅, L1-06 ✅, L1-07 ✅, L1-11 ✅
Unlocks: L3-02 (URL Shortener), L3-04 (Rate Limiter)

**Gate 3 — Social Systems**
Prerequisites: Gate 2 complete + L1-03 ✅, L1-04 ✅, L1-10 ✅, L2-01 ✅, L2-02 ✅
Unlocks: L3-03 (Social News Feed), L3-05 (Web Crawler), L3-09 (Chat System)

**Gate 4 — Storage and Search**
Prerequisites: Gate 3 complete + L1-09 ✅, L2-05 ✅, L2-06 ✅, L2-10 ✅
Unlocks: L3-06 (Search Autocomplete), L3-07 (Distributed Cache), L3-08 (File Storage)

**Gate 5 — Advanced Systems**
Prerequisites: Gate 4 complete + L1-12 ✅, L1-13 ✅, L2-04 ✅, L2-08 ✅, L2-09 ✅, L2-11 ✅
Unlocks: L3-10 (Video Streaming), L3-11 (Ride Sharing), L3-12 (Payment), L3-13 (Ticketing)

**Gate 6 — Modern Systems**
Prerequisites: Gate 5 complete + L1-14 ✅, L2-12 ✅, L2-13 ✅
Unlocks: L3-14 (Recommendation), L3-15 (Task Scheduler), L3-16 (Location), L3-17 (Leaderboard), L3-18 (Collaborative Editing), L3-19 (Live Streaming)

**If a gate just became fully satisfied** (all prerequisites were already ✅ before this lesson, and this lesson's completion made the last one tick), do all of the following:

1. **Announce the unlock** clearly in chat:
   ```
   🔓 Gate [N] Unlocked: [Gate Name]
   You've completed the prerequisites for: [list of unlocked archetypes]
   
   Suggested next step: schedule a mock interview for [primary archetype].
   When you're ready, invoke `/interview [topic]` — e.g., `/interview notification system`.
   Continue learning? Next lesson: [next lesson code and title].
   ```

2. **Add the unlocked interviews to progress.md** under a new section called **Scheduled Interviews** (create it if it doesn't exist):
   ```markdown
   ## Scheduled Interviews
   
   | Gate | Question | Unlocked | Status |
   |------|----------|----------|--------|
   | Gate 1 | L3-01 Notification System | 2026-04-21 | ☐ Pending |
   ```

3. **Add the L3 archetype entries to 00_overview.md** Layer 3 table, updating their status from ☐ to 🔓 (unlocked but not yet attempted).

Do not add interviews to the scheduled list if they were already unlocked in a previous session.

---

### Step 8 — Post-Interview Gap Recording

This step runs after a mock interview completes (not during a lesson). When Laksh finishes an interview session:

1. Read the session file from `interviews/<company>/<role>/session_NNN.md`
2. Extract all weaknesses, gaps, and probe questions that were answered poorly
3. For each gap found, add it to `curriculum/progress.md` under **Identified Knowledge Gaps** with:
   - A description of the specific gap (not "was unclear on caching" but "couldn't explain cache invalidation under write-heavy load with Redis Cluster")
   - Severity based on how badly it affected the design
   - The lesson code that covers it (check 00_overview.md for which lessons cover which concepts)
   - Status ☐
4. If the gap maps to a lesson not yet completed, ensure that lesson appears in the recommended order before the next interview of the same type

---

## Curriculum Complete Mode

Triggered when: every L1, L2, and all currently scheduled L3 entries in progress.md are marked ✅.

Do NOT say "you've completed everything." The curriculum is never finished — mastery requires breadth (new questions) and depth (retrying weak areas under fresh constraints).

### Step 1 — Diagnose the current state

Read:
- `curriculum/progress.md` — Scheduled Interviews table (what's been done, scores, dates)
- `interviews/` directory — all session files, scores, and gap notes
- `curriculum/00_overview.md` — current archetype list and status

Build a picture of: which archetypes are weakest by score, which haven't been attempted recently (>30 days), and which gaps are still open.

### Step 2 — Schedule a refresher (retry a weak archetype)

Pick the archetype that meets the most of these criteria:
- Lowest score across all attempts
- Most open gaps in progress.md that map to this archetype's foundations
- Longest time since last attempt

Add a retry entry to progress.md under Scheduled Interviews:

```markdown
| Refresh | L3-01 Notification System (retry #2) | 2026-04-21 | ☐ Pending |
```

Retries are not the same as original attempts. For a retry, the interviewer should:
- Add a constraint twist that wasn't in the original (e.g., "now design it for 1 billion users" or "assume APNs is unavailable, what's your fallback?")
- Focus probes on the specific gaps recorded from the previous session
- Hold the same scoring bar but note whether the specific prior weaknesses improved

### Step 3 — Add a brand-new interview question

#### Determine the sourcing mode

First, check `CLAUDE.md` for the Companies & Target Roles table. Count how many companies are listed with a target role.

**If target companies exist → Company-targeted mode**
**If the table is empty or no companies are listed → Open mode**

---

#### Company-targeted mode

Search for questions anchored to the listed companies. Rotate through them so you don't always research the same one — check which company's questions were most recently added to 00_overview.md and pick a different one this time.

Search queries:
- `"[company] system design interview questions [year]"`
- `"[company] engineering blog system design"`
- `site:glassdoor.com "[company]" "system design" interview`
- `site:levels.fyi "[company]" system design`

Target the company's known product surface for question relevance:
- **Apple**: iCloud sync, Siri backend, App Store search ranking, Maps, FaceTime, Apple Pay, Spotlight indexing
- **Google**: Search ranking pipeline, YouTube recommendations, Google Maps ETA, Gmail, Drive sync, Meet
- **Meta**: News Feed ranking, Instagram Stories delivery, WhatsApp message sync, Marketplace search, Ads targeting

---

#### Open mode (no company targets)

When there's no company target, use a priority stack. Work down the stack in order — the first source that produces a valid new question wins.

**Priority 1 — Constraint escalation on a mastered archetype**

Look at the Completed Mock Interviews table in progress.md. Find any archetype with score ≥ 20/25 that has only been attempted once. Design a harder variant:

- Add a scale constraint 10–100× beyond the original (e.g., notification system → 2 billion users across 50 countries with per-country regulatory opt-out requirements)
- Add a reliability constraint (e.g., "exactly-once delivery with audit trail for financial notifications")
- Add a cross-domain requirement (e.g., "now add a real-time personalization layer that ranks notifications by predicted engagement")
- Flip a core assumption (e.g., "design the same system but the client is an IoT sensor with 2KB RAM, not a smartphone")

This is the cheapest signal source — you already have all the context. No search needed.

**Priority 2 — Frequency signal (what's actually being asked right now)**

Search:
- `site:glassdoor.com system design interview 2025 2026`
- `site:interviewing.io system design questions`
- `site:levels.fyi system design interview experience`
- `"system design interview" questions 2025 site:reddit.com`

Extract questions mentioned more than once across sources. Cross-reference against the existing L3 list. Pick the most-mentioned question not already covered.

**Priority 3 — Industry trend signal (what's becoming interview-relevant)**

Search:
- `"system design" "LLM" OR "AI" interview question 2025`
- `emerging system design topics senior engineer 2025 2026`
- `"design a" system OR infrastructure site:github.com OR site:blog.bytebytego.com`
- Engineering blog posts from Uber, Airbnb, Stripe, Netflix, LinkedIn (rotate) about infrastructure problems they solved in the last 12 months

Good trend signal: if a company published a blog post about solving a specific infrastructure problem, that problem is likely to become an interview question at that company within 6–18 months.

**Priority 4 — Cross-domain combination (hardest, most differentiating)**

Take two archetypes from the completed L3 list and find a real product that requires both simultaneously. Construct a question that cannot be solved by applying either archetype in isolation.

Examples:
- Notification system + Recommendation system → "Design a personalized push notification system that decides what to send, when to send it, and at what frequency, with per-user fatigue modeling"
- Collaborative editing + Live streaming → "Design a live coding interview platform where interviewer and candidate both edit code in real-time and the session is recorded and played back"
- Leaderboard + Location-based service → "Design a real-time proximity leaderboard for a running app that shows your rank among all runners within 5km of you"

Cross-domain questions are rare in interviews but are the hardest to prepare for without deliberate practice. They also demonstrate the deepest system design fluency.

---

#### Selection criteria (applies to both modes)

From the search results or constructed variants, pick **one question** that satisfies all of:
- Not already in the L3 archetype list in 00_overview.md (check by topic, not wording)
- Difficulty appropriate for current score trajectory: if avg score <15/25, pick simpler; 15–20, mid; 20+, harder or cross-domain
- At least 70% of required foundations are covered by completed L1/L2 lessons (if not, go to Step 5 first)
- Clearly scoped enough to be answerable in a 45-minute mock interview

Add it as a new archetype in both `00_overview.md` and progress.md:

**In 00_overview.md**, append to the Layer 3 table:
```markdown
| L3-[next number] | [New Question] | [L1 prerequisites] | [L2 prerequisites] | 🆕 |
```

**In progress.md Scheduled Interviews**:
```markdown
| New | L3-[N] [Question Title] | 2026-04-21 | ☐ Pending |
```

### Step 4 — Announce both

Tell Laksh clearly what was scheduled and why:

```
All current lessons complete.

Scheduled:
1. RETRY: [archetype] — focusing on [specific prior gaps]. Run `/interview [topic]`.
2. NEW: [new question] — sourced from [company] engineering blog / interview reports.
   This tests [which foundations] under [what constraint]. Run `/interview [topic]` when ready.

Suggested order: do the retry first to close known gaps, then tackle the new question fresh.
```

### Step 5 — Check for new foundations to teach first

If the new question requires a foundation or pattern not yet in the curriculum, add a new lesson before scheduling the interview:

1. Add the new topic to `00_overview.md` in the appropriate layer (L1 if it's a base technology, L2 if it's a pattern).
2. Set it as the **Next Lesson** in progress.md.
3. Tell Laksh: "The new interview requires [topic], which isn't in the curriculum yet. Teaching that first before the interview."

This ensures the learning loop generates its own curriculum expansion, not just its own interview expansion.

---

## Lesson Content Guide

### Layer 1 Foundation Lessons

**L1-01: Networking**
Research: TCP 3-way handshake, TCP vs UDP, HTTP/1.1 keep-alive vs HTTP/2 multiplexing, WebSockets upgrade process, Server-Sent Events, long polling vs short polling, HTTP/3 QUIC adoption stats
Key numbers: TCP port range 1-65535, HTTP/2 default port 443, WebSocket upgrade via HTTP 101, APNs port 5223

**L1-02: Push Delivery (APNs & FCM)**
Research: Apple APNs documentation, FCM architecture, device token lifecycle
Key insight: Device is CLIENT. OS maintains ONE persistent connection to APNs (port 5223). Your backend sends to APNs via HTTP/2 POST. APNs delivers via persistent connection to device.
Key numbers: APNs port 5223 (device), APNs port 443 (provider), max payload 4KB (APNs), store-and-forward window varies by notification type

**L1-03: DNS & CDN**
Research: DNS resolution chain, CDN PoP architecture, cache-control headers, CDN cache invalidation strategies

**L1-04: Load Balancing**
Research: L4 vs L7 differences, consistent hashing algorithm, session affinity, health check patterns

**L1-05: Databases — Relational**
Research: B-tree vs LSM-tree indexes, WAL (write-ahead log), MVCC, primary-replica lag, read replica use cases

**L1-06: Databases — NoSQL**
Research: Cassandra ring architecture, DynamoDB partition keys, Redis data structures, when columnar vs document vs KV

**L1-07: Caching**
Research: Redis sentinel vs cluster, cache-aside pattern implementation, cache stampede / thundering herd, hot key problem

**L1-08: Message Queues & Streaming**
Research: Kafka partition, offset, consumer group internals; SQS visibility timeout; exactly-once semantics

**L1-09: Object Storage**
Research: S3 multipart upload, presigned URLs, eventual consistency model, storage classes

**L1-10: Distributed Systems**
Research: CAP theorem proof intuition, Raft leader election, vector clocks, CRDTs

**L1-11: API Design**
Research: REST idempotency, gRPC streaming modes, GraphQL N+1 problem, API gateway patterns

**L1-12: Security**
Research: TLS handshake, JWT structure and verification, OAuth2 flows, API rate limiting at gateway

**L1-13: Observability & Monitoring**
Research: The three pillars (metrics, distributed traces, logs), Prometheus metrics types, OpenTelemetry, SLO vs SLI vs SLA, alerting on burn rate vs threshold
Key insight: "How do you know it's working?" — every interviewer probes this. Observability is not optional in 2025.

**L1-14: AI/ML Infrastructure Basics**
Research: Feature store architecture (Feast, Tecton), vector databases (Pinecone, pgvector), embedding generation pipeline, online vs offline inference, cold start problem, model serving latency targets
Key numbers: typical embedding dimensions (768, 1536), p99 inference latency targets (<100ms online serving)

### Layer 2 Pattern Lessons

**L2-01: Fan-out Strategies** — fan-out on write vs read vs hybrid; when each breaks
**L2-02: Event-Driven Architecture** — pub/sub, event sourcing, choreography vs orchestration
**L2-03: CQRS** — command/query separation, separate read/write models
**L2-04: Saga Pattern** — distributed transactions, compensating transactions, no 2PC
**L2-05: Sharding Strategies** — range, hash, directory-based; hotspots, resharding
**L2-06: Replication Patterns** — primary-replica, multi-primary, leaderless (Dynamo-style)
**L2-07: Rate Limiting Algorithms** — token bucket, leaky bucket, sliding window log, sliding window counter
**L2-08: Reliability Patterns** — circuit breaker, bulkhead, retry with jitter, timeout, graceful degradation
**L2-09: Notification Pipeline Pattern** — priority queue separation, per-tier workers, dead letter queues
**L2-10: Search & Indexing** — inverted index, WAL, Elasticsearch architecture
**L2-11: Outbox Pattern** — transactional outbox, dual-write problem, Debezium CDC
**L2-12: Backpressure & Flow Control** — consumer lag, load shedding, queue depth monitoring
**L2-13: Data Pipeline Patterns** — Lambda vs Kappa architecture, reprocessing via log replay

### Layer 3 Archetype Lessons

Layer 3 entries are not taught as lessons — they are mock interviews. When the user reaches an L3 entry as the next lesson, redirect to `/interview`.

---

## Interviewer Mode vs Teacher Mode

This skill runs in **Teacher Mode**. When in teacher mode:
- Explaining gaps is allowed and encouraged
- Hints are free — this is learning, not scoring
- Connect every concept back to "how does this change a system design?"
- After teaching a concept, always ask: "Where would you use this in the notification system we designed in session_001?"

When the user invokes `/interview`, switch to Interviewer Mode (strict, no hints, scored).

---

## Quality Bar

A lesson is complete when the user can:
1. Explain the concept in one sentence without jargon
2. State the main tradeoff
3. Name one system design question where it's relevant
4. Identify the mistake a candidate makes when they don't know it
