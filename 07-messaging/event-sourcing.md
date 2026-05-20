# Event Sourcing

> **TL;DR** — **Event sourcing** stores the **history of changes** as an immutable, append-only sequence of events rather than the **current state** as mutable rows. Current state is derived by replaying events. The benefits are immense: **perfect audit log**, **time-travel queries** ("what did the order look like on Tuesday?"), **temporal debugging** ("how did we get into this state?"), **easy projections** (rebuild a read model differently), and **natural fit for event-driven architectures**. The costs are real: **everything is harder than CRUD** — querying current state, schema evolution, snapshots, eventual consistency, and the learning curve. Event sourcing is a **commitment**, not a feature flag. Use it where audit, history, or replay is core to the domain (banking, billing, healthcare, e-commerce orders, regulatory compliance). Use plain CRUD when it isn't.

---

## 1. The Core Idea

Traditional CRUD stores the **current state**:

```sql
UPDATE accounts SET balance = 80 WHERE id = 42;
```

After this, you have no idea what the previous balance was, why it changed, or when. The row is overwritten.

Event sourcing stores the **events that produced the state**:

```
Event 1: AccountOpened    { id: 42, owner: alice, initial: 0 }
Event 2: MoneyDeposited   { id: 42, amount: 100 }
Event 3: MoneyWithdrawn   { id: 42, amount: 20 }
```

Current state (balance = 80) is **derived** by folding the events:

```python
state = {}
for event in events:
    state = apply(state, event)
# state = {balance: 80, owner: alice}
```

The events are the source of truth. State is a projection.

---

## 2. Why Event Sourcing

### 2.1 Perfect audit log
Every change has its own event, timestamped, attributed. Compliance, forensics, debugging — solved by definition.

### 2.2 Time-travel queries
"What was the state on March 15?" — fold events up to that timestamp.

### 2.3 Replay for new projections
Add a new read model. Replay all events into it. Done. No expensive backfills from incomplete data.

### 2.4 Natural fit for event-driven systems
The events you store are the events you publish. The event log IS the integration contract.

### 2.5 Easier debugging in some cases
"Why is the balance wrong?" Look at the event history. The bug is somewhere in apply() or in a published event.

### 2.6 Decoupled reads and writes
Reads use projections optimized for the query. Writes use events optimized for the domain. See [CQRS →](./cqrs.md).

### 2.7 Models complex domains well
Insurance, banking, supply chain, healthcare — domains where the history is part of the value. The state-only model loses information they care about.

---

## 3. Why Not Event Sourcing

### 3.1 Everything is harder than CRUD
- Querying current state requires a projection.
- "Find all users named Alice" can't be done over the event log; needs a projection.
- Joins are projections.
- Reports are projections.

### 3.2 Schema evolution
Events live forever. Three years from now, you change a field. Old events don't have it. Migration strategies needed:
- Upcasters (translate old events to new shape at read time).
- Backfill new events (rewrite history; usually wrong).
- Versioned event types (`OrderPlaced_v1`, `OrderPlaced_v2`).

### 3.3 Snapshots required for performance
Replaying 10M events to compute current state every request is slow. Periodic snapshots help, but add complexity.

### 3.4 Eventual consistency
The write side (events) commits before the read side (projections) is updated. Reads can lag.

### 3.5 Mental model
Developers used to CRUD struggle for months. Frameworks help but don't eliminate the curve.

### 3.6 Tooling
Standard ORMs don't apply. Need libraries (EventStoreDB, Axon, marten, Akka Persistence) or hand-rolled.

The honest take: **most systems should NOT be event-sourced.** Use it where the domain demands it.

---

## 4. The Anatomy

### 4.1 Event store
The append-only log of events. Options:
- **Purpose-built**: EventStoreDB, Marten (Postgres-based), Axon Server.
- **Kafka** — topics with infinite retention or compaction.
- **DynamoDB / Postgres** — append-only tables with `(aggregate_id, version, type, payload, timestamp)`.

Schema (in Postgres for clarity):
```sql
CREATE TABLE events (
  aggregate_id UUID NOT NULL,
  version      INT  NOT NULL,
  event_type   TEXT NOT NULL,
  payload      JSONB NOT NULL,
  metadata     JSONB,
  occurred_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (aggregate_id, version)
);
CREATE INDEX events_occurred_at_idx ON events(occurred_at);
```

The `(aggregate_id, version)` unique constraint enforces optimistic concurrency: writers specify the expected next version; insert fails if someone else got there first.

### 4.2 Aggregate
The unit of consistency. Usually a single business entity (`Order`, `Account`). Events are grouped by aggregate. Loading an aggregate = reading all its events; saving = appending new ones.

### 4.3 Projections (read models)
Derived state, computed by consuming the event stream. Stored in the database optimal for queries — SQL table, Elasticsearch index, Redis hash, materialized view.

A given event can feed many projections.

### 4.4 Snapshots
Periodic captures of an aggregate's current state. Loading an aggregate = load latest snapshot + events since. Speeds up replay.

```
snapshot @ version 5000  +  events 5001..5234  = current state
```

Snapshot every N events (1000, 5000) or on time interval. Snapshots are an optimization, not a correctness requirement.

---

## 5. The Write Path

```
1. Receive command: PlaceOrder(user, items)
2. Load aggregate: replay events for order_id (or load snapshot + tail).
3. Validate command against current state (e.g., user has not been banned).
4. Create new event(s): OrderPlaced.
5. Append events to event store with expected version (optimistic CC).
6. On success, publish events to broker for projections / other services.
```

```python
def handle_place_order(cmd):
    events = store.load(cmd.order_id)
    order = replay(events, Order())
    new_events = order.handle(cmd)  # may raise if invalid
    store.append(cmd.order_id, expected_version=len(events), new_events)
    publish(new_events)
```

The `expected_version` check prevents two writers from racing — exactly the optimistic concurrency pattern from databases.

---

## 6. The Read Path

```
1. Subscribe to event stream.
2. For each event, update one or more projections.
3. Queries hit the projection.
```

```python
def project_orders_summary(event):
    if event.type == "OrderPlaced":
        db.insert("orders_summary", id=event.order_id, status="placed", total=event.total)
    elif event.type == "OrderPaid":
        db.update("orders_summary", id=event.order_id, status="paid")
    ...
```

Projections are idempotent: applying the same event twice produces the same projection state. (Track the last-applied event offset per projection.)

---

## 7. Versioning Events

Events are forever. They need to evolve safely.

### 7.1 Add only optional fields
The simplest path. Old events have no value; new code defaults the field.

### 7.2 Upcasters
A function that takes an old event and returns the new shape at read time.

```python
def upcast(event):
    if event.version == 1 and event.type == "OrderPlaced":
        return {**event, "currency": "USD"}  # default for old events
    return event
```

This keeps the event store unchanged while letting consumers see a unified shape.

### 7.3 Versioned event types
Introduce `OrderPlaced_v2`. Old consumers handle v1; new consumers handle both. Eventually retire v1.

### 7.4 Never rewrite history
"Let's update old events to have the new field with a backfilled value." This breaks the immutability promise. Use upcasters or new event types instead.

---

## 8. Snapshots

Without snapshots, replaying 1M events to compute one aggregate is slow.

```
snapshot table:
  aggregate_id  | version | state
  order_42      | 5000    | {status: paid, items: [...]}

load(order_42):
  snapshot = SELECT FROM snapshots WHERE aggregate_id = 'order_42'
            ORDER BY version DESC LIMIT 1
  events   = SELECT FROM events WHERE aggregate_id = 'order_42'
            AND version > snapshot.version
            ORDER BY version
  state = apply(snapshot.state, events)
```

Snapshots are themselves a projection — derived data. If projections need to be rebuilt, snapshots can be too. Don't treat snapshots as canonical state.

How often? When folding `N` events takes too long. Typical: every 1k–10k events, or every hour for chatty aggregates.

---

## 9. Event Sourcing + CQRS

The natural pairing.

```
COMMANDS
   ┌─────────────┐
   │ command-side│ ──── appends events ────► event store
   └─────────────┘
                                                │
                                                ▼
                                        consumer (projector)
                                                │
                                                ▼
   READ MODELS                          ┌───────────────┐
   queries ◄──────────────────────────  │ read DB(s),   │
                                        │ search, cache │
                                        └───────────────┘
```

Writes go through the event-sourced aggregate. Reads come from projections optimized for the query (multiple read models for different views). See [CQRS →](./cqrs.md).

---

## 10. Common Patterns

### 10.1 Event-sourced aggregate + outbox publish
Append events to event store and publish via outbox in the same transaction. Atomic. See [Outbox Pattern →](./outbox-pattern.md).

### 10.2 Projection rebuild
Need a new read model? Replay the event stream from offset 0. New projection builds itself; switch over once caught up. Magical compared to CRUD migrations.

### 10.3 Time travel
"What did this order look like on Tuesday?" — replay events up to that timestamp into a fresh projection.

### 10.4 Saga state
Saga orchestrator can be event-sourced: every transition is an event. The orchestrator's state is derived from its event log. Easy to debug and resume.

### 10.5 Audit and compliance
The event log IS the audit log. Regulators love this.

---

## 11. Storage Options

| Tool | Approach | Strength |
|---|---|---|
| **EventStoreDB** | Purpose-built, native streams + subscriptions | Designed for ES from day one |
| **Marten** (.NET / Postgres) | Postgres-based, schema-less ES + document DB | Pragmatic; PG underneath |
| **Axon Server** (JVM) | JVM ecosystem; works with Axon Framework | Tightly integrated |
| **Kafka + log compaction** | Topic-as-store | Already have Kafka? Use it |
| **DynamoDB** | Append-only table on aggregate_id + version | Cloud-native, scalable |
| **Postgres** | DIY event table | Familiar; works |
| **Akka Persistence** | Actor-model integration | Scala/Java actor system |

Kafka-based event sourcing: each aggregate maps to a partition key; events stream in. Compacted topics for snapshots. Caveats: querying a single aggregate's events isn't Kafka's strong suit (it's a stream, not an indexed store) — pair with a DB.

---

## 12. Worked Example: Bank Account

### Events
- `AccountOpened(id, owner, opened_at)`
- `Deposited(id, amount, source)`
- `Withdrawn(id, amount, target)`
- `Overdrafted(id, attempted, balance)`
- `AccountFrozen(id, reason, frozen_at)`
- `AccountUnfrozen(id, unfrozen_at)`
- `AccountClosed(id, closed_at)`

### Aggregate state
```python
class Account:
    id: str
    owner: str
    balance: int = 0
    frozen: bool = False
    closed: bool = False
```

### Apply
```python
def apply(state, event):
    if event.type == "AccountOpened":
        return Account(id=event.id, owner=event.owner)
    if event.type == "Deposited":
        state.balance += event.amount; return state
    if event.type == "Withdrawn":
        state.balance -= event.amount; return state
    if event.type == "AccountFrozen":
        state.frozen = True; return state
    # ...
```

### Command
```python
def withdraw(account, amount):
    if account.closed:    raise AccountClosedError
    if account.frozen:    raise AccountFrozenError
    if account.balance < amount:
        return [Overdrafted(account.id, amount, account.balance)]
    return [Withdrawn(account.id, amount)]
```

### Projection (current accounts table)
```sql
INSERT INTO accounts_current (id, owner, balance, frozen, closed)
ON CONFLICT (id) DO UPDATE SET ...
```

### What you get
- Historical balance at any moment.
- Every fraud event has a permanent record.
- New report "average balance over last 30 days" = a projection.
- Regulatory audit: just hand over the events.

### What's hard
- "Find all accounts with balance > $1M" — needs the projection, fast.
- Renaming an event field three years from now — needs upcasters.

For a banking domain, the trade-off is worth it. For most CRUD admin apps, it isn't.

---

## 13. Pitfalls and Anti-Patterns

### 13.1 Events that are really commands
`UserShouldBeBilled` is wrong; `BillingRequested` or `BillingTriggered` is closer to a fact.

### 13.2 Internal-implementation events
`UsersTableRowUpdated` is a CDC event, not a domain event. Domain events express business meaning.

### 13.3 Big-bang event redesign
Going from CRUD to ES across the whole system. Almost never works. Pick one bounded context (orders, billing) and event-source that; leave others CRUD.

### 13.4 Ignoring projections at scale
Projections must keep up with the write rate. Plan for parallel projectors and partitioning of read models.

### 13.5 Trying to query the event store directly
"How many orders are placed?" — not the event store's job. Project to a queryable store.

### 13.6 Forgetting eventual consistency
"User placed an order, immediately queried orders, got none." Yes. Projection hasn't caught up. UI must handle.

### 13.7 Schema drift without discipline
Events are forever. Schemas evolve carefully. No "let's just update the old events."

### 13.8 Storing only the current state in events
`OrderUpdated(new_state)` defeats the purpose. Granular events that show *what changed* are the whole point.

### 13.9 Replay too eagerly
"Let's replay from scratch every deploy" — works at small scale, dies at large. Use snapshots.

---

## 14. Common Mistakes

- **Event sourcing the whole system.** Pick a bounded context; leave the rest CRUD.
- **Designing events around the database, not the business.** Domain events express business semantics.
- **No snapshots.** Replay times grow linearly with history. Death-by-slow-load.
- **Mutable events.** Defeats immutability. Use upcasters or versioned types.
- **No clear aggregate boundary.** An event spans multiple aggregates → consistency unclear.
- **Projections that aren't idempotent.** Replay produces wrong state.
- **Storing events without metadata.** No `occurred_at`, no `correlation_id`, no `version`. Debugging nightmare.
- **Treating Kafka as the canonical store** without thinking about retention. Compaction or infinite retention required.
- **Trying to use ES with framework-level magic** before understanding the fundamentals. Frameworks help; they don't replace clear thinking about events.
- **No team buy-in.** Mid-project switch from ES to CRUD because half the team doesn't get it. Commit upfront.

---

## 15. When to Choose Event Sourcing

```
USE ES when:
  - Audit, history, compliance is core (banking, healthcare, legal).
  - You need time-travel queries.
  - Multiple read models from same data.
  - The domain naturally produces events.
  - You're already heavily event-driven.
  - You have time/team to commit.

AVOID ES when:
  - The domain is simple CRUD (users, blog posts).
  - Strong read-after-write consistency required everywhere.
  - Team is unfamiliar and project is time-pressured.
  - You're choosing for "cool" not "right."
```

---

## 16. Cheat Card

```
WHAT          store events (history), derive state by replay

EVENTS        past-tense, immutable, business-meaningful
              never internal implementation details

AGGREGATE     unit of consistency; load all events for aggregate

WRITE PATH    load aggregate → validate → append events → publish

READ PATH     project events → query optimized read model

SNAPSHOTS     periodic state captures; speed up loading

CQRS          natural pairing; separate write and read models

VERSIONING    additive + upcasters + versioned event types
              never rewrite history

PAIRED WITH   outbox (atomic publish), saga (orchestration)

WHEN GOOD     audit, history, time-travel, complex domain

WHEN BAD      simple CRUD, time-pressured team, sync everywhere

PITFALLS      no snapshots, mutable events, replay-without-snapshot,
              treating event store as query DB, schema drift

RULE          The event log is the source of truth.
              Everything else is a projection.
```

---

## 17. Resources

### Books
- *Versioning in an Event Sourced System* — Greg Young (short, essential).
- *Domain-Driven Design* — Eric Evans (the domain modeling context for ES).
- *Implementing Domain-Driven Design* — Vaughn Vernon (practical DDD + ES).
- *Designing Event-Driven Systems* — Ben Stopford.
- *Event Sourcing with Kafka* — Adam Bellemare.

### Articles
- "What is event sourcing?" — Martin Fowler.
- "Event Sourcing" — Greg Young's foundational talks.
- "Things to consider before adopting event sourcing" — various.
- "The world's simplest event sourcing example" — eventstore.com.

### Videos
- Greg Young — "Event Sourcing", "CQRS and Event Sourcing".
- Martin Kleppmann — "Turning the database inside-out".
- Eric Evans — "DDD and event sourcing".

### Documentation
- **EventStoreDB**: <https://www.eventstore.com/eventstoredb>
- **Marten (Postgres)**: <https://martendb.io/events/>
- **Axon Framework**: <https://docs.axoniq.io/>
- **Akka Persistence**: <https://doc.akka.io/docs/akka/current/typed/persistence.html>

### Tools
- **EventStoreDB**, **Marten**, **Axon**, **Akka Persistence**.
- **Kafka with compacted topics + Kafka Streams**.
- **AWS DynamoDB Streams**.

### Adjacent reading
- [Event-Driven Architecture →](./event-driven-architecture.md)
- [CQRS →](./cqrs.md)
- [Saga Pattern →](./saga-pattern.md)
- [Outbox Pattern →](./outbox-pattern.md)
- [Kafka Deep Dive →](./kafka.md)
- [Bounded Contexts & Aggregates →](../14-architecture/bounded-contexts.md)
- [Domain-Driven Design →](../14-architecture/ddd.md)

---

*Previous:* [← Event-Driven Architecture](./event-driven-architecture.md)  |  *Next:* [CQRS →](./cqrs.md)
