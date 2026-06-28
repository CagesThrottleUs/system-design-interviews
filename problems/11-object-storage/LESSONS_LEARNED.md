# Lessons Learned — Object Storage (S3-like)

---

## Real Interview Stories

### Story 1 — Amazon, L5 SDE (2024)

The candidate described a design with three replicas of every object across three availability zones. The interviewer asked: "You've got 500 PB of user data. What's your monthly storage bill at $0.023/GB?" The candidate did the math live: 500 PB × 3 replicas = 1.5 EB. At $0.023/GB that's roughly $34 billion per year — clearly absurd. The interviewer pushed: "At this scale, Amazon doesn't use replication. Do you know what they use instead?"

The candidate had not prepared erasure coding. They knew the term but could not explain how it works, what the overhead is, or why it achieves better durability at lower cost. The debrief was "strong problem decomposition but insufficient depth on storage fundamentals for L5." **The lesson: S3 internals are table stakes at Amazon. Know erasure coding.**

---

### Story 2 — Meta, E5 (2023)

The candidate designed a system where uploads went: Client → Load Balancer → App Server → Object Store. The interviewer asked them to estimate the bandwidth requirements on the app server tier for 1 million concurrent uploads of average 10 MB files.

The candidate worked it out: 1M concurrent × 10 MB = 10 PB of data moving through app servers. Obviously impossible. The interviewer asked: "How would you fix this?" The candidate then invented something ad hoc — a "transfer service" — without knowing the industry term (pre-signed URLs) or how HMAC signing works.

The candidate passed but got feedback that they "re-derived an existing well-known pattern rather than applying it." **The lesson: pre-signed URLs are a named, well-known pattern. Know the name, know how HMAC signing works, know the expiry mechanics.**

---

### Story 3 — Dropbox, Senior Engineer (2024)

The interviewer asked the candidate to design the "write path" in detail. The candidate's design stored metadata in the same service as the objects themselves. The interviewer probed: "Your metadata lookup is slowing down as the namespace grows to 10 billion keys. How do you scale it without scaling your object storage layer?"

The candidate struggled because metadata and data were coupled. They proposed sharding but couldn't explain how to shard without also repartitioning all the data. The interviewer pointed out that Dropbox Magic Pocket separates metadata (sharded MySQL) from data nodes entirely — they scale independently.

**The lesson: state the metadata/data separation early, and justify it by saying their access patterns, hardware requirements, and scaling needs are orthogonal. Coupling them forces you to scale both together even when only one is the bottleneck.**

---

## General Pattern Mistakes

**Mistake 1: Treating object storage like a file system.** File systems have directories, hard links, random byte-range writes, and POSIX semantics. Object storage is a flat key-value namespace: you write the whole object, you read the whole object (or a range), and there are no directories — prefixes like `photos/` are just string matching on keys. Candidates who design "folders" in object storage are showing they don't understand the abstraction.

**Mistake 2: Handling metadata as an afterthought.** A common design shows a data node cluster and then, at the end, adds "oh and there's a database for metadata." The metadata service is actually the hot path for every single operation. Its latency determines object access latency. Its availability determines system availability. Design it as a first-class component with explicit sharding strategy, replication, and failover.

**Mistake 3: Not designing for partial failure on large uploads.** A client uploading a 50 GB file may experience a dropped connection 90% of the way through. Without multipart upload design, the answer to "what happens?" is "the client starts over." For large objects this is catastrophic. Interviewers probe this explicitly: "Walk me through what happens if the client disconnects after uploading 30 GB of a 50 GB object."

**Mistake 4: Ignoring the garbage collection problem.** Every delete leaves data on disk. Every abandoned multipart upload occupies space. Every overwritten object version occupies space. Who cleans this up? The answer "we just delete it on write" is wrong — you need safe deletion windows for concurrent readers. A garbage collector with tombstone-based soft deletion is the industry pattern.

**Mistake 5: Not calling out the consistency model.** The question "can a client immediately read an object after uploading it?" has a nuanced answer. S3 guaranteed only eventual consistency until December 2020. The reason: metadata propagation across replicas took non-zero time. Strong consistency requires all metadata writes to be synchronous and visible before returning 200 to the uploader. This is a design choice with latency implications, not a free lunch.

---

## Self-Assessment Checklist

After your practice session, score yourself:

**Fundamentals:**
- [ ] Correctly distinguished object vs block vs file storage in the first 5 minutes
- [ ] Separated metadata service from data nodes from the start, with justification
- [ ] Named erasure coding as the durability mechanism, not just "replication"

**Depth:**
- [ ] Explained chunk size trade-off (metadata overhead vs retry cost)
- [ ] Described the three-step multipart upload protocol (initiate/upload/complete)
- [ ] Explained pre-signed URL flow end-to-end (HMAC signing, expiry, direct client-to-node upload)

**Failure scenarios:**
- [ ] Addressed what happens when a data node fails mid-upload
- [ ] Addressed what happens when a client disconnects mid-multipart-upload
- [ ] Named the garbage collector as the component that handles orphaned parts and soft-deleted objects

**Advanced:**
- [ ] Mentioned that S3 only achieved strong consistency in December 2020
- [ ] Calculated erasure coding overhead (1.8×) vs replication overhead (3×) and stated the cost implication
- [ ] Addressed how consistency works for cross-region reads (eventual is acceptable)

---

## Remediation Targets

**If you couldn't explain erasure coding:** Study Reed-Solomon codes. Know the parameters: for S3-style 5+4, you produce 9 shards from 5 data shards, and can recover from any 4 simultaneous failures. Understand that 9 shards × (1/5 of original data) = 1.8× overhead.

**If you didn't know pre-signed URLs:** Study how AWS S3 presigned URLs work. The key mechanic is HMAC-SHA256 over the canonical request (method, bucket, key, expiry, headers). The data node validates the signature at request time without contacting the auth service.

**If you couldn't describe multipart upload:** Read the AWS S3 multipart upload documentation. Know: minimum part size (5 MB), maximum parts (10,000), maximum object (5 TB), the UploadId lifecycle, and what happens to incomplete uploads.

**If you missed the metadata/data separation:** Read the Dropbox Magic Pocket blog post (dropbox.tech/infrastructure/magic-pocket-infrastructure). It describes exactly why they separated the metadata tier from the OSD (object storage device) tier.
