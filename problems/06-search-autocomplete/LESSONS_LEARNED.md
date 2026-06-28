# Lessons Learned — Search Autocomplete / Typeahead

## Real Interview Stories

**Story 1: Google L5 Candidate — The Traversal Trap**
Company: Google | Level: L5 (Senior Software Engineer) | Round: System Design

What happened: Candidate proposed storing queries in a Trie and traversing to find completions at query time. Described the traversal algorithm in detail, correctly explaining DFS from the prefix node. Interviewer asked: "How long does it take to return completions for the prefix 'a'?" Candidate paused.

Where it went wrong: The candidate had not thought about the data distribution. In a real search corpus, "a" is a prefix to hundreds of millions of queries. Traversal is O(subtree size). For a single letter, this is catastrophic.

Consequence: Interviewer redirected to "how would you speed this up?" Candidate recovered with caching but lost 8 minutes of deep-dive time. Final score: "Below bar" on the scalability dimension.

What to do instead: Before designing data structures, ask "what is the distribution of prefixes?" Single-character prefixes are the hot path. Any solution that does work proportional to subtree size fails. Lead with precomputed top-K.

---

**Story 2: Amazon SDE2 Candidate — The Real-Time Write Trap**
Company: Amazon | Level: SDE2 | Round: System Design (LP+SD combined)

What happened: Candidate correctly identified the need for popularity-ranked completions. Then proposed: "Every time a user submits a search, we immediately update the frequency counter in the trie for that query." Interviewer probed: "What's the write volume on this path?"

Where it went wrong: 10M DAU × 10 searches = 100M writes/day = ~1,157 writes/second. Updating a shared trie node that thousands of users share (e.g., "iphone") under high concurrency requires locks or atomic operations. The candidate's design would serialize writes to hot nodes, creating a bottleneck on the most popular queries.

Consequence: Candidate couldn't explain how to handle concurrent writes to shared trie nodes without degrading read performance. Interview ended early on this thread.

What to do instead: Decouple writes from reads completely. Log every search query to Kafka. An async aggregation job (Spark/Flink) recomputes frequencies offline. The trie is rebuilt daily from aggregated data. Write path never touches the serving trie.

---

**Story 3: Twitter/X SDE Candidate — Personalization Scope Creep**
Company: Twitter | Level: SDE | Round: System Design

What happened: Candidate immediately jumped to personalized completions without checking whether personalization was in scope. Spent 15 minutes designing a user-specific completion model with embedding similarity and personal search history, completely missing the global popularity index.

Where it went wrong: The interviewer's rubric was "can this candidate build a correct global autocomplete system?" The candidate optimized for a Phase 2 feature that wasn't asked for, leaving the core system unspecified.

Consequence: When interviewer asked "walk me through what happens when a user types 'iph' for the first time — what does your system return?", candidate couldn't answer because global fallback was never designed.

What to do instead: Ask explicitly: "Should completions be personalized, or global popularity only?" Build the global system completely first. Add personalization as a named extension at the end.

## General Pattern Mistakes

**Mistake 1: Conflating Trie Traversal with Top-K Lookup**
What happens: Candidate implements a trie correctly for prefix matching but then proposes traversing the subtree at query time to find completions, sorting by frequency.
Why wrong: Subtree traversal for short prefixes is O(N) where N = millions of nodes. For "a", this is seconds of computation. The answer must be O(1) or O(log K).
Fix: Store top-K completions at each trie node. Query time is O(prefix length) for trie traversal + O(1) to read the precomputed list.

**Mistake 2: Making Writes Synchronous**
What happens: Candidate updates the popularity index on every search submission, in the hot path.
Why wrong: Creates write amplification on the busiest nodes, contention on shared counters, and tight coupling between the read and write paths. A slow popularity update blocks the search submission.
Fix: Fire-and-forget log to Kafka. Async aggregation job recomputes popularity offline. The serving index is read-only and rebuilt periodically.

**Mistake 3: Forgetting the Debounce**
What happens: Candidate designs for the raw keystroke volume (every key = one API call).
Why wrong: Overestimates required server capacity by 5-10×. User typing "iphone" sends 6 requests per search if no debounce. Also misses a free 80% load reduction that the frontend already provides.
Fix: State upfront: "Client debounces at 150ms. This reduces API calls per search from ~10 to ~2. Effective RPS is 20,000, not 100,000."

**Mistake 4: Under-designing the Blocklist**
What happens: Candidate treats blocklist as a simple word filter, applied client-side or cached in the CDN response.
Why wrong: Blocked queries cached at CDN will continue being served until TTL expires. Applying client-side means the server still computes and returns blocked results — just hidden from the user. A blocked result could still appear in server logs, rate limiting, or CDN responses.
Fix: Apply blocklist server-side, before caching. Cache the filtered response. Use short TTL (10 minutes) on cached responses to limit stale blocklist exposure.

**Mistake 5: No Multi-Level Caching**
What happens: Candidate proposes one cache (Redis) and calls it done.
Why wrong: Single-layer cache still sends ~30,000 RPS to Redis at peak. Adding CDN caching for the most common prefixes (one and two characters, common three-character combos) can absorb 60-70% of traffic at the edge, before hitting origin.
Fix: CDN caches responses for short prefixes with Cache-Control: public, max-age=600. Redis caches the full prefix corpus. Prefix Index serves cache misses.

## Self-Assessment Checklist
- [ ] I calculated the read RPS from first principles (DAU × searches × keystrokes ÷ seconds) before debounce adjustment
- [ ] I explained WHY on-the-fly trie traversal fails for short prefixes
- [ ] I proposed precomputed top-K stored at each trie node
- [ ] I separated the read path (serving) from the write path (index updates) completely
- [ ] I designed the write path as async (Kafka → batch aggregation → index rebuild)
- [ ] I proposed a multi-level cache (CDN + Redis + Index)
- [ ] I addressed blocklist and where in the pipeline it applies
- [ ] I mentioned client-side debounce as the first latency optimization
- [ ] I quantified the index size and confirmed it fits in memory

## Remediation Targets
- If you missed precomputed top-K: Study the Top-K Frequent Elements problem on LeetCode (347). Then apply to tries — each internal node stores its own top-K.
- If you missed async writes: Study Kafka + Flink/Spark streaming architectures. The search log → frequency update pipeline is a textbook stream processing use case.
- If you missed debounce: It's a frontend concept — `setTimeout` cleared on each keystroke. Understand why it matters for API design (reduces QPS by 5-10×).
- If you ran out of time: Practice the Requirements → Estimation → HLD → one Deep Dive flow. Budget 5 minutes max on requirements, 5 on estimation, 15 on HLD, 15 on deep dive.
