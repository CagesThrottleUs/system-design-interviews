# Mock Interview Script — Collaborative Editor
*For the interviewer. Do not share with candidate.*

## Pre-Session Setup
- Problem: Design a real-time collaborative document editor (Google Docs scale)
- Target level: Google L5 / Notion Senior / Figma Senior — this is an Advanced problem
- Time: 45 minutes
- Key skill being tested: understanding of distributed state synchronization — OT vs CRDT
- Most candidates fail here; be patient but rigorous

## Opening (read aloud, verbatim)
"Design a real-time collaborative text editor — multiple users editing the same document simultaneously, like Google Docs. You have 45 minutes. Start with clarifying questions."

*Start timer. Say nothing else.*

## Phase 1: Requirements (0–8 min)

**Answers if asked:**
| Question | Answer |
|----------|--------|
| Scale? | 10M documents; 100K active at once; up to 100 concurrent editors |
| Rich text or plain text? | Rich text (bold, italic, lists) — simplify to plain text if needed |
| Real-time cursor presence? | Yes |
| Offline editing? | Yes — edits made offline sync on reconnect |
| Undo? | Yes — user can undo their own changes |
| Version history? | Yes — can see who changed what |
| Permissions? | Owner/Editor/Viewer — mention but no deep dive needed |

**Red flag:** Candidate doesn't ask about conflict resolution at all.

**Green flag:** Candidate immediately asks "how do we handle two users editing the same position?" before asking about scale.

## Phase 2: Estimation (8–13 min)

**Expected:**
- 100K active docs × 5 editors × 3 keystrokes/sec = 1.5M ops/sec (ceiling)
- Realistically: 20% of peak = 300K ops/sec
- Storage: 300K × 50 bytes = 15 MB/sec = 1.3 TB/day

**Key question:** "What's the highest-latency-sensitive operation in this system?"
Expected answer: "Local edit — must be 0ms. Sync to other users can be < 500ms."

## Phase 3: High-Level Design (13–28 min)

**THE critical question — ask at minute 18:**
"Alice is at position 5 in the document. Bob is also at position 5. Alice types 'X'. Bob types 'Y'. Both operations reach the server at the same millisecond. What does the document look like? Is Alice's 'X' there? Is Bob's 'Y' there? Do all users see the same thing?"

**Evaluate the response carefully:**
- "Whoever arrives first wins" → WRONG. One edit is lost. Push back: "The user whose edit is lost watches their text disappear. Is this acceptable?"
- "We use a lock so only one can edit at a time" → WRONG for collaborative editing. Push back: "Google Docs doesn't lock the document. Both users edit freely."
- "We use OT / CRDT" → Correct direction. Ask them to explain how it works.
- Cannot answer → Major gap. Give hint: "Have you heard of Operational Transformation or CRDTs?"

**After conflict resolution is addressed, ask about:**
1. "How does Alice's browser show her own character immediately without waiting for the server?"
2. "Bob is offline for 60 seconds and makes edits. He reconnects. What happens?"

**Expected components:**
- WebSocket connections per document room
- Operation server with OT or CRDT engine
- Operation log (Cassandra, append-only)
- Document snapshots (S3, periodic)
- Presence service (Redis pub/sub, separate from ops)

## Phase 4: Deep Dive (28–40 min)

**Option A — The Core Algorithm (for candidates who mentioned OT or CRDT):**
"Pick one — OT or CRDT — and walk me through exactly what happens when Alice inserts 'X' at position 5 and Bob inserts 'Y' at position 5 simultaneously. Show me the operations, the transformation or merge, and the final document state."

*This question separates engineers who understand the algorithm from those who name-dropped it.*

**Option B — Cold Start (if they didn't mention snapshots):**
"A user opens a document that has been edited 100,000 times over 2 years. Walk me through what happens. How long does it take to load? What needs to happen before the user can start editing?"

**Option C — Offline Reconnect:**
"Charlie edited the document offline for 60 seconds, making 30 operations. During that time, 20 operations from other users were committed to the server. Charlie reconnects. Walk me through exactly what happens — from Charlie's reconnect message to the point where all clients have a consistent view."

**Option D — Undo in Collaborative Context:**
"Alice types 'Hello' (5 chars). Bob types 'World' (5 chars) after Alice. Alice presses Ctrl+Z five times. What happens? Does Bob's 'World' disappear? How does your system implement undo correctly in a collaborative context?"

## Phase 5: Failure & Scaling Probe (40–44 min)

Pick TWO:
1. "Your operation server crashes. There are 500 active editors across 100 documents. What happens? Do they lose their work?"
2. "CRDT tombstones are accumulating — deleted characters are never garbage collected. A 5-year-old document has 10MB of tombstones for 50KB of visible text. What do you do?"
3. "Your Cassandra op log is getting expensive — 1.4 PB after 3 years. How do you reduce storage cost without losing history?"
4. "Two operation servers receive the same client operation (network retry after timeout). How does your system handle the duplicate?"

## Closing (44–45 min)
"Final question: if you were going to implement this from scratch, would you choose OT or CRDTs, and why?"

## Scoring Checklist

| Dimension | Evidence | Score (1-5) |
|-----------|----------|-------------|
| Requirements | Asked about conflict resolution, offline, undo — not just scale | |
| Core algorithm | Named OT or CRDT; explained WHY either solves the concurrent edit problem | |
| Client-side design | Optimistic local apply; pending ops queue | |
| Persistence | Cassandra op log + S3 snapshots; partition key reasoning | |
| Presence | Separated from op pipeline; ephemeral; Redis pub/sub | |
| Offline/reconnect | Described reconnect flow with revision-based delta sync | |
| Depth | Could explain OT transformation function OR CRDT insert mechanics at a concrete level | |
