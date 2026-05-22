# Clean Architecture / Onion Architecture

> **TL;DR** — **Clean Architecture** (Robert C. Martin, 2012) and **Onion Architecture** (Jeffrey Palermo, 2008) are siblings of **Hexagonal** with more named layers. All three obey **The Dependency Rule**: source code dependencies always point **inward**, toward higher-level policy. At the center: **Entities** (enterprise-wide business rules). One ring out: **Use Cases** (application-specific rules). Next: **Interface Adapters** (gateways, presenters, controllers). Outermost: **Frameworks & Drivers** (web, DB, devices). Like Hexagonal, the goal is to keep the business logic independent of frameworks, databases, and UIs — so you can change any of those without rewriting the core. The diagrams are more elaborate, but the **one rule** is the whole pattern.

---

## 1. The Idea

```
                   ┌─────────────────────────────────────────────┐
                   │  Frameworks & Drivers                       │
                   │   Web · DB · Devices · External Interfaces  │
                   │  ┌───────────────────────────────────────┐  │
                   │  │  Interface Adapters                   │  │
                   │  │  Controllers · Presenters · Gateways  │  │
                   │  │  ┌─────────────────────────────────┐  │  │
                   │  │  │  Application Business Rules    │  │  │
                   │  │  │  (Use Cases)                   │  │  │
                   │  │  │  ┌─────────────────────────┐   │  │  │
                   │  │  │  │  Enterprise Rules      │   │  │  │
                   │  │  │  │  (Entities)            │   │  │  │
                   │  │  │  └─────────────────────────┘   │  │  │
                   │  │  └─────────────────────────────────┘  │  │
                   │  └───────────────────────────────────────┘  │
                   └─────────────────────────────────────────────┘

                          dependencies point INWARD
```

The **Dependency Rule** (capital-R in Martin's book):

> Source code dependencies must point only inward, toward higher-level policies.

A module in an inner ring **never** mentions a module in an outer ring. An entity doesn't know about a use case; a use case doesn't know about a controller; nothing in the core knows about a framework.

---

## 2. The Four Rings, Concretely

### Entities (innermost — Enterprise Business Rules)
- The most general, reusable rules.
- Apply across many applications in the same business.
- Pure data + behavior; no I/O, no framework.

```python
class Account:
    def deposit(self, amount):
        if amount <= 0: raise ValueError(...)
        self.balance += amount

    def withdraw(self, amount):
        if amount > self.balance: raise OverdraftError(...)
        self.balance -= amount
```

### Use Cases (Application Business Rules)
- Application-specific orchestration.
- One use case per user-visible operation: `TransferFunds`, `PlaceOrder`, `CancelSubscription`.
- Coordinates entities and ports (interfaces).
- Owns transactions.

```python
class TransferFundsUseCase:
    def __init__(self, accounts: AccountRepo, ledger: LedgerPort, clock: Clock):
        self.accounts, self.ledger, self.clock = accounts, ledger, clock

    def execute(self, cmd: TransferFundsCommand) -> TransferResult:
        with self.accounts.tx():
            src = self.accounts.get(cmd.source_id)
            dst = self.accounts.get(cmd.dest_id)
            src.withdraw(cmd.amount)
            dst.deposit(cmd.amount)
            self.accounts.save(src); self.accounts.save(dst)
            self.ledger.record(Transfer(src.id, dst.id, cmd.amount, self.clock.now()))
        return TransferResult(ok=True)
```

### Interface Adapters
- Translate between the use case's data shapes and the outside world.
- **Controllers** — convert HTTP into command objects, call the use case.
- **Presenters** — format the use case's output for the UI.
- **Gateways** — implement secondary ports (DB, queue, external API).

```python
@app.post("/transfers")
def create_transfer(req: TransferRequest):
    cmd = TransferFundsCommand(source_id=req.from_account, dest_id=req.to_account, amount=req.amount)
    result = transfer_uc.execute(cmd)
    return present_transfer(result)
```

### Frameworks & Drivers (outermost)
- Web frameworks (FastAPI, Spring, Rails).
- Databases (Postgres, MongoDB).
- Message brokers, cloud SDKs, GUI frameworks, third-party clients.
- Glue code that wires the system together at `main()`.

These are the **detail**. They should be the **last** thing to influence the design — and the first thing you can swap.

---

## 3. The One Rule, Strictly

```
Allowed:   inner imports nothing from outer
Forbidden: anything in the core importing a framework type
```

| OK | Not OK |
| --- | --- |
| Use case imports Entity | Entity imports Use case |
| Controller imports Use case command DTO | Use case imports HTTP framework |
| Gateway imports `psycopg2` | Use case imports `psycopg2` |
| `main()` imports everything to wire things up | Anywhere else importing everything |

When the inner ring needs something from the outer ring, the inner ring **defines an interface** (a port) that the outer ring **implements**. This is the **Dependency Inversion Principle** in action.

```python
# in use case ring
class AccountRepo(Protocol):
    def get(self, id: AccountID) -> Account: ...
    def save(self, account: Account) -> None: ...

# in interface-adapters ring
class PostgresAccountRepo(AccountRepo):
    def __init__(self, pool): self.pool = pool
    def get(self, id): ...   # SQL here
    def save(self, account): ...
```

The use case depends on the **abstraction** it owns; the Postgres adapter depends inward to satisfy the interface.

---

## 4. Clean vs Hexagonal vs Onion

These three architectures are functionally identical at the level of the rule. The differences are:

| Aspect | Hexagonal | Onion | Clean |
| --- | --- | --- | --- |
| Author | Cockburn, 2005 | Palermo, 2008 | Martin, 2012 |
| Core | Domain + use cases | Domain | Entities + Use Cases |
| Outer terminology | Adapters | Application/Infrastructure rings | Interface Adapters + Frameworks |
| Number of named rings | 2 (in/out) | 3–4 | 4 |
| Famous diagram | Hexagon | Bullseye / onion | Bullseye with squared edges |
| Emphasis | Driving vs driven ports | Domain at the center | Use cases as a distinct ring |

Pick the vocabulary your team has read about. **Don't argue about which is "best."** Argue about whether dependencies actually flow inward.

---

## 5. Boundaries, Data Crossing Them

Each time data crosses a boundary, you may transform it:

```
HTTP request body (DTO)
     │ controller maps
     ▼
Use case Command
     │ use case maps
     ▼
Entity operation
     │ saved via repo
     ▼
Persistence row
```

Reverse for outputs. Each shape exists because each ring has different concerns:

- HTTP DTOs change with the public API contract.
- Use case Commands are stable internal types.
- Entities embody business rules.
- Persistence rows reflect schema.

Pure Clean Architecture mandates these mappings. Pragmatic teams accept some merging — e.g., using the entity directly as the persistence shape when the schema is simple and stable.

---

## 6. Worked Example — Use Case End to End

A subscription-upgrade endpoint:

```python
# entities/subscription.py — pure
class Subscription:
    def upgrade(self, new_plan: Plan, today: date):
        if not self.is_active: raise SubscriptionInactiveError()
        if new_plan.price_cents < self.plan.price_cents:
            raise DowngradeNotAllowedError()
        proration = self.compute_proration(new_plan, today)
        self.plan = new_plan
        self.last_change_at = today
        return proration

# use_cases/upgrade.py — uses entity + ports
@dataclass
class UpgradeSubscriptionCommand:
    subscription_id: SubscriptionID
    new_plan_id: PlanID
    actor_id: UserID

class UpgradeSubscriptionUseCase:
    def __init__(self, subs: SubscriptionRepo, plans: PlanRepo,
                 payments: PaymentGateway, clock: Clock, events: EventBus):
        ...
    def execute(self, cmd):
        sub = self.subs.get(cmd.subscription_id)
        plan = self.plans.get(cmd.new_plan_id)
        with self.subs.tx():
            proration = sub.upgrade(plan, self.clock.today())
            self.payments.charge(sub.account_id, proration)
            self.subs.save(sub)
            self.events.publish(SubscriptionUpgraded(sub.id, plan.id, self.clock.now()))

# adapters/http/controller.py — controller
@app.post("/subscriptions/{sub_id}/upgrade")
def upgrade(sub_id, req: UpgradeRequest, user=Depends(current_user)):
    try:
        upgrade_uc.execute(UpgradeSubscriptionCommand(sub_id, req.plan_id, user.id))
        return 204
    except DowngradeNotAllowedError:
        return JSONResponse(status_code=400, content={"error": "downgrade_not_allowed"})
    except SubscriptionInactiveError:
        return JSONResponse(status_code=409, content={"error": "inactive"})

# adapters/postgres/subscription_repo.py — gateway
class PostgresSubscriptionRepo(SubscriptionRepo):
    def get(self, id): ...
    def save(self, sub): ...
    def tx(self): ...
```

This pattern repeats per use case. The shape is recognizable, the responsibilities are clean, the use case is fully testable without DB or HTTP.

---

## 7. Testing Pyramid in Clean Architecture

Clean codebases tend to grow a clean test pyramid:

```
              ▲
              │ end-to-end (HTTP → DB)         few, slow
         ┌────┴────┐
         │ adapter  │ adapter tests              moderate
         └────┬────┘
              │
         ┌────┴────────┐
         │   use case   │ use case tests with    many, fast
         │   tests      │ in-memory adapters
         └────┬────────┘
              │
       ┌──────┴───────┐
       │  entity unit  │ entity tests           very many, very fast
       │    tests      │
       └──────────────┘
```

- **Entity tests** — purest. Microseconds. Cover business rules deeply.
- **Use case tests** — run with in-memory adapters. Milliseconds. Cover behavior.
- **Adapter tests** — integration against the real Postgres / external API. Slower, scoped per adapter.
- **End-to-end** — a few smoke tests through the whole stack.

A team that adopts Clean usually finds they can run 10,000+ tests in seconds. That's the payoff.

---

## 8. Modules and File Layout

A common Clean layout:

```
src/
  entities/
    account.py
    subscription.py
  use_cases/
    transfer_funds.py
    upgrade_subscription.py
    ports/
      account_repo.py
      subscription_repo.py
      payment_gateway.py
  adapters/
    http/
      controllers.py
      schemas.py
    postgres/
      account_repo.py
      subscription_repo.py
    stripe/
      payment_gateway.py
  main.py
```

Variants exist (group by feature, group by ring, mix). The important thing: enforce in CI that `entities/` and `use_cases/` cannot import from `adapters/`. ArchUnit (Java), dependency-cruiser (Node), import-linter (Python), etc.

For larger apps, layer the structure per **bounded context**:

```
src/
  billing/
    entities/  use_cases/  adapters/
  identity/
    entities/  use_cases/  adapters/
```

Each context has its own onion. See [Bounded Contexts & Aggregates →](./bounded-contexts.md).

---

## 9. When Clean Pays Off

Pays off:
- Domain-heavy systems (financial, healthcare, complex SaaS, ERP).
- Long-lived codebases (10+ year horizon).
- Teams that need to swap or upgrade infrastructure without rewriting business logic.
- Heavy investment in unit testing.
- Systems with multiple delivery channels (HTTP + queue + CLI + batch).

Doesn't pay off:
- Throwaway prototypes.
- CRUD apps where each endpoint is "table in, JSON out."
- One-week MVP.
- Tiny teams who'll only ever ship to one stack.

The honest take: **most enterprise Java/.NET shops** would benefit from a layered+modular monolith with a few Clean touches. **Most Rails/Django CRUD apps** are fine as layered. **Most domain-heavy products** benefit from going Clean from day one.

---

## 10. Common Mistakes / Anti-Patterns

- **Framework annotations on entities.** `@Entity` from JPA / `BaseModel` from SQLAlchemy on a class in the entities ring = no Clean Architecture. Pure POJO/POPO/POGO entities.
- **Use case classes that are 1000 lines.** Split per user-visible operation. One class per use case is the convention.
- **Anemic entities with use case doing everything.** Push behavior onto entities. "Tell, don't ask."
- **Controllers fat with logic.** Move it down into use cases.
- **Pass-through use cases.** `getOrder(id) → repo.get(id)`. If the use case adds nothing, consider exposing the read directly via a separate read model (CQRS).
- **All use cases share a base class with framework dependencies.** Re-introduces what you wanted to avoid.
- **Five layers of DTOs.** Sometimes it's right; often you can reuse the use case Command type as the HTTP body.
- **Ports nobody implements differently.** If `PostgresOrderRepo` is the only implementation ever, the port is just an interface tax. Keep it for tests; don't pretend it's "swappable."
- **Treating Clean as a one-size-fits-all.** A six-method CRUD endpoint shouldn't have a use case + presenter + gateway. Adapt the rigor to the value.
- **Mapping fatigue.** Convert too many times, lose track of intent. Reduce shapes when defensible.
- **Forgetting transactions.** Use case spans multiple repo writes without a transaction → partial failures.
- **Ignoring read models.** Clean's left lobe is great for writes. For reads (lists, search, reports), pure use cases with mapped queries are slow. Use a dedicated read model.
- **"It's Clean" as religion.** Don't argue dogma; argue dependencies.

---

## 11. Cheat Card

```
CLEAN / ONION  inward-only dependencies (THE ONE RULE)

RINGS (Clean)
  Entities                 enterprise rules — pure, framework-free
  Use Cases               application rules — orchestration, transactions
  Interface Adapters      controllers · presenters · gateways
  Frameworks & Drivers    web · DB · cloud SDKs · UI · main()

WHEN INNER NEEDS OUTER    define a PORT in inner, implement in outer
                          (Dependency Inversion)

WHAT INNER MUST NOT DO    import framework types · know about HTTP/SQL
                          · contain `if env == "prod"` ·
                          import other rings' files

TESTS                     entity tests (μs)
                          use case tests w/ in-mem adapters (ms)
                          adapter tests vs real infra (s)
                          handful of E2E

LAYOUT                    by ring, by feature, or by bounded context — all OK
                          ENFORCE in CI (ArchUnit / dep-cruiser / import-linter)

NEIGHBORS                 Hexagonal — same idea, fewer named rings
                          Onion — same idea, sometimes 3 rings

GOOD FIT                  domain-heavy, long-lived, multi-driver, test-friendly
BAD FIT                   pure CRUD, throwaway, tiny team

RULE: if your business code imports anything you can google-search, it's not in the core.
```

---

## 12. Resources

### Books
- *Clean Architecture* — Robert C. Martin. The book that named it.
- *Get Your Hands Dirty on Clean Architecture* — Tom Hombergs. Most useful Java walk-through.
- *Implementing Domain-Driven Design* — Vaughn Vernon. Clean + DDD.
- *Domain Modeling Made Functional* — Scott Wlaschin. Clean in F#.
- *Architecture Patterns with Python* — Percival & Gregory. Clean-ish in Python.

### Articles
- "The Clean Architecture" — Uncle Bob (2012): <https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html>
- "Onion Architecture" — Jeffrey Palermo: <https://jeffreypalermo.com/2008/07/the-onion-architecture-part-1/>
- "Hexagonal vs Clean vs Onion" — Herberto Graça.
- "Cosmos: SOLID python with FastAPI" — Bob Gregory.

### Videos
- "Clean Architecture" — Robert C. Martin keynotes.
- "Architecture: The Lost Years" — Robert C. Martin.
- "DDD + Clean Architecture in Practice" — Tom Hombergs, NDC.

### Tools
- **Enforcement:** ArchUnit (Java/Kotlin), dependency-cruiser (Node/TS), import-linter (Python), deptrac (PHP), go-arch-lint (Go).
- **DI:** Spring, Wire, Dagger, Inversify, NestJS, FastAPI Depends.
- **Mapping:** MapStruct, AutoMapper, ModelMapper, attrs/dataclasses (Python).

### Adjacent reading
- [Layered Architecture →](./layered.md)
- [Hexagonal / Ports & Adapters →](./hexagonal.md)
- [Domain-Driven Design →](./ddd.md)
- [Bounded Contexts & Aggregates →](./bounded-contexts.md)
- [Microservices Architecture →](./microservices.md)
- [CQRS →](../07-messaging/cqrs.md)

---

*Previous:* [← Hexagonal / Ports & Adapters](./hexagonal.md)  |  *Next:* [Microservices Architecture →](./microservices.md)
