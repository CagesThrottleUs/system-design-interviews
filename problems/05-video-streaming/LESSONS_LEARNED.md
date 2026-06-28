# Lessons Learned — Video Streaming (YouTube / Netflix)

---

## Interview Stories

### Story 1 — Netflix Interview: "We'll use S3 for everything"

A candidate interviewing for a senior backend role on Netflix's streaming team drew a clean architecture: S3 for raw video, S3 for transcoded video, CloudFront CDN on top. The interviewer let it run for ten minutes, then asked: "Netflix serves around 95% of its traffic from its own CDN. Are you familiar with Open Connect?"

The candidate wasn't. They knew S3 and CloudFront — AWS-native thinking. They didn't know that at Netflix's scale (~95% of global streaming traffic, 280 million subscribers across 190 countries as of 2024), paying public cloud egress rates is economically untenable. Netflix solved this by building Open Connect: purpose-built appliances shipped directly to ISP data centers. 19,000+ appliances at 1,500+ ISP locations in 100+ countries. A single appliance delivers 100 Gbps. The ISP gets free equipment that reduces their peering load; Netflix gets near-zero egress cost for the bulk of their traffic.

The candidate's design was architecturally sound for a startup. But for a Netflix-scale interview, not knowing the CDN cost model is the equivalent of answering a database question without knowing B-tree indexes exist. The interviewer is evaluating whether you understand the real constraints that drive real architectural decisions — not just whether you can draw boxes.

**Pattern:** Candidates with AWS-first thinking map every problem to AWS services. This works at startup scale. At hyperscaler scale, the economics of managed services break down, and purpose-built infrastructure emerges. Know both worlds.

**Correct answer:** "At YouTube/Netflix scale, I'd use ISP co-location CDN — either Google's own CDN (which powers YouTube) or the Open Connect model — to eliminate egress costs. For a startup, CloudFront or Fastly is the right call. The difference is the cost model at 1+ PB/day egress."

---

### Story 2 — Google/YouTube Interview: The 2G Question

A candidate was thirty minutes into a strong Google interview. Clean boxes: upload service, transcoding workers, CDN, metadata DB. The interviewer nodded through the whole thing. Then: "A user is on 2G in rural India. Their connection is 128 Kbps. They press play on a 1080p video. What happens in your system?"

Silence. The candidate had designed the CDN. They had designed the transcoding pipeline to produce multiple resolutions. But they had never connected the dots to explain *how* the player selects the right resolution in real time. They said: "The CDN will serve the lower resolution." The interviewer pressed: "How does the CDN know to do that?"

The candidate didn't know about adaptive bitrate streaming. They didn't know about HLS manifests listing multiple renditions. They didn't know about the client-side ABR algorithm that measures bandwidth per segment and selects the appropriate rendition. Their system produced multiple resolutions but had no mechanism to deliver the right one to a given user.

This is the single most common failure mode in video streaming interviews. Transcoding multiple resolutions is table stakes. Explaining how the player uses those resolutions adaptively — that is the answer that separates L4 from L5.

**The answer that should have been given:**

"The player downloads the master manifest first — a small text file listing all renditions: 240p at 400 Kbps, 360p at 800 Kbps, 720p at 2.5 Mbps, 1080p at 5 Mbps. The player starts with the lowest rendition to ensure immediate playback start. After the first 4-second segment downloads, it measures the actual download speed: if it received 200 KB in 4 seconds that's 400 Kbps. The ABR algorithm compares 400 Kbps to the rendition ladder and selects 240p. The player continues measuring each segment. If bandwidth improves, it steps up. If it drops, it steps down — switching happens at segment boundaries, so there is never a mid-playback disruption. DASH reduces rebuffering by ~30% on mobile vs fixed-bitrate streaming (VdoCipher, 2024)."

**Pattern:** Candidates correctly design the server-side (transcoding multiple resolutions) but fail to explain the client-side (how the player uses those renditions). The ABR algorithm and HLS/DASH manifest format are non-negotiable topics at senior level.

---

### Story 3 — Amazon (Prime Video Context): Synchronous Transcoding Disaster

A candidate was designing a video platform for an Amazon interview. They proposed: "After the upload completes, the backend transcodes the video and then returns the video URL to the user." The interviewer asked: "How long does transcoding take for a 2-hour movie?"

The candidate didn't know. The interviewer helped: "Let's say 20 minutes. Walk me through the HTTP request lifecycle."

The candidate realized: the upload HTTP request would stay open for 20 minutes. The upload service thread would be blocked for 20 minutes. If the user's connection dropped during that time, the entire process would have to restart. The candidate tried to recover: "We could use a webhook callback." The interviewer asked: "Where does the transcoding happen?" The candidate said: "In the upload service." The interviewer: "So the upload service is transcoding 1,000 concurrent uploads simultaneously?"

The candidate's design was a single-process system doing synchronous CPU-bound work inside a network-facing service. At 500 hours of video uploaded per minute, this design collapses immediately.

**The correct design:**

The upload service does exactly three things: accept chunks, write to blob storage, publish a job to the transcoding queue. It responds in milliseconds with `{ "video_id": "...", "status": "processing" }`. Transcoding workers are a separate, horizontally scalable fleet that consume from the queue. Workers are stateless — each processes one job, writes output, marks the job done, pulls the next. If a worker crashes, the message re-appears in the queue after the visibility timeout and another worker picks it up. This is the async task queue pattern: isolate CPU-heavy work from network-facing services.

**Pattern:** Candidates conflate the upload service with the processing service. The network-facing service (upload) must respond fast. CPU-heavy work (transcoding) must be decoupled into async workers with a durable queue between them.

---

## General Pattern Mistakes

### Mistake 1 — Not scoping the problem

"Design YouTube" is not a well-defined problem. A candidate who jumps straight into drawing components without scoping VoD vs live, short-form vs long-form, upload focus vs playback focus, will design either the wrong system or an impossibly large one. Strong candidates spend 3-5 minutes asking clarifying questions, then state explicitly: "I'll focus on the VoD upload pipeline and adaptive playback path. I'll defer live streaming, recommendations, and search." The interviewer expects and respects this scoping. Skipping it signals poor engineering judgment.

### Mistake 2 — No napkin math

A candidate who can't estimate "how much storage does 500 hours/min upload add per day?" cannot be trusted to make capacity decisions. The math is not hard — it's 500 × 300 MB × 60 min × 24 hr ≈ 216 TB/day raw before transcoding multiplier. Knowing that total transcoded storage adds ~1 PB/day contextualizes every design decision about object storage tiering, CDN cache sizing, and transcoding compute. Candidates who skip math are guessing at architecture.

### Mistake 3 — Treating CDN as a black box

"Just put a CDN in front of it" with no further explanation is a junior answer. A senior candidate explains: (a) pull vs push CDN strategy, (b) when you pre-position content vs rely on pull-on-miss, (c) at scale, why ISP co-location eliminates egress costs, (d) how CDN cache TTLs affect manifest staleness. The CDN is the most performance-critical component in video streaming. Treating it as a commodity abstraction misses half the architecture.

### Mistake 4 — Forgetting failure modes

Transcoding failures are not edge cases — they are expected events at 500 hours/min upload rate. A candidate who doesn't design for: "what happens when a worker crashes mid-encode?", "what happens when encoding fails for 4K but succeeds for 1080p?", "what happens when the creator's upload connection drops at 60%?" — has not finished designing the system. Failure handling is not extra credit at senior level; it is the baseline.

### Mistake 5 — Single-rendition thinking

Designing a system that produces one output per upload ("we transcode to 1080p") and calling it done is an L3 answer. L5 requires: (a) multiple renditions per video (10+), (b) master manifest listing all renditions, (c) client-side ABR algorithm, (d) segment-level granularity (2–10s chunks), (e) per-rendition failure handling. Netflix encodes ~120 streams per title across resolutions, bitrates, audio tracks, and subtitle formats (techinterview.org, 2024). Knowing the scale of what "transcoding" actually means changes the design.

---

## Self-Assessment Checklist

Before leaving this problem, verify you can answer all of these without looking at the solution:

**Upload path:**
- [ ] Can you explain why transcoding must be async, not synchronous?
- [ ] Can you describe the resumable upload protocol (initiate → chunk → query offset → resume)?
- [ ] Can you explain what the transcoding worker does step by step, from raw blob to final manifest?
- [ ] What happens when a transcoding worker crashes mid-encode?
- [ ] What is the failure granularity — per video or per rendition?

**Playback path:**
- [ ] Can you explain what a master HLS manifest contains?
- [ ] Can you explain the ABR algorithm at the player level (the bandwidth-measurement loop)?
- [ ] What is the difference between HLS and DASH? When would you choose each?
- [ ] What is the difference between CDN pull and CDN push? When would you use each?

**Scale:**
- [ ] Can you derive storage added per day given upload rate and average file size?
- [ ] Can you derive egress bandwidth needed given DAU and average watch time?
- [ ] Can you explain why ISP co-location CDN emerges at hyperscaler scale?

**Advanced:**
- [ ] What is per-title encoding? What problem does it solve? What was Netflix's claimed improvement?
- [ ] How does Netflix achieve ~95% traffic via Open Connect? What is the ISP incentive?
- [ ] Can you name the capacity of a single Open Connect Appliance?

---

## Remediation Targets

If you missed questions in the self-assessment, study these specific areas:

| Gap | Study Target |
|-----|-------------|
| ABR algorithm | HLS RFC 8216; DASH-IF IOP guidelines; Akamai blog on ABR algorithms |
| Resumable uploads | Google Resumable Upload API spec; S3 multipart upload docs |
| Transcoding pipeline | FFmpeg documentation; Netflix TechBlog "Per-Title Encode Optimization" |
| CDN ISP co-location | Netflix openconnect.netflix.com; APNIC Blog "Netflix and the CDN edge" (2018) |
| Per-title encoding | Netflix TechBlog "Per-Title Encode Optimization" (Dec 2015); "VMAF: The Journey Continues" (2016) |
| Capacity estimation | Work through the math in SOLUTION_GUIDE.md without looking at the answers |
