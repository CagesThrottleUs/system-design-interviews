# Solution Guide — Object Storage (S3-like)

Read after your attempt. If you haven't attempted yet, close this file.

---

## Component Map

| Component | Role | Technology Choice | Why |
|-----------|------|-------------------|-----|
| API Service | REST endpoints (PUT/GET/DELETE/presign) | Stateless Go/Java servers | Horizontal scale; no session state; throughput matters |
| Metadata Service | `bucket/key → chunk locations, checksum, size` | MySQL/PostgreSQL (sharded) | Structured, relational; transactions matter for versioning |
| Data Nodes | Store raw chunk bytes | Custom block device or ext4 on commodity HDDs | Cheap, high-density; erasure coding handles failures |
| Placement Service | Decides which data nodes receive new chunks | In-memory placement algorithm | Rack-aware, load-balanced distribution |
| Erasure Coding Engine | Encodes chunks into data+parity shards | Reed-Solomon (5+4 or 6+3 scheme) | 1.8× overhead vs 3× for replication, far cheaper at scale |
| Replication Monitor | Detects under-replicated chunks, triggers repair | Background daemon with heartbeat | Survives node and rack failures within durability SLA |
| Pre-Signed URL Service | Signs time-limited access tokens | HMAC-SHA256 over canonical request | Clients upload/download directly; app server never sees bytes |
| Garbage Collector | Reclaims bytes from soft-deleted objects | Background job, tombstone TTL | Safe deletion after propagation; prevents use-after-delete |

---

## Architecture Diagram

```
WRITE PATH (upload)
───────────────────────────────────────────────────────────────

  Client
    │  PUT /bucket/key (large file)
    ▼
┌────────────────┐
│   API Service  │
│  (stateless)   │
└────────────────┘
    │                               │
    │  1. Write metadata record     │  2. Get placement (which nodes)
    ▼                               ▼
┌────────────────┐         ┌─────────────────┐
│ Metadata DB    │         │ Placement Service│
│ (MySQL sharded)│         │ (in-memory)      │
└────────────────┘         └─────────────────┘
                                    │
                 3. Stream chunks in parallel
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
               ┌────────┐     ┌────────┐     ┌────────┐
               │Data    │     │Data    │     │Data    │
               │Node A  │     │Node B  │     │Node C  │
               │(AZ-1)  │     │(AZ-1)  │     │(AZ-2)  │
               └────────┘     └────────┘     └────────┘
                (shard 0)     (shard 1)      (shard 2)
                            + parity shards on Nodes D, E (AZ-3)


READ PATH (download)
───────────────────────────────────────────────────────────────

  Client
    │  GET /bucket/key
    ▼
┌────────────────┐
│   API Service  │  ──▶  Metadata DB  ──▶  chunk locations
└────────────────┘
    │
    │  Fetch chunks in parallel from data nodes
    │  (or redirect to pre-signed data node URL)
    ▼
  Reassemble → stream to client


PRE-SIGNED URL FLOW (direct upload, bypass app servers)
───────────────────────────────────────────────────────────────

  Client App                API Service              S3 / Data Nodes
    │  POST /presign             │                         │
    │ ─────────────────────────▶│                         │
    │                           │  Generate HMAC token    │
    │  Returns presigned URL    │                         │
    │ ◀─────────────────────────│                         │
    │                           │                         │
    │  PUT directly with token  │                         │
    │ ────────────────────────────────────────────────────▶
    │                           │   App never sees bytes  │
```

---

## Capacity Math

**Object count and storage:**
- 10 billion objects × 1 MB average = **10 PB raw storage**
- With 5+4 erasure coding (1.8× overhead): 10 PB × 1.8 = **18 PB physical storage**
- With naive 3× replication: 10 PB × 3 = **30 PB physical storage**
- Savings with erasure coding at this scale: **12 PB** — at ~$20/TB/year = **$240M/year saved**
- This is why AWS, Dropbox, and Google all use erasure coding, not replication, for cold/warm storage

**Write QPS:**
- 10M DAU × 5 uploads/day = 50 million uploads/day
- 50M ÷ 86,400 = **~580 writes/sec average**
- Peak (2× average): **~1,200 writes/sec**

**Read QPS:**
- Read:write = 10:1 → **~5,800 reads/sec average**
- Peak: **~12,000 reads/sec**

**Metadata database size:**
- Per object: bucket (64B) + key (128B) + locations (3× node_id = 24B) + checksum (32B) + size (8B) + content-type (32B) + timestamps (16B) ≈ **~500 bytes/object**
- 10 billion objects × 500 B = **5 TB of metadata**
- Fits in a sharded MySQL cluster; 10 shards × 500 GB each is very manageable

**Chunk strategy:**
- Recommended chunk size: **64 MB** (S3 minimum part is 5 MB; optimal is 16–64 MB)
- Too small (1 MB): 5 TB file = 5 million metadata entries — metadata overhead dominates
- Too large (1 GB): a single chunk failure during upload = retry entire gigabyte
- At 64 MB: 5 TB file = ~80,000 chunks — reasonable metadata overhead, retries are cheap

---

## API Design

**Upload (small object, <5 MB):**
```
PUT /v1/{bucket}/{key}
Headers: Content-Type, Content-Length, X-Checksum-SHA256
Body: object bytes
Response 200: { "etag": "sha256:abc123...", "size": 4096 }
```

**Multipart upload (large objects):**
```
# Step 1: Initiate
POST /v1/{bucket}/{key}?uploads
Response 200: { "uploadId": "u_xyz789" }

# Step 2: Upload part (parallelizable)
PUT /v1/{bucket}/{key}?partNumber=3&uploadId=u_xyz789
Body: part bytes (5 MB–5 GB each)
Response 200: { "etag": "sha256:part_etag_3" }

# Step 3: Complete
POST /v1/{bucket}/{key}?uploadId=u_xyz789
Body: { "parts": [{"partNumber": 1, "etag": "..."}, ...] }
Response 200: { "key": "...", "etag": "...", "size": ... }

# Abort
DELETE /v1/{bucket}/{key}?uploadId=u_xyz789
```

**Generate pre-signed URL:**
```
POST /v1/presign
Body: { "bucket": "...", "key": "...", "method": "PUT", "expiresIn": 3600 }
Response 200: { "url": "https://...", "expiresAt": "2025-01-01T12:00:00Z" }
```

**Download:**
```
GET /v1/{bucket}/{key}
Optional headers: Range: bytes=0-1048575  (range request support)
Response 200: object bytes, streaming
```

---

## Data Model

**objects table (Metadata DB — sharded by bucket_id):**
```sql
CREATE TABLE objects (
  bucket_id     BIGINT NOT NULL,
  object_key    VARCHAR(1024) NOT NULL,
  version_id    VARCHAR(32) NOT NULL DEFAULT 'latest',
  size_bytes    BIGINT NOT NULL,
  checksum      CHAR(64) NOT NULL,           -- SHA-256 hex
  content_type  VARCHAR(128),
  storage_class VARCHAR(16) DEFAULT 'STANDARD',
  created_at    TIMESTAMP NOT NULL,
  deleted_at    TIMESTAMP,                    -- soft delete
  PRIMARY KEY (bucket_id, object_key, version_id)
);
```

**chunks table (maps object → physical storage):**
```sql
CREATE TABLE chunks (
  chunk_id      BIGINT PRIMARY KEY AUTO_INCREMENT,
  bucket_id     BIGINT NOT NULL,
  object_key    VARCHAR(1024) NOT NULL,
  version_id    VARCHAR(32) NOT NULL,
  part_number   INT NOT NULL,
  chunk_hash    CHAR(64) NOT NULL,
  size_bytes    INT NOT NULL,
  node_ids      JSON NOT NULL,               -- ["node_a", "node_b", "node_c"]
  INDEX idx_object (bucket_id, object_key, version_id)
);
```

**DB type reasoning:** Relational DB (MySQL/PostgreSQL) for metadata because:
- Transactions matter: marking all chunks of a deleted object must be atomic
- Range queries needed: list all objects in a bucket with prefix
- Schema is well-defined and stable
- At 5 TB of metadata, sharded MySQL is proven at this scale (Dropbox runs sharded MySQL)
- NOT a key-value store: you need range scans, prefix filtering, and joins between objects/chunks

---

## Key Design Decisions

### Decision 1: Erasure Coding vs Replication

| Dimension | Erasure Coding (5+4) | 3× Replication |
|-----------|---------------------|----------------|
| **Storage overhead** | 1.8× | 3× |
| **Write cost** | Must compute+write 9 shards (more CPU) | Write 3 copies (simpler) |
| **Read cost** | Can reconstruct from any 5 of 9 shards | Read from any replica (simpler) |
| **Fault tolerance** | Tolerates 4 simultaneous node failures | Tolerates 2 simultaneous failures |
| **Rebuild I/O on failure** | Must read 5 shards to rebuild 1 lost shard | Read 1 replica to rebuild |

**Choice: Erasure coding for cold/warm storage.** At petabyte scale, halving storage costs saves hundreds of millions of dollars annually. AWS S3, Dropbox Magic Pocket, and Google GCS all use erasure coding. Use replication only for hot/small objects where rebuild latency matters.

**Trade-off accepted:** Higher write CPU (encoding) and higher read I/O on failures (must contact 5 nodes instead of 1). Acceptable because writes are less frequent than reads, and node failures are rare.

---

### Decision 2: Metadata Service Separated from Data Nodes

| Dimension | Unified (metadata + data together) | Separated services |
|-----------|------------------------------------|--------------------|
| **Scale independently** | No — metadata and data scale together | Yes — metadata fits in MySQL; data needs cheap HDDs |
| **Latency** | Combined lookup is one call | Two calls: metadata lookup + data fetch |
| **Failure isolation** | Metadata DB down = all reads fail | Metadata down ≠ data node down |
| **Cost** | Metadata on fast SSDs wastes money | Metadata on SSDs, data on cheap HDDs |

**Choice: Separated.** The access patterns are completely different: metadata is small (500 bytes/object), random-access, needs SQL queries. Data is large, sequential, needs bulk throughput. Coupling them would require expensive SSDs for petabytes of data. Dropbox's Magic Pocket and S3's internal architecture both separate these layers.

**Trade-off accepted:** Every read requires two round trips: metadata lookup + data fetch. Mitigated by caching hot metadata in Redis.

---

### Decision 3: Pre-Signed URLs vs Server-Side Proxy

| Dimension | Pre-Signed URL (direct client → storage) | Server-Side Proxy |
|-----------|------------------------------------------|-------------------|
| **Bandwidth cost** | App server sees zero bytes | App server proxies all bytes |
| **Scalability** | App servers scale independently of data | App servers must scale with data volume |
| **Security** | Time-limited, scoped, HMAC-signed | Full control, no expiry complexity |
| **Implementation** | Client needs to handle S3 PUT directly | Simpler client code |

**Choice: Pre-signed URLs for large uploads/downloads.** At 580 writes/sec × average 1 MB = **580 MB/sec** flowing through app servers without pre-signed URLs. At 10 GB/s peak, app servers become the bottleneck. Pre-signed URLs eliminate this completely. AWS S3, GCS, and Azure Blob all use this pattern.

**Trade-off accepted:** Client SDK complexity. Clients must handle multipart upload themselves (or use an SDK that abstracts it). The SDK burden is worth the bandwidth elimination.

---

## Deep Dive: Multipart Upload for Large Objects

Multipart upload solves three problems simultaneously: bandwidth saturation, failure recovery, and parallelism.

**The problem without multipart:** A 50 GB backup upload takes ~50 seconds on a 8 Gbps connection. If the connection drops at second 49, you restart from zero. The entire upload must be serialized through one TCP stream.

**How multipart solves it:**

1. **Initiate** → server creates an UploadId and reserves metadata entry
2. **Upload parts** (parallelizable) → each part (e.g., 64 MB) is a separate HTTP PUT. Parts may arrive out of order. Each returns an ETag (checksum of that part). Parts are stored as independent chunks on data nodes.
3. **Complete** → client sends manifest of all PartNumber+ETag pairs. Server validates checksums, assembles the final object metadata record, and makes the object visible to readers. Until Complete, the object does not exist from readers' perspective.
4. **Abort** → deletes all uploaded parts. Garbage collector runs periodically to clean up orphaned parts from abandoned uploads.

**S3 real limits:** Min part: 5 MB (except last). Max parts: 10,000. Max object: 5 TB. These constraints are worth knowing for interviews.

**Parallel upload strategy:** A client with 10 Gbps bandwidth uploading a 1 TB object with 64 MB parts = ~15,625 parts. With 16 parallel streams: 15,625 / 16 ≈ 977 serial batches. At 64 MB × 10 Gbps, each part takes ~51 ms. Total upload time: ≈ 50 seconds vs 825 seconds single-stream. **16× speedup.**

**Failure recovery:** If part 500 fails, only part 500 needs retry. Not all 15,625 parts.

---

## Failure Modes & Mitigations

| Failure | Impact | Detection | Mitigation |
|---------|--------|-----------|------------|
| Data node disk failure | Chunks on that disk unreadable | Heartbeat + checksum mismatch | Erasure coding: reconstruct from 5 of remaining 8 shards. Replication monitor triggers background repair. |
| Data node server crash | All chunks on that node unavailable | Heartbeat timeout (30 sec) | Erasure coded objects survive. Monitor marks node failed, triggers copy of all under-replicated chunks to healthy nodes. |
| Metadata DB primary failure | Writes stall; reads served by replica | Replication lag spike | Replica promotion (automated, <30 sec). During failover: reads from replica (slightly stale), writes queued. |
| Partial write (client disconnect mid-upload) | Object in inconsistent state | UploadId with no Complete after TTL | Garbage collector deletes orphaned multipart uploads after 24 hours. Object never exposed to readers. |
| Network partition between AZs | Cross-AZ chunks unavailable | Erasure coding placement tracks AZ diversity | With 5+4 across 3 AZs, losing one AZ loses at most 3 of 9 shards. Object still reconstructable from 5 remaining. |
| Checksum mismatch on read | Silent data corruption | Per-chunk SHA-256 on every read | Detect corruption immediately. Reconstruct from other shards. Alert ops. Never serve corrupt data. |
| Hot object thundering herd | Data nodes overloaded | Request rate per key | CDN cache for hot objects. Pre-signed URLs spread reads directly to data nodes, bypassing app server layer. |

---

## What Strong Candidates Do

1. **Separate metadata from data from the start** — and justify it with different access patterns and different hardware requirements (SSD for metadata, cheap HDD for data)
2. **Motivate erasure coding with cost math** — not just "it's more durable." The cost delta at petabyte scale is the actual reason companies implement it
3. **Design the chunk size decision** — explain that 64 MB is the sweet spot between metadata overhead (too small) and retry cost (too large)
4. **Show the pre-signed URL flow** — know that app servers routing upload bytes is a fatal scaling mistake
5. **Address the multipart state machine** — UploadId, parts, Complete/Abort, garbage collection of orphaned uploads
6. **Call out strong consistency** — mention that S3 only became strongly consistent in December 2020; explain why it matters (stale metadata returns wrong chunk locations)
7. **Name the replication monitor** — someone has to detect under-replicated chunks and trigger repair. This is often the missing component.

---

## What Average Candidates Miss

1. **Conflating object and block storage** — block storage (EBS) is byte-addressable, attached to one instance, latency-sensitive. Object storage is high-latency, metadata-addressed, massively scalable. They are not the same thing.
2. **Routing upload bytes through app servers** — uploads of 1 TB through your app servers makes them the bottleneck. Pre-signed URLs are the industry-standard solution.
3. **Ignoring metadata as a first-class service** — the metadata lookup path is on every single read and write. It must be fast, durable, and independently scalable.
4. **Assuming 3× replication everywhere** — at petabyte scale, 3× replication costs 60% more storage than 5+4 erasure coding. Not mentioning this at Amazon (whose core product this is) signals shallow knowledge.
5. **No garbage collection design** — deleted objects and abandoned multipart uploads occupy disk until someone cleans them up. Who is that? How? When?
6. **Ignoring the eventually-consistent window** — before S3's December 2020 update, a PUT followed immediately by a GET could return 404. Candidates who don't mention consistency model evolution miss a key correctness nuance.
