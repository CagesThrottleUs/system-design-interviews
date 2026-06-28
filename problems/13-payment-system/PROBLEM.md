# Payment System

**Difficulty:** Advanced
**Category:** Consistency-critical, Financial systems
**Time Box:** 45 min
**Key Concepts:** Idempotency keys, double-entry ledger, SAGA pattern for distributed transactions, PSP integration (PCI DSS), reconciliation
**Asked at:** Stripe, Uber, Airbnb, Amazon

---

## Problem Statement

Your company runs a marketplace where buyers purchase goods from sellers. The company takes a commission on each transaction. You need to design the payment system that processes these payments end-to-end: the buyer pays, the company takes its cut, the seller gets the remainder, and every cent is accounted for in an audit-ready ledger.

This is not a CRUD application. The core challenge is correctness under failure. Any component — your servers, the payment processor, the network — can fail at any moment. When a server crashes immediately after charging a card but before recording the transaction, do you charge the buyer again? When the payment processor returns a timeout instead of success or failure, what do you do? When your ledger disagrees with the bank's statement by $50,000 at month-end, how do you find the discrepancy?

The stakes are not abstract: a double-charge destroys customer trust and violates consumer protection laws. A missed charge is direct revenue loss. A ledger that doesn't reconcile with the bank is a compliance violation. Every design decision must answer: "What happens when this component fails?"

For context on real-world scale: Stripe processes hundreds of billions of dollars annually. Uber's LedgerStore holds trillions of immutable ledger entries and processed 1 trillion entries migrated from DynamoDB in 2024 at a cost savings of $6 million per year. Airbnb's payment system, after migrating to service-oriented architecture in phases, achieved up to 150× performance gains on unified data reads.

---

## Actors

| Actor | Description | Primary Action |
|-------|-------------|----------------|
| **Buyer** | Customer making a purchase | Initiates payment via checkout |
| **Seller** | Merchant receiving payment minus commission | Receives payout after settlement |
| **Platform** | Your company (marketplace operator) | Collects commission, orchestrates flow |
| **PSP** | Payment Service Provider (Stripe, Adyen, Braintree) | Authorizes and captures card charges |
| **Bank** | Issuing and acquiring banks | Settle funds between accounts |
| **Ops/Finance** | Internal team | Monitors reconciliation, resolves exceptions |

---

## Functional Requirements

**Core (MVP):**
- FR1: Process a payment from buyer to platform to seller atomically — if any step fails, no money moves
- FR2: Ensure idempotency — retrying a failed payment must never result in a double charge
- FR3: Maintain an immutable double-entry ledger recording every movement of money
- FR4: Never store raw card numbers — integrate with a PSP using tokenization
- FR5: Support payment failure and cancellation with automatic refund processing

**Secondary:**
- FR6: Support multi-currency transactions with real-time exchange rates
- FR7: Async payout to sellers (T+1 or T+2 settlement cycle)
- FR8: Reconcile internal ledger against bank statements automatically
- FR9: Detect and flag potentially fraudulent transactions for manual review
- FR10: Provide buyers and sellers with transaction history

---

## Non-Functional Requirements

| Requirement | Target | Rationale |
|-------------|--------|-----------|
| Idempotency | Exactly-once charge per transaction intent | Double-charge is catastrophic; legal and trust consequences |
| Consistency | CP (consistency over availability) | Money must not be lost or duplicated; AP is wrong for payments |
| Payment latency (p99) | < 3 seconds | User-perceivable payment flow; card networks have their own latency |
| Ledger durability | Never lose a committed entry | Ledger is the source of financial truth |
| Availability | 99.99% | Checkout failures = lost revenue; 4 nines is industry standard |
| Audit trail | All state transitions logged with actor + timestamp | Regulatory requirement; fraud investigation requirement |
| PCI DSS compliance | Never store, transmit, or process raw card data in your systems | Card data in your DB = massive liability and audit scope |

---

## Capacity Estimation Hints

Use these to derive your own numbers before the solution guide:

- **Daily transactions:** 1 million purchases/day
- **Average transaction value:** $50
- **Read:write ratio for ledger:** 10:1 (reads are dashboards, reports, history)
- **Peak multiplier:** 5× average (flash sales, seasonal spikes)
- **Refund rate:** ~2% of transactions within 30 days
- **Payout batch size:** Sellers paid out daily (batch of all settled transactions for that seller)

Derive: write QPS, peak QPS, ledger rows per transaction, daily ledger growth, storage per year.

---

## Good Clarifying Questions to Ask

### Semantics
1. What happens if the payment processor returns a timeout — do we charge the card or not? This determines whether you need exactly-once semantics or at-least-once + dedup. (This is the most important clarifying question)
2. What is the consistency model? If I charge a card successfully, can my ledger service fail to record it? (Must be no — this drives the transactional design)
3. Is this a marketplace (buyer pays seller, platform takes commission) or a direct payment (buyer pays platform)? (Changes the number of ledger entries per transaction)

### Scale
4. How many transactions per day? Are there predictable peak periods (flash sales, holidays)? (Drives capacity and rate limiting design)
5. What geographies? Single country or global? (Multi-currency, regulatory requirements differ by country)

### Compliance
6. Do we need to be PCI DSS compliant? (If yes: never store card numbers, tokenize everything at the edge)
7. Are there regulatory requirements around payout timing? (Some jurisdictions require T+1 payout)

### Features
8. Do we need real-time fraud detection, or is post-hoc flagging acceptable? (Real-time = in-payment-path, adds latency; post-hoc = async, cheaper)
9. Do we need multi-party transactions (multiple sellers in one order)? (Each seller is a separate payout leg)
10. What's the dispute/chargeback workflow? (Chargebacks are initiated by the bank, not the buyer — separate flow from refunds)

---

## Key Components

<details>
<summary>Reveal components</summary>

- **Payment Orchestrator** — coordinates the multi-step payment saga (authorize → capture → ledger → payout queue); the "brain" of the flow
- **Idempotency Service** — deduplicates incoming payment requests using idempotency keys; stores (key, status, response) pairs
- **PSP Adapter** — wraps the external Payment Service Provider API; handles timeouts, retries, and tokenized card references
- **Ledger Service** — append-only double-entry ledger; records every debit and credit as immutable entries
- **Payout Service** — batches seller payouts and initiates bank transfers; runs on settlement schedule
- **Reconciliation Service** — nightly job that compares internal ledger against bank settlement files; flags discrepancies
- **Outbox** — transactional outbox pattern for reliable event publishing from DB write to downstream services
- **Fraud Detection** — async scoring engine that flags transactions for review; does not block the payment path

</details>
