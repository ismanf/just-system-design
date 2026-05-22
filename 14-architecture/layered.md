# Layered Architecture

> **TL;DR** — **Layered architecture** (a.k.a. **n-tier**) organizes a system into horizontal layers — *Presentation → Application/Service → Domain → Persistence* — where each layer only talks to the layer directly below it. It's the default architecture for most enterprise applications and remains the right starting point for monoliths. Its great strength is **simplicity and obviousness**: any new engineer can read the codebase by walking the layers. Its great weakness is that the **domain depends on the database**, which makes substitution, testing, and long-term flexibility harder — exactly what Hexagonal and Clean Architectures try to fix. Used pragmatically, with strict downward-only dependencies and one clean DB-access layer, it scales surprisingly far. Used dogmatically (every method in every layer for every entity), it produces a flood of pass-through code.

---

## 1. The Idea

A layered system stacks responsibilities. A request enters the top, descends through layers, returns. Each layer has one job and a contract for what's above and below it.

```
┌─────────────────────────────────────┐
│   Presentation layer                │   HTTP handlers, gRPC controllers,
│   (controllers / API / UI)          │   request/response shapes
└────────────────┬────────────────────┘
                 │ DTOs
                 ▼
┌─────────────────────────────────────┐
│   Application / Service layer       │   use cases, orchestration,
│                                     │   transactions, authorization
└────────────────┬────────────────────┘
                 │ commands / queries
                 ▼
┌─────────────────────────────────────┐
│   Domain layer                      │   entities, value objects,
│   (business logic)                  │   business rules
└────────────────┬────────────────────┘
                 │ repository interfaces
                 ▼
┌─────────────────────────────────────┐
│   Persistence / Data layer          │   DB access, ORM, SQL, caches,
│                                     │   external API clients
└─────────────────────────────────────┘
```

Each layer **calls down**, never up. Each layer can **skip a layer down** only when there's a good reason (most teams forbid it).

This shape is the bones of nearly every Spring, Rails, Django, ASP.NET, and Express enterprise app ever shipped.

---

## 2. Why It Won

Layered architecture became dominant because:

- **It's obvious.** A new engineer can find anything in minutes.
- **It maps to team structure.** Backend devs work in service/domain, DBAs in persistence, frontend in presentation.
- **Frameworks bake it in.** Spring's `@Controller / @Service / @Repository`. Rails' `app/controllers / app/models / app/services`. ASP.NET MVC.
- **It separates concerns at coarse grain.** Enough to make the codebase navigable.
- **Testing is straightforward for the top layers** — mock the layer below.

For greenfield monoliths, it's the right default. The mistakes start when teams pursue architectural purity beyond what helps.

---

## 3. The Layers in Detail

### Presentation
- HTTP/gRPC controllers, GraphQL resolvers, CLI handlers, message-queue consumers.
- **Owns the protocol.** Parses requests, validates surface-level shape, formats responses.
- **Does not own business rules.** A controller that decides whether a user can edit an invoice belongs in the service layer.
- Should be thin (≤50 lines per handler). If it grows, push logic down.

### Application / Service
- The verbs of the system: `createOrder`, `cancelSubscription`, `processPayment`.
- Coordinates multiple domain operations.
- Owns **transactions** — start one, complete or rollback.
- Owns **authorization** above what middleware handles.
- Translates between presentation DTOs and domain objects.
- This is where "use cases" live in Clean Architecture terms.

### Domain
- Pure business logic — what makes your system unique.
- Entities (`Order`, `Invoice`, `User`), value objects (`Money`, `Address`), domain services.
- Should not import anything from web frameworks, ORMs, or external clients.
- **Hardest to get right, most valuable when you do.**

### Persistence
- Reads and writes to the DB.
- ORM models, query builders, raw SQL.
- Cache reads.
- External HTTP API clients (sometimes pulled into a separate Integration layer).
- The layer most coupled to vendor decisions.

---

## 4. Open vs Closed Layers

A subtle distinction:

- **Closed layer:** requests must pass through it. The presentation layer cannot reach the domain directly — it goes through application.
- **Open layer:** can be bypassed when convenient.

| Approach | Pros | Cons |
| --- | --- | --- |
| **All closed** | Pure, predictable | Lots of pass-through code |
| **Open with rules** | Less code, still navigable | Discipline-dependent |

Pragmatic recommendation: closed by default, with **one** authorized shortcut — e.g., read-only queries skipping the service layer for a simple list endpoint. Document the exceptions; don't let them spread.

---

## 5. Dependencies Always Flow Down

The defining rule: **lower layers don't import higher layers**.

| OK | Not OK |
| --- | --- |
| Controller imports Service | Service imports Controller |
| Service imports Domain | Domain imports Service |
| Service imports Repository interface | Domain imports DB session object |

Violations creep in subtly:
- "Just use the User entity in the controller" → domain leaks upward (fine in monoliths, dangerous when API contracts diverge from domain).
- "Just inject the HTTP request into the service" → presentation leaks down.
- "Just use the ORM model in the controller" → persistence leaks all the way up.

Static checks (ArchUnit for Java, dependency-cruiser for Node, import-linter for Python) can enforce these rules in CI.

---

## 6. A Walkthrough — One Request

A request to `POST /orders` in a layered Spring-style app:

```
Browser sends JSON
  ↓
@RestController OrderController.placeOrder(req)
  - validates required fields
  - maps to CreateOrderCommand
  ↓
@Service OrderService.placeOrder(cmd)
  - @Transactional begins
  - authorize: cmd.user can purchase
  - call domain: Order.create(cmd) → Order entity
  - call PricingService.price(order)
  - call InventoryService.reserve(order)
  - call PaymentService.charge(order)
  - call orderRepository.save(order)
  - publish OrderPlaced event
  - @Transactional commits
  ↓
@Repository OrderRepository.save(order)
  - INSERT into orders / order_lines via JPA
  ↓
Database
```

Each layer does one thing. The handler is short. The service orchestrates. The domain enforces invariants. The repository persists.

---

## 7. What Layered Doesn't Solve

Layered architecture answers: *what code goes where?* It doesn't answer:

- **Which database?** That's an infrastructure choice that gets baked in early.
- **What's the team boundary?** Layers are horizontal; teams usually want vertical slices (a feature owns its end-to-end stack).
- **How does the domain stay testable when persistence is an ORM with annotated entities?** Often it doesn't — the domain becomes ORM-shaped.
- **What if one feature has different latency/availability needs than another?** Layered handles all features uniformly.

These are the gaps that **Hexagonal** ([Hexagonal / Ports & Adapters →](./hexagonal.md)) and **Clean Architecture** ([Clean Architecture →](./clean-architecture.md)) directly address — by inverting the dependency: persistence depends on domain, not the other way around.

---

## 8. Vertical Slices — A Common Refinement

A pure horizontal layout means a single feature (`Orders`) is spread across `controllers/order/`, `services/order/`, `repositories/order/`. Many teams refactor to **vertical slices** while keeping the layer concepts:

```
src/
  orders/
    controller.rb       (or controller.go)
    service.rb
    domain.rb
    repository.rb
  invoices/
    controller.rb
    ...
```

Same layered model, but co-located by feature. Teams own folders, not horizontal strata. This is increasingly the dominant flavor in modern Rails / Django / Go codebases.

Beyond co-location, Jimmy Bogard's **Vertical Slice Architecture** for .NET goes further — each feature is its own self-contained module with no shared service layer.

---

## 9. Monolithic Layered ≠ Bad

The pendulum swung hard to microservices in the 2010s. Many teams now find that a **well-layered monolith** is the right answer for years longer than they thought. Examples:

- **Shopify** — famously a Rails monolith with a layered+modular structure ("modular monolith"). Powers ~10% of online commerce.
- **GitHub** — Rails monolith with strong layering and service extraction at edges.
- **Basecamp** — Rails monolith since forever.
- **Stack Overflow** — historically a layered .NET monolith.

The pattern: start layered + monolithic; extract services only when team or scale demands force it. Premature microservices buys distributed-systems complexity for no benefit.

---

## 10. Anti-Patterns Specific to Layered

### Anemic domain model
Entities are bags of getters/setters with no behavior. All logic lives in services. The "domain layer" becomes a folder of data classes.

Sign: an `Order` class has `getStatus()` and `setStatus()` but no `cancel()`, `markPaid()`, `refund()` methods.

Fix: put behavior on entities. `order.cancel()` rather than `OrderService.cancel(order)`.

### Pass-through methods
Every layer has a method that just calls the layer below:

```java
class OrderController { Order get(id) { return orderService.get(id); } }
class OrderService    { Order get(id) { return orderRepo.get(id); } }
class OrderRepository { Order get(id) { return em.find(Order.class, id); } }
```

Three calls to do one DB load. If the layers add no value, collapse them. Layers should justify their existence.

### Lasagna code
So many layers, sub-layers, and helper layers that following a request takes 20 file jumps. Especially common in big enterprise Java codebases circa 2008.

### Persistence-driven domain
The domain mirrors the DB schema. Foreign key in DB → field in entity. Adding a feature means migration first, domain second. The domain has lost its independence; you're really just CRUD-ing tables.

Fix: Let the domain be shaped by the **language of the business**, not by SQL. See [Domain-Driven Design →](./ddd.md).

### Layered as a religion
Refusing to write a 30-line `presentation → DB` simple read because "it doesn't go through the service layer." Pragmatism wins.

---

## 11. When Layered Is Wrong

Move beyond pure layered when:
- Multiple **teams** need independent ownership and deploys → service decomposition.
- The **domain** has multiple distinct, contradicting models → bounded contexts, possibly DDD-style modular monolith.
- **Event-driven** flows dominate (high-volume async) — layers help less than event topology.
- You need **hexagonal-style swappability** (mocking DB for tests, supporting multiple DBs).
- Latency/scale needs differ wildly between features.

Layered is the simplest pattern that works for most of an organization, most of the time. The patterns in later sections of this chapter are responses to specific limits of pure layered.

---

## 12. Common Mistakes / Anti-Patterns

- **Anemic domain.** Logic in services, dumb data in entities. Push behavior onto entities.
- **Pass-through layers.** If a layer doesn't add value, remove it.
- **Domain importing ORM.** Couples domain to a vendor; can't test or swap.
- **Controllers with business logic.** Move it down; controllers stay thin.
- **Service holding HTTP request objects.** Push presentation concerns out of services.
- **Repository returning ORM-managed entities all the way to controllers.** Detach or map; otherwise lazy-loading bugs and N+1 queries appear far from the source.
- **Skipping layers ad hoc.** Inconsistent code paths; some go through service, others don't.
- **One giant service class.** `OrderService` with 200 methods. Split by use case.
- **No transaction boundary.** Each repo call is its own transaction; multi-step writes go inconsistent on partial failure.
- **DTOs that are just renamed entities.** No mapping value; just noise. Either map meaningfully or expose entities at the seam.
- **Generic CRUD repositories with no domain meaning.** `genericRepo.findBy(...)` everywhere. Lose intent; add custom finders.
- **Architecture-by-folder without enforcement.** Imports cross layers; nothing checks it. Add ArchUnit / linter rules.

---

## 13. Cheat Card

```
LAYERED        Presentation → Application/Service → Domain → Persistence
RULE           dependencies always flow DOWN
CLOSED         requests must pass through each layer (default)

LAYER JOBS
  Presentation     protocol, request/response, thin (~50 lines/handler)
  Application      use cases, transactions, authorization, orchestration
  Domain           entities, value objects, business rules (no framework!)
  Persistence      DB access, external clients, cache

VARIANTS
  Vertical slices  organize by feature, keep layer concepts
  Modular monolith vertical slices + module boundaries enforced
  Hexagonal/Clean  next step when domain must be independent of infra

GOOD WHEN
  monolith · CRUD-heavy · small/medium team · standard frameworks

DON'T
  anemic domains · pass-through layers · ORM-shaped domain
  · business logic in controllers · transactions per-statement

WHEN TO LEAVE
  team boundaries demand independent deploys → microservices
  domain needs swappable infra / heavy testing → hexagonal/clean
  high-volume async dominates → event-driven

RULE: closed layers + thin presentation + rich domain + transactional service.
       Refactor toward verticals when teams grow.
```

---

## 14. Resources

### Books
- *Patterns of Enterprise Application Architecture* — Martin Fowler. The reference for layered designs.
- *Building Evolutionary Architectures* — Ford, Parsons, Kua.
- *Software Architecture: The Hard Parts* — Ford, Richards, Sadalage, Dehghani.
- *Domain-Driven Design* — Eric Evans. The original.
- *Clean Architecture* — Robert C. Martin.

### Articles
- "PresentationDomainDataLayering" — Martin Fowler: <https://martinfowler.com/bliki/PresentationDomainDataLayering.html>
- "Vertical Slice Architecture" — Jimmy Bogard.
- "The Modular Monolith" — Simon Brown, Kamil Grzybek.
- "Anemic Domain Model" — Martin Fowler.

### Videos
- "Modular Monoliths" — Simon Brown talks.
- "Vertical Slice Architecture" — Jimmy Bogard, NDC.

### Tools
- **ArchUnit** (Java), **dependency-cruiser** (Node), **import-linter** (Python), **deptrac** (PHP), **go-arch-lint** — enforce layer boundaries in CI.
- Frameworks that bake in layered: Spring, Rails, Django, ASP.NET, Phoenix.

### Adjacent reading
- [Hexagonal / Ports & Adapters →](./hexagonal.md)
- [Clean Architecture / Onion →](./clean-architecture.md)
- [Microservices Architecture →](./microservices.md)
- [Domain-Driven Design →](./ddd.md)
- [Bounded Contexts & Aggregates →](./bounded-contexts.md)
- [Monolith vs Microservices vs Serverless →](../01-foundations/monolith-microservices-serverless.md)

---

*Up:* [README ↑](../README.md)  |  *Next:* [Hexagonal / Ports & Adapters →](./hexagonal.md)
