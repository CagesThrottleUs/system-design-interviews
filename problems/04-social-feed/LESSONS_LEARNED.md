# Lessons Learned: Social Feed

Real failure patterns observed in system design interviews at top companies. Each story reconstructs the exact moment the interview went wrong and what should have happened instead.

---

## Story 1: Meta E5 Loop — The Fan-out Trap

**Company:** Meta (Menlo Park, Infra loop, 2024)  
**Role:** E5 Software Engineer  
**Candidate background:** 7 years experience, strong distributed systems background, had read Alex Xu's book

### What happened

The candidate opened strongly. Functional requirements in 3 minutes. Scale estimates were accurate: 300M DAU, 35K feed reads/sec, 115 posts/sec writes. The interviewer was impressed.

Then the candidate drew the write path. They sketched: Client → Post Service → Fan-out Worker → Redis timeline per follower. Correct structure. But they labeled every user the same way. No celebrity handling. No threshold. Just one fan-out path for everyone.

The interviewer asked: **"Selena Gomez posts a photo. Walk me through what happens in your system."**

The candidate said: "The Post Service writes the post, publishes to Kafka, the Fan-out Worker picks it up and writes the post ID into each follower's timeline in Redis."

The interviewer waited.

The candidate continued: "With 405 million followers, that's 405 million Redis writes..."

A pause.

"...which would take a while."

The interviewer asked: "How long?"

The candidate didn't have the math. They said "a few minutes, maybe." The correct answer is 11 hours at 10K writes/sec — the system is permanently behind, never catching up.

The interviewer then asked: "So how would you fix it?"

The candidate said: "Maybe... a faster write path? Parallelizing the fan-out workers more?"

This is the second mistake. More workers doesn't fix the celebrity problem — you still need to issue 405M writes. The fix is architectural: don't fan-out celebrities at write time at all.

The candidate eventually arrived at the hybrid model after 8 minutes of prodding. But arriving at the answer under prompting is not the same as arriving at it before being asked. At E5, the interviewer expects the candidate to identify the celebrity problem proactively, before drawing the write path.

### What should have happened

Before drawing anything, during requirements clarification:

> "One thing I want to flag early — are there users with very large follower counts? Like celebrities or public figures with millions of followers? Because that completely changes the fan-out strategy, and I want to make sure I'm designing for the real constraint."

Then, during capacity estimation:

> "If Selena Gomez has 405 million followers and posts 10 times a day, a naive fan-out-on-write approach would require 405 million Redis writes per post. At 10K writes/sec, that's 11 hours — she'd post again before we finished. So we can't treat her like a regular user. I'll use a hybrid model: fan-out-on-write for regular users below a threshold — say 10,000 followers — and fan-out-on-read for celebrities above that threshold."

This is what separates L4 from L5. The L4 candidate discovers the problem when pushed. The L5 candidate pre-empts it.

---

## Story 2: Twitter/X — The Missing Merge Step

**Company:** Twitter/X (Infrastructure design interview, 2023)  
**Role:** Senior Software Engineer (L5 equivalent)  
**Candidate background:** 5 years at a mid-size company, strong Redis knowledge

### What happened

This candidate was technically excellent at the Redis layer. They correctly identified the hybrid model without prompting. They drew the two fan-out paths. They named the celebrity threshold at 50K followers. The interviewer was satisfied with the write path.

Then the interviewer asked about the read path: **"A user follows 3 regular users and Cristiano Ronaldo. They open their feed. What does the Feed Service do?"**

The candidate said: "The Feed Service fetches their precomputed timeline from Redis — ZREVRANGEBYSCORE — and returns the top 20 posts."

The interviewer: "Where are Cristiano Ronaldo's posts?"

The candidate froze. They had designed a write path that correctly skipped fan-out for Ronaldo. But they had never designed where those posts would be stored or how they'd be retrieved at read time.

After a long pause, they said: "Oh — I'd need to fetch Ronaldo's recent posts separately and merge them."

The interviewer: "Show me that on the whiteboard."

The candidate drew a box labeled "Celebrity Post Fetch" but couldn't explain the data model. They said "we'd query the post database for Ronaldo's recent posts" — which is correct but incomplete. At Twitter's scale, querying Postgres for a celebrity's recent posts on every feed load at 35K QPS would obliterate the database.

The correct answer is a celebrity post cache: a separate Redis list per celebrity (e.g., `celeb_posts:{celebrity_id}`) that stores their last 50 post IDs. Fan-out for celebrities writes to this list instead of individual follower timelines. Feed Service reads this list for each followed celebrity and merges the results with the precomputed timeline before pagination.

The candidate recovered partially but never fully articulated the merge step. The interviewer marked "incomplete feed assembly" in their notes.

### What should have happened

When presenting the read path, explicitly narrate the merge:

> "The Feed Service does three things. First, it fetches the precomputed timeline from Redis — that's the sorted set of post IDs from regular users they follow. Second, it identifies which celebrities this user follows — I'll cache that as a separate set. Third, for each followed celebrity, it fetches recent post IDs from the celebrity post cache, which is a Redis list I maintain separately. Then it merges these two lists — precomputed timeline plus celebrity posts — sorts by timestamp in memory, applies the cursor, and paginates to 20. The merge step is what makes the hybrid model complete. Without it, celebrity posts never appear in anyone's feed."

Always name the merge step. Draw it explicitly on the whiteboard. It is the load-bearing connection between the two halves of the hybrid model.

---

## Story 3: Google — The Inactive User Oversight

**Company:** Google (YouTube Home Feed variant, L5 Systems Design, 2024)  
**Role:** L5 Software Engineer  
**Candidate background:** 8 years, background in video infrastructure

### What happened

The candidate presented a solid hybrid fan-out design. The math was correct. The architecture was sound. When asked about optimizations, they discussed cursor-based pagination, Redis sharding, Kafka partitioning by author_id to preserve ordering. The interviewer was nodding.

Then the interviewer asked: **"You mentioned 300M DAU. Of the 405 million followers Selena Gomez has, how many are daily active?"**

The candidate paused. They had been computing fan-out based on total follower count, not active follower count.

The interviewer continued: "What percentage of followers do you think open the app in any given 30-day window?"

The candidate admitted they hadn't thought about it. Industry estimates suggest 20–40% of followers are genuinely active in any 30-day window. For Selena Gomez: even at the 50% activity rate, that's still 202 million followers — but at 20%, it's 81 million. The distinction matters less for celebrities (who use fan-on-read anyway) but enormously for regular users.

For a regular user with 500 followers: if only 30% are active in the last 30 days, fan-out should write to ~150 timelines, not 500. That's a 3x reduction in write volume — entirely free, requires only tracking last-active timestamp per user.

The interviewer noted: "Your design is functionally correct but operationally expensive. You're burning 3x the Redis write budget by not filtering inactive users."

### What should have happened

During the fan-out design:

> "One optimization I want to flag: I won't fan-out to users who haven't opened the app in the last 30 days. Their timeline cache is cold anyway — they'll get a cache miss on first login and we can rebuild it then. The write savings are significant: if only 30% of followers are active, we reduce fan-out writes by 70%. I'll track last_active_at per user in the follow graph cache and skip cold users during fan-out. When a cold user logs back in, the Feed Service detects a missing timeline key and triggers a background rebuild by pulling recent posts from their followees."

This optimization demonstrates operational maturity — understanding the difference between correct and efficient, and knowing that at 300M users, even a 2x write reduction translates directly to cost and stability.

---

## General Pattern Mistakes

### Mistake 1: "Hybrid" Without Numbers

Saying "use a hybrid approach for celebrities" without defining what makes a celebrity. The threshold is the design. Without a number, the statement is meaningless.

**Wrong:** "For celebrity users, we'll use fan-out-on-read."  
**Right:** "For users with more than 10,000 followers, we skip individual fan-out and write to a celebrity post cache instead. That threshold is configurable at runtime, and we tune it based on fan-out worker capacity."

### Mistake 2: Offset Pagination

Using `LIMIT 20 OFFSET 200` for feed pagination. This breaks under concurrent inserts (new posts shift every offset), is O(N) on the database, and cannot handle the load at 35K feed loads/sec.

**Wrong:** Offset-based pagination.  
**Right:** Cursor-based pagination where cursor = timestamp of the last post seen. `ZREVRANGEBYSCORE timeline:{user_id} {cursor_ts} -inf LIMIT 0 20`.

### Mistake 3: Storing Full Post Content in Timeline Cache

The timeline cache stores post IDs (16 bytes each), not full post content. Storing 1KB posts per entry × 200 entries × 300M users = 60 TB in Redis. That's neither feasible nor necessary. Timeline cache is an index; Post Store is the source of truth.

**Wrong:** Store full post objects in the timeline sorted set.  
**Right:** Store post_ids in the timeline. Hydrate from a separate post cache (Redis or Memcached) or the Post Store on read.

### Mistake 4: Single Post Database

At 10M posts/day × 365 = 3.65B posts/year, a single unsharded Postgres instance will not sustain the write load or the storage. The Post Store must be designed for horizontal scaling from day one.

**Wrong:** "We'll store posts in Postgres."  
**Right:** "We'll shard the Post Store by author_id using consistent hashing. Alternatively, Cassandra provides wide-column storage that's naturally distributed and write-optimized — it's what Instagram used before their PostgreSQL migration for activity feeds (where Cassandra gave ~75% cost savings vs Redis for long-tail timelines). Source: Instagram Engineering, 2012."

### Mistake 5: No Unfollow Handling

Candidates design the follow path and forget unfollow. When User A unfollows User B, posts from User B that are currently sitting in User A's precomputed timeline cache should no longer appear.

Options:
- **Lazy removal:** At feed read time, filter out posts from users not in the follow graph. Requires a follow set lookup per feed load.
- **Eager removal:** On unfollow, remove all of User B's posts from User A's timeline cache. Expensive if User B has posted frequently.

Best approach: lazy removal with a small additional lookup cost per feed page. This avoids expensive cleanup on unfollow and self-corrects naturally.

---

## Self-Assessment Checklist

Before calling your solution complete, verify you can answer each question cold:

**Architecture:**
- [ ] Can I explain the hybrid fan-out model in two sentences without the whiteboard?
- [ ] Can I name the celebrity threshold and justify the number?
- [ ] Can I explain the merge step at read time — where celebrity posts come from and how they join the precomputed timeline?
- [ ] Can I draw the data flow for a regular post from POST request to timeline cache entry in under 2 minutes?
- [ ] Can I draw the data flow for a Selena Gomez post (celebrity) from POST request to appearing in a follower's feed in under 2 minutes?

**Math:**
- [ ] Can I derive 34,722 feed reads/sec from the given assumptions?
- [ ] Can I derive the celebrity fan-out problem (405M writes, 11 hours at 10K/sec) on demand?
- [ ] Can I size the Redis timeline cache (1 TB for 300M users)?
- [ ] Can I explain the read:write ratio (~300:1) and what it implies?

**Edge Cases:**
- [ ] How does feed behave when a user's timeline cache is cold (first login or long absence)?
- [ ] How does post deletion work? Is it immediate or eventual?
- [ ] How does an unfollow affect the existing timeline cache?
- [ ] What happens to a fan-out if a follower's Redis key doesn't exist (user never logged in)?
- [ ] What happens when a regular user crosses the celebrity threshold?

**Failure Scenarios:**
- [ ] Redis node failure — what degrades and how?
- [ ] Fan-out Kafka consumer lag — how do I detect it and what's the impact?
- [ ] Post Store unavailable — can I still read feeds?

---

## Remediation Targets

If you failed or struggled on any of the above, study in this order:

| Gap | Study Target |
|-----|-------------|
| Celebrity problem math | Twitter High Scalability post (2013); do the 405M × 10K/s calculation from memory |
| Merge step | Implement a toy version: two sorted arrays merged by timestamp — then map to Redis ZREVRANGEBYSCORE + LRANGE |
| Redis sorted set operations | Redis documentation: ZADD, ZREVRANGEBYSCORE, ZREMRANGEBYRANK — understand time complexity of each |
| Cursor-based pagination | Read Slack's or Stripe's API pagination design; implement cursor from a sorted set |
| Cassandra vs Postgres for fan-out | Instagram Engineering Blog 2012: "Storing hundreds of millions of simple key-value pairs in Redis" — real production decision rationale |
| Inactive user optimization | Think through the math: 30% active rate → 70% write reduction; design the last_active_at lookup |
| Kafka fan-out pattern | High Scalability: "How Twitter Uses Redis to Scale" — consumer group scaling pattern |
