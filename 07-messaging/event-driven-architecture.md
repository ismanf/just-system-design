# Event-Driven Architecture

> **TL;DR** — **Event-driven architecture (EDA)** is a style where services communicate by producing and consuming **events** rather than calling each other synchronously. An event is a notarized fact: "order 42 was placed at 14:03." Producers don't know consumers; consumers don't know producers. The broker (Kafka, RabbitMQ, Pub/Sub) is the contract. EDA trades **synchronous request/response** for **asynchronous flow**, gaining **loose coupling**, **scalability**, and **resilience**, and giving up **immediate feedback**, **simple debugging**, and **strong consistency**. It works beautifully when business processes are naturally asynchronous (orders, notifications, analytics) and badly when you force synchronous semantics through it. The hard problems are **event design** (what to publish), **schema evolution** (events live forever), **ordering** (per-partition), **at-least-once duplicates** (need idempotent consumers), and **debugging across services** (need tracing).

---

## 1. The Shape

```
SYNCHRONOUS / REQUEST-RESPONSE
   order-service ──HTTP──► inventory-service
                 ◄────────
   order-service ──HTTP──► payment-service
                 ◄────────
   order-service ──HTTP──► notification-service
                 ◄────────

   tight coupling: order-service must know about everyone.
   slow path: as slow as the slowest dependency.
   any one fails → order fails.


EVENT-DRIVEN
   order-service ──► OrderPlaced ──► [broker]
                                       │
                            ┌──────────┼──────────┐
                            ▼          ▼          ▼
                       inventory   payment   notification
                       service    service    service
                       (react independently)

   loose coupling: order-service knows the broker.
   independent failure: any consumer can fail/recover without blocking.
   eventual consistency: side effects happen "soon" not "now".
```

The center of EDA is the **event**: an immutable, past-tense statement of something that happened.

---

## 2. Events vs Commands vs Queries

These three messaging shapes are often confused.

| Type | Tense | Owner | Reply? |
|---|---|---|---|
| **Command** | "Do X" (imperative) | Producer asks consumer to act | Sometimes |
| **Event** | "X happened" (past tense, fact) | Producer broadcasts; consumers choose | No |
| **Query** | "What is X?" (interrogative) | Consumer asks producer | Yes |

Commands and queries are typically synchronous. Events are typically asynchronous and published to a broker.

The classic mistake: calling something an "event" but treating it as a command (`UserShouldBeCharged` — that's a command). Real events are facts: `OrderPlaced`, `UserSignedUp`, `PaymentSucceeded`.

---

## 3. Why EDA

### 3.1 Loose coupling
Producer and consumer never call each other directly. Adding a new consumer doesn't require changes to the producer. Want analytics? Add an `analytics` consumer of `OrderPlaced`. Done.

### 3.2 Scalability
Producer and consumer scale independently. If `notifications` is slow, orders still get placed; notifications just accumulate in the queue.

### 3.3 Resilience
If `payment-service` is down, the `OrderPlaced` event still gets stored. When payment comes back, it processes the backlog.

### 3.4 Audit / replay
With a stream-based broker (Kafka), every event is a permanent record. Replay to rebuild state, run new analytics, debug.

### 3.5 Natural fit for many domains
E-commerce, finance, logistics, analytics — these are inherently async pipelines. "Place order → reserve inventory → charge → ship → notify" is naturally a chain of events.

---

## 4. Why Not EDA

It's not free. Costs:

### 4.1 Eventual consistency
"Place order" returns immediately, but inventory may take seconds to reflect. UI must handle "pending" states.

### 4.2 Harder debugging
A user complaint about a missing email becomes a 5-service investigation. Need distributed tracing (OpenTelemetry, Jaeger) to follow the event flow.

### 4.3 Schema evolution discipline
Events live forever. A producer change can break consumers a year from now. Need schema registry + compat rules.

### 4.4 Duplicate / out-of-order handling
At-least-once delivery means consumers must be idempotent. Out-of-order events require versioning logic.

### 4.5 Operational complexity
You've added a broker, a schema registry, monitoring, DLQs, multiple consumer services. Lots of moving parts.

### 4.6 Wrong fit for sync needs
"User clicks Buy → show success page." If you do this fully async, you can't say "success" until inventory/payment ack — and at that point you've reinvented synchronous over messaging, which is the worst of both worlds.

The honest take: **EDA is a great fit for parts of a system, not the whole system.** Most production architectures are hybrid.

---

## 5. Event Shapes

Several common patterns, each with trade-offs.

### 5.1 Notification event (thin)
Tells you something happened, with minimal data.

```json
{
  "type": "OrderPlaced",
  "id": "evt_123",
  "order_id": "ord_42",
  "occurred_at": "2026-05-19T14:03:12Z"
}
```

Consumers fetch details via API. Loose coupling on schema; tight coupling on availability of the producer.

### 5.2 State-carrying event (fat)
Carries the full state of the entity.

```json
{
  "type": "OrderPlaced",
  "id": "evt_123",
  "order": {
    "id": "ord_42",
    "user_id": "user_99",
    "items": [...],
    "total": 49.99,
    "currency": "USD",
    "status": "placed"
  },
  "occurred_at": "2026-05-19T14:03:12Z"
}
```

Consumers self-contained; producer can go down without affecting consumers. Event payload bigger; schema becomes a contract.

### 5.3 Event-sourced delta
Carries only the change.

```json
{
  "type": "OrderStatusChanged",
  "id": "evt_124",
  "order_id": "ord_42",
  "from": "placed",
  "to": "paid",
  "occurred_at": "2026-05-19T14:05:30Z"
}
```

Compact; requires consumers to track state. See [Event Sourcing →](./event-sourcing.md).

### Which to use?
- **Thin** for small ecosystems where consumers always have producer access.
- **Fat** for loose coupling across teams or where the producer might be down.
- **Delta** for event sourcing setups.

Many systems combine: a fat `OrderPlaced` and subsequent thin `OrderStatusChanged` deltas.

---

## 6. Schemas and Compatibility

Events live forever. The consumer reading today's event a year from now may be running ancient code. The producer publishing today may produce slightly different events next year. **Schema discipline is mandatory.**

### Formats
- **JSON** with **JSON Schema** — easy, verbose, weak typing.
- **Avro** — compact, strong typing, native schema evolution rules.
- **Protobuf** — compact, strong typing, popular in gRPC world.
- **MessagePack / CBOR** — binary JSON-ish.

For event streams, **Avro** is most common, with **Protobuf** rising. JSON survives in smaller setups.

### Compatibility rules
- **Backward** — new schema can read old data. (Adding optional fields is fine; removing required ones isn't.)
- **Forward** — old code can read new data. (Adding required fields breaks this.)
- **Full** — both directions.

Pick **backward compatibility** as the minimum for event streams. **Full** for the most critical events.

### Schema registry
Confluent Schema Registry, AWS Glue Schema Registry, Apicurio. Producers register schemas; consumers fetch by ID. Compatibility enforced at registration time. Use one.

---

## 7. Event Choreography vs Orchestration

Two ways to coordinate multi-service workflows.

### Choreography
Each service reacts to events independently. No central coordinator.

```
   order-service ──► OrderPlaced ──► inventory  ──► InventoryReserved
                                                          │
                                                          ▼
                                                   payment-service ──► PaymentCharged
                                                                              │
                                                                              ▼
                                                                       fulfillment
```

- **Pros**: maximally decoupled, no central bottleneck.
- **Cons**: hard to see the overall flow ("where is order 42?"); failure handling is distributed; cycles possible.

### Orchestration
A central coordinator (saga orchestrator, workflow engine) drives the flow.

```
   order-service ──► OrderPlaced ──► orchestrator
                                          │
                                  ┌───────┼───────┐
                                  ▼       ▼       ▼
                              inventory payment fulfillment
                              (commanded one step at a time)
```

- **Pros**: explicit, easy to debug, easy to add compensation.
- **Cons**: central coordinator can become bottleneck; tighter coupling to orchestrator schema.

Tools: **Temporal**, **Cadence**, **Camunda**, **AWS Step Functions**, **Netflix Conductor**.

Most large systems use a mix: choreography for simple fan-out, orchestration for complex business workflows. See [Saga Pattern →](./saga-pattern.md).

---

## 8. Patterns Inside EDA

### 8.1 Event bus
A single Kafka cluster (or equivalent) is the central event bus. Every domain publishes to it; every consumer subscribes. The default for org-wide microservice EDA.

### 8.2 Domain events
Events express **business meaning**, not internal data shape changes. `OrderPlaced` not `OrdersTableRowInserted`. The right granularity is "things the business cares about."

### 8.3 Integration events vs domain events
- **Domain events** — inside a bounded context, fine-grained, may contain internal data.
- **Integration events** — across contexts, cleaned-up, stable API.

A common pattern: a service emits domain events internally, transforms them into integration events for the central bus.

### 8.4 Event-carried state transfer
A fat event carries everything a consumer needs. Consumers project to their own local read store. Producer becomes the source of truth for that event type; consumers cache. Reduces synchronous dependencies.

### 8.5 CQRS via events
Writes produce events; reads come from event-derived projections. See [CQRS →](./cqrs.md).

### 8.6 Outbox pattern
Transactional write of DB row + outbox row; a separate process publishes the outbox. Guarantees atomicity between state change and event publish. See [Outbox Pattern →](./outbox-pattern.md).

### 8.7 Saga pattern
Long-running, multi-service transaction broken into local steps with compensating actions. See [Saga Pattern →](./saga-pattern.md).

---

## 9. Idempotency Is Mandatory

Brokers deliver at-least-once typically. Network failures cause retries. Consumers WILL see the same event multiple times. They must be **idempotent**.

Strategies:
- **Natural idempotency** — applying the same write twice gives the same state. `SET status = 'paid'`.
- **Dedup key** — keep a table of `processed_event_ids`. Skip if already processed.
- **Versioned writes** — `UPDATE order SET version=5 WHERE version=4`. Reject if version doesn't match.

For an event-driven system to be reliable, every consumer must handle duplicates. Test it. See [Idempotency →](../03-apis/idempotency.md).

---

## 10. Ordering

Events for the same entity should arrive in order. Cross-entity, usually not necessary.

The standard pattern: **partition by entity ID** (`order_id`, `user_id`). All events for one order go to the same partition; consumer reads them in order. Different orders processed in parallel.

This works because most consistency requirements are **per-entity**, not global.

When global ordering matters (rare): use a single partition. Throughput is capped by what one consumer can do; usually unacceptable.

---

## 11. Observability

The biggest practical pain of EDA: when something goes wrong, where do you look?

### Required tools
- **Distributed tracing** — OpenTelemetry / Jaeger / Datadog APM. Propagate trace IDs through event headers.
- **Structured logging** — JSON logs with `event_id`, `correlation_id`, `entity_id`.
- **Event metadata** — every event has `id`, `occurred_at`, `producer`, `schema_version`, `trace_id`.
- **DLQ monitoring** — dead-letter topics with alerting.
- **Consumer lag dashboards** — per-group, per-partition.
- **End-to-end SLOs** — "from order placed to email sent < 5 sec p99."

Without these, debugging EDA is archaeology.

---

## 12. Worked Example: An E-Commerce Order

```
1. User clicks "Buy".
2. order-service writes order to DB (status: placed).
3. order-service publishes OrderPlaced via outbox pattern.
4. inventory-service consumes OrderPlaced.
   - Reserves stock.
   - Publishes InventoryReserved.
5. payment-service consumes OrderPlaced.
   - Charges card.
   - Publishes PaymentSucceeded (or PaymentFailed).
6. order-service consumes InventoryReserved + PaymentSucceeded.
   - When both received → publish OrderConfirmed.
   - If PaymentFailed → publish OrderCanceled (compensating).
7. fulfillment-service consumes OrderConfirmed → schedules shipment.
8. notification-service consumes OrderConfirmed → sends email.
9. analytics consumes everything → updates dashboards.
```

Notice:
- Each service is independent and self-contained.
- A failure in `notification-service` doesn't block fulfillment.
- The "transaction" across services is the **saga**, not a 2PC. See [Saga Pattern →](./saga-pattern.md).
- Each event is fat enough to be processed standalone.
- Schemas are versioned in a registry.
- Every consumer is idempotent.

---

## 13. Common Mistakes

- **"Events" that are really commands.** `ChargeUser` is a command. `OrderPlaced` is an event.
- **No schema discipline.** Producer changes shape; consumers break next week.
- **No idempotency on consumers.** Duplicates cause double-charge, double-email, double-anything.
- **Synchronous "wait for event" patterns.** "Place order, then wait for OrderConfirmed before returning." You've reinvented sync over async with worse latency.
- **One huge consumer per event type.** Bottleneck. Scale consumer groups.
- **No DLQ.** Bad events block consumption forever.
- **Cross-event ordering assumption.** "OrderConfirmed comes after OrderPlaced" — true only if same partition; otherwise no guarantee.
- **Cyclic event flows.** Service A's event triggers service B, which triggers an event service A consumes, which... oops, infinite loop. Detect cycles in design.
- **Missing trace IDs.** No way to follow a single business action across services.
- **Treating the broker as a database.** Kafka has retention; if you need permanent state, project to a DB.
- **Going fully event-driven for synchronous user actions.** "Place order" needs *some* sync feedback. Hybrid is fine.

---

## 14. When to Choose EDA vs Sync

```
Use synchronous (REST/gRPC) when:
  - The caller needs the result immediately.
  - The operation is read-only or simple.
  - Latency must be sub-100ms.
  - There's no fan-out.

Use events when:
  - Multiple consumers want to react.
  - Producer doesn't need a response.
  - Operation is naturally async (notifications, ETL, analytics).
  - You want replay / history.

Hybrid (most production systems):
  - Sync for the user-facing path that needs immediate response.
  - Events for downstream side effects.
  - Outbox pattern bridges the two.
```

A pragmatic line: **the user's primary action is synchronous; everything that happens because of it is events**.

---

## 15. Cheat Card

```
EVENT          past-tense fact published to a broker
                producer doesn't know consumers

EDA TRADE-OFFS
  WIN          loose coupling, scalability, resilience, replay
  COST         eventual consistency, harder debug, schema discipline,
                duplicate handling, ops complexity

EVENT vs CMD   event = fact, command = instruction
                events use past tense, commands use imperative

SHAPES         thin (notify) · fat (state-carrying) · delta (sourced)

SCHEMA         Avro / Protobuf + registry, backward compat min

COORDINATION   choreography (decentralized) vs orchestration (Temporal)

REQUIRED
  idempotent consumers (at-least-once + retries)
  per-entity ordering via partition key
  DLQ + lag monitoring
  schema registry
  distributed tracing

PITFALLS       commands disguised as events,
                no idempotency, no DLQ, no schema,
                cyclic flows, sync semantics over async

RULE           Use events where the business is async.
                Use sync where the user needs immediate answers.
```

---

## 16. Resources

### Books
- *Building Event-Driven Microservices* — Adam Bellemare.
- *Designing Event-Driven Systems* — Ben Stopford.
- *Enterprise Integration Patterns* — Hohpe & Woolf.
- *Designing Data-Intensive Applications* — Kleppmann (Ch 11).

### Articles
- "Event-Driven Architecture" — Martin Fowler.
- "What do you mean by 'Event-Driven'?" — Martin Fowler.
- "Event-Carried State Transfer" — Martin Fowler.
- "Domain events and event sourcing" — Vaughn Vernon.
- "Events and integration events" — Microsoft architecture blog.

### Documentation
- **Confluent — Event-driven architectures**: <https://www.confluent.io/learn/event-driven-architecture/>
- **AWS EDA patterns**: <https://docs.aws.amazon.com/eventbridge/latest/userguide/eb-event-driven-architecture.html>
- **Azure architecture — Event-driven**: <https://learn.microsoft.com/en-us/azure/architecture/guide/architecture-styles/event-driven>

### Videos
- ByteByteGo — "Event-Driven Architecture".
- Martin Fowler — "The Many Meanings of Event-Driven Architecture".
- Greg Young — "Event Sourcing & CQRS".

### Tools
- **Brokers**: Kafka, Pulsar, RabbitMQ, SQS+SNS, Pub/Sub.
- **Schema registry**: Confluent, Apicurio, AWS Glue.
- **Orchestration**: Temporal, Cadence, Camunda, Step Functions.
- **Tracing**: OpenTelemetry, Jaeger, Datadog APM.

### Adjacent reading
- [Queue vs Pub/Sub vs Stream →](./queue-vs-pubsub-vs-stream.md)
- [Kafka Deep Dive →](./kafka.md)
- [Event Sourcing →](./event-sourcing.md)
- [CQRS →](./cqrs.md)
- [Saga Pattern →](./saga-pattern.md)
- [Outbox Pattern →](./outbox-pattern.md)
- [Delivery Guarantees →](./delivery-guarantees.md)
- [Idempotency →](../03-apis/idempotency.md)
- [Microservices Architecture →](../14-architecture/microservices.md)

---

*Previous:* [← Message Brokers](./message-brokers.md)  |  *Next:* [Event Sourcing →](./event-sourcing.md)
