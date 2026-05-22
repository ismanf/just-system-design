# Domain-Driven Design (DDD)

> **TL;DR** — **Domain-Driven Design** (Eric Evans, 2003) is an approach to building complex software where the **domain model is the primary artifact**: the code structure mirrors the way domain experts talk and think about the business. DDD is two layers: **strategic** (bounded contexts, context maps, ubiquitous language) for organizing the system into coherent islands; **tactical** (entities, value objects, aggregates, domain services, repositories, domain events) for modeling within an island. The strategic patterns are the more valuable half — they tell you **where to draw boundaries**, which is the hardest question in microservices, modular monoliths, and any system bigger than one team. DDD pays off when the **domain is complex**; for CRUD apps it's overkill. It is not a process or a framework — it's a way to design software whose structure tracks the business it serves.

---

## 1. The Premise

In most software, the hard part isn't the technology; it's the **domain** — the business rules, edge cases, special cases that experts spent decades figuring out. If your code structure doesn't match how the business actually works, you fight your own architecture forever.

DDD's central claim:

> The model and design shape each other. The model — the way the team thinks about the domain — is the most important deliverable, and it must be reflected directly in the code.

Concretely:

- Domain experts and developers share a **ubiquitous language**.
- The code uses that language verbatim. `Order.cancel()`, not `OrderManager.processStateTransition(STATE_CANCELLED)`.
- The model is refined continuously as understanding deepens.
- The system is decomposed into **bounded contexts** where the language is consistent.

When done well, DDD produces codebases that read like the business. When done badly, it's a vocabulary fad layered onto a layered architecture. The difference is whether the model actually drives the design.

---

## 2. Strategic Design (the more important half)

Strategic DDD is about **organizing the system**:

- **Ubiquitous language** — shared vocabulary across experts and code.
- **Bounded contexts** — the boundaries within which a model and its language are consistent.
- **Context maps** — how bounded contexts relate to each other.
- **Subdomains** — core, supporting, generic — to focus effort where it matters.

This is the half that informs service boundaries, team structure, and architectural decisions. It maps directly onto microservices, modular monoliths, and team topology.

See [Bounded Contexts & Aggregates →](./bounded-contexts.md) for the deep dive on bounded contexts.

### Ubiquitous language

A single, project-wide language for talking about the domain — used by analysts, product, designers, and engineers, **and reflected in code**.

If domain experts say "policy," "claim," "subrogation," your code uses those words. Not "record," "entity," "row." Not "do_subro_stuff()" but `claim.subrogate()`.

When the same word means different things to different teams (`Order` in Sales vs `Order` in Fulfillment), that's a sign of separate **bounded contexts** — each with its own language.

### Subdomain types

Evans separates subdomains by strategic importance:

| Type | Description | Investment |
| --- | --- | --- |
| **Core domain** | The differentiator — why customers pick you | Highest investment, best engineers, deepest design |
| **Supporting subdomain** | Necessary but not differentiating | Moderate; in-house if no off-the-shelf fits |
| **Generic subdomain** | Solved problem (auth, billing, search) | Buy / use OSS; do not build |

A frequent failure: spending senior engineers on generic problems (yet another auth system) while the core domain stays under-modeled. Identify your core domain and invest accordingly.

### Context maps

How bounded contexts relate at the boundaries:

- **Shared kernel** — two contexts share a small subset of model (rare, fragile, only by tight collaboration).
- **Customer / supplier** — downstream context's needs influence upstream's design; explicit collaboration.
- **Conformist** — downstream simply uses upstream's model as is.
- **Anti-corruption layer (ACL)** — downstream translates upstream's model into its own to avoid contamination.
- **Open Host Service / Published Language** — upstream offers a stable public model.
- **Separate ways** — no integration; cheaper to keep apart.
- **Big Ball of Mud** — no clear model; the worst end-state, often a legacy.

Drawing the context map for an enterprise is one of the most useful DDD outputs — it exposes which teams must collaborate vs which can move independently.

---

## 3. Tactical Design

Tactical DDD is about how to model **within** a bounded context. The building blocks:

### Entities
Objects with **identity** that persists over time. Two `User`s with the same email are distinct if their IDs differ. Identity matters more than attributes.

```python
class Order:
    def __init__(self, id: OrderId, ...): self.id = id; ...
    def __eq__(self, other): return self.id == other.id
```

### Value objects
Defined by their attributes. Two `Money(USD, 4200)` are equal because their values are equal. **Immutable** by convention.

```python
@dataclass(frozen=True)
class Money:
    currency: Currency
    cents: int
    def add(self, other): return Money(self.currency, self.cents + other.cents)
```

Value objects are underrated. Replacing primitive obsession (`int amount, str currency, date charged_at`) with value objects (`Money`, `BillingDate`) makes invalid states unrepresentable.

### Aggregates
A cluster of objects treated as a unit for consistency. One **aggregate root** (an entity) is the only entry point; outsiders reference only the root by ID.

```
Order (aggregate root)
  ├── OrderLine (entity within aggregate)
  ├── ShippingAddress (value object)
  └── Total (value object)
```

Rules:
- All changes go through the root.
- Invariants (e.g., total = sum of lines, status transitions) are enforced inside the aggregate.
- Aggregates are the transactional unit — one transaction per aggregate.
- Reference other aggregates by ID only, not by direct object reference.

See [Bounded Contexts & Aggregates →](./bounded-contexts.md).

### Domain services
Logic that doesn't belong to any single entity. Naming domain services as **verbs** is common — `TransferFundsService`, `PriceCalculator`.

```python
class FundsTransferService:
    def transfer(src: Account, dst: Account, amount: Money) -> Transfer:
        src.withdraw(amount); dst.deposit(amount)
        return Transfer(src.id, dst.id, amount, today())
```

Caution: too many domain services and too few entity methods produces the **anemic domain model** anti-pattern. Push behavior onto entities first; create domain services only when behavior is genuinely between aggregates.

### Repositories
Abstract collection-like interface for persisting and loading aggregates. The repository belongs to the domain layer; the implementation lives in infrastructure.

```python
class OrderRepository(Protocol):
    def get(self, id: OrderId) -> Order: ...
    def save(self, order: Order) -> None: ...
```

Repositories return **aggregates**, not arbitrary rows. The persistence shape is a detail.

### Factories
Encapsulate complex aggregate creation. When constructors get parameter-heavy or invariants must be enforced at birth, use a factory.

```python
class OrderFactory:
    def create_from_cart(self, cart: Cart, user: User) -> Order: ...
```

### Domain events
Things that happened in the domain, expressed in past tense: `OrderPlaced`, `PaymentRefunded`, `SubscriptionUpgraded`.

```python
@dataclass
class OrderPlaced:
    order_id: OrderId
    user_id: UserId
    placed_at: datetime
```

Aggregates record domain events as side effects of their operations:

```python
def place(self, ...):
    self.status = Placed
    self.events.append(OrderPlaced(self.id, self.user_id, now()))
```

These domain events power eventual consistency between aggregates and between bounded contexts. They are the input to event-driven microservices. See [Event-Driven Microservices →](./event-driven-microservices.md), [Event Sourcing →](../07-messaging/event-sourcing.md).

### Modules / packages
Logical groupings of related domain objects. In code, they become folders/packages with the same name as concepts in the language.

---

## 4. The Process — Not Linear

DDD isn't a checklist; it's a continuous activity:

```
   Talk to experts ──►  Build ubiquitous language
        │                          │
        ▼                          ▼
   Find bounded contexts  ◄──►  Refine the model
        │                          │
        ▼                          ▼
   Reflect in code  ◄──────►  Discover new insights
                    iterate
```

**Event Storming** (Alberto Brandolini) is the most common modern technique. Workshop with experts + engineers; whiteboard; orange stickies for domain events ("OrderPlaced"); blue for commands ("PlaceOrder"); pink for actors; yellow for read models; purple for policies. Walk the timeline. Discover bounded contexts where the language shifts.

The output is far more valuable than UML. Teams that haven't done it underestimate how much it surfaces.

---

## 5. DDD and Architecture

DDD is **agnostic to architecture style**. It pairs with:

| Pairing | How |
| --- | --- |
| **Layered / Clean / Hexagonal** | Tactical DDD sits in the domain layer; infrastructure adapts |
| **Microservices** | Bounded contexts → services. Aggregates → consistency boundaries |
| **Modular monolith** | Bounded contexts → modules; aggregates → internal boundaries |
| **Event-driven** | Domain events → events on the wire |
| **CQRS / Event Sourcing** | Aggregates emit events; read models project them |

The two most common combinations in real codebases:
- **DDD + Clean Architecture** (Java/.NET/Python world).
- **DDD + Modular Monolith** (Rails/Django/Go world, with strong module boundaries).

See [Clean Architecture →](./clean-architecture.md), [Microservices →](./microservices.md).

---

## 6. A Worked Example

A simplified insurance domain:

### Strategic
- Subdomains: **Policy** (core — pricing and rules are differentiator), **Claims** (core), **Billing** (supporting), **Auth** (generic — buy/OIDC), **Notifications** (generic — SES).
- Bounded contexts: Policy, Claims, Billing, Customer.
- Context map: Claims and Policy are tightly related; Claims subscribes to `PolicyIssued` events. Billing is downstream of both via published events. Customer is the master record; both reference customer by ID via an Open Host Service.

### Tactical (inside the Claims context)
- Aggregate root: `Claim` with `ClaimLine` entities and `Money` value objects.
- Domain services: `ClaimSettlementService` that interacts with reinsurance.
- Repositories: `ClaimRepository`.
- Factories: `ClaimFactory.from_fnol(fnol)` builds a Claim from a First Notice of Loss.
- Domain events: `ClaimReported`, `ClaimApproved`, `ClaimPaid`.

### Code sketch
```python
class Claim:                                # aggregate root
    def __init__(self, id: ClaimId, policy_id: PolicyId, ...):
        self.id = id; self.policy_id = policy_id
        self.lines: list[ClaimLine] = []
        self.status = ClaimStatus.Reported
        self._events: list = []

    def approve(self, adjuster: AdjusterId, today: date):
        if self.status != ClaimStatus.Reviewed:
            raise InvalidTransition()
        self.status = ClaimStatus.Approved
        self._events.append(ClaimApproved(self.id, adjuster, today))

    def pay(self, payment: Payment):
        if self.status != ClaimStatus.Approved:
            raise InvalidTransition()
        self.status = ClaimStatus.Paid
        self._events.append(ClaimPaid(self.id, payment.amount, payment.at))
```

The vocabulary (`Claim`, `Adjuster`, `approve`, `pay`, `Paid`) is the business's vocabulary. Reviewers, lawyers, and engineers read this and agree.

---

## 7. When DDD Pays Off

Strong fit:
- The **domain is complex** — many rules, many edge cases, regulatory considerations.
- The product will live and evolve for years.
- Domain experts are available and engaged.
- The team can invest time in modeling, not just shipping.
- Multiple teams need coherent boundaries.

Weak fit:
- Mostly CRUD with shallow business logic.
- Throwaway / MVP.
- Tiny team, single domain area, no need for boundaries.
- Generic problems (build an admin panel).

The honest framing: **strategic DDD (bounded contexts, ubiquitous language) is broadly useful**; **tactical DDD is overkill** unless the domain genuinely has depth.

Many teams adopt DDD vocabulary without modeling work, end up with extra files and the same anemic CRUD, and blame the methodology. The work is in the modeling, not the file structure.

---

## 8. DDD Anti-Patterns

- **DDD vocabulary, anemic model.** Folders named `domain/`, but entities are bags of getters/setters. Logic in services. Pattern fail.
- **Aggregates too large.** "An entire customer" with policies, claims, billing, addresses all in one aggregate. Slow saves, contention, transactional headaches. Aggregates are usually small.
- **Aggregates too small.** Every entity its own aggregate. Loses invariant enforcement; constant cross-aggregate references.
- **Shared model across contexts.** One `Customer` class used by Sales, Billing, and Support — each needing different fields. Stretches and breaks.
- **Tactical DDD without strategic.** Aggregates inside the wrong contexts. Boundaries shift, refactor cost compounds.
- **Strategic DDD as decoration.** "We have bounded contexts" — but they share databases, share teams, and don't enforce anything.
- **Big upfront modeling.** Spend months drawing context maps; ship nothing. DDD is iterative.
- **Domain primitives ignored.** `int dollars, str currency` everywhere instead of a `Money` value object. Invalid combinations.
- **CRUD in disguise.** "We have a domain layer" that's literally `repo.save(model)` everywhere.
- **DDD as religion.** Refusing to use a simpler pattern because "that's not DDD."
- **Domain layer that imports ORM/framework.** Then it's not the domain layer.
- **Forgetting to keep the language updated.** "The model's name doesn't match what we call it anymore." Rename — DDD is continuous.

---

## 9. Practical Adoption — A Pragmatic Path

If you're considering DDD, here's a low-risk roll-in:

1. **Run an Event Storming workshop.** Get domain experts and engineers in a room (or virtual board). Map the domain in events. Identify candidate bounded contexts. This is often the highest-value DDD activity teams ever do.
2. **Pick one bounded context.** The most complex one with the most rules.
3. **Adopt the ubiquitous language** — rename code to match how experts talk. Surprisingly hard; surprisingly clarifying.
4. **Refactor that context** to have proper aggregates, value objects, domain events.
5. **Publish domain events** across context boundaries — the start of event-driven integration.
6. **Iterate.** Add other contexts as the team grows or the domain warrants.

Don't try to "do DDD" as a whole-codebase rewrite. The benefit comes from rigor in the high-value places.

---

## 10. Modern Variants and Influences

DDD has shaped many newer ideas:

- **Bounded contexts → microservices.** Direct lineage. Sam Newman cites DDD as the right tool for service boundaries.
- **Event Sourcing + CQRS** — Greg Young's work, deeply DDD-flavored.
- **Domain modeling with algebraic data types** — Scott Wlaschin's *Domain Modeling Made Functional* (F#).
- **Sociotechnical architecture** — Team Topologies (Skelton & Pais) builds on DDD's strategic insights.
- **Aggregates as consistency boundaries** — modern distributed-systems thinking borrows directly.

---

## 11. Common Mistakes / Anti-Patterns

(See §8 — main ones are anemic models, badly sized aggregates, shared models across contexts, tactical-without-strategic, decoration without enforcement, and CRUD in disguise.)

Additional:

- **Importing DDD blindly into greenfield CRUD.** Pattern fatigue, no payoff.
- **Skipping the conversation.** DDD with no domain experts in the room is just architecture cosplay.
- **Treating bounded contexts as fixed forever.** They evolve as understanding deepens.
- **No code-language alignment.** Talking about `Quote` in meetings, naming the class `OfferDraft` in code.
- **Letting a single team own all contexts.** Conway's law eventually forces realignment.

---

## 12. Cheat Card

```
DDD = code structure mirrors the business — driven by a shared language.

STRATEGIC (the high-value half)
  Ubiquitous language — same words in experts and code
  Bounded contexts   — model + language are consistent within a boundary
  Context map        — how contexts relate (shared kernel, ACL, conformist, ...)
  Subdomains         — core (invest) · supporting · generic (buy)

TACTICAL (within a context)
  Entity         identity over time (Order, Claim, User)
  Value object   identity = its fields, immutable (Money, Address)
  Aggregate      cluster with one root; transactional unit; ref others by ID
  Domain service logic between aggregates (TransferFunds, PricingService)
  Repository     abstract persistence, returns aggregates
  Factory        encapsulates complex creation
  Domain event   past-tense fact (OrderPlaced)
  Module         logical groupings named in the language

PROCESS
  iterative · talk to experts · Event Storming · code reflects insights ·
  refactor language continuously

PAIRS WELL WITH
  Clean/Hexagonal · microservices · modular monolith · event-driven · CQRS / ES

GOOD FIT     complex domains · long-lived systems · multiple teams
WEAK FIT     CRUD · MVP · tiny team · generic problems

ANTI-PATTERNS
  anemic model · DDD vocabulary without modeling · aggregates too big/small ·
  shared model across contexts · tactical without strategic ·
  big upfront modeling · primitive obsession · domain imports framework

RULE: DDD pays off when the model is doing real work.
       Strategic first; tactical where rigor earns its keep.
```

---

## 13. Resources

### Books
- *Domain-Driven Design* — Eric Evans. The original ("the Blue Book"). Heavy but foundational.
- *Implementing Domain-Driven Design* — Vaughn Vernon ("the Red Book"). More practical, pairs well with Evans.
- *Domain-Driven Design Distilled* — Vaughn Vernon. The short version.
- *Patterns, Principles, and Practices of Domain-Driven Design* — Scott Millett, Nick Tune.
- *Domain Modeling Made Functional* — Scott Wlaschin. F# / functional flavor; concise and excellent.
- *Learning Domain-Driven Design* — Vlad Khononov. Modern, pragmatic intro.
- *Team Topologies* — Skelton & Pais. Sociotechnical, builds on DDD.

### Articles
- "DDD Crew" — collaborative DDD resources: <https://ddd-crew.github.io/>
- "Bounded Context Canvas" — DDD Crew.
- "Aggregate Design Canvas" — Nick Tune / DDD Crew.
- "Anemic Domain Model" — Martin Fowler.
- "DDD, Hexagonal, Onion, Clean, CQRS: How I put it all together" — Herberto Graça.

### Videos
- "DDD Europe" conference talks — many free on YouTube.
- "Event Storming" — Alberto Brandolini.
- "Domain Driven Design: The Good Parts" — Jimmy Bogard (NDC).
- "Domain Modeling Made Functional" — Scott Wlaschin (NDC).

### Tools / artifacts
- **Event Storming** — physical or Miro / Mural / EventStorming.com.
- **Bounded Context Canvas** — template.
- **Aggregate Design Canvas** — template.
- **Context Mapper** — visualization tool for context maps.
- **AsyncAPI / OpenAPI** — formalize contracts at context boundaries.

### Adjacent reading
- [Bounded Contexts & Aggregates →](./bounded-contexts.md)
- [Microservices Architecture →](./microservices.md)
- [Clean Architecture / Onion →](./clean-architecture.md)
- [Hexagonal / Ports & Adapters →](./hexagonal.md)
- [Event-Driven Microservices →](./event-driven-microservices.md)
- [Event Sourcing →](../07-messaging/event-sourcing.md)
- [CQRS →](../07-messaging/cqrs.md)
- [Strangler Fig Pattern →](./strangler-fig.md)

---

*Previous:* [← Lambda vs Kappa Architecture](./lambda-kappa.md)  |  *Next:* [Bounded Contexts & Aggregates →](./bounded-contexts.md)
