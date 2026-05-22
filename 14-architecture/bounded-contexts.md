# Bounded Contexts & Aggregates

> **TL;DR** — A **bounded context** is the linguistic and conceptual island within which a domain model and its terms are consistent. Outside the boundary, the same word may mean something different. Bounded contexts are the most important DDD concept because they're how you **decompose** a system — into modules, services, or teams — and they answer the otherwise-impossible question: *"where should this service boundary go?"* Within a bounded context, **aggregates** are the smaller boundaries — clusters of objects (entity + value objects) treated as a single unit for **consistency**. The aggregate root is the only entry point; all changes go through it; one transaction = one aggregate. Get these two boundaries right and most architecture problems get smaller. Get them wrong and you spend the next decade fighting the model.

---

## 1. Why These Two Concepts Matter

In any system bigger than one team or one module, two questions dominate:

1. **Where do we put the cuts** between services, modules, or teams?
2. **What can be atomically consistent**, and what must be eventually consistent?

DDD's answer:
- **Bounded contexts** answer question 1.
- **Aggregates** answer question 2.

Almost every architectural mistake — distributed monolith, shared databases, sprawling microservices, twisty business logic — traces back to getting one of these wrong.

---

## 2. Bounded Contexts

A **bounded context** is a *boundary within which a domain model and its terms are unambiguous*. Inside the boundary, `Customer` means one specific thing. Outside, `Customer` might exist but mean something different.

### The classic example

```
Sales context:
  Customer = a person we're actively quoting; we track preferences, contact attempts, demos
  Order = a draft + confirmed + lost order

Fulfillment context:
  Customer = a shipping recipient; we track address, package status
  Order = a list of items to pick, pack, ship

Billing context:
  Customer = a paying entity; we track payment methods, invoices, dunning state
  Order = a transaction that needs payment + accounting entries
```

Same words, different concepts. Forcing one canonical model across all three contexts (an "enterprise customer schema") is the SOA-era mistake that DDD specifically rejected. Each context has its own model, its own language, its own database.

### Bounded contexts are linguistic, not just technical

The boundary is wherever a term's meaning shifts. The signs:

- Two teams describing the same word differently.
- Disagreement on what fields a concept should have.
- A schema field meaning different things to different consumers.
- "Domain experts" disagreeing because they're experts in different sub-domains.

Each shift is a candidate boundary.

### Bounded contexts → modules / services

A bounded context becomes:

- A **module** in a modular monolith.
- A **service** (or small cluster of services) in microservices.
- A **schema** in a shared DB (if you must — usually a smell).
- A **team's ownership area** in the org.

This is the most useful output of DDD: **service boundaries follow contexts, not entities or tech layers**. Sales, Fulfillment, Billing as three contexts → three services (or three modules). Not "User service, Order service, Address service" — those slice across the domain in ways that produce coupling.

See [Microservices Architecture →](./microservices.md) and [Strangler Fig Pattern →](./strangler-fig.md) for why this matters in practice.

### Context map — how contexts relate

Two contexts almost always need to interact. The **context map** describes the relationships:

| Relationship | Meaning |
| --- | --- |
| **Shared Kernel** | Two contexts share a small, jointly-maintained model. Rare. Fragile. |
| **Customer / Supplier** | Downstream context's needs influence the upstream's roadmap. |
| **Conformist** | Downstream just uses upstream's model as-is, without translation. |
| **Anti-Corruption Layer (ACL)** | Downstream translates upstream's model into its own to avoid contamination. |
| **Open Host Service** | Upstream offers a stable public protocol/API; many downstreams consume it. |
| **Published Language** | Upstream publishes a formal contract (JSON Schema, OpenAPI, Avro, Protobuf). |
| **Separate Ways** | No integration — cheaper to keep apart. |
| **Big Ball of Mud** | No clear model. Often the legacy. |

Drawing a context map is one of the most powerful design exercises. It exposes:
- Where the **anti-corruption layers** must live.
- Which teams must collaborate vs which can move independently.
- Where the **published languages** (stable contracts) belong.

### Subdomains vs bounded contexts

These overlap but are distinct:

- **Subdomain** is a part of the **business** (Sales, Fulfillment, Billing).
- **Bounded context** is a part of the **software** (the Sales service, the Billing module).

In a clean design they often correspond 1:1. In legacy systems they often don't. Mapping subdomains to bounded contexts is the strategic work of DDD.

See [Domain-Driven Design →](./ddd.md).

---

## 3. Aggregates — The Inner Boundary

Within a bounded context, the next boundary is the **aggregate**.

An aggregate is **a cluster of associated objects treated as a single unit for the purpose of consistency**. It has:

- **An aggregate root** — one entity that all outsiders reference; the public face.
- **Internal entities and value objects** — accessible only via the root.
- **Invariants** — rules that must always hold (e.g., "order total = sum of lines").
- **One transactional boundary** — one transaction may modify one aggregate; multi-aggregate changes are eventually consistent.

```
ORDER (aggregate root)
  ├── OrderLine[]    (entities within aggregate)
  ├── ShippingAddress (value object)
  ├── Total           (value object — computed from lines)
  └── Status          (value object — enum)

Invariants:
  - Total = sum(OrderLine.subtotal)
  - Status transitions follow a state machine
  - Once shipped, no new lines may be added
```

### Aggregate rules (Vaughn Vernon's "Effective Aggregate Design")

1. **Reference other aggregates only by ID**, not by direct object reference.
2. **Modify only one aggregate per transaction.** Multi-aggregate changes use domain events and eventual consistency.
3. **Keep aggregates small.** A common rule: as small as possible while preserving invariants.
4. **Use the aggregate root as the only entry point.** Outsiders don't touch internal entities directly.
5. **Apply invariants atomically.** Inside the aggregate, all changes are transactional.

### Why one-aggregate-per-transaction matters

Imagine `Order` and `Customer` are separate aggregates, and you want to place an order while decrementing the customer's loyalty points:

```
BEGIN;
INSERT INTO orders (...) VALUES (...);     -- modifies Order aggregate
UPDATE customers SET points = points - 10  -- modifies Customer aggregate
  WHERE id = ?;
COMMIT;
```

In a single-DB monolith, this might "work" — but in a multi-service / distributed system it doesn't. The discipline forces you to think clearly: which is the master? Will you publish an `OrderPlaced` event that the Customer aggregate reacts to, eventually decrementing points? In most cases, yes — and that's correct.

This pattern scales out naturally to microservices. See [Saga Pattern →](../07-messaging/saga-pattern.md), [Event-Driven Microservices →](./event-driven-microservices.md).

### Big aggregates vs small aggregates

Aggregates too big:
- Slow saves (whole aggregate must be loaded and saved).
- Lock contention.
- Memory pressure (loading 10,000 order lines for one update).
- Transactions that touch too much.

Aggregates too small:
- Invariants can't be enforced atomically.
- Constant cross-aggregate coordination.
- "Eventual consistency" everywhere even for things that should be local.

The right size is **the smallest cluster that protects the invariants you actually need to enforce together**.

Heuristic: if invariant X must hold immediately after every change, X must be inside an aggregate. If "eventually" is okay, X spans aggregates and lives in a domain process / saga.

---

## 4. Worked Example — E-Commerce

### Bounded contexts (strategic)

| Context | Owns |
| --- | --- |
| **Catalog** | Products, categories, search facets, pricing rules |
| **Cart** | A user's active shopping cart |
| **Orders** | Placed orders, status transitions, invoices |
| **Inventory** | Stock per SKU, reservations, replenishment |
| **Payments** | Payment intents, charges, refunds |
| **Fulfillment** | Pick / pack / ship operations |
| **Identity** | Users, auth, profile |
| **Customer Support** | Tickets, history of interactions |

Each has its own service / module, its own data store, its own language. "Order" in Orders is a state machine; "Order" in Fulfillment is a manifest; they communicate via events (`OrderPlaced`, `OrderShipped`).

### Aggregates within Orders

`Order` (root)
- contains `OrderLine` entities.
- contains `ShippingAddress`, `BillingAddress`, `Money` value objects.
- references `CustomerId`, `CartId`, `PaymentId` by ID only.
- invariants: total = sum(lines), status machine (Placed → Paid → Shipped → Delivered or Cancelled).

`Invoice` (root)
- contains `InvoiceLine` entities.
- references `OrderId`, `CustomerId`.
- invariant: tax total recomputable from lines; once issued, immutable.

`Refund` (root)
- references `OrderId`, `PaymentId`.
- invariants: cannot exceed original charge; one Refund per order generally.

Note: an Order, Invoice, and Refund could feel like one big aggregate. But the **invariants don't span them**. The Refund doesn't need atomic consistency with the Order's status — there's a workflow (saga) where each step transitions one aggregate and emits events.

---

## 5. Storage Implications

Aggregates map cleanly to storage choices:

- **One DB schema per bounded context.** Even in a monolith.
- **One DB table per aggregate root**, with internal entities in child tables (one-to-many).
- **No cross-aggregate foreign keys** (use IDs without DB-level FK constraints, or carefully).
- **No cross-context foreign keys at all.**

A common smell: a foreign key from `Orders.customers_id` to `Customers.id` across two services. The FK ties their schemas, defeats the point of separation. Use the ID, but no constraint at the DB level.

For event sourcing, each aggregate has its own event stream. See [Event Sourcing →](../07-messaging/event-sourcing.md).

---

## 6. Communication Across Aggregates and Contexts

The honest map:

- **Within an aggregate** — direct method calls, single transaction, synchronous, strongly consistent.
- **Between aggregates in the same context** — domain events, eventual consistency, possibly orchestrated.
- **Between bounded contexts** — published events on a broker, ACL on the receiving side, eventual consistency, well-defined published language.

If you find yourself trying to **synchronously update two aggregates** or two contexts in one transaction, you've made a mistake in modeling. Either:

- The aggregates should be one (collapse).
- The "transaction" should be a saga with compensations.
- The "synchronous" call should be an event.

This discipline directly maps to scaling out to microservices. See [Microservices Architecture →](./microservices.md), [Event-Driven Microservices →](./event-driven-microservices.md), [Saga Pattern →](../07-messaging/saga-pattern.md).

---

## 7. Anti-Corruption Layer (Practical)

When two contexts must integrate but their models disagree, the receiving side wraps the upstream in an **anti-corruption layer (ACL)**:

```
Upstream context's wire model (Customer with 50 fields, legacy semantics)
        │
        ▼
┌─────────────────────┐
│  ACL translator     │
│  - filters fields   │
│  - renames concepts │
│  - normalizes units │
└─────────┬───────────┘
          ▼
Local domain model (Customer with 8 fields, your semantics)
```

The ACL keeps the local domain clean. The cost is one more layer; the benefit is you can evolve internally without being held hostage to upstream's quirks. Essential when integrating with legacy or third-party systems.

---

## 8. Discovering the Boundaries

Several techniques exist; **Event Storming** is the most popular:

1. Workshop with experts and engineers.
2. Orange stickies for domain events (`OrderPlaced`, `PaymentCharged`).
3. Blue stickies for commands (`PlaceOrder`, `ChargePayment`).
4. Pink stickies for actors.
5. Walk the timeline; identify clusters of events with shared language.
6. Each cluster is a candidate bounded context.
7. Look for "translation needed" gaps — those are context boundaries.

Other techniques: **Domain Storytelling**, **Context Mapping Workshop**, **DDD Crew's Bounded Context Canvas**.

These are not optional ceremony — they're how teams discover the structure that's actually in the domain. Skipping them and "just designing" produces boundaries that fight the domain.

---

## 9. Common Patterns — and Names for Them

**Customer/Supplier** — the downstream context's roadmap is influenced by the upstream's; the two teams collaborate explicitly on the contract.

**Conformist** — downstream uses upstream's model as-is. Cheap, but accepts upstream's coupling.

**Anti-Corruption Layer** — downstream translates. More work, more isolation.

**Open Host Service / Published Language** — upstream offers a stable public protocol; many downstreams integrate via that.

**Separate Ways** — contexts that don't really need to talk; keep them apart.

**Big Ball of Mud** — no clear boundaries. Often the legacy that needs strangling.

Knowing the labels gives you a vocabulary for talking about the integration choices.

---

## 10. Common Mistakes / Anti-Patterns

- **Shared model across contexts.** "There should be one Customer class for everything." No — each context has its own.
- **Database FKs across contexts.** Couples schemas; defeats independent evolution.
- **Aggregates too big.** Save the whole customer with all orders, all invoices, all reviews — slow, contended, hard to reason about.
- **Aggregates too small** — every entity an aggregate; can't enforce invariants atomically; constant cross-aggregate dance.
- **Multi-aggregate transactions.** Works in a monolith — until it doesn't. Better to model the workflow as a saga from the start.
- **Direct object references between aggregates.** `order.customer.email` instead of `order.customerId`. Breaks aggregate isolation; loads more than necessary.
- **Ignoring invariants.** "Order total = sum of lines" enforced only in the UI. Inevitably violated.
- **Bounded contexts that match technical layers.** "Database context," "API context" — those are not contexts; those are tiers.
- **Bounded contexts as folder names without enforcement.** Boundaries exist on paper; imports cross them freely.
- **No context map.** Teams integrate ad-hoc; coupling grows without anybody seeing.
- **Treating bounded contexts as forever.** They evolve. Refactor.
- **No ACL when consuming legacy.** Legacy concepts contaminate the new model.
- **Skipping Event Storming.** "We'll figure it out as we go" → wrong boundaries → expensive migration.
- **Centralized canonical model.** "Enterprise data model" projects that try to unify across contexts. Doesn't work in practice.
- **Aggregates as containers for unrelated data.** A `User` aggregate that contains everything about a user (preferences, orders, claims, billing) — too big, conflates contexts.

---

## 11. When to Apply

Strategic value (bounded contexts, context maps) is broadly applicable. Any system with more than one team or one domain area benefits.

Tactical value (aggregates, value objects, domain events) pays off most when:

- The domain has invariants worth enforcing in code (not just SQL constraints).
- The team will live with this codebase for years.
- Complex state transitions exist.
- Eventual consistency between sub-systems is acceptable (and required).

For a simple CRUD app with no real invariants, full DDD aggregates are heavy. Domain-shaped naming and clear bounded contexts still help.

---

## 12. Cheat Card

```
BOUNDED CONTEXT  the boundary within which a model + its language is consistent.
                  outside: same word may mean something different.

USE IT TO        cut services / modules / teams.
                  contexts ≈ subdomains ≈ team ownership areas.

CONTEXT MAP RELATIONS
  Shared Kernel · Customer/Supplier · Conformist · Anti-Corruption Layer ·
  Open Host Service · Published Language · Separate Ways · Big Ball of Mud

AGGREGATE  cluster treated as one unit for consistency.
            one root entity · invariants enforced inside ·
            one transaction touches one aggregate ·
            reference other aggregates by ID ·
            keep small as possible while preserving invariants.

ACROSS AGGREGATES / CONTEXTS
  use domain events + eventual consistency + saga
  if you want atomic multi-aggregate transaction, your model is wrong

STORAGE
  one DB schema per context · table-per-aggregate-root + child tables ·
  no cross-aggregate or cross-context FKs

DISCOVERY
  Event Storming · Domain Storytelling · Bounded Context Canvas ·
  Aggregate Design Canvas

WHEN INVARIANT MUST BE IMMEDIATE → same aggregate
WHEN "EVENTUALLY" IS OK         → different aggregates, events between them

ANTI-PATTERNS
  shared model across contexts · cross-context FKs · aggregates too big/small ·
  multi-aggregate transactions · direct object refs between aggregates ·
  centralized canonical model · ignoring invariants ·
  context-as-folder with no enforcement

RULE: bounded contexts cut the system; aggregates cut the transactions.
       Get both right; ride them for years.
```

---

## 13. Resources

### Books
- *Domain-Driven Design* — Eric Evans. Original treatment of bounded contexts and aggregates.
- *Implementing Domain-Driven Design* — Vaughn Vernon. Most practical aggregate guidance.
- *Domain-Driven Design Distilled* — Vaughn Vernon. The short version.
- *Learning Domain-Driven Design* — Vlad Khononov. Modern, pragmatic.
- *Domain Modeling Made Functional* — Scott Wlaschin. Functional flavor.
- *Team Topologies* — Skelton & Pais. Where contexts meet org structure.

### Articles
- "BoundedContext" — Martin Fowler: <https://martinfowler.com/bliki/BoundedContext.html>
- "Effective Aggregate Design" — Vaughn Vernon (3-part essay): <https://www.dddcommunity.org/library/vernon_2011/>
- "Strategic Domain-Driven Design" — DDD Crew: <https://github.com/ddd-crew/bounded-context-canvas>
- "Aggregate Design Canvas" — Nick Tune / DDD Crew.

### Videos
- "Aggregate Design Choices" — Indu Alagarsamy, DDD Europe.
- "Strategic Domain-Driven Design" — Nick Tune, NDC.
- "DDD Europe" conference channel.

### Tools / artifacts
- **EventStorming** boards (Miro / Mural / physical).
- **Bounded Context Canvas** — DDD Crew template.
- **Aggregate Design Canvas** — DDD Crew template.
- **Context Mapper** — DSL + visualization for context maps: <https://contextmapper.org/>

### Adjacent reading
- [Domain-Driven Design →](./ddd.md)
- [Microservices Architecture →](./microservices.md)
- [Event-Driven Microservices →](./event-driven-microservices.md)
- [Clean Architecture / Onion →](./clean-architecture.md)
- [Hexagonal / Ports & Adapters →](./hexagonal.md)
- [Saga Pattern →](../07-messaging/saga-pattern.md)
- [Event Sourcing →](../07-messaging/event-sourcing.md)
- [Strangler Fig Pattern →](./strangler-fig.md)
- [Database Federation →](../04-databases/federation.md)

---

*Previous:* [← Domain-Driven Design (DDD)](./ddd.md)  |  *Up:* [README ↑](../README.md)
