# Mock Interview Script — Payment System

**For the interviewer only. Do not share with the candidate.**

---

## Pre-Session Setup

**Target level:** Senior engineer (L5/ICT3 equivalent)
**Expected duration:** 45 minutes
**This is an Advanced problem.** Expect depth on consistency, idempotency, and failure scenarios. Shallow designs should score in the 2–3 range.
**Red flags:** AP for ledger, no idempotency key mechanism, floating-point money, no reconciliation, SAGA without compensation actions
**Green flags:** CP first, idempotency key schema named, outbox pattern, double-entry ledger with integer cents, PSP timeout handling, reconciliation service

---

## Opening

"I'd like you to design a payment system for an e-commerce marketplace. Buyers pay sellers. The platform takes a 10% commission. Design the system that processes these payments end-to-end — from the buyer clicking 'Pay' through the seller receiving their payout. I want you to assume this handles 1 million transactions per day at an average of $50 each."

*Let the candidate ask clarifying questions. If they rush to design, say: "Before we design anything, what questions do you have?"*

---

## Phase 1: Requirements (0–8 min)

**The question that separates senior from mid-level:**
"What happens if the payment processor returns a timeout — no response within 30 seconds?"
- If the candidate doesn't ask this themselves, ask it now as a requirement clarification
- Expected answer: "We need to retry with an idempotency key so the PSP can deduplicate"
- Weak answer: "We retry" (no mention of deduplication)
- Bad answer: "We assume it failed and ask the user to try again" (assumes failure, may have been a charge)

**Other clarifying questions the candidate should ask:**
- Consistency model (CP vs AP)?
- Do we store card numbers? (No — PSP tokenization)
- Single country or global? (multi-currency complexity)
- Need reconciliation/audit trail? (Yes — regulatory)

**Gate:** Candidate must have stated CP, PSP tokenization, and timeout handling strategy before proceeding.

---

## Phase 2: Estimation (8–13 min)

**Guide candidate through key numbers:**
- 1M transactions/day ÷ 86,400 = **11.6 writes/sec avg, ~58 peak**
- Each transaction → 3 ledger entries (buyer debit, seller credit, platform credit)
- 3M ledger entries/day × 200 bytes = **600 MB/day** of ledger data
- Idempotency keys: 1M × 500 bytes = **500 MB/day** (expire after 24h, steady-state)

**Key probe:** "You have 11.6 writes/sec average. Does that number concern you?" Answer: No — it's very low. The challenge is not scale, it's correctness. This problem is about consistency and failure recovery, not about throughput.

---

## Phase 3: HLD (13–28 min)

**Expected components:**
1. Payment API with idempotency key validation
2. Payment Orchestrator (SAGA coordinator)
3. PSP Adapter with retry
4. Ledger Service
5. Payout Service (async)
6. Reconciliation Service

**Critical probe — idempotency:** "Walk me through exactly how your idempotency system works. Client retries the same payment — what happens step by step, including what's in your database?"
- Expected: idempotency_keys table with (key, response_status, response_body, locked, expires_at); on retry, return stored response verbatim

**Critical probe — SAGA:** "You're in the middle of a payment. PSP has captured the charge. Your ledger write fails. What happens next?"
- Expected: compensating transaction — issue PSP refund API call, write REFUND ledger entry, mark payment as FAILED
- Weak: "We retry the ledger write" (doesn't address what happens if it keeps failing)
- Bad: "Roll back the transaction" (can't roll back a PSP capture — it's an external system)

**Critical probe — ledger:** "Walk me through exactly what rows get written to the ledger for a $50 transaction with 10% commission."
- Expected: DEBIT Buyer $50, CREDIT Platform $5, CREDIT Seller $45 — three rows in same transaction

---

## Phase 4: Deep Dive (28–40 min)

Choose the weakest area:

**Option A — PSP Timeout:**
"Walk me through the complete sequence of events when Stripe returns a timeout. Include: what your code does immediately, what data is in what state, what happens after the timeout, and how you ensure the buyer is not charged twice."

**Option B — Outbox Pattern:**
"After a payment completes, you need to trigger a payout notification to the seller and a receipt email to the buyer. Walk me through how you ensure both of these happen reliably even if your payment service crashes immediately after writing to the database."
Expected: transactional outbox (write event to outbox table in same DB transaction; background poller publishes to Kafka; marks published)

**Option C — Reconciliation:**
"Your nightly reconciliation job runs and finds 50 transactions in Stripe's settlement file that don't have matching entries in your ledger. Walk me through what that means, how it happened, and what your process is for investigating and resolving each one."
Expected: phantom charges (PSP charged, ledger missing = SAGA failure), investigation via Temporal workflow history, resolution = write retroactive ledger entry or issue refund

---

## Phase 5: Failure & Scaling (40–44 min)

**Probe:** "You're processing Black Friday traffic — 10× normal volume, 116 writes/sec peak. Your PostgreSQL ledger primary goes down during peak. Walk me through what happens to in-flight payments and your recovery plan."
Expected: in-flight SAGAs resume when DB recovers (Temporal persists state); brief window of payment failures; replica promotion < 30 sec; SAGA retries complete automatically

**Probe:** "A buyer reports being charged twice for the same order. Walk me through how you investigate."
Expected: look up order_id in payments table → check idempotency_keys for any duplicate keys → compare with PSP transaction history → if double-charge confirmed, issue refund; if idempotency worked correctly, client is confused (show them payment history)

---

## Closing

"Two questions:
1. Where in this design is money most at risk of being silently lost?
2. If you had one more hour, what one component would you spend it on?"

Strong answers: money most at risk = the window between PSP capture and ledger write (SAGA step 2→3 gap). Component to improve: reconciliation automation to reduce manual ops review time.

---

## Scoring Checklist

| Area | Weight | 1 (Poor) | 3 (OK) | 5 (Strong) |
|------|--------|----------|--------|------------|
| Requirements clarification | 10% | Didn't ask about timeouts or consistency | Asked about consistency, missed timeout | Asked about PSP timeout, CP semantics, PCI DSS |
| HLD completeness | 30% | Missing idempotency or SAGA or ledger | All three present, reconciliation missing | All five components with outbox, reconciliation explained |
| LLD depth | 30% | No schema, floating-point money | Schema present with integer cents | Double-entry ledger schema, idempotency table with locked field, outbox table |
| Tradeoff reasoning | 20% | Choices made without justification | CP vs AP mentioned | SAGA vs 2PC tradeoff, outbox vs direct publish tradeoff, integer vs float for money all justified |
| Communication clarity | 10% | Jumped around, hard to follow | Logical structure | Clear "if X fails, then Y happens" narration throughout |
