# Mock Interview Script — Search Autocomplete / Typeahead
*For the interviewer. Do not share with candidate.*

## Pre-Session Setup
- Problem: Design a search autocomplete / typeahead system
- Target level: Apple ICT3 / Google L5 / Amazon SDE2
- Time: 45 minutes
- Have a timer visible. Note the exact minute of each phase transition.
- Do not volunteer information. Answer only what is asked.

## Opening (read aloud, verbatim)
"Design a search autocomplete system. As a user types in a search box, the system should display the top search query completions in real time. Think of Google Search or the Amazon search bar. You have 45 minutes. Please start by asking me any clarifying questions you have."

*Start timer. Say nothing else.*

## Phase 1: Requirements Clarification (0–8 min)

**What to look for:**
Strong candidates ask about scale before jumping to design. They want to know DAU, QPS, and latency targets. They ask about personalization (is it in scope?). They ask about freshness (how quickly must trending queries appear?).

**Answers to give if asked:**
| Question | Answer |
|----------|--------|
| How many users? | 10 million DAU |
| How many QPS? | You figure it out from the DAU |
| Personalized or global? | Global popularity only for MVP |
| How fast must completions appear? | Sub-100ms end-to-end, p99 |
| How many results? | Top 5 |
| How quickly must trending queries appear? | Within 24 hours |
| Multiple languages? | English only for MVP |
| Blocklist? | Yes, blocked queries must never appear |
| Mobile + web? | Both |

**Red flags in requirements phase:**
- Jumps immediately into design without asking any questions
- Asks about personalization without first confirming global popularity is sufficient
- Does not ask about latency targets

## Phase 2: Estimation (8–13 min)

**If they skip estimation:** "Before we get to the design, can you do a quick back-of-envelope on the read QPS and storage requirements?"

**Expected calculation:**
- Read QPS: 10M DAU × 10 searches × 10 keystrokes / 86,400s ≈ 11,574 RPS baseline
- With debounce (if they mention it): ~2-4 RPS per search → ~2,315-4,630 RPS baseline
- Peak 5×: ~12,000-25,000 RPS
- Index size: should arrive at "a few GB, fits in memory"

**If stuck on debounce:** "What do you know about how search bars typically implement autocomplete on the client side?"

**If stuck on index size:** "How many unique queries might be in the index? What's the average query length?"

## Phase 3: High-Level Design (13–28 min)

**Expected components:**
- Client-side debounce
- CDN edge cache (for short prefixes)
- Stateless autocomplete service
- Redis cache layer
- Distributed trie / prefix index
- Async write path: search logs → Kafka → Spark/Flink → Trie Builder
- Blocklist filter

**Hint progression (use only if candidate is stuck):**
1. (0 hints given) Stay silent. Let them struggle.
2. "What data structure would you use to store the queries and their prefixes?"
3. "Once you have the trie, how long does it take to find top-5 completions for the prefix 'a'?"
4. (If they still don't see the traversal problem) "How many queries start with the letter 'a' in a billion-query corpus?"
5. "What if you precomputed the answer at each trie node?"

**Key question to ask at 20 min:** "Walk me through exactly what happens from the moment I type the letter 'i' to when I see completions on screen. Be specific."

**Key question to ask at 25 min:** "How does the popularity data get updated? If a new trending query appears today, when does it show up in completions?"

## Phase 4: Deep Dive (28–40 min)

**Pick ONE of these deep dives based on what the candidate has covered:**

**Option A — Trie and Top-K (if they haven't addressed precomputed top-K):**
"You've proposed a trie. Let's go deeper. At each node in the trie, what exactly do you store? How does that change as the trie grows to 1 billion queries?"

**Option B — Sharding (if they proposed a distributed trie without explaining sharding):**
"Your trie is too large for one machine. How do you shard it? What are the trade-offs between sharding by prefix range versus consistent hashing?"

**Option C — Freshness (if they haven't addressed the write path in detail):**
"Walk me through the complete pipeline for how a new trending query — say, the name of a product that launched this morning — appears in autocomplete by end of day. What are all the steps?"

**Option D — Cache invalidation (if they have a good design otherwise):**
"You have completions cached at CDN and Redis. The admin just added a query to the blocklist. How do you ensure that query stops appearing in completions immediately?"

## Phase 5: Failure & Scaling Probe (40–44 min)

Ask TWO of these (pick based on gaps observed):

1. "Your Redis cache goes down. Walk me through exactly what happens. Does autocomplete still work?"
2. "The trie rebuild job fails tonight. What do users experience tomorrow morning?"
3. "You're getting 10× normal traffic due to a viral product launch. Your autocomplete service is at capacity. What fails first? How do you respond?"
4. "How would your design change if you needed completions within 50ms instead of 100ms?"
5. "Describe a scenario where a user sees a completion that was just added to the blocklist 30 seconds ago. How does this happen and how do you prevent it?"

## Closing (44–45 min)
"We're almost out of time. Two quick questions: What do you think is the weakest part of your design? And what would you add in a second iteration?"

*Stop here. Do not ask follow-ups. Take notes for scoring.*

## Scoring Checklist

| Dimension | Evidence of Excellence | Score (1-5) |
|-----------|----------------------|-------------|
| Requirements | Asked about scale, latency, freshness, and personalization scope before designing | |
| Estimation | Correctly derived RPS from DAU; estimated index fits in memory; mentioned debounce effect on QPS | |
| Data Structure | Proposed precomputed top-K at each trie node; explained why on-the-fly traversal fails | |
| Read Path | Designed multi-level caching (CDN + Redis + Index); discussed latency budget allocation | |
| Write Path | Completely separated write path as async; Kafka → batch aggregation → index rebuild | |
| Failure Modes | Addressed Redis failure, Trie Builder failure, and cache invalidation without prompting | |
| Communication | Explained trade-offs, not just solutions; drove the conversation rather than waiting for prompts | |

**Hire bar:** 4+ on all dimensions, or 5 on 4 dimensions with 3 on remaining 3.
**Strong hire bar:** 5 on 5+ dimensions.
