# Lessons Learned — Collaborative Editor

## Real Interview Stories

**Story 1: Google L5 — "Just Use WebSockets"**
Company: Google | Level: L5 | Round: System Design

What happened: Candidate proposed a clean WebSocket architecture for sending keystrokes to a server, which broadcasts to all connected clients. Interviewer set the scenario: "Alice is on character 5, types 'X'. Bob is on character 5, types 'Y'. Both send their operations at the same moment. What does the document look like?" Candidate said: "Whoever arrives at the server first wins."

Where it went wrong: Last-write-wins means the other user's character is silently discarded. One user watched their typing disappear. The candidate had not considered that collaborative editing requires a merge, not a conflict resolution.

Consequence: Interviewer spent 10 minutes dragging the candidate toward OT/CRDT concepts. Candidate had never heard of either. Marked as not meeting bar — "does not understand distributed state synchronization."

What to do instead: Before designing the architecture, state the core problem: "The challenge is concurrent edit conflict resolution. I'll use CRDTs: each character gets a unique ID and a parent reference, making operations commutative. Any order of application produces identical results." Then build the architecture around this.

---

**Story 2: Figma Senior Engineer — OT vs CRDT Deep Dive**
Company: Figma | Level: Senior SWE | Round: System Design (collaborative design tool — same core problem)

What happened: Candidate correctly identified CRDTs as the right approach (Figma famously uses CRDTs). Explained that characters get unique IDs and positions are relative. Interviewer asked: "Walk me through what happens when Alice deletes a character that Bob simultaneously moved his cursor past."

Where it went wrong: Candidate couldn't handle deletions in the CRDT model. They explained insertions correctly but froze when asked about tombstoning — deleted characters in CRDTs must be kept as tombstones (invisible but present) because other clients may reference them as parent pointers.

Consequence: Interviewer concluded the candidate understood CRDT at a surface level but not deeply enough for a company where CRDTs are core infrastructure. Deferred hire.

What to do instead: Know the full CRDT model including tombstones. When a character is "deleted," it is marked as deleted (tombstoned) but retained in the structure. Other characters that were inserted after it maintain their parent reference — the parent still exists, just invisible. Tombstone GC happens asynchronously when all clients confirm they've processed the delete.

---

**Story 3: Notion Senior SWE — Missing Cold Start Design**
Company: Notion | Level: Senior SWE | Round: System Design

What happened: Candidate designed an excellent real-time collaboration system — WebSockets, CRDT operations, Cassandra op log. Interviewer asked: "A user opens a document for the first time. Your Cassandra op log has 100,000 operations. What happens?"

Where it went wrong: Candidate had no snapshot mechanism. Their design required replaying all 100,000 operations on document open. For a large document with heavy edit history, this could take 10+ seconds on the client.

Consequence: Interviewer asked: "What is the user experience? They click the document and wait 10 seconds?" Candidate couldn't provide a solution. Marked down on scalability of the read path.

What to do instead: Always design a snapshot mechanism. Every N operations (e.g., 1,000) or every 24 hours, take a full snapshot of the document state and persist to S3. Document load = fetch latest snapshot + replay only ops since snapshot. Cold start for a 100,000-op document: load snapshot at op 99,000, replay ops 99,001-100,000 (max 999 ops). Startup time: milliseconds, not seconds.

## General Pattern Mistakes

**Mistake 1: Last-Write-Wins Conflict Resolution**
What happens: Candidate proposes "whoever arrives at the server first wins" for concurrent edits.
Why wrong: The losing user's edit is silently discarded. They watch their typing disappear. This is the exact user experience Google Docs was designed to prevent.
Fix: Both edits must be preserved. OT transforms positions so both fit. CRDT assigns unique IDs so both coexist deterministically.

**Mistake 2: Treating Collaborative Editing as "Just Sync"**
What happens: Candidate proposes "send all keystrokes to server, broadcast to all clients." No conflict resolution discussed.
Why wrong: Two users typing at the same position simultaneously is a race condition, not a sync issue. Broadcasting raw operations without transformation produces divergent states.
Fix: Every operation must include a revision number (OT) or a parent ID (CRDT). The server must transform or merge concurrent operations before broadcasting.

**Mistake 3: No Snapshot Mechanism**
What happens: Candidate designs an op log but no snapshot. Document state is always reconstructed by replaying all ops.
Why wrong: A document with 100K edits requires 100K op replays on every open. At 3 ops/second average, that's 33,000 seconds of accumulated edit history. Replay is not instantaneous.
Fix: Periodic snapshots (every 1,000 ops or 24h). Cold start = latest snapshot + ops since snapshot. Maximum replay: 999 ops.

**Mistake 4: Mixing Presence into the Op Log**
What happens: Candidate treats cursor position updates as document operations, persisted alongside content ops.
Why wrong: Cursor positions are ephemeral and high-frequency (every 500ms per user). Persisting them inflates the op log, makes history noisy, and complicates undo (must skip presence ops).
Fix: Separate presence channel (Redis pub/sub). No persistence. Cursor positions reset on reconnect — visible within 500ms.

**Mistake 5: Blocking Local Apply on Server Confirmation**
What happens: Candidate designs the client to wait for server ack before showing the typed character.
Why wrong: Network round-trip is 50-200ms minimum. Every keystroke has visible lag. The editor feels broken.
Fix: Optimistic local apply — show character immediately, reconcile with server later. If server transforms the op, client replays with the correct transformation. In practice, < 0.1% of ops need reconciliation.

## Self-Assessment Checklist
- [ ] I named the core problem: concurrent edits at the same position must both be preserved
- [ ] I explained OT vs CRDT as a design choice and named a preference with rationale
- [ ] I designed optimistic local apply (0ms keystroke latency)
- [ ] I separated presence (ephemeral, Redis) from document ops (persistent, Cassandra)
- [ ] I designed a snapshot mechanism to bound cold-start cost
- [ ] I described the offline reconnect flow (send last_server_revision, server sends delta)
- [ ] I designed the operation log schema with doc_id as partition key
- [ ] I addressed undo in a collaborative context (selective undo, not global stack pop)

## Remediation Targets
- If you can't explain OT: Read "Operational Transformation in Real-Time Group Editors" (Ellis & Gibbs, 1989) summary or the Jupiter Collaboration System paper. Practice transforming two concurrent insert operations by hand.
- If you can't explain CRDT: Read Martin Kleppmann's "A Conflict-Free Replicated JSON Datatype" (2017). Study Yjs source code for a production CRDT implementation.
- If you missed snapshots: Practice the "event sourcing with snapshots" pattern. It appears in many system design problems where state is rebuilt from a log.
- If you missed optimistic apply: Study "optimistic concurrency control" in database contexts. Same principle: apply locally, reconcile on conflict.
