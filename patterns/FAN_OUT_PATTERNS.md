# Fan-out Patterns

## Intent

Distribute a single write event to multiple downstream consumers, trading write amplification for read speed (or vice versa), depending on which resource is the bottleneck.

---

## The Problem

Every social feed is fundamentally an aggregation problem: when you open Instagram, you need to see posts from the 400 people you follow, ranked by recency or relevance, returned in under 200 milliseconds. The naive implementation reaches for a database query: `SELECT posts.* FROM posts JOIN follows ON posts.user_id = follows.followee_id WHERE follows.follower_id = ? ORDER BY posts.created_at DESC LIMIT 50`. On a small dataset, this works fine. At scale, it is catastrophic.

Consider the numbers. Instagram in 2022 served over 500 million daily active users, with the average user following around 300 accounts. A single feed load must aggregate posts across 300 accounts, sort them, and return a page — all under strict latency SLAs. If each account has posted, say, 100 posts in the last week, the database must scan, join, and sort up to 30,000 rows per feed request. With 500 million users opening their feed even once a day, this is roughly 15 trillion row scans per day before any other database traffic. No relational database, not even a well-sharded one, survives this read amplification intact.

The root tension is a fundamental read-vs-write amplification tradeoff. Every system designer must make an explicit choice: pay the cost at write time (fan-out on write, also called the push model) or pay it at read time (fan-out on read, the pull model). Neither is universally correct. The wrong choice, or the failure to implement a hybrid, is one of the most common design mistakes in feed system interviews.

---

## Fan-out on Write (Push Model)

When a user posts, the system immediately writes that post reference into every follower's precomputed feed. Structurally, this is often implemented as a sorted set in Redis keyed by user ID — for example, `FEED:{user_id}` — where each entry is a post ID ranked by the post's creation timestamp as the score. When user A (with 800 followers) posts, the application or a background fan-out worker executes 800 `ZADD FEED:{follower_id} {timestamp} {post_id}` operations, one per follower. The feed for any user is then a single `ZREVRANGE FEED:{user_id} 0 49` call — O(log N + K) where K is the page size — typically sub-millisecond from a well-provisioned Redis cluster.

This model works exceptionally well when reads dramatically outnumber writes and when most users have a manageable follower count. The average Twitter user as of 2013 had around 200 followers, and users open their feed far more often than they post. Absorbing the write amplification at post time — once — pays off across hundreds or thousands of subsequent feed reads. Facebook's original news feed, Pinterest's home feed, and early Twitter all used variants of this approach.

The model breaks the moment you introduce high-follower accounts, commonly called the "celebrity problem." When Katy Perry posts a tweet, her follower count in 2013 was approximately 50 million. A single post triggers 50 million write operations to fan out to every follower's feed. Twitter's engineering team documented in 2013 that a single viral tweet from a celebrity account could take up to 5 minutes to fully propagate through their fan-out system, because the write throughput required to enqueue 50 million sorted-set operations exceeded what their infrastructure could absorb in real time. The system queued the writes and processed them over minutes, during which followers with precomputed feeds simply did not see the post. This is not acceptable for a real-time social network.

---

## Fan-out on Read (Pull Model)

When a user opens their feed, the system queries the recent posts of all accounts they follow in real time and merges them. Concretely: fetch the latest 50 posts from each followed account, merge the result sets by timestamp, and return the top 50. This eliminates the write-time amplification entirely — posting is a single write to the posts table, with no fan-out work at all.

The cost is paid entirely at read time. For a user following 300 accounts, generating their feed requires 300 parallel (or batched) database lookups, followed by an in-memory merge sort of potentially thousands of rows. Latency is proportional to following count and database query latency. For casual users, this may be acceptable. For power users following thousands of accounts — a pattern common on Twitter, LinkedIn, and in financial news aggregators — feed latency can balloon to seconds, far beyond any reasonable SLA.

There is one class of account for which this model is ideal: accounts followed by millions of people. The account with 50 million followers benefits enormously from pull-at-read-time because their posts are read lazily — only when their followers actually open their feeds — rather than written eagerly to 50 million sorted sets the moment they post. The system's write path for high-follower accounts is a single row insert, just like any other post. The read-time cost is paid by each reader's fan-out query, but that cost is diffused across millions of independent reads rather than concentrated in a single fan-out event.

The pull model breaks when followed counts grow large. A user following 1,000 accounts — not unusual for an engaged Twitter or LinkedIn user — may need 1,000 parallel database queries per feed load. Even with connection pooling and caching, this is O(N) query pressure on the read path, and it grows linearly with following count. At some following-count threshold, the read latency becomes unacceptable.

---

## Hybrid Approach

The practical solution that both Twitter and Instagram converged on is a hybrid: precompute feeds (fan-out on write) for regular users while pulling celebrity content at read time and merging it in.

The system classifies accounts into two buckets based on a follower-count threshold. Regular accounts — those with fewer than some threshold, reported as approximately 1 million followers in Instagram's 2020 engineering documentation — participate in fan-out on write. When they post, their post ID is pushed into the precomputed feed sorted sets of all their followers. Celebrity accounts — those above the threshold — are excluded from fan-out entirely. Their posts are stored only in the canonical posts store, never pushed to individual follower feeds.

At read time, the system performs a two-step retrieval. First, it fetches the user's precomputed feed from Redis, which contains post IDs from all regular accounts they follow. Second, it identifies which celebrity accounts the user follows, queries the canonical posts store for each celebrity's recent posts, and merges both result sets by timestamp in application memory before returning the final ranked page. The merge is a standard k-way sorted merge, O(K log C) where K is the total number of candidate posts and C is the number of celebrity accounts followed. Because the precomputed feed is already sorted and the celebrity posts are fetched as small, already-sorted windows, the merge is fast enough to be done synchronously on the read path without meaningfully adding to latency.

Instagram documented this architectural shift around 2020, noting that it eliminated the write-amplification spikes they had been observing when high-follower accounts posted. Twitter made a similar architectural decision around 2013–2014, described in their engineering blog, after the sustained operational pain of fan-out storms from celebrity accounts during live events (the Super Bowl, award shows) caused write throughput to saturate their Redis cluster.

The threshold for "celebrity" is a tunable parameter, not a fixed number. The right threshold depends on the write throughput your fan-out infrastructure can absorb, the read latency budget you have for the merge step, and the distribution of follower counts in your user base. A system with faster storage (NVMe-backed Redis clusters) can afford a higher threshold before switching to pull. A system with tight read latency SLAs needs a lower threshold to keep the merge step cheap.

---

## Notification Fan-out vs Feed Fan-out

Feed fan-out and notification fan-out look similar on the surface — both distribute one event to many recipients — but they have fundamentally different constraints that drive different architectural choices.

Feed fan-out is tolerant of slight staleness. If a post takes 200 milliseconds to appear in a follower's feed, no one notices or cares. Feeds are typically "best effort recent" — the user will refresh and the post will appear. This tolerance for eventual consistency means feed fan-out can use asynchronous background workers, absorb brief queuing delays, and drop non-critical updates (such as posts from inactive accounts whose followers have not opened the app in weeks) without user impact.

Notification fan-out has the opposite constraint: near-real-time delivery with delivery guarantees. If a user receives a push notification about a message, and the message does not appear when they tap the notification, the experience is broken. Notifications are delivered via APNs (Apple) and FCM (Google), both of which are push protocols with their own delivery semantics — at-least-once delivery, with acknowledgement required. This means the notification fan-out pipeline must use a durable message queue (Kafka at large scale, SQS at AWS-native shops) that retains messages until acknowledged, supports retry on failure, and can be consumed by a worker fleet that calls APNs/FCM HTTP APIs per recipient.

Feed fan-out at companies like Twitter and Instagram typically writes directly to Redis sorted sets, either from the application process or a dedicated fan-out service, without an intermediate durable queue. Speed matters more than strict durability — a missed feed entry is a minor inconvenience, not a broken feature. Notification fan-out, by contrast, uses Kafka or a similar durable log because losing a notification is a visible user-facing failure. The operational complexity of maintaining delivery acknowledgements, retry queues, and dead-letter queues is justified by the delivery guarantee requirement that feed fan-out does not share.

---

## Trigger Signals

- User is designing a social feed, news feed, or content recommendation feed
- Question involves "how does a post reach all followers?" or "how do you scale a feed?"
- Design involves a system where one entity (user, device, event) triggers actions for many other entities
- System has highly skewed distribution in write or read fan-out (celebrities, hot partitions, viral events)
- Feed latency SLAs are specified (< 200ms read is a signal toward write fan-out)
- Write throughput during peak events (Super Bowl, breaking news) is a concern

---

## Problems Where It Applies

**Twitter/X Feed:** The canonical fan-out problem. Fan-out on write for regular users, pull for celebrities (Katy Perry, verified accounts above threshold), merged at read time. Twitter's 2013 engineering posts are the primary public reference for the hybrid architecture.

**TikTok/Instagram Reels:** Video feeds have the same fan-out structure but add a ranking model on top of recency. The fan-out delivers candidate post IDs into a precomputed feed; a separate ranking pass (often a lightweight ML model) reorders them before delivery. Fan-out is still the mechanism; ranking is layered on top.

**Notification System:** Fan-out from event (like, comment, message, system alert) to recipient device via APNs/FCM. Uses Kafka as the durable fan-out queue rather than Redis sorted sets because delivery guarantees are required.

**WhatsApp/Messenger Group Messages:** A message to a group of N members must be delivered to N devices. For small groups (< 256 members), this is direct fan-out per recipient. For larger groups or broadcast channels (WhatsApp announced channels with up to millions of followers in 2023), this becomes a classical celebrity-style fan-out problem requiring the same write-vs-read tradeoff.

---

## Trade-offs Table

| Dimension | Fan-out on Write | Fan-out on Read | Hybrid |
|---|---|---|---|
| **Write Amplification** | High — one post triggers N writes (N = follower count) | None — one post = one write | Moderate — N writes only for non-celebrity accounts |
| **Read Amplification** | None — feed is precomputed | High — O(following count) queries per feed load | Low — precomputed feed + small celebrity merge |
| **Feed Read Latency** | Very low — single Redis lookup | High and variable — grows with following count | Low — bounded by merge step cost |
| **Celebrity Problem** | Severe — 50M followers = 50M writes per post | None — celebrities pull cheaply | Solved — celebrities excluded from write fan-out |
| **Storage Cost** | High — every post stored once per follower | Low — posts stored once | Moderate — most posts stored per follower; celebrity posts stored once |
| **System Complexity** | Low — simple sorted set writes | Low — simple query per followed user | High — requires follower-count classification, merge logic, and dual retrieval paths |

---

## Real-World Usage

**Twitter (hybrid since ~2013–2014):** Twitter's engineering blog published in 2013 describes the write fan-out architecture and the celebrity problem in detail. The solution they implemented maintained fan-out on write for regular users while switching to a pull model for accounts above a follower-count threshold. At the time, this was described as a significant architectural undertaking — retrofitting the pull path into what had been a pure push system required changes to the feed generation service, the Redis data model, and the API layer that merged both result sets.

**Instagram (hybrid documented ~2020):** Instagram's engineering team documented their hybrid fan-out architecture in 2020, describing the account classification system that determines whether a new post triggers a fan-out write or is left for pull-at-read-time. Instagram's threshold is not publicly disclosed, but the system design principles they described match the hybrid model: precomputed feeds for the majority of accounts, real-time merge for high-follower accounts.

**Facebook:** Facebook's news feed has used a variant of fan-out on write for most of its history, combined with a ranking model that determines which of the candidate posts in a user's precomputed feed surface to the top. Facebook's 2009 Cassandra whitepaper describes an early version of their precomputed inbox model. Their scale (3+ billion monthly active users as of 2023) means even the hybrid model requires careful management of fan-out write throughput, particularly during live events.

---

## Common Mistakes

**Applying pure fan-out on write without acknowledging the celebrity problem.** Candidates describe a clean Redis sorted-set write for every follower and never mention what happens when a user with 10 million followers posts. A good interviewer will immediately ask "what happens when Cristiano Ronaldo posts?" The correct answer requires either a follower-count threshold that switches to pull, or a background-queue fan-out that can absorb the write volume over several minutes (with the tradeoff that celebrity posts are stale in follower feeds).

**Confusing fan-out on write with synchronous writes in the request path.** Writing to 800 Redis sorted sets is not fast enough to do synchronously before returning the HTTP response to the posting user. The post write should succeed immediately (write to the canonical posts store), and fan-out should happen asynchronously via a background worker or queue. Candidates who put fan-out in the synchronous write path will fail under any load spike and will cause unacceptable p99 latency on the posting API.

**Conflating notification fan-out with feed fan-out.** Both distribute one event to many recipients, but notification fan-out requires durable queuing and delivery acknowledgement while feed fan-out can use fire-and-forget writes to Redis. Designing notification fan-out as direct Redis writes loses messages on worker crashes. Designing feed fan-out through a fully durable Kafka pipeline adds unnecessary operational complexity and latency for a use case where eventual consistency is acceptable.

**Neglecting the merge step in hybrid designs.** Candidates who describe the hybrid approach often hand-wave the read-time merge: "we just combine the two feeds." The merge has real cost — it requires knowing which celebrity accounts the requesting user follows (a database or cache lookup), fetching recent posts for each, and performing an in-memory sort. For a user following 50 celebrity accounts, this is 50 cache lookups plus a merge sort. The design must account for the latency budget of this merge step and consider caching the celebrity post lists with a short TTL rather than querying them cold on every feed load.
