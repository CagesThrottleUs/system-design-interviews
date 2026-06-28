# Mock Interview Script — Notification System

---

## Pre-Session Setup

Read this section before the session begins. The candidate should not read past this page until the interviewer says "I'm starting the timer."

**For the interviewer or AI:** Your role is not to help the candidate succeed — it is to accurately evaluate their current level. Do not nod, confirm, or give any signal that a direction is correct. Respond to "is this right?" with "what makes you confident in that?" Do not offer hints unless the candidate explicitly asks for one, and note each hint given (each hint costs 0.5 points from the Deep Dive score). Do not rescue a candidate who is silent — let the silence stand for up to 60 seconds before prompting with "walk me through your thinking."

**For the candidate:** Do not read the problem statement until the interviewer tells you to begin. During the session, narrate your reasoning aloud. If you are uncertain, say so and reason from first principles. If you need to change direction, say "I want to revisit my earlier choice about X because Y" — pivoting explicitly is a positive signal, not a negative one. Do not ask the interviewer to confirm your direction.

**Materials needed:** A blank sheet of paper or a whiteboard. A pen. A timer visible to both parties (the interviewer tracks it; the candidate should not be watching it).

**Duration:** 45 minutes for the session. 15 minutes for debrief (Mode 2 / Partner only). AI sessions write the debrief to PROPOSED_SOLUTION.md automatically.

---

## Opening Prompt

*Read this aloud, or if using AI interview mode, send this as your first message after typing `/interview notification-system`:*

---

> "I'd like you to design the notification infrastructure for a large social platform. The platform has on the order of 100 million daily active users. When users interact with content — liking posts, sending direct messages, mentioning others — the relevant users receive push notifications on their mobile devices and browsers.
>
> You have 45 minutes. I'll start timing when you tell me you're ready. Please think aloud throughout the session — I want to hear your reasoning as you go, not just your conclusions. At any point I may ask follow-up questions; answer them as you would in a real interview.
>
> Before you start: do you have any immediate clarifying questions about what I've described, or are you ready to begin?"

*Wait for the candidate to respond. If they ask a clarifying question before starting the timer, answer it from the Q&A table below and then start the timer when they say they are ready. If they say "ready," start the timer immediately.*

---

## Session Timing Guide

| Phase | Time Window | What to Look For | When to Probe |
|-------|------------|-----------------|---------------|
| Requirements Clarification | 0–5 min | 5-category framework coverage (scale, delivery, latency, ordering, compliance) | If candidate starts drawing before asking any questions, say: "Before you start designing — do you have any questions about the requirements?" |
| Capacity Estimation | 5–10 min | Derivation chain visible, bottleneck identified | If candidate states numbers without derivation, ask: "How did you arrive at that number?" |
| High-Level Design | 10–20 min | End-to-end data flow, all major components named, protocols labeled | If candidate goes deep on one component before finishing the topology, say: "Can you finish the end-to-end flow before we go deeper there?" |
| Deep Dive | 20–40 min | Candidate-chosen component, tradeoffs surfaced, alternatives named | If candidate picks a low-complexity component, ask: "Is that the hardest part of the system to get right?" |
| Failure Modes & Scaling | 40–45 min | SPOF identified, cascade failure scenario, scaling inflection point | Ask at least one of the three probes at the bottom of this document |

---

## Q&A Table — Clarifying Questions

The following covers the questions a strong candidate will ask in the first 5 minutes. Answer these from the table; do not elaborate beyond what is given here.

| Candidate Question | Strong Answer to Give | Weak Answer Indicator | Follow-up Probe |
|-------------------|----------------------|----------------------|----------------|
| What delivery guarantee do we need? / Does every notification have to arrive? | "At-least-once. It's more acceptable to send a notification twice than to silently drop one. For security-related notifications like password resets, we treat them as high priority — if they can't be delivered within 60 seconds, they should not be delivered at all." | Candidate does not ask; proceeds with an implicit assumption | "You've chosen [eventual consistency / strong consistency] for your delivery model — what delivery guarantee does that imply?" |
| What platforms do we need to support? | "iOS, Android, and web browsers. Roughly 70% iOS, 25% Android, 5% web." | Candidate assumes a single platform or doesn't ask | "How does your design change if a user has devices on multiple platforms?" |
| What's the expected notification volume? | "On average, each user receives about 3 notifications per day. Peak traffic can be up to 5× the average." | Candidate uses arbitrary numbers with no basis | "Walk me through how you derived that QPS number." |
| Do we need to handle users with very large follower counts? / Is there a celebrity problem? | "Yes. Some accounts have 10 to 50 million followers. A single post from one of those accounts can trigger tens of millions of notification events." | Candidate designs a uniform fan-out without considering the distribution | "How does your fan-out design perform when a single event needs to generate 10 million delivery tasks?" |
| How do we handle offline users? | "Notifications should be queued and delivered when the user comes back online, subject to a TTL. For most notification types the TTL is 24 hours. For a password reset it's 60 seconds." | Candidate does not address offline behavior at all | "What happens in your system if the target user's device is offline for 6 hours?" |
| Do we need to rate-limit or batch notifications? | "Batching is a v2 concern. For now, each event generates one notification. If a post gets 10,000 likes, the user receives 10,000 like notifications. That's clearly bad UX but out of scope for this session." | Candidate spends time on batching logic at the expense of core delivery architecture | Redirect: "Let's set batching aside and make sure the core delivery path is solid first." |
| Should we handle notification preferences? / Can users opt out? | "That's a v2 feature. Assume all notifications are delivered to all devices unless a token is explicitly invalid." | No impact on design for v1; do not probe | — |

---

## Phase-by-Phase Hints

*Only reveal hints if the candidate explicitly asks for one. State: "I can give you a hint, but it will cost 0.5 points from your Deep Dive score. Do you still want it?" Track how many hints are given.*

### Phase 1 — Requirements Clarification

| Hint Level | Hint Text |
|-----------|-----------|
| Least specific (hint 1) | "Think about what could go wrong between a publisher sending an event and a user receiving a notification on their phone. What properties does the delivery need to have?" |
| Moderate (hint 2) | "There are 5 dimensions worth clarifying for almost any async delivery system: scale, delivery semantics, latency SLAs, ordering guarantees, and compliance. Have you covered all five?" |
| Most specific (hint 3) | "Ask specifically: what happens if we try to deliver a notification and the device is offline? And: is it worse to deliver a notification twice, or to miss it entirely?" |

### Phase 2 — Capacity Estimation

| Hint Level | Hint Text |
|-----------|-----------|
| Least specific (hint 1) | "Start from the number of users and work forward — how many notifications does the system handle per second?" |
| Moderate (hint 2) | "You have 100M DAU and an average of 3 notifications per user per day. Derive QPS from there, then multiply by the number of devices per user to get the actual delivery task rate." |
| Most specific (hint 3) | "The bottleneck in a notification system is usually not ingestion — it's fan-out. Calculate how many per-device delivery tasks per second you're generating, and ask which component in your design has to process all of them." |

### Phase 3 — High-Level Design

| Hint Level | Hint Text |
|-----------|-----------|
| Least specific (hint 1) | "Think about what sits between the publisher service that fires a like event and the APNs server that delivers the push. What are the intermediate steps?" |
| Moderate (hint 2) | "Most notification systems have at least these layers: an ingestion API, a durable queue, a fan-out layer that resolves user IDs to device tokens, per-platform delivery workers, and a device token store. Have you addressed all of these?" |
| Most specific (hint 3) | "Draw the path from a publisher POSTing a notification event to the final HTTP call to APNs. Name every component that touches that data in transit, and label what protocol connects each pair of components." |

### Phase 4 — Deep Dive

| Hint Level | Hint Text |
|-----------|-----------|
| Least specific (hint 1) | "Where does the hardest engineering problem in this system live? It's probably not in the component you've been describing." |
| Moderate (hint 2) | "The 1-to-N amplification at the fan-out layer is where most interesting problems in notification systems live. What happens when a single event needs to generate 10 million per-device delivery records?" |
| Most specific (hint 3) | "Consider: your fan-out service consumes events and writes per-device delivery records. If a single celebrity with 10M followers posts something, how long does it take your fan-out service to process that event, and what happens to every other user's notifications during that time?" |

### Phase 5 — Failure Modes

| Hint Level | Hint Text |
|-----------|-----------|
| Least specific (hint 1) | "Walk through each component in your HLD and ask: what breaks here, and what does the system look like to the end user when that happens?" |
| Moderate (hint 2) | "Think about: what happens if APNs starts returning 429 rate limit errors? What happens if your fan-out consumer group falls behind? What happens if a device token that APNs tells you is invalid is still in your database?" |
| Most specific (hint 3) | "The stale token problem is a real operational hazard: APNs returns HTTP 410 when a token is permanently invalid (app uninstalled, device wiped). How does your system react to that 410 response, and what happens to delivery attempts for that device until the token is purged?" |

---

## Extension Probe

*Ask this after the candidate has completed the core design. It should feel like a natural evolution of the problem, not a trick.*

> "Let's say the product team comes back with a new requirement: users should be able to configure notification preferences. Some users want to turn off 'like' notifications entirely. Some want to set quiet hours from 10pm to 8am in their local timezone. Some want to receive certain types of notifications only on their phone, not on their laptop. How does your current design need to change to accommodate this?"

**What to look for:** The candidate should identify that preferences need to be checked during fan-out (before writing delivery records, not after). A weak answer adds a preferences filter after delivery records are written — that wastes work. A strong answer inserts a preferences lookup in the fan-out service between "resolve device tokens" and "write delivery records," and notes that this lookup must be cached (because at 17K fan-out tasks/sec, a per-task database read for preferences would be the new bottleneck).

---

## Failure Mode Probe

> "What happens if the message queue falls behind — say the fan-out consumer group is processing 30 seconds behind ingestion and the lag is growing? Walk me through exactly what a user experiences, step by step."

**What to look for:** The candidate should trace the user experience through the system, not just describe the infrastructure state. "Notifications are delayed" is not sufficient. A strong answer says: "The publisher's ingestion call succeeds (the event is in Kafka), so the publisher sees no error. The user's friend sees no indication that the notification is queued. From the recipient user's perspective, they receive notifications with increasing latency — a like that happened 30 seconds ago arrives now, and that lag grows over time if we don't recover. Recovery requires scaling the fan-out consumer group horizontally, which Kafka supports by adding consumers to the group up to the partition count. If the partition count is the ceiling, we also need to add partitions." A good candidate also notes what to monitor: consumer group lag metric, with an alert threshold.

---

## Scaling Probe

> "This design works at 100 million users. How does it change at 1 billion users? What breaks first?"

**What to look for:** The candidate should identify that the device token store (currently sized at ~16GB) grows to ~160GB — still manageable. The delivery status store (currently at 4.5TB/30 days) grows to 45TB — still manageable with partitioning. The real inflection point is the fan-out layer: at 1B DAU with a 5× peak multiplier, you're generating ~435,000 per-device delivery tasks per second at peak. That requires significant horizontal scaling of the fan-out consumer group and the per-platform delivery worker pools. The celebrity problem also gets worse: a single event from a 100M-follower account triggers 100M fan-out tasks, which at 8,700 tasks/sec takes three hours to process — meaning you absolutely need a tiered fan-out strategy (push for normal accounts, lazy/pull for celebrity accounts) at this scale. A strong candidate identifies this inflection point and proposes the tiered fan-out without being prompted.

---

## Closing Question

*Ask this in the final 2 minutes of the session:*

> "What's the weakest part of your design? If you had another 30 minutes, what would you work on first?"

**What to look for:** Self-awareness is a signal. A candidate who says "I think the design is solid" without identifying a real gap is showing low self-awareness. A strong answer identifies a genuine weakness — "I didn't have time to properly design the retry strategy for the delivery workers — I know I need exponential backoff with jitter to avoid thundering herd on APNs, but I didn't work through the exact backoff parameters or the maximum retry budget" — and explains what they would do with more time. The weakest part of most candidates' designs is the token invalidation feedback loop (what happens when APNs returns 410) and the priority queuing system.

---

## Scoring Checklist (For Interviewer / AI)

Use this during the session to track evidence for each dimension. Circle Y or N in real time.

### Requirements Clarification (10%)

- [ ] Asked about delivery semantics (at-least-once / at-most-once / exactly-once)
- [ ] Asked about scale (DAU, notification volume, or QPS)
- [ ] Asked about latency SLA or urgency
- [ ] Asked about offline behavior / TTL
- [ ] Asked about platform support (iOS / Android / web) or device multiplicity

Score: 0 = none checked | 1 = 1–2 checked | 2 = 3–4 checked | 3 = all 5 or 4 with clear judgment about which to skip

### Capacity Estimation (10%)

- [ ] Derived QPS from DAU (not just stated it)
- [ ] Calculated per-device delivery task rate (not just per-notification)
- [ ] Estimated storage (payload + status records)
- [ ] Identified a bottleneck component from the math

Score: 0 = skipped | 1 = stated numbers without derivation | 2 = derivation chain present, bottleneck missing | 3 = full chain, bottleneck named, used to drive an architectural choice

### HLD Completeness (20%)

- [ ] Named ingestion API / notification service
- [ ] Named a durable queue (Kafka, SQS, or equivalent)
- [ ] Named a fan-out layer with responsibility explained
- [ ] Named a device token store with technology justified
- [ ] Named per-platform delivery workers (APNs / FCM)
- [ ] Named a delivery status store
- [ ] Drew or described end-to-end data flow with protocols labeled

Score: 0 = fewer than 3 components | 1 = 3–4 components, no full flow | 2 = all major components, partial flow or missing protocol labels | 3 = complete topology, labeled flows, technology choices justified

### API Design (10%)

- [ ] Defined at least one endpoint with HTTP method and path
- [ ] Showed request schema (body fields)
- [ ] Showed response schema (fields + HTTP status codes)
- [ ] Included an idempotency mechanism (idempotency key or conditional write)
- [ ] Defined device token registration or deregistration endpoint

Score: 0 = no API design | 1 = endpoint names only | 2 = request/response shape defined | 3 = full design with idempotency, error codes, and at least one lifecycle endpoint (register/deregister)

### Data Model (15%)

- [ ] Defined a device token schema with correct partition/primary key
- [ ] Defined a notifications table or collection
- [ ] Defined a delivery records table or collection
- [ ] Included at least one index with rationale
- [ ] Justified database technology against access pattern

Score: 0 = no data model | 1 = field names only, no keys or indexes | 2 = schemas with keys and at least one index | 3 = full schemas, index strategy, technology justified against access patterns, sharding or partitioning strategy noted

### Deep Dive Quality (25%)

- [ ] Chose a high-complexity component (fan-out, delivery reliability, or retry strategy)
- [ ] Identified the specific hard problem within that component
- [ ] Proposed a specific solution (not just "use more servers")
- [ ] Named the tradeoff in the chosen solution
- [ ] Named at least one alternative and explained why it was rejected
- [ ] Surfaced a failure scenario specific to the deep-dived component

Score: 0 = no deep dive or component indistinguishable from HLD | 1 = went deeper but no tradeoffs | 2 = specific solution with one tradeoff or alternative | 3 = chose the right component, surfaced the core hard problem, proposed solution with named tradeoffs, named and rejected alternatives

### Failure Modes (10%)

- [ ] Named at least one failure mode proactively (without being asked)
- [ ] Proposed a concrete mitigation (not just "add monitoring")
- [ ] Identified the SPOF in their design
- [ ] Discussed retry strategy with backoff semantics
- [ ] Named a cascade failure scenario (component A failing causes component B to fail)

Score: 0 = no failure modes | 1 = one failure mode named without mitigation | 2 = 2–3 failure modes with concrete mitigations | 3 = SPOF identified, cascade scenario named, retry strategy specified, monitoring/alerting mentioned

---

**Total Hints Given:** ___ × 0.5 = ___ point deduction from Deep Dive score

**Final Score:** ___ / 21 (minus hint deductions)
