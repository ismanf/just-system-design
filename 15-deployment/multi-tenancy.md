# Multi-Tenancy

> **TL;DR** — A **multi-tenant** system serves multiple customer organizations (tenants) from a shared pool of code and infrastructure, while making each tenant's data, traffic, and behavior appear isolated. The opposite is **single-tenant** (every customer gets their own deployment). The spectrum between them — shared DB / shared schema, shared DB / per-tenant schema, DB-per-tenant, cluster-per-tenant — is the single biggest architectural decision for any B2B SaaS. The trade-off: **shared infrastructure is cheap and easy to operate; isolation is what enterprise customers, regulators, and noisy-neighbor incidents demand**. Most successful SaaS starts shared and adds isolation tiers for high-value or regulated tenants over time. The four hard problems are always the same: **data isolation, performance isolation, security isolation, and operational complexity** — and every level of the stack (network, compute, database, cache, queue, observability) has to answer for each.

---

## 1. The big picture

```
Single-tenant                      Multi-tenant
─────────────                      ────────────

[ Tenant A ] [ Tenant B ] [ ... ]   [ Tenant A ┐
   own DB      own DB                Tenant B  ├─► shared app
   own app     own app               Tenant C  │   shared DB
   own infra   own infra             Tenant D ─┘   shared infra
```

Single-tenant: every customer has their own stack. Maximum isolation, minimum efficiency. Common in regulated industries, government, white-label deployments.

Multi-tenant: every customer shares the same stack, separated only by data scoping. Maximum efficiency, minimum isolation. Common in SaaS: GitHub, Slack, Notion, Linear, Stripe (heavy multi-tenant + sharding).

Reality: most successful SaaS sits somewhere between. The bulk of tenants run on shared infrastructure; **enterprise tier** or **regulated tier** tenants get their own database, cluster, region, or VPC.

---

## 2. Why this matters more than people expect

Multi-tenancy decisions touch:

- **Cost.** Shared infra is 10–100× cheaper per tenant.
- **Performance.** Noisy-neighbor effects ruin p99 if isolation is weak.
- **Security & compliance.** Cross-tenant data leak is the #1 SaaS catastrophe. SOC 2, ISO 27001, HIPAA, GDPR all care about isolation.
- **Operations.** Schema migrations, backups, restores, capacity planning — all complicated by tenants.
- **Sales motion.** Enterprise wants "your data isn't on the same DB as our competitor's." You either have an answer or you lose the deal.
- **Pricing.** Per-seat? Per-tenant? Usage-based? Multi-tenancy shape constrains pricing.
- **Onboarding speed.** Multi-tenant: seconds. Per-tenant infra: minutes to hours.

The earlier you make these decisions explicit, the cheaper they are to live with. Retrofitting tenant isolation onto a fully shared system is one of the harder migrations in software. Conversely, over-isolating early (a cluster per tenant before you have ten customers) burns operational time you don't have.

---

## 3. The four isolation dimensions

| Dimension | Question | Mechanisms |
|---|---|---|
| **Data** | Can tenant A read/modify tenant B's data? | Row scoping, schema, separate DB, separate cluster |
| **Performance** | Can tenant A's load impact tenant B's latency? | Quotas, rate limits, dedicated compute, cell architecture |
| **Security** | Can a tenant escape the abstraction? | RBAC, network policies, encryption, VPC isolation |
| **Operational** | Can we operate, observe, and bill per tenant? | Tenant-aware metrics, logs, deploys, runbooks |

Every multi-tenancy decision is really a choice along these four axes. You can be highly isolated on data and barely isolated on performance, or vice versa. Pick the right combination per tier.

---

## 4. The data-isolation spectrum

This is the choice everyone agonizes over first.

### 4.1 Shared database, shared schema, `tenant_id` column

Every table has a `tenant_id`. Every query filters on it.

```sql
SELECT * FROM orders WHERE tenant_id = $1 AND id = $2;
```

**Pros**: cheapest, simplest operationally, easy schema migrations (one DDL).
**Cons**: weakest isolation. One missed WHERE clause = cross-tenant data leak. Backup/restore of a single tenant is non-trivial. Hot tenants can dominate the DB. Noisy-neighbor risk is high.

Mitigations:
- **Row-Level Security (RLS)** in Postgres — the database enforces tenant scoping regardless of application bugs. Set `app.tenant_id` per connection, define policies on every table.
- **ORM hooks** — automatic tenant scoping in the data layer.
- **Code review discipline** — flag any query missing `tenant_id`.
- **Per-tenant query budgets** at the application or proxy layer.

Used by: many early-stage SaaS, Notion, Linear, Vercel KV, anything where tenants are small and similar.

### 4.2 Shared database, schema-per-tenant

Each tenant gets its own Postgres schema (or MySQL database). Same physical DB.

**Pros**: better logical isolation, easier per-tenant backups, can apply schema migrations per tenant (and pause for big tenants).
**Cons**: schema explosion (thousands of schemas can hurt Postgres performance), connection pool harder to share, schema migration is now N operations.

Used by: some Postgres-heavy SaaS that need stronger logical isolation.

### 4.3 Database-per-tenant

Each tenant gets a dedicated database server (or PG/MySQL instance). Often one physical host serves multiple small tenants; large tenants get dedicated hardware.

**Pros**: strong data isolation, per-tenant tuning, per-tenant restore is just restoring a DB, billable infra cost per tenant.
**Cons**: 10× operational cost. Migrations are now fleet operations. Connection pooling per tenant. Onboarding is slow unless heavily automated.

Used by: Heroku Postgres (per-customer DB), Crunchy Bridge, Snowflake (per-account warehouses), enterprise tiers of many SaaS.

### 4.4 Cluster / region / cell per tenant (or tier)

A whole stack — app + DB + cache + queue — dedicated to a tenant or a cohort of tenants. Connected to **cell-based architecture** ([cell architecture →](../11-reliability/cell-architecture.md)).

**Pros**: maximum isolation, regulatory comfort, blast radius is one cell.
**Cons**: most expensive. Need strong automation: Terraform / Pulumi modules per cell, GitOps for deploys, per-cell observability.

Used by: AWS itself, Slack's Enterprise Grid, Salesforce Hyperforce, healthcare SaaS, defense SaaS, anything needing hard regulatory boundaries.

---

## 5. Performance isolation

A noisy tenant — one running a 10M-row export at noon — can starve everyone else. Solutions stack:

- **Per-tenant rate limits.** API-gateway or app-level. Token bucket / leaky bucket per tenant key.
- **Per-tenant quotas.** Max queries/sec, max storage, max queue depth. Hard caps with clear errors.
- **Connection budgets.** Cap concurrent DB connections per tenant. PgBouncer with per-user limits, or in-app semaphores.
- **Job isolation.** Tenant-aware queues. A heavy export from one tenant doesn't block normal jobs from others (per-tenant queue partitions, or priority queues with fairness).
- **Compute isolation.** Pin tenants to specific pod pools (heavy or trial-only nodes). Kubernetes taints + tolerations.
- **Caching policies.** Per-tenant cache budgets to prevent one tenant flooding the cache.
- **Cell architecture.** The hardest tenants get their own cells.

Without these, you'll meet the noisy-neighbor problem the first time a customer enthusiastically tests a bulk import.

---

## 6. Security isolation

A real cross-tenant data leak is the worst day a SaaS can have. The defenses must layer.

### Application layer
- All queries scoped by `tenant_id` enforced in the data layer, not sprinkled in handlers.
- Tenant-scoped auth tokens. The token *is* the tenant context, validated server-side.
- Cross-tenant API calls are explicitly disallowed unless you have a tenant-relationship model.
- Tenant context flowed through tracing (every span tagged `tenant.id`).

### Database layer
- RLS where supported (Postgres). Set tenant context per connection.
- Per-tenant credentials when DB-per-tenant.
- Encryption at rest, with per-tenant key when high-security customers demand it.
- Audit logs that include tenant ID.

### Network layer
- Per-tenant VPCs / subnets for enterprise / regulated tier.
- Egress controls — tenant A's webhook destinations shouldn't be confusable with B's.
- mTLS between services for shared infra. See [Service Mesh →](../03-apis/service-mesh.md).

### Compute layer
- Container per tenant for sensitive operations. Even better: micro-VM (Kata, Firecracker) for hostile workloads (CI runners, code execution sandboxes).
- No shared local disk between tenants.

### Process and culture
- Tenant-leak fire drills. Red-team a missing `tenant_id`.
- Static analysis to flag tenant-naive queries.
- Code review checklists.
- Onboarding training that emphasizes the rule: every query, every cache key, every metric, every log line carries a tenant ID.

---

## 7. Operational isolation

Even with perfect security, multi-tenancy creates operational pain points.

### Migrations
- Shared schema: one DDL, applies to all tenants. Use expand-contract for big changes.
- Per-tenant schema/DB: N migrations. Tooling (Atlas, Sqitch, Flyway with per-tenant scope) becomes essential.
- Big tenants get migrated in maintenance windows separate from small ones. Don't ALTER TABLE on a 5B-row tenant during a small-customer's nap.

### Backups and restores
- Per-tenant backup: trivial in DB-per-tenant; hard in shared schema (need to filter dumps).
- Per-tenant restore: same. Customers will ask. Plan ahead.

### Observability
- Every metric must have a `tenant_id` label.
- Every log must carry a `tenant_id` field.
- Every trace span must include a `tenant_id` attribute.
- Dashboards filterable per tenant.
- Per-tenant SLOs for enterprise customers — they will read these.

But: **cardinality kills metrics**. Tens of thousands of tenants as label values can blow up Prometheus. Solutions:
- Use **exemplars** for per-tenant drill-down rather than tenant as a metric label.
- Pre-aggregate top N tenants by activity; bucket the rest.
- Use logs + traces for per-tenant analytics; keep metrics low-cardinality.

### Billing
- Usage metering per tenant (calls, storage, computed minutes).
- Reconciliation between the metering system and the actual logs/traces.
- Soft caps with notification; hard caps with abuse handling.

### Deploys
- Most deploys ship to all tenants simultaneously.
- For risky changes: shard deploys by tenant tier (canary on free tier first).
- Feature flags per-tenant (see [Feature Flags →](./feature-flags.md)) for gated rollouts.

### Incident response
- Tenant impact assessment in the first 5 minutes. "Is one tenant affected, or all?"
- Per-tenant runbooks for known recurring issues.

---

## 8. Architecture patterns

### Pool model
Every tenant shares the entire stack; only data is scoped. Simplest, cheapest. Default for early SaaS.

### Silo model
Every tenant has its own stack — compute, DB, network. Highest isolation. Used for enterprise tiers and regulated customers.

### Pool + Silo (hybrid)
Most production SaaS. Free / SMB tenants share. Enterprise customers run in dedicated cells. The application code is the same; the deployment differs.

### Bridge model
Pool for compute, silo for data. Each tenant gets their own DB, but they all hit the same app servers. Common compromise.

### Cell architecture
Tenants assigned to "cells" — independent stacks of bounded size. New tenants flow to new cells once a cell is full. Blast radius = one cell. See [Cell-Based Architecture →](../11-reliability/cell-architecture.md). Used by Slack (Enterprise Grid), Stripe internally, AWS.

The tenant-to-cell mapping is a routing layer — a tenant lookup table, plus a tenant-aware router (often the API gateway).

---

## 9. Worked example — a real SaaS architecture sketch

A B2B SaaS with three pricing tiers:

```
                       ┌──────────────────┐
                       │  Tenant Router   │  (tenant_id → cell)
                       └─────────┬────────┘
                                 │
        ┌────────────────┬───────┴───────┬───────────────┐
        ▼                ▼               ▼               ▼
  ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌─────────────┐
  │  Cell A  │    │  Cell B  │    │  Cell C  │    │  Enterprise │
  │ (Free)   │    │   (Pro)  │    │   (Pro)  │    │     XYZ     │
  │ shared   │    │  shared  │    │  shared  │    │  dedicated  │
  │ DB+app   │    │  DB+app  │    │  DB+app  │    │  cluster    │
  └──────────┘    └──────────┘    └──────────┘    └─────────────┘
   1000s of        500 tenants     500 tenants     1 tenant
   small tenants                                   (in own VPC,
   on RLS                                          own region opt)
```

- Free tenants: pooled, RLS-protected, aggressive quotas. Cell size: thousands of tenants.
- Pro tenants: pooled in multiple cells, hundreds per cell. Tenants assigned at sign-up with re-balancing for fairness.
- Enterprise: own cell, own VPC, optionally own region for data residency.

The same application code runs everywhere; the deployment topology differs.

---

## 10. Tenant onboarding

The shape of onboarding determines a lot:

- **Pool model**: API call → INSERT into tenants table → tenant is live in milliseconds.
- **Schema-per-tenant**: API call → create schema → run migration → tenant live in seconds.
- **DB-per-tenant**: API call → provision DB → bootstrap schema → register tenant → live in minutes (or precreated from a hot pool).
- **Cell-per-tenant**: API call → Terraform apply → tenant live in tens of minutes (or use precreated cells).

The "tenant is live in milliseconds" is the SaaS dream. For high-isolation tiers, you keep a **pool of warm cells** ready to assign, so onboarding stays fast.

---

## 11. Tenant offboarding (and deletion)

This is the GDPR / "right to be forgotten" problem. You must be able to delete a tenant's data — completely, verifiably, including backups.

Hard parts:
- Cross-tenant references (audit logs that contain user emails).
- Search indexes (Elasticsearch shards mixing tenants).
- Cached data.
- Long-lived backups — typically you mark for deletion and let retention drop them.
- Third-party data shipped to vendors (analytics, marketing tools).

A multi-tenant data model should be designed for deletion from day one. Single-table tenant scoping makes "delete from X where tenant_id = ?" straightforward. Cross-cutting indexes and aggregations make it hard.

---

## 12. Common Mistakes / Anti-Patterns

- **Tenant ID added "later."** Retrofitting is hard. Every table, every cache key, every metric, every log line — touch them all.
- **Tenant ID only in the application layer.** Without DB-level enforcement (RLS), one missed WHERE leaks. Belt **and** suspenders.
- **Shared cache without per-tenant key prefixes.** Stale data, leaked data. Use `tenant:42:user:99` keys.
- **No rate limiting per tenant.** One customer DDoSes another via shared infra. Quotas per tenant always.
- **Tenant ID as a high-cardinality metric label.** Prometheus melts. Use logs/traces for per-tenant drill-down; metrics for aggregates.
- **One DB per tenant before you have automation for it.** You'll spend more on humans than on machines.
- **One DB for everyone at 10,000 customers.** Hot tenants destroy the cluster. Plan migration paths to dedicated DBs for top-X tenants.
- **Mixing tenant cardinalities in one cell.** A million-row tenant next to a 10-row tenant fights for the same DB cache.
- **No tenant context in tracing.** Debugging multi-tenant systems without `tenant.id` on spans is misery.
- **Forgetting tenant isolation in background jobs.** A worker pulling jobs from a shared queue can leak across tenants if not careful.
- **"Just hardcode an exception for this big customer."** That's how single-tenant forks happen. Resist; build the tier instead.
- **No per-tenant SLO tracking.** Enterprise customers will ask. Have a number.
- **Sloppy onboarding/offboarding.** Half-provisioned tenants in odd states; failed deletions leaving data in S3.
- **Sales selling isolation you haven't built.** "Dedicated infrastructure" must mean something specific and verifiable.

---

## 13. Cheat Card

```
PURPOSE   Serve many customer organizations from shared (or
          partially shared) infrastructure, with isolation that
          matches each tenant's tier and risk.

ISOLATION DIMENSIONS
  Data         shared schema → schema-per-tenant → DB-per-tenant → cell
  Performance  rate limits + quotas + dedicated pools
  Security     RLS, mTLS, VPC, per-tenant keys
  Operational  per-tenant metrics, logs, traces, billing

MODELS
  Pool       all shared; data scoped by tenant_id  (cheap, simple)
  Silo       per-tenant stack                       (max isolation)
  Bridge     pooled compute, siloed data            (common compromise)
  Cell       pools-of-pools; one tenant = one cell  (enterprise / regulated)

WHEN TO USE WHICH
  Free / SMB     pool with RLS
  Pro / Mid     pool in multiple cells, strong quotas
  Enterprise    silo or dedicated cell, own VPC, own keys
  Regulated     dedicated cell + region + customer-managed keys

NON-NEGOTIABLES
  tenant_id on every row, key, metric label/log/trace
  RLS or equivalent DB-level enforcement
  Per-tenant rate limits + quotas
  Tenant-aware deploys (canary tiers, feature flags per tenant)
  Tenant offboarding plan (GDPR delete works end-to-end)

PITFALLS
  Adding tenant_id later
  Tenant ID enforced only in app code
  No per-tenant rate limit → noisy neighbor outage
  Tenant ID as Prometheus label → cardinality bomb
  Mixed-size tenants in one pool → big ones starve small ones
  Sales selling isolation you haven't built
  No per-tenant tracing → debugging is misery

RULE   Pick the lowest-isolation tier that wins the customer.
       Build the higher tiers as tenants demand them. Don't
       over-engineer day one; don't under-engineer day 100.
```

---

## 14. Resources

### Books
- *Software Architecture: The Hard Parts* — Ford, Richards, Sadalage, Dehghani. Has multi-tenancy sections.
- *Designing Data-Intensive Applications* — Martin Kleppmann. Foundational, especially partitioning.
- *Multi-Tenant Software Architecture* — AWS / various whitepapers.

### Documentation
- **AWS SaaS Factory** — Multi-tenant patterns: <https://aws.amazon.com/partners/programs/saas-factory/>
- **Azure** — Multi-tenant architectural guidance: <https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/overview>
- **Postgres Row-Level Security** — <https://www.postgresql.org/docs/current/ddl-rowsecurity.html>
- **Stripe** — engineering essays on tenant + workload isolation.

### Articles
- "Multi-tenant SaaS architecture" — AWS whitepaper.
- "Cells: A pattern for limiting the blast radius" — Werner Vogels: <https://www.allthingsdistributed.com/2019/08/modular-cell-based-architecture.html>
- "How GitLab is multi-tenant" — GitLab engineering blog.
- "Building multi-tenant SaaS on Kubernetes" — various KubeCon talks.
- "How Notion built its data model" — Notion engineering.

### Videos
- *AWS re:Invent* — many SaaS Factory sessions per year.
- *Slack Enterprise Grid architecture* — past Slack engineering talks.
- ByteByteGo — "Multi-Tenant SaaS Architecture."

### Tools
- **Postgres RLS** — built-in tenant isolation.
- **PgBouncer** — per-user connection pooling.
- **Istio / Linkerd** — mTLS and per-tenant routing.
- **OPA / Kyverno** — policy enforcement for per-tenant deploys.
- **Crossplane** — provision per-tenant cloud resources via K8s.
- **Vault** — per-tenant secret management.
- **OpenCost / Kubecost** — per-tenant cost attribution.

### Adjacent reading
- [Multi-Tenant SaaS Architecture →](../19-advanced/multi-tenant-saas.md)
- [Blast Radius & Cell-Based Architecture →](../11-reliability/cell-architecture.md)
- [Sharding & Partitioning →](../04-databases/sharding-partitioning.md)
- [Container Orchestration (Kubernetes) →](./kubernetes.md)
- [Infrastructure as Code →](./iac.md)
- [Rate Limiting →](../03-apis/rate-limiting.md)
- [Service Mesh →](../03-apis/service-mesh.md)
- [Zero Trust Architecture →](../12-security/zero-trust.md)
- [Database Migrations at Scale →](../04-databases/migrations.md)

---

*Previous:* [← Immutable Infrastructure](./immutable-infra.md)  |  *Up:* [README ↑](../README.md)
