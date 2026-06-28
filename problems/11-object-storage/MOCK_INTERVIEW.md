# Mock Interview Script — Object Storage (S3-like)

**For the interviewer only. Do not share with the candidate.**

---

## Pre-Session Setup

**Target level:** Senior engineer (L5/ICT3 equivalent)
**Expected duration:** 45 minutes
**Red flags that should reduce score:** Routing upload bytes through app servers, no metadata/data separation, ignoring erasure coding, no multipart design, no garbage collection
**Green flags that should increase score:** Proactive erasure coding justification with cost math, pre-signed URL flow, named multipart upload protocol, strong consistency discussion with the Dec 2020 S3 note

---

## Opening

"I'd like you to design an object storage service — think S3 or GCS. Customers can upload arbitrary files, download them by key, and delete them. Files range from a few kilobytes to several terabytes. Design the system for a company that has 10 million active users and is targeting 11 nines of durability."

*Then stop. Let the candidate ask clarifying questions.*

---

## Phase 1: Requirements (0–8 min)

**Expected clarifying questions from candidate:**
- Object size range? (Good — leads to multipart discussion)
- Read-to-write ratio? (Good — drives caching)
- Cross-region replication? (Good — drives consistency discussion)
- Strong vs eventual consistency? (Good — shows awareness of consistency model)
- Object versioning? (OK — secondary feature)

**If candidate doesn't ask about consistency, prompt:** "One of your customers uploads an object and immediately tries to read it. Is that read guaranteed to succeed?"

**If candidate doesn't ask about file size distribution, prompt:** "You mentioned 10 million users. What if I told you some objects are 5 TB database dumps? Does that change anything?"

**Gate to next phase:** Candidate must have clarified scale, consistency expectation, and large-file handling before proceeding.

---

## Phase 2: Estimation (8–13 min)

**Let candidate work through math, then prompt if they miss critical numbers:**

Expected numbers:
- Write QPS: ~580 writes/sec (50M uploads/day ÷ 86,400)
- Read QPS: ~5,800 reads/sec (10:1 ratio)
- Total storage: ~10 PB raw (10B objects × 1 MB average)
- With erasure coding (1.8×): 18 PB physical
- With 3× replication: 30 PB physical — and if candidate doesn't notice the 12 PB difference, point it out

**Key probe (if candidate proposes replication):** "At 10 PB of data, what's the cost difference between 3× replication and 5+4 erasure coding at $0.023/GB/month?"
- Replication: 30 PB × $0.023/GB = **$690M/year**
- Erasure coding: 18 PB × $0.023/GB = **$414M/year**
- Difference: **$276M/year** — this is why S3 uses erasure coding

---

## Phase 3: HLD (13–28 min)

**Expected components from candidate (in approximately this order):**
1. API service (stateless, horizontally scalable)
2. Metadata service (separated from data nodes) — probe if they couple these
3. Data nodes (commodity storage, multiple AZs)
4. Chunk/erasure coding layer
5. Pre-signed URL mechanism

**Critical probe — bandwidth:** "Walk me through a 1 TB upload. Where do the bytes flow? Specifically, do they go through your app servers?"
- If yes: "How many app server instances would you need to handle 580 uploads/sec averaging 1 MB each? What's the aggregate bandwidth?"
- Expected follow-up: candidate should arrive at pre-signed URLs

**Critical probe — metadata separation:** "Why is metadata in a separate service and not on the data nodes?"
- Expected answer: different access patterns (random small reads vs sequential large), different hardware needs (SSD vs HDD), independent scaling

**Critical probe — erasure coding:** "You mentioned erasure coding. Walk me through what happens when a data node fails."
- Expected: 5+4 means 9 shards; any 5 reconstruct the original; node failure means 1 shard missing; reconstruction reads 5 remaining shards; writes the recovered shard to a new node

---

## Phase 4: Deep Dive (28–40 min)

Choose one based on where the candidate is weakest:

**Option A — Multipart Upload:**
"Walk me through the complete lifecycle of a 50 GB file upload. Specifically: what is an UploadId, when does the object become visible to readers, what happens if the client disconnects after uploading 30 GB, and who cleans up the orphaned parts?"

**Option B — Pre-Signed URL:**
"A mobile app needs to upload directly to your storage service without routing bytes through your app servers. Walk me through the pre-signed URL mechanism — how is it generated, what does it contain, how does the storage node validate it without contacting your auth service?"

**Option C — Consistency:**
"Your customer uploads an object. One millisecond later, a different client in a different availability zone reads that key. Is the read guaranteed to succeed? What tradeoffs did you make in your design to achieve that, and what did those tradeoffs cost?"

---

## Phase 5: Failure & Scaling (40–44 min)

**Failure probe:** "Your metadata database primary goes down. Walk me through what happens: what reads/writes fail, what degrades gracefully, what's your recovery path, and what's the SLA impact during failover?"

Expected: reads served from read replica (slightly stale but functional), writes queue or fail fast, automated primary promotion in <30 seconds, SLA impact is <30 seconds of write unavailability if designed correctly.

**Scaling probe:** "You've grown to 100 PB. Your metadata DB now has 200 billion rows. What breaks first?"
Expected: metadata DB sharding strategy needs revisiting, possibly migrating from MySQL to a distributed KV store for metadata. Candidate should identify this as the next bottleneck and propose a sharding scheme.

---

## Closing

"Let's wrap up. Two questions:
1. What's the weakest part of your design that you'd want to fix with more time?
2. If this were a real project, what would you prototype first to validate your assumptions?"

---

## Scoring Checklist

| Area | Weight | 1 (Poor) | 3 (OK) | 5 (Strong) |
|------|--------|----------|--------|------------|
| Requirements clarification | 10% | Jumped straight to design | Asked about scale and features | Asked about consistency, large files, cross-region |
| HLD completeness | 30% | Missing metadata or data tier | Both tiers present, pre-signed URL mentioned | Full separation, erasure coding motivated, pre-signed URL flow clear |
| LLD depth | 30% | No multipart, no GC, no chunk math | Multipart described, chunk size mentioned | Full multipart lifecycle, GC design, chunk size tradeoff quantified |
| Tradeoff reasoning | 20% | Choices made without justification | Some tradeoffs named | Erasure coding cost math, pre-signed URL vs proxy tradeoff, consistency model choice all quantified |
| Communication clarity | 10% | Rambling, unclear structure | Logical flow, occasional detours | Clear narrative, proactively addressed failure modes |
