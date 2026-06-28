# Collaborative Editor (Google Docs / Notion)
**Difficulty:** Advanced
**Category:** Distributed State / Conflict Resolution / Real-Time
**Time Box:** 45 min
**Key Concepts:** Operational Transformation, CRDTs, Conflict Resolution, Presence, Undo/Redo
**Asked at:** Google, Figma, Notion, Dropbox, Microsoft, Atlassian, Linear

## Problem Statement

You are building a real-time collaborative document editor — think Google Docs or Notion. Multiple users can edit the same document simultaneously from their own devices. When Alice types "Hello" and Bob simultaneously types "World" in the same document, both edits must be preserved and all users must see the same final document. When a user loses network connectivity for 30 seconds and comes back, all their offline changes must merge cleanly with changes made by others during that time.

This problem sits at the intersection of distributed systems, data structures, and user experience. The core challenge is not "how do we sync changes quickly" but "how do we resolve conflicting concurrent edits in a way that is deterministic, correct, and feels natural to the user."

## Actors
| Actor | Description |
|-------|-------------|
| Editor | User actively typing in the document |
| Viewer | User viewing the document (no edits) |
| Presence System | Tracks cursor positions and online status of all collaborators |
| Persistence Layer | Durably stores the document and its edit history |

## Functional Requirements
**Core (MVP):**
- FR1: Multiple users can edit the same document simultaneously
- FR2: All users see the same document state (eventual consistency, converges to same result)
- FR3: No edit is silently lost — both concurrent edits are preserved where possible
- FR4: Each user sees other collaborators' cursors in real-time
- FR5: Users can undo their own changes without affecting others' edits

**Secondary:**
- FR6: Offline editing — user can edit without connectivity; changes sync on reconnect
- FR7: Edit history — users can see who changed what and when
- FR8: Document permissions: owner, editor, commenter, viewer

## Non-Functional Requirements
| Requirement | Target | Rationale |
|-------------|--------|-----------|
| Local edit latency | 0ms (applied locally before server confirms) | Editor must feel snappy; never block on network |
| Sync latency | < 500ms for collaborators to see each other's edits | Beyond this, collaboration feels asynchronous |
| Convergence guarantee | All clients eventually see identical document | Core correctness requirement |
| Conflict resolution | Deterministic, automatic | No "merge conflict" dialogs — must resolve silently |
| Availability | 99.9% | Document editing is high-value but not safety-critical |
| Scale | 10M documents, 100 concurrent editors per document | Long tail: most docs have 1-2 editors; some have 100+ |

## Capacity Estimation Hints
- Total documents: 10 million
- Documents with active editors right now: 100,000 (1% active at once)
- Average concurrent editors per active document: 5
- Keystrokes per editor per second (active typing): 3
- Total operations/second: 100,000 docs × 5 editors × 3 keystrokes = 1.5M ops/sec
- Average operation size: 50 bytes
- Document size: average 50 KB; max 10 MB

## Good Clarifying Questions to Ask

**Scale:**
- How many concurrent editors per document? (100 is high; most systems cap lower)
- How many total documents? How many active at once?
- What's the maximum document size?

**Conflict resolution:**
- When two users type at the same position simultaneously, what should happen?
- Should we use Operational Transformation or CRDTs? (Advanced: shows familiarity)
- Do we need offline editing support?

**Features:**
- Is real-time cursor presence required?
- Do we need per-character attribution (track changes)?
- What types of content? Plain text only, or rich text (bold, lists, images)?

**Undo:**
- Does undo affect only my changes, or everyone's?
- What is the undo scope — within my session, or globally?

## Key Components
<details><summary>Reveal components</summary>

- **Client Editor (local)**: Applies operations immediately to local state; sends operations to server; receives remote ops and applies them
- **Operation Server**: Receives client operations; applies transformation if needed; broadcasts to other clients; persists to op log
- **CRDT / OT Engine**: Core algorithm that resolves concurrent edits — either OT (requires central server for transformation) or CRDT (peer-to-peer capable, preferred for new systems)
- **Operation Log (Cassandra)**: Append-only log of all operations; enables history, undo, and reconnect sync
- **Document Snapshot Store (S3/blob)**: Periodic full snapshots of document state; reduces replay cost on cold start
- **Presence Service**: Tracks cursor positions and user activity; ephemeral (Redis pub/sub or WebSocket broadcast); separate from document state
- **WebSocket Gateway**: Maintains persistent connections per document room; fan-out operations to all editors
</details>
