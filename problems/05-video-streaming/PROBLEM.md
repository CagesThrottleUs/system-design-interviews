# Video Streaming (YouTube / Netflix)

**Difficulty:** Advanced
**Category:** CDN, Async processing, Storage-heavy
**Time Box:** 45 min
**Key Concepts:** Adaptive bitrate streaming (ABR), transcoding pipelines, CDN architecture, blob storage, resumable uploads, distributed async workers
**Asked at:** Google/YouTube (structured whiteboard), Netflix (team-specific, often based on real systems), Amazon (fits AWS Prime Video / IVS context), Meta (video delivery at scale), Apple (ICT3 level for media teams). Confirmed also at Productboard ("System design of a video streaming service such as YouTube" — Glassdoor).

---

## Problem Statement

Imagine you're on a train. Your LTE signal is strong as you leave the station — your 4K video plays flawlessly. Ten minutes later you pass through a tunnel and your connection drops to 2G. With a naive streaming system, the video freezes for thirty seconds while it buffers. You abandon it. With a well-designed system, the quality drops gracefully from 4K to 360p within a second, playback never pauses, and the moment you exit the tunnel the quality climbs back up. That behavior is not magic — it is the result of deliberate engineering decisions made years before you ever started that video.

Now flip the perspective to the creator. A filmmaker uploads a 2-hour 4K movie. They expect it to be available globally — to a viewer in Tokyo, São Paulo, and Lagos — within thirty minutes of upload. That movie arrives as a single massive file. Your system must chunk it, transcode it into over a hundred format variants (multiple resolutions, multiple bitrates, multiple audio tracks, multiple subtitle files), distribute those variants to thousands of CDN edge nodes worldwide, and do all of this asynchronously without blocking the upload response or losing a single byte if the transcoding worker crashes halfway through.

The engineering challenge at YouTube scale: 500 hours of video are uploaded every minute (YouTube official stat, Tubefilter, May 2019). 2.5 billion users consume content monthly (VdoCipher, 2024). Netflix, the clearest design analogue for VoD architecture, serves ~95% of its global streaming traffic through its own Open Connect CDN — not over the public internet backbone — with 19,000+ purpose-built appliances deployed directly inside ISP networks in 100+ countries (Netflix openconnect.netflix.com, 2025; APNIC blog, 2018). A single Open Connect Appliance can push 100 Gbps (Netflix TechBlog). That is what "serving video at scale" actually means.

---

## Actors

| Actor | Description | Primary Actions |
|-------|-------------|-----------------|
| **Creator** | Uploader — individual user, studio, or automated pipeline | Upload video, track processing status, update metadata |
| **Viewer** | End user watching content | Fetch manifest, stream chunks, seek, resume, rate |
| **CDN Edge Node** | Geographically distributed cache server | Serve video chunks, pull from origin on miss |
| **Transcoding Worker** | Async compute worker | Pull raw video from blob store, encode to target format, push renditions back |

---

## Functional Requirements

**Core (MVP):**
- FR1: Creators can upload a video file (up to 4 hours, up to 50 GB) and receive a video ID
- FR2: The system transcodes uploaded video into multiple resolutions (240p, 360p, 480p, 720p, 1080p, 4K) and bitrates asynchronously
- FR3: Viewers can play back a video with adaptive bitrate streaming — quality adjusts automatically to available bandwidth
- FR4: Creators can track transcoding progress (queued / processing / ready / failed)
- FR5: Viewers can resume playback from where they left off across devices

**Secondary:**
- FR6: Search and discovery — viewers can search by title, description, tags
- FR7: Recommendations — personalized video feed
- FR8: Creator analytics — view counts, watch time, retention curves
- FR9: Comments, likes, subscriptions

---

## Non-Functional Requirements

| Requirement | Target | Rationale |
|-------------|--------|-----------|
| Upload processing time | < 30 min for a 1-hour 1080p source | Creator expectation for timely publishing |
| Playback start latency | < 2 seconds (p95) | Nielsen research: users abandon video that doesn't start within 2 seconds |
| Rebuffering ratio | < 0.1% of watch time | Industry benchmark; Netflix targets < 0.1% (Netflix TechBlog) |
| Upload availability | 99.9% | Uploads can be retried; slightly lower SLA acceptable |
| Playback availability | 99.99% | Every viewer minute of downtime costs revenue and trust |
| Durability | 99.9999999% (11 nines) for raw and transcoded assets | Videos are irreplaceable creator content |
| CDN cache hit rate | > 95% | Cache misses route to origin; 5% miss rate at YouTube scale still means billions of origin hits |
| Geographic latency (p95) | < 50 ms to nearest CDN edge | Last-mile latency dominates; content must be pre-positioned |

---

## Capacity Estimation Hints

Use these assumptions to derive your own numbers before consulting the solution guide.

- **DAU:** 2 billion daily active users
- **Upload rate:** 500 hours of video uploaded per minute
- **Average source video size:** 300 MB per hour at 1080p (compressed upload)
- **Average watch session:** 30 minutes per user per day
- **Average streaming bitrate (adaptive, mixed quality):** 2.5 Mbps
- **Transcoding multiplier:** assume ~10 output renditions per source video, each ~60% the size of the source at equivalent duration

Work through:
1. Raw storage added per day from uploads
2. Total transcoded storage added per day (multiply by rendition count and compression ratios)
3. Egress bandwidth needed to serve 2B DAU × 30 min × 2.5 Mbps
4. How many CDN edge servers are needed if each serves 40 Gbps?

---

## Clarifying Questions to Ask the Interviewer

Ask these before drawing a single box. The answers change the architecture materially.

### Scope
1. **VoD or live streaming, or both?** Live requires a fundamentally different ingest pipeline — RTMP ingest, real-time segmentation, sub-10-second latency targets. VoD and live share CDN infrastructure but diverge everywhere upstream.
2. **Short-form (TikTok-style, < 3 min) or long-form (YouTube/Netflix-style, up to 4 hours)?** Short-form can be processed synchronously; long-form requires async distributed transcoding. Buffer strategy and CDN pre-positioning differ significantly.
3. **Are we building the full platform or focusing on a specific subsystem?** Scoping to the upload pipeline + playback pipeline is a complete 45-minute problem. Adding recommendations, search, and comments multiplies scope.

### Scale & Distribution
4. **What is the geographic distribution of users?** If 80% are in one region, CDN topology simplifies. Global-first changes CDN co-location strategy.
5. **What upload file size limits should we support?** 500 MB vs 50 GB changes chunking strategy and resumable upload requirements.
6. **Do we need to support 4K / HDR, or is 1080p the ceiling?** 4K requires 6–8× the bitrate of 1080p; this materially affects transcoding compute and storage budgets.

### Delivery Semantics
7. **What is the acceptable rebuffering target?** This drives CDN edge density and ABR algorithm aggressiveness.
8. **Do we need offline playback / download?** Requires DRM key management and local storage handling — a non-trivial addition.

### Operational
9. **What happens if a transcoding job fails halfway through a 2-hour video?** Do we restart from the beginning or resume from the last checkpoint? The answer determines whether you design idempotent chunk-level transcoding.
10. **Who pays for egress?** ISP co-location (Open Connect model) exists precisely because paying public cloud egress rates at Netflix/YouTube scale is economically untenable — knowing this signals senior-level CDN thinking.

---

## Key Components (hints)

<details>
<summary>Reveal components</summary>

**Upload Path:**
- **Upload Service** — handles chunked HTTP uploads, writes raw chunks to blob storage, enqueues transcoding job
- **Blob Storage** — raw video store (S3-equivalent); also stores all transcoded renditions
- **Transcoding Queue** — durable message queue (SQS / Kafka) holding per-video transcoding jobs
- **Transcoding Workers** — stateless compute fleet (can be GPU-accelerated); reads raw video, produces renditions, writes back to blob store
- **Metadata Service** — stores video metadata (title, creator, status, manifest locations); backed by a relational DB + cache

**Playback Path:**
- **API Gateway / Video API** — serves manifest requests, resolves video IDs to CDN manifest URLs
- **Manifest (Playlist) Files** — HLS `.m3u8` or DASH `.mpd` files listing available renditions and chunk URLs
- **CDN Edge Nodes** — geographically distributed; serve chunks with < 50 ms last-mile latency
- **ABR Player** — client-side logic that monitors download speed and selects the appropriate rendition in real time

**Supporting:**
- **Watch History Service** — stores per-user per-video resume position
- **Notification / Webhook Service** — notifies creator when transcoding completes
- **DRM / Encryption Layer** (advanced) — AES-128 or Widevine for premium content protection

</details>
