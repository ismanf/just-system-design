# Hexagonal / Ports & Adapters

> **TL;DR** — **Hexagonal architecture** (Alistair Cockburn, 2005; aka **Ports & Adapters**) inverts the dependency rule of Layered architecture: the **domain sits in the center**, and everything external — HTTP, databases, message brokers, external APIs — connects to it through **ports** (interfaces defined by the domain) implemented by **adapters** (infrastructure code). The point isn't the hexagon shape; it's *who depends on whom*. Done right, you can swap Postgres for SQLite in tests, replace REST with gRPC without touching domain logic, and run the domain in-process with no I/O. Done wrong, it's an extra layer of indirection that buys nothing. Hexagonal pays off for domain-heavy applications, long-lived codebases, and teams that value testability over short-term throughput.

---

## 1. The Idea

In Layered architecture, persistence is at the bottom and the domain depends on it. Hexagonal flips this: the **domain depends on nothing**; everything depends on the domain.

```
                    ┌────────────────────────────────┐
                    │                                │
   HTTP request ───►│  HTTP Adapter (driver)         │──┐
                    │                                │  │
                    └────────────────────────────────┘  │
                                                        │ calls
                                                        ▼
                                          ┌────────────────────────────┐
                                          │                            │
                                          │       DOMAIN (core)        │
                                          │   - entities               │
                                          │   - business rules         │
                                          │   - use cases              │
                                          │   - defines port interfaces│
                                          │                            │
                                          └─────────────┬──────────────┘
                                                        │ defines
                                                        ▼
                    ┌────────────────────────────────┐  │
                    │  DB Adapter (driven)           │◄─┘ implements
                    │  - Postgres / SQLite / Mock    │
                    │                                │
                    └────────────────────────────────┘
```

Two kinds of ports:

- **Driving (primary) ports** — how the world calls the domain. Used by HTTP handlers, CLI, message consumers. The domain exposes a use-case interface; adapters drive it.
- **Driven (secondary) ports** — what the domain needs from the world. Database, email, payment provider, clock, random number generator. The domain defines the interface; adapters supply implementations.

The hexagon shape (six sides) is just artistic — it could be any polygon. The point is: many adapters, one domain, **interfaces owned by the domain**.

---

## 2. Why Invert the Dependency

In layered, the domain calls persistence. So the domain knows about the DB, ORM, connection pool. To unit-test the domain, you need a DB (or a heavy mock).

In hexagonal, the domain calls a `UserRepository` interface it owns. The Postgres adapter implements that interface. To unit-test the domain, you pass an in-memory implementation.

Benefits:

- **Swap infrastructure freely.** Postgres → DynamoDB, REST → gRPC, no change to the domain.
- **Test domain in isolation.** Run thousands of tests per second with in-memory fakes.
- **Multiple "drivers."** Same domain, exposed over HTTP, gRPC, CLI, queue, scheduled jobs — each is just an adapter.
- **Boundary explicit.** The domain knows what it needs (`AccountRepository`, `EmailSender`); infrastructure is forced to fit.

The cost: more interfaces and indirection. For tiny CRUD apps, the layered approach is simpler. For systems with real business rules, the inversion pays dividends.

---

## 3. Ports, Adapters, and the Domain

### The Domain (the hexagon's interior)
- Entities, value objects, domain services, use cases.
- No imports from infrastructure: no `import org.springframework`, no `import psycopg2`.
- Defines interfaces for everything it needs from the outside.

### Driving (primary) ports
- "How am I used?" — interfaces the application offers.
- Example: `CreateOrderUseCase.execute(CreateOrderCommand): Order`.
- Adapters: HTTP controllers, CLI, message consumers, scheduled jobs.

### Driven (secondary) ports
- "What do I need?" — interfaces the application requires.
- Example: `OrderRepository`, `EmailSender`, `PaymentGateway`, `Clock`.
- Adapters: Postgres repo, SES sender, Stripe client, system clock.

### Adapters
- Glue between the world and the domain.
- Driving adapters call the domain's use cases.
- Driven adapters implement the domain's interfaces.
- All vendor-specific code lives here.

---

## 4. A Worked Example

A small payments service in Go-flavored pseudocode:

```go
// domain/order.go — no imports from net/http, sql, kafka, etc.
package domain

type Order struct {
    ID       OrderID
    UserID   UserID
    Lines    []OrderLine
    Status   OrderStatus
}

func (o *Order) Cancel(reason string) error {
    if o.Status == Shipped { return ErrCannotCancel }
    o.Status = Cancelled
    return nil
}

// domain/ports.go — interfaces the domain needs
type OrderRepository interface {
    FindByID(ctx Context, id OrderID) (*Order, error)
    Save(ctx Context, order *Order) error
}
type EmailSender interface {
    Send(to, subject, body string) error
}
type Clock interface {
    Now() time.Time
}

// domain/usecases.go — driving port
type CancelOrderUseCase struct {
    orders OrderRepository
    email  EmailSender
    clock  Clock
}
func (uc *CancelOrderUseCase) Execute(ctx Context, cmd CancelOrderCommand) error {
    order, err := uc.orders.FindByID(ctx, cmd.OrderID)
    if err != nil { return err }
    if err := order.Cancel(cmd.Reason); err != nil { return err }
    if err := uc.orders.Save(ctx, order); err != nil { return err }
    return uc.email.Send(order.userEmail(), "Order cancelled", ...)
}

// adapters/http/order_handler.go — driving adapter
func (h *Handler) Cancel(w ResponseWriter, r *Request) {
    cmd := parseCancelOrderRequest(r)
    if err := h.cancelUC.Execute(r.Context(), cmd); err != nil {
        respond(w, mapError(err))
        return
    }
    respond(w, 204, nil)
}

// adapters/postgres/order_repo.go — driven adapter
type PostgresOrderRepo struct { db *sql.DB }
func (r *PostgresOrderRepo) FindByID(ctx Context, id OrderID) (*Order, error) {
    // SELECT ... — vendor-specific
}
func (r *PostgresOrderRepo) Save(ctx Context, o *Order) error {
    // INSERT/UPDATE ... — vendor-specific
}

// main.go — wire it up
db := openPostgres(cfg)
ses := openSES(cfg)
orders := &PostgresOrderRepo{db}
email := &SESEmailSender{ses}
clock := &SystemClock{}

cancelUC := &CancelOrderUseCase{orders, email, clock}
handler := &Handler{cancelUC}
http.ListenAndServe(":8080", handler)
```

The domain has zero infrastructure imports. Swap `PostgresOrderRepo` for `InMemoryOrderRepo` in tests, swap `SESEmailSender` for a fake — all done.

---

## 5. Testing in a Hexagonal World

This is where it shines.

```go
func TestCancelOrder_AlreadyShipped(t *testing.T) {
    orders := &InMemoryOrderRepo{}
    email  := &FakeEmailSender{}
    orders.Save(ctx, &Order{ID: "ord_1", Status: Shipped})

    uc := &CancelOrderUseCase{orders, email, &FixedClock{}}
    err := uc.Execute(ctx, CancelOrderCommand{OrderID: "ord_1"})

    require.ErrorIs(t, err, ErrCannotCancel)
    require.Empty(t, email.Sent)
}
```

No DB, no HTTP, no real clock. The test runs in milliseconds and tests **what the system does**, not what the database does. Hundreds of these can run in seconds.

Many teams adopt hexagonal mostly for this reason — testability transforms how fast they can iterate.

---

## 6. Hexagonal in Different Languages

Languages with strong interfaces (Java, Kotlin, C#, Go, TypeScript, Rust, Scala) make hexagonal natural. Dynamic languages (Python, Ruby) can do it via protocols / duck typing.

| Language | Ports = | Adapters = |
| --- | --- | --- |
| Java / Kotlin | Interfaces | Classes implementing them, injected via Spring |
| Go | Interfaces (satisfied implicitly) | Structs with methods |
| TypeScript | Interfaces | Classes / objects |
| Python | `Protocol` (PEP 544) or abstract base classes | Classes |
| Ruby | Duck-typed contracts | Modules / classes |
| Rust | Traits | Structs implementing them |

The **dependency injection (DI)** mechanism (manual, container, framework) doesn't matter — passing the adapter into the use case is the whole "DI" you need.

---

## 7. Hexagonal vs Clean vs Onion

These three are close cousins. Differences are mostly nomenclature:

| | Hexagonal (Cockburn, 2005) | Onion (Palermo, 2008) | Clean (Martin, 2012) |
| --- | --- | --- | --- |
| Core | Domain + use cases | Domain at center | Entities + use cases at center |
| Outer rings | Adapters | Application services, infra | Interface adapters, frameworks/drivers |
| Direction | Inward dependency | Inward dependency | Inward dependency (one rule) |
| Layers | Two (inside / outside) | Several rings | Four rings |

The **inward dependency rule** is identical in all three. Hexagonal is the most minimal — just ports and adapters. Clean adds named layers (Entities, Use Cases, Adapters, Frameworks). Onion is somewhere in between.

In practice: pick the names from whichever book your team has read. The architecture is the same. See [Clean Architecture →](./clean-architecture.md).

---

## 8. Pitfalls and Real-World Limits

### Over-abstracting trivial cases
For a CRUD endpoint that returns a row, an entire ports/adapters scaffolding is overkill. Hexagonal pays off when the domain has real behavior. For thin CRUD apps, layered is simpler.

### Anemic domain trap
Hexagonal doesn't automatically produce a rich domain. You can have hexagonal-shaped folders with logic still living in services. The pattern doesn't substitute for domain modeling. See [DDD →](./ddd.md).

### ORM coupling sneaking in
Letting JPA `@Entity` annotations into the domain layer — even if the file is named `domain/Order.java`, you've coupled to Hibernate. Solutions:
- **Plain domain entities** + separate JPA entities + a mapper.
- Or accept the coupling (pragmatic, especially in small teams).
- Or use a less-invasive persistence approach (Spring Data JDBC, Doobie, Diesel, raw SQL).

### Mapping fatigue
Hexagonal often produces multiple representations of the same concept: HTTP DTO, application command, domain entity, persistence row. Mapping between them is real work. Tools (MapStruct, AutoMapper) help; sometimes accepting one-step less mapping is the right call.

### Hexagonal cargo-culting
Folders named `ports/`, `adapters/`, `domain/` with no actual inversion — adapters knowing about other adapters, domain importing `pg`, etc. The pattern is meaningless without enforcement.

### "Every external call is a port"
A logger, a metrics client, a random UUID generator — sometimes these are worth abstracting as ports, sometimes they're just side effects you let into the domain because abstracting them buys little. Pragmatism.

---

## 9. Hexagonal in Microservices

Each service is a hexagon. Drivers and driven ports become the service's API surfaces:

- Driving: HTTP, gRPC, Kafka consumer.
- Driven: DB, downstream HTTP, Kafka producer.

This makes each service consistent, internally testable, and decoupled from infra choices. Many companies that adopt microservices also adopt hexagonal per-service — the patterns compose well.

The microservices boundary is **also** a "port" in the larger sense: contracts between services. See [Microservices →](./microservices.md).

---

## 10. When to Use Hexagonal

Strong fit when:
- The domain has real business rules (not just CRUD).
- You expect the codebase to live 5+ years.
- You want fast, infra-free unit tests.
- You may need to support multiple drivers (HTTP + queue + CLI) for the same domain.
- You may swap a DB / vendor / framework someday.

Weak fit when:
- Pure CRUD app with a few endpoints.
- Throwaway / prototype.
- Tight deadline, small team, mainstream stack — layered is faster to write.
- The domain has almost no behavior — moving rows between tables.

The right answer is usually "start layered, refactor toward hexagonal as the domain gets heavier."

---

## 11. Common Mistakes / Anti-Patterns

- **Domain importing the framework.** `@Entity` from JPA in the domain class. Couples you to a specific ORM.
- **Adapters knowing each other.** HTTP adapter calling DB adapter directly. They should both go through the domain.
- **No actual inversion.** Folders named after the pattern but dependencies flow the wrong way.
- **Anemic core.** Hexagonal scaffolding with all logic in services. The "domain" is data classes.
- **Mappers everywhere.** Five layers of objects per request; mapping is half the code. Simplify.
- **One port per dependency, always.** Sometimes a direct call to `time.Now()` is fine. Don't abstract everything.
- **Pure-domain dogma.** Refusing to use a library that would simplify code because "the domain must be pure." Pragmatism.
- **Ignoring transactions.** Use cases that span multiple repo calls without explicit transaction boundaries. Decide where transactions live (usually around the use case).
- **Using hexagonal as the only architecture rule.** Tells you what depends on what; doesn't tell you what entities and aggregates look like (that's DDD).
- **Confusing test-doubles with adapters.** Tests use in-memory adapters; that's fine. But test doubles aren't a substitute for real adapters in production.
- **Pursuing infrastructure-swap that never happens.** Most teams never swap their DB. But the *test* benefit alone is usually worth it.
- **Replacing layered with hexagonal in a giant rewrite.** Refactor incrementally.

---

## 12. Cheat Card

```
HEXAGONAL = Ports & Adapters
RULE       Domain depends on NOTHING.   Everything depends on the domain.

PORTS (interfaces, OWNED BY domain)
  driving (primary)   how the world calls the domain
                      e.g., CreateOrderUseCase
  driven  (secondary) what the domain needs
                      e.g., OrderRepository, EmailSender, Clock

ADAPTERS (implementations, OWNED BY infrastructure)
  driving adapters    HTTP / gRPC / CLI / consumers
  driven adapters     Postgres repo / SES sender / Stripe client

WIRE UP    in main(): construct concrete adapters, inject into use cases

TESTS      use in-memory adapters; run the domain in ms, no DB, no HTTP

GOOD FIT
  domain-heavy systems · long-lived codebases · multiple drivers ·
  testability matters · vendor independence matters

WEAK FIT
  pure CRUD · throwaway · tight schedule · trivial domain

VARIANTS   Clean Architecture (Martin) · Onion (Palermo) — same inversion,
           different names + rings

ANTI-PATTERNS
  framework annotations in domain · adapters calling adapters ·
  anemic core · mappers galore · port-everything fetish

RULE: if your domain imports a framework, you don't have hexagonal yet.
```

---

## 13. Resources

### Books
- *Hexagonal Architecture Explained* — Alistair Cockburn, Juan Manuel Garrido de Paz.
- *Get Your Hands Dirty on Clean Architecture* — Tom Hombergs. Excellent hexagonal walk-through in Java.
- *Clean Architecture* — Robert C. Martin.
- *Implementing Domain-Driven Design* — Vaughn Vernon. Hexagonal + DDD.
- *Domain Modeling Made Functional* — Scott Wlaschin. Hexagonal in F#.

### Articles
- "Hexagonal Architecture" — Alistair Cockburn (original): <https://alistair.cockburn.us/hexagonal-architecture/>
- "Ports and Adapters Architecture" — Herberto Graça.
- "The Clean Architecture" — Uncle Bob.
- "Hexagonal Architecture in Three Examples" — Tom Hombergs.

### Videos
- "Ports and Adapters / Hexagonal" — Alistair Cockburn talks.
- "Clean Architecture in Practice" — various NDC / GOTO talks.
- "Hexagonal Go" — Go conference talks.

### Tools
- **DI containers:** Spring, Dagger, Wire, Inversify, NestJS DI.
- **Boundary enforcement:** ArchUnit (Java), dependency-cruiser (Node), import-linter (Python), go-arch-lint (Go).
- **Mapping:** MapStruct, AutoMapper, ModelMapper.

### Adjacent reading
- [Layered Architecture →](./layered.md)
- [Clean Architecture / Onion →](./clean-architecture.md)
- [Domain-Driven Design →](./ddd.md)
- [Bounded Contexts & Aggregates →](./bounded-contexts.md)
- [Microservices Architecture →](./microservices.md)

---

*Previous:* [← Layered Architecture](./layered.md)  |  *Next:* [Clean Architecture / Onion →](./clean-architecture.md)
