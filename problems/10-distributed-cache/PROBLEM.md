# Distributed Cache (Redis-like System)
**Difficulty:** Advanced
**Category:** Infrastructure / Distributed Systems
**Time Box:** 45 min
**Key Concepts:** Consistent Hashing, Eviction Policies, Replication, Cache Invalidation, Thundering Herd
**Asked at:** Amazon, Google, Microsoft, Twitter, Facebook/Meta, Netflix, Uber

## Problem Statement

You are building a distributed in-memory cache system — think Redis or Memcached. The system stores key-value pairs in memory for fast retrieval, acting as a high-performance layer between application servers and slower persistent databases. Application servers write to the cache and read from it; on a cache miss, they fall back to the database and populate the cache for future reads.

This is not a question about "how do you use Redis." It is a question about how you would build the distributed caching infrastructure itself. You must design the cache nodes, the data partitioning across nodes, the replication strategy, the eviction policy, and the client-side logic for finding the right node. The system must handle node additions and removals without cascading cache misses and must prevent the database from being overwhelmed when the cache is cold.

## Actors
| Actor | Description |
|-------|-------------|
| Application Server | Client of the cache; issues GET/SET/DELETE operations |
| Cache Node | Stores key-value pairs in memory; handles eviction when full |
| Cache Client (library) | SDK embedded in application server; routes keys to correct nodes |
| Admin | Monitors cache hit rate, memory usage; adds/removes nodes |

## Functional Requirements
**Core (MVP):**
- FR1: GET(key) → value or null (cache miss)
- FR2: SET(key, value, ttl) → store key with expiration
- FR3: DELETE(key) → remove key
- FR4: Horizontal scaling: add and remove cache nodes without manual resharding
- FR5: TTL support: keys expire automatically after a set time

**Secondary:**
- FR6: Eviction policy: when cache is full, evict least-recently-used (LRU) keys
- FR7: Replication: each key has at least one replica on a different node for fault tolerance
- FR8: Atomic operations: INCR, DECR, SETNX (set if not exists) for counters and distributed locks

## Non-Functional Requirements
| Requirement | Target | Rationale |
|-------------|--------|-----------|
| Read/write latency | < 1ms p99 (in-memory operations) | Cache's value proposition is speed |
| Availability | 99.99% | Cache failure passes load to database; cascading failure risk |
| Scalability | Linear: 2× nodes = 2× capacity and throughput | No resharding cost on node addition |
| Node failure isolation | Single node failure affects only its key range | No global resharding on failure |
| Memory efficiency | > 80% of allocated memory holds data (low overhead) | Memory is expensive; minimize metadata |
| Eviction correctness | LRU eviction removes the truly least recently used key | Approximate LRU acceptable (Redis approach) |

## Capacity Estimation Hints
- Total cache size: 64 TB (total key-value data to cache across the cluster)
- Typical node size: 32 GB RAM per node → 2,048 nodes needed
- Average key-value size: 1 KB
- Read/write ratio: 100:1 (heavily read-optimized)
- Read QPS: 10 million reads/sec
- Write QPS: 100,000 writes/sec
- Peak multiplier: 5×

## Good Clarifying Questions to Ask

**Scale:**
- How much total data to cache? How large is each cached item?
- What is the read/write ratio?
- What is the expected QPS at peak?

**Fault tolerance:**
- What happens when a cache node goes down? Can we lose data?
- How many node failures must we tolerate simultaneously?

**Eviction:**
- What eviction policy? LRU is default; is there a reason for LFU or TTL-only?
- What is the target cache hit rate?

**Consistency:**
- Is strong consistency needed, or is eventual consistency acceptable?
- Can two application servers read different values for the same key briefly?

**Operations:**
- Do we need atomic operations (INCR, SETNX)?
- Do we need pub/sub or data structure types (lists, sets), or just string key-value?

## Key Components
<details><summary>Reveal components</summary>

- **Cache Node**: In-memory hash table; LRU eviction via doubly linked list + hashmap; TTL expiry via min-heap or lazy deletion
- **Consistent Hash Ring**: Client-side data structure mapping keys to nodes; virtual nodes for load balance
- **Cache Client Library**: Embedded in app server; maintains ring state; handles GET/SET routing; connection pooling
- **Replication**: Each write goes to primary node + one/two replicas (nodes next clockwise on ring)
- **Gossip Protocol**: Nodes detect each other's health; propagate membership changes without central coordinator
- **Coordinator Service (optional)**: ZooKeeper/etcd alternative to gossip for cluster membership
- **Eviction Engine**: Background thread monitoring memory usage; triggers LRU eviction when above threshold
</details>
