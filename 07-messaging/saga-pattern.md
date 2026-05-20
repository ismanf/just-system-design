# Saga Pattern for Distributed Transactions

> **TL;DR** — A **saga** is a sequence of local transactions across multiple services, where each step's failure is handled by **compensating actions** for the steps that already succeeded. Sagas replace distributed transactions (2PC) in microservice architectures because **2PC doesn't scale and is unavailable in practice**. There are two flavors: **choreography** (each service emits events; others react; no central coordinator) — simple but hard to reason about beyond 3–4 steps; and **orchestration** (a central saga orchestrator drives each step) — clearer at scale, easier to monitor, requires a workflow engine (Temporal, Cadence, Camunda, Step Functions). The hard parts are: **designing compensations** (refunds, releases, undo actions — not always possible to fully reverse), **idempotency** (every step may retry), **isolation** (other transactions see intermediate states), and **failure observability** (where is order 42 stuck?). Use sagas for genuinely cross-service workflows. Don't reinvent 2PC.

---

## 1. The Problem Sagas Solve

In a monolith with one database, a multi-step business operation is a single transaction:

```sql
BEGIN;
  INSERT INTO orders (...);
  UPDATE inventory SET qty = qty - 1 WHERE id = 42;
  INSERT INTO charges (...);
COMMIT;
```

Either all three succeed, or none. ACID.

In microservices, each step is owned by a different service with its own database. No global transaction.

```
order-service writes order ──► inventory-service decrement ──► payment-service charge
       |                              |                                |
   own DB                          own DB                            own DB
```

What happens if payment fails after inventory decrement? You have an inventory hole. What about partial network failure between inventory and payment? Unclear.

The textbook fix is **Two-Phase Commit (2PC)** — a coordinator asks each participant to "prepare," then "commit" or "abort." See [2PC and 3PC →](../08-distributed-systems/2pc-3pc.md).

The reality:
- **2PC blocks** — any participant holds resources during prepare. If the coordinator dies, participants are stuck.
- **2PC requires participants to implement XA-style protocols** — most modern data stores don't.
- **2PC scales badly** — coordination latency × N services × failure modes.

**Sagas** are the pragmatic alternative: accept eventual consistency, use **compensating transactions** to undo on failure.

---

## 2. The Saga Pattern

```
   T1   →  T2   →  T3   →  T4   succeed
   T1   →  T2   →  T3   →  T4   T4 fails
                          ←  C3  compensate T3
                  ←  C2  compensate T2
          ←  C1  compensate T1
```

Each `Ti` is a local transaction in one service.
Each `Ci` is a **compensating transaction** that undoes Ti.

If step Ti fails, execute Ci-1, Ci-2, ... C1 in reverse order. The system ends in a state "as if T1 through Ti-1 never happened."

**Compensations are forward-only.** You don't "roll back" T2; you execute C2, which is a new local transaction whose business effect cancels T2.

```
T2 = "decrement inventory by 1"
C2 = "increment inventory by 1"        (same effect, opposite sign)

T3 = "charge $50 on credit card"
C3 = "refund $50"                       (Stripe knows the original; refund.create with idempotency)
```

---

## 3. Compensations Aren't Always Pretty

Some actions are easy to compensate:
- Inventory decrement → increment.
- Email queued → cancel before send.
- Reservation → release.

Some are hard or impossible:
- Email already sent → can't unsend. Send an apology.
- Money already transferred → refund (possible but visible).
- Audit log entry → don't compensate; the record is the truth.
- External notification → unwind whatever side-effects you can.

The pragmatic approach: design for **forward-recoverability**. The system should always end in a sensible state, even if some side effects can't be undone perfectly.

For unrecoverable steps, place them **last** in the saga so all reversible work is done first.

---

## 4. Two Flavors

### 4.1 Choreography

Each service listens for events and emits new ones. No central coordinator.

```
   order-service           ──► OrderPlaced
                                    │
                            ┌───────┴────────┐
                            ▼                ▼
                       inventory-svc   payment-svc
                       reserves stock  charges card
                            │                │
                            ▼                ▼
                      InventoryReserved  PaymentSucceeded
                            │                │
                            └──────┬─────────┘
                                   ▼
                       order-service confirms order
                       (waits for both)
```

If `PaymentFailed` arrives:
- `inventory-svc` listens for `PaymentFailed` → releases reservation.
- `order-service` cancels the order.

**Pros**
- No central piece. Each service is autonomous.
- Loose coupling.
- Simple for 2–3 step flows.

**Cons**
- Hard to see the overall flow. Where is order 42 stuck?
- Hard to add steps without touching many services.
- Cyclic dependencies sneak in.
- Difficult to test end-to-end.
- Debugging is archaeology.

**When**
- Short, simple sagas (2–3 steps).
- Highly autonomous teams.

### 4.2 Orchestration

A central **saga orchestrator** drives each step:

```
   client  ──► saga-orchestrator (state machine)
                       │
                       ▼ Step 1: reserve inventory
                  inventory-svc
                       │ ACK
                       ▼ Step 2: charge payment
                  payment-svc
                       │ ACK
                       ▼ Step 3: confirm order
                  order-service
                       │ ACK
                       ▼ done

   if any step fails, orchestrator triggers compensations in reverse.
```

The orchestrator is its own service. Its state lives in a DB (often event-sourced).

**Pros**
- Explicit flow — easy to see what step we're on.
- Easy to add/remove steps.
- Centralized timeout / retry / compensation logic.
- Good monitoring.

**Cons**
- Adds a new service (the orchestrator).
- Tighter coupling to orchestrator's schema.
- Potential bottleneck (mitigated by partitioning).

**When**
- Sagas with > 3 steps.
- Complex compensation paths.
- Need visibility for support / ops.

### Tools for orchestration
- **Temporal** — modern, powerful, code-as-workflow. Best-in-class.
- **Cadence** — Uber's open-source predecessor to Temporal.
- **Camunda** — BPMN-based workflow engine.
- **AWS Step Functions** — managed, JSON-defined state machine.
- **Netflix Conductor** — open-source.
- **Apache Airflow** — used for ETL more than sagas but possible.

---

## 5. Saga Step Lifecycle

Each step in a saga has a lifecycle:

```
                ┌──── retry ────┐
                │               │
   PENDING ──► EXECUTING ──► SUCCESS
       │           │           │
       │           ▼           │
       │       FAILED          │
       │           │           │
       │           ▼           │
       └──► COMPENSATING ──► COMPENSATED
```

The orchestrator persists state at each transition. On crash, it recovers from the persisted state.

---

## 6. Idempotency Is Mandatory

Every step in a saga must be **idempotent** because:
- The orchestrator may retry on timeout.
- Brokers deliver at-least-once.
- Compensations may run multiple times.

Use idempotency keys per saga-step:

```
key = saga_id + step_id + attempt_id (or just saga_id + step_id)
```

The participating service stores `processed_keys` and short-circuits duplicates.

For external APIs (Stripe, etc.) pass the idempotency key through. Stripe's idempotency is your safety net.

See [Idempotency →](../03-apis/idempotency.md).

---

## 7. Isolation: The ACID-D Problem

In a single-DB transaction, other readers don't see your intermediate state (isolation). In a saga, **partial state is visible**:

```
saga progress:    T1 → T2 → ... (still executing)
other reader:     sees results of T1 but not T2
```

This is fundamental. You can't have transaction isolation across services without 2PC.

Mitigations:
- **Semantic locks** — mark records as "in saga" so readers know it's not committed (`order.status = 'pending'`).
- **Commutative updates** — operations that compose regardless of order (counters, set adds).
- **Re-read after saga complete** — readers wait for confirmation events.
- **Acceptance** — users see "pending"/"processing" status; it's UX honesty about distributed work.

---

## 8. Worked Example: Order Saga

Order → reserve inventory → charge payment → confirm shipment.

### Choreography version
```
order-service writes order (status: pending) → emit OrderPlaced

inventory-service consumes OrderPlaced
  → reserve stock
  → emit InventoryReserved (or InventoryUnavailable)

payment-service consumes InventoryReserved
  → charge card
  → emit PaymentSucceeded (or PaymentFailed)

order-service consumes PaymentSucceeded
  → mark order confirmed → emit OrderConfirmed

shipment-service consumes OrderConfirmed
  → schedule shipment

if inventory unavailable:
  order-service consumes InventoryUnavailable
  → mark order canceled

if payment failed:
  inventory-service consumes PaymentFailed
  → release reservation
  order-service consumes PaymentFailed
  → mark order canceled
```

### Orchestration version
```python
# Pseudo-code in Temporal-style
@workflow
def OrderSaga(order_id):
    try:
        await reserve_inventory(order_id)
    except InventoryUnavailable:
        await cancel_order(order_id, reason="no_stock")
        return

    try:
        await charge_payment(order_id)
    except PaymentFailed:
        await release_inventory(order_id)
        await cancel_order(order_id, reason="payment_failed")
        return

    await confirm_order(order_id)
    await schedule_shipment(order_id)
```

Each call is a "task" the orchestrator runs against the right service. Failures are caught and trigger compensations. The orchestrator persists state at every step.

Temporal makes this read like normal code while underneath:
- Each `await` is durable.
- On crash, the workflow resumes.
- Long-running workflows (days, weeks) are fine.
- Visibility tooling shows where each workflow is.

---

## 9. Saga vs Outbox

These are complementary:

- **Outbox pattern** — atomically write business state + event publish. Solves "did I publish?" at the producer.
- **Saga** — coordinate multi-service workflow with compensations. Solves "what if step 3 fails?" at the choreographer/orchestrator.

A saga **uses the outbox pattern** for each step's event publish. See [Outbox Pattern →](./outbox-pattern.md).

---

## 10. Saga vs Event Sourcing

Compatible. Many sagas are event-sourced:
- The orchestrator's state log = the saga's event log.
- Replay = resume after crash.
- See [Event Sourcing →](./event-sourcing.md).

Temporal effectively gives you event-sourced workflows by default — every "decision" is a durable event.

---

## 11. Common Patterns

### 11.1 Forward-only saga
No compensations possible. Steps are designed to be retried until success (idempotent and durable). When failure is permanent, raise an alert; humans handle the exception.

Examples: cleanup tasks where partial success doesn't matter; idempotent provisioning.

### 11.2 Backward-recovery saga
The classic saga with compensations. The default mental model.

### 11.3 Mixed saga
First N steps have compensations; final step is "fire and forget" (e.g., emit a fact). If the first N succeed, the saga is "committed" even if the final step fails.

### 11.4 Pivot transaction
A special step that's either:
- The point of no return (everything before can be compensated; after, it can't).
- The first non-compensable step (place it as late as possible).

---

## 12. Observability for Sagas

Sagas span services. Observability is the make-or-break property.

- **Saga ID** in every event/log/trace. Pass it through.
- **Saga status dashboard** — every saga, its current step, time in step, last event. The orchestrator's DB is your single source of truth.
- **Failure alerts** — sagas in compensating or stuck for too long.
- **Distributed tracing** — OpenTelemetry / Jaeger / Datadog. Each saga step is a span.
- **Replay tooling** — given a saga ID, retrieve all events and inspect.

Temporal's UI lets you click on any workflow and see every step's input, output, and any retries. This is gold for ops.

---

## 13. When NOT to Use Sagas

- **Single-service transaction** — use a DB transaction.
- **Two-service flow where one can do both** — see if you can merge concerns.
- **The "transaction" is just notification, not coordinated state changes** — use plain events.
- **The compensations are impossible** — rethink the design. Maybe pre-allocate / pre-reserve everything synchronously.
- **You actually need 2PC** — rare, but real (e.g., banking with multiple ledgers). Use it; accept the cost.

---

## 14. Common Mistakes

- **Compensations that aren't idempotent.** Multiple retries leave system in worse state.
- **Compensations that aren't tested.** Happy path works in dev; broken compensation surfaces only during a real failure.
- **Choreography for too-complex flows.** Beyond 3 steps, debugging is hell. Move to orchestration.
- **No saga ID.** Can't correlate across services.
- **No isolation strategy.** Other services see partial state and act on it.
- **Treating the orchestrator's state as throwaway.** Crashes lose mid-flight sagas.
- **Pivot transaction in the wrong place.** Non-compensable step happens before fully-compensable ones; can't recover.
- **Saga that depends on synchronous calls.** Defeats async benefits. Use queues / events between steps.
- **Trying to make sagas isolated like ACID transactions.** They aren't. Embrace eventual consistency.
- **No DLQ for permanently-stuck sagas.** They sit forever; nobody notices.

---

## 15. Cheat Card

```
WHAT          multi-step workflow across services
              with compensating transactions on failure

WHY           microservices = no global transaction
              2PC doesn't scale; use sagas instead

TWO FLAVORS
  choreography  events between services, no coordinator
                 simple ≤3 steps; hard to debug at scale
  orchestration central saga orchestrator drives steps
                 explicit, easier to monitor, more steps

COMPENSATIONS  forward-only ops that undo previous steps
               place non-compensable steps last

REQUIRED
  saga ID propagated everywhere
  idempotent steps (and compensations)
  durable orchestrator state (or event log)
  observability: dashboard + tracing + DLQ
  isolation strategy: semantic locks / pending statuses

TOOLS          Temporal, Cadence, Camunda, Step Functions

NOT FOR        single-service tx (use DB tx)
               flows with impossible compensations
               flows demanding ACID isolation

PITFALLS       non-idempotent compensations, untested error paths,
               choreography for big flows, no saga ID,
               wrong pivot, no DLQ for stuck sagas

RULE           Design for failure first. The happy path is easy;
               the compensation paths are the whole point.
```

---

## 16. Resources

### Papers
- "Sagas" — Garcia-Molina & Salem, 1987 (original paper).

### Books
- *Microservices Patterns* — Chris Richardson (excellent saga coverage).
- *Designing Data-Intensive Applications* — Kleppmann.
- *Building Microservices* — Sam Newman.

### Articles
- "Pattern: Saga" — Chris Richardson, microservices.io.
- "Sagas without code" — Bernd Rücker (Camunda).
- "Why we built Temporal" — Maxim Fateev.
- "Long-running workflows with Temporal" — Stripe / Datadog / etc.

### Documentation
- **Temporal docs**: <https://docs.temporal.io/>
- **AWS Step Functions**: <https://docs.aws.amazon.com/step-functions/>
- **Camunda 8 / Zeebe**: <https://docs.camunda.io/>
- **Netflix Conductor**: <https://conductor.netflix.com/>

### Videos
- Chris Richardson — "Sagas in microservices".
- Maxim Fateev — Temporal foundations.
- ByteByteGo — "Saga Pattern".

### Tools
- **Temporal**, **Cadence**, **Camunda 8**, **AWS Step Functions**, **Netflix Conductor**.
- Lightweight: Eventuate Tram, axon-saga.

### Adjacent reading
- [Event-Driven Architecture →](./event-driven-architecture.md)
- [Event Sourcing →](./event-sourcing.md)
- [Outbox Pattern →](./outbox-pattern.md)
- [Delivery Guarantees →](./delivery-guarantees.md)
- [Idempotency →](../03-apis/idempotency.md)
- [2PC and 3PC →](../08-distributed-systems/2pc-3pc.md)
- [Microservices Architecture →](../14-architecture/microservices.md)

---

*Previous:* [← Batch vs Stream Processing](./batch-vs-stream.md)  |  *Next:* [Outbox Pattern →](./outbox-pattern.md)
