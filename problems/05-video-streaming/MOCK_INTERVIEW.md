# Mock Interview Script — Video Streaming (YouTube / Netflix)

**Interviewer:** Senior Staff Engineer / Engineering Manager
**Target Level:** Apple ICT3 (Senior Engineer equivalent)
**Time Box:** 45 minutes
**Format:** Whiteboard / virtual whiteboard

---

## Interviewer Mindset

This interview probes three things that separate L5 from L4: (1) systems intuition — do you know how video streaming actually works in production?, (2) failure thinking — can you design for the error cases, not just the happy path?, and (3) scale reasoning — can you do the math and let it inform architecture?

Do not help the candidate. Do not hint. If they go down the wrong path, let them. If they ask for a hint, note it and give only a minimal nudge. The goal is an accurate signal on where their knowledge actually ends.

---

## Phase 0 — Opening (1 min)

**Say exactly this:**

> "Let's design a video streaming platform — think YouTube or Netflix. You have 45 minutes. I'm looking for a high-level system design and then we'll go deep on one component together. Where do you want to start?"

Then stop talking. Let them lead.

**Watch for:**
- Do they ask clarifying questions, or do they start drawing immediately?
- Starting to draw without any scoping is a yellow flag.

---

## Phase 1 — Requirements Clarification (target: 5 min, cut at 8 min)

Strong candidates will ask several of these. If they ask none after 60 seconds of silent thinking, prompt:

> "Before you start drawing, what would you want to know about the requirements?"

### Questions you should hear (the more the better):

| Question | Why it matters |
|----------|---------------|
| "VoD or live streaming?" | Live requires RTMP ingest, < 10s latency target, different CDN strategy |
| "Short-form or long-form?" | Affects transcoding approach, buffer strategy |
| "What's the upload size limit?" | Drives resumable upload requirement |
| "4K or 1080p ceiling?" | 4K = 6-8× bitrate → storage and compute cost change |
| "Geographic distribution?" | Informs CDN PoP density |
| "What's our rebuffering target?" | Drives ABR aggressiveness and CDN edge density |
| "Offline download needed?" | Adds DRM layer — non-trivial |
| "What does 'design YouTube' mean — full platform or specific subsystem?" | Scoping is a senior skill |

**Intervene if they try to go full platform scope:** "Let's focus on the upload pipeline and playback path. Recommendations and search are out of scope for today."

**Your answers to give when asked:**
- VoD only (no live streaming)
- Long-form (up to 4 hours, up to 50 GB)
- Global distribution
- 4K supported
- Rebuffering < 0.1% of watch time
- No offline download required

---

## Phase 2 — High-Level Design (target: 15-18 min)

Let the candidate draw. Intervene with these probes after they finish a component:

### After they draw the upload component:

> "Walk me through what happens when a creator hits 'upload' on a 2-hour 4K movie. Step by step."

This is **the critical probe of this entire interview.** It is the moment that separates L4 from L5. See scoring section for what each response quality implies.

**Listen for:**
- Do they mention chunked upload? (Required)
- Do they say the upload service transcodes synchronously? (Fail)
- Do they mention blob storage for raw video? (Required)
- Do they mention enqueueing a transcoding job? (Required)
- Do they say the upload response is immediate with `status: processing`? (Strong)
- Do they mention multiple renditions being queued separately? (Strong)

**If they design synchronous transcoding, challenge immediately:**

> "Transcoding a 2-hour 4K movie — how long does that take?"

Wait for their answer. Then:

> "So the upload HTTP connection stays open for that entire time? What happens if the creator's connection drops at minute 15?"

Let them redesign. If they can't reach async independently:

> "What pattern do we use in distributed systems when we have CPU-heavy work that doesn't need to block the user response?" [Note: hint given, affects score]

### After they draw the CDN:

> "How does the video actually get to a viewer in São Paulo? Walk me through the CDN architecture."

**Listen for:**
- Do they mention pull vs push CDN?
- Do they mention ISP co-location (even roughly — "we'd want to put content inside ISP networks")?
- Do they know that Netflix serves ~95% of traffic via Open Connect (not AWS)?
- Do they explain how a CDN cache miss resolves?

**If they say "we'll use CloudFront or S3 directly":**

> "At Netflix scale — 280 million subscribers, 1+ PB of egress per day — what does it cost to pay AWS egress rates for all of that?"

Let them think. Then:

> "Netflix has a public program called Open Connect. Are you familiar with it?"

Note whether they knew it independently vs needed to be told.

### After they draw the playback component:

> "A user in rural India has a 128 Kbps 2G connection. They press play on a 4K video in your system. What happens?"

**Listen for:**
- Do they mention ABR / adaptive bitrate streaming?
- Do they know what a HLS master manifest contains (multiple renditions with bandwidth declarations)?
- Can they explain the client-side ABR algorithm — bandwidth measurement per segment, rendition selection?
- Do they know segment duration (2–10 seconds per chunk)?
- Do they mention HLS vs DASH by name?

**If they say "the CDN will serve the right resolution":**

> "How does the CDN know which resolution to serve? Where does that decision get made?"

The correct answer: the client (player) makes the decision, not the CDN. The CDN just serves whatever segment the player requests.

**If they have no answer on adaptive bitrate:**

> "Have you heard of HLS or DASH?" [Note: hint given, affects score]

---

## Phase 3 — Deep Dive: Transcoding Pipeline (target: 12-15 min)

Transition with:

> "Let's go deeper on the transcoding pipeline. I want to understand how your system handles production-scale failure."

### Probe 1 — Failure recovery:

> "A transcoding worker is 40% through encoding the 4K rendition of a 2-hour movie and crashes. Walk me through exactly what happens in your system."

**Listen for:**
- Visibility timeout on the queue message (SQS / Kafka re-delivery)
- A different worker picks up the job
- Idempotent write (re-running overwrites partial output safely)
- Per-rendition failure granularity (only the 4K job retries, not all 10 renditions)

**If they say "we restart from the beginning":**

> "The source file is 50 GB. Streaming that from blob storage twice for a re-run — is that a problem?"

Accept "the idempotent write is the same cost regardless" — that's correct.

### Probe 2 — Partial failure:

> "4K encoding fails after 3 attempts — a codec bug in your FFmpeg build. But 1080p, 720p, and all lower renditions succeeded. What does your system do?"

**Listen for:**
- Per-rendition status in the jobs table
- Video marked `partially_ready` with available renditions listed
- Player falls back to highest available rendition (1080p)
- Creator notified of partial failure
- Dead-letter queue for manual inspection of failed jobs

**If they say "the whole video is unavailable":**

> "Why would you block a user from watching the 1080p version because 4K encoding failed?"

### Probe 3 — Scale:

> "How many transcoding workers do you need to keep up with 500 hours of video uploaded per minute?"

Walk through the math with them:

> "Let's say encoding at 1080p runs at 5× realtime on a modern CPU. 500 hours of video per minute, 10 renditions each. What's your calculation?"

**Expected calculation:**
- 500 hours/min × 10 renditions = 5,000 encoding-hours per minute of real time
- At 5× realtime: 5,000 / 5 = 1,000 server-equivalents concurrently
- Peak factor: maybe 1,500–2,000 during upload bursts

**Listen for:** Can they do this math? Can they propose auto-scaling from a baseline with burst capacity?

### Probe 4 (advanced, only if time allows):

> "Netflix uses something called per-title encoding. Have you heard of it, and how would you implement it in your transcoding pipeline?"

**What to listen for:**
- Know that per-title encoding analyzes each video's visual complexity to find the optimal bitrate per rendition (not a fixed ladder)
- Netflix debuted this in December 2015 (PSNR metric), switched to VMAF in 2016
- Result: ~20% average bitrate reduction across the catalog
- Implementation: a preprocessing step before encoding that samples frames, measures complexity, and outputs a custom bitrate ladder for that specific video

---

## Phase 4 — Wrap-up (target: 3-5 min)

> "What would you do differently if you had 3 more months to build this?"

> "What is the weakest part of your design right now?"

> "If this system goes down at 2 AM, what's your runbook?"

---

## Scoring Checklist

### Requirements Clarification (10%)
- [ ] Asked VoD vs live (2 pts)
- [ ] Asked about upload size limit or long-form vs short-form (2 pts)
- [ ] Asked about geographic distribution (2 pts)
- [ ] Asked about rebuffering target or latency SLA (2 pts)
- [ ] Scoped the problem explicitly before drawing (2 pts)

**Score: /10**

### High-Level Design (30%)
- [ ] Upload path: chunked upload, blob storage for raw, async queue, immediate response (6 pts)
- [ ] Transcoding: async workers, separate fleet, not inside upload service (6 pts)
- [ ] Multiple renditions produced (not single output) (4 pts)
- [ ] CDN: explained pull vs push, or ISP co-location model (6 pts)
- [ ] Playback: manifest → ABR → chunk request chain explained (8 pts)

**Score: /30**

### Low-Level Design (30%)
- [ ] Transcoding failure: visibility timeout re-enqueues, different worker retries (6 pts)
- [ ] Idempotent writes: re-running encoding is safe (4 pts)
- [ ] Per-rendition failure granularity (not per-video) (4 pts)
- [ ] ABR algorithm explained at player level (bandwidth measurement, segment-level switching) (8 pts)
- [ ] Resumable upload protocol (initiate, chunk, query offset, resume) (4 pts)
- [ ] Data model: videos table with status, transcoding_jobs table per rendition (4 pts)

**Score: /30**

### Tradeoff Reasoning (20%)
- [ ] HLS vs DASH: named both, explained difference, chose one with rationale (5 pts)
- [ ] CDN push vs pull: when to use each (5 pts)
- [ ] Sync vs async transcoding: rejected sync with explicit reasoning (5 pts)
- [ ] Per-title encoding mentioned (even briefly) as an optimization (5 pts)

**Score: /20**

### Communication Clarity (10%)
- [ ] Clear structure: uploaded → transcoded → distributed → played (3 pts)
- [ ] No dead ends — recovered from mistakes proactively (3 pts)
- [ ] Drove toward constraints before solutions (4 pts)

**Score: /10**

---

## Scoring Scale

| Total | Interpretation |
|-------|---------------|
| 85–100 | Strong hire. Deep systems knowledge, failure-first thinking, knows production CDN model. |
| 70–84 | Lean hire. Good fundamentals, missed one or two key concepts (usually ISP CDN or ABR algorithm). |
| 55–69 | Borderline. Solid on happy path, weak on failure modes and scale reasoning. |
| 40–54 | No hire at ICT3. Core gaps: missing ABR, missing async transcoding, or both. |
| < 40 | Strong no hire. Fundamental misunderstanding of async processing or adaptive streaming. |

---

## The Critical Probe (Expanded)

**"A user uploads a 2-hour 4K movie. Walk me through what happens in your system, step by step."**

This single probe reveals more than all other questions combined. Here is the rubric:

| Response | Signal | Score |
|----------|--------|-------|
| "The upload service receives the file, transcodes it, and returns a URL." | Does not know async processing is required. L3 signal. | 0/6 on upload path |
| "The upload service receives the file in chunks, stores it to blob storage, and then starts transcoding asynchronously." | Knows async is needed; doesn't detail the queue mechanism or multiple renditions. L4 lower. | 3/6 |
| "The upload service receives chunks via resumable HTTP, writes to blob storage, publishes a job per rendition to a transcoding queue, and returns `status: processing` immediately. Workers consume from the queue — one per rendition — encode using FFmpeg, write segments back to blob, and generate HLS manifests. When all renditions complete, the video status flips to ready." | Knows the full pipeline, multiple renditions, manifest generation. L5 signal. | 6/6 |
| Above + failure recovery (visibility timeout, idempotent writes, DLQ) + CDN pre-positioning after completion | Exceptional. L5+ signal. | 6/6 + bonus |

If the candidate hits the L5 answer on this probe without any hints, the rest of the interview is confirmation. If they stumble here, the rest of the interview is remediation diagnosis.
