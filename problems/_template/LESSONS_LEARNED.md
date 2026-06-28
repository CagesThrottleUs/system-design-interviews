# Lessons Learned — Common Failure Patterns in System Design Interviews

This document captures the recurring patterns that cause otherwise strong engineers to underperform in system design interviews. Use it after each session to audit whether any of these patterns appeared in your work.

---

## Real Interview Stories

The following accounts are composite reconstructions based on common interview debriefs and publicly shared post-mortems from engineering interview processes. Names are fictional; the failure patterns are real.

| Company | Level | Mistake Made | Consequence in Session | Fix |
|---------|-------|--------------|----------------------|-----|
| Meta | E5 | Candidate jumped directly to database schema design for a notification system without asking about delivery semantics. Assumed strong consistency was required because "notifications are important." | Interviewer asked about the fan-out problem 20 minutes in. Candidate's strong-consistency design collapsed because it couldn't handle 10M writes/sec. Had to backtrack and redesign under time pressure, finishing with a disjointed architecture. | Ask delivery semantics as the first question. "Do we need at-least-once, at-most-once, or exactly-once delivery?" determines the entire architecture. Strong consistency and high-throughput fan-out are in tension; you need to know which matters more before you draw a box. |
| Apple | ICT4 | Candidate correctly identified Kafka as the right queue technology but spent 12 of the 20 HLD minutes explaining Kafka's internal architecture (log segments, ISR, controller election) rather than describing the notification system. | Interviewer had to interrupt and ask "how does data flow from a publisher triggering a like event to the user's phone?" Candidate hadn't drawn the end-to-end flow at all. HLD phase ended with the architecture incomplete. | HLD is about your system, not about the technology's internals. One sentence naming and justifying the technology choice; the rest of the time on your system's components and data flows. |
| Google | L5 | Candidate proposed a relational database for device token storage without considering access patterns. When probed on how they'd query tokens for a given user, described a SELECT with a WHERE clause. When probed on write throughput (token refresh on every app launch for 250M devices), had no answer for how a single SQL instance handles that. | Interviewer pressed for 5 minutes on "how does this scale?" Candidate proposed sharding but couldn't explain the shard key or how cross-shard queries would work for their described schema. Interviewer moved on without resolution. | Always state the primary access pattern before naming a database technology. "The query is: give me all tokens for user X, so I need a key-value store that can scan a user's row efficiently — Cassandra fits this pattern." |

---

## General Pattern Mistakes

### Pattern 1: Jumping to Code-Level Detail in HLD

**What happens:** The candidate begins the high-level design phase by writing a database schema, defining exact field types, listing indexes, or describing an algorithm. Within the first 10 minutes of HLD, they are knee-deep in the data model for a single component while three other components haven't been named yet.

**Why wrong:** High-level design is about system topology — how components relate, communicate, and depend on each other. When you go deep on one component before establishing the overall shape of the system, you signal to the interviewer that you lack the discipline to zoom out. More practically, you may spend 15 minutes perfecting a data model for a component that turns out to be the wrong choice once you discover (5 minutes later) that the scale requirements demand a different architecture. HLD depth before breadth is a form of premature optimization.

**Fix:** Enforce a personal rule: in the first 10 minutes of HLD, name every major component in your design and connect them with arrows. Label each arrow with a protocol or data type. Only after you have the full topology on the whiteboard should you start going deeper on any single component. If the interviewer asks you to go deeper early, name what you're skipping: "I want to note that I haven't addressed the delivery worker layer yet — I'll come back to it after we finish the fan-out service."

---

### Pattern 2: Stating Numbers Without Derivation

**What happens:** During capacity estimation, the candidate says "so we'll have about 3,500 writes per second" and moves on. No calculation visible. No derivation of where that number came from. The number might even be roughly correct, but the interviewer has no way to know whether the candidate derived it from first principles or guessed.

**Why wrong:** Interviewers don't care about the number — they care about whether you can derive it. The derivation chain (DAU → events/user/day → events/sec → peak multiplier → QPS) is the skill being tested. A candidate who states numbers without showing the chain cannot be trusted to estimate systems they haven't seen before, which is exactly what senior engineers have to do regularly. Additionally, the derivation chain is how you identify the bottleneck — you can't find the bottleneck if you skip the math.

**Fix:** Write out every step of the derivation chain, even if it feels tedious. "100M DAU, 3 notifications per user per day = 300M notifications per day. 300M / 86,400 seconds ≈ 3,500 writes per second. With 5× peak, that's 17,500 writes per second at peak." Then circle the bottleneck tier and say why it's the bottleneck. The whole thing takes two minutes and demonstrates a level of rigor that compounds across the rest of your design.

---

### Pattern 3: Treating Failure Modes as an Afterthought

**What happens:** The candidate completes a perfectly functional HLD and deep dive, and then when the interviewer asks "what happens if this component fails?", the candidate looks at the diagram for the first time and starts improvising. They say "we'd add retries" without specifying the retry strategy. They say "we'd have a backup" without explaining what consistency model that backup uses or how failover is triggered.

**Why wrong:** This pattern signals that failure modes were not part of the design process — they were tacked on after the fact. A senior engineer at Apple or Google thinks about failure modes while designing, not after. The components they choose, the protocols they use, and the boundaries they draw are all shaped by their understanding of what can go wrong. When failure mode thinking is deferred, it often produces inconsistencies: the candidate proposes a synchronous database write in the hot path, gets asked "what if the database is down?", and realizes their ingestion path has no durability guarantee.

**Fix:** As you draw each component, ask yourself: "what fails here, and what does that do to the system?" Write failure annotations directly on your HLD diagram if it helps. When you propose a queue, say "the queue gives us durability — if the delivery worker crashes, the message is not lost." This shows that failure mode thinking is integrated into your design process, not a bolt-on.

---

### Pattern 4: Choosing the Deep Dive Component for the Wrong Reason

**What happens:** When given the opportunity to choose which component to deep-dive, the candidate picks the component they're most comfortable with rather than the component that is most architecturally interesting or most relevant to the problem's core challenge. In a notification system, this looks like spending 15 minutes on the notification payload schema (a simple CRUD model) while not touching the fan-out service (where all the interesting scalability challenges live).

**Why wrong:** The deep dive is your opportunity to demonstrate that you can identify the hardest part of a system and reason through it. Choosing the easy component signals either that you don't know where the complexity is (which is a gap in your systems thinking), or that you're avoiding the hard part (which signals intellectual cowardice). Neither is a good signal. Interviewers know where the complexity is in every problem they give — they are watching to see if you find it too.

**Fix:** Before choosing a component for the deep dive, ask yourself: "what is the problem in this system that is hardest to solve at scale?" That component is your deep dive. For a notification system, it's the fan-out service. For a feed system, it's the ranking and aggregation pipeline. For a rate limiter, it's the distributed counter and its consistency guarantees. Go where the dragons are.

---

### Pattern 5: Using Vague Technology Names Without Justification

**What happens:** Throughout the design, the candidate names technologies as if they were magic solutions. "We'll use Redis for caching." "We'll use Kafka for the queue." "We'll use Cassandra for the database." Each technology choice is stated as a conclusion without an argument. When the interviewer asks "why Kafka and not RabbitMQ?", the candidate can only say "it's more scalable" without being able to name the specific property of Kafka that makes it the better choice for this problem.

**Why wrong:** The technology name is irrelevant. What the interviewer is evaluating is whether you understand the properties of the technology that make it the right fit. "We'll use Kafka because its durable log model lets us replay notification events during an incident without losing messages, and its consumer group semantics let us scale the fan-out service horizontally without coordination" is a 3/3 answer. "We'll use Kafka because it's scalable" is a 1/3 answer.

**Fix:** Train yourself to always follow a technology choice with a "because" clause that references either (a) an access pattern the technology serves well, (b) a scalability property the problem requires, or (c) an operational tradeoff you're accepting. You should also name the runner-up alternative and explain why you rejected it.

---

## Self-Assessment Checklist

Answer these questions honestly within 10 minutes of completing a session, before reading the solution guide.

1. Did I ask about delivery semantics (at-least/at-most/exactly-once) before choosing any queue or database technology?

2. Did I estimate QPS from DAU before drawing the architecture — and did I use that estimate to make at least one architectural choice?

3. Did I name every major component in the end-to-end data flow before going deep on any single component?

4. Did I label every arrow in my HLD diagram with a protocol or data type?

5. Did I design at least one API endpoint with a request/response schema and an idempotency mechanism?

6. Did I justify my database choice(s) against the primary access pattern, not just against vague "scalability" intuition?

7. Did I choose a deep-dive component that is architecturally interesting for this problem — not just the component I'm most comfortable with?

8. Did I surface at least one failure mode during the design process (not just in response to an interviewer question)?

9. Can I name two alternatives to my major design choices and explain in one sentence why I rejected each?

10. Did I stop and summarize my design for 60 seconds before diving into the deep dive, so the interviewer could follow the thread?

---

## Remediation Targets

| Checklist Item Failed | Gap Category | Recommended Drill |
|----------------------|-------------|-------------------|
| Q1 — Didn't ask delivery semantics | Requirements clarification | Drill: practice the 5-category requirements framework on 3 new problems without drawing anything until all 5 categories are addressed |
| Q2 — Skipped or disconnected capacity estimation | Capacity estimation | Drill: estimate QPS, storage, and bandwidth for 5 different problems from scratch using only DAU as the starting number |
| Q3 — Went deep before establishing breadth | HLD discipline | Drill: set a timer for 8 minutes, name all components and draw all arrows before that timer ends — if you haven't finished the full topology, stop and restart |
| Q4 — Unlabeled arrows | HLD communication | Drill: review the last 3 HLD diagrams you drew and add protocol/data labels to every arrow |
| Q5 — No API design or missing idempotency | API design | Drill: design 3 APIs for write-heavy operations with idempotency keys, pagination, and error codes |
| Q6 — Database choice not justified | Technology selection | Drill: for each of the 6 major database types (relational, wide-column, document, key-value, time-series, graph), write down the access pattern it serves best and the access pattern it handles worst |
| Q7 — Wrong deep dive choice | Systems thinking | Drill: for each problem in the problem bank, identify the "hardest component" before looking at any solution — compare your choice against SOLUTION_GUIDE.md |
| Q8 — No failure modes in design | Operational thinking | Drill: for the last HLD you designed, identify 3 failure modes and write the detection method, mitigation, and residual risk for each |
| Q9 — Cannot name alternatives | Technology breadth | Drill: for each major technology in the problem (queue, cache, database), write down 2 alternatives and the specific tradeoff that makes them worse for this problem |
| Q10 — No summary before deep dive | Communication | Drill: practice the 30-second system summary: name all components, name the primary data flow, and name the hardest problem — all in 30 seconds |
