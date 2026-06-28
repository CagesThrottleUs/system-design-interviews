# Problem 04: Social Feed (Twitter / Instagram)

**Difficulty:** Intermediate  
**Category:** Fan-out, Read-Heavy Systems, Caching, Distributed Architecture  
**Commonly Asked By:** Meta (E4–E6), Twitter/X, Instagram, Google, LinkedIn, ByteDance, Snap, Pinterest  
**Estimated Duration:** 45–50 minutes

---

## Problem Statement

You are a new infrastructure engineer at a social media company. The product team has just described the core user experience:

> "A user follows 500 people. They open the app. Within one second, they're scrolling through the most recent posts from everyone they follow. It feels instant. It always works. This is the product."

The company has grown to 300 million daily active users. Ten million posts are created every day. Each user on average follows 500 accounts, and each account on average has 500 followers.

The mobile app calls a `/feed` endpoint on load. That endpoint must return 20 posts in under 200 milliseconds at the 99th percentile. Right now, the system is buckling. Engineers are waking up at 3am. The on-call rotation is hell. Design a system that makes the feed reliable, fast, and cheap to operate at this scale.

**Your job is to design the backend architecture for this feed system.**

You are not designing the ranking algorithm, the recommendation system, or the ads pipeline — you are designing the infrastructure that gets the right posts in front of users, fast, at scale.

---

## Actors

| Actor | Description |
|-------|-------------|
| **Post Author** | Authenticated user who creates text/image/video posts |
| **Feed Consumer** | Authenticated user who loads their personalized feed |
| **Follower** | User who follows one or more authors; their feed is populated by followed authors' posts |
| **Celebrity / Influencer** | User with an extremely large follower count (think: public figures, brands) |

---

## Functional Requirements

| ID | Requirement |
|----|-------------|
| **FR1** | Users can create posts (text, image, video). A post is immediately visible on the author's profile. |
| **FR2** | Users can follow and unfollow other users. |
| **FR3** | A user's feed shows posts from accounts they follow, ordered by recency (newest first) by default. |
| **FR4** | The feed supports pagination — users can scroll to load older posts. |
| **FR5** | New posts should appear in follower feeds within a reasonable time (eventual consistency is acceptable; a few seconds of delay is fine). |
| **FR6** | Deleting a post should cause it to no longer appear in follower feeds. |

---

## Non-Functional Requirements

| Metric | Target | Source / Rationale |
|--------|--------|--------------------|
| **Feed load latency** | < 200ms p99 | Instagram's stated internal target; what users perceive as "instant" |
| **Feed load latency (p50)** | < 50ms | Typical median |
| **Post fan-out latency** | < 5 seconds for 95% of posts | Eventual consistency; real-time not required |
| **Availability** | 99.99% for feed reads | Feed failure is revenue-impacting; post writes can tolerate brief outage |
| **Consistency** | Eventual | Feed does not require linearizability; a post appearing 3 seconds late is acceptable |
| **Scale** | 300M DAU, 10M posts/day | Given |
| **Throughput (reads)** | ~35,000 feed loads/second (sustained) | 300M DAU × 10 feed loads/day ÷ 86,400s |
| **Throughput (writes)** | ~115 posts/second (sustained) | 10M posts/day ÷ 86,400s |
| **Storage** | Assume posts are small (text ≤ 280 chars); media stored separately | Simplification |

---

## Capacity Estimation Hints

Use these numbers to ground your design. You should derive the bottlenecks from them.

| Parameter | Value |
|-----------|-------|
| Daily Active Users | 300M |
| Average follows per user | 500 |
| Average followers per user | 500 |
| Posts created per day | 10M |
| Feed loads per user per day | ~10 |
| Posts returned per feed page | 20 |
| Average post size (metadata only, no media) | ~1 KB |
| Timeline cache entry size (post ID + score) | ~16 bytes |
| Timeline cache entries per user | ~200 (last N posts) |

Work out: how many writes does a single post trigger? What happens when a user with 50 million followers posts? What is the ratio of reads to writes?

---

## Clarifying Questions

A strong candidate asks these before drawing anything. They reveal the constraints that define the architecture.

1. **Feed ordering:** Is the feed strictly reverse-chronological, or does algorithmic ranking (engagement, relevance) apply? This affects whether we can precompute or must rank at read time.

2. **Celebrity / influencer accounts:** Are there users with millions of followers? How should the system handle them differently, if at all? (This is the most important question in the entire problem.)

3. **Feed staleness tolerance:** If I post something, how quickly must it appear in my followers' feeds? Is a 5-second delay acceptable? 30 seconds?

4. **Pagination strategy:** Cursor-based or offset-based? What happens when a user leaves the app and returns — do they see posts that were created while they were away, or does the feed reset to "current"?

5. **Read:write ratio:** The system is described as read-heavy. What is the expected ratio? (This determines how aggressively to precompute.)

6. **Follow graph scale:** Are there users with millions of followers? Thousands? What is the maximum follower count we need to handle without special casing?

7. **Post types and media:** Does the feed include images/video, or just text? (Affects payload size and storage strategy, but not fan-out architecture.)

8. **Deletion propagation:** When a user deletes a post, how quickly must it disappear from follower feeds? Immediately, or eventual?

9. **Geography:** Are users globally distributed? Does feed latency need to be consistent across regions?

10. **Active user ratio:** What fraction of followers are "active" (opened the app recently)? Pushing to inactive users wastes resources.

---

## Key Components

<details>
<summary>Expand only after your first attempt</summary>

These are the major components you should arrive at. If any are missing from your design, revisit the relevant requirement.

- **Post Service** — ingests new posts, stores them in a durable post store
- **Fan-out Service** — triggered by new post; writes post references into follower timelines
- **Follow Graph Store** — maps user → list of followers; queried during fan-out
- **Timeline Cache** — per-user precomputed feed (likely Redis sorted set by timestamp)
- **Feed Service** — serves `/feed` endpoint; reads from timeline cache + merges celebrity posts
- **Post Store** — durable storage of post content (relational or document DB)
- **Media Store** — separate blob/object storage for images and video
- **Message Queue** — decouples post creation from fan-out to handle bursts asynchronically

</details>
