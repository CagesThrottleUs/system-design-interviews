# Learning Progress

## Current Position

**Layer:** 1 — Foundations  
**Current Lesson:** L1-02 — Push Delivery: APNs & FCM (PDF delivered 2026-04-30)
**Next Lesson:** L1-08 — Message Queues & Streaming  
**Reason:** L1-02 directly fixes the session_001 critical gap (device token, OS-level relay, store-and-forward). L1-08 is the final Gate 1 prerequisite before the Notification System retry mock.

## Completed Lessons

| Lesson | Topic | Date | File |
|--------|-------|------|------|
| L1-01 | Networking: TCP, UDP, HTTP, WebSockets | 2026-04-21 | [L1-01-networking.tex](layer1-foundations/L1-01-networking.tex) |

## Completed Mock Interviews

| Session | Question | Score | Date | Key Gap |
|---------|----------|-------|------|---------|
| session_001 | Notification System | 9/25 | 2026-04-15 | APNs/FCM delivery model; requirements discipline |

## Identified Knowledge Gaps

| Gap | Severity | Covered By | Status |
|-----|----------|------------|--------|
| APNs/FCM — device is client not server | Critical | L1-02 | ☐ |
| Persistent TCP connections vs HTTP polling | High | L1-01 | ✅ |
| Message queue async pipelines (Kafka/SQS) | High | L1-08 | ☐ |
| Priority queue separation for notification tiers | Medium | L2-09 | ☐ |
| Requirements phase discipline (no design in requirements) | Soft skill | Practice | ☐ |
| Observability — "how do you know it's working?" | High | L1-13 | ☐ |
| Outbox pattern — reliable event publishing | Medium | L2-11 | ☐ |

## Session Notes

### 2026-04-15
- First session — diagnostic
- Showed good meta-cognition and self-correction
- Framework thinking solid after coaching
- Critical gap: APNs/FCM delivery mechanism
- Recommended: L1-01 → L1-02 → L1-08 before next mock

## Scheduled Interviews

| Gate | Question | Unlocked | Status |
|------|----------|----------|--------|
| — | No interviews unlocked yet. Complete L1-01 + L1-02 + L1-08 to unlock Gate 1 (Notification System retry). | — | — |

## Lesson Format Preferences

- Lessons delivered as compiled LaTeX PDFs (pdflatex) — opened immediately after compile
- No voice mode unless explicitly requested
- Self-quiz with 5 interviewer-probe questions and model answers in every lesson
- Curiosity Corner with ≥5 deep Q&A entries in every lesson
- Progress and gap tracking updated automatically after every lesson
