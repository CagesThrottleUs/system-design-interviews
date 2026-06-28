# Message Queue Patterns

## Intent

Decouple producers from consumers by routing work through a durable intermediary, enabling asynchronous processing, traffic buffering, and reliable fan-out across independent services.

---

## The Problem

Synchronous request-response coupling is the default design posture for most engineers because it is simple to reason about: you call a function or an HTTP endpoint, you wait for a result, you proceed. This model works well until three failure modes appear — and they appear in almost every production system at scale.

The first failure mode is a slow downstream. Video transcoding takes 10 minutes. PDF generation takes 30 seconds. Machine learning inference takes 2 seconds. If your API handler waits synchronously for any of these operations, clients hold open HTTP connections for the duration. At 100 concurrent uploads, you hold 100 connections for 10 minutes each. Your connection pool exhausts, new requests queue, timeouts cascade, and the system appears down even though the actual transcoding workers are functioning perfectly. The problem is not the work; it is the synchronous coupling of the upload path to the processing path.

The second failure mode is a temporarily unavailable downstream. Your email delivery service — Postmark, SendGrid, or an internal SMTP relay — goes down for 20 minutes during a deployment. Every call to `send_email()` in your application fails with a network error. Without a queue, those emails are simply lost. The user who just registered never gets their welcome email. The order confirmation goes nowhere. The system has no memory that the work was requested. With a queue, the email request is persisted durably the moment it is enqueued; the consumer processes it when the email service comes back online.

The third failure mode is fan-out to multiple consumers. An "order placed" event in an e-commerce system must simultaneously trigger billing (charge the card), fulfillment (pick the item), inventory (decrement stock), notifications (send confirmation email and SMS), and analytics (record the conversion). If the order service calls each of these synchronously in sequence, a failure in fulfillment blocks the notification. A slow analytics write delays the response to the customer. And adding a sixth consumer — say, a fraud detection service — requires modifying the order service itself. A message queue inverts this relationship: the order service emits one event; every interested consumer subscribes independently. Adding a new consumer requires no change to the producer.

---

## Core Queue Semantics

### At-most-once Delivery

At-most-once delivery guarantees that a message is delivered zero or one time — it will never be delivered twice, but it may be dropped. The implementation is straightforward: delete the message from the queue (or fire-and-forget without acknowledgment) before or without waiting for processing confirmation. If the consumer crashes mid-processing, the message is simply gone — no retry, no redelivery.

This semantics is appropriate for workloads where occasional loss is acceptable but duplication is not. Metrics collection is the canonical example: if your application emits 10,000 data points per second and one data point is dropped during a network hiccup, your dashboards are slightly inaccurate for that second. This is tolerable. But if the metrics consumer receives the same data point twice and double-counts it, your throughput graphs show a spike that never happened — which is worse for on-call alerting than the slight inaccuracy from a dropped point.

Real-time gaming telemetry, sampling-based analytics pipelines, and heartbeat signals are domains where at-most-once is the right tradeoff.

### At-least-once Delivery

At-least-once delivery guarantees that a message will be delivered until explicitly acknowledged — it may be delivered multiple times, but it will never be permanently lost. The implementation requires the consumer to acknowledge successful processing after completing the work. If the consumer crashes after processing but before sending the acknowledgment, the queue has no way to know the work was done, so it redelivers the message to another consumer (or the same consumer after a recovery).

This is the default delivery semantics for virtually every production system that cares about data integrity. The implication is critical: every consumer in an at-least-once system must be idempotent. An idempotent consumer produces the same outcome regardless of how many times it receives the same message. Inserting a row with a unique constraint on `message_id` is idempotent — the second insert fails silently. Incrementing a counter without deduplication is not idempotent — each redelivery increments by one again.

Idempotency is not free. Common implementation patterns include maintaining a `processed_message_ids` store (a Redis set or a database table indexed by message ID) and checking before processing; using database upsert semantics keyed on a stable message identifier; or designing operations that are naturally idempotent (set-based operations rather than increment-based).

### Exactly-once Delivery

Exactly-once delivery — no loss and no duplication — is the holy grail of distributed messaging. Within a single system it is achievable with reasonable overhead. Across a distributed system it is far more difficult, because it requires coordinated consensus at every hop: the producer commit, the queue storage, and the consumer processing must all succeed atomically or all fail atomically.

Apache Kafka introduced its transactional API in version 0.11 (2017), which provides true exactly-once delivery within the Kafka cluster — the producer's write and the consumer's offset commit are atomic. However, this only covers the Kafka-internal path. If the consumer writes to an external database, the Kafka commit and the database write are two separate operations that can fail independently. True end-to-end exactly-once requires either a distributed transaction across Kafka and the external store, or — more practically — idempotent consumers combined with at-least-once delivery.

In practice, when engineers say "exactly-once" in a system design interview, they mean at-least-once delivery with idempotent consumers and a deduplication layer. This is the production-realistic answer. The deduplication store (a Redis set of seen message IDs with a TTL matching the message retention period) prevents double-processing; the at-least-once delivery guarantee prevents loss. The result is effectively-once behavior from the end-to-end system's perspective without requiring distributed transaction coordination.

---

## Kafka vs SQS vs RabbitMQ

### Apache Kafka

Kafka is not a message queue in the traditional sense. It is a distributed commit log — an append-only, partitioned, replicated sequence of records. Producers append events to topics; those events are retained for a configurable period (from hours to indefinitely, bounded only by disk). Consumers do not pop messages off the queue and delete them; instead, each consumer tracks its own offset into the log and advances it as it reads. Because messages are never deleted upon consumption, multiple independent consumers can read the same topic simultaneously, each at their own pace, and any consumer can replay from an arbitrary point in history.

This architecture makes Kafka exceptional for three use cases: event sourcing (the log is the source of truth; the database is a materialized view of the log), audit trails (every event is retained for compliance lookups), and stream processing (Kafka Streams and Apache Flink consume the log and transform it into derived streams). LinkedIn, Kafka's creator, processes approximately 7 trillion messages per day as of 2023 across their activity, monitoring, and recommendation pipeline. Kafka's throughput — sustained writes in the hundreds of MB/sec per broker — is unmatched among general-purpose messaging systems.

The cost is operational complexity. Kafka requires careful partition count planning before deployment: partitions are the unit of parallelism and cannot be reduced after creation (only increased). Consumer group lag requires active monitoring — if consumers fall behind, the queue of unprocessed messages grows until retention expires. Schema evolution requires a schema registry (Confluent Schema Registry or AWS Glue Schema Registry) to ensure producers and consumers agree on message format as fields are added or removed. Kafka is not a drop-in managed service in the way SQS is — even with managed offerings like Confluent Cloud or Amazon MSK, engineering teams need to understand its internals to operate it successfully.

### Amazon SQS

SQS is a fully managed, traditional point-to-point queue. Producers enqueue messages; consumers dequeue and delete them. There is no log, no offset, no replay. Once a message is deleted from the queue (or expires after up to 14 days), it is gone.

SQS's standard queues offer at-least-once delivery and best-effort ordering. FIFO (First-In-First-Out) queues offer exactly-once processing within a 5-minute deduplication window and strict ordering, but are capped at 3,000 messages per second with batching (300 without). The deduplication ID is the mechanism: within a 5-minute window, SQS drops any message with a duplicate `MessageDeduplicationId`, making the producer responsible for assigning stable IDs to idempotency groups.

The killer feature of SQS is operational overhead: zero. No cluster to provision, no replication factor to configure, no partition count to plan. SQS scales transparently and bills per API call. For teams that need reliable task queue semantics and do not need replay, SQS is almost always the right answer.

The visibility timeout deserves attention because it is the mechanism behind at-least-once delivery in SQS and is a common source of bugs. When a consumer calls `ReceiveMessage`, the message is not deleted — it is made invisible to all other consumers for the visibility timeout duration (30 seconds by default, configurable up to 12 hours). If the consumer processes the message and calls `DeleteMessage`, the message is removed. If the consumer crashes or its processing takes longer than the visibility timeout, the message becomes visible again and is redelivered to another consumer. The consumer must ensure that its processing time reliably completes within the visibility timeout, or must actively extend the timeout via `ChangeMessageVisibility` for long-running jobs.

### RabbitMQ

RabbitMQ is a message broker that implements the AMQP (Advanced Message Queuing Protocol) 0-9-1 specification. Its core abstraction is the exchange: producers publish messages to exchanges, not directly to queues. Exchanges route messages to queues based on routing keys and binding rules. Exchange types determine routing behavior: a `direct` exchange routes messages to queues whose binding key exactly matches the routing key; a `topic` exchange supports wildcard routing (`orders.*.created` matches `orders.usa.created` and `orders.uk.created`); a `fanout` exchange copies every message to all bound queues regardless of routing key; a `headers` exchange routes based on message headers rather than the routing key.

This routing flexibility is RabbitMQ's core differentiator. Complex event routing — "send all fraud alerts to the security queue, all order events to both fulfillment and billing queues, and all events above a certain priority to the emergency queue" — is expressible natively in RabbitMQ's exchange topology without application-layer routing logic. Kafka and SQS require the application to implement equivalent fan-out through code.

RabbitMQ's practical ceiling is lower than Kafka's. Performance benchmarks from the RabbitMQ team (2022 blog, "RabbitMQ Performance Measurements") show sustained throughput in the range of 40,000–100,000 messages per second on typical hardware, compared to Kafka's millions per second per partition. For the majority of production workloads that do not approach those numbers, RabbitMQ's simpler mental model and rich routing are compelling. The operational model — a cluster of 3 nodes with mirrored queues or, in newer versions, quorum queues — is more familiar to engineers accustomed to managing traditional databases.

RabbitMQ does not support replay. Once a message is acknowledged and deleted, it is gone. This makes RabbitMQ unsuitable for event sourcing or any workload that requires auditing the historical message stream.

---

## Consumer Group Patterns

In Kafka, the consumer group is the mechanism for horizontal scale-out combined with independent consumption. A topic is divided into P partitions. A consumer group has C consumers. Each partition is assigned to exactly one consumer in the group at a time; if C < P, some consumers handle multiple partitions; if C > P, some consumers are idle (there is no benefit to having more consumers than partitions). When a consumer in the group fails, Kafka rebalances partition assignments among the remaining consumers.

Multiple consumer groups reading the same topic each receive all messages independently. This is the pub/sub model: the "order placed" topic might have a `billing-consumer-group`, a `fulfillment-consumer-group`, and an `analytics-consumer-group`, each consuming at their own pace and maintaining their own offset. Adding a new downstream consumer — say, a fraud detection service — requires creating a new consumer group without any change to the producer or existing consumers. This decoupling is one of the primary reasons Kafka is the default choice for systems where many downstream services must react to the same event stream.

In SQS, a standard queue distributes messages across consumers using the competing consumers pattern: multiple consumers all poll the same queue, and SQS delivers each message to at most one consumer (within a visibility timeout window). This enables horizontal scale-out but provides no built-in pub/sub. To achieve fan-out to multiple independent consumers in SQS, the standard pattern is SNS → multiple SQS queues: an SNS topic receives the event and fans it out to multiple SQS queues, one per consumer service. Each SQS queue delivers to its consumer independently, providing the same isolation guarantee as Kafka consumer groups, but with the operational simplicity of SQS.

---

## Dead Letter Queues

A dead letter queue (DLQ) is a secondary queue that receives messages that have failed processing beyond a configurable retry threshold. Every production message queue should have a DLQ configured. This is not an optimization — it is a correctness requirement.

Without a DLQ, a poison pill message — one that consistently fails processing due to malformed data, an unexpected null, a schema mismatch between producer and consumer, or a bug introduced in a new consumer deployment — will loop indefinitely. The consumer receives the message, fails, returns it to the queue, and repeats. The message consumes processing capacity on every retry cycle. It may block subsequent messages in FIFO queues. It generates alert noise in error logs. And because nothing ever removes it from the queue, it continues looping until someone manually purges it or the retention period expires, by which time every other message may have also expired.

With a DLQ, after N failed delivery attempts (commonly 3 or 5), the message is moved out of the hot queue and into the DLQ, where it stops consuming processing capacity. The DLQ is also your forensics tool: when you receive an alert that DLQ depth has grown, you inspect the messages to understand exactly what failed and why. You then either fix the consumer bug (and replay the DLQ messages into the main queue), fix the producer to stop sending malformed messages, or explicitly discard messages that are unrecoverable.

Operationally, DLQ depth must be alarmed. A DLQ at zero depth with no alarm provides false confidence — you will not know when poison pills start accumulating. A sensible alarm triggers at DLQ depth ≥ 1 for critical queues and DLQ depth > N (some tolerable threshold) for high-volume queues where a small number of DLQ messages is expected from transient errors.

---

## Back-pressure

Back-pressure is the signal that propagates upstream when a system is overloaded. In a messaging context, back-pressure manifests when producers enqueue messages faster than consumers can process them — the queue depth grows.

An unbounded queue absorbs this growth indefinitely. This sounds like a feature — the system keeps accepting work even when consumers are slow — but it is a hidden failure mode. As the queue grows, the latency between when a message is enqueued and when it is processed increases proportionally. An "order placed" event that should trigger a confirmation email within 5 seconds might sit in a queue for 45 minutes if consumer throughput is overwhelmed during a flash sale. The user experience degrades silently while the queue depth metric tells a story that only operators are watching.

Bounded queues apply explicit back-pressure: when the queue reaches its capacity limit, new enqueue operations are rejected with an error. The producer must decide what to do with that rejection — options include retrying with exponential backoff (which risks compounding the overload if many producers retry simultaneously), applying a circuit breaker (stop accepting new work from upstream until the queue clears), writing the overflow to a persistent fallback store, or dropping the message (acceptable only for non-critical data). Rejecting the message forces the back-pressure signal to propagate upstream to the user or the load balancer, where it can be handled explicitly.

Kafka bounds storage consumption through its topic retention configuration: messages older than the retention period (7 days by default) are deleted regardless of whether they have been consumed. This does not limit queue depth per se, but it prevents the log from growing without bound on disk. SQS messages have a maximum retention of 14 days and a maximum message size of 256 KB; messages older than the retention period are automatically deleted. RabbitMQ supports queue-level message TTL (`x-message-ttl`) and queue-level maximum length (`x-max-length`) with configurable overflow behavior (drop head, reject publishers, or drop newest) — these are the tools for implementing bounded queues in RabbitMQ.

---

## Trigger Signals

- An HTTP endpoint takes >5 seconds because it is waiting for a slow downstream (transcoding, ML inference, external API, email delivery)
- A downstream service outage causes data loss because callers receive errors and do not retry
- One event must trigger multiple independent downstream services (fan-out)
- Write throughput to the primary database must be smoothed — batching database writes through a queue reduces peak write pressure
- Audit trail requirement: every event that occurred must be queryable retrospectively
- A consumer service is slower than the producer and you need to buffer the difference without blocking the producer
- You need to decouple deployment schedules — producer and consumer must be able to deploy independently without coordinating downtime

---

## Problems Where It Applies

**Notification System.** A user action (a new follower, a comment, a like) generates a notification event. This event must fan out to multiple delivery channels: push notification (APNs/FCM), email, SMS, and in-app notification. A Kafka topic or SNS-to-SQS fan-out pattern delivers the event to independent consumers for each channel. Each channel consumer can fail, scale, and deploy independently. Without a queue, the notification service must synchronously call each channel in sequence — a failed SMS delivery would block the push notification.

**Design YouTube.** When a user uploads a video, the upload service writes the raw file to blob storage and enqueues a transcoding job with the storage path and target resolutions (360p, 720p, 1080p, 4K). Separate transcoding workers consume the queue, each handling one resolution per job. The upload API returns immediately after enqueuing — the user sees "Upload complete; processing in progress" rather than waiting 10 minutes. YouTube processes approximately 500 hours of video uploaded per minute as of 2022 (YouTube Official Blog, 2022); a synchronous transcoding architecture could not absorb that volume without catastrophic connection exhaustion.

**Web Crawler.** The URL frontier is a queue. The crawler discovers N new URLs on each page it visits. Rather than recursively following links synchronously (which would exhaust stack depth and connection pools), each discovered URL is enqueued. A pool of crawl workers consumes the queue, fetches each URL, parses links, and enqueues new discoveries. Priority queues allow high-value domains to be crawled ahead of lower-priority ones. The queue also provides deduplication: a bloom filter or database check prevents re-enqueuing already-crawled URLs.

**Design Uber.** GPS location updates from drivers arrive at approximately 4-second intervals per driver during an active trip. At 1 million concurrent drivers (Uber served ~5.4 million trips per day as of 2019 per their annual report, with peak concurrency in the hundreds of thousands), this represents hundreds of thousands of location events per second. A Kafka topic partitioned by `driver_id` accepts these events from mobile clients and delivers them to multiple consumers: the matching service (to update driver positions for nearby rider requests), the trip service (to update live tracking), and the analytics pipeline (to calculate ETAs and surge pricing). Each consumer processes the same event stream independently at its own throughput.

---

## Trade-offs Table

| Dimension | Kafka | Amazon SQS | RabbitMQ |
|---|---|---|---|
| Model | Distributed commit log (persistent, replayable) | Traditional point-to-point queue (consume-and-delete) | Broker with exchange-based routing (consume-and-delete) |
| Throughput | Millions of messages/sec per cluster (LinkedIn: 7 trillion/day, 2023) | ~3,000 msg/sec (FIFO); near-unlimited (Standard) | ~40,000–100,000 msg/sec per cluster |
| Replay | Yes — consumers maintain offsets; arbitrary replay from any point | No — messages deleted after consumption | No — messages deleted after ack |
| Ordering | Per-partition FIFO; no global ordering | FIFO queues: per-message-group ordering; Standard: best-effort | Per-queue FIFO; priority queues available |
| Routing | Topic-based; consumer groups for fan-out; no built-in content routing | None native; fan-out via SNS → multiple SQS queues | Rich: direct, topic (wildcard), fanout, headers exchanges |
| Operational Cost | High — partition planning, consumer lag monitoring, schema registry | Zero — fully managed, zero configuration | Medium — cluster management, quorum queue configuration |
| Best Use Case | Event sourcing, audit logs, stream processing, high-throughput fan-out | Task queues, job dispatch, decoupled microservices, AWS-native workloads | Complex routing, lower-throughput workloads, AMQP ecosystem |

---

## Real-World Usage

**LinkedIn (Kafka, 2023).** LinkedIn created Kafka in 2011 to handle their activity feed pipeline. As of 2023, LinkedIn's Kafka deployment processes approximately 7 trillion messages per day across use cases including member activity events (profile views, connection requests, job applications), site monitoring (every service emits metrics to Kafka topics consumed by their monitoring pipeline), and stream processing (Samza, their stream processing framework, consumes Kafka topics to generate the LinkedIn feed in real time). The scale — 7 trillion messages per day across clusters in multiple data centers — is achieved through aggressive partition tuning, producer batching (`linger.ms` and `batch.size` configuration), and compression (Snappy codec). (LinkedIn Engineering Blog, "Kafka at LinkedIn," 2023.)

**Uber (Kafka, 2019).** Uber's event streaming architecture, described in their engineering blog post "Building Reliable Reprocessing and Dead Letter Queues with Apache Kafka" (2019), uses Kafka as the backbone for real-time trip events. Driver GPS updates, trip state transitions (requested, accepted, started, completed), and payment events all flow through Kafka topics. Uber's specific contribution to Kafka patterns was the productionized DLQ implementation: rather than using Kafka's native retry mechanisms (which can cause ordering issues), Uber routes failed messages to a separate retry topic with an exponential backoff delay, and after N retries, to a dead letter topic for manual inspection. This pattern is now widely adopted as the standard Kafka DLQ strategy. (Uber Engineering Blog, 2019.)

**Discord (Kafka, 2020).** Discord uses Kafka for message fan-out to online members. When a user sends a message to a Discord server (guild) with 500,000 members, the message event is published to a Kafka topic. Consumers look up which of those 500,000 members are currently online, connected to which gateway servers, and fan-out the message event to each gateway server for delivery over WebSocket. Discord reported maintaining 12 million concurrent WebSocket connections as of 2024 (Discord Engineering Blog, "How Discord Stores Trillions of Messages," 2023). The fan-out problem — a single message potentially triggering hundreds of thousands of downstream deliveries — is precisely the workload that a distributed commit log with multiple consumer groups handles efficiently. (Discord Engineering Blog, 2020.)

---

## Common Mistakes

**Assuming exactly-once semantics without implementing idempotency.** This is the most common and most dangerous mistake in production messaging systems. Engineers sketch a design where "Kafka handles exactly-once" and proceed with a consumer that increments a counter, charges a payment, or sends a notification on each message. When the consumer is redeployed, when a pod crashes after processing but before committing the offset, or when a network partition causes the offset commit to fail, the message is redelivered. The consumer processes it again. The payment is charged twice. The idempotency layer — a `processed_event_ids` table with upsert semantics, a Redis set with a message ID TTL — is not optional; it is load-bearing. Every consumer that performs a non-idempotent side effect must be designed with explicit deduplication. The correct interview answer explicitly names idempotency as a requirement the moment at-least-once delivery is chosen.

**No DLQ in the design.** In every system design that involves a message queue, the DLQ is part of the correct answer. Not mentioning it signals that the candidate has not operated a production messaging system. A queue without a DLQ is a queue that will silently lose messages (if messages expire after max retries) or loop forever on poison pills (if messages never expire). The DLQ must be mentioned alongside the main queue, and the answer should include a brief note on monitoring DLQ depth.

**Using a queue when synchronous communication is correct.** A message queue is the right tool when the response is not needed immediately, when the downstream is slow or unreliable, or when fan-out is required. It is the wrong tool when the caller needs a result to proceed — a login authentication check, a read operation, a synchronous validation. Engineers who have recently learned about messaging sometimes over-apply it, converting all service communication to async and then re-implementing synchronous behavior through callback queues and correlation IDs. This complexity is only justified when the asynchronous properties are actually needed.

**Choosing the wrong partition count in Kafka.** Partition count is one of the most consequential decisions in a Kafka deployment and one of the few that cannot be reversed (increasing is possible; decreasing is not without recreation). Too few partitions: consumer parallelism is limited to the number of partitions, capping throughput. Too many: each partition has overhead in Kafka's controller (leader election, metadata propagation), and consumer group rebalances take longer. The rule of thumb used by Confluent (Confluent Engineering Blog, 2020) is to target partition counts that result in each partition handling approximately 10 MB/sec of throughput at peak, with headroom for anticipated growth. For interview discussions, the key point is acknowledging that partition count is a permanent decision requiring upfront capacity planning.

**Not modeling back-pressure.** A queue design that lacks bounded capacity or consumer autoscaling will fail silently during traffic spikes. If consumers process 1,000 messages per second and a flash sale sends 10,000 messages per second for 30 minutes, the queue depth grows by 540,000 messages. At 1,000 msg/sec processing, clearing that backlog takes 9 minutes after the spike ends — during which time all "real-time" notifications are 9 minutes stale. The correct answer describes either consumer autoscaling (add more consumer pods when queue depth exceeds a threshold), bounded queue behavior with producer-side backoff, or graceful degradation (notify users their confirmation will be slightly delayed). Ignoring back-pressure in a design that advertises low latency is a contradiction that an experienced interviewer will probe.
