# Consistent Hashing

## Intent

Map keys to nodes such that adding or removing a node remaps only `1/n` of keys rather than nearly all of them.

---

## The Problem It Solves

Naive sharding assigns a key to a node using `shard = hash(key) % n`. This works well when `n` is fixed. The catastrophe arrives when you need to scale. Suppose you have 100 keys distributed across 4 nodes with `mod-4` hashing. Each key's shard is determined by which of `{0, 1, 2, 3}` its hash falls into. Add a fifth node and the formula becomes `mod-5`, mapping each key into `{0, 1, 2, 3, 4}`. The only keys that survive unmoved are those whose hash value is evenly divisible by both 4 and 5 — a multiple of 20 — which statistically represents roughly 1-in-5 of your keys. The other ~80% of keys now point to a different node. That is not a partial remapping. It is a near-total redistribution.

The production consequences of ~80% key remapping are severe. In a distributed cache this means a cache invalidation storm: every remapped key is now a cache miss, and all those misses fall directly on your database simultaneously. A cache that was absorbing, say, 90% of read traffic suddenly forwards the bulk of requests to the database in a matter of seconds. Instagram's Memcached layer, which serves hundreds of millions of users, has documented this class of problem — cache warm-up after a topology change accounts for measurable latency spikes. In a partitioned database layer the consequences are different but equally bad: hot partitions form as write traffic concentrates on the nodes that received the remapped keys, and connection pools on those nodes saturate. Services that appeared healthy under steady state suddenly spike in p99 latency past their SLA.

The underlying problem is that `mod-n` hashing creates a global dependency: every node's identity is entangled with every other node's. Remove node 2 from a 5-node ring and you do not just redistribute node 2's keys — you recompute every key's assignment because the denominator changed. Consistent hashing breaks this global dependency. Each key's assignment depends on only one small arc of a fixed address space, so adding or removing a node only disturbs the keys that were assigned to that arc.

---

## How It Works

Consistent hashing places both nodes and keys on a conceptual ring of hash values, typically using a 32-bit or 64-bit hash space (0 to 2³² − 1 wrapped end-to-end). Each node is hashed to a position on the ring. Each key is also hashed to a position, and the key is owned by the first node encountered when traversing the ring clockwise from the key's position.

```
          0°
          │
    N3 ───┤──── N0
         /        \
        /          \
  K5   /            \  K1
      │              │
   N2 │              │ N1
      │              │
  K4   \            /  K2, K3
        \          /
         \        /
          180°
```

In this diagram, key `K1` falls between `N0` and `N1` clockwise, so `N1` owns it. Key `K4` falls between `N2` and `N3`, so `N3` owns it. When `N0` is removed, only the keys between `N3` and `N0` (the arc that `N0` previously owned) need to be reassigned — to `N1`. All other keys stay where they are. With `n` nodes, adding or removing one node remaps `1/n` of keys on average.

Bare consistent hashing with one hash position per node has a critical flaw: uneven load distribution. If the nodes happen to hash to positions that are clustered together on one side of the ring, one node owns a large arc (and thus many more keys) while another owns a tiny arc. The solution is **virtual nodes** (vnodes): each physical node is hashed to multiple positions on the ring under different names (e.g., `node_A_1`, `node_A_2`, ... `node_A_150`). All positions that land on a virtual node belonging to physical node A route to node A. With enough vnodes, each physical node owns a statistically even share of the ring regardless of where the hash function places the positions. Amazon DynamoDB uses 150–200 vnodes per physical node (as of approximately 2022) to achieve this statistical evenness. Apache Cassandra defaults to 256 vnodes per node.

The token-based variant, used in Cassandra's older configuration mode, pre-assigns explicit token ranges to each node rather than relying on hash randomness for placement. This gives operators precise control over key distribution at the cost of manual management when nodes join or leave.

---

## Trigger Signals

- Interviewer asks you to design a distributed cache (Memcached, Redis cluster).
- The system needs horizontal scaling of a storage or compute tier.
- The problem involves routing requests to one of many stateful backend nodes.
- Interviewer mentions a cluster that grows and shrinks (auto-scaling, spot instances).
- You need to minimize cache invalidation impact during scaling events.
- The design involves sharding user data across database partitions.

---

## Problems Where It Applies

**Distributed Cache (e.g., Design Memcached/Redis Cluster):** Client-side consistent hashing determines which cache node owns a given key. When a cache node is added during a traffic spike, only `1/n` of keys remap — the remaining keys continue serving from their existing nodes, avoiding a total cold-start storm.

**Design Uber / Lyft (Driver Location Sharding):** Driver location updates arrive at high frequency. Rather than broadcasting every update to all nodes, consistent hashing routes each driver's updates to a deterministic shard server. When the location service scales out during peak hours (Friday night surge), only the drivers whose shard falls on the new arc need to be redistributed.

**Design Twitter Feed (User Data Sharding):** User records and tweet timelines are partitioned by user ID. Consistent hashing ensures that a given user's data consistently lands on the same shard, enabling co-location of a user's tweets, followers, and timeline. Adding a new database shard migrates only the slice of the ring that the new shard takes over, rather than reshuffling the entire user corpus.

---

## Trade-offs

**What consistent hashing does not solve: skewed key distribution (hot partitions).** If a small number of keys receive disproportionate traffic — think a celebrity's user ID in a social graph, or a viral URL in a shortener — those keys will all hash to the same arc regardless of how the ring is partitioned. Consistent hashing distributes keys evenly in expectation, but not traffic. Hot partitions require application-level mitigations: key splitting, read replicas on hot shards, or dedicated overflow handling.

**When simpler solutions are fine.** If your cluster size is fixed and topology changes are rare (e.g., a quarterly node upgrade), `mod-n` hashing with a coordinated migration window may be operationally simpler. A 15-minute maintenance window to rehash is often less complex to reason about than a consistent hashing layer with vnode management. Read replicas can absorb read scaling without any resharding at all.

**Vnode overhead.** Each physical node with 150–200 vnodes means the routing table contains 150–200× more entries. In a large cluster (hundreds of physical nodes), the routing metadata can reach tens of thousands of entries. This is manageable in practice, but it is non-trivial to serialize, distribute, and keep consistent across clients.

**Replication complexity.** Consistent hashing defines which node is the primary owner of a key, but replication requires coordinating with the next `N` nodes clockwise on the ring (DynamoDB's N=3 convention). When a node joins, data must be streamed from its neighbors. When a node leaves, its data must be redistributed before it goes offline. Both operations require careful sequencing to avoid data loss, and the implementation complexity is non-trivial.

---

## Real-World Usage

**Amazon DynamoDB** uses consistent hashing as the foundation of its partitioning scheme. Each partition (the unit of storage and throughput) maps to a contiguous token range on the ring. DynamoDB's 2022 re:Invent presentations describe how the system uses approximately 150–200 vnodes per physical host to ensure balanced distribution. When a table auto-scales (splits a hot partition), the system assigns a new token range to the new partition without affecting unrelated ranges.

**Apache Cassandra** has used consistent hashing since its initial open-source release in 2008, inherited from Amazon's Dynamo paper (2007). Its virtual node model (introduced in Cassandra 1.2, circa 2012) defaults to 256 vnodes per node. When a node is decommissioned, Cassandra streams only the token ranges that the departing node owned to its neighbors — not a full cluster reshuffle.

**Discord** uses consistent hashing for routing in their sharded message storage architecture. In Discord's 2022 engineering blog posts describing their migration to a Rust-based storage engine (Scylla), they discuss how their message data is sharded by channel ID using consistent hashing. This ensures that all messages for a given channel land on the same set of storage nodes, which is critical for their read patterns (most reads are recent messages in a single channel). Discord reported in 2022 that their infrastructure serves roughly 4 billion messages per day, and consistent partitioning by channel is what makes that read pattern efficient.

---

## Common Mistakes

**1. Describing the ring as storing data.** The ring is a routing abstraction — a mathematical construct for key-to-node mapping. The data lives on the nodes themselves. Candidates sometimes describe "storing the key on the ring" as if the ring is a physical storage layer. The ring is metadata; the nodes hold the actual bytes.

**2. Forgetting replication entirely.** Consistent hashing with no replication means a node failure loses all keys in its arc. Interviewers expect you to specify your replication factor (N=3 is typical), how you select replicas (next N nodes clockwise), and what your read/write quorum requirements are (R + W > N for strong consistency). Omitting this makes the design appear single-point-of-failure.

**3. Claiming consistent hashing solves the hot key problem.** It does not. If the interviewer introduces a celebrity user or a trending URL, consistent hashing ensures that key routes to one place every time — which is exactly the problem, not the solution. The correct answer involves application-level strategies (key splitting with a random suffix, local in-process caching, read replicas) layered on top of consistent hashing.

**4. Confusing vnodes with the replication factor.** These are separate concerns. Vnodes are multiple hash ring positions per physical node, used for load balance. Replication factor determines how many distinct physical nodes hold copies of each key. A node can have 200 vnodes and a replication factor of 3; those are orthogonal parameters.

**5. Using consistent hashing when a simpler solution suffices.** If the interviewer's problem has a fixed, small number of partitions (say, 8 database shards with no expected growth), proposing consistent hashing introduces unnecessary complexity. Recognizing when the problem actually calls for the pattern — versus when static range partitioning or a directory-based approach is simpler — signals engineering judgment. Always state the tradeoff explicitly.
