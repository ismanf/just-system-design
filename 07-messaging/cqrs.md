# CQRS — Command Query Responsibility Segregation

> **TL;DR** — **CQRS** separates the **write model** (commands) from the **read model** (queries). Instead of one model serving both, you have a command side that validates and persists changes, and one or more read sides optimized for specific queries. The two sides may share a database, or — increasingly — use entirely different stores (Postgres for writes, Elasticsearch / Redis / materialized views for reads). The wins are **independent scaling**, **query-tuned read models**, and a **clean separation of write semantics from read shape**. The costs are **eventual consistency** (the read side lags the write side), **operational complexity** (more stores, projectors, schema changes), and **developer overhead** (two models instead of one). CQRS shines for **complex domains with very different read and write patterns** — analytics dashboards, search, multi-view UIs — and is **overkill for simple CRUD**. CQRS pairs naturally with [Event Sourcing →](./event-sourcing.md) but they're independent: you can do either alone.

---

## 1. The Idea in One Picture

```
TRADITIONAL CRUD
   client ──► one model (CRUD repo) ──► one database

   reads and writes share the schema, the indexes, the ORM,
   the queries, and the contention.


CQRS
   client ──commands──► WRITE side ──► write store
                                          │
                                          publishes events / CDC
                                          │
                                          ▼
                                     projector(s)
                                          │
                                          ▼
   client ──queries───► READ side ◄── read store(s)
                          (denormalized, search-optimized, cached)
```

The write side and read side are **different code paths**, potentially **different storage technologies**, and almost always **different data shapes**.

---

## 2. Why Split

### 2.1 Reads and writes have different shapes
- Writes need normalization, integrity, transactions.
- Reads need denormalization, indexes, fan-out.

A single model compromises both. CQRS lets each be optimal.

### 2.2 Scaling asymmetry
- Reads typically dominate (10:1, 100:1).
- Read side can scale independently — multiple Elasticsearch indexes, Redis caches, materialized views.
- Write side stays small.

### 2.3 Different queries need different stores
- "Show me my order history" → relational.
- "Search products by partial text" → Elasticsearch.
- "Top 10 sellers this hour" → time-series or pre-aggregated.

In CRUD you contort one DB. In CQRS each store solves its problem.

### 2.4 Write-side correctness; read-side performance
The write side enforces invariants ("can't withdraw past zero"). The read side is denormalized and tolerant to staleness ("user dashboard").

### 2.5 Independent evolution
Add a new read model? Build a new projection without touching writes. Drop a read view? Stop projecting. Independent deploy.

---

## 3. CQRS vs Event Sourcing

A common confusion. They're separate ideas.

| Pattern | What it is | Required? |
|---|---|---|
| **CQRS** | Separate read and write models | Standalone or with ES |
| **Event Sourcing** | Store events; derive state | Standalone or with CQRS |
| **Both** | Write side appends events; read side projects | Common modern combo |

You can have:
- **CRUD + CQRS** — two models, both backed by Postgres tables. The write side updates one table; a trigger / CDC populates a denormalized read table.
- **Event Sourcing without CQRS** — events drive a single read view that's also the write target's projection.
- **ES + CQRS** — the canonical combo. Writes produce events; events project into read models.

For this page, focus on **CQRS as the separation pattern**, with ES as the most common pairing.

---

## 4. Levels of CQRS

### 4.1 Same database, different models
Share Postgres. Writes update normalized tables. Reads use views, materialized views, or denormalized tables maintained by triggers.

- **Pros**: simple ops; one DB to manage.
- **Cons**: no scaling separation; eventual consistency limited to view refresh.

This is "CQRS lite" — many systems benefit just from this.

### 4.2 Same database, async projection
Writes commit to one schema. A change-data-capture (CDC) or domain-event consumer writes to a denormalized read schema in the same DB.

- **Pros**: cleaner separation; possible to add read schemas without write-side migration.
- **Cons**: still one DB for capacity.

### 4.3 Different databases (full CQRS)
Writes to Postgres. Reads from Elasticsearch + Redis + materialized BigQuery tables. Events / CDC update read stores.

- **Pros**: every read model perfectly tuned.
- **Cons**: many stores; ops burden; consistency lag.

The "level" you adopt is the dial. Most teams start at 4.1 and only graduate as needs grow.

---

## 5. Write Side Anatomy

The command side:

```
1. Accept command (Place Order, Update Profile).
2. Load aggregate or current state.
3. Validate against business rules.
4. Persist the change (UPDATE or APPEND event).
5. Publish event(s) for projections / external consumers.
```

Commands are imperative and may fail (validation, optimistic concurrency, conflicts). They return either success + new state, or failure + reason. The write side is the system of record.

### Write store options
- **Postgres / MySQL** — most cases. ACID, transactions, foreign keys.
- **DynamoDB** / **MongoDB** — when scale demands a key-value model.
- **Event store** (EventStoreDB, Postgres-as-event-log) — for event sourcing.

---

## 6. Read Side Anatomy

The query side:

```
1. Receive query (Get Profile, Search Orders, Dashboard Stats).
2. Hit the read store directly — no business logic, no validation.
3. Return data.
```

Queries are idempotent and never modify state. The read side serves data; it doesn't enforce rules.

### Read store options
- **Materialized views** in the same DB — simplest.
- **Separate SQL replica** — read-only, ETL'd from writes.
- **Elasticsearch / OpenSearch** — full-text search.
- **Redis** — hot lookups, top-K lists.
- **OLAP store** (ClickHouse, Druid, BigQuery) — aggregations.
- **DocumentDB** — denormalized views.

A given query goes to the read store best suited for it.

---

## 7. Projecting Events into Read Models

```
   write commits ──► event/CDC stream ──► projector
                                              │
                                              ▼
                                          read store
```

Properties a projector must have:
- **Idempotent** — applying same event twice produces same state.
- **Order-aware** within a partition (per-aggregate).
- **Resumable** — track last-applied offset; recover from crashes.
- **Replayable** — can rebuild the read model from scratch by replaying events.

### Idempotency tactics
- Upsert by ID (INSERT ... ON CONFLICT).
- Version-checked update (`WHERE current_version < event.version`).
- Maintain `last_event_id_processed` per projection.

### Replay tactics
For a new read model:
1. Define the new schema.
2. Start a projector that reads from offset 0.
3. Catch up to current.
4. Switch reads to the new model.

This is **way easier** than backfilling in CRUD systems.

---

## 8. Eventual Consistency

The hardest reality of CQRS: writes commit before reads see them.

```
t=0      user POST /order
t=10ms   write side commits, returns 201
t=10ms   read side still doesn't have it
t=200ms  projector catches up
t=200ms+ user GET /orders/42 → finds it
```

For the user: "I just placed an order, but it's not in my list."

### Mitigations
- **Synchronous projection** for critical views — projector runs in the same transaction. Defeats some CQRS benefits but works.
- **Read-your-writes via session sticky** — if the user just wrote, route their next reads to the write model.
- **Return the new entity in the write response** — the UI can show it immediately without re-querying.
- **Polling / WebSocket update** — the UI subscribes for changes after a write.
- **Honest UI** — show "pending" until the read model catches up.

Most teams use a mix. Stripe-style: critical writes propagate within milliseconds; non-critical accept the lag.

---

## 9. Worked Example: An E-Commerce System

### Domain
Users browse a catalog, add to cart, place orders, search by various criteria.

### Write side
- **Postgres** holds: users, products, orders.
- Commands: `CreateUser`, `UpdateProduct`, `PlaceOrder`, `UpdateOrderStatus`.
- Each emits domain events: `UserCreated`, `ProductUpdated`, `OrderPlaced`, etc.

### Events flow to projectors

```
events ──► projector A → Elasticsearch (search "products by partial text")
       ──► projector B → Redis (top-10 sellers, hot-lookups)
       ──► projector C → BigQuery (analytics, aggregate reports)
       ──► projector D → Postgres denormalized "order_history_view"
                          (per-user order list with product snapshots)
```

### Read side
- Search: hit ES.
- "My orders" page: hit `order_history_view`.
- Admin dashboard: hit BigQuery.
- Product detail: hit Postgres products table directly (or Redis cache).

### Benefits
- Search auto-completion works fast because of ES.
- "My orders" never joins 6 tables.
- Analytics doesn't touch the write DB.
- Add a new "trending products" view? Just add a projector.

### Costs
- Four read stores to operate.
- Schema-evolution discipline across event consumers.
- Lag visible in some flows.

For an e-commerce system, this is worth it. For an internal admin tool, it isn't.

---

## 10. Patterns

### 10.1 Single-DB CQRS
- One Postgres.
- Writes to normalized tables.
- Reads from materialized views or denormalized read tables maintained by triggers / event consumers.
- Easy starting point.

### 10.2 Polyglot persistence
- Write store: Postgres.
- Search: Elasticsearch.
- Cache: Redis.
- Analytics: BigQuery.
- Each populated via events / CDC.

### 10.3 ES + CQRS + Projections
- Events are the write log.
- Projectors build many read models.
- The default for serious event-sourced systems.

### 10.4 Read replicas as light CQRS
- Use DB read replicas for queries.
- Not full CQRS but addresses the "writes contend with reads" problem.
- Often the right pragmatic step before going harder.

---

## 11. Operational Concerns

### Schema migrations
- Write side: normal migrations.
- Read side: drop and rebuild from events. Best in event-sourced setups; in CRUD-CQRS, more complex.

### Backfill
- Need a new field on a read model? Replay events (if event-sourced) or backfill from write side (CRUD).

### Lag monitoring
- Track `read_model.last_event_offset` vs `event_store.latest_offset`. Alert on growth.

### Projector failures
- A bug in a projector → stale read model. Roll back projector, fix, replay.
- Don't update projected data manually — it'll drift from the event log.

### Schema evolution across projections
- Many projectors consume the same events. Adding a field is fine; removing one breaks consumers.
- Use schema registry + compatibility checks.

---

## 12. When to Use CQRS

### Reach for CQRS when:
- **Read and write workloads have very different shapes** — analytics, search, multi-view UIs.
- **You want polyglot persistence** — different stores for different queries.
- **Independent scaling matters** — reads explode; writes stay small.
- **You're already event-driven / event-sourced**.
- **You want to add new read views without touching writes**.

### Avoid CQRS when:
- **The model is simple** — a CRUD admin app, a blog, a basic SaaS.
- **The team is small and new to it**.
- **You need strong read-after-write consistency** — possible but defeats benefits.
- **Operational maturity is low** — you'll struggle with multi-store consistency.

A pragmatic ladder:
1. Start with CRUD.
2. Add read replicas when reads strain the write DB.
3. Add materialized views or denormalized read tables when query shape diverges.
4. Add specialized read stores (ES, Redis) when needed.
5. Adopt full CQRS + event sourcing when domain demands.

Step 5 is rare; the others are common.

---

## 13. Common Mistakes

- **Adopting CQRS for simple CRUD.** Massive overhead for no benefit.
- **Treating the read store as authoritative.** It's a projection. Source of truth is the write side / event log.
- **Updating read stores directly without re-deriving from events.** Drift. State diverges. Hard to recover.
- **Synchronous projection for everything.** Loses the scaling benefit.
- **No replay capability.** New read model = expensive backfill from incomplete data.
- **Forgetting eventual consistency in the UI.** "I placed an order; it's not showing." UI must handle.
- **One projector per event, monolithic.** Failure cascades. Multiple specialized projectors, each idempotent.
- **No event schema discipline.** Adding a field breaks 5 projectors a week from now.
- **Conflating CQRS with event sourcing.** They're independent; understand both.
- **No monitoring of projection lag.** Reads silently stale.

---

## 14. Cheat Card

```
CQRS         separate write model (commands) from read model (queries)

WRITE SIDE   command → validate → persist → publish event
              source of truth; ACID; small footprint

READ SIDE    query → hit projected store
              denormalized, query-tuned, eventually consistent

PAIRS WITH   event sourcing (common), CDC, materialized views

LEVELS       single DB with views → async projections →
              polyglot read stores

CONSISTENCY  eventual; UI must handle
              mitigations: sync projection on critical paths,
              return new entity in write response, polling/SSE

WHEN GOOD    differing read/write shapes, independent scaling,
              polyglot persistence, event-driven systems

WHEN BAD     simple CRUD, strong consistency everywhere,
              small team without ES familiarity

PITFALLS     CQRS for the sake of it, direct read-store updates,
              no replay path, no projection-lag monitoring

RULE         Reach for CQRS when one model can't serve both
              read and write workloads well. Not before.
```

---

## 15. Resources

### Books
- *Implementing Domain-Driven Design* — Vaughn Vernon.
- *Patterns, Principles, and Practices of Domain-Driven Design* — Scott Millett, Nick Tune.
- *Designing Data-Intensive Applications* — Kleppmann (the derived-data chapter).
- *Microservices Patterns* — Chris Richardson.

### Articles
- "CQRS" — Martin Fowler.
- "CQRS Documents" — Greg Young (the foundational early writeups).
- "CQRS and ES — when to use, when to avoid" — various.
- "Event Sourcing and CQRS" — Microsoft architecture docs.

### Videos
- Greg Young — "CQRS and Event Sourcing".
- Martin Fowler — "What is CQRS?".
- Udi Dahan — "CQRS and beyond".

### Documentation
- **Microsoft — CQRS pattern**: <https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs>
- **AWS — CQRS pattern**: <https://docs.aws.amazon.com/prescriptive-guidance/latest/cloud-design-patterns/cqrs.html>
- **Axon Framework CQRS docs**.

### Tools
- **Event stores**: EventStoreDB, Axon Server, Marten.
- **Projectors**: Kafka Streams, Flink, Debezium + custom consumers.
- **Read stores**: Elasticsearch, Redis, ClickHouse, BigQuery, Postgres materialized views.

### Adjacent reading
- [Event Sourcing →](./event-sourcing.md)
- [Event-Driven Architecture →](./event-driven-architecture.md)
- [Outbox Pattern →](./outbox-pattern.md)
- [Kafka Deep Dive →](./kafka.md)
- [Change Data Capture (CDC) →](../04-databases/cdc.md)
- [Read Replicas & Write-Through Patterns →](../04-databases/read-replicas.md)
- [Search Engines →](../04-databases/search-engines.md)

---

*Previous:* [← Event Sourcing](./event-sourcing.md)  |  *Next:* [Delivery Guarantees →](./delivery-guarantees.md)
