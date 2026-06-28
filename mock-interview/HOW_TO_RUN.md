# How to Run a Mock Interview Session

This document covers four distinct modes for practicing system design. Each mode has a different purpose, a different cognitive load, and a different ideal time in your preparation arc. Read all four before deciding which to use on a given day.

---

## Mode 1: Solo Practice (45 Minutes)

### The Philosophy

The most important thing about solo practice is that you do not read the problem bank before you start. This sounds obvious, but the temptation to "just scan the topics" is real and corrosive. The moment you know that today's problem is about rate limiting, your brain starts loading relevant knowledge before you've had to demonstrate that you'd reach for it unprompted. That's not what a real interview tests. A real interview tests whether you can navigate uncertainty, ask the right clarifying questions, and produce a coherent design under time pressure — not whether you can perform a design you've already half-done in your head.

The solo session simulates that experience. Set a 45-minute timer before you open any problem file. Work on paper or in a blank document. Think aloud — narrate your reasoning even though no one is listening, because the act of narrating forces you to commit to reasoning chains rather than skating over gaps. When the timer ends, stop. Do not add one more thing. The discipline of stopping on time is part of what you're training.

### Timing Breakdown

The 45 minutes map to five phases. These are not hard rules — a real interview has natural variation — but they are useful guardrails until you develop your own rhythm.

**Phase 1 — Requirements Clarification (0–5 minutes)**

Before drawing a single box, you must understand what you are building. The problem statement will give you a surface description. Your job is to surface the constraints that will determine your architectural choices. Ask about scale (what is the DAU, what is peak QPS?), ask about delivery semantics (does every notification need to be delivered, or is best-effort acceptable?), ask about latency SLAs (is a 5-second delivery delay acceptable?), ask about consistency (if a user views their feed on two devices simultaneously, do they need to see the same data?), and ask about compliance or data residency if the problem involves user data.

The instinct to skip this phase and jump to drawing components is one of the most reliable signals of a weak candidate. Requirements clarification is not bureaucratic — it is how you learn which of three fundamentally different architectures is correct for this problem. A notification system for a bank has different durability requirements than a notification system for a social game. You cannot know which one you are building until you ask.

**Phase 2 — Capacity Estimation (5–10 minutes)**

With requirements in hand, do a quick back-of-envelope calculation. Start from DAU, derive writes per second and reads per second, estimate storage growth per year, estimate bandwidth. The goal is not a precise number — it is a sense of the bottleneck. Are you I/O bound? Is storage the constraint? Will you need sharding? Is this the kind of scale where a single database handles it easily, or the kind where you need a distributed data layer from day one?

Do the math on paper. The numbers you derive will constrain every choice you make in the next 35 minutes. If you skip this step, you may find yourself designing an in-memory fan-out service when the math would have told you that 50TB/year of notification history requires a tiered storage strategy from the start.

**Phase 3 — High-Level Design Sketch (10–20 minutes)**

Now draw. Start with the client and work your way through the system. Name each component clearly. Label the data flows. Indicate protocols where they matter (REST, gRPC, WebSocket, APNS). Do not go deep on any single component yet — your goal at this phase is to show that you understand the full end-to-end flow. A strong HLD identifies the major services, the primary data store, any queues or event buses, and the external integrations.

Resist the urge to immediately start designing the database schema or the retry logic. Those belong in the deep dive. Premature depth on one component at the cost of missing another is a common failure mode.

**Phase 4 — Deep Dive on 1–2 Components (20–40 minutes)**

Pick the two most interesting or most complex components in your design and go deep. The best deep dives are ones where you can show the problem is harder than it looks, propose a solution, name the tradeoff, and name the alternative you considered and rejected. For a notification system, this might be the fan-out service (how do you handle a user with 20M followers without blocking?) and the delivery layer (how do you guarantee at-least-once delivery to APNs without thundering herd on retries?).

As you go deep, surface the failure modes naturally. "If the message queue falls behind, here's what happens and how we handle it" is more impressive than waiting to be asked.

**Phase 5 — Failure Modes and Scaling Questions (40–45 minutes)**

In a real interview, this is when the interviewer probes. In solo practice, prompt yourself. Ask: what happens if this component fails? What breaks at 10x current scale? Where is the single point of failure? What did I not have time to design? Be honest. A strong candidate can identify the weakest part of their own design — it signals that they understand the system well enough to know where the dragons are.

### After the Session

Score yourself using EVALUATION_RUBRIC.md **before** reading SOLUTION_GUIDE.md. This matters. If you read the solution first and then score yourself, you will unconsciously inflate your score by retrofitting what you read. Score yourself on what you actually said and drew during the session, then read the solution and see where you diverged.

---

## Mode 2: Partner Session

### Setup

A partner session requires one person playing the interviewer and one person solving. The session runs 45 minutes for the design portion followed by a 15-minute structured debrief. The two roles alternate across sessions — never do more than two consecutive sessions in the same role, because each role teaches something different.

The interviewer should read the problem file and SOLUTION_GUIDE.md before the session. This is not cheating — the interviewer's job is to probe intelligently, not to solve the problem. You cannot probe intelligently for depth if you don't know what depth looks like. The interviewer should also read EVALUATION_RUBRIC.md so that during the session they can mentally note which dimensions are being addressed and which are being skipped.

During the session, the interviewer does not coach. When the candidate asks "is this right?", the answer is always "what do you think?" or "walk me through your reasoning." The interviewer's interventions are probes: "how does that component handle failure?", "what's the alternative you considered?", "you said eventual consistency is fine here — under what scenario does that cause a user-visible problem?".

### What the Interviewer Should Focus On

The interviewer's primary job is not to grade answers — it is to create the conditions for the candidate to demonstrate depth. A good interviewer asks open-ended follow-up questions when the candidate makes a claim, asks about tradeoffs when the candidate makes a choice, and asks for specifics when the candidate is being vague. "We'll use a message queue" is a claim. "Why Kafka and not RabbitMQ for this use case?" is a probe. "What happens when the consumer falls behind?" is a deeper probe.

The interviewer should not accept hand-waving. When a candidate says "and then we'd have some cache layer", that is the moment to ask what the cache key looks like, what the eviction strategy is, and whether the cache can serve stale data or must always be consistent with the database. The candidate's ability to answer these questions without hesitation is what separates a 2 from a 3 on any dimension.

### How to Give Useful Feedback

The debrief should reference EVALUATION_RUBRIC.md explicitly. Go through each of the 7 dimensions. For each one, give the candidate a specific thing they said or drew that earned credit, and a specific thing that was missing or weak. Avoid global statements like "your HLD was pretty good" — they give the candidate nothing to work on. "You identified the fan-out problem and proposed a push-to-pull fallback for high-follower accounts, but you never named an alternative approach or explained why you rejected pull-only delivery" is feedback the candidate can act on in the next session.

Use the anti-pattern list in the rubric explicitly. If the candidate skipped capacity estimation, name it. If the candidate's API design had no idempotency mechanism, say so. If the candidate chose the wrong component to deep-dive on (spent 15 minutes on the database schema when the real complexity was the delivery layer), explain why that was a weak choice and what a stronger choice would have been.

---

## Mode 3: AI Interview (Claude Code)

### How to Invoke

Open the repository in Claude Code. You can start a session in one of two ways. The most direct is to type `/interview [problem name]` where the problem name matches a file in the `problems/` directory — for example, `/interview notification-system`. If you want Claude to select the problem (which adds an element of surprise closer to a real interview), type `/interview` with no argument and Claude will pick based on which problems you haven't recently attempted.

Once the session begins, Claude will read the problem prompt aloud (in text form) and ask you to confirm you're ready. From that moment, the session is live. Claude plays a strict interviewer — it does not validate your answers mid-session, it does not confirm whether your direction is correct, and it does not offer suggestions unless you ask (at a cost to your score, noted in the rubric). When you ask "is this approach right?", Claude will say "what makes you confident in that choice?" and wait.

### What Makes the AI Interview Different from Solo

The primary difference is real-time probing. In solo practice, you decide when to surface tradeoffs and failure modes. In an AI session, Claude decides — and it will always ask about the things you skipped. If you don't ask about consistency requirements in Phase 1, Claude will probe later: "you chose a SQL database for this — what consistency model does that give you, and is that what the requirements need?" You cannot choose to gloss over a dimension because Claude will not let you.

The second difference is harder to quantify: the experience of articulating your reasoning in real time to an interlocutor, even an AI one, changes the cognitive texture of the exercise. It is more stressful and more revealing than staring at a blank doc alone. That stress is productive — it surfaces the gaps between what you think you understand and what you can actually communicate under pressure.

After the session, Claude scores your performance against the rubric and writes the results to `SESSION_SCORECARD.md` in the problem directory. It will fill in specific evidence quotes, anti-patterns observed, and a recommended drill for your top gap.

### Practical Notes

Do not ask Claude to explain concepts mid-session — that ends the interview and begins teaching mode. If you're in an interview session and you blank on something, name it: "I know there's a technique for handling backpressure in this scenario but I'm blanking on the details — let me reason from first principles." That is far better than silence, and it is how a real senior engineer handles a gap.

---

## Mode 4: Deep Dive / Learning Mode

### What This Mode Is For

Learning mode is not an interview. It is a study session. The goal is not to practice performing under pressure — it is to close specific knowledge gaps. Use it when you've done a solo or AI session and your scorecard identified a particular dimension or concept you need to strengthen before you attempt the problem again.

### How to Run It

Start by reading PROBLEM.md, then close it and sketch the architecture on paper. Do not look anything up yet — you want a baseline of what you currently know. Then open SOLUTION_GUIDE.md and read it section by section, pausing after each section to compare it to your sketch. Where they diverge, ask yourself why. Sometimes your divergence reflects a legitimate alternative approach — note it. Sometimes it reflects a gap — write that down too.

After reading the solution, read LESSONS_LEARNED.md. This file contains the failure patterns and real interview mistakes that are most common for this problem. Pay particular attention to the "General Pattern Mistakes" section — these are not problem-specific, they appear across many problems, and eliminating them from your repertoire has compounding value.

Finish the session by writing three things you didn't know before in your personal notes. Not three things you saw in the solution — three things that surprised you or that you would not have produced on your own. If you can't find three, you probably need to go back and read more carefully.

### When to Use Learning Mode vs Solo Practice

Use learning mode when you've identified a specific gap and you need to fill it before your next performance session. Use solo practice (or AI interview) to measure your current ability level under real conditions. Alternating between the two — learn, then practice, then learn to close the gap revealed by the practice, then practice again — is the most effective preparation structure.

---

## Signs You Are Ready to Interview

You are ready for a real interview when the following are consistently true, not occasionally true.

You can derive QPS from DAU from scratch in under three minutes, without looking up a reference table. This means you've internalized the conversion chain (DAU to DAU/day to DAU/second to read/write ratio to peak multiplier) so well that it runs automatically.

You automatically ask about consistency requirements before drawing any database component. This means you've internalized that the consistency model is a prerequisite, not an afterthought, and your first instinct when you see "database" in a design is to ask what consistency semantics the use case actually requires.

You can name two alternatives to every major component in your HLD without being prompted. You don't choose Kafka — you choose Kafka over RabbitMQ and Kinesis, and you can explain in one sentence why for this specific problem.

You can articulate failure modes for the top three components in any design you produce before the interviewer asks. This is the sign of genuine systems thinking rather than pattern-matching — you see the fragile points because you understand how the components interact, not because you memorized a failure modes list.

Your session scorecards show no dimension averaging below 1.5/3 across the last five sessions.

---

## Signs You Are Becoming a Better Engineer (Not Just Better at Interviews)

These are harder to measure but more important to notice.

You catch yourself thinking about failure modes in your daily work before they happen — when you're reviewing a PR or designing an internal service, you instinctively ask "what happens if this queue fills up?" or "what happens if this database replica falls behind?" That's the habit of failure mode analysis transferring from the interview context into your actual engineering.

You ask "what's the bottleneck?" before optimizing anything, because you've internalized from capacity estimation exercises that optimization effort applied to a non-bottleneck produces no meaningful improvement. You've stopped optimizing by intuition.

You find yourself asking clarifying questions more naturally in product discussions — not because you learned to do it in interviews, but because you've practiced the five-category requirements framework enough times that the gaps in a requirements document feel obvious and uncomfortable.

These are the durable skills. The interview prep is the mechanism; becoming a better systems thinker is the actual goal.
