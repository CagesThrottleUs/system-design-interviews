# Saga Pattern

## Intent

Coordinate a multi-step business transaction across independent services by breaking it into a sequence of local transactions, each paired with a compensating transaction that semantically undoes its effect if a later step fails.

---

## The Problem

Consider a checkout flow: a user places an order, the payment service charges their credit card, the inventory service reserves the item, and the fulfillment service creates a shipment. Each of these operations touches a different database, owned by a different service, running on a different server. The business requirement is clear — all three must succeed together, or none should take effect. If fulfillment fails after the card has been charged and the inventory reserved, the user has been billed for an item they will never receive and the inventory count is wrong.

The traditional database solution to multi-step atomicity is the two-phase commit (2PC). In 2PC, a central coordinator first asks all participants whether they are ready to commit (the prepare phase), and only if all say yes does it instruct them to commit. If any participant says no, or if any participant fails to respond, the coordinator instructs all to abort. This guarantees atomicity across multiple databases — but it does so at a severe cost.

2PC is a blocking protocol. During the interval between the prepare and commit phases, every participant holds locks on the rows it has modified. If the coordinator crashes after sending prepare but before sending commit, all participants remain locked indefinitely — waiting for a coordinator that may never return. This is not a theoretical edge case; it is a common operational failure mode. At the scale of Amazon, Stripe, or Uber — where the distributed transaction coordinator might handle millions of transactions per second and coordinator failover is measured in tens of seconds — indefinitely locked rows are a service-level incident. The entire checkout flow, or the entire payment processing pipeline, stalls until the coordinator recovers and replays its decision log.

Beyond locking, 2PC introduces a single point of failure in the coordinator and is sensitive to network partitions in a way that distributed systems cannot tolerate. As described in Brewer's CAP theorem and documented in Amazon's Dynamo paper (2007), the practical engineering choice for internet-scale systems is to accept that 2PC is incompatible with high availability under partition. The result is that no major internet company uses 2PC for cross-service transactions at scale. The Saga pattern is the production-proven alternative.

---

## How Sagas Work

A saga decomposes a distributed transaction into a sequence of local transactions, where each step publishes an event or sends a command that triggers the next step. If any step fails, the saga executes a series of compensating transactions in reverse order to undo the effects of the steps that already succeeded. There is no global coordinator holding locks; each service commits its local transaction independently and immediately, making the system available to process other work while the saga progresses.

### Choreography-based Saga

In the choreography model, there is no central orchestrator. Services communicate entirely through events. The payment service charges the card and emits a `PaymentCharged` event to a shared event bus (Kafka, SNS, or similar). The inventory service subscribes to `PaymentCharged`, reserves the item, and emits `ItemReserved`. The fulfillment service subscribes to `ItemReserved`, creates the shipment, and emits `ShipmentCreated`. Each service knows only about the events it subscribes to and the events it emits — it has no knowledge of the overall saga or its other participants.

The appeal is architectural simplicity in the steady state. There is no new service to build and operate. Each team owns their service end-to-end, including the events it emits and the compensating logic it implements. Loose coupling is a genuine property — you can add a new step to the saga (say, a fraud-check service that listens to `PaymentCharged`) without modifying any existing service.

The difficulty emerges under failure and during debugging. If the fulfillment service emits a `ShipmentFailed` event, the payment service and inventory service must both be listening for that event to trigger their compensations. Who is responsible for ensuring both compensations run? If the `ShipmentFailed` event is emitted but the inventory service's compensation handler crashes before executing, the inventory reservation is never released. Detecting this requires querying multiple services and correlating events by order ID — there is no single place in the system that knows the overall saga state. For an operations team investigating "why does order #12345 show payment charged but no shipment created?", the answer requires querying the payment service, inventory service, fulfillment service, and event bus, then manually reconstructing the timeline. At scale, this debugging overhead is a significant operational tax.

### Orchestration-based Saga

In the orchestration model, a central saga orchestrator — a durable state machine — drives the entire sequence by sending explicit commands to each service and waiting for responses. The orchestrator knows the full saga definition: step 1 is charge payment, step 2 is reserve inventory, step 3 is create shipment. It transitions between states based on the responses it receives and is the single source of truth for where any given saga instance currently sits in its lifecycle.

If the fulfillment service responds with a failure, the orchestrator, having full knowledge of which steps have already succeeded, drives the compensating sequence: send `CancelReservation` to inventory, wait for acknowledgement, then send `RefundPayment` to payment. The compensation sequence is explicit, ordered, and executed by one component rather than distributed across all participants.

Uber documented their trip lifecycle as an orchestration-based saga in their 2020 engineering blog post describing Cadence, their workflow orchestration system. A trip at Uber involves a sequence of steps — request matching, driver assignment, pickup confirmation, trip tracking, and payment — each backed by a different microservice. The Cadence orchestrator maintains the full state of every trip instance and drives compensations (cancel the trip, release the driver, refund the rider) if any step fails. Uber reported that this architecture eliminated entire classes of inconsistency bugs that had plagued their earlier choreography-based approach, because the compensation logic was centralized and explicitly tested rather than distributed and emergent.

The tradeoff is the orchestrator itself. It is a new service that must be built, operated, scaled, and made highly available. If the orchestrator becomes unavailable, no new sagas can be started and in-flight sagas cannot advance — it is a soft single point of failure for the business flow, though not a data-loss risk if the state is durably persisted. The orchestrator can also become a development bottleneck if every saga change requires modifying a central component owned by a platform team rather than the service teams whose business logic it coordinates.

### Compensating Transactions

The key intellectual distinction in the Saga pattern is that compensating transactions are not rollbacks. This is a subtle but critical point that candidates frequently miss.

In a traditional database rollback, the transaction never committed — the row changes are discarded at the storage level, and the world returns to exactly the state it was in before the transaction began. No external system observed the changes. A compensating transaction in a saga operates in a world where the original transaction has already committed, been acknowledged, and potentially had downstream effects. The payment service charged a card — the bank received the authorization request and approved it. The user may have received a confirmation email. An audit log entry was written. A compensating "refund" transaction does not erase these facts; it performs a new operation (issue a refund) that semantically reverses the financial effect while leaving the original charge in the audit history.

This distinction has concrete design consequences. If you design your compensations assuming they can fully undo the original action, you will miss cases where the original action had side effects that cannot be undone: notification emails cannot be unsent, external API calls cannot be un-made, and audit log entries should not be deleted. A well-designed saga acknowledges that some actions are irreversible and designs the compensation to be a semantic undo — the best possible reversal of effect — not a physical erasure.

The compensation for "item reserved" is "cancel reservation." The compensation for "payment charged" is "issue refund." The compensation for "shipment created" is "cancel shipment" — possible only within a time window before the shipment is dispatched, after which the compensation becomes "initiate return." The time-bounded nature of some compensations is a design constraint that must be made explicit in the saga definition.

---

## Idempotency Requirement

Every saga step must be idempotent, because the messaging infrastructure will deliver messages at least once, not exactly once. Consider this sequence: the orchestrator sends a `ChargePayment` command to the payment service. The payment service charges the card and writes a row to its database, but before it can send back the acknowledgement, its network connection drops. The orchestrator, having received no response after a timeout, retries the `ChargePayment` command. If the payment service processes this command naively — look up the order, charge the card — the user is double-charged.

The standard solution is idempotency keys: a unique identifier (typically the saga instance ID concatenated with the step name, for example `order_789_charge_payment`) that the payment service stores alongside the transaction. On retry, the service checks whether this idempotency key has already been processed; if it has, it returns the previous result immediately without executing the operation again. Stripe documented their idempotency key pattern in engineering blog posts and their API documentation around 2022–2023, noting that idempotency keys are mandatory for any payment API call that might be retried. Stripe's PaymentIntent object is itself a saga with an explicit state machine — the intent moves through states (created → requires_payment_method → requires_confirmation → processing → succeeded or requires_action) and each state transition is idempotent given the same idempotency key.

Idempotency at the step level is a prerequisite, not an afterthought. Designing a saga step without considering what happens on retry is designing for the happy path only.

---

## Trigger Signals

- Multi-step business flow where each step touches a different database or service
- Need for distributed consistency without 2PC or global transactions
- Any flow with a clear "undo" requirement: bookings, orders, enrollments, financial transfers
- High availability requirement that cannot tolerate coordinator-induced blocking
- Systems at the scale where lock contention across services would be a throughput bottleneck

---

## Problems Where It Applies

**Design a Payment System:** The archetypal saga. Steps are: create payment intent, charge card (call payment processor), update order status, trigger fulfillment, send confirmation. Each step is a local transaction with a defined compensation (void the charge, cancel the order, halt fulfillment). Stripe's PaymentIntent state machine is a production implementation of this saga.

**Design a Notification System:** Multi-step flow from "create notification record" → "enqueue to message queue" → "deliver to APNs/FCM" → "mark delivered." If delivery fails, the compensation marks the notification as failed and potentially re-enqueues with backoff. The saga pattern explains why notification status tracking requires explicit state rather than inferring it from delivery acknowledgement alone.

**Booking Systems (Airlines, Hotels, Concerts):** Reserve seat → charge payment → issue ticket. The "reserve seat" step has a time-bounded compensation — if payment fails, the seat reservation must be released before the reservation timeout expires. This time constraint is a common source of bugs in naive implementations and must be modeled explicitly in the saga.

---

## Trade-offs

Sagas trade atomicity for availability. The system remains available to process other work while a saga is in flight, but there is a window of inconsistency — the payment is charged before the inventory is reserved, and the world can observe this intermediate state. For most business flows, this is acceptable; the window is milliseconds to seconds, not the indefinitely locked state that 2PC can produce under coordinator failure.

Debugging distributed saga state is harder than debugging a monolithic database transaction. An in-flight saga instance may have committed steps in service A, queued commands in service B, and not yet reached service C. Reconstructing "why is this order stuck?" requires correlating logs and state across multiple services, which demands good distributed tracing (a Jaeger or Zipkin trace for the saga instance ID), explicit saga state persistence, and ideally a saga monitoring dashboard that shows the step progression for any given instance.

Defining compensating transactions for every step is a non-trivial design exercise, particularly for steps with time-bounded reversibility or steps whose effects propagate to external systems. Every saga design session should produce an explicit table: for each step, what is the compensation, and under what conditions is the compensation impossible? Those impossible-compensation scenarios are the risk surface of the design and deserve explicit product and engineering alignment.

---

## Real-World Usage

**Uber's Trip Lifecycle (~2020, Cadence workflow engine):** Uber's 2020 engineering blog post describing Cadence details how the full trip lifecycle — request, dispatch, pickup, trip, payment — is modeled as an orchestration-based saga. The Cadence workflow engine persists the saga state durably (in Cassandra), drives the step sequence, and handles compensations (cancel the trip, reverse the charge) when steps fail. Uber reported that Cadence-based sagas replaced a prior system that used a combination of database state machines and background jobs, eliminating race conditions that had caused intermittent double-charges and stranded trips.

**Stripe's PaymentIntent (~2022–2023 Stripe engineering documentation):** Stripe's PaymentIntent API is a publicly documented orchestration saga. The PaymentIntent object represents the full lifecycle of a payment attempt: it transitions through explicit states (requires_payment_method, requires_confirmation, processing, succeeded, canceled, requires_action for 3DS). Each state transition is driven by explicit API calls with idempotency keys. Compensations are explicit: a succeeded PaymentIntent can be reversed with a Refund object, not by modifying the original intent. Stripe's engineering blog posts on idempotency (circa 2022) describe the key design decisions that make the saga retry-safe.

---

## Common Mistakes

**Treating compensating transactions as database rollbacks.** Candidates say "if step 3 fails, we rollback steps 1 and 2." Rollback implies the operations never happened at database level. In a saga, they did happen — they committed to independent databases. The correct framing is "we execute compensating transactions that semantically reverse the effect." The distinction matters because compensations can fail too (the refund API might be down), and compensating a committed operation has different error handling requirements than a database rollback.

**Designing saga steps without idempotency.** Candidates sketch the happy path — charge, reserve, ship — and never address what happens when the orchestrator retries a command after a network timeout. The payment service must be idempotent with respect to a given charge request, or the user gets double-charged on retry. Idempotency keys at every step are mandatory, not optional, and must be designed as first-class data in the API contract.

**Choosing choreography for complex sagas.** Choreography feels elegant for simple two-step sagas, but candidates who apply it to a five-step saga with multiple compensation paths are describing a system where the overall saga state is invisible to any single component. For any saga with more than three steps, or with branching compensation logic, the operational complexity of choreography — debugging across multiple services, ensuring all compensation listeners are registered — typically outweighs the architectural purity of no central orchestrator. Orchestration should be the default for complex flows; choreography is a valid choice for simple, stable, two-step patterns.

**Missing the time-bounded compensation problem.** Some saga steps can only be compensated within a time window. An airline seat reservation can be released before departure; after departure, the seat cannot be unreserved. A shipped package can be intercepted before it leaves the warehouse; after dispatch, the compensation becomes "initiate return." Candidates who do not surface these time constraints are designing a saga that will silently fail to compensate in the most operationally stressful scenarios — where the system is already under load and a step failure triggers a compensation that discovers it is no longer possible.
