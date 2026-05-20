# Geographically Distributed Systems (Multi-Region)

> **TL;DR** — A **multi-region system** runs copies of itself in two or more geographically separated cloud regions or datacenters, so users get lower latency, the business survives regional failures, and data residency requirements are met. The catch: cross-region links are slow (50–200 ms), expensive (egress fees), and *unreliable enough that you must design for partition*. Five architectural shapes dominate: **single-region with multi-region failover**, **active-passive**, **read-local-write-global**, **active-active (multi-master)**, and **regional sharding** (each region owns its users). Each makes a different trade between availability, consistency, latency, and complexity. There is no "just turn on multi-region" button — every layer (DNS, LB, application, database, cache, queue, secrets) has its own multi-region story, and the database is almost always the hardest. The honest take: most companies need multi-AZ from day one; they need multi-region only when the cost of an hour of regional downtime exceeds the cost of operating multi-region permanently.

---

## 1. The Stakes

```
Single region                       Multi-region
─────────────                       ────────────
1 AZ failure   = degraded           1 AZ failure  = unnoticed
1 region down  = total outage       1 region down = degraded, not down
Latency        = local only         Latency       = local for regional users
Cost           = baseline           Cost          = +30–100%
Complexity     = single brain        Complexity   = many brains, hard problems
Data residency = single jurisdiction Data residency = configurable per user
```

You go multi-region when **at least one** of these is forcing your hand:

1. **Availability target** above ~99.95% requires surviving a regional failure.
2. **Geographic latency** for far-away users is unacceptable (e.g., 200 ms RTT for European users on US-only stack).
3. **Data residency** forces you to keep certain users' data in certain regions (GDPR, China data law, financial regulators, government clients).
4. **Disaster recovery** RPO/RTO targets cannot be met with cross-region backups alone.

If none of those applies, **multi-AZ is enough**. Most outages are AZ-scope, not region-scope. Multi-region is expensive, error-prone, and worth it only when warranted.

---

## 2. AZ vs Region — The Terms

A quick anchor:

- **Availability Zone (AZ)** — independent power, cooling, network within a metropolitan area. RTT between AZs in the same region is 1–3 ms.
- **Region** — a cluster of AZs in one geographic area (us-east-1, eu-west-2, ap-southeast-1). RTT between regions is 50–200 ms.
- **Edge location / PoP** — a small presence (CDN, DNS, sometimes compute) much closer to users. Doesn't replace region.

The numbers (rough, AWS-like):

| Path | RTT |
|---|---|
| Same AZ | 0.5 ms |
| Same region, different AZ | 1–3 ms |
| Same continent, different region | 30–80 ms |
| Cross-continent | 100–200 ms |
| Antipodes | 250–350 ms |

These are physics-bounded. No software trick brings cross-Atlantic RTT below ~70 ms — that's the speed of light through fiber. Plan for it.

---

## 3. The Five Architectural Shapes

### 3.1 Single-region with cross-region backup
- Active region runs everything.
- Backups, snapshots, and async replication land in a second region.
- DR plan: in case of regional disaster, restore in the second region.

**Pros**: cheapest, simplest, real DR capability.
**Cons**: RTO measured in hours; data loss = replication lag at moment of failure.
**Use when**: availability target is 99.9%; downtime is survivable; cost matters.

### 3.2 Active-passive (warm standby)
- Primary region serves all traffic.
- Secondary region runs idle / hot enough to take over on DNS / health-check failover.
- Async replication keeps secondary close to current.

**Pros**: minutes-RTO failover; near-zero data loss on planned failover.
**Cons**: secondary capacity sits idle; failover is risky and rarely-tested.
**Use when**: 99.95% target; one-region writes are fine; failover events are rare.

### 3.3 Read-local, write-global
- Reads served from the nearest region for low latency.
- All writes go to one "leader" region; replicate read-only to others.
- Common for read-heavy global apps (news, e-commerce browsing).

**Pros**: low-latency reads everywhere; simple write story (one primary).
**Cons**: writes from distant users have high latency; primary region is a critical SPOF.
**Use when**: read/write ratio is 100:1+; users tolerate higher write latency.

### 3.4 Active-active (multi-master)
- Every region accepts writes.
- Conflict resolution required (last-writer-wins, CRDTs, application-level merge).
- Replication is bi-directional or n-way; usually async.

**Pros**: lowest write latency everywhere; survives region failure with no failover.
**Cons**: conflict resolution is hard; eventual consistency is the only realistic model; debugging is brutal.
**Use when**: globally distributed users + low-latency writes are mandatory; you accept eventual consistency.

### 3.5 Regional sharding (geo sharding)
- Each region is the **single owner** of a subset of users / tenants.
- No cross-region writes for owned data; cross-region reads if a user lives in another region.
- Compliance-friendly (EU data in EU).

**Pros**: clean ownership; ACID per region; compliance easy.
**Cons**: cross-region collaboration is awkward; user mobility breaks the model; global queries require fan-out.
**Use when**: users naturally cluster geographically; data residency drives the architecture.

These shapes compose. A typical pattern: **regional sharding** at the top level (user is in one home region) + **active-active replication of certain global data** (account metadata, billing, catalog) + **read replicas** within each region.

---

## 4. Per-Layer Choices

Multi-region isn't one decision — it's one decision per layer.

### DNS
- **Geo-DNS / Latency-based routing** — route user to nearest healthy region. (Route 53 latency policy, Cloud DNS, NS1.)
- **Health checks** at the DNS layer enable automatic failover.
- TTLs matter: short TTLs (30–60 s) allow fast failover; clients and resolvers may ignore TTLs anyway.

### Load balancers
- **Global LBs**: AWS Global Accelerator, GCP Global LB, Azure Front Door. Anycast IP fronts multiple regional backends.
- **Regional LBs** inside each region.
- Health checks at both levels.

### CDN / edge
- Static and cacheable content served from edge PoPs (CloudFront, Fastly, Cloudflare, Akamai).
- Edge logic (Cloudflare Workers, CloudFront Functions, Lambda@Edge) for cheap geographic logic.

### Application tier
- Stateless services are easy — deploy identical fleets in each region.
- Routing: nearest region by default; sticky-by-user-region for stateful sessions.

### Cache layer
- Regional caches in each region (Redis, Memcached, app cache).
- Cross-region cache replication is rare and usually wrong — local cache + regional DB read is faster.
- Watch for staleness across regions when data crosses.

### Databases — the hard part
The single hardest piece. Options:

| Approach | Examples | Trade-off |
|---|---|---|
| Single primary, async cross-region replicas | Postgres streaming, MySQL replica | Lag; replicas read-only; failover risky |
| Single primary, sync replica | Postgres synchronous_standby | Write latency = RTT; primary blocks if replica down |
| Multi-leader async | BDR for Postgres, MySQL Group Replication | Conflict resolution required |
| Globally-consistent distributed | Spanner, CockroachDB, YugabyteDB, FoundationDB | Strong consistency via consensus; write latency includes quorum RTT |
| Regional sharding | App-level, Vitess, Citus | Per-region ACID; cross-region is app concern |
| Active-active KV with CRDTs | DynamoDB Global Tables, Redis Enterprise Active-Active, Riak | Eventual consistency; designed for last-writer-wins or CRDT-merge |

The most common production answer: **single primary + async replicas + regional sharding for users**, with Spanner/Cockroach for the few datasets that truly need global ACID.

### Object storage
- S3 Cross-Region Replication, GCS multi-region buckets, Azure RA-GRS.
- Async; lag in seconds typically.
- Costs: storage in both regions + replication egress.

### Message brokers / queues
- **MirrorMaker 2** for Kafka cross-region replication.
- Confluent Multi-Region Clusters with synchronous + observer setups.
- SQS / SNS — regional only; cross-region requires explicit forwarding.
- Bus design: regional buses with cross-region fan-out where needed.

### Secrets, config, IAM
- Replicate KMS keys (multi-region KMS in AWS, customer-managed in GCP).
- Vault clusters per region with disaster-recovery replication.
- Identity providers: Cognito user pools, Auth0, Okta all have multi-region stories — verify yours.

### Observability
- Logs, metrics, traces from every region.
- Aggregate to a central region (or multi-region observability stack).
- Beware: cross-region traffic from observability alone can be expensive.

Every layer needs an explicit multi-region answer. Multi-region projects fail when one layer's plan is "we'll figure it out later."

---

## 5. The Latency Cost of Consistency

Strong consistency across regions costs round-trips. The math:

- **Quorum-based** consensus (Raft, Paxos): a write must replicate to a majority. Cross-region quorum = at least one cross-region RTT (50–200 ms) added to write latency.
- **Synchronous replica** in another region: write latency = cross-region RTT.
- **Async replication**: write latency = local; replicas trail by some lag window.

Trade examples:
- **Spanner**: 5 ms intra-zone reads, ~7 ms commits in a region, much more for cross-region commits — explicitly trades latency for ACID at global scale.
- **CockroachDB**: 5–10 ms regional commits; configurable replica placement balances latency vs availability.
- **DynamoDB Global Tables**: writes commit locally; cross-region replicated async; last-writer-wins on conflicts.
- **Postgres with synchronous_standby in another region**: writes pay full RTT for each commit (~80 ms cross-continent).

**PACELC** formalizes this: in normal operation, you pick between **L**atency and **C**onsistency. See [PACELC →](../08-distributed-systems/pacelc.md).

A useful rule: **most data doesn't need global ACID**. Catalog, content, sessions, telemetry, derived data — all fine with eventual consistency. The thin slice that does need ACID (account balances, inventory at the moment of purchase, financial transactions) deserves a globally-consistent store; everything else gets async replication.

---

## 6. Network Topology

Cloud providers offer dedicated network products for multi-region traffic:

- **AWS Transit Gateway / Cloud WAN** — region-to-region private routing.
- **AWS Global Accelerator** — anycast IPs over AWS backbone.
- **GCP Cloud Interconnect / Network Connectivity Center**.
- **Azure ExpressRoute Global Reach / Virtual WAN**.

Why bother:
- Lower latency than public internet (often).
- Lower jitter.
- Lower egress cost.
- Better security posture (private addressing).

For latency-sensitive multi-region paths, dedicated network paths can shave 20–30 ms and save real money.

---

## 7. Cost — The Hidden Tax

Multi-region costs more than 2× single-region. Drivers:

- **Idle capacity**: warm standby resources don't earn anything.
- **Data transfer**: cross-region egress is $0.02–0.05/GB; replication can dominate.
- **Replicated storage**: 2× S3, RDS, DynamoDB capacity.
- **Cross-region API calls**: chatty calls add cost and latency.
- **Observability**: shipping logs and traces across regions adds up.
- **Operational overhead**: more dashboards, alerts, on-call complexity.

Common rough estimate: **multi-region active-active is 2–3× single-region cost**. Read-local, write-global active-passive is closer to 1.5×. Regional sharding can be cheaper than active-active because each region is a separate cluster, not bidirectionally replicated.

Always model the bill **per layer** before committing.

---

## 8. Failure Modes

### Region failure
What you went multi-region for. With good DNS / health checks and a tested failover, this is what saves you. Without testing, you'll discover the standby hasn't actually replicated some critical state.

### Cross-region partition
The link between regions is down, but each region is up. Now you have:
- Two leaders thinking they're the only one (split-brain).
- Replicas falling further behind without bound.
- Active-active diverging.

Mitigations: explicit quorum across regions (Spanner, Cockroach); manual operator intervention (active-passive); CRDTs / conflict resolution (active-active KV).

### Slow region (degraded, not down)
Worse than a clean failure. Health checks pass, latency is 5×, retries pile up, downstream effects cascade. Tooling needed: latency-based health checks, circuit breakers per region, per-region SLO dashboards.

### Replication lag spike
A primary surge produces a backlog the replica can't keep up with. Read replicas serve stale data; failover means data loss; commits to primary slow if synchronous. Plan to detect and react.

### Cross-region API timeout cascade
Service A in us-east calls service B in eu-west; B is slow; A's requests pile up. Always **timeout cross-region calls aggressively** and prefer local fallbacks.

### Data residency violations
A bug routes EU user traffic to US region; user's data ends up in US storage. Compliance incident. Mitigations: strict routing at the LB layer; per-region data-plane enforcement; periodic audits.

### Failover not actually tested
Most multi-region outages where the failover "would have saved us" are failures of untested failover. **Run game days quarterly.** See [Chaos Engineering →](../11-reliability/chaos-engineering.md).

---

## 9. Worked Example — A Global SaaS

A B2B SaaS with US and EU customers, GDPR compliance, 99.99% target.

```
LAYER          CHOICE
─────          ──────
DNS            Route 53 geo + latency routing → nearest healthy region
CDN            CloudFront / Cloudflare → assets served from edge
Edge logic     Cloudflare Workers for auth / region-routing decisions
LB             Regional ALB in us-east-1 and eu-west-1
App            Stateless K8s clusters per region (HPA on each)
Cache          Redis Cluster per region (no cross-region replication)
DB primary     Per-region Postgres (regional sharding by tenant_home_region)
DB cross-rgn   Async logical replication of "global" tables (billing
               plans, feature flags, system catalog)
Spanner/CRDB   For a small slice of inventory and licenses requiring
               global ACID
Object storage S3 buckets in each region; CRR for backups + DR
Message bus    Kafka per region; MirrorMaker for cross-region streams
Search         OpenSearch per region; reindex from CDC per region
Observability  Centralized Datadog / Honeycomb; per-region tagging
Secrets        AWS KMS multi-region keys; Vault with DR replication
Identity       Cognito user pool per region; user pinned at signup
```

Result:
- An EU customer's data lives in EU. ✅ GDPR.
- A US outage doesn't affect EU customers. ✅ Availability.
- A user who moves regions requires manual migration (rare). Tolerable.
- Cross-region data flows: global tables (slow async, OK), inventory (Spanner, paid latency tax for ACID).
- Cost: ~1.8× single-region. Justified by SLA and compliance.

This pattern is roughly Shopify, Atlassian, GitHub Enterprise Cloud, Slack — at varying levels of fidelity.

---

## 10. Failover — The Hardest Part

Failover is a special kind of system. It rarely runs; when it runs, stakes are highest; the engineers running it are stressed; the runbook is six months stale.

### Patterns

**Pilot light**: minimal capacity in secondary; scale up on failover. Cheap; slow.
**Warm standby**: meaningful capacity in secondary; scale up on failover. Costly; medium speed.
**Hot standby**: full capacity in secondary; flip traffic immediately. Most expensive; fastest.
**Active-active**: no failover per se — capacity already in both; route around the failed region.

### Mechanics

```
Detection:
  Health checks → DNS records updated → clients re-resolve

  Time to traffic shift = detection + propagation
  Typical: 30 s – 5 min depending on TTL behavior

Database:
  Promote standby to primary (or accept multi-region writes)
  Reverse replication direction (later)

State that must be drained:
  Inflight requests → must complete or fail cleanly
  Sticky sessions → users need to re-auth or session sync

Idempotency:
  Retries during failover hit different region → must be idempotent
```

### Testing
- Game days: scheduled failover drills.
- DNS shifts to standby region; verify traffic, data, monitoring.
- Reverse and verify.

If failover hasn't been tested in the last 90 days, assume it's broken.

---

## 11. Data Residency & Compliance

Regulatory drivers:
- **GDPR** — EU personal data must be stored / processed with appropriate safeguards.
- **Schrems II** — restrictions on transfers from EU to US; affects choice of cloud regions.
- **Russia, China, India** — data localization laws requiring data inside borders.
- **HIPAA / PCI / FedRAMP** — sector-specific controls; multi-region deployment must keep guarantees in each region.

Architectural implications:
- **Regional sharding** is the cleanest answer to localization requirements.
- **Cross-region replication of personal data** may need contractual safeguards or be prohibited outright.
- **Edge logic** for stripping PII before cross-region transfer.
- **Audit logs** of cross-region data movement.

The right move is to design the data model so localization is the default — once you've replicated PII to the wrong region, "fixing it" usually means a forensic data-deletion project.

---

## 12. Common Mistakes / Anti-Patterns

- **Going multi-region for availability you can't measure.** If your 30-day measured uptime is 99.7%, fix that before adding cross-region complexity.
- **"Failover never tested" runbook.** Statistically: it doesn't work.
- **Cross-region synchronous writes for the whole DB.** Every write pays 50–200 ms. Restrict to the small ACID-critical slice.
- **Cross-region chatty service calls.** A request that crosses regions 5 times costs hundreds of ms before any logic runs.
- **Replicating everything.** Most data doesn't need cross-region. Triage what's critical.
- **DNS-only failover with TTL hopes.** Resolvers ignore TTLs; clients cache; failover takes longer than the runbook promises.
- **No per-region observability.** Aggregate dashboards hide regional failures.
- **Active-active without conflict resolution plan.** Conflicts happen. Decide ahead of time what last-writer-wins or merge logic looks like.
- **Single global Redis / cache cluster.** Cross-region cache reads defeat the purpose; per-region caches with regional DB reads are simpler and faster.
- **Underestimating egress costs.** Modeled storage cost is half the bill; egress is the other half.
- **Multi-region as a 6-month project, not a multi-quarter program.** Every layer takes work; security review, runbooks, drills, education.
- **Forgetting auth.** Cross-region auth tokens, session state, and identity providers are minefields.
- **Geographic routing without checking user mobility.** Salesforce-like B2B: users travel. Treat region binding as a property of data, not session.

---

## 13. Decision Rule

```
Is your availability target ≥ 99.95% AND can you tolerate regional
downtime?
  Tolerate         → multi-AZ is enough.
  Cannot tolerate  → multi-region.

Is data residency required by law or contract?
  Yes → regional sharding from day one.

Are far-away users seeing unacceptable latency?
  Yes → read-local at minimum; consider regional sharding or
        active-active if write latency also matters.

Is the cost of regional downtime per hour < cost of multi-region
operation per year?
  Yes → don't go multi-region; multi-AZ + cross-region DR backups
        suffice.

When you do go multi-region:
  - Default to async replication, not synchronous.
  - Default to regional sharding for ownership of user data.
  - Use globally-consistent stores (Spanner, Cockroach) only for the
    thin slice that truly needs it.
  - Test failover every 90 days.
  - Model the cost per layer before approving.
```

---

## 14. Cheat Card

```
PURPOSE     Run in 2+ regions for availability, geographic latency,
            DR, and data residency. Pay for it in latency,
            consistency, complexity, and cost.

WHEN        Availability ≥ 99.95% required
            Far users need <50 ms latency
            Data residency forced
            Hour of regional downtime > year of multi-region cost

SHAPES
  1. Single + DR backup     cheapest; hours of RTO
  2. Active-passive         minutes of RTO; cold/warm standby
  3. Read-local, write-glob low read latency, slow writes from far
  4. Active-active          no failover; conflict-resolution required
  5. Regional sharding      per-user-region ownership; compliance-clean

LATENCY     Same region   1–3 ms RTT
            Same continent 30–80 ms RTT
            Cross-Atlantic 70–100 ms
            Cross-Pacific 100–200 ms

DB OPTIONS
  Async replicas         simple, lag, no cross-region writes
  Sync cross-region      slow (RTT/commit)
  Globally consistent    Spanner, Cockroach, Yugabyte (use sparingly)
  Active-active KV       DynamoDB Global Tables, Redis Active-Active
  Regional sharding      app-level + Vitess/Citus

LAYERS      Every layer needs a multi-region plan: DNS, LB, CDN, app,
            cache, DB, object, broker, secrets, identity, observability

COST        +30–100% over single-region; egress, idle capacity,
            duplicate storage, observability all add up

PITFALLS    Untested failover · cross-region synchronous writes ·
            chatty cross-region calls · single global cache ·
            replicating everything · DNS-only failover ·
            active-active without conflict plan · forgetting auth

RULE        Multi-AZ before multi-region. Most data doesn't need
            global ACID. Test failover quarterly or it doesn't
            work. Regional sharding is the cleanest answer for
            compliance and most B2B SaaS.
```

---

## 15. Resources

### Books
- *Designing Data-Intensive Applications* — Martin Kleppmann. Replication and partitioning chapters frame the multi-region problem.
- *Site Reliability Engineering* — Google. Chapters on global load balancing and reliability.
- *Database Reliability Engineering* — Campbell & Majors. Practical DR and multi-region for databases.

### Papers
- "Spanner: Google's Globally-Distributed Database" — Corbett et al., OSDI 2012.
- "CockroachDB: The Resilient Geo-Distributed SQL Database" — Taft et al., SIGMOD 2020.
- "Amazon Aurora: On Avoiding Distributed Consensus" — Verbitski et al., SIGMOD 2017.
- "DynamoDB Global Tables: Multi-region, Active-active Replication" — AWS technical docs.
- "Don't Settle for Eventual: Scalable Causal Consistency for Wide-Area Storage with COPS" — Lloyd et al.

### Articles
- "Building Global Services at Stripe" — Stripe engineering on multi-region rollout.
- "Multi-Region Cell-based Architecture" — AWS Architecture blog.
- "How Shopify Scaled to Black Friday" — multi-region + cellular architecture.
- "GitHub's Move to a Multi-Region Setup" — GitHub blog.
- "Multi-Region with Aurora Global Database" — AWS docs.
- "Cross-Region Replication with DynamoDB Global Tables" — AWS docs.
- "Multi-Region Postgres" — Crunchy Data and pg blogs.

### Videos
- AWS re:Invent — "Multi-Region Architectures" track (annual).
- Google Cloud Next — Spanner and global services deep-dives.
- ByteByteGo — "Multi-Region Architecture" overview.
- CockroachDB / Yugabyte / Spanner conference talks.
- "GitOps for Multi-Region Kubernetes" — KubeCon talks.

### Tools
- **Route 53 / Cloud DNS / NS1** — geo / latency routing.
- **AWS Global Accelerator / GCP Global LB / Azure Front Door** — anycast.
- **CloudFront / Fastly / Cloudflare / Akamai** — CDN + edge logic.
- **Spanner / CockroachDB / YugabyteDB / FoundationDB** — globally consistent DBs.
- **DynamoDB Global Tables** — multi-region active-active KV.
- **Vitess / Citus** — sharded SQL, often combined with regional sharding.
- **MirrorMaker 2 / Confluent Replicator** — Kafka cross-region.
- **HashiCorp Vault** — multi-region secrets with DR replication.
- **AWS Aurora Global Database** — managed cross-region Postgres/MySQL with <1 s lag.

### Adjacent reading
- [CAP Theorem](../08-distributed-systems/cap-theorem.md)
- [PACELC Theorem](../08-distributed-systems/pacelc.md)
- [Consistency Models](../08-distributed-systems/consistency-models.md)
- [Replication (Master-Slave, Master-Master, Multi-Region)](../04-databases/replication.md)
- [Database Sharding Strategies](./sharding-strategies.md)
- [Failover & Disaster Recovery](../11-reliability/failover-dr.md)
- [Blast Radius & Cell-Based Architecture](../11-reliability/cell-architecture.md)
- [CDN — Content Delivery Networks](../05-caching/cdn.md)
- [Chaos Engineering](../11-reliability/chaos-engineering.md)

---

*Previous:* [← Backpressure](./backpressure.md)  |  *Up:* [README ↑](../README.md)
