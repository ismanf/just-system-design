# Blast Radius & Cell-Based Architecture

> **TL;DR** — **Blast radius** is the set of users, requests, or data affected when something fails. The goal of resilient design is to **shrink the blast radius** — a bug, bad deploy, or hot tenant should affect a small, bounded subset, not "everyone." **Cell-based architecture** is the most rigorous form of blast-radius control: partition the entire stack — compute, database, cache, queue, routing — into independent **cells**, each serving a subset of users. A failure inside cell-3 affects only cell-3's users; cells 1, 2, 4, ..., N keep working. Pioneered at Amazon and now standard at AWS itself (DynamoDB, S3, Route 53 internally), Stripe, Shopify, Slack, and others. The trade is operational complexity (running N copies of everything, schema changes across cells, cross-cell aggregation) for a fundamentally better failure profile. Cells aren't free, but they're the difference between "a bad deploy took down the product" and "a bad deploy took down 5% of customers, and we caught it before rolling further."

---

## 1. Blast Radius — The Core Concept

```
small blast radius ──── large blast radius
                        
single key                                 single tenant
                                                                    
single cell                                single AZ
                                                                    
single region                              global control plane
```

When something fails, *who* is affected? The smaller the answer, the better the architecture.

```
Bug in a deploy                    Outage shape

Whole-fleet deploy        → 100% of users impacted
Per-region deploy         → users in one region impacted (33% in 3-region)
Per-AZ deploy             → users in one AZ impacted (smaller)
Per-cell deploy (10 cells)→ 10% of users impacted
Per-cell + canary cell    → 1% of users impacted; rest unaffected
```

Cell-based architecture is one approach. Others include progressive deploys, feature flags, regional sharding, per-tenant isolation. The common thread: **partition the failure domain**.

---

## 2. Why "Everyone Got Hit" Happens

The default architecture pattern in most cloud applications:

```
                    ┌──────────────┐
                    │   Load LB    │
                    └──────┬───────┘
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
         ┌─────────┐  ┌─────────┐  ┌─────────┐
         │ app-1   │  │ app-2   │  │ app-3   │  ← same code everywhere
         └────┬────┘  └────┬────┘  └────┬────┘
              │            │            │
              └────────────┼────────────┘
                           ▼
                    ┌──────────────┐
                    │   Database   │  ← shared by all
                    └──────────────┘
```

Symptoms of one big blast radius:
- One bad row hits every read path.
- One bad query takes down the only DB.
- One bad deploy lands on all instances.
- One slow tenant starves the others.
- One regional outage takes down everything.
- One bug in a shared dependency cascades.

The architecture is correct *for many small services*. At scale, it's a single fragility — what AWS sometimes calls a "one-control-plane" architecture. Every failure radiates to everyone.

---

## 3. Cell-Based Architecture — The Idea

```
                    ┌────────────────────┐
                    │   Cell Router      │
                    │   (thin, stable)   │
                    └─────────┬──────────┘
                              │
        ┌─────────┬───────────┼───────────┬─────────┐
        ▼         ▼           ▼           ▼         ▼
     CELL 1    CELL 2      CELL 3      CELL 4    CELL 5
     ──────    ──────      ──────      ──────    ──────
     ┌────┐    ┌────┐      ┌────┐      ┌────┐    ┌────┐
     │ LB │    │ LB │      │ LB │      │ LB │    │ LB │
     │app │    │app │      │app │      │app │    │app │
     │ DB │    │ DB │      │ DB │      │ DB │    │ DB │
     │$$$ │    │$$$ │      │$$$ │      │$$$ │    │$$$ │
     │MQ  │    │MQ  │      │MQ  │      │MQ  │    │MQ  │
     └────┘    └────┘      └────┘      └────┘    └────┘
     users     users       users       users     users
     A–F       G–L         M–R         S–X       Y–Z
```

A **cell** is a complete, self-contained stack — compute, database, cache, queue, configuration — serving a subset of users (or tenants, or data). The **cell router** is the thin layer that maps each request to a cell.

Properties:
- **Independent**: a failure in cell-3 doesn't propagate to other cells.
- **Identical**: same code, same architecture, just different instances.
- **Isolated data**: cell-N's data lives in cell-N's database, not in a shared central one.
- **Per-cell deploys**: roll a change to one cell, observe, then to others.
- **Per-cell capacity**: each cell sized for its share of traffic.

The router is the special piece — it must be reliable, but it's intentionally simple. The router's job is *only* to look up "which cell?" and forward. It holds no state, runs no business logic, and is the only thing shared.

---

## 4. The Pattern's Origin

The pattern came from **Amazon's retail platform** in the mid-2000s. Their problem: as the site grew, an outage in any of many backend services could take down checkout. The architecture solution: shard the entire stack per user / per region / per service domain.

AWS later formalized the pattern internally. Services like **DynamoDB, Route 53, S3** are built as fleets of cells under the hood. AWS publishes about this in the [Builders' Library](https://aws.amazon.com/builders-library/) — see "Reliability, constant work, and a good cup of coffee" and "Workload isolation using shuffle-sharding."

The pattern spread:
- **Stripe** — uses cell-style isolation for major customer segments.
- **Shopify** — per-shop (effectively per-tenant) cell architecture inside regional pods.
- **Slack** — Vitess-based per-team isolation that approximates cell architecture for the DB layer.
- **GitHub** — moved toward cell-style architecture for the platform.
- **Figma** — multi-tenant cells for performance and isolation.

The pattern is mainstream in large multi-tenant systems and increasingly the default at scale.

---

## 5. Cell Sizing and Mapping

Two questions: how big should a cell be, and how do you map users to cells?

### Cell sizing

Trade-offs:
- **Smaller cells** → smaller blast radius, more cells to operate.
- **Larger cells** → fewer cells, larger blast radius per failure.

Typical sizing:
- **AWS internal**: cells often serve thousands to millions of customers each.
- **Mid-size SaaS**: cells per ~5–10k tenants.
- **Per-customer cells (for whales)**: enterprise customers can get dedicated cells.

A reasonable starting point: **cells should be small enough that losing one is acceptable for ~30 minutes, large enough that you don't need 1000 of them**.

### Mapping users to cells

Several approaches (similar to [Sharding Strategies →](../10-scalability/sharding-strategies.md)):

**Hash**: `cell = hash(user_id) % N`. Simple, even distribution. Bad if you need to add cells (consistent hashing helps).

**Range**: cells own contiguous ID ranges. Easy to reason about; rebalancing is awkward.

**Directory / lookup**: a routing table maps `user_id → cell_id`. Flexible — assign new tenants to less-loaded cells, isolate whales. Slack's, Stripe's, Shopify's models look like this. The router caches the directory aggressively.

**Region + within-region cell**: top-level by region (EU, US), then cell within. Common for multi-region.

**Random**: less common; useful when "user identity" isn't a natural key.

The directory approach is most flexible and most common at scale. It's worth the extra dependency.

---

## 6. Shuffle Sharding — The Math Magic

A pure cell architecture has one problem: a customer is assigned to one cell. If their cell goes down, *that customer is down*. They might be 100% of one cell's traffic; they're not protected by the rest of the system.

**Shuffle sharding** (AWS Builders' Library) is a clever twist:

```
Standard cell mapping
   user → 1 cell

Shuffle sharding
   user → a virtual cell composed of N cells from the larger pool
        (the user's "shard" is a unique subset of cells, e.g. cells {3, 7, 12})
```

```
Without shuffle sharding:
  User A → Cell 3
  Cell 3 down → User A 100% impacted

With shuffle sharding (2 of 8 cells per user):
  User A → Cells {3, 7}
  User B → Cells {3, 12}
  User C → Cells {7, 12}
  Cell 3 down → users with cell 3 are degraded but can serve from
    their other cell. User A: 50% capacity. User C: unaffected.
```

The math: with N cells and shard size K, the probability that a given pair of users shares the *exact* same shard is `1 / C(N, K)`. With N=8, K=2: `1/28` (~3.6%). With N=20, K=3: `1/1140` (~0.09%). At scale, blast radius for a single-cell failure is reduced to a small fraction of customers experiencing reduced capacity (not full outage).

AWS Route 53 famously uses shuffle sharding to limit blast radius across their fleet.

---

## 7. Per-Cell Deploys

A cell-based architecture is most powerful when deploys are also cellular:

```
Cell 1   ━━━━━━━━━━━━━━━━━━━━━━━━━ deploy v2 first (canary cell)
Cell 2   ━━━━━━━━━━━━━━━━━━━━━━━━━ wait + observe
Cell 3   ━━━━━━━━━━━━━━━━━━━━━━━━━ deploy v2 next
Cell 4–N ━━━━━━━━━━━━━━━━━━━━━━━━━ progressively roll
```

A bad deploy:
- Affects only the rolled-out cells.
- Is caught by SLO alerts on the canary cell before reaching the rest.
- Can be rolled back per-cell while others continue serving.

This is the same idea as canary deploys, but with **architectural isolation** instead of just traffic-splitting. A bad deploy in a canary deployment can still affect the shared DB; a bad deploy in a canary *cell* can't, because the cell has its own DB.

---

## 8. The Cell Router — The Crown Jewel

The router is the one piece shared by all cells. If it fails, the whole system fails. So:

### Router design principles
- **Minimal logic.** Only look up "which cell?" and forward. No business logic.
- **Stateless.** Easy to replicate, easy to fail over.
- **Highly available.** Multi-region, multi-AZ, often shuffle-sharded itself.
- **Heavily cached.** Routing tables change rarely; cache aggressively per-instance.
- **Read-only at runtime.** Routing decisions don't write to anything that's failover-sensitive.
- **Fail-safe.** If the directory service is down, use cached mappings.

Implementations:
- **Edge logic**: a Cloudflare Worker or Lambda@Edge that examines the request and routes.
- **API gateway**: routes based on JWT claim or header.
- **DNS-based**: each cell has its own subdomain (`cell-3.example.com`), client knows its cell.
- **Service mesh**: sidecar routes based on headers.

### What the router knows
- Mapping from "tenant key" (user ID, tenant ID, region) to cell ID.
- Cell endpoints (DNS names, IPs).
- Cell health (skip failed cells, but only if safe to retry on another).

### What the router doesn't know
- Business rules, request validation, anything beyond routing.
- Per-cell internal state.

Stripe's internal router; Shopify's pod router; Slack's cell-aware load balancer — all variations on this theme.

---

## 9. Per-Cell Resources

Each cell has its own:

```
- Application instances              (compute)
- Database primary + replicas        (state)
- Cache cluster                      (state)
- Message queue                      (state)
- Background workers                 (compute)
- Storage buckets                    (state)
- Secrets / config                   (config)
- Observability streams              (signal)
```

The shared pieces (intentionally small):
- Cell router.
- Directory of tenants → cells.
- Global identity provider (or replicated).
- Top-level CDN / DNS.
- Aggregated observability for cross-cell views.

Every other dependency lives **inside the cell**. The cell is the unit of failure isolation.

---

## 10. Cross-Cell Operations — The Hard Part

What if a user in cell-3 needs to interact with a user in cell-7? The hardest part of cell architecture.

Options:

### Federation / explicit cross-cell APIs
Cell-3 calls cell-7 for the data. Cross-cell calls are slower and reduce isolation; minimize them.

### Read replication
Replicate certain global data (catalog, public profiles, company data) to all cells. Each cell reads locally; updates async-replicated.

### Per-tenant interactions only
Design the product so a single user's data stays in their cell. Cross-tenant operations are rare and explicit (sending an invoice across companies, sharing a doc).

### Cross-cell event streams
Kafka or similar fans out cell-level events to a central topic; cells consume what they need.

### Routing on shared keys
If a logical operation involves cells A and B, route based on a stable key (e.g., the originating user) and let the destination cell pull what it needs.

In practice: **design the product so that most operations stay within one cell**. The hard ones are explicit cross-cell flows, designed deliberately, instrumented heavily, and accepted as occasional sources of correlated failure.

---

## 11. Schema and Migration Across Cells

A migration that touches every cell is hard. Patterns:

### Per-cell rolling migration
Run the migration on cell-1; verify; cell-2; ...; cell-N. Schema changes ship cell-by-cell. Long-running; safer.

### Forward-compatible code
Deploy code that reads old + new schema simultaneously; then migrate data per cell; then deploy code that reads only new schema. Classic two-step migration, multiplied by N cells. See [Migrations at Scale →](../04-databases/migrations.md).

### Tooling
Cell-aware migration runners that respect cell ordering, track per-cell state, retry per-cell on failure, abort per-cell on issues.

### Cross-cell consistency
Across cells, no transaction. Across cells, eventual consistency or sagas.

The operational overhead is real. Cell architecture imposes the discipline that **every change is a rolling deploy across cells** — which is a feature for blast radius and a burden for velocity.

---

## 12. Observability Across Cells

Without cell-aware observability, a cell-based system is a nightmare to operate. Required telemetry:

- **Cell-labeled metrics**: every metric tagged with `cell_id`.
- **Cell-labeled traces**: spans carry the cell identifier.
- **Per-cell dashboards**: see cell-3's health independent of others.
- **Cross-cell aggregation**: total system health, but with the ability to drill into per-cell views.
- **Alerts by cell**: a single bad cell shouldn't drown out other alerts.
- **Cell distribution metrics**: traffic / load per cell — visible at a glance to spot imbalance.

The investment in observability scales with the number of cells. Many teams underestimate this and learn the hard way.

---

## 13. Cost and Operational Burden

The honest discussion:

### Costs
- **More resources**: every cell needs its own DB, cache, etc. Cells don't share resources (that's the point), so capacity per-cell is provisioned independently.
- **Idle capacity**: each cell needs N+1 capacity, multiplied across cells.
- **Operational tooling**: deploy, monitor, debug N stacks instead of one.
- **Cross-cell complexity**: code paths for federation, routing tables, distributed query/aggregation.

### Benefits
- **Smaller blast radius**: bad deploys, bad data, hot tenants contained.
- **Cleaner failover**: a region full of cells can fail to another region without cross-cell coupling.
- **Compliance / tenancy guarantees**: dedicated cells for sensitive customers.
- **Capacity planning per cell**: easier to size and reason about.
- **Cellular load testing**: pick one cell; pound it; learn its limits without affecting prod.

Cell architecture is most justified when:
- Multi-tenant systems with varying tenant sizes.
- High availability targets (>99.95%).
- Regulatory or contractual isolation requirements.
- Operations team has the scale to operate N cells.

It's overkill for early-stage products, internal tools, or single-tenant systems.

---

## 14. Worked Example — A Cell-Based SaaS

A B2B SaaS at 10M users, 50k tenants:

```
TOP LEVEL: REGION
  US-EAST  (40% of users)
  EU-WEST  (35%)
  AP-SE-1  (25%)

WITHIN EACH REGION: CELLS
  Cell-S (small):  ~200 cells, each serving ~50 tenants
  Cell-M (medium): ~50 cells, each serving ~5 tenants
  Cell-L (large):  ~10 cells, each serving 1 large enterprise tenant

ROUTER
  Cloudflare Worker → tenant_id from JWT → directory lookup →
  resolve to cell endpoint → forward.

PER CELL
  - Kubernetes namespace
  - Postgres primary + 2 replicas (Aurora cluster per cell)
  - Redis (3-node)
  - Kafka topic prefix per cell
  - S3 bucket per cell
  - Observability tagged with cell_id

OPERATIONS
  - Deploys: roll cell-by-cell, starting with canary cells
  - Schema migrations: per-cell, tracked
  - Disaster recovery: per-cell backups + cross-region snapshots

CROSS-CELL
  - Catalog (product list, etc.): replicated to all cells (read-only)
  - User identity: central IdP
  - Cross-tenant operations: rare; explicit; logged

OBSERVABILITY
  - Every metric / trace / log carries cell_id
  - Per-cell SLO dashboards
  - Aggregate "fleet health" view
  - Alert routing: cell-specific to cell-specific on-call
```

The shape: a thin router + many independent cells. A failure in cell-S-47 affects 50 tenants for 10 minutes; everyone else doesn't notice.

This is roughly what Stripe, Shopify, and Slack look like at the platform level.

---

## 15. When NOT to Use Cell Architecture

Cell architecture is heavy. Skip it when:

- **Small scale**: <1M users, single tenant, single region. Not enough blast radius to control.
- **Limited operations capacity**: running 1 stack well is hard enough. Don't multiply.
- **Workload not naturally partitionable**: if every operation crosses many tenants, cells are friction with no benefit.
- **Early-stage product**: optimizing for blast radius before product-market fit is premature.
- **Truly single-tenant systems**: one customer; one stack; cells are over-engineering.

A better progression for most teams:
1. Single-AZ deploy.
2. Multi-AZ.
3. Multi-region.
4. Regional sharding for compliance.
5. Cell architecture for blast radius.

Most companies stop at step 3 or 4 forever and never need cells. The ones at step 5 have specific reliability or scale targets that justify the cost.

---

## 16. Real-World Examples

### AWS DynamoDB
Built as fleets of cells, each cell handling a partition of the global keyspace. A single-cell failure affects only the partitions hosted in that cell.

### AWS Route 53
Famously shuffle-sharded across many cells. Provided extreme isolation: failures touch tiny fractions of customers.

### AWS S3
Built on cellular architecture; thousands of internal "fleets" each handling subsets of buckets.

### Stripe
Customer-facing API + payment processing split across cells; large enterprise customers can be assigned dedicated cells.

### Shopify
Per-shop sharding inside region "pods." Each shop is in exactly one pod; pods are cells.

### Slack
Vitess-based sharding by team, effectively cellular at the DB layer. Failure of one team's keyspace is contained.

### Figma
Multi-tenant cells, with hot tenants moved to dedicated cells.

### GitHub
Moved toward cellular architecture for repository hosting and CI runners.

### Cloudflare
Architecture isolates customers across cells; certain large customers get dedicated cells.

---

## 17. Common Mistakes / Anti-Patterns

- **Cells with shared state.** A "cell architecture" where every cell talks to the same Postgres primary isn't cellular at all.
- **Heavyweight router.** A router that does business logic, validation, or authn is itself a blast-radius problem.
- **No per-cell observability.** Can't see which cell is failing; can't isolate.
- **Cross-cell operations baked into critical paths.** Every cross-cell call recouples the cells.
- **Same deploy to all cells at once.** Defeats the canary-cell benefit.
- **No directory of tenant → cell.** Hard-coded mappings; can't rebalance.
- **Cells too small.** 1000 cells of 10 tenants each. Operational nightmare.
- **Cells too large.** 3 cells of 33% traffic each. Blast radius isn't meaningfully smaller than no cells.
- **Schema migrations not cell-aware.** Half-migrated state across cells; can't roll forward or back.
- **No shuffle sharding for routing.** Single-cell-failure = subset 100% impacted.
- **Premature cellular architecture.** Building 50 cells at 10k users.
- **Cross-cell aggregation in the hot path.** "Show me all users" query touches every cell.
- **Cells without independent capacity planning.** One cell over-loaded, others idle.
- **Single global cache or queue.** Cells share that; one bad cell breaks them all.

---

## 18. Decision Rule

```
Do you need to limit blast radius beyond what regions / AZs provide?
  NO  → multi-AZ + multi-region + per-tenant rate limits + canary
        deploys is probably enough.
  YES → cell architecture might fit.

Is your workload naturally partitionable (per tenant, per region,
per data subset)?
  NO  → cells will be friction. Reconsider.
  YES → continue.

Do you have the operational maturity to run N stacks?
  NO  → build it up first (observability, runbooks, tooling).
  YES → continue.

Is your scale large enough that cell count makes sense?
  NO  → wait. Premature cells = pain.
  YES → continue.

If all yes:
  - Start with regional sharding (cells = regions).
  - Add cells within a region as growth and isolation needs justify.
  - Build cell-aware tooling: deploys, migrations, observability.
  - Add shuffle sharding for routing if single-cell-failure isolation
    isn't enough.
  - Hard rule: keep the cell router minimal and stateless.
  - Hard rule: cross-cell operations are explicit, instrumented,
    expected to occasionally fail.

Always:
  - Per-cell deploys with canary cells.
  - Per-cell observability.
  - Per-cell SLOs.
  - Tested per-cell failover.
```

---

## 19. Cheat Card

```
PURPOSE     Shrink blast radius. Convert "everyone is down" into
            "this 5% of users is down for 10 minutes."

CELL        Self-contained stack (compute + DB + cache + queue + config)
            serving a subset of users. Identical across cells.

ROUTER      Thin, stateless, highly-available layer that maps
            requests → cells. Holds no business logic.

PROPERTIES  Independent · identical · isolated data · per-cell deploys ·
            per-cell capacity · cross-cell ops minimized

SHUFFLE SHARDING
            User assigned to K cells out of N (instead of 1).
            Single-cell failure: user has reduced capacity, not full
            outage. Math gives strong isolation across users.

PER-CELL    Each cell owns:
            - app instances · DB primary + replicas · cache · queue
            - workers · storage buckets · secrets · observability

SHARED      - Cell router · directory · IdP · CDN · top-level DNS
              (kept intentionally tiny)

DEPLOYS     Roll cell-by-cell. Canary cell first. Bad deploy affects
            one cell, not the fleet.

CROSS-CELL  Federation · read replication · per-tenant boundaries ·
            event fan-out. Minimize; cross-cell calls reduce isolation.

OBSERVABILITY
            cell_id on every metric/trace/log. Per-cell dashboards.
            Per-cell SLOs. Cell health visible at a glance.

WHEN YES    Multi-tenant SaaS · high availability targets ·
            compliance / isolation needs · operationally mature team
WHEN NO     Small scale · single-tenant · non-partitionable workload ·
            premature optimization · limited ops capacity

PITFALLS    Shared state · heavyweight router · no per-cell view ·
            cross-cell ops on hot paths · same-deploy-everywhere ·
            cells too small / too large · no shuffle sharding ·
            single global cache or queue

RULE        Multi-AZ first. Multi-region next. Cells when blast radius
            still matters more than complexity. The cell router stays
            small; everything else moves into the cell.
```

---

## 20. Resources

### Books
- *The Site Reliability Workbook* — Google. Chapters on reducing blast radius.
- *Designing Data-Intensive Applications* — Martin Kleppmann.
- *Patterns of Distributed Systems* — Unmesh Joshi.

### Articles
- "Workload Isolation Using Shuffle-Sharding" — AWS Builders' Library: <https://aws.amazon.com/builders-library/workload-isolation-using-shuffle-sharding/>
- "Reliability, Constant Work, and a Good Cup of Coffee" — Marc Brooker, AWS.
- "Static Stability Using Availability Zones" — AWS Builders' Library.
- "Cells: A Pattern for Resilient Systems" — AWS.
- "How Stripe Designs Cells" — Stripe engineering blog posts.
- "Shopify's Pods" — Shopify engineering.
- "Slack's Cellularization" — Slack engineering blog.
- "Figma's Move to Cells" — Figma engineering.
- "Cell-Based Architecture" — Hector Garcia-Molina, summarized in various reliability writeups.

### Videos
- AWS re:Invent — "Reliability Patterns" tracks (recurring). Look for sessions by Marc Brooker.
- "Cells: An Architecture for High Availability" — various Amazon talks.
- "How Shopify Survives Black Friday" — Shopify engineering talks.
- SREcon — talks on blast radius and cell architecture.
- ByteByteGo — "Cell-Based Architecture" overview.

### Tools
- **Kubernetes** — namespace-based cell isolation patterns.
- **AWS** — multi-account / multi-region deployment for cell separation.
- **Vitess** — keyspace sharding approximates cells for MySQL.
- **Citus** — schema-per-tenant or distributed Postgres as cellular DB.
- **Istio / Linkerd** — cell-aware routing via service mesh.
- **Spinnaker / Argo CD** — per-cell deployment pipelines.

### Adjacent reading
- [Fault Tolerance Patterns](./fault-tolerance.md)
- [Failover & Disaster Recovery](./failover-dr.md)
- [Graceful Degradation](./graceful-degradation.md)
- [Chaos Engineering](./chaos-engineering.md)
- [Multi-Region](../10-scalability/multi-region.md)
- [Database Sharding Strategies](../10-scalability/sharding-strategies.md)
- [Hot Partition Problem](../10-scalability/hot-partitions.md)
- [Multi-Tenancy](../15-deployment/multi-tenancy.md)
- [Bulkhead Pattern](./bulkhead.md)

---

*Previous:* [← Chaos Engineering](./chaos-engineering.md)  |  *Next:* [Idempotent Operations & Retries →](./idempotency-retries.md)
