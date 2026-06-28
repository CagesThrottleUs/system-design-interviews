# Lessons Learned — Distributed Cache

## Real Interview Stories

**Story 1: Amazon Senior SDE — The Modulo Hash Cascade**
Company: Amazon | Level: Senior SDE | Round: System Design

What happened: Candidate proposed distributing cache keys using consistent hashing — correctly. But when the interviewer asked "why not just use key % number_of_nodes?", candidate said: "It's less efficient." Interviewer pressed: "What specifically goes wrong?"

Where it went wrong: Candidate couldn't quantify the failure. They knew consistent hashing was "better" but not why. The critical answer is: with modulo hashing, adding or removing one node causes ~(N-1)/N ≈ all keys to map to a different node. For a 100-node cluster, a single node addition reshards 99% of all keys simultaneously — every one of those keys is now a cache miss that hits the database. This was a documented failure mode in Amazon's Dynamo and Facebook's Memcached deployments.

Consequence: Interviewer concluded the candidate had studied the pattern without understanding the mechanism. "Can describe the solution but not diagnose the problem" — a significant gap at senior level.

What to do instead: Practice the exact quantification. "With N nodes and modulo hashing, adding 1 node reshards (N-1)/N ≈ 99% of keys. With consistent hashing, adding 1 node reshards 1/(N+1) ≈ 0.05% of keys — the ring arc of the new node. Every other key is untouched."

---

**Story 2: Google L6 — The O(N) LRU Mistake**
Company: Google | Level: L6 (Staff Engineer) | Round: System Design

What happened: Candidate correctly proposed LRU eviction. Interviewer asked: "How do you implement LRU? What data structure?" Candidate: "I keep a timestamp on each entry. When I need to evict, I find the entry with the oldest timestamp."

Where it went wrong: Finding the entry with the oldest timestamp requires scanning all N entries — O(N). For a node with 30 million keys, this is catastrophic. Eviction would take seconds. The cache would stall.

Consequence: Interviewer immediately asked "what's the time complexity of that?" Candidate computed it and froze. Spent 8 minutes trying to improve the design. Never proposed the O(1) doubly linked list + hashmap solution.

What to do instead: Know the O(1) LRU implementation cold. It is a fundamental data structures interview question that appears specifically in system design as an implementation detail of eviction. Doubly linked list (head = MRU, tail = LRU) + hashmap (key → list node). GET: O(1) move to head. SET: O(1) insert at head. Eviction: O(1) remove tail.

---

**Story 3: Netflix Senior SWE — Thundering Herd Surprise**
Company: Netflix | Level: Senior SWE | Round: System Design (CDN/cache design)

What happened: Candidate designed a clean distributed cache with consistent hashing, LRU eviction, and replication. Interviewer set the scenario: "You've cached the metadata for a hugely popular new movie release — every user who opens Netflix in the next hour will request this key. The TTL expires after 5 minutes. At the exact moment the TTL expires, 50,000 users are simultaneously loading the app. What happens?"

Where it went wrong: Candidate said "they'll all miss, hit the database, and the key gets repopulated." Did not identify that 50,000 simultaneous identical database queries is a problem — a thundering herd that could saturate and crash the database.

Consequence: Interviewer said "the database gets 50,000 identical queries in 100ms. It was designed for 200 queries/second. It falls over. Now your cache is also cold. Describe what happens to the service." Candidate had no answer for the cascade.

What to do instead: Proactively identify thundering herd as a design concern whenever discussing cache TTL. Propose SETNX-based single-flight locking: "When a key expires, only one requester queries the database. Others wait with backoff. The database receives exactly one query per unique key miss, not N queries."

## General Pattern Mistakes

**Mistake 1: Proposing Modulo Hashing Without Knowing Its Failure Mode**
What happens: Candidate uses key % N for partitioning and either doesn't know about consistent hashing or uses it without explaining why.
Why wrong: Modulo hashing reshards (N-1)/N of all keys on any node addition or removal. For a cache, this means (N-1)/N of all queries become cache misses simultaneously — a thundering herd on the database at the moment of any scaling event.
Fix: "With N = 100 nodes, adding 1 node reshards 99% of keys simultaneously — every miss hits the database at once. Consistent hashing reshards only 1/(N+1) ≈ 1% of keys. The rest are unaffected."

**Mistake 2: O(N) LRU Implementation**
What happens: Candidate proposes sorting by access timestamp or iterating to find LRU on eviction.
Why wrong: O(N) eviction for N = 30 million keys means seconds of CPU per eviction event. Cache write performance collapses.
Fix: Know the O(1) doubly linked list + hashmap LRU by heart. It appears in this interview every time. Practice implementing it in 10 minutes.

**Mistake 3: No Thundering Herd Mitigation**
What happens: Candidate describes TTL correctly but doesn't address what happens when a popular key expires under load.
Why wrong: Popular keys are read millions of times per second. On expiry, millions of simultaneous misses hit the database — a spike that saturates and crashes it.
Fix: SETNX lock (single-flight pattern), staggered TTLs with jitter, or probabilistic early expiry. Pick one, explain it.

**Mistake 4: Ignoring Hot Keys**
What happens: Candidate partitions keys across nodes uniformly using consistent hashing and declares load distribution solved.
Why wrong: Consistent hashing distributes keys uniformly, but not access frequency. A single key read 10M times/second is always served by one node. That node becomes a hotspot regardless of how many nodes exist.
Fix: Identify hot keys (monitor access frequency per key). For detected hot keys: replicate the key to multiple nodes and distribute reads round-robin across replicas. This is separate from the consistency replication — it's a read scaling strategy.

**Mistake 5: Conflating Cache Write Strategies**
What happens: Candidate says "write to cache and database" for all writes without distinguishing use cases.
Why wrong: Write-through (cache + DB together) adds DB write latency to every cache write. For high-frequency counters (rate limiters, session counters), this defeats the cache. Write-behind buffers in cache and async flushes — fast but lossy on crash.
Fix: Name the three strategies, when to use each: write-through (user profile data — consistency critical), write-around (write-once audit logs — never re-read), write-behind (counters, session data — write-heavy, loss tolerable).

## Self-Assessment Checklist
- [ ] I explained WHY modulo hashing fails quantitatively (reshards (N-1)/N of keys)
- [ ] I described consistent hashing mechanics: hash ring, keys and nodes both on ring, clockwise assignment
- [ ] I proposed virtual nodes and explained why they solve uneven distribution
- [ ] I described O(1) LRU using doubly linked list + hashmap
- [ ] I addressed thundering herd proactively and proposed a mitigation
- [ ] I named all three write strategies (write-through, write-around, write-behind) with use cases
- [ ] I addressed node failure and its impact on cache hit rate
- [ ] I identified hot keys as a problem separate from load distribution

## Remediation Targets
- If you can't explain consistent hashing quantitatively: Practice with "chord" distributed hash table papers. Implement a consistent hash ring in code.
- If you can't implement O(1) LRU: This is LeetCode #146. Solve it until you can implement it from memory in 10 minutes.
- If you missed thundering herd: Study the "cache stampede" problem. Facebook's NSDI 2013 "Scaling Memcache" paper describes their lease solution. Read it.
- If you confused write strategies: Write-through / write-around / write-behind: draw all three as flowcharts. Practice naming which to use for: user profile (write-through), audit log (write-around), rate limiter (write-behind).
