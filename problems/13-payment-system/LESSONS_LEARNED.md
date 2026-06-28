# Lessons Learned — Payment System

---

## Real Interview Stories

### Story 1 — Stripe, L4 (2024)

The interviewer asked the candidate to design the payment retry mechanism. The candidate proposed: "If the PSP call fails, retry up to 3 times with exponential backoff." The interviewer asked: "What if the first attempt succeeded but you never received the confirmation?" The candidate said: "Then we retry and the PSP will tell us it already processed." 

The interviewer probed further: "How does the PSP know it already processed it? What specifically does your retry request include that tells Stripe 'this is a retry of request X'?" The candidate couldn't answer — they had implicitly assumed the PSP would handle deduplication, but had no mechanism to carry the deduplication key. The interviewer walked through the Idempotency-Key header pattern: the client-generated UUID that travels from client → your API → PSP, ensuring the PSP deduplicates correctly.

The candidate failed to advance past L4 partly because of this. The feedback was: "Didn't understand the semantics of distributed retry in financial systems." **The lesson: know that idempotency keys must flow end-to-end, from client header through every intermediate service call to the PSP. The mechanism is the UUID in the HTTP header, stored server-side with the response, for 24 hours.**

---

### Story 2 — Uber, Senior Engineer (2023)

The interviewer asked the candidate to walk through the failure case where the server charges the PSP successfully, but then the server crashes before recording the transaction in the ledger. The candidate proposed: "We check if the payment ID exists in the ledger on startup and re-process any missing ones." 

The interviewer asked: "How do you know the payment was captured if the ledger doesn't have it?" The candidate said: "We'd check the database." The interviewer: "The server crashed — the database write never happened. How do you find out the charge occurred?"

The candidate was stuck. The correct answer is the SAGA's audit trail plus the PSP's own transaction history. The payment orchestrator records its step-by-step progress durably (Temporal persists workflow state). The reconciliation service compares the PSP's settlement file against the ledger nightly — any PSP charge without a ledger entry triggers an investigation. The candidate had no reconciliation component in their design at all.

**The lesson: reconciliation is not an optional nice-to-have. It is how you discover all the ways your system fails silently. Every payment company — Stripe, Uber, Airbnb — has a reconciliation process. Not mentioning it is a signal that you haven't thought about financial correctness at scale.**

---

### Story 3 — Airbnb, E5 (2024)

The candidate designed a payment system with a single monolithic "PaymentService" that handled auth, capture, ledger write, and payout notification in a single function call. The interviewer asked: "Walk me through what happens if the host's payout notification call at the end of your function fails after the guest has been charged."

The candidate said: "The function returns an error to the caller." The interviewer: "So the guest was charged but the host was never notified and never paid?" Candidate: "We'd retry the whole function." Interviewer: "But the function starts by charging the card. If the guest was already charged, your retry re-charges them."

The candidate spent 10 minutes working toward a solution. They eventually landed on "check if the charge exists before charging" — which is partial idempotency, not a SAGA. The interviewer guided them toward the compensating transaction pattern: the payout notification failure is handled by a separate, retriable async step that cannot double-charge because the charge step already completed.

**The lesson: monolithic payment functions are fragile. The SAGA pattern exists precisely because a multi-step process that can fail at any step needs each step to be independently retriable and compensatable. The charge step, the ledger write step, and the payout notification step are separate concerns with separate failure modes.**

---

## General Pattern Mistakes

**Mistake 1: Proposing AP (eventual consistency) for the ledger.** Payments are textbook CP (consistency over availability). If your system must choose between "temporarily unavailable" and "recording a wrong ledger entry," it must choose unavailable. A ledger entry that double-counts $50 is worth more downtime than a brief payment outage. Candidates who propose eventual consistency for financial data haven't internalized the CP requirement.

**Mistake 2: Ignoring the timeout ambiguity entirely.** Many candidates handle clear success (200) and clear failure (402) but ignore the timeout case. This is the most dangerous case — you don't know if the charge happened. The solution is idempotent retry with PSP-side deduplication via the Idempotency-Key header. Not addressing this is a red flag at Stripe, Uber, and Airbnb.

**Mistake 3: Double-entry ledger as just a "transactions table."** Proposing a table with columns `(from_account, to_account, amount)` and calling it a ledger. This is not double-entry bookkeeping. Double-entry means every transaction generates separate DEBIT and CREDIT entries that sum to zero. This property enables accounting invariants: at any time, sum of all CREDIT entries - sum of all DEBIT entries = net balance, auditable by any external party.

**Mistake 4: Direct Kafka publish without the outbox pattern.** Proposing: "write payment to DB, then publish event to Kafka." The failure case: DB write succeeds, server crashes, event never published. Payout service never fires. Seller never gets paid. The transactional outbox pattern (write event to DB table in same transaction, background poller publishes it) solves this. Not knowing the pattern by name is acceptable; not knowing the failure mode of direct publish is not.

**Mistake 5: No SAGA compensation design.** Candidates who describe SAGA as "try each step, if it fails roll back" without specifying WHAT each compensation action is. "Roll back" is not a transaction — it's a new action. If step 3 fails, "roll back step 2" means "issue a PSP refund via the PSP refund API, then write a REFUND ledger entry." Every step must have a concrete named compensation action.

---

## Self-Assessment Checklist

**Fundamentals:**
- [ ] Clarified consistency model (CP) before designing
- [ ] Clarified PSP timeout semantics before designing
- [ ] Never mentioned storing raw card numbers; immediately named PSP tokenization

**Idempotency:**
- [ ] Named idempotency keys (UUID in header, server-side cache with response)
- [ ] Explained the table schema: (key, status, response_body, locked, expires_at)
- [ ] Explained idempotency on BOTH charge endpoint AND refund endpoint

**SAGA:**
- [ ] Named all steps of the payment saga (authorize → capture → ledger → payout queue)
- [ ] Named the compensating transaction for at least two failure points
- [ ] Named Temporal (or equivalent) as the orchestration engine for durable execution

**Ledger:**
- [ ] Proposed double-entry format (DEBIT + CREDIT entries summing to zero per transaction)
- [ ] Used integer cents, not FLOAT, for money amounts
- [ ] Named the table as append-only (no UPDATE, no DELETE)

**Reconciliation:**
- [ ] Included reconciliation service in the design
- [ ] Explained what it compares (internal ledger vs. PSP settlement file)
- [ ] Named two categories of discrepancies: phantom charges, missing settlements

**Failures:**
- [ ] Addressed PSP timeout with specific solution (idempotent retry with same key)
- [ ] Addressed server crash after capture (SAGA state recovery + reconciliation)
- [ ] Addressed the outbox pattern for reliable event publishing

---

## Remediation Targets

**If you didn't know idempotency keys:** Read the Stripe idempotency blog post (stripe.com/blog/idempotency) and Brandur Leach's article (brandur.org/idempotency-keys). The key insight is that idempotency is a per-boundary property — every external call (PSP, downstream service, DB write) needs its own idempotency guarantee.

**If you didn't know double-entry bookkeeping:** Look up double-entry accounting. The invariant: every transaction generates equal debits and credits. A $50 purchase generates: DR Buyer $50 | CR Platform $5 | CR Seller $45. Sum of CR = Sum of DR = $50. This invariant makes it impossible to silently lose money in the ledger.

**If you didn't know the outbox pattern:** Read about the transactional outbox pattern (microservices.io/patterns/data/transactional-outbox.html). The key: write events to a table in the same DB transaction as your business data, then publish them asynchronously. Ensures events are never lost, even if the process crashes.

**If you couldn't describe SAGA compensating transactions:** Practice end-to-end failure narration. Start from a happy path, pick a step to fail, and walk through each compensating action. "The capture API call times out. We retry with same idempotency key. PSP returns the original result. If success: continue. If failure: we void the authorization via PSP void API and write a VOID entry to the ledger."
