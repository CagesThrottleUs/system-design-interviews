# Database Sharding

## Intent

Horizontally partition a database table across multiple independent nodes so that each node owns a disjoint subset of the data, allowing write throughput, storage capacity, and query parallelism to scale beyond what a single machine can provide.

---

## The Problem

A single PostgreSQL instance is a powerful piece of software, but it is ultimately constrained by the machine it runs on. For read-heavy workloads, a well-tuned Postgres instance on modern hardware can handle roughly 10,000–50,000 queries per second; write throughput is lower, typically 5,000–15,000 writes per second, because every write must flush to the WAL. Storage is the slower-moving constraint: a single Postgres instance becomes operationally painful at roughly 5–10 TB of practical storage before the cracks start to show.

What do those cracks look like? At 1 TB, a query that touches a 50-million-row table might return in 10ms because the index fits in the OS page cache and the query planner has fresh statistics. At 8 TB, the same query on a 400-million-row table might take 200ms or more. This happens for several compounding reasons. The query planner's statistics — maintained by `ANALYZE` — become stale faster than `autovacuum` can refresh them, so the planner chooses suboptimal execution plans. B-tree indexes grow deeper as tables expand: a tree with 400M leaf nodes has one or two additional levels compared to 50M rows, meaning more block reads per lookup. Vacuum operations — which reclaim space from dead tuples — take hours rather than minutes on large tables, and during that time write amplification and table bloat degrade performance further. Full table scans, even ones you thought were unreachable because of indexes, start appearing in `pg_stat_statements` because the planner has decided a sequential scan is cheaper than a deeply nested index scan at this scale.

Read replicas address the read side of the problem by fanning out SELECT queries across multiple standby nodes. Instagram operated a PostgreSQL primary with multiple read replicas for years. But read replicas do not help you if writes are your bottleneck, and they do not help at all with storage: every replica holds a full copy of every byte, so a 10 TB primary means 10 TB per replica. When Instagram reached approximately 400 million photos stored as of 2012, they had no choice but to shard their PostgreSQL cluster — no single machine could hold that data, and no replica strategy could distribute write pressure.

Sharding partitions the data so that each node owns a slice. With 8 shards, each node holds one-eighth of the data. Vacuum runs eight times faster. Index trees are eight times shallower. Writes are distributed across eight independent WAL streams. The tradeoff is that your application now must know which shard to query for any given row, and operations that cross shards — JOINs, aggregations, transactions — must be orchestrated at the application level rather than inside the database engine.

---

## Sharding Strategies

### Range-based Sharding

Range-based sharding assigns rows to shards based on contiguous ranges of the shard key. Shard 1 holds `user_id` 1–1,000,000; shard 2 holds 1,000,001–2,000,000; and so on. The routing logic is trivially simple: divide the key by the shard size and you know the destination. Range queries are efficient: "find all orders placed in Q3 2024" maps cleanly to a single shard (or a small set of shards) if the key is `created_at`, because the range falls entirely within one shard's key space.

The fundamental problem with range-based sharding is the hot shard. New data, by definition, always arrives at the high end of the key range. If you shard by `created_at`, every write tonight lands on the shard holding today's timestamp. Historical shards sit idle while the current shard handles 100% of write traffic. This is not a theoretical concern — it is the dominant failure mode of range-based sharding in production. Monotonically increasing keys such as auto-increment IDs, UUIDs v1, and timestamps all exhibit this pathology to varying degrees.

Range-based sharding works well when the write pattern is not append-only. Sharding a `products` table by `product_id` range in an e-commerce catalog that gets updated uniformly across all products avoids the hot shard problem because no range is "newer" than another.

### Hash-based Sharding

Hash-based sharding computes `hash(shard_key) mod N` to determine which of the N shards holds a given row. A well-chosen hash function distributes writes uniformly across all shards regardless of how the key values are distributed — even if all keys are monotonically increasing, the hash output is uniformly distributed. This eliminates the hot shard problem.

The cost is range query expressiveness. A query asking for "all users with IDs between 1,000,000 and 2,000,000" cannot be routed to a single shard because adjacent IDs hash to arbitrary shards. The application must scatter the query to all N shards, collect the results, and merge them. For most user-facing queries that look up a specific entity by its primary key, this is not a problem — the hash gives you the exact shard. For analytics queries over ranges, it is expensive.

Instagram's 2012 sharding migration (Instagram Engineering Blog, 2012) used hash-based sharding on `user_id`. Given that Instagram's primary access pattern is "load this user's photos" — a point lookup, not a range scan — hash-based distribution was the correct choice. Their hash function mapped each `user_id` to a logical shard, and each logical shard mapped to a physical PostgreSQL instance (they over-partitioned into logical shards to make future rebalancing cheaper).

### Directory-based Sharding

A directory-based (or lookup-based) shard router maintains an explicit mapping from entity identifiers to shard locations. Instead of computing a deterministic function over the key, the application asks a shard directory service: "which shard holds `user_id` 8372941?" and routes accordingly.

This indirection layer enables flexibility that neither range nor hash sharding can provide. You can move specific entities to specific shards at any time without changing the routing algorithm — just update the directory entry. You can place high-traffic entities (celebrity accounts with hundreds of millions of followers) on dedicated, beefier shards. You can add new shards and gradually migrate entities to them without remapping the entire keyspace at once.

The cost is availability and latency. The shard directory is a single point of failure for all database operations. A directory outage means no queries can be routed, even if every shard is healthy. In practice, directory-based systems mitigate this by caching shard mappings aggressively at the application layer (a TTL-based local cache means most requests never hit the directory service at all), replicating the directory across multiple availability zones, and treating directory entries as write-rarely/read-often data. But the architectural complexity is real.

### Consistent Hashing for Sharding

Consistent hashing is a technique described in detail in the [CONSISTENT_HASHING.md](CONSISTENT_HASHING.md) pattern. Its relevance to sharding is the resharding problem: with naive hash-based sharding (`hash(key) mod N`), adding a new shard changes N, which changes the destination for nearly every key. Almost all data must be moved. With consistent hashing, adding a node to the ring only requires migrating the keys that fall between the new node and its predecessor — approximately `1/N` of the total dataset, where N is the number of nodes.

Cassandra and DynamoDB both use consistent hashing as their sharding mechanism precisely because it minimizes resharding cost. The tradeoff is that consistent hashing adds complexity to the routing layer and, without virtual nodes, can produce uneven distributions when the number of physical nodes is small.

---

## The Resharding Problem

Adding a shard to a live production system is one of the most operationally dangerous operations in distributed systems engineering. The data must move; traffic must be routed correctly at all times; and the system must remain available throughout.

The naive approach — take the database offline, dump all data, recalculate shard assignments, import to the new topology — is unacceptable for any system with an SLA. The window required to move terabytes of data is measured in hours or days.

The production approach is a coordinated double-write and backfill migration:

**Step 1: Double-write.** Before moving any data, configure the application to write new records to both the old shard topology and the new topology. This ensures no new data is lost as the migration proceeds. Reads still go to the old topology at this point — the new topology is being populated asynchronously.

**Step 2: Backfill.** Run a background process that reads existing records from the old topology and writes them to the new topology. This process must be throttled to avoid overwhelming either system. It must also be idempotent: if it crashes and restarts, records that were already migrated will be written again, and the new topology must handle duplicates gracefully (upsert semantics rather than insert).

**Step 3: Verify consistency.** Before cutting over reads, compare record counts and checksums between old and new topologies for a representative sample. The new topology should have a superset of records from the old topology (because double-writes started before backfill completed, some records exist only in the new topology).

**Step 4: Cut over reads.** Shift read traffic to the new topology. At this point both topologies are receiving writes and both can serve reads. Monitor for a period — typically 24 to 72 hours — to ensure the new topology behaves correctly under production load.

**Step 5: Stop writing to the old topology.** Once confidence is established, remove the old topology from the write path.

**Step 6: Decommission.** After a safe hold period (during which you could revert if a problem emerges), decommission the old topology.

This process is inherently slow and requires careful coordination. It is one of the strongest arguments for over-sharding at initial design time: if you start with 256 logical shards even when you only have 4 physical nodes (64 logical shards per node), adding a fifth node only requires moving 256/5 ≈ 51 logical shards rather than resharding every row.

---

## Cross-Shard Queries

Once data is distributed across shards, the database engine can no longer transparently handle operations that span multiple shards. Every feature of SQL that involves multiple rows — JOINs, GROUP BY aggregations, window functions, ORDER BY with pagination — must now be implemented at the application layer.

Consider a simple query: `SELECT COUNT(*) FROM orders WHERE status = 'pending'`. On a single Postgres instance, this is a fast index scan. After sharding by `user_id`, this query must be sent to every shard in parallel, each shard returns its local count, and the application sums the results. The fan-out to all N shards is unavoidable because pending orders could exist on any shard.

JOINs are more painful. If `users` is sharded by `user_id` and `orders` is sharded by `user_id` (same key), then "get the user profile along with their last 10 orders" hits a single shard — both tables for a given user live together. But if you need to JOIN `orders` to `products` (sharded by `product_id`), you are executing a cross-shard JOIN: the orders live on user-keyed shards, the products live on product-keyed shards, and no shard holds both. The application must fetch orders from the user shard, extract the `product_id` values, and then query the product shards for those IDs. This is exactly how Shopify implements cross-entity lookups after sharding — denormalize the data you need most often alongside the entity you query from, and accept that some queries require application-level aggregation.

The design implication is significant: when designing a sharded system, you must identify your access patterns before choosing shard keys. The shard key should co-locate entities that are queried together. If the most common query is "all orders for a user," shard both users and orders on `user_id`. If the most common query is "all orders for a product," shard on `product_id`. You cannot optimize for both simultaneously on the same table without duplication.

---

## Hotspot Sharding

Hash-based sharding distributes data evenly in aggregate, but aggregate uniformity does not prevent hotspots on individual rows. If your shard key is `user_id` and a celebrity user has 150 million followers, every read query for that user's timeline hits the same shard. The hash distributes the celebrity to shard 7 (for example), and every one of those 150 million follower feed requests goes to shard 7. The other shards handle their fair share of average users; shard 7 melts under the load of one user's traffic.

Several strategies address this:

**Composite shard keys.** Instead of hashing purely on `user_id`, hash on `(user_id, time_bucket)` or `(user_id, content_type)`. This splits a high-traffic entity's data across multiple shards based on the secondary dimension. Reads for "celebrity's posts from this week" go to a different shard than "celebrity's posts from last month." The application must query both shards when the time range spans buckets, but individual shard load is bounded.

**Adding entropy.** Append a small random integer (0–7) to the shard key: `hash(user_id + random_suffix) mod N`. Writes for a hot entity distribute across 8 virtual shards. Reads must query all 8 and merge. This trades read scatter for write distribution — appropriate when writes are the bottleneck.

**Dedicated hot-entity shards.** Using directory-based routing, detect entities that exceed a traffic threshold (e.g., >10k reads/second sustained) and move them to dedicated, heavily provisioned shards. Twitter's approach for celebrity accounts was effectively this — dedicated infrastructure for accounts above a follower threshold. This works but requires detection logic and operational runbooks for promoting and demoting entities.

**Application-level caching.** For read hotspots specifically, a Redis or Memcached layer in front of the hot shard is often the most pragmatic solution. If the celebrity's profile data is cached with a 30-second TTL, 99%+ of reads never reach the database shard at all. This is not a sharding strategy per se, but it is how most production systems handle read-dominant hotspots in practice.

---

## Trigger Signals

- Single Postgres (or MySQL, MongoDB) instance CPU is consistently >70% during peak
- Storage on the primary has grown beyond 5 TB and vacuum/ANALYZE are taking hours
- Write throughput has exceeded the WAL write capacity of a single machine (~10–15k writes/sec sustained)
- Backup windows extend beyond the available maintenance window (pg_dump on 8 TB takes 6–12 hours)
- Adding read replicas no longer helps because writes are the bottleneck, not reads
- Index sizes have grown to where they no longer fit in RAM and query times are degrading
- Table lock contention during schema migrations is causing multi-minute outages

---

## Problems Where It Applies

**Design Twitter / Instagram Feed.** User data and post data are natural sharding candidates on `user_id`. Feed generation requires querying the shards of all followed users, which is a classic cross-shard fan-out problem — the design answer is typically to pre-compute feeds and write to a feed store, reducing the fan-out to write-time rather than read-time.

**Design WhatsApp / Messenger.** Message data is commonly sharded by `conversation_id` (not `user_id`) so that all messages in a conversation are co-located. This makes "load the last 50 messages in this conversation" a single-shard query. Cross-user queries ("all conversations for this user") require an index table mapping `user_id` → `[conversation_id]` that may itself be sharded separately.

**Design a Payment System.** Transaction data is sharded by `account_id`. The challenge here is distributed transactions: a transfer between two accounts might span two different shards. The design answer is typically to use a saga pattern rather than a distributed ACID transaction — debit account A, enqueue a compensating transaction, credit account B, with idempotency and rollback logic at each step.

**Design YouTube.** Video metadata (title, description, view count, upload status) is sharded by `video_id`. View count updates are particularly interesting: every view is a write to the same row, which concentrates writes on one shard for popular videos. The production approach (used by YouTube) is to buffer view counts in Redis or a similar fast counter store and flush aggregated increments to the database in batches, reducing per-video write pressure by orders of magnitude.

---

## Trade-offs Table

| Strategy | Distribution Evenness | Range Query Support | Resharding Cost | Hot Shard Risk | Implementation Complexity |
|---|---|---|---|---|---|
| Range-based | Poor for append-only keys; good for uniform distributions | Excellent — queries map to one or few shards | High — key ranges must be recalculated | High for monotonic keys (timestamps, auto-increment IDs) | Low — simple arithmetic routing |
| Hash-based | Excellent — hash functions distribute uniformly | None — range queries fan out to all shards | High — changing N invalidates nearly all key mappings | Low for data; possible for traffic (celebrity problem) | Low — simple modulo routing |
| Directory-based | Configurable — can be tuned per entity | Supported if directory encodes range groupings | Low — only need to update directory entries | Low — hot entities can be moved to dedicated shards | High — shard directory is a stateful SPOF requiring replication and caching |
| Consistent Hashing | Good with virtual nodes; uneven without them | None — same scatter problem as hash-based | Very Low — adding a node moves ~1/N of keys | Low to medium | Medium-High — ring management, virtual node configuration |

---

## Real-World Usage

**Instagram (2012).** When Instagram's photo count approached 400 million, they sharded their PostgreSQL cluster on `user_id` using hash-based routing with logical over-partitioning — they created 2,048 logical shards mapped to a smaller number of physical PostgreSQL instances. This over-partitioning decision was deliberate: when they needed to add physical capacity, they moved logical shards rather than remapping all data. Their shard directory was maintained in Zookeeper. They later adopted CitusDB (now Citus, a Postgres extension) to manage the sharded topology with native SQL support. (Instagram Engineering Blog, "Sharding & IDs at Instagram," 2012.)

**Slack (2019).** Slack shards on `workspace_id` (they call it "team ID"). Every workspace's channels, messages, and user memberships live together on one shard, which means the overwhelmingly common query — "load messages in this channel for this workspace" — is always a single-shard operation. The tradeoff is that very large workspaces (enterprise customers with tens of thousands of users and millions of messages) can overwhelm a single shard. Slack addressed this by detecting workspace size thresholds and either provisioning larger hardware for those shards or applying secondary sharding within a workspace. (Slack Engineering Blog, "Scaling Slack," 2019.)

**Notion (2021).** Notion began on a single PostgreSQL instance. As their user base grew after going public in 2020, they encountered the canonical degradation story: vacuums taking too long, index bloat, query planner choosing bad plans on large tables. They migrated to a sharded Postgres architecture using `block_id` (Notion's fundamental data unit) as the shard key. Their migration used the double-write + backfill approach described above, running for several months with dual-write before cutting over. They documented encountering exactly the cross-shard query complexity that theory predicts: several features required redesign because they had assumed co-located data. (Notion Engineering Blog, "Herding elephants: Lessons learned from sharding Postgres at Notion," 2021.)

---

## Common Mistakes

**Sharding too early.** Sharding adds enormous operational and application complexity. Most products never need to shard — a well-tuned single Postgres instance with read replicas can handle the scale of the vast majority of applications. Engineers who have read about Instagram and Notion's sharding migrations often propose sharding as a "scalable design" before there is any evidence the system will hit single-node limits. The correct interview answer is: "I would start with a single primary and read replicas; here are the specific signals that would trigger a sharding decision." Premature sharding is a red flag to experienced interviewers.

**Choosing a shard key that creates a hot shard immediately.** Sharding by `created_at` when the workload is primarily append-only (new records are always the most-accessed records) means 100% of writes and most reads land on the last shard. This defeats the entire purpose of sharding. The shard key must match the access pattern: use a key whose value distribution matches the distribution of queries.

**Forgetting to shard indexes alongside data.** When you shard a table, every index on that table is also split. The foreign key relationships, unique constraints, and secondary indexes that PostgreSQL enforced automatically on a single instance must now be enforced at the application layer. A unique constraint on `email` across all users cannot be enforced by a single shard — the application must query all shards to check uniqueness before inserting. Engineers frequently design shard schemas without thinking through which constraints can no longer be enforced natively.

**Ignoring the cross-shard transaction problem.** An operation that must atomically update two rows on different shards cannot be wrapped in a single SQL transaction. Engineers propose `BEGIN; UPDATE shard_1...; UPDATE shard_2...; COMMIT;` without acknowledging that this is impossible — the transaction coordinator would need to span two independent database instances, which requires a distributed transaction protocol (2PC). The correct answer involves sagas, outbox patterns, or accepting eventual consistency for cross-shard writes. Not acknowledging this problem in an interview is a major red flag.

**Not planning for resharding before going live.** Systems that launch with a fixed shard count and no resharding plan eventually face the worst possible resharding scenario: an overloaded production system that must be resharded urgently with no tooling and no tested procedure. The correct approach is to over-partition at launch (use 256 or 1024 logical shards rather than N physical nodes), write and test the resharding tooling in a staging environment before you ever need it, and maintain runbooks for the migration procedure.

**Selecting user-facing IDs that leak shard assignment.** If your shard key is embedded in the public-facing entity ID (a common pattern for routing efficiency), you reveal your shard topology to external observers. This has security implications (an attacker can target specific shards by crafting IDs) and makes resharding harder (moving a shard changes the ID → shard mapping that external clients have cached). Instagram's solution was to use opaque IDs where the shard assignment is encoded in bits that are not exposed directly in the API.
