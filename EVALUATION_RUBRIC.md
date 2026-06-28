# System Design Interview Self-Evaluation Rubric

This rubric is for scoring your own performance after completing a system design practice session, calibrated to Apple ICT3 (senior engineer equivalent). After each practice attempt, score every dimension honestly against what you actually said — not what you meant to say or knew in your head. Once scored, identify your lowest-scoring dimension and go to `GAP_REMEDIATION.md` to complete the targeted drill before your next attempt.

---

## Dimension 1: Requirements Scoping — 10%

**What it measures:** Whether the candidate identifies both functional and non-functional requirements, makes explicit out-of-scope decisions, and states assumptions before touching the design.

**Score 0:** Candidate jumps directly into drawing architecture within 60 seconds. No requirements discussed at all. The interviewer asks "can you design a chat system?" and the candidate immediately responds "so we'll have a web server, an API layer, a database..." — no questions asked, no scope declared, no NFRs named.

**Score 1:** Candidate asks 1-2 questions but misses critical NFRs. Asks "how many users?" but does not ask about read/write ratio, latency SLA, or consistency requirements. No out-of-scope declarations made. Proceeds to design after the first answer without pushing further.

**Score 2:** Identifies core functional requirements and asks about DAU/MAU, read/write ratio, and 2-3 NFRs. Makes some scope declaration ("I'll skip reactions for now") but does not explicitly surface the consistency vs. availability tradeoff or ask which latency SLA matters most.

**Score 3:** Systematically covers all five categories — scale/load, delivery semantics, latency SLAs, ordering/deduplication, compliance/opt-out. Makes explicit "I am treating X as out of scope for this session" declarations. States assumptions that would materially change the architecture if wrong. Explicitly asks which NFR is the hardest constraint.

**Anti-patterns:**
- Saying "I'll just assume this is Google scale" without asking — not because the scale is wrong, but because the direction of the assumption matters; assuming too large drives over-engineering while assuming too small produces a toy design, and neither serves you.
- Asking "what's the DAU?" and then never using that number in any estimation or architectural decision — this is requirements-gathering theater that signals the candidate does not actually know what to do with the answer.
- Treating all requirements as equally important — a strong candidate explicitly prioritizes which NFRs will drive the most architectural decisions, e.g., "consistency seems like the hardest constraint here; everything else I can optimize around."

**Strong answer example:**
> "Before I draw anything — can I take 3 minutes on requirements? Functionally, I want to scope to: send a message, receive a message, and view message history. I'm explicitly leaving out read receipts, reactions, and voice/video calls unless you want to add them. For scale: are we targeting WhatsApp-level 2 billion users, or is this an enterprise internal tool with maybe a few million? That changes the fan-out architecture significantly. On latency: for message delivery, what's the SLA — 100ms? 500ms? And on consistency — if a message appears slightly out of order for a second or two before correcting itself, is that acceptable, or do we need strict per-conversation ordering guarantees? Those two answers will drive my storage and queue choices more than anything else."

---

## Dimension 2: Capacity Estimation — 15%

**What it measures:** Whether the candidate translates DAU/MAU into QPS, storage, and bandwidth; whether the estimation identifies the bottleneck resource rather than being arithmetic theater; whether the estimation output is explicitly connected to subsequent architectural decisions.

**Score 0:** No estimation attempted. Candidate proceeds directly from requirements to architecture without any quantification. Cannot explain later why they chose a distributed cache over a single database, or why they need sharding at all.

**Score 1:** Mentions "it needs to scale" or "we'll need multiple servers" without any numbers. Or attempts estimation but the numbers are off by multiple orders of magnitude (e.g., calculates 10 QPS for a system with 1 billion DAU) without noticing or correcting.

**Score 2:** Calculates DAU → daily requests → peak QPS correctly. Arrives at a rough storage estimate. But the estimation is disconnected from architecture — candidate never says "this number tells me the bottleneck is X" and never references the estimation when making later design choices.

**Score 3:** Drives estimation SLA-first — identifies which operation type is most latency-sensitive and works backward from there. Calculates QPS, storage, and bandwidth with correct order-of-magnitude reasoning, showing the derivation steps rather than asserting numbers. Explicitly connects estimation output to architectural decisions: "We're at 500K writes per second, which tells me a single database is impossible and sharding is a day-one decision, not a future optimization."

**Anti-patterns:**
- Completing the estimation section and then completely ignoring all the numbers in the rest of the session — this is the most common failure mode; estimation becomes a checkbox that satisfies the interviewer's first prompt but influences nothing.
- Stating "1 million QPS" without any derivation — suspiciously round numbers with no math shown signal that the candidate memorized the answer rather than computed it.
- Calculating storage without mentioning retention policy or data growth over time — a system that stores 5TB per day indefinitely is architecturally different from one that stores 5TB per day with a 90-day TTL.

**Strong answer example:**
> "Let me estimate before I design. 500 million DAU, each user sends an average of 10 messages per day — that's 5 billion messages per day. Peak is roughly 3x average, so 180,000 writes per second at peak. Each message is around 1KB of payload plus metadata, so storage is 5TB per day before any compression. After 2:1 compression, roughly 2.5TB per day, or about 900TB per year if we retain everything. At 180K writes per second, a single relational database is the first bottleneck — we cross the write capacity ceiling of a well-tuned Postgres around 50K writes per second. So sharding is not optional; it drives the entire data model. That's the constraint I'll design around first."

---

## Dimension 3: API Design — 10%

**What it measures:** Whether the candidate models resources correctly, uses HTTP methods with proper semantics, handles authentication, pagination, and idempotency where required, and designs the API from the consumer's perspective.

**Score 0:** No API design discussed at all. Candidate describes the system entirely in terms of internal components — message queues, databases, fan-out workers — without ever specifying the external interface that clients would call.

**Score 1:** Mentions "a POST endpoint to send a message" without URL structure, request/response shape, auth headers, pagination, or error contracts. The endpoint description is too vague to implement.

**Score 2:** Defines 2-3 core endpoints with correct HTTP verbs and reasonable URL structure. Includes request and response fields. Mentions authentication ("Bearer token in the header") but does not address cursor-based pagination on list endpoints or idempotency on write endpoints.

**Score 3:** Correctly models resources rather than actions. Chooses HTTP verbs semantically — explains why PUT vs PATCH vs POST for the specific operation. Includes cursor-based pagination (not offset) on list endpoints with explicit reasoning. Includes idempotency keys on write operations where at-least-once delivery matters. Specifies error response shape. Mentions rate limiting at the API gateway layer as a reliability concern.

**Anti-patterns:**
- Using POST for every operation including reads — e.g., "POST /messages/get" — this indicates RPC-style thinking rather than resource modeling, which signals gaps in REST semantics.
- Designing endpoints around internal implementation rather than external consumer needs — e.g., naming an endpoint `/database/insert_message` or `/fan_out_worker/enqueue` reveals the internal architecture to the client and creates coupling that is expensive to change later.
- Omitting pagination entirely on endpoints that return lists — an endpoint that returns all messages in a conversation with no pagination will fail in production the first time a long conversation is requested, and leaving this out in an interview signals a blind spot in reliability thinking.

**Strong answer example:**
> "For the core send operation: POST /v1/conversations/{conversation_id}/messages, JSON body with content, content_type, and an idempotency_key field. The idempotency key is non-negotiable here because we're using at-least-once delivery and a client that times out will retry — without the idempotency key, retries create duplicate messages. Response is 201 Created with the full message object including a server-assigned message_id and server timestamp. For history retrieval: GET /v1/conversations/{conversation_id}/messages with a cursor parameter and a limit parameter, default limit 50. Cursor-based rather than offset because offset pagination is broken when new messages arrive mid-page — the cursor anchors to a specific message_id and timestamp so you always get a consistent window."

---

## Dimension 4: Data Model & Storage — 15%

**What it measures:** Whether the candidate selects the right database type for the access patterns, designs a schema with an explicit index strategy, and aligns storage choices to the data access patterns established during requirements.

**Score 0:** Says "I'll use a database" without specifying SQL vs NoSQL, relational vs document, or any rationale. No schema, no index discussion, no acknowledgment that the storage choice matters.

**Score 1:** Picks a specific database (e.g., "I'd use PostgreSQL") but cannot articulate why over alternatives. Schema is vague — "we'd have a messages table with the relevant fields." No index discussion, no acknowledgment of how the query patterns drive the schema design.

**Score 2:** Correctly picks a DB type for the primary use case with a brief justification. Defines core tables or collections with primary keys and a foreign key relationship. Mentions one index for the most obvious query pattern. Does not discuss secondary storage (blob store, cache), does not address eventual consistency implications.

**Score 3:** Explicitly matches storage choices to the access patterns identified earlier — "we need high write throughput with time-ordered reads by conversation_id, which is a textbook Cassandra use case." Designs schema with explicit primary key strategy, clustering key rationale, and at least two index decisions explained. Identifies the secondary storage concerns (separate blob store for media, cache for hot conversations). Acknowledges where eventual consistency in the storage tier creates observable tradeoffs at the application layer.

**Anti-patterns:**
- Defaulting to PostgreSQL for every system regardless of access patterns — choosing Postgres for a 180K writes-per-second messaging system without acknowledging the sharding complexity this creates signals an incomplete mental model of when relational databases break down.
- Designing tables without reference to query patterns — a schema with a composite primary key that does not match the actual query access pattern will produce correct-looking SQL that is full table scans in production; schema design divorced from access patterns is the most common data modeling failure.
- Forgetting that read-heavy and write-heavy systems require fundamentally different storage architectures — choosing a strongly consistent relational store for a 100:1 read-to-write system is a correctness choice that trades read performance; this tradeoff must be explicit.

**Strong answer example:**
> "The message store has a very specific access pattern: high write throughput, and reads that are always time-ordered within a single conversation. That pattern maps directly to Cassandra — I'd use conversation_id as the partition key and a composite clustering key of (created_at, message_id) in descending order, so the most recent messages are physically adjacent on disk and history queries are a range scan without a sort step. For user profiles and conversation metadata — relationships and joins matter there, so that's Postgres. Media attachments go to S3; I store only the S3 key in the message record, not the blob itself. The index I need to add is a secondary index on sender_user_id for the sent-messages view, but I'd materialize that in a separate Cassandra table keyed on user_id to avoid Cassandra's secondary index limitations under high cardinality."

---

## Dimension 5: High-Level Architecture — 20%

**What it measures:** Whether the candidate draws the correct components, avoids unnecessary single points of failure, shows clear data flow, and gives each component a single well-defined responsibility that is motivated by a specific requirement or constraint.

**Score 0:** Draws a single box labeled "server" connected to a single box labeled "database." No component separation. No data flow direction. No discussion of why any component exists or what it is responsible for.

**Score 1:** Has multiple components but the division is arbitrary — components are added because they seem like things you would have in a large system rather than because each one solves a specific problem identified during requirements. Data flow is ambiguous or has unexplained circular paths.

**Score 2:** Correct major components for the use case — API gateway, application servers, message queue, database, cache. Data flow is mostly correct. A few obvious single points of failure that would be surfaced by an interviewer prompt. Does not proactively identify which components are stateless vs stateful.

**Score 3:** Every component exists because of a specific requirement or constraint — explicitly named at the time of introducing the component. No component has more than one stated responsibility. Data flow shows both the synchronous happy path and the asynchronous paths. Proactively distinguishes stateless components (horizontally scalable with no coordination overhead) from stateful components (require careful scaling strategy). Identifies SPOF candidates without being asked and explains mitigation.

**Anti-patterns:**
- Adding a Kafka queue without explaining what problem it solves — queues are not free; they add operational complexity, tail latency, and a new failure surface; introducing Kafka without articulating "this decouples X from Y, which is necessary because Z" signals a cargo-cult architecture choice.
- Designing a cache without specifying what it caches, how it gets populated (write-through, write-behind, or cache-aside), what the TTL is, and what the eviction policy is — "I'd put Redis in front of the database" is not a cache design.
- Drawing a load balancer in front of every tier without discussing what it is balancing — round-robin assumes stateless servers; session affinity has different implications for WebSocket connections; not distinguishing these signals that the load balancer is a diagram decoration rather than a considered component.

**Strong answer example:**
> "Here is the HLD. The client connects over WebSocket to an API gateway that handles auth and rate limiting — the gateway is deliberately stateless so it scales horizontally with zero coordination. Behind the gateway, the message service is the write path: it writes the message to Cassandra, publishes an event to Kafka, and acknowledges the sender. I'm separating the fan-out from the write path because a synchronous fan-out to 500 recipients would block the sender's response for seconds. The fan-out worker consumes from Kafka and handles delivery to online recipients over WebSocket and to offline recipients via the push notification service. Redis sits in front of Cassandra for recent message history reads — the last 50 messages in a conversation are fetched hundreds of times per day and rarely change; that's a perfect cache use case. The two stateful components are Cassandra and Kafka; both require coordination on scaling and are my primary SPOF candidates."

---

## Dimension 6: Deep Dive & Tradeoffs — 20%

**What it measures:** Whether the candidate identifies the 1-2 most interesting components to go deep on, articulates tradeoffs with specific alternatives named and explicitly rejected, and demonstrates design depth beyond component selection.

**Score 0:** No deep dive at all. Stops at the HLD diagram. When asked "how does the fan-out actually work at 10K recipients?" responds with component names ("the fan-out worker reads from Kafka and sends to clients") rather than mechanisms.

**Score 1:** Attempts a deep dive on the wrong component — goes deep on a simple CRUD layer or a well-understood commodity component like the load balancer, rather than on the fan-out logic, the consistency mechanism, or whatever has the most interesting engineering tradeoffs for this specific system. Tradeoffs are abstract: "it has pros and cons."

**Score 2:** Picks a reasonable component to go deep on. Explains the internal mechanism with reasonable correctness. Names one specific tradeoff but does not fully explain the decision criteria or commit clearly to a choice.

**Score 3:** Proactively selects the 1-2 components with the most interesting engineering tradeoffs — not just the most complex-sounding components. For each, names the alternatives explicitly and states the decision criterion before committing to a choice. Shows awareness of second-order effects: "write-ahead replication improves read consistency but means writes block when the replica is slow, which is the failure mode I care most about." Takes a position and defends it under pushback.

**Anti-patterns:**
- Spending 5 minutes explaining how a load balancer distributes traffic round-robin — this is not deep-dive territory; it is a well-known commodity behavior that every engineer should know; choosing this as a deep dive signals an inability to identify where the interesting engineering actually lives.
- Presenting a tradeoff without taking a position: "We could use fan-out on write or fan-out on read — both have pros and cons" — this is not a tradeoff analysis; it is a refusal to commit; the interviewer wants to see you reason under uncertainty and arrive at a defensible conclusion.
- Describing a design choice that optimizes for the wrong dimension for the stated requirements — e.g., choosing eventual consistency for a payment deduplication system "because it's faster" when the requirements explicitly included exactly-once semantics.

**Strong answer example:**
> "I want to go deep on the fan-out design because that is where this system will break at scale, and there are two fundamentally different approaches. Fan-out on write means we push the message to every recipient's inbox at send time. Fan-out on read means we store it once in the sender's outbox and compute the recipient list at read time. For a messaging system where conversations typically have 2 to 100 participants and read QPS is 10x write QPS, fan-out on write is the correct choice — we pay a higher cost per write to get O(1) reads, and that tradeoff is favorable because reads dominate. The failure mode I am trading away is large groups: if a group with 50,000 members sends a message, fan-out on write generates 50,000 writes synchronously. My mitigation is a threshold: under 1,000 recipients, fan-out on write; over 1,000 recipients, fan-out on read with pre-materialization for members who were active in the last 24 hours. This hybrid captures 95% of conversations in the fast path while keeping the worst-case bounded."

---

## Dimension 7: Failure Modes & Scaling — 10%

**What it measures:** Whether the candidate identifies what breaks first under load, describes graceful degradation paths with user-visible impact specified, and explains how the architecture evolves at meaningful scale thresholds.

**Score 0:** No failure discussion initiated. When the interviewer asks "what happens if your Kafka cluster goes down?" the candidate responds with "we'd fix it" or "we'd use a backup cluster" without describing how the system behaves for users during the outage or what data guarantees are preserved.

**Score 1:** Identifies one obvious failure mode when directly asked, but the mitigation is vague — "we'd add redundancy" or "we'd have retries." Treats all failures as binary up/down events. No discussion of graceful degradation. Cannot describe what the user experiences during a partial failure.

**Score 2:** Identifies 2-3 failure modes proactively. Describes a retry mechanism or circuit breaker for one of them. Can describe the first scaling step (add more application servers) but does not reason through the stateful components — where "add more servers" does not apply.

**Score 3:** Identifies the component most likely to be the first bottleneck under load — this should match the bottleneck identified during capacity estimation. Describes failure modes with the user-visible impact specified for each one. Distinguishes graceful degradation (reduced functionality, still serving) from hard failure (service down). Explains the scaling evolution at meaningful thresholds: what changes architecturally at 1M users, 10M users, and 100M users, and which earlier decisions will need to be revisited at which scale.

**Anti-patterns:**
- Saying "we'd add more servers" for every scaling challenge — horizontal scaling solves stateless component problems; for stateful components like the message store, the message queue, and the session store, "more servers" is not a complete answer and naming it as one signals unfamiliarity with distributed systems coordination costs.
- Treating all failure severities as equivalent — a design that can articulate the difference between "message delivery delayed by 30 seconds but not lost" and "message permanently lost" is more sophisticated than one that treats both as "the system is down"; interviewers respond strongly to candidates who reason about failure impact on users rather than failure impact on infrastructure.
- Not knowing the CAP theorem implications of the storage choice made in Dimension 4 — if you chose Cassandra for availability and partition tolerance, you must be able to describe what the consistency failure looks like: stale reads, write conflicts, and the repair mechanism; choosing AP storage and not knowing its failure behavior is an inconsistency that a senior interviewer will probe.

**Strong answer example:**
> "The first component to saturate under load is Kafka consumer lag — specifically the fan-out worker falling behind during a traffic spike. When that happens, I do not want to fail hard; I want to degrade gracefully. My degradation strategy: if a message has been sitting in the queue longer than 60 seconds, we stop attempting real-time push delivery for that message and fall back to a pull model — the recipient sees the message when they next open the app. That degradation is invisible to 90% of users and prevents the queue backup from cascading into message loss. For Cassandra, the hot partition risk is the failure mode I track at scale — a very active group chat can concentrate writes on a single partition key until that node is overwhelmed. My mitigation is a write-time shard suffix appended to conversation_id, at the cost of scatter-gather reads for history. At 100 million users, the architectural decision I would revisit is the single global Kafka cluster — I would move to per-region clusters with cross-region replication for international delivery, because global cluster latency from Asia-Pacific to US becomes visible in the end-to-end delivery SLA."

---

## Scoring Summary

| Dimension | Weight | Max Points |
|-----------|--------|------------|
| Requirements Scoping | 10% | 3 |
| Capacity Estimation | 15% | 3 |
| API Design | 10% | 3 |
| Data Model & Storage | 15% | 3 |
| High-Level Architecture | 20% | 3 |
| Deep Dive & Tradeoffs | 20% | 3 |
| Failure Modes & Scaling | 10% | 3 |
| **Total** | **100%** | **21** |

**Weighted score formula:**

```
weighted_score = sum of (dimension_score * dimension_weight_as_decimal)
```

For example: Requirements 2 × 0.10 + Estimation 2 × 0.15 + API 1 × 0.10 + Data 2 × 0.15 + HLA 2 × 0.20 + Deep Dive 1 × 0.20 + Failure 1 × 0.10 = 1.65 out of 3.0

Or use raw score out of 21: **15 = interview-ready. 18+ = strong senior performance.**

---

## Post-Session Protocol

**Step 1.** Calculate your raw score. Write down your score for each dimension honestly — not what you intended to say but what you actually said out loud or put on the whiteboard. A dimension where you thought of the right answer but did not say it counts as 0 for that dimension. This is the hardest part of self-evaluation and the part most candidates get wrong by giving themselves credit for internal reasoning that never reached the interviewer.

**Step 2.** Identify your lowest-scoring dimension. If two dimensions are tied at the same score, pick the one with higher weight, since it has greater impact on your weighted performance. If Requirements Scoping (10%) and High-Level Architecture (20%) are both at score 1, work on HLA first. This dimension is your primary gap for the next practice session.

**Step 3.** Go to `GAP_REMEDIATION.md`, find the section for your primary gap dimension, and complete the listed drill before your next attempt. Do not do two drills simultaneously — depth of deliberate practice on one gap produces faster improvement than shallow practice spread across many dimensions. One focused drill, one session, then re-attempt and re-score.
