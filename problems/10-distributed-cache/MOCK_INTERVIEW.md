# Mock Interview Script — Distributed Cache
*For the interviewer. Do not share with candidate.*

## Pre-Session Setup
- Problem: Design a distributed in-memory cache (Redis/Memcached-like)
- Target level: Google L5 / Amazon Senior / Meta E5 — Advanced problem
- Time: 45 minutes
- Key traps: consistent hashing justification, O(1) LRU, thundering herd

## Opening (read aloud, verbatim)
"Design a distributed in-memory cache system — like Redis or Memcached — that application servers use to reduce database load. The cache should support GET, SET, and DELETE operations on key-value pairs, with TTL support and LRU eviction. You need to design the cache system itself, not just describe how to use an existing cache. You have 45 minutes. Start with clarifying questions."

*Start timer.*

## Phase 1: Requirements (0–8 min)

**Answers if asked:**
| Question | Answer |
|----------|--------|
| Total data size? | 64 TB |
| Node size? | 32 GB per node |
| Read/write ratio? | 100:1 |
| Peak QPS? | 10M reads/sec, 100K writes/sec |
| Strong consistency? | Eventual is fine; brief staleness acceptable |
| Atomic operations? | Yes — INCR, SETNX needed |
| Data structure types? | String key-value only (no lists, sets) |
| Replication? | Yes — at least one replica per key |
| Node failure tolerance? | One node failure should not lose data |

**Red flag:** Candidate jumps to "I'll just use Redis" without designing the system.

**Green flag:** Candidate asks about total data size and node size to derive node count.

## Phase 2: Estimation (8–13 min)

**Expected:**
- Nodes: 64 TB / 32 GB = 2,048 nodes (primary), 4,096 with replication
- Read throughput: 10M/sec across 2,048 nodes ≈ 5K reads/node/sec (trivial)
- Write: 100K/sec across 2,048 nodes ≈ 50 writes/node/sec (trivial)
- Bottleneck: NETWORK bandwidth (10M reads × 1 KB = 10 GB/sec from cluster)

**Key question:** "What is the performance bottleneck in this system?" Expected: network, not compute.

## Phase 3: High-Level Design (13–28 min)

**Expected components:**
- Consistent hash ring (with virtual nodes)
- Per-node LRU eviction with O(1) data structure
- Replication (next N nodes on ring)
- TTL expiry (lazy + periodic active)
- Thundering herd mitigation

**Critical question at 18 min:** "You have 2,048 cache nodes. I add one more. How many keys need to move? Compare this to if you used key % 2048."

*Evaluate: can they compute 1/2049 ≈ 0.05% for consistent hashing vs 2047/2048 ≈ 99.95% for modulo?*

**If they say consistent hashing without explaining:** "Why not just use key modulo number-of-nodes? What exactly goes wrong?"

**Critical question at 23 min:** "A popular key expires. 50,000 concurrent users request this key at the same moment. Walk me through what happens in your system."

*Evaluate: do they identify the thundering herd? Do they propose a mitigation?*

**If they propose LRU without explaining implementation:** "How do you implement LRU such that GET and SET are both O(1)? What data structure?"

## Phase 4: Deep Dive (28–40 min)

**Option A — Consistent Hashing Deep Dive:**
"Walk me through consistent hashing in detail. How is the ring structured? How does a key get assigned to a node? What happens when a node is added — step by step, which keys move and where do they move to?"

**Option B — LRU Implementation:**
"Implement LRU eviction. I want O(1) GET, O(1) SET, and O(1) eviction. Walk me through the data structure and the operations."

*This is LeetCode #146 — if they haven't practiced it, they'll struggle.*

**Option C — Replication:**
"Walk me through your replication strategy. Key X is on Node 15 (primary). I add Node 16 adjacent to Node 15 on the ring. What happens to replicas? If Node 15 fails, how does the system detect the failure and how do reads and writes proceed?"

**Option D — Thundering Herd:**
"Design a thundering herd protection mechanism. Walk through the implementation of SETNX-based single-flight locking. What are the failure cases? What happens if the lock holder crashes before populating the cache?"

## Phase 5: Failure & Scaling Probe (40–44 min)

Pick TWO:
1. "Node 500 goes down. Walk through exactly what happens to reads and writes for keys that were on Node 500. Do users see errors? Degraded performance?"
2. "Your cache cluster is at 85% memory utilization. Eviction is happening constantly. Cache hit rate drops to 60%. The database is overwhelmed. Walk me through your response."
3. "One key is being accessed 10 million times per second — it's a configuration value that every request reads. Your cache node serving that key is at 100% CPU. Consistent hashing puts all those reads on one node. How do you fix this?"
4. "An app server has a stale copy of the consistent hash ring — it thinks Node 500 is still alive, but it died 30 seconds ago. The request goes to Node 500. What happens? How does the client recover?"

## Closing (44–45 min)
"Final question: what is the hardest operational problem in running a distributed cache at this scale?"

## Scoring Checklist

| Dimension | Evidence | Score (1-5) |
|-----------|----------|-------------|
| Requirements | Asked about data size, read/write ratio, consistency requirements | |
| Estimation | Derived node count; identified network as bottleneck | |
| Consistent hashing | Explained WHY modulo fails; quantified key migration cost | |
| Virtual nodes | Explained uneven load problem; v-nodes as solution | |
| LRU implementation | Named doubly linked list + hashmap; O(1) for all operations | |
| Thundering herd | Identified proactively; proposed mitigation mechanism | |
| Failure handling | Node failure impact; hot key problem | |
