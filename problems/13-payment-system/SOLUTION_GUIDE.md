# Solution Guide — Payment System

Read after your attempt. If you haven't attempted yet, close this file.

---

## Component Map

| Component | Role | Technology Choice | Why |
|-----------|------|-------------------|-----|
| Payment API | REST endpoint receiving payment intent | Stateless Java/Go service | Horizontal scale; validates idempotency key on every POST |
| Idempotency Service | Deduplicates retry requests | PostgreSQL table keyed on (idempotency_key) | ACID — the idempotency table write must be atomic with the payment record creation |
| Payment Orchestrator | Coordinates SAGA: auth → capture → ledger → payout queue | Temporal.io workflow | Durable execution, automatic retry with compensation, visible state |
| PSP Adapter | Wraps Stripe/Adyen API calls | Adapter pattern, retry + exponential backoff | External boundary; timeouts must be handled correctly |
| Ledger Service | Immutable double-entry ledger | PostgreSQL append-only + partitioned by date | ACID transactions; never UPDATE, only INSERT; audit requirement |
| Payout Service | Batch seller payouts | Background scheduler + bank transfer API | T+1/T+2 settlement; not on the payment hot path |
| Outbox | Reliable event publishing | Transactional outbox in same DB as payment | Prevents lost events when app crashes after DB write but before publish |
| Reconciliation Service | Nightly ledger vs. bank statement comparison | Spark/batch job | Large-scale set comparison; runs overnight off hot path |

---

## Architecture Diagram

```
PAYMENT FLOW (the happy path and failure paths)
──────────────────────────────────────────────────────────────────────

  Buyer
    │  POST /payments { amount, currency, card_token, idempotency_key }
    ▼
┌──────────────────┐
│  Payment API     │
│  (stateless)     │
└──────────────────┘
    │
    │  1. Check idempotency key (upsert into idempotency table)
    ▼
┌──────────────────┐
│  Idempotency DB  │  ──── key already exists? Return cached response.
│  (PostgreSQL)    │  ──── key new? Continue.
└──────────────────┘
    │
    │  2. Create payment record (status: PENDING)
    │  3. Publish to payment queue (via outbox)
    ▼
┌──────────────────────────────────────────────────────┐
│  Payment Orchestrator (Temporal Workflow)             │
│                                                       │
│  Step 1: PSP Authorize                                │
│    → call PSP Adapter with card_token + amount        │
│    → PSP returns auth_id (or declines)                │
│    → on timeout: query PSP status API (key insight!)  │
│                                                       │
│  Step 2: PSP Capture                                  │
│    → converts authorization to actual charge          │
│                                                       │
│  Step 3: Write Ledger (double-entry)                  │
│    → INSERT buyer debit + seller credit + platform   │
│      commission credit (single atomic transaction)    │
│                                                       │
│  Step 4: Enqueue Seller Payout                        │
│    → add seller's net amount to payout queue          │
│                                                       │
│  Compensation (on any step failure):                  │
│  Step 1 fail → no money moved, return DECLINED        │
│  Step 2 fail → void the authorization                 │
│  Step 3 fail → reverse the capture (PSP refund API)  │
│  Step 4 fail → retry payout queue enqueue             │
└──────────────────────────────────────────────────────┘
    │                              │
    ▼                              ▼
┌──────────────┐          ┌──────────────────┐
│  Ledger DB   │          │  Payout Queue    │
│  (PG append) │          │  (Kafka/SQS)     │
└──────────────┘          └──────────────────┘
                                   │
                                   ▼
                          ┌──────────────────┐
                          │  Payout Service  │
                          │  (batch, T+1)    │
                          └──────────────────┘
                                   │
                                   ▼
                          ┌──────────────────┐
                          │  Bank Transfer   │
                          │  (ACH/Wire)      │
                          └──────────────────┘


RECONCILIATION PATH (async, runs nightly)
──────────────────────────────────────────

  Bank Settlement File (CSV/SWIFT MT940)
    │
    ▼
┌────────────────────────┐     ┌──────────────────────┐
│  Reconciliation Job    │────▶│  Internal Ledger DB  │
│  (Spark batch)         │     └──────────────────────┘
└────────────────────────┘
    │
    ├── Match: PSP transaction ID in file ↔ ledger entry
    ├── Unmatched in bank but not ledger → "phantom charge" → alert ops
    ├── Unmatched in ledger but not bank → "missing settlement" → alert ops
    └── Amount mismatch → alert ops with diff
```

---

## Capacity Math

**Transaction volume:**
- 1 million transactions/day ÷ 86,400 sec = **11.6 writes/sec average**
- Peak (5× average): **58 writes/sec**
- Sounds low — but each transaction generates multiple DB writes (idempotency record, payment record, 3 ledger entries, payout queue message)

**Ledger entries per transaction:**
- Double-entry: every transaction generates at least 3 ledger entries
  - DR: Buyer Account $50.00
  - CR: Platform Account $5.00 (10% commission)
  - CR: Seller Account $45.00
- 1M transactions/day × 3 entries = **3 million ledger rows/day**
- Per row ~200 bytes → **600 MB/day** → **219 GB/year** of raw ledger data
- Over 5 years: **~1 TB** — fully manageable in partitioned PostgreSQL

**Idempotency key storage:**
- 1M keys/day × 500 bytes/key = **500 MB/day**
- Keys expire after 24 hours → rolling 24-hour window ≈ 500 MB total in idempotency table

**PSP timeout handling:**
- If PSP returns timeout on 0.1% of calls: 1M × 0.001 = **1,000 ambiguous calls/day**
- Each requires a status query to PSP — must be designed for, not ignored

---

## API Design

**Initiate payment:**
```
POST /v1/payments
Headers:
  Idempotency-Key: <client-generated UUID v4>
  Authorization: Bearer <session_token>
Body:
  {
    "amount": 5000,           // in cents to avoid floating-point errors
    "currency": "USD",
    "card_token": "tok_visa_123",  // from PSP tokenization
    "seller_id": "seller_abc",
    "order_id": "ord_xyz"
  }
Response 200:
  { "payment_id": "pay_123", "status": "COMPLETED", "captured_amount": 5000 }
Response 402:
  { "payment_id": "pay_123", "status": "DECLINED", "reason": "insufficient_funds" }
```

**Note:** Idempotency-Key in the header, not the body. This is the Stripe convention. The server stores `(idempotency_key, response)` — if the same key appears again within 24 hours, return the stored response verbatim, regardless of outcome (even if the first attempt returned a 500).

**Get payment status:**
```
GET /v1/payments/{payment_id}
Response 200:
  { "payment_id": "...", "status": "COMPLETED|PENDING|FAILED|REFUNDED",
    "amount": 5000, "currency": "USD", "created_at": "..." }
```

**Initiate refund:**
```
POST /v1/payments/{payment_id}/refunds
Headers: Idempotency-Key: <new UUID>
Body: { "amount": 5000, "reason": "customer_request" }
Response 200: { "refund_id": "ref_123", "status": "PROCESSING" }
```

**Payout (internal, called by scheduler):**
```
POST /v1/payouts/batch
Body: { "seller_id": "...", "amount": 45000, "settlement_date": "2025-01-15" }
```

---

## Data Model

**payments table (source of truth for payment state):**
```sql
CREATE TABLE payments (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  idempotency_key  VARCHAR(128) UNIQUE NOT NULL,
  order_id         UUID NOT NULL,
  buyer_id         UUID NOT NULL,
  seller_id        UUID NOT NULL,
  amount_cents     BIGINT NOT NULL,
  currency         CHAR(3) NOT NULL,
  status           VARCHAR(20) NOT NULL,    -- PENDING, AUTHORIZED, CAPTURED, FAILED, REFUNDED
  psp_auth_id      VARCHAR(128),
  psp_charge_id    VARCHAR(128),
  failure_reason   TEXT,
  created_at       TIMESTAMP NOT NULL DEFAULT NOW(),
  updated_at       TIMESTAMP NOT NULL DEFAULT NOW(),
  INDEX idx_order (order_id),
  INDEX idx_buyer (buyer_id),
  INDEX idx_idem (idempotency_key)
);
```

**ledger_entries table (immutable, append-only):**
```sql
CREATE TABLE ledger_entries (
  id               BIGSERIAL PRIMARY KEY,
  payment_id       UUID NOT NULL REFERENCES payments(id),
  entry_type       VARCHAR(20) NOT NULL,    -- DEBIT or CREDIT
  account_id       UUID NOT NULL,           -- buyer/seller/platform account
  account_type     VARCHAR(20) NOT NULL,    -- BUYER, SELLER, PLATFORM
  amount_cents     BIGINT NOT NULL,         -- always positive; direction = entry_type
  currency         CHAR(3) NOT NULL,
  created_at       TIMESTAMP NOT NULL DEFAULT NOW(),
  -- CRITICAL: no UPDATE ever issued on this table
  INDEX idx_payment (payment_id),
  INDEX idx_account (account_id, created_at)
) PARTITION BY RANGE (created_at);         -- monthly partitions for query performance
```

**idempotency_keys table:**
```sql
CREATE TABLE idempotency_keys (
  key              VARCHAR(128) PRIMARY KEY,
  request_path     VARCHAR(256) NOT NULL,
  response_status  SMALLINT NOT NULL,
  response_body    JSONB NOT NULL,
  locked           BOOLEAN DEFAULT TRUE,    -- TRUE while first request in flight
  created_at       TIMESTAMP NOT NULL DEFAULT NOW(),
  expires_at       TIMESTAMP NOT NULL DEFAULT NOW() + INTERVAL '24 hours'
);
```

**outbox table (transactional outbox for reliable event publishing):**
```sql
CREATE TABLE outbox (
  id               BIGSERIAL PRIMARY KEY,
  aggregate_id     UUID NOT NULL,           -- payment_id
  event_type       VARCHAR(64) NOT NULL,    -- PAYMENT_COMPLETED, PAYMENT_FAILED
  payload          JSONB NOT NULL,
  published        BOOLEAN DEFAULT FALSE,
  created_at       TIMESTAMP NOT NULL DEFAULT NOW(),
  INDEX idx_unpublished (published, created_at) WHERE published = FALSE
);
```

**Key DB design notes:**
- Money is always stored as integers (cents/smallest currency unit), never as FLOAT or DECIMAL. Floating-point arithmetic on money is a bug.
- Ledger table is append-only. No UPDATE. No DELETE. Ever. An audit requires the ability to replay every entry.
- Idempotency and payment records are written in the **same transaction** — atomically.

---

## Key Design Decisions

### Decision 1: SAGA Pattern vs. 2PC for Distributed Transaction

The payment touches multiple systems: PSP (external), ledger (internal DB), payout queue. These cannot participate in a single ACID transaction.

| Dimension | 2PC (Two-Phase Commit) | SAGA (Orchestrated) |
|-----------|----------------------|---------------------|
| **Consistency** | Strong atomic commit | Eventually consistent with compensating transactions |
| **Latency** | Synchronous across all participants | Asynchronous steps |
| **External systems** | PSP cannot participate in 2PC | PSP participates via its own API |
| **Failure recovery** | Coordinator crash = all participants blocked indefinitely | Temporal persists state; resumes after crash |
| **Complexity** | Lower if all on same DB | Higher — must design compensating transactions |

**Choice: SAGA with orchestration via Temporal.** 2PC is impossible here because the PSP is an external service with no 2PC support. SAGA allows each step to be local, with compensating transactions on failure. Temporal provides durable workflow execution — if the orchestrator crashes, it resumes from the last checkpoint.

**Compensating transactions (critical to name explicitly):**
- Capture fails → void the authorization
- Ledger write fails → reverse the capture (refund via PSP refund API)
- Payout queue publish fails → mark payment as settled, enqueue payout on next retry

**Trade-off accepted:** Eventual consistency during the saga. Between when the PSP captures the charge and when the ledger records it, there is a brief window where the money is in transit. This is unavoidable with distributed systems. Exactly-once is achieved at the end of the saga, not during.

---

### Decision 2: Idempotency at Every Boundary

The most dangerous failure in payment systems is the idempotency failure: charging a card twice for one purchase.

**The failure sequence:**
1. Client sends `POST /payments` with `Idempotency-Key: k1`
2. Server receives request, charges PSP successfully
3. Server crashes before writing payment record to DB
4. Client retries with the same `Idempotency-Key: k1`
5. Without idempotency: server charges PSP again → double charge

**The fix — Stripe's approach:**
- The idempotency table stores `(key, status, response_body)` in the **same transaction** as the payment record
- On retry with same key: return `response_body` verbatim, never re-execute
- The "locked" state handles concurrent retries: first request sets `locked=TRUE`; concurrent request with same key waits until first completes, then returns cached result
- Keys expire after 24 hours — TTL prevents infinite storage growth

**Brandur Leach's insight (brandur.org, Oct 2017):** The atomic phases pattern. Every foreign-state mutation (PSP call) must be in its own atomic phase. If the mutation succeeds but the checkpoint fails, the idempotency key lets us skip re-executing the mutation on retry.

**Trade-off accepted:** Idempotency table is written synchronously on every payment request, adding one DB write to the critical path. At 58 peak writes/sec, this is trivial.

---

### Decision 3: Outbox Pattern for Reliable Event Publishing

After a payment completes, the system needs to notify the payout service, update the seller's balance, trigger fraud analysis, and send a receipt email. These are all async events.

| Dimension | Direct publish to Kafka | Transactional Outbox |
|-----------|------------------------|---------------------|
| **Reliability** | If app crashes after DB write but before publish → event lost | Write to outbox table in same transaction as payment record → atomic |
| **Order** | Not guaranteed | Guaranteed (outbox poller reads in insert order) |
| **Latency** | Low (direct) | Slightly higher (outbox poller interval, typically 100ms) |
| **Operational complexity** | Simpler | Requires outbox table + poller process |

**Choice: Transactional outbox.** The critical scenario is: payment is written to DB successfully, but the process crashes before Kafka publish. Direct Kafka publish loses the event. Outbox pattern: write the payment record and the outbox event in one transaction — both committed or both rolled back. A separate poller reads unpublished outbox entries and publishes them to Kafka, marking them published after acknowledgement.

**Trade-off accepted:** 100ms additional latency for downstream event delivery. The outbox poller adds operational complexity. Worth it because lost payment events cause sellers to never receive payouts — a severe financial failure.

---

## Deep Dive: PSP Timeout Handling (The Hardest Part)

The PSP (Stripe, Adyen) is an external service. External calls fail in one of three ways:
1. **Clear success:** PSP returns 200 with auth_id
2. **Clear failure:** PSP returns 402 (declined) or 4xx (bad request)
3. **Ambiguous timeout:** No response within 30 seconds — you don't know if the charge happened

Case 3 is the catastrophic case. If you retry and the charge did happen, you double-charge. If you don't retry and it didn't happen, the payment is lost.

**The solution:**

Every PSP request is identified by a unique, client-generated **idempotency key** (sent as a header: `Idempotency-Key: <uuid>`). The PSP deduplicates on this key. If the charge happened, it returns the same result. If it didn't happen, it executes it now.

On timeout: wait 30–60 seconds (give the PSP time to complete), then retry with the **same idempotency key**. The PSP returns the original result — success or failure — without re-charging.

**This is why idempotency keys must flow end-to-end:**

```
Client             Payment API          PSP Adapter          Stripe
  │                    │                     │                  │
  │  POST /payments    │                     │                  │
  │  idem-key: k1 ────▶│                     │                  │
  │                    │  POST /charge       │                  │
  │                    │  idem-key: k1-psp ─▶│                  │
  │                    │                     │  POST /charge    │
  │                    │                     │  idem-key: k1-psp│
  │                    │                     │ ────────────────▶│
  │                    │                     │  ← TIMEOUT       │
  │                    │                     │                  │
  │                    │          (30s wait) │                  │
  │                    │                     │  POST /charge    │
  │                    │                     │  idem-key: k1-psp│
  │                    │                     │ ────────────────▶│
  │                    │                     │  ← 200 captured  │
  │                    │  ← charge_id        │                  │
  │  ← 200 completed   │                     │                  │
```

The key insight: `k1-psp` is derived from the outer idempotency key `k1`, so it's stable across retries. Stripe deduplicates on `k1-psp` and returns the same charge regardless of how many times we retry.

---

## Failure Modes & Mitigations

| Failure | Financial Impact | Detection | Mitigation |
|---------|-----------------|-----------|------------|
| PSP timeout | Unknown — charge may or may not have occurred | HTTP timeout exception | Retry with same PSP idempotency key; PSP returns original outcome |
| Server crash after PSP charge, before ledger write | Charge captured, ledger missing | Reconciliation flags PSP transaction without ledger entry | SAGA compensating transaction: PSP shows capture → write ledger retroactively; or issue refund if intent was lost |
| Duplicate payment request (network retry) | Double charge if not handled | Duplicate idempotency key in table | Idempotency table: return cached response, never re-execute |
| Ledger DB failure | Can't record completed charges | Writes fail, payment stuck in CAPTURED state | Outbox pattern: event published to queue even if ledger write fails initially; retry from queue |
| PSP outage | No payments can be processed | PSP health check + error rate spike | Circuit breaker on PSP adapter; queue payment intents; retry when PSP recovers |
| Reconciliation mismatch | Revenue loss or phantom charges | Nightly reconciliation job alerts | Automated matching; ops team reviews exceptions; audit trail enables manual reconstruction |
| Currency conversion error | Wrong amount charged | Transaction log amount differs from FX rate applied | Store FX rate as part of transaction record; reconcile at settlement time using same rate |

---

## What Strong Candidates Do

1. **Clarify the timeout semantics first.** "What do we do when the PSP returns a timeout?" is the single most important question in payment system design. Candidates who ask it before drawing any diagram show they understand the core problem.
2. **Name idempotency keys as the solution, not just "retry with dedup."** The Stripe pattern (UUID in header, server-side (key, response) cache, 24-hour TTL) is a named, well-known solution. Using the correct vocabulary signals deep familiarity.
3. **Draw double-entry ledger entries explicitly.** For a $50 transaction with 10% commission: DR Buyer $50, CR Platform $5, CR Seller $45. Sum of credits = sum of debits = $50. This is accounting correctness, not just data storage.
4. **Name the outbox pattern by name** and explain why direct Kafka publish after DB write is unreliable (crash between the two).
5. **Include the reconciliation service.** Nightly comparison of internal ledger against PSP settlement files and bank statements. Without reconciliation, you will eventually lose track of money — every large payment company has had this bug.
6. **Explain SAGA compensating transactions end-to-end** for at least one failure scenario (e.g., "if the ledger write fails after capture, we issue a full refund via the PSP refund API and write a REFUND ledger entry").

---

## What Average Candidates Miss

1. **Treating payments like a regular API.** Proposing eventual consistency or AP for the ledger. Money is CP-only. If your system goes temporarily unavailable rather than recording a wrong ledger entry, that is the correct trade-off.
2. **Not handling PSP timeouts.** Designing retry logic that re-submits the payment request without an idempotency key — this double-charges. At Stripe, this is a disqualifying omission for senior roles.
3. **Storing card numbers.** Any mention of "save the card number" in your own database is an immediate PCI DSS red flag. The PSP vault is the correct answer — your system stores only the PSP token.
4. **No reconciliation service.** The reconciliation gap — about 1% of revenue in companies without reconciliation — is a known, documented problem. Not mentioning it signals unfamiliarity with how payments actually work at scale.
5. **Floating-point money.** Using FLOAT or DECIMAL for currency amounts. Money must be integers (cents). $5.10 in floating point becomes $5.099999... — never acceptable in financial systems.
6. **No idempotency on the refund endpoint.** Refunds have the same double-execution risk as charges. Every payment API endpoint that changes financial state needs an idempotency key.
