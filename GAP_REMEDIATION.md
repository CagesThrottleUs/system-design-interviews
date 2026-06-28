# Gap Remediation Guide

After scoring yourself with `EVALUATION_RUBRIC.md`, use this guide to run targeted drills for each weak dimension. Every section follows the same structure: a scenario-style symptom, the root cause, a specific time-boxed drill, a memorizable mini-checklist, and a worked example with real numbers and reasoning. Work through the sections where your rubric score was 1–2 first; revisit 3-scored sections after your first full mock interview.

---

## Gap 1: Requirements Scoping

**Symptom**

You're five minutes into a URL Shortener design and you already have an architecture diagram. You've drawn an API gateway, a key-generation service, a Redis cache, and a PostgreSQL table — it looks polished. The interviewer hasn't stopped you, but the clarity of the problem statement has assumed a lot: are these URLs public or private? Is there an analytics requirement? Is the target scale a startup's internal tool or a global TinyURL serving 1B redirects per day? You've drawn a complete system for an unstated problem, and now every question the interviewer asks reveals a different assumption baked invisibly into your diagram.

**Root Cause**

Most engineers receive context passively in day-to-day work — the ticket has acceptance criteria, the PM answered questions in the planning session, and Slack threads filled in the gaps before the sprint started. The skill of actively extracting requirements in real time under pressure is not exercised regularly. In interviews, the temptation to demonstrate knowledge quickly causes premature starts: the brain pattern-matches "URL Shortener" to a stored architecture and begins drawing before the problem is fully specified.

**Drill**

Take any problem from the problem bank (Rate Limiter works well). Set a 10-minute timer. Do not draw anything and do not name any architectural component — no databases, no queues, no caches. Your only output is a list of exactly 5 requirement questions. Each question must be followed by a single sentence explaining what changes in the architecture if the answer goes one way versus the other. "What is the DAU?" is not sufficient. The required format is: "What is the DAU? If it's under 10M, a single database shard is sufficient; if it's over 100M, we need horizontal sharding from day one, which changes the key-generation strategy and the data model." If you cannot write the architectural consequence, the question is not specific enough. Stop the timer at 10 minutes and review: do all 5 questions have concrete architectural consequences stated?

**Mini-Checklist**

1. Scale: Ask for DAU, peak QPS, and read/write ratio — these three numbers drive every capacity decision.
2. Latency: Ask which operations are user-blocking and what the SLA is — this determines whether you need a cache and at what tier.
3. Consistency: Ask whether users can see stale data and for how long — this determines replication strategy and cache TTL.
4. Delivery semantics: Ask whether at-least-once, at-most-once, or exactly-once delivery is required — this determines whether a queue is needed and what kind.
5. Explicit non-goals: Declare what you are leaving out of scope before the interviewer assumes you forgot it.

**Worked Example**

Applying the drill to the Rate Limiter problem. Five requirements questions with architectural consequences:

Question 1: "Is the rate limiter per user, per IP, per API key, or per service? If it's per user, the rate limit state is tied to an authenticated identity and stored by user_id; if it's per IP, we must deal with NAT — a single household or corporate network may share one IP, which makes per-IP limits too aggressive and forces a hybrid approach."

Question 2: "What is the QPS this rate limiter must handle? If the rate-limited API receives 500K req/s, the rate limiter itself must process 500K checks per second without adding latency to the hot path. This immediately rules out any solution backed by a SQL database and requires an in-memory store like Redis."

Question 3: "What is the latency budget for the rate-limit check? If the overall API SLA is 50ms p99, and the rate limiter is synchronous in the request path, it must complete in under 5ms. This rules out any approach that requires a network round trip to a distant region."

Question 4: "Should the rate limiter fail open or fail closed when the backing store (Redis) is unavailable? Fail open means all traffic is allowed when Redis is down — we risk overload but keep the product running. Fail closed means all traffic is rejected when Redis is down — we protect downstream services but bring down the product. This is a business decision that must be answered before the design is finalized."

Question 5: "Is this a global rate limit (aggregate across all servers) or a per-server rate limit? Per-server limits are simpler (local memory, no coordination) but can be bypassed by hitting different servers. Global limits require a shared state store (Redis) but provide accurate enforcement. If the product requires strict enforcement — for billing or abuse prevention — we need global limits."

---

## Gap 2: Capacity Estimation

**Symptom**

You've done estimation — you wrote down the DAU, calculated QPS, and even derived a storage number. The interviewer nodded. Then you started drawing the architecture and the estimation disappeared entirely. You placed Kafka in the design because it's a streaming system and this feels like a streaming problem. When the interviewer asks "why Kafka and not a simple database table as a queue?", you say "because of the scale." When asked "what about the scale specifically?", you go quiet, because the estimation you did 10 minutes ago is on a piece of paper you never consulted again. The estimation was performed but not used.

**Root Cause**

Most engineers treat estimation as a ritual — a section of the interview to complete before moving on to the "real" part. They haven't internalized that estimation is a compass that should be consulted every time a component is introduced into the design. The root cause is practicing estimation in isolation as a math skill rather than as an input to architectural decision-making. Estimation practiced alone produces a number; estimation practiced as a design input produces a justified architecture.

**Drill**

Take the Notification System problem. Set a 15-minute timer. Your constraint: you cannot name any architectural component — no Kafka, no Redis, no database, no queue — until you have shown, using only arithmetic, that a simpler approach is insufficient. Start from 500M DAU and derive the write QPS. Then explicitly show why a single PostgreSQL server fails at that number. Then propose the next-simplest solution and show why that fails. Continue until arithmetic forces you to the right architecture. Every component you eventually name must be preceded by a sentence of the form: "A [simpler component] fails because [arithmetic reason], therefore we need [this component]."

**Mini-Checklist**

1. Start from DAU: daily active users → events per user per day → requests per second (average), then multiply by 3 for peak.
2. Storage: (message or object size in bytes) × (daily volume) × (retention in years) — store this number and reference it when choosing storage systems.
3. Bandwidth: peak QPS × average payload size in KB — this determines network tier and CDN requirements.
4. Bottleneck identification: after estimating all tiers, ask "which component is the first to saturate?" — this is where the interesting design work happens.
5. Rule of thumb reference: PostgreSQL ~10K writes/s, Redis ~200K ops/s, Kafka ~1M msgs/s per partition cluster, a single app server ~10K–50K req/s depending on work complexity.

**Worked Example**

Applying the drill to the Notification System. Starting from 500M DAU. Users receive an average of 5 notifications per day. Total notifications per day: 2.5B. Average rate: 2.5B / 86,400 = ~29,000 notifications/second. Peak (3x): ~87,000/second.

A single PostgreSQL server handles ~10,000 writes/second reliably under production load. At peak 87K/s, a single database is 8.7x over capacity. Therefore we cannot use a single database as the notification write path. Next simplest: a database cluster with 10 shards. 87K/s / 10 shards = 8.7K/s per shard, within PostgreSQL limits. But now consider the fan-out: assume 10% of notifications require fan-out to followers (social context). For a user with 5,000 followers, one notification event becomes 5,000 write operations. At 8,700 such fan-out events per second (10% of 87K), we get 8,700 × 5,000 = 43.5M write operations per second at peak. A 10-shard database cluster handles 100K writes/s total. 43.5M writes/s exceeds that by 435x. Therefore we cannot fan-out synchronously into the database. We need a write buffer — a queue — to absorb the fan-out spike and drain into storage asynchronously. A Kafka cluster with 50 partitions handles ~50M msgs/s, which covers the peak fan-out. This is the first place Kafka appears in the design, and it is justified by the arithmetic, not by intuition.

Storage: each notification record is ~200 bytes (user_id, type, payload, timestamp, read_flag). 2.5B × 200 bytes × 90 days retention = ~45TB. This drives the choice of a time-partitioned storage system with a 90-day TTL rather than a general-purpose relational database.

---

## Gap 3: API Design

**Symptom**

The interviewer says "can you walk me through the API for this?" and you describe it in 45 seconds: "a POST endpoint to shorten a URL, and a GET endpoint to redirect." The interviewer asks "what does the POST request body look like?" and you say "the long URL." Asked about error cases, you say "404 if not found, 500 for server errors." Asked about idempotency, you pause — you haven't thought about what happens when the client's POST times out and retries, creating two short codes for the same long URL. The API description revealed that you thought about the system in terms of database operations (INSERT and SELECT) rather than in terms of a contract between a client and a service.

**Root Cause**

Engineers who primarily work on backend services often think about data operations rather than API contracts. The mental model is "what does my handler do to the database" rather than "what does my client need, and what guarantees does this contract provide?" The skill of designing an API for external consumers — including pagination semantics, idempotency keys, error shapes that are machine-readable, and versioning strategy — requires deliberate practice because it is distinct from designing an internal function signature.

**Drill**

Take the URL Shortener problem. Design the complete API for this service in 10 minutes. For each endpoint, write: the full HTTP method and URL path; the complete request shape including headers, path parameters, query parameters, and request body with field types; the complete success response shape with HTTP status code and all response fields; and at least 2 error responses with HTTP status codes and machine-readable error codes (not just status codes). After you finish, audit your own design for three things: (1) Is there any endpoint that would produce a duplicate or inconsistent result if called twice? If yes, add an idempotency mechanism. (2) Is there any list endpoint? If yes, it must have cursor-based pagination. (3) Is auth specified on every endpoint, including whether it is required or optional?

**Mini-Checklist**

1. Resources not actions: design around nouns (/messages, /links) not verbs (/sendMessage, /createLink).
2. Correct HTTP semantics: POST creates, PUT replaces entirely, PATCH updates partially, DELETE removes — never use POST for reads.
3. List endpoints always need pagination: cursor-based (opaque token) for large datasets, offset-based only for small bounded lists.
4. Mutating endpoints that can be retried need idempotency keys in the request, with server-side deduplication for the key's lifetime.
5. Every endpoint needs its auth requirement stated: unauthenticated, bearer token, API key, or OAuth scope.

**Worked Example**

Applying the drill to the URL Shortener. Complete API:

`POST /v1/links` — Auth: Bearer token required. Body: `{"long_url": "string (required, valid URL)", "custom_alias": "string (optional, 4–20 chars, alphanumeric)", "expires_at": "string (optional, ISO 8601 datetime)", "idempotency_key": "string (optional, UUID v4)"}`. Response 201: `{"short_code": "abc123", "short_url": "https://sho.rt/abc123", "long_url": "...", "created_at": "2026-06-28T00:00:00Z", "expires_at": "2027-06-28T00:00:00Z"}`. The idempotency_key is critical: if the client's POST times out and retries with the same key, the server returns the original 201 response with the same short_code rather than creating a duplicate. Error 409 CONFLICT: `{"error": "ALIAS_TAKEN", "message": "custom alias 'mylink' is already in use"}`. Error 422 UNPROCESSABLE: `{"error": "INVALID_URL", "message": "long_url must be a valid absolute URL with http or https scheme"}`.

`GET /{short_code}` — Auth: none (public endpoint). No /v1 prefix because this is a vanity URL in production. Response 302 redirect to long_url with Cache-Control: max-age=300. Error 404: `{"error": "NOT_FOUND"}`. Error 410: `{"error": "EXPIRED", "message": "This link expired on 2025-01-01"}` — the 410 Gone is important; it tells search engines and clients to remove the link rather than retry.

`GET /v1/links` — Auth: Bearer token required. Query params: `cursor` (opaque pagination token), `limit` (1–100, default 20). Response 200: `{"items": [...], "next_cursor": "eyJ...", "has_more": true}`. Cursor-based pagination is required here because a user could have thousands of links and offset-based pagination would be inconsistent if new links are created mid-query.

`DELETE /v1/links/{short_code}` — Auth: Bearer token. Response 204 no body. Error 403: `{"error": "FORBIDDEN", "message": "You do not own this link"}`. Error 404: `{"error": "NOT_FOUND"}`.

---

## Gap 4: Data Model & Storage

**Symptom**

You've chosen PostgreSQL for your notification system. The interviewer asks "why not Cassandra?" and you say "PostgreSQL is reliable and my team uses it." Asked about indexes, you say "we'd add indexes on the columns we query frequently." The schema you've drawn has table names and column names, but no indication of the primary key structure, no composite indexes shown, and no explanation of how the top queries — "give me all unread notifications for user X ordered by time" — would actually execute against this schema. The storage choice was made by familiarity, and the schema was designed by instinct rather than by access patterns.

**Root Cause**

Storage selection in practice defaults to "what the team already runs." Engineers don't get regular practice at choosing between PostgreSQL, Cassandra, DynamoDB, MongoDB, and Redis from first principles. The skill of access-pattern-first schema design — writing the top queries first, then designing the schema to serve those specific queries efficiently — is rarely exercised because most engineers design schemas once during a project's genesis and then maintain them for years without revisiting the original decision.

**Drill**

Take the Design Twitter Feed problem. Write down the top 3 queries this system must execute, ordered by frequency. For each query: write the exact query in pseudocode (e.g., "SELECT tweet_id, content, author_id FROM tweets WHERE user_id IN (following_list) ORDER BY created_at DESC LIMIT 50"), identify the access pattern type (key-value lookup, range scan, fan-out write, graph traversal), and name the storage system that best serves that access pattern with a single-sentence justification. Then design the schema — tables, collections, or column families — to serve those three specific queries. You are not designing a normalized data model; you are designing a query-shaped data model.

**Mini-Checklist**

1. Write the top 3 queries before touching the schema — the schema serves the queries, not vice versa.
2. Read/write ratio drives optimization: high read → denormalize and cache; high write → normalize and batch.
3. SQL vs NoSQL decision axis: if you need flexible queries and strong consistency, SQL; if you need predictable latency at high write volume with known access patterns, Cassandra/DynamoDB.
4. Name every index and the query it serves — an unnamed index is a sign you haven't thought through the query plan.
5. Identify hot data vs cold data: hot (last 7 days of notifications, active user sessions) goes in Redis or SSD-backed storage; cold (historical logs, archived posts) goes in object storage.

**Worked Example**

Applying the drill to the Design Twitter Feed. Top three queries:

Query 1 (highest frequency, every timeline load): "Get the 50 most recent tweets from all accounts user X follows, with tweet content and author metadata, ordered by created_at descending." Access pattern: range scan with fan-in across many user IDs. If implemented naively with a SQL JOIN at read time — `SELECT t.* FROM tweets t JOIN follows f ON t.author_id = f.followed_id WHERE f.follower_id = X ORDER BY t.created_at DESC LIMIT 50` — this becomes a JOIN across potentially 1000 followed accounts and millions of tweets. At 100M DAU each loading their timeline 3 times per day, that's 300M such queries per day, ~3,500/s average. A SQL JOIN at this volume and cardinality will not meet a <200ms SLA. This access pattern maps to Cassandra with a pre-materialized timeline table: partition key `follower_id`, clustering key `tweet_timestamp DESC, tweet_id`. Timeline reads become a single-partition range scan — O(1) partition lookup, O(k) rows scanned for k=50.

Query 2 (high frequency, on every tweet creation): "Write this tweet to the timelines of all of the author's followers." Access pattern: fan-out write. For a user with 10K followers, one tweet = 10K Cassandra writes to 10K different partitions. This is handled asynchronously by a fan-out worker pool reading from a Kafka topic. The fan-out table schema: `CREATE TABLE user_timeline (follower_id UUID, tweet_ts TIMESTAMP, tweet_id UUID, author_id UUID, content TEXT, PRIMARY KEY (follower_id, tweet_ts, tweet_id)) WITH CLUSTERING ORDER BY (tweet_ts DESC)`.

Query 3 (medium frequency, on profile page load): "Get the tweet count, follower count, and following count for user X." Access pattern: key-value lookup by user_id. These are counters that change frequently (every follow/unfollow, every tweet). Storing them in the main users table in PostgreSQL with `FOR UPDATE` locking creates contention for high-follower accounts. Better: a Redis hash `user:{user_id}:counts` with fields `tweets`, `followers`, `following`, updated atomically with HINCRBY. Writes to PostgreSQL as the source of truth happen asynchronously every 5 minutes for durability.

---

## Gap 5: High-Level Architecture

**Symptom**

You've drawn a diagram with eight components. It looks like the architecture diagrams you've seen in blog posts — an API gateway, a load balancer, two microservices, a Kafka queue, a Redis cache, a primary database, and a CDN. When the interviewer asks "why is there a message queue between the API server and the notification service?", you say "for scalability and decoupling." When asked "decoupling from what specifically?" you elaborate that queues are good for decoupling producers and consumers. The queue is present because queues are a thing that appears in architectures, not because you derived the need for one from the requirements you established earlier. The diagram is a collection of components rather than a derived solution.

**Root Cause**

Familiarity with standard architectural patterns — the queue, the cache, the CDN, the load balancer — without the underlying reasoning for when they are and are not necessary produces component-collection architectures. The anti-pattern is "collect the known components" rather than "derive the minimal components needed." This is very common in candidates who studied architecture by reading solution guides: they absorbed the conclusions without reconstructing the reasoning paths that produced them.

**Drill**

Take the Web Crawler problem. Draw an HLD in 20 minutes with one constraint: before you draw any component onto the diagram, you must write one sentence on paper explaining what breaks if you do NOT include it. If you cannot complete the sentence "Without this component, [specific named failure] would occur," you cannot add the component. At the end of 20 minutes, review your diagram. Every component should have a written justification of its necessity sitting next to it. Discard any component that only has a vague justification like "for performance" or "for scalability."

**Mini-Checklist**

1. Every component solves a named, specific problem — state it when you draw the component.
2. Every component has a single clear responsibility — if you can't state it in 10 words, the component is doing too much.
3. Identify stateless components explicitly and note that they scale horizontally by adding instances.
4. Identify stateful components explicitly and explain the scaling strategy for each (sharding key, replication factor, leader election).
5. Trace one request from client to storage and back before declaring the HLD complete — gaps in the trace reveal missing components.

**Worked Example**

Applying the drill to the Web Crawler. Starting from the seed URL and deriving components in necessity order.

A scheduler receives seed URLs. Without a URL frontier (distributed queue), there is no way to prioritize URLs by crawl priority, distribute work across multiple crawler instances, or rate-limit per domain. Add a distributed priority queue (Kafka or a Redis sorted set). Without a deduplication store, crawlers will follow links in cycles and crawl the same URL indefinitely. Add a visited-URL store — a Redis Bloom filter for approximate deduplication with a 1% false positive rate is sufficient; exact deduplication would require a 200-byte hash entry per URL, and at 10B URLs that's 2TB of exact dedup storage versus ~1.2GB for a Bloom filter. Without a politeness layer, all crawlers will simultaneously fetch from the same domain, violating robots.txt and getting IP-banned. Add a per-domain rate limiter enforcing a configurable crawl delay, implemented as a Redis key per domain with TTL equal to the crawl delay. Without a DNS cache local to the crawlers, every URL fetch incurs a full DNS resolution round trip — at 10K URLs/second across 100 crawler workers, that's 10K DNS queries/second against external resolvers, which will be rate-limited. Add a local DNS cache with a 5-minute TTL per domain. Without a content store, crawlers have nowhere to write raw HTML — their job is fetch-and-store, but storing directly in a database limits throughput to DB write capacity. Add S3-compatible object storage for raw HTML (write once, read occasionally for parsing). Without a parser worker pool consuming from a separate Kafka topic, parsing would block crawling — they have different throughput characteristics. Add an async parser layer consuming a "fetched" topic and producing a "parsed links" topic that feeds back to the URL frontier.

The final diagram has 7 components, each with a named necessity. No component is present "for scalability."

---

## Gap 6: Deep Dive & Tradeoffs

**Symptom**

You're 35 minutes into the design and the interviewer says "let's go deeper on the fan-out mechanism." You explain confidently that when a user posts a tweet, a fan-out worker reads the author's follower list and writes the tweet to each follower's timeline table. The interviewer asks "what are the tradeoffs of fan-out-on-write versus fan-out-on-read?" You explain that fan-out-on-write has high write amplification but low read latency, while fan-out-on-read has low write load but high read latency. The interviewer asks "so when does Twitter use one versus the other?" You say "I believe they use a hybrid." Asked "what is the specific criterion that determines which approach to use for a given user?", you pause — because you know that Twitter uses a hybrid with a celebrity exception, but you don't know the decision criterion. You consumed the conclusion without the reasoning.

**Root Cause**

Reading solution guides before attempting problems produces a false impression of understanding. You know the answer (hybrid fan-out with celebrity exception) but not the reasoning path (the decision criterion is follower count vs. fan-out cost at a threshold, typically around 10K–100K followers). This is a pattern across every tradeoff in system design: candidates memorize which option is chosen in famous systems without understanding why, which breaks down the moment an interviewer probes one level deeper. The root cause is consuming solutions rather than reconstructing arguments.

**Drill**

Take any design decision from a problem you've already studied. For example: "Why does a distributed rate limiter use Redis instead of a local in-memory counter?" Write down the answer you believe. Then spend 5 minutes writing the steelman case for the alternative — a 3-sentence argument for why local in-memory counters are strictly better than Redis for rate limiting. If you cannot construct a genuine argument for the alternative, you don't understand the tradeoff; you've memorized the conclusion. Then write the decision criterion: the single condition under which you would switch from your chosen approach to the alternative. Repeat for 3 design decisions per session.

**Mini-Checklist**

1. For every major design choice, name the 2–3 alternatives explicitly before justifying your choice.
2. State the decision criterion: the specific property of THIS system that makes your chosen option better.
3. Name what you are trading away: the real cost of your choice, not a generic "it's more complex."
4. Identify which requirement, if changed, would flip your decision — this proves you understand the tradeoff space.
5. Proactively volunteer the 1–2 most interesting components to deep-dive on; don't wait to be asked.

**Worked Example**

Applying the drill to the fan-out mechanism in Twitter Feed design.

Fan-out-on-write: when user A posts, a worker immediately writes the tweet to the timeline tables of all of A's followers. Read latency: O(1) — a single partition read. Write amplification: O(followers) per tweet. For a user with 100 followers, one tweet = 100 writes. For a user with 50M followers (a celebrity), one tweet = 50M writes, which at 200 bytes per timeline entry = 10GB of writes per celebrity tweet. If a celebrity posts 5 times per day and there are 1,000 such accounts, that's 50B writes per day from celebrity posts alone = ~580K writes/second purely from celebrity fan-out. The decision criterion: fan-out-on-write is appropriate when the follower count is below a threshold where the write amplification is bounded and the fan-out completes within an acceptable time window (e.g., under 30 seconds for all followers to see the tweet).

Fan-out-on-read: no precomputed timeline. When user B loads their feed, the system fetches all tweet IDs from accounts B follows, merges and sorts them, and returns the result. Write amplification: O(1) per tweet. Read cost: O(following_count × tweets_per_account_per_timewindow). For a user following 500 accounts, a timeline read requires 500 lookups and a merge sort of potentially 50,000 recent tweets. At 300M timeline reads per day, this is extremely expensive.

The decision criterion for the hybrid: use fan-out-on-write for users with follower_count < threshold (e.g., 10,000 followers). For users above the threshold (celebrities), skip precomputed fan-out. At read time, merge the precomputed timeline (from followed non-celebrities) with a real-time fetch of the followed celebrities' recent tweets. The threshold is calibrated so that the real-time fetch at read time touches at most ~1,000 accounts (the number of celebrities a typical user follows is small). This bounds both write amplification (no 50M-write fan-outs) and read cost (1,000 real-time fetches instead of 500,000).

The requirement that would flip this: if read latency SLA were loosened to 2 seconds instead of 200ms, fan-out-on-read becomes viable. If write cost became irrelevant (infinite write IOPS), fan-out-on-write always wins for read latency.

---

## Gap 7: Failure Modes & Scaling

**Symptom**

You've just presented a clean HLD with six components. The interviewer asks "what happens when your Redis cache goes down?" You say "we'd fall back to the database." The interviewer asks "and what does that mean for the system?" You say "it would be slower." The interviewer presses: "how much slower, and what would a user experience?" You have no answer. You haven't thought about what percentage of traffic Redis was absorbing, how that translates to database load, whether the database can handle the sudden traffic, or whether the fallback is automatic (circuit breaker) or manual (a deployment). The failure mode was named but not analyzed.

**Root Cause**

Failure analysis is not a daily engineering activity outside of on-call rotations and incident postmortems. Engineers who have not been on-call for production systems haven't built the habit of asking "what breaks next?" for every component in a design. The skill requires not just knowing that components can fail but having a practiced mental model for how failure cascades through a system and what the user experiences at each stage of degradation.

**Drill**

Take any architecture you've designed. For each component in the diagram, write a failure scenario in exactly two sentences: "If [component] fails, then [specific user-visible impact with a quantity or timeframe if possible]. The mitigation is [specific named mechanism], which means [the tradeoff that mitigation introduces]." Do this for every component before you call the architecture complete. An HLD with six components should have six failure scenarios written before you discuss the design. If you cannot write the user-visible impact for a component, you have not thought through that component's role in the system.

**Mini-Checklist**

1. Identify the bottleneck component from your estimation output — this is the first component that breaks under load, and it should match.
2. For each component: user-visible impact of hard failure (total component loss) vs. soft failure (degraded performance).
3. For each failure: hard failure (total outage) or graceful degradation (partial service with automatic recovery)?
4. For each mitigation: does it require a human (deployment, escalation) or is it automatic (circuit breaker, retry with backoff, replica promotion)?
5. At 10x current scale, which component's scaling strategy must change architecturally — not just by adding more instances?

**Worked Example**

Applying the drill to the Notification System with six components: API Gateway, Kafka, Fan-out Worker Pool, Notification DB (Cassandra), APNs/FCM delivery, Push Device.

Component 1 — Kafka broker failure: If Kafka becomes unavailable, notification producers cannot enqueue events and the API returns 503 to callers for the duration of the outage (typically 30–90 seconds for leader re-election). The mitigation is producer-side retry with exponential backoff and a local in-memory queue that buffers up to 10 seconds of events — this means in a brief Kafka blip, notifications are delayed but not lost, at the cost of ~50MB of memory per API server instance.

Component 2 — Fan-out Worker slow (not failed, just lagging): If fan-out workers fall behind due to a spike in posts from high-follower accounts, consumer lag grows and notification delivery latency increases from the target 5 seconds to minutes. User-visible impact: users see delayed notifications with no error — the worst UX outcome because there's no signal that something is wrong. The mitigation is Kafka consumer group lag monitoring with an alert at >100K messages of lag; the auto-scaler adds fan-out worker instances when lag exceeds 50K messages. Tradeoff: autoscaling takes ~90 seconds to provision, so the first 90 seconds of any spike still sees lag.

Component 3 — Cassandra notification DB (partial shard failure): If the Cassandra node responsible for user partition range X fails, reads and writes for those users fail until the replica is promoted (typically 15–30 seconds for automatic promotion via Cassandra's internal gossip). User-visible impact: affected users cannot load their notification inbox and new notifications to those users are queued in Kafka rather than committed to storage. The mitigation is a replication factor of 3 with LOCAL_QUORUM consistency; a single node failure does not cause data loss, only a brief write rejection while the coordinator retries against the remaining replicas. Tradeoff: LOCAL_QUORUM adds ~1ms of write latency versus eventual consistency, but this is acceptable for a notification inbox.

Component 4 — APNs/FCM delivery service unavailable (external dependency): If Apple's APNs is unreachable (this has happened — Apple had APNs outages in 2020 and 2022), push notifications cannot be delivered to iOS devices. User-visible impact: iOS users receive no push notifications for the duration of the outage. The mitigation is a circuit breaker that opens after 5% of APNs requests fail in a 30-second window — failed deliveries are marked as "pending" in the notification record and a retry job attempts delivery every 5 minutes for up to 24 hours. Tradeoff: messages accumulate in the retry queue during the outage; when APNs recovers, the burst of retries must be rate-limited to avoid being throttled by Apple.

Component 5 — Cassandra at 10x scale: At 10x current scale (5B DAU hypothetically), total notifications reach 870K/second at peak. A single Cassandra cluster in one region becomes both a throughput bottleneck and a single point of regional failure. The architectural change required at 10x is not adding more Cassandra nodes to the existing cluster but moving to a multi-region active-active setup with regional Cassandra clusters and a global notification routing layer. This is an architectural change, not a horizontal scaling operation — the data model must be redesigned with a region-aware partition key to ensure cross-region consistency only where it is absolutely needed (user preference reads) and eventual consistency elsewhere (notification delivery).

---

## Weekly Drill Plan

Monday is for requirements and estimation in isolation. Take the URL Shortener and Rate Limiter problems. For each one, set a 10-minute timer and do only requirements extraction — no design, no estimation, no component names. Produce exactly 5 requirement questions, each paired with a one-sentence architectural consequence. Then set a second 10-minute timer and do only capacity estimation for the same problem — numbers only, no components. Derive QPS, storage, and bandwidth from first principles, and identify the bottleneck tier. The explicit prohibition on mixing these two skills is intentional: Monday trains you to be thorough in both phases independently, so that when you combine them in a mock interview, neither one is rushed by pressure from the other.

Tuesday is for API design and data modeling in isolation. Take the Notification System. Spend 15 minutes on the API alone: all endpoints, full request and response shapes, idempotency analysis for every mutating endpoint, and an explicit auth mechanism on every route. Then spend 15 minutes on the data model alone: write the top 3 queries first in pseudocode, identify the access pattern for each, and design the schema to serve those specific queries with named indexes. The prohibition on drawing any HLD component during Tuesday is intentional: many candidates shortcut API and data model design by jumping to the architecture, which causes them to design the schema around the infrastructure they've already drawn rather than around the queries the product actually needs.

Wednesday is for full timed practice on a Tier 2 problem. Set a 45-minute timer and attempt the Design Twitter Feed end to end: requirements scoping, capacity estimation, API design, data model, HLD with derived components, one deep dive with named tradeoffs, and at least three failure mode analyses. After the timer, score yourself on all 7 rubric dimensions without looking at the worked examples in this document. Write down your scores. Identify the single lowest-scoring dimension. Spend the remaining 30 minutes doing the targeted drill from this document for that specific dimension. The goal of Wednesday is to discover your current weakest dimension under realistic time pressure, then do targeted repair immediately while the failure is fresh.

Thursday is for tradeoffs fluency. Pick three architectural decisions from problems you have already studied: for example, fan-out-on-write versus fan-out-on-read for Twitter Feed, token bucket versus sliding window for Rate Limiter, and Cassandra versus PostgreSQL for notification storage. For each decision, write the steelman case for the alternative you would not choose — a 3-sentence genuine argument for why the non-obvious option is better. If you cannot write a genuine steelman, you don't understand the tradeoff; you've memorized a conclusion. After writing all three steelmans, write the single decision criterion for each: the specific measurable property of the system that would make you choose the alternative over your default. Thursday builds the tradeoff fluency that separates strong candidates from average ones: average candidates know which option is used in famous systems; strong candidates know why and under what conditions the answer changes.

Friday is for a full mock interview simulation. Invoke `/interview` in Claude Code and run a complete 45-minute mock interview on a problem you have not studied yet — choose a Tier 1 or Tier 2 problem from the curriculum problem bank that is new to you. After the session ends and you receive your score, compare the per-dimension scores against your Wednesday scores on the same dimensions. Determine whether your Thursday tradeoffs drill improved your Gap 6 score, and whether your Monday and Tuesday isolated drills have carried over to improve your requirements scoping and data model scores in a full timed attempt. Write a brief session log entry noting your two most improved dimensions and your single most persistent weak dimension — this entry becomes the input for the following Monday's drill plan, keeping the weekly cycle targeted rather than generic.
