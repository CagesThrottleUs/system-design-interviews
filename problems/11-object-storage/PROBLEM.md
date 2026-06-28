# Object Storage (S3-like)

**Difficulty:** Intermediate
**Category:** Storage-heavy, Infrastructure
**Time Box:** 45 min
**Key Concepts:** Object vs block vs file storage, chunked multipart upload, metadata/data service separation, erasure coding vs replication, pre-signed URLs
**Asked at:** Amazon (directly relevant — S3 is their product), Google, Meta, Dropbox

---

## Problem Statement

Your company runs a cloud platform where customers store and retrieve arbitrary files — backups, media assets, application artifacts, user uploads. The product interface is simple: upload a file with a key, download it by key, delete it. But the engineering challenge is everything underneath: files range from tiny thumbnails (4 KB) to multi-terabyte database dumps, and the system must survive individual disk failures, entire server failures, and even datacenter outages without losing a single byte.

Today, files are stored on a fleet of NFS servers. Engineers are hitting the ceiling: large uploads saturate a single server's bandwidth, metadata queries slow down as the namespace grows past 100 million objects, and a single NFS server failure takes down all writes. The task is to design the next-generation object storage layer — one that separates concerns cleanly, scales storage and metadata independently, and makes large-file uploads a first-class operation.

For context on real-world scale: AWS S3 stores over 350 trillion objects as of 2024 and processes over 100 million requests per second. Dropbox's Magic Pocket service, launched in 2016, stores over 500 petabytes of user data with 12 nines of durability across custom-built cells.

---

## Actors

| Actor | Description | Primary Action |
|-------|-------------|----------------|
| **API Client** | Application or end user uploading/downloading files | PUT, GET, DELETE via REST API |
| **Uploader** | Client performing large multipart uploads | Initiate → upload parts → complete |
| **Consumer** | Service reading files (e.g., image resizer, CDN origin) | GET by object key |
| **Admin / Platform** | Internal tooling managing buckets, quotas, lifecycle policies | Bucket management, policy enforcement |

---

## Functional Requirements

**Core (MVP):**
- FR1: Upload an object given a bucket name and key; return a success confirmation
- FR2: Download an object by bucket + key
- FR3: Delete an object by bucket + key
- FR4: Support objects from 1 byte to 5 TB in size
- FR5: Generate a time-limited pre-signed URL granting temporary access to a private object

**Secondary:**
- FR6: List objects in a bucket with optional prefix filtering
- FR7: Support multipart upload: initiate, upload N parts in parallel, complete or abort
- FR8: Support object versioning (multiple versions of same key)
- FR9: Bucket-level lifecycle policies (e.g., delete objects older than 90 days)

---

## Non-Functional Requirements

| Requirement | Target | Rationale |
|-------------|--------|-----------|
| Durability | 11 nines (99.999999999%) | S3's published SLA; losing objects is catastrophic for customers |
| Availability | 99.99% | Object stores underpin other services; downtime cascades |
| Upload throughput | > 1 GB/s per large object | Multi-terabyte backups must complete in hours, not days |
| Download latency (p99, small objects) | < 100 ms | Thumbnail/asset reads must feel fast; CDN fronts hot objects |
| Metadata lookup latency (p99) | < 20 ms | Every request must find the object location before serving data |
| Storage cost | Minimize overhead | At petabyte scale, 1% overhead = millions of dollars/year |
| Consistency | Strong for metadata; eventual acceptable for cross-region replication | Clients expect to immediately read what they just wrote (same region) |

---

## Capacity Estimation Hints

Use these to derive your own numbers before the solution guide:

- **DAU:** 10 million active users
- **Average objects per user:** 1,000 (mix of photos, documents, backups)
- **Average object size:** 1 MB (mix of 4 KB thumbnails to 50 GB videos; median is ~1 MB)
- **Upload ratio:** each user uploads ~5 objects/day on average
- **Read:write ratio:** 10:1 (objects are read more than they are written)
- **Total object count:** 10 billion objects at scale

Derive: daily write QPS, daily read QPS, total storage, bandwidth requirements, metadata database size.

---

## Good Clarifying Questions to Ask

### Scale
1. How many objects total, and what is the distribution of object sizes? (determines if you need chunking, and whether metadata fits in a single DB)
2. What is the expected read-to-write ratio? (drives caching strategy for hot objects)
3. Do we need cross-region replication, or is single-region durability sufficient? (dramatically changes replication design)

### Consistency
4. If a client uploads an object and immediately reads it, must the read succeed? (strong read-after-write vs eventual consistency — S3 only guaranteed eventual until 2020)
5. If two clients upload to the same key simultaneously, which wins? (last-writer-wins? conditional writes?)

### Durability
6. What is the target durability SLA? (11 nines requires erasure coding, not just replication)
7. Must we survive a full availability zone failure? (drives cross-AZ placement policy)

### Features
8. Do we need multipart upload for large objects? At what threshold? (anything over ~100 MB benefits significantly)
9. Do we need object versioning? (if yes, storage costs grow; affects delete semantics)
10. Do we need access control at the object level, or just bucket level? (pre-signed URLs, bucket policies, IAM integration)

---

## Key Components

<details>
<summary>Reveal components</summary>

- **API Gateway** — exposes PUT/GET/DELETE/presign REST endpoints; validates auth and routes to correct service
- **Metadata Service** — stores `{bucket/key → object location, size, checksum, content-type, version}`; separate from data nodes
- **Data Node** — stores raw object bytes; multiple nodes per rack, multiple racks per AZ
- **Chunk Manager** — for large objects, splits into fixed-size chunks, distributes across data nodes, tracks chunk locations in metadata
- **Erasure Coding Engine** — encodes chunks into data+parity shards; tolerates N simultaneous node failures
- **Replication Manager** — ensures durability targets are maintained; triggers re-replication when nodes fail
- **Pre-Signed URL Service** — signs time-limited URLs using HMAC; validates on GET without requiring the client to hold credentials
- **Garbage Collector** — reclaims storage from deleted or overwritten objects after safe deletion window

</details>
