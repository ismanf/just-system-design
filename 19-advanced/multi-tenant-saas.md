# Multi-Tenant SaaS Architecture

> **TL;DR** — A **multi-tenant SaaS** serves many customer organizations from shared software, with **logical isolation** between their data, performance, security, and operations. The architectural spectrum runs from **pooled** (every tenant shares everything, separated only by `tenant_id` columns) through **bridge** (shared compute, dedicated databases per tenant) to **silo** (every tenant gets its own dedicated stack, often in its own VPC or region). Most successful SaaS evolves through this spectrum: **start pooled for low-tier customers, add bridge/silo tiers as enterprise demands appear**. The defining technical concerns are **data isolation** (tenant boundaries), **performance isolation** (noisy neighbor), **security isolation** (cross-tenant leakage), **operational isolation** (per-tenant deploys, observability, billing). This page goes deeper on the patterns sketched in [Multi-Tenancy →](../15-deployment/multi-tenancy.md), with focus on the architectural choices that matter most for SaaS specifically: **tenant routing, identity, customization, lifecycle, and the moment you need to split your monolithic SaaS into per-tenant stacks**.

---

## 1. The big picture

```
                       ┌────────────────────────┐
                       │ Identity / tenant      │
                       │ resolution layer       │
                       │  subdomain / claim / DID│
                       └──────────┬─────────────┘
                                  │
              ┌───────────────────┼─────────────────────┐
              ▼                   ▼                     ▼
        ┌──────────┐         ┌──────────┐         ┌────────────┐
        │ Pool     │         │ Pool     │         │ Silo:      │
        │  (small  │         │  (mid    │         │ Enterprise │
        │   plans) │         │   plans) │         │ "Acme Co." │
        │ shared DB│         │ shared DB│         │ own VPC,   │
        │ + RLS    │         │ + RLS    │         │ DB, keys   │
        └──────────┘         └──────────┘         └────────────┘
```

Every tenant request resolves to:
1. **Which tenant am I?** — derived from subdomain, JWT claim, custom domain, or API key.
2. **Which pool / cell / silo serves this tenant?** — a routing decision the platform makes.
3. **Which data partition?** — `tenant_id` filter, schema, or dedicated database.

The architecture decides who shares what.

For the foundational concepts and isolation-dimension framework, read [Multi-Tenancy →](../15-deployment/multi-tenancy.md) first. This page focuses on **SaaS-specific patterns** that build on those primitives.

---

## 2. The three architectural tiers (revisited for SaaS)

### Pool

Every tenant shares everything. Data scoped by `tenant_id`. Cheapest. Fastest onboarding (an INSERT). Mostly what early-stage and self-serve SaaS uses.

When pool starts to hurt:
- A few tenants are 100× the size of others — query optimizer chokes.
- Enterprise customer demands "dedicated infrastructure."
- Compliance demands data residency / customer-managed keys.
- One tenant's workload affects another's (noisy neighbor).

### Bridge

Pooled compute, siloed data. App servers are shared; each tenant gets its own database (or schema). Solves data-isolation concerns without spinning up entire stacks. Common compromise for mid-market.

### Silo (cell / pod / customer-instance)

Dedicated everything — own cluster, own database, own region, own keys. Used for top-tier enterprise customers and regulated industries (HIPAA, FedRAMP, FINRA).

Most successful SaaS in 2026 runs **all three tiers simultaneously**:

```
Free / Pro tenants    →  Pool with RLS
Business / Team tier  →  Pool in dedicated cells (Bridge-ish)
Enterprise            →  Silo with VPC / customer keys / customer region
```

Sales tier maps to architecture tier. Engineering keeps **one codebase** that deploys differently.

---

## 3. Tenant resolution — how to identify "who is asking"

The very first decision on every request: *which tenant?*

### Subdomain

`acme.myapp.com` → tenant `acme`. The dominant pattern for B2B SaaS. Stripe (`dashboard.stripe.com`), Slack (`acme.slack.com`), Notion (`acme.notion.so`).

Pros: clean URLs, no headers needed, easy to share, separate cookies per tenant.
Cons: TLS wildcard certificate management, custom domains add complexity, can leak tenant existence.

### Custom domain

`portal.acme.com` (Acme's domain) → tenant `acme`. The "white-label" feature most enterprise SaaS eventually adds.

Implementation: tenant adds a CNAME to your edge; you provision a TLS cert (Let's Encrypt + ACME, or your CA); you route incoming requests by `Host` header.

### Path prefix

`myapp.com/acme/...` → tenant `acme`. Less common. Easier to set up but uglier and shares cookies across tenants by default.

### JWT / OAuth claim

After authentication, the bearer token contains a `tenant_id` claim. Used for API requests, mobile apps, internal services.

### Header

`X-Tenant-ID: acme` from authenticated clients. Used in service-to-service contexts and admin tools.

In practice: **subdomain + JWT claim** is the most common combination — subdomain for human users, JWT for API.

---

## 4. Tenant context propagation

Once you know the tenant, **every layer below must know it too**. Lose tenant context anywhere and you risk cross-tenant data leakage.

Patterns:

- **Async-local storage / context.** Node `AsyncLocalStorage`, Python `contextvars`, Go `context.Context`, Java `ThreadLocal` (or scoped contexts in modern frameworks). Store `tenant_id` at the start of the request; every downstream call reads it.
- **DB connection per tenant.** PostgreSQL: `SET app.tenant_id = '...'` at connection acquire; RLS policies read it.
- **HTTP propagation.** Every outbound RPC carries `X-Tenant-ID`.
- **Tracing.** Every span tagged with `tenant.id`. Critical for debugging.
- **Logging.** Every log line includes `tenant_id`. Otherwise debugging multi-tenant production is misery.
- **Metrics.** Tenant ID as a metric label — but watch cardinality (a million tenants = a million label values = exploded Prometheus).

The discipline: **`tenant_id` should be as load-bearing as `user_id`**. Forget it once, and your audit trail has a hole.

---

## 5. Database isolation patterns at the SaaS layer

The deep dive on data isolation lives in [Multi-Tenancy →](../15-deployment/multi-tenancy.md). For SaaS specifically:

### Postgres + Row-Level Security

The pooled-data sweet spot. Enable RLS:

```sql
CREATE POLICY tenant_isolation ON orders
  USING (tenant_id = current_setting('app.tenant_id')::int);
```

Set `app.tenant_id` per connection at acquire. Every query for the wrong tenant returns zero rows, even if the application code is buggy. **Defense in depth** — the application enforces and the database enforces.

### Schema-per-tenant

Each tenant gets its own Postgres schema (or MySQL database). Same physical instance. Used by Heroku, some Atlassian products historically.

Pros: stronger logical isolation, per-tenant backup/restore.
Cons: Postgres performance degrades past thousands of schemas; migrations are N operations; connection pool harder to share.

### Database-per-tenant (managed)

Modern serverless databases (Neon, Turso, PlanetScale, Supabase) make "DB per tenant" cheaper than it used to be. Provisioning a fresh DB takes seconds, idles cost ~$0.

Used by: many newer SaaS products optimizing for strong per-tenant isolation + the speed of a shared platform.

### Cell architecture

Tenants assigned to **cells** — independent stacks of bounded size (~100–1000 tenants per cell). New tenants flow to new cells once a cell fills. Blast radius = one cell. See [Cell-Based Architecture →](../11-reliability/cell-architecture.md). Used by Slack Enterprise Grid, Stripe internally, Salesforce Hyperforce.

The picture in production: most successful SaaS combines several. Free users on pooled RLS, mid-tier in dedicated cells, enterprise in silos.

---

## 6. Identity — users, tenants, and the matrix between them

A user can belong to multiple tenants (working at Acme and consulting for Beta Corp). A tenant has many users. This **N:M relationship** is the source of much SaaS auth complexity.

Patterns:

### Single-tenant-per-user

User belongs to one tenant. Easy. Login → JWT with tenant_id. Used by self-serve products where switching is rare.

### Multi-tenant-per-user

User belongs to many tenants. Stripe lets you be on the dashboard of multiple Stripe accounts; Slack lets you be in many workspaces; GitHub lets you be in many orgs.

Implementation: identity (user_id) is a separate entity from tenant membership. Login establishes user identity; selecting a tenant grants a tenant-scoped session.

### SSO and SCIM

Enterprise customers want their SSO IdP (Okta, Azure AD, OneLogin) to be the source of truth for who can log in. They want **SCIM** to push/pull user lifecycle (add user, remove user, change role).

Implementations: every enterprise SaaS supports SAML, OIDC, and SCIM. Frameworks like **WorkOS**, **Auth0 Enterprise**, **Stytch B2B**, **Clerk B2B**, **PropelAuth** abstract this. See [SSO →](../12-security/sso.md).

### Custom permission models

RBAC, ABAC, ReBAC — every SaaS evolves a richer permission model over time. Auth0 / Permit.io / OpenFGA / Zanzibar-derived systems are common. See [RBAC, ABAC, ReBAC →](../12-security/access-control.md).

The lesson: **identity is harder than data isolation**, and the cost of getting it wrong is enterprise-customer-losing. Plan for SSO and SCIM by the time you hit your first enterprise tier.

---

## 7. Customization without forking

Enterprise customers demand customization. Custom branding, custom workflows, custom fields, custom data schemas. Done wrong, this becomes "every customer has a unique codebase you can't maintain."

Strategies, from least to most invasive:

### Configuration (settings, feature flags)

Per-tenant configuration values. Logos, colors, feature toggles, default behaviors. Stored in a tenant settings store. Cheap, safe, reversible.

### Custom fields

Allow tenants to add their own fields to your standard objects ("Customer.account_manager: string"). Stored as JSONB column or separate `tenant_custom_fields` table. Salesforce-style.

### Workflows / rules engines

Tenant defines their workflow ("when an order > $10k, route to manager"). You provide the engine; they provide the configuration. Often a simple state machine or rule DSL.

### Embedded extensions / WASM / sandboxed code

Tenant uploads JavaScript / WASM / Lua / Python that runs in your platform. Strong sandboxing required. Increasingly common as plugin platforms mature.

### Per-tenant feature flags

Toggle features per tenant. Critical for staged enterprise rollouts. See [Feature Flags →](../15-deployment/feature-flags.md).

### Branded experience

Custom domains, custom emails (DKIM/SPF on customer domain), custom SSO with tenant's brand, white-label.

**What you don't do**: per-tenant code branches. The moment you fork the codebase per customer, you've lost the SaaS model.

---

## 8. Tenant lifecycle — onboarding, suspension, deletion

### Onboarding

The shape determines a lot:

- **Pool model**: API call → INSERT into tenants table → tenant live in milliseconds.
- **Schema-per-tenant**: API call → create schema → run migration → tenant live in seconds.
- **DB-per-tenant**: API call → provision DB → bootstrap schema → register tenant → live in seconds to minutes.
- **Cell-per-tenant**: API call → Terraform / Crossplane apply → tenant live in tens of minutes (or use precreated cells).

For high-isolation tiers, keep a **pool of warm cells** ready to assign, so onboarding stays fast.

### Trial / freemium → paid conversion

Trials should be the same tier as eventual paid plans. Don't make the upgrade a "migration" — it's a flag flip plus a Stripe subscription. Otherwise you'll have orphaned trial data and confused customers.

### Suspension

A delinquent tenant gets suspended. Patterns:

- **Read-only**: data accessible but no new writes. Lowest-friction.
- **Hidden**: redirect to a billing page, data preserved.
- **Frozen**: API returns 402 / 403; data preserved for N days.
- **Deleted**: data marked deleted, GC'd after retention.

Always preserve data through suspension. Customers come back; deleting is unrecoverable.

### Deletion — the GDPR problem

You **must** be able to fully delete a tenant: data, backups (eventually), search indexes, caches, replicas, third-party syncs. End-to-end deletion is a real engineering project.

Hard parts:
- Cross-tenant references (audit logs that contain customer emails).
- Search indexes (Elasticsearch shards mixing tenants).
- Cached data (Redis, CDN).
- Long-lived backups (typically marked for deletion, dropped on retention).
- Third-party syncs (analytics tools, marketing tools, CRMs). Each integration needs a delete-propagation path.

Design for deletion from day one. Single-table tenant scoping makes "delete from X where tenant_id = ?" tractable. Cross-cutting indexes and denormalized aggregates make it hard.

### Data export

Tenants want to export their data. Provide a portable format (JSON, CSV, Parquet). Standard for B2B SaaS; legally required in some jurisdictions.

---

## 9. Per-tenant operations and observability

### Per-tenant metrics

Every metric tagged with tenant context. But tenant_id as a metric label can explode Prometheus cardinality (millions of tenants × dozens of metrics = billions of time series). Solutions:

- **Aggregate top-N tenants by activity; bucket the rest.**
- **Use logs and traces for per-tenant drill-down**, not metrics.
- **Exemplars** to link aggregate metrics to specific tenant traces.

### Per-tenant SLOs

Enterprise customers ask "what's your SLO for our tenant?" Pool tenants share the platform's SLO. Silo tenants can have dedicated SLOs (often contractual).

### Per-tenant billing / usage metering

The metric system feeds billing. Be **reconcilable** — the bill must match what the customer can see in their own dashboards. Drift here destroys trust faster than almost any other bug.

Patterns:
- Event-based usage (every API call, every storage byte-hour).
- Aggregation pipeline that produces invoiceable usage records.
- Reconciliation between the metering pipeline and the truth source (e.g., the actual DB).

Tools: **Stripe Metering, Lago, Orb, Metronome** — managed usage-based billing.

### Per-tenant deploys

Rare but real:
- Enterprise tenants on the silo tier deploy on their own schedule (sometimes weeks behind).
- Region-specific deploys.
- Feature flag per-tenant for staged enterprise rollouts.

Don't promise this lightly. It's operationally expensive.

### Cross-tenant blast radius

Cell architecture limits blast radius. A bad deploy hits one cell; other cells are fine. Silos take this further — one tenant's outage doesn't touch others.

For pooled tenants: a bug or DB outage hits everyone. Tier your customer communications accordingly.

---

## 10. Pricing models and architectural fit

Architecture and pricing reinforce each other:

| Pricing model | Architecture fit | Examples |
|---|---|---|
| **Per-seat** (monthly) | Pool with user counts | Slack, Notion, Linear |
| **Per-tenant flat** | Any | Many small B2B tools |
| **Usage-based** | Pool with metering | Stripe, AWS, Vercel, OpenAI API |
| **Hybrid (seat + usage)** | Pool with both meters | Datadog, Snowflake |
| **Enterprise (negotiated)** | Silo / dedicated | All enterprise tiers |

Usage-based billing demands **fine-grained metering** that integrates with the data plane. Get this right early or it's a multi-quarter migration later.

---

## 11. The case for sharing — and the case against

The pool model is cheap and easy. The silo model is safe and isolated. The trade-off isn't fixed; it shifts as the business matures.

### Reasons to keep tenants pooled

- Cost: shared infra is 10–100× cheaper per tenant.
- Operational simplicity: one cluster, one DB, one deploy.
- Speed of feature rollout: one place to ship.
- Cross-tenant features (benchmarking, peer comparison) — only possible with shared data.

### Reasons to silo

- Enterprise sales demand it.
- Compliance demands it (HIPAA, FedRAMP, regional data residency).
- Noisy neighbors hurt p99 in pooled tiers.
- Security blast-radius concerns ("if you're breached, who else is exposed?").
- Performance — one customer's huge data set drags everyone in their pool.
- Custom regulatory needs per region.

A common pattern: **majority of tenants in pool, top 1% in silos**. The economics work because the silo tier is paying enough to justify the dedicated infrastructure.

---

## 12. Stripe / Slack / Linear-style case patterns

Three reference architectures from the public record:

### Stripe (heavily pooled, sharded internally)

- Single global API surface; tenants identified by account ID.
- Heavy internal sharding for throughput, but tenants don't see it.
- Idempotency, versioning, and observability all tenant-aware. See [Idempotency →](../03-apis/idempotency.md), [API Versioning →](../03-apis/versioning.md).
- Some regulated workloads (Stripe Treasury) live in dedicated cells.

### Slack (cell-based)

- Workspaces (tenants) live in cells.
- Enterprise Grid is a higher-tier offering with dedicated infrastructure and cross-workspace identity.
- Real-time messaging is per-cell; cross-cell federation handles Enterprise Grid.

### Linear (pool with strong customer mental model)

- Per-workspace data in shared Postgres with strict RLS.
- Single deployment serves everyone.
- Heavy real-time sync layer (their custom CRDT-style sync engine) per workspace.

### Notion, Figma

Hybrid models. Notion's data structure (block-based document graph) lives in shared databases sharded by workspace. Figma's CRDT-based realtime collab layer is per-document, scaled horizontally.

The common thread: **strong tenant_id discipline, careful blast-radius planning, and a clear migration path to silos for enterprise**.

---

## 13. Common Mistakes / Anti-Patterns

- **Adding tenant_id "later."** Touching every table, cache key, log, metric, trace, queue — months of work.
- **Application-layer tenant filtering only.** No DB-level enforcement (RLS). One missed WHERE clause = cross-tenant leak.
- **Per-tenant code branches.** "Just this once" becomes "we have 47 forks." Use configuration / flags / extensions.
- **No tenant_id on logs and traces.** Multi-tenant debugging is impossible.
- **Mixing huge and tiny tenants in one pool.** Big tenants destroy small tenants' p99.
- **No rate limiting per tenant.** One tenant DDoSes another via shared infra.
- **Sales selling isolation you haven't built.** "Dedicated infrastructure" must mean something specific. Define and verify.
- **No tenant offboarding plan.** GDPR delete request comes; you scramble. Build delete from day one.
- **Forgetting cross-tenant aggregations.** Want a "platform analytics dashboard"? Now you have a cross-tenant query that bypasses RLS. Plan its security carefully.
- **One DB / one cluster forever, no silo plan.** First big enterprise prospect asks for dedicated; you say no; you lose the deal.
- **No usage metering plan early.** Adding it later means re-instrumenting everything.
- **Treating "decentralized cells" as totally independent.** Almost always there's a control plane (auth, billing, deploy) that's shared. Plan its blast radius.
- **Storing PII in caches and search indexes without per-tenant scoping.** Deletion becomes impossible to verify.
- **Multi-tenant routing via path prefix with shared cookies.** Tenants see each other's cookies. Use subdomains.
- **Subdomain wildcard cert with no isolation between subdomains.** A bug in one tenant's content can XSS into another's.
- **Background workers pulling from a shared queue with no tenant awareness.** A heavy tenant starves others' jobs.

---

## 14. Cheat Card

```
PURPOSE   Serve many customer organizations from a shared SaaS
          platform — with isolation per tenant on data, performance,
          security, and operations, scaling tier by tier.

ARCHITECTURAL TIERS
  Pool     all shared; data scoped by tenant_id (cheapest)
  Bridge   shared compute, per-tenant data
  Silo     dedicated stack per tenant (strongest isolation)

WHEN EACH WINS
  Free / SMB        Pool with RLS
  Pro / Mid         Pool in cells, strong quotas
  Enterprise        Silo with VPC, dedicated DB, customer keys
  Regulated         Silo + region + customer-managed keys

TENANT RESOLUTION
  Subdomain (acme.app.com)        — default B2B
  Custom domain (portal.acme.com) — enterprise
  JWT claim                        — APIs and services
  Header (X-Tenant-ID)             — service-to-service

CONTEXT PROPAGATION (NON-NEGOTIABLE)
  tenant_id on every DB query (RLS enforces too)
  tenant_id on every log, metric, trace
  Propagate via async-local / context across all calls

IDENTITY MATRIX
  User : Tenant = N : M (always plan for many tenants per user)
  SSO + SCIM by the first enterprise tier
  RBAC → ABAC → ReBAC as the model evolves

CUSTOMIZATION (DO, DON'T FORK)
  Per-tenant config / settings
  Custom fields on standard objects
  Workflow / rules engines
  Sandboxed extensions (WASM / JS)
  Per-tenant feature flags
  White-label branding

LIFECYCLE
  Onboarding: keep <1 minute for self-serve, <30 min for silo
  Suspension: read-only / hidden / frozen — preserve data
  Deletion: GDPR-grade end-to-end (data, backups, indexes, caches)
  Data export: portable format always

OBSERVABILITY
  Per-tenant logs, traces (with id)
  Per-tenant metrics with care (cardinality)
  Per-tenant SLOs for enterprise tier
  Usage metering reconcilable with billing

PRICING / ARCHITECTURE
  Per-seat       → pool with user counts
  Usage-based    → pool with metering pipeline
  Enterprise     → silo with negotiated terms

PITFALLS
  Adding tenant_id later
  App-only filtering, no RLS
  Per-tenant code forks
  Logs/metrics without tenant_id
  Mixed-cardinality tenants in one pool
  No per-tenant rate limiting
  No tenant deletion plan
  Sales selling isolation you haven't built
  Path-prefix routing with shared cookies

RULE   Pick the lowest-isolation tier that wins the customer.
       Build higher tiers as they demand them. Always plan for
       tenant_id everywhere, deletion from day one, SSO/SCIM
       before your first enterprise contract.
```

---

## 15. Resources

### Books and references
- *Software Architecture: The Hard Parts* — Ford, Richards, Sadalage, Dehghani. Has multi-tenancy chapters.
- *Designing Data-Intensive Applications* — Kleppmann. Partitioning chapter applies.
- *Building Multi-Tenant SaaS Architectures* — Tod Golding (AWS SaaS Factory).

### Documentation
- **AWS SaaS Factory** — Multi-tenant patterns: <https://aws.amazon.com/partners/programs/saas-factory/>
- **Azure** — Multi-tenant architectural guidance: <https://learn.microsoft.com/en-us/azure/architecture/guide/multitenant/overview>
- **Google Cloud Architecture Framework** — multi-tenancy chapter.
- **Postgres Row-Level Security** — <https://www.postgresql.org/docs/current/ddl-rowsecurity.html>

### Articles
- "Cells: A pattern for limiting blast radius" — Werner Vogels (AWS): <https://www.allthingsdistributed.com/2019/08/modular-cell-based-architecture.html>
- "How GitLab is multi-tenant" — GitLab engineering blog.
- "How Slack scales workspaces" — Slack engineering.
- "Building multi-tenant Postgres" — Citus / Crunchy Data blogs.
- "Architecting Multi-Tenant Solutions on AWS" — AWS whitepaper.
- "The pyramid of multi-tenant SaaS pricing" — Kyle Poyar and others.

### Videos
- *AWS re:Invent SaaS Factory sessions* — annual.
- *Slack Enterprise Grid architecture* — past Slack engineering talks.
- *Building multi-tenant SaaS on Kubernetes* — KubeCon talks.
- ByteByteGo — "Multi-Tenant SaaS Architecture."

### Tools
- **Postgres RLS** — built-in tenant isolation.
- **PgBouncer** — per-user connection pooling.
- **Istio / Linkerd** — mTLS and per-tenant routing.
- **OPA / Kyverno** — policy enforcement.
- **Crossplane** — per-tenant cloud resource provisioning.
- **OpenFGA / Permit.io / Authz.io** — Zanzibar-style permissions.
- **WorkOS / Auth0 / Stytch / Clerk** — SSO + SCIM + enterprise auth.
- **Stripe Metering / Lago / Orb / Metronome** — usage-based billing.
- **OpenCost / Kubecost** — per-tenant cost attribution.

### Adjacent reading
- [Multi-Tenancy →](../15-deployment/multi-tenancy.md) — foundational deep dive.
- [Cell-Based Architecture →](../11-reliability/cell-architecture.md)
- [Sharding & Partitioning →](../04-databases/sharding-partitioning.md)
- [Service Mesh →](../03-apis/service-mesh.md)
- [API Gateway →](../03-apis/api-gateway.md)
- [Zero Trust Architecture →](../12-security/zero-trust.md)
- [RBAC, ABAC, ReBAC →](../12-security/access-control.md)
- [SSO — Single Sign-On →](../12-security/sso.md)
- [Rate Limiting →](../03-apis/rate-limiting.md)
- [Feature Flags & Dark Launches →](../15-deployment/feature-flags.md)
- [Infrastructure as Code →](../15-deployment/iac.md)
- [Database Migrations at Scale →](../04-databases/migrations.md)

---

*Previous:* [← QUIC & HTTP/3 Internals](./quic.md)  |  *Up:* [README ↑](../README.md)
