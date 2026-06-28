# Mock Interview Script: Social Feed

**Interviewer persona:** Staff Engineer, infrastructure background, has seen this question 40+ times. Will not volunteer information. Will push on every claim. Rewards proactive identification of hard cases.

**Time target:** 45-50 minutes  
**Level calibration:** Apple ICT3 / Meta E5 / Google L5

---

## Phase 0: Setup (2 minutes)

> "Let's get started. I'd like you to design a social feed system — similar to Twitter's timeline or Instagram's feed. Users can post content, follow other users, and see a feed of posts from the accounts they follow. I'll let you drive, and I'll jump in with questions as we go. You've got about 45 minutes. Take it away."

*[Wait. Do not add anything unless 90 seconds pass with no response.]*

*[If candidate asks where to start:]* "Wherever makes sense to you."

---

## Phase 1: Requirements Clarification (Target: 5 minutes)

*Allow the candidate to ask clarifying questions. Score them on which questions they ask and which they miss.*

**If candidate asks about scale:**  
> "300 million daily active users. 10 million posts per day."

**If candidate asks about reads vs writes:**  
> "Users check their feed roughly 10 times per day. What does that tell you?"

**If candidate asks about follower counts / celebrity accounts:**  
> "Yes, we have some very large accounts. Think public figures with millions of followers. That's an interesting question — why are you asking?"  
*(Let them explain. This is a good sign.)*

**If candidate does NOT ask about celebrity accounts within 5 minutes:**  
*Do not volunteer it. Let them proceed. You will surface it during Phase 3.*

**If candidate asks about algorithmic vs chronological feed:**  
> "Start with reverse-chronological. We can discuss ranking as an extension."

**If candidate asks about post deletion:**  
> "When a user deletes a post, it should no longer appear in feeds. Exact timing is flexible."

**If candidate asks about consistency:**  
> "Feed doesn't need to be real-time. A few seconds of delay is acceptable."

**If candidate asks about geography:**  
> "Assume a single region for now. Multi-region is an extension."

*[After 5-6 minutes, if requirements are still going:]* "Sounds like you have a good handle on the requirements. Want to start talking about the architecture?"

---

## Phase 2: High-Level Design (Target: 15 minutes)

*Let the candidate draw. Do not interrupt except to probe vague statements.*

**If candidate mentions "write to followers' feeds" without addressing celebrities:**  
> "Okay, so when a user posts, you write to each follower's feed. Walk me through that for a specific user. Say, Selena Gomez."

**If candidate says "fan-out to all followers":**  
> "She has about 405 million followers. What does that mean for your fan-out?"

**If candidate gives a vague number ("a lot of writes"):**  
> "Let's get specific. If Redis handles 10,000 writes per second per instance, how long does it take to complete the fan-out for one of her posts?"

**If candidate correctly identifies the celebrity problem unprompted:**  
*[nod, make a brief note, let them continue — this is a strong positive signal]*  
> "Interesting. So how do you handle that?"

**If candidate proposes the hybrid model:**  
> "Where's the threshold? How do you decide who's a 'celebrity'?"

**If candidate can't give a specific threshold:**  
> "If a user has 10 thousand followers versus 10 million followers, is the treatment the same or different?"

**If candidate proposes fan-out-on-read for ALL users:**  
> "How many DB queries does a single feed load generate for a user who follows 500 accounts?"  
*(Answer: 500. At 35K feed loads/sec, that's 17.5M queries/sec — not feasible.)*

**If candidate proposes fan-out-on-write for ALL users:**  
> "What happens when Selena Gomez posts?"  
*(Surface the math.)*

**If candidate draws the queue/async fan-out:**  
> "Why a queue here instead of writing directly to Redis?"

---

## Phase 3: THE Critical Probe (This is the moment that separates L4 from L5)

**Timing:** ~25 minutes in. Regardless of where the candidate is in their design.

> "I want to make sure I understand your design end-to-end. Selena Gomez posts a photo of herself in Milan. Walk me through everything that happens in your system — from the moment she hits post to the moment her 405 million followers see it in their feed. Don't skip any steps."

*[This is the most important question in the entire interview. Listen carefully.]*

**Signals to look for:**

| What they say | What it means |
|---------------|---------------|
| "Posts go into Kafka, fan-out workers write to each follower's Redis timeline" with no celebrity caveat | Did not internalize the celebrity problem — L4 ceiling |
| "She's above the celebrity threshold so we skip individual fan-out" + describes celebrity post cache | Understood write path — halfway there |
| Describes the celebrity post cache AND the merge step at read time | Fully understands the hybrid — strong L5 signal |
| Gives the approximate math unprompted (405M × 10K writes/sec = 11 hours) | Exceptional — rare |

**Follow-up probes:**

If they describe the write path correctly but stop:  
> "Good. Now a user who follows Selena opens their feed ten minutes after she posts. What does the Feed Service do to include her post?"

If they say "query the timeline cache":  
> "But you said you didn't fan-out to individual timelines. So how is her post in there?"

*[The correct answer: the Feed Service fetches from the celebrity post cache separately and merges at read time. If they can't explain this, probe deeper.]*

> "You've designed two separate data structures — the precomputed timeline and the celebrity post cache. How do these come together into a single feed response?"

*[This is the merge step question. If they can't answer it, they have a hole in their design.]*

---

## Phase 4: Data Model Deep Dive (Target: 8 minutes)

> "Let's talk about your data model. What does a timeline entry look like in Redis? What exactly are you storing?"

**If they say "the full post":**  
> "Let's think about memory. If you store 1KB posts per user, 200 entries each, for 300 million users — how much RAM is that?"  
*(Answer: 300M × 200 × 1KB = 60 TB — not feasible.)*

**If they correctly say post_ids only:**  
> "Good. Then how do you get the actual post content when you're assembling the feed response?"

**If they propose a sorted set (correct):**  
> "Walk me through the ZADD and ZREVRANGEBYSCORE operations. What's the score? What's the member?"

**If they propose a list (incorrect):**  
> "How do you do cursor-based pagination with a list if new posts can be inserted at arbitrary positions?"

**On pagination:**  
> "A user reads their feed and scrolls to the bottom. They want to see more. How does pagination work? What information travels between the client and the server to mark their position?"

**If they say "offset":**  
> "What happens to offsets when a new post is inserted at the top of the feed between two page loads?"  
*(Answer: every offset shifts by one — the user gets duplicate or skipped posts.)*

---

## Phase 5: Failure Modes (Target: 5 minutes)

> "Let's talk about failure. Your Redis cluster loses one node. What happens to users whose timeline happened to be on that node?"

*[Looking for: graceful degradation — feed falls through to DB or returns a degraded but functional feed; not silent failure.]*

> "Your fan-out Kafka consumers fall behind. Posts aren't being written to timelines. How do you detect that, and what does the user experience look like while it's happening?"

*[Looking for: consumer lag monitoring via Kafka offsets; feeds go stale but reads still work; recovery is automatic when consumers catch up.]*

> "A user who hasn't opened the app in 3 months logs back in. Their timeline cache key has been evicted from Redis. What happens?"

*[Looking for: cache miss → feed rebuild from post store; candidate may describe a lazy rebuild triggered by the Feed Service, or a background warm-up job.]*

---

## Phase 6: Extensions (Time permitting)

These are positive signals if raised; not blockers if omitted.

> "You've designed a reverse-chronological feed. Instagram and Twitter/X both moved to algorithmic ranking. What would change in your architecture to support that?"

*[Looking for: larger candidate pool from cache, scoring/ranking service in the read path, ML inference latency constraints, ranking can't exceed the 200ms SLA.]*

> "What if a user follows 50,000 accounts? Is your design still fast at read time?"

*[Looking for: celebrity merge with 50,000 celebrity caches is expensive; need a cap on celebrity follows, or a different strategy for power users.]*

> "How would you extend this to multiple geographic regions?"

*[Looking for: fan-out must be aware of region; celebrity post caches replicated globally; follow graph replicated with eventual consistency.]*

---

## Phase 7: Wrap-up (2 minutes)

> "We're coming up on time. Two quick questions: In your design, what's the weakest component — the one you'd lose sleep over in production?"

> "If you had another hour, what would you add or change?"

> "Thanks — I think we've covered a lot. Any questions for me?"

---

## Scoring Checklist

### Requirements & Estimation (3 points)

- [ ] **+1** — Asked about celebrity accounts / large follower counts during requirements
- [ ] **+1** — Correctly derived feed read QPS (~35K/sec) or a reasonable estimate
- [ ] **+1** — Identified read:write asymmetry and used it to justify precomputed timelines

### Write Path Design (6 points)

- [ ] **+1** — Correctly identified fan-out-on-write as the baseline approach
- [ ] **+1** — Identified the celebrity problem (large follower count breaks naive fan-out)
- [ ] **+1** — Proposed hybrid: fan-out-on-write for regular users, fan-out-on-read for celebrities
- [ ] **+2** — Named a specific celebrity threshold and justified it
- [ ] **+1** — Mentioned inactive user optimization (skip cold timelines during fan-out)

### Read Path Design (6 points)

- [ ] **+1** — Described Redis sorted set as timeline cache structure
- [ ] **+1** — Explained post_id only in cache (not full post content)
- [ ] **+2** — Described the merge step: precomputed timeline + celebrity post cache merged at read time
- [ ] **+1** — Correctly described cursor-based pagination (not offset)
- [ ] **+1** — Described post hydration as a separate step from timeline fetch

### Data Model & Failure (4 points)

- [ ] **+1** — Mentioned need to shard post store (author_id sharding or distributed DB)
- [ ] **+1** — Described graceful degradation when Redis is unavailable
- [ ] **+1** — Handled post deletion (soft delete + hydration-time filter, OR cache eviction)
- [ ] **+1** — Handled unfollow (lazy or eager timeline cleanup)

### Communication (2 points)

- [ ] **+1** — Design was presented proactively (not extracted through prompting)
- [ ] **+1** — Math and trade-offs were stated clearly without ambiguity

---

**Total: /21**

| Score | Signal |
|-------|--------|
| 18–21 | Exceptional — exceeds L5 bar; hire strong |
| 14–17 | Strong — meets L5 bar; hire |
| 10–13 | Adequate — meets L4 bar; may need coaching on L5 signals |
| 6–9   | Developing — understands the problem but can't design at scale; no hire L5 |
| 0–5   | Fundamental gaps — no hire |

---

## Interviewer Notes on Common Deflections

**"We can shard Redis later."**  
> "We're at 300M DAU today. At what point do you shard, and how do you migrate without downtime?"

**"We'd use Elasticsearch for the feed."**  
> "Elasticsearch is optimized for search, not time-ordered fan-out. What specific property of Elasticsearch makes it right here versus Redis sorted sets?"

**"Kafka will handle the burst."**  
> "Kafka handles the burst by buffering, not by reducing the total write work. If fan-out workers can't keep up with the queue's ingestion rate, what happens to the consumer lag over time?"

**"We can cache the follow graph in memory."**  
> "Memory of what — the fan-out workers? For 300M users with 500 follows each, how much memory is that follow graph?"  
*(Answer: 300M × 500 × 8 bytes per user_id = 1.2 TB — doesn't fit in a single machine.)*
