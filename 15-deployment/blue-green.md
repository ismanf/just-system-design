# Blue-Green Deployment

> **TL;DR** — **Blue-green deployment** runs two identical production environments — "blue" (currently live) and "green" (the new version) — and swaps traffic between them atomically by flipping a load balancer, DNS, or router rule. The win is **instant rollback** (flip back) and **zero overlap risk** (only one version takes traffic at a time). The cost is **2× infrastructure** during cutover and **harder shared-state migrations** (databases don't get a second copy). Blue-green shines when you need a sharp, no-mixed-versions transition — payments, financial reporting, or risky binary changes. For most modern services, **canary** or **rolling** is cheaper and safer in practice. Treat blue-green as one tool in the deployment toolbox, not the default.

---

## 1. The big picture

```
         ┌─────────────────────┐
         │   Load Balancer /   │
         │   Router / DNS      │
         └──────────┬──────────┘
                    │   100%
        ┌───────────┼────────────┐
        ▼                        ▼
  ┌──────────┐             ┌──────────┐
  │  BLUE    │             │  GREEN   │
  │  v1.4.2  │  ◄────────► │  v1.5.0  │
  │  LIVE    │             │  WARM    │
  └──────────┘             └──────────┘
        │                        │
        └─────────┬──────────────┘
                  ▼
            ┌──────────┐
            │ Database │
            │ (shared) │
            └──────────┘
```

Two complete environments, identical except for the version. Traffic goes to one at a time. To deploy:

1. Deploy the new version to the idle environment.
2. Warm it up — run smoke tests, send synthetic traffic, watch metrics.
3. Atomically switch the LB to point at the new environment.
4. Keep the old environment alive long enough to roll back if needed.
5. After validation, the old environment becomes the next "idle" target.

**Atomic** here is doing a lot of work. We'll come back to it.

---

## 2. Why this pattern exists

The motivation: **rolling deployments mix versions for the duration of the rollout**. If v2 has a bug, every request during that window has a chance of hitting it. Half your users see v1, half see v2, until rollout completes. Rolling back is just another rolling deployment in reverse — slow.

Blue-green's promise:

- **All-or-nothing traffic switch.** No mixed-version window.
- **Instant rollback.** Flip the LB back; old environment is still running.
- **Clean validation.** Smoke-test green with real production traffic shape before any user sees it (mirror traffic, synthetic load).
- **Predictable timing.** Cutover is one operation, not a multi-minute ramp.

This was the dominant pattern in the early 2010s — Netflix and Amazon both popularized variants. Today it has been displaced by canary and rolling for most cases, but it still wins for specific scenarios listed in §6.

---

## 3. The atomic switch — how it really works

"Flip the LB" sounds simple. The mechanics matter.

### LB-level switch (the good way)

```
Before:   LB → target_group_blue   (instances v1.4.2)
After:    LB → target_group_green  (instances v1.5.0)
```

On AWS ALB, you swap the listener's default target group. On nginx, you reload with a new `upstream` block. On Envoy, a control-plane push updates the cluster assignment. The change is effectively atomic at the LB — the LB serves the old config until the new one is loaded and active, then routes all new connections to the new target. **In-flight requests on the old environment complete normally.**

### DNS-level switch (the slow way)

Change the A/CNAME record. Wait for TTL. Some clients honor TTL, some don't (browsers, mobile apps, NAT boxes cache aggressively). DNS switches are **not atomic** — they can take minutes to hours to fully propagate.

DNS blue-green is acceptable only when you control all clients or when slow drain is OK.

### Router/feature-flag switch

Set a flag, route based on the flag. Implemented inside the application or via an edge proxy. More flexible (per-region, per-user) but adds code paths.

### Connection draining

Old environment keeps serving in-flight connections while the LB drains it. Long-lived connections (WebSockets, SSE, HTTP/2 with keepalive) need explicit handling — graceful shutdown, server-initiated close, or sticky session expiry.

---

## 4. The database problem — the hard part

Two app environments. **One database.** That asymmetry is the source of every interesting blue-green issue.

### The rule

**Schema changes must be backward and forward compatible across the two versions you're swapping.** If v1.4.2 (blue) and v1.5.0 (green) need different schemas, you cannot blue-green directly. You must do a **multi-phase migration**:

1. **Expand** — add new columns/tables/indexes. Both versions still work.
2. **Migrate** — backfill data into new shape. Old version still writes old shape.
3. **Cutover** — deploy new version that writes new shape. Old version still readable.
4. **Contract** — once new version is stable, drop old columns/tables.

This is the **expand-contract pattern** (a.k.a. parallel-change). It's how Stripe, GitHub, and most large engineering orgs do schema migrations safely. See [Database Migrations at Scale →](../04-databases/migrations.md).

### What doesn't work

- **Migration in the same release as a breaking schema change.** Old version sees broken schema.
- **Two databases, one for blue, one for green.** Now you have a sync problem worse than the version problem.
- **"Just take a brief outage."** Sometimes acceptable, but defeats the point of blue-green.

### Stateful side effects

- **Caches** — Redis, Memcached. Green needs warm caches or you'll see a thundering herd at cutover. Pre-warm or accept a brief degradation.
- **Background jobs / queue consumers** — both blue and green may be consuming from the same queue. Decide whether to drain blue first, run both, or partition by job type.
- **WebSockets / long polls** — clients on blue stay on blue until they reconnect. Force-reconnect or live with a long tail.
- **Sticky sessions** — invalidate or migrate. Otherwise users on blue stay on blue across the cutover.
- **Background timers / cron** — make sure only one environment runs them, or you'll fire jobs twice.

---

## 5. Worked example — a service on AWS

A web service with an ALB, two Auto Scaling Groups (or two ECS task definitions), and an RDS Postgres.

```
                    ┌─────────────┐
                    │     ALB     │
                    │  listener   │
                    └──────┬──────┘
                           │ default target group
              ┌────────────┴────────────┐
              │                         │
        ┌─────▼──────┐           ┌──────▼─────┐
        │   TG-Blue  │           │  TG-Green  │
        │ (5 EC2 v1) │           │ (5 EC2 v2) │
        └────────────┘           └────────────┘
                  │                  │
                  └────────┬─────────┘
                           ▼
                    ┌────────────┐
                    │  RDS PG    │
                    └────────────┘
```

Workflow:

1. Run schema migration in **expand** mode (add columns, indexes) — both versions still happy.
2. Deploy v2 to TG-Green (idle).
3. Health checks on TG-Green green up. Smoke tests pass.
4. Optionally mirror live traffic to TG-Green (shadow) — see how it behaves on real load.
5. Change the ALB listener's default action from TG-Blue → TG-Green. New connections go green.
6. Drain TG-Blue (ALB stops sending new requests; existing ones complete).
7. Watch error rate, latency, business metrics for 15–60 minutes.
8. If healthy, deploy next version to TG-Blue (now idle). Repeat.
9. If unhealthy, change listener back to TG-Blue. Rollback in seconds.
10. After full stabilization, run **contract** migration to drop old columns.

The whole thing is reproducible, scripted, and atomic at the LB layer.

---

## 6. When blue-green wins

- **High-risk binary changes** — anything where mixed-version interaction is unsafe. Major framework upgrades, language runtime swaps, encryption format changes.
- **Compliance or audit needs a clean cutover** — financial reporting deadlines, regulated systems where "the version is X starting at time T."
- **Long-running deploys where rolling is too slow** — if your fleet is 1000 instances and rolling takes hours, blue-green's atomic flip is faster.
- **You want instant, atomic rollback** — the old version is *running*, not just *deployable*. Rollback is one LB rule.
- **Behavioral A/B at the request boundary** — although canary handles this better.
- **Database-light services** — services with little or no schema involvement get blue-green's full benefit without the expand-contract pain.

## 7. When blue-green loses

- **Cost-sensitive environments** — you're paying for 2× instances during the cutover window, sometimes for hours.
- **Frequent deploys (multiple per day)** — the overhead of warm-up and validation amortizes poorly.
- **Heavy stateful systems** — caches, in-memory state, long-running jobs — make a clean cutover messy.
- **Microservices with many services deploying independently** — coordinating blue/green across 50 services is unrealistic. Canary or rolling fits this shape better.
- **You want to limit blast radius rather than commit fully** — canary's gradual ramp gives you observation time at small percentages. Blue-green is 0% or 100%.

---

## 8. Blue-green vs canary vs rolling — at a glance

| Property | Blue-Green | Canary | Rolling |
|---|---|---|---|
| Mixed-version window | None at cutover | Yes (intentional) | Yes (during rollout) |
| Rollback speed | Instant (flip) | Fast (route 0%) | Slow (re-deploy) |
| Infrastructure cost | 2× during cutover | 1.05–1.2× | 1× + maxSurge |
| Validates on production traffic | After cutover | Yes (small slice) | Yes (during rollout) |
| Database-friendly | Hardest | Medium | Medium |
| Best for | Atomic, risky cutovers | Gradual ramps, observation | Default for most apps |

See [Canary Deployment →](./canary.md), [Rolling Deployment →](./rolling.md).

---

## 9. Variants

### Symmetric blue-green
The textbook version above — two identical environments.

### Hot-warm
Blue at full size, green at small idle size. Scale green up before cutover. Saves cost during normal operation. Loses some instant-rollback guarantees if blue isn't kept warm enough.

### Two-stage blue-green (with canary blend)
Cut over a small slice first (1%, 10%) via weighted target groups, then flip the rest. This is essentially canary-into-blue-green and gets the best of both at the cost of complexity.

### Red-black (Netflix's term)
Exactly blue-green. Red = old, black = new. The name predates the blue-green term in some places.

### Blue-green at the cluster level (Kubernetes)
Two Deployments (`api-blue` and `api-green`) behind a Service whose selector switches. Argo Rollouts and Flagger automate this with metric analysis. See [Kubernetes →](./kubernetes.md).

---

## 10. Operational checklist

```
PRE-DEPLOY
[ ] Schema migration in expand mode is done and verified
[ ] Green environment provisioned, same size as blue
[ ] Image/build artifact pinned by digest
[ ] Health checks pass on green
[ ] Smoke tests pass on green via internal endpoint
[ ] Shadow / mirror traffic checked for errors
[ ] Rollback runbook tested in a dry run

CUTOVER
[ ] Notify on-call, post in deploys channel
[ ] Flip the LB
[ ] Confirm traffic shift in dashboards within 60s
[ ] Watch p99 latency, error rate, business KPIs
[ ] Keep blue running, healthy, ready for instant rollback

POST-DEPLOY (15–60 min watch)
[ ] No anomalies in error budget
[ ] Queue depth, cache hit rate, DB connections normal
[ ] Async / background workers behaving
[ ] Long-lived connections migrated or expiring cleanly

WIND-DOWN
[ ] Decommission blue (after retention window)
[ ] Run contract migration (drop old schema bits)
[ ] Update artifacts: blue is now the idle pool

ROLLBACK (if needed)
[ ] Flip LB back to blue (single command, scripted)
[ ] Communicate the rollback
[ ] File incident if needed
[ ] Do NOT run contract migration until next attempt succeeds
```

---

## 11. Common Mistakes / Anti-Patterns

- **Coupling schema migrations to the cutover.** The whole point is atomicity; mixing in DDL ruins rollback. Always expand-contract.
- **Rolling back code but not data.** If green wrote new-shape data and blue can't read it, your "instant rollback" causes worse damage than the original bug.
- **Not warming green's caches.** Cutover lands on cold caches → DB load spike → false-positive failure → unnecessary rollback.
- **Forgetting background jobs/cron.** Both environments fire cron jobs → duplicates. Or neither does (only blue had cron enabled).
- **DNS-based cutover with no plan for stuck clients.** Mobile apps with aggressive caches can stay on the old IP for hours.
- **Sticky sessions across the cutover.** Some users stay on blue indefinitely. Force-invalidate or migrate sessions.
- **Treating blue-green as the only deploy strategy.** It's expensive overkill for small, low-risk changes.
- **Letting blue/green drift.** Over time, the "idle" environment accumulates differences. Treat them as immutable, redeployed every cycle.
- **Skipping the validation window.** Blue-green's value depends on watching green under real load before retiring blue.
- **No load on green before cutover.** Green has zero traffic, then suddenly 100%. Discovery: it had a memory leak that only shows under load.
- **Long-lived WebSocket / SSE connections not gracefully migrated.** Blue keeps half your real-time users for days.

---

## 12. Cheat Card

```
PURPOSE   Atomic, all-or-nothing version switch with instant rollback,
          by running two identical environments behind one router.

MECHANIC
  Blue  = current live
  Green = new version, warmed up alongside
  Flip LB / target group → 100% green
  Keep blue alive as the rollback target
  Rollback = flip back, in seconds

SCHEMA RULE
  Expand first → cutover → contract later
  Both versions must be compatible with the same DB

WHEN TO USE
  Risky, high-stakes binary changes
  When mixed-version traffic is unsafe
  When you need atomic cutover at a known time
  When instant rollback is the must-have property

WHEN NOT TO USE
  Cost-sensitive, frequent deploys
  Heavy in-memory / cache state
  Big stateful migrations on a single DB
  Microservices fleet deploying independently

PITFALLS
  Coupling schema change to cutover
  Cold green caches → false-positive failure
  Sticky sessions / WebSockets left on blue
  Duplicate background jobs / cron on both
  Drift between blue and green over time
  No validation window before retiring blue

RULE   Make the cutover boring: two identical environments, an
       LB flip, and a database that doesn't care which one is live.
```

---

## 13. Resources

### Documentation
- **AWS** — Blue/Green deployments on ECS/CodeDeploy: <https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-type-bluegreen.html>
- **Argo Rollouts** — Blue-Green strategy: <https://argoproj.github.io/argo-rollouts/features/bluegreen/>
- **Spinnaker** — Red/Black pipelines: <https://spinnaker.io>

### Articles
- "BlueGreenDeployment" — Martin Fowler: <https://martinfowler.com/bliki/BlueGreenDeployment.html>
- "Parallel Change" — Martin Fowler: <https://martinfowler.com/bliki/ParallelChange.html>
- "Online migrations at scale" — Brandur Leach (Stripe): <https://stripe.com/blog/online-migrations>
- "How we deploy" — GitHub engineering blog (variants on blue-green).
- Netflix Tech Blog — early posts on Asgard and red/black.

### Videos
- *Continuous Deployment at scale* — Jez Humble.
- ByteByteGo — "Blue-Green Deployment Explained."

### Tools
- **Argo Rollouts** — Kubernetes-native blue-green and canary.
- **Spinnaker** — Multi-cloud deployment pipelines.
- **AWS CodeDeploy** — managed blue-green for EC2/ECS/Lambda.
- **Flagger** — service-mesh-driven progressive delivery.

### Adjacent reading
- [Canary Deployment →](./canary.md)
- [Rolling Deployment →](./rolling.md)
- [Feature Flags & Dark Launches →](./feature-flags.md)
- [Immutable Infrastructure →](./immutable-infra.md)
- [CI/CD Pipelines →](./cicd.md)
- [Database Migrations at Scale →](../04-databases/migrations.md)
- [Failover & Disaster Recovery →](../11-reliability/failover-dr.md)

---

*Previous:* [← Container Orchestration (Kubernetes)](./kubernetes.md)  |  *Next:* [Canary Deployment →](./canary.md)
