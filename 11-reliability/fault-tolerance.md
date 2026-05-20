# Fault Tolerance Patterns

> **TL;DR** — **Fault tolerance** is the property of a system that keeps working — usually in a degraded but useful state — when one or more of its components fail. Every distributed system fails constantly: disks die, networks partition, dependencies time out, processes get killed, code has bugs. The choice isn't "do we tolerate failure" — it's "*which* failures do we tolerate, and *how*." The toolkit is a small, well-understood set of patterns: **redundancy**, **isolation**, **timeouts**, **retries with backoff**, **circuit breakers**, **bulkheads**, **graceful degradation**, **failover**, **idempotency**, **idempotent retries**, **failure detectors**, and **defaults / fallbacks**. They compose. They are mostly inexpensive. They are the difference between "we lost a region today" and "nobody noticed we lost a region today." This page is the map; subsequent pages dive into each pattern.

---

## 1. The Premise

Three propositions you must accept before fault tolerance makes sense:

1. **Every component will fail.** Disks, NICs, racks, regions, DNS, the deployment that went out two minutes ago. Failure is not exceptional; it is *normal*.
2. **Failures correlate.** A bad config touches every node. A region's network drops every connection to it. One slow downstream slows everything that depends on it.
3. **Recovery is part of behavior.** A "working" system is one that not only handles requests in steady state, but recovers cleanly from arbitrary subsets of failures.

If you accept these three, the engineering question shifts from "how do we prevent failure?" (you can't) to "how do we limit blast radius, detect quickly, recover gracefully, and shed load when overwhelmed?"

---

## 2. The Pattern Map

```
                     PREVENT
                       (hard)
                          ▲
                          │
                          │
       ┌──────────────────┴──────────────────┐
       │             FAULT TOLERANCE         │
       │                                     │
       │   detect → contain → degrade →      │
       │       recover → learn               │
       │                                     │
       └──────────────────┬──────────────────┘
                          │
                          ▼
                    SURVIVE & SERVE
```

Six classes of pattern fit underneath. Each has its own dedicated page; this one is the overview and the connective tissue.

| Class | Patterns | Goal |
|---|---|---|
| Redundancy | Replication, replicas, EC, multi-AZ | Survive a failed copy |
| Isolation | Bulkheads, cells, partitions, per-tenant | Limit blast radius |
| Time & retry | Timeouts, retries, exponential backoff, jitter | Survive transient failures |
| Detection | Health checks, heartbeats, circuit breakers, failure detectors | Notice fast |
| Degradation | Graceful degradation, fallbacks, defaults | Stay useful when broken |
| Recovery | Failover, replay, idempotent retries, self-healing | Return to healthy state |

The map is small. The discipline is in applying it consistently and knowing the failure modes of each piece.

---

## 3. Redundancy — Survive a Lost Copy

The oldest fault-tolerance idea: keep more than one copy of everything important.

```
Hardware            RAID, ECC RAM, hot-spare disks, dual-NIC, dual-PSU
Compute             Multiple instances behind a load balancer
Storage             3× replication / Reed-Solomon erasure coding
Database            Primary + replicas (sync or async)
Network             Multiple ISPs, multi-VPC, multi-region
DNS                 Multiple authoritative providers
Identity            Replicated IdPs, fallback auth
People              On-call rotations, runbooks, multiple people who know X
```

The math is straightforward: if each copy fails independently with probability *p*, the chance of *all N* failing is *p^N*. For *p* = 1% and *N* = 3, that's 1 in a million. Real systems aren't independent (correlated failures dominate), but the principle holds — more redundancy buys headroom, at a cost.

Trade-offs:
- **Storage cost** for replication or EC (see [Erasure Coding vs Replication →](../09-storage/erasure-coding.md)).
- **Latency cost** for sync replication (writes wait for replicas).
- **Consistency cost** for async replication (replicas trail; failover may lose data).
- **Operational cost** of running N of everything.

Redundancy without **placement awareness** is theater. Three replicas on one rack is one rack-failure away from total loss. Always spread across failure domains.

---

## 4. Isolation — Limit the Blast Radius

A single bad query, bad config, or runaway client can cause cascading damage. **Isolation** ensures that bad behavior is contained.

```
Thread isolation           per-downstream thread pool (bulkhead)
Connection isolation       per-downstream connection pool
Process isolation          containers, VMs, separate fleets
Network isolation          VPC, security groups, service mesh policies
Tenant isolation           per-tenant resources or quotas
Failure-domain isolation   AZs, regions, cells

Goal: a failure of class X affects at most a known, bounded subset.
```

The strongest form is **cell-based architecture** — partition the entire stack (compute, DB, cache, queues) into independent cells, each serving a subset of users. A failure inside cell-3 affects only cell-3's users. See [Cell-Based Architecture →](./cell-architecture.md).

Weaker but cheaper forms — thread pool bulkheads, connection pools, quotas, rate limits — protect within a process or service. See [Bulkhead →](./bulkhead.md).

---

## 5. Time & Retry — Survive Transients

Most network failures are transient: a packet drops, a TCP connection resets, a slow downstream recovers in seconds. The toolkit:

```
TIMEOUTS                cap how long a call can hang
                        every network call must have one
RETRIES                 try again, bounded
EXPONENTIAL BACKOFF     wait longer between retries
JITTER                  randomize to break synchronized retries
DEADLINES               end-to-end budget across hops
```

These four combined transform "everything died for 4 seconds" into "users saw a 200 ms blip." See [Retry, Timeout, Backoff →](./retry-timeout-backoff.md).

The single biggest fault-tolerance bug in real systems is **retries without backoff or jitter**. A slow downstream + naïve retries amplifies traffic 3–10×, finishing the kill the downstream started.

---

## 6. Detection — Know Fast

You can't recover from what you don't see. Detection mechanisms:

```
HEALTH CHECKS               LB pings /healthz; pulls unhealthy backends
HEARTBEATS                  active-active "I'm alive" pings between peers
GOSSIP                      decentralized membership and failure spread
FAILURE DETECTORS           φ-accumulating (Cassandra) and SWIM
CIRCUIT BREAKERS            track downstream error/latency; trip open
SUPERVISION                 process tree restarts failed children (Erlang/OTP)
WATCHDOGS                   external service notices the absence of heartbeats
```

The hardest detection problem in distributed systems is distinguishing **dead** from **slow** — a node that hasn't responded in 5 seconds may be down, restarting, or just temporarily overloaded. Bad detectors cause spurious failovers; conservative detectors miss real failures. See [Failover →](./failover-dr.md) and [Health Checks & Heartbeats →](../13-observability/health-checks.md).

**Circuit breakers** are detection that trades correctness for latency: rather than waiting for slow calls to time out, trip the breaker and fail fast. See [Circuit Breaker →](./circuit-breaker.md).

---

## 7. Degradation — Stay Useful When Broken

When a dependency fails, you have three honest options:

```
FAIL    return an error to the user
DEGRADE return a reduced or stale answer
HIDE    silently substitute a default
```

Failing is *honest* and sometimes correct (you can't process a payment without the payment gateway). **Degrading** is preferable when partial functionality is still valuable:

- Search engine down → show cached homepage instead of search results.
- Recommendation service down → show popular items instead of personalized recs.
- Profile photo service slow → show initials.
- Sidebar widget timing out → render page without it.

Hiding is dangerous — silent defaults mask problems and confuse users. Use hiding only when the silent answer is *correct*, not just "an answer."

See [Graceful Degradation →](./graceful-degradation.md).

---

## 8. Recovery — Get Back to Healthy

Tolerating a failure is not enough — you have to recover. Recovery patterns:

```
FAILOVER                promote standby; redirect traffic
REPLAY                  reprocess a WAL/event log up to a target
IDEMPOTENT RETRIES      safe to repeat without duplication
QUEUE / OUTBOX          buffer work; drain when downstream is back
SELF-HEALING            controllers (K8s, ASGs) replace bad nodes
COMPENSATION (SAGA)     undo partial work via compensating actions
SNAPSHOTS + WAL         restore + replay to recover state
```

The most operationally important recovery property is **idempotency**: an operation can be applied multiple times with the same effect as one application. Without idempotency, every retry is a potential double-charge, double-send, or double-write. See [Idempotent Operations & Retries →](./idempotency-retries.md).

---

## 9. The Pattern Catalog

A reference table — each row has its own page or section:

| Pattern | Solves | Cost | Trap |
|---|---|---|---|
| Replication / replicas | Single-instance failure | 2–3× storage / compute | Replica lag; failover risk |
| Erasure coding | Storage failure at scale | CPU + repair amplification | Slow random reads |
| Multi-AZ deployment | One-AZ failure | Cross-AZ latency, egress cost | Misconfigured placement |
| Multi-region | Regional outage | Hard, expensive | Untested failover |
| Timeouts | Hung calls | Spurious failures if too tight | None set → hang forever |
| Retries + backoff + jitter | Transient errors | Retry amplification | Without backoff: cascades |
| Circuit breaker | Slow downstream | False trips under spiky load | Threshold tuning |
| Bulkhead | Resource exhaustion by one downstream | More resources to provision | Under-provisioning per pool |
| Cell-based architecture | Total-service blast radius | Operational complexity | Hard to evolve schema across cells |
| Rate limiting | Abuse / overload | Rejecting legitimate users | Set too low |
| Backpressure | Producer outrunning consumer | Latency / rejection at front | Unbounded queues |
| Load shedding | Overload at the edge | Some users get 503 | Adaptive thresholds needed |
| Graceful degradation | Dependency failure | Engineering each fallback path | Hidden defaults misleading |
| Idempotency | Duplicate effects from retries | Storage for idempotency keys | Forgetting to apply it |
| Sagas | Distributed transactions | Application complexity | Compensations not commutative |
| Outbox pattern | Atomic publish | Schema + worker | Forgotten cleanup |
| Health checks + LB | Pull bad nodes from rotation | Probe traffic + tuning | "Healthy" check that's too shallow |
| Heartbeats / gossip | Membership / detection | Background chatter | Slow detection vs false positives |
| Failover / failback | Regional or component disasters | Standby capacity, drills | Untested → doesn't work |
| Chaos engineering | Find latent failures | Engineering time, brave culture | Without observability, unsafe |

The takeaway: there are perhaps two dozen patterns total. Most production systems use 8–12 of them. Pick the ones that apply, apply them everywhere they apply, and verify with chaos drills.

---

## 10. Composition — Patterns Layer

Patterns compose along the call path. A request from a user to your service to a database hits multiple patterns in sequence:

```
Client → CDN → LB → API gateway → Service → DB → Reply
   ↑          ↑           ↑           ↑       ↑
   retries    rate limit  bulkhead    circuit retries
   timeouts   load shed   timeout     breaker  idempotent
                          deadlines           failover
                                              connection pool
                                              transaction
```

Each hop:
- Has a **timeout** scoped tight enough that the user's deadline holds.
- Has a **circuit breaker** around the downstream.
- Has a **retry policy** with bounded count + backoff + jitter.
- Lives in an **isolation boundary** (bulkhead, thread pool, connection pool).
- **Degrades** if the downstream is unreachable.
- **Sheds load** if it's overwhelmed.

Skip any of those and a single slow downstream takes down everything that calls it. Apply all of them consistently and a single slow downstream is a logged warning.

---

## 11. Failures You Must Plan For

A non-exhaustive checklist. Read each row and confirm your system has a defined behavior:

- A single instance crashes mid-request.
- A node hangs (CPU pinned, no response, not crashed).
- A network partition splits the cluster.
- The DB primary fails over; some recent writes are lost.
- A bad deploy goes live everywhere.
- The cache fills up; latency spikes.
- A downstream service is slow but not down (p99 → 30 s).
- A downstream service is down completely.
- A client retry storm hits at 10× normal load.
- A noisy neighbor consumes 80% of shared capacity.
- An AZ loses power.
- A region loses connectivity.
- DNS resolution fails / returns wrong values.
- TLS certificate expires.
- A disk fills up.
- A subscription / quota silently exhausts.
- An IAM credential rotates and breaks a downstream auth.
- A scheduled job overlaps itself.
- Time skews on a node (NTP misbehaves).
- A hot key forms in cache / DB.
- An incident response itself causes a second incident.

A mature system has tested behavior for every row above. Most systems have tested behavior for fewer than half. The gap is where outages live.

---

## 12. Production Discipline

Patterns alone don't make a system fault tolerant. The discipline:

### Runbooks
Every alert ties to a documented response. Without runbooks, on-call relies on tribal knowledge.

### Game days
Quarterly disaster drills — kill a region, fail a DB, drop a dependency. Surface latent failures while engineers are calm and rested. See [Chaos Engineering →](./chaos-engineering.md).

### Postmortems
Blameless, written, shared, with action items tracked. Treat outages as data, not punishment.

### Defense in depth
No single mechanism is enough. Layer redundancy + isolation + degradation. Assume each layer will fail individually.

### Observability
You can't recover from what you can't see. Logs, metrics, traces, alerts on burn rate. See [Three Pillars →](../13-observability/three-pillars.md).

### Idempotency by default
Make safe-to-retry the default discipline, not a feature.

### Error budget policy
Tie reliability work to product velocity. Spend the budget; don't drift past it. See [SLA/SLO →](./sla-slo-sli.md).

### Production readiness reviews
Before a new service ships, walk through this list with someone who'll be on-call for it.

---

## 13. Common Mistakes / Anti-Patterns

- **Optimizing for prevention over recovery.** Bug-free code doesn't exist; design for graceful failure.
- **No timeouts on remote calls.** A hung call holds resources forever; eventually the whole service hangs.
- **Retries without backoff.** Retry storms turn a small failure into total collapse.
- **Health checks that don't check anything real.** `/healthz` that returns 200 OK without verifying dependencies hides everything.
- **Replication factor 2.** A single failure leaves you with no redundancy.
- **All replicas in one AZ / rack.** Correlated failure kills all copies.
- **Untested failover.** It probably doesn't work.
- **No isolation between tenants.** One bad customer takes everyone down.
- **No idempotency on retried operations.** Duplicates, double-charges, double-sends.
- **Hiding errors with silent defaults.** Users see nonsense; ops sees no alerts.
- **Caching the unreliable answer.** Stale "OK" hides a downstream outage.
- **Cascading retries down a deep call chain.** Each hop multiplies the load; the bottom drowns.
- **Single point of trust** (one DNS provider, one cert authority, one cloud account, one credential).
- **Reliance on humans during incidents.** Humans are slow and stressed. Automate the obvious.
- **Treating reliability as someone else's job.** Engineering teams who don't own their SLOs build systems that don't meet them.
- **One big monolithic blast radius.** A bad deploy or bad row takes the whole product down.
- **Chaos testing only when it's "safe."** It's never safe; do it carefully but consistently.

---

## 14. Decision Rule

```
For every remote call:
   - Timeout set?            Yes.
   - Retries bounded?         Yes (or zero).
   - Backoff + jitter?        Yes.
   - Idempotent?              Yes, or do not retry.
   - Circuit breaker?         If downstream is slow-fail-prone.
   - Bulkhead / pool size?    Yes.
   - Fallback / degradation?  If user value > 0 without it.

For every resource:
   - Redundant across failure domains?
   - Health-checked?
   - Quota'd / rate-limited?
   - Backed by a tested failover?

For every error path:
   - Logged with context?
   - Metric'd / alertable?
   - Has a runbook?

For the system as a whole:
   - SLO defined and tracked?
   - Error-budget policy enforced?
   - Chaos drill in the last 90 days?
   - Postmortems with action items closed?
```

---

## 15. Cheat Card

```
PURPOSE     Keep the system useful when components fail. Plan for
            failure as the normal case, not the exception.

PATTERN MAP
  Redundancy        replicas, EC, multi-AZ, multi-region
  Isolation         bulkheads, cells, pools, quotas
  Time & retry      timeouts, retries, backoff, jitter, deadlines
  Detection         health checks, heartbeats, breakers, supervisors
  Degradation       fallbacks, defaults, partial answers
  Recovery          failover, replay, idempotency, sagas

LAYER       Apply patterns at EVERY hop. A request from user to DB
            traverses 5–10 places where a pattern matters.

FAILURES TO PLAN FOR
  Crash · hang · partition · failover · bad deploy · slow downstream ·
  down downstream · retry storm · noisy neighbor · AZ loss · region
  loss · DNS · TLS expiry · disk full · IAM rotation · time skew ·
  hot key · self-inflicted incident-response error

DISCIPLINE  Runbooks · game days · postmortems · defense in depth ·
            observability · idempotency by default · error budget ·
            production readiness review

PITFALLS    No timeouts · retries without backoff · health checks
            that lie · RF=2 · all replicas in one AZ · untested
            failover · no tenant isolation · cached-OK masking outages ·
            cascading retries · single point of trust

RULE        You can't prevent failure. You can pick which failures
            you tolerate and how. Defense in depth, tested. The
            difference between mature systems and fragile ones is
            not the absence of failure — it's the absence of
            *surprise*.
```

---

## 16. Resources

### Books
- *Release It!* — Michael Nygard. The canonical book on stability patterns; every chapter is a fault tolerance pattern.
- *Designing Data-Intensive Applications* — Martin Kleppmann. Chapter 8 on the trouble with distributed systems.
- *Site Reliability Engineering* — Google. Operational discipline as fault tolerance.
- *Patterns of Distributed Systems* — Unmesh Joshi. Pattern-level treatment of consensus, replication, etc.
- *Reactive Design Patterns* — Roland Kuhn. Backpressure, isolation, supervision in reactive systems.
- *Building Secure & Reliable Systems* — Google. Reliability + security; modern overview.

### Articles
- "Hints for Computer System Design" — Butler Lampson. Old, deep, still relevant.
- "Why Do Computers Stop?" — Jim Gray, 1985. The original treatment.
- "Stability Patterns" — Michael Nygard's chapter, summarized many places online.
- "The Tail at Scale" — Dean & Barroso, ACM 2013. Tail latency as a reliability problem.
- "How Complex Systems Fail" — Richard Cook. 18 short observations every operator should read.
- Honeycomb / Stripe / Netflix / AWS engineering blogs — operational pattern essays.
- AWS Well-Architected Reliability Pillar — practical checklist.

### Videos
- "Stop Rate Limiting! Capacity Management Done Right" — Jon Moore.
- "How Complex Systems Fail" — Richard Cook, conference talks based on his paper.
- SREcon talks — many on operational patterns.
- ByteByteGo — "Fault Tolerance" overview.

### Tools
- **resilience4j / Polly / Hystrix (archived)** — circuit breaker, retry, bulkhead libraries.
- **Istio / Linkerd / Envoy** — service-mesh-level patterns.
- **Chaos Mesh / Chaos Monkey / Gremlin / LitmusChaos** — chaos engineering.
- **Kubernetes** — supervision, restarts, pod disruption budgets, readiness probes.

### Adjacent reading
- [SLA, SLO, SLI, Error Budgets](./sla-slo-sli.md)
- [Circuit Breaker Pattern →](./circuit-breaker.md)
- [Retry, Timeout, and Exponential Backoff →](./retry-timeout-backoff.md)
- [Bulkhead Pattern →](./bulkhead.md)
- [Graceful Degradation →](./graceful-degradation.md)
- [Failover & Disaster Recovery →](./failover-dr.md)
- [Chaos Engineering →](./chaos-engineering.md)
- [Blast Radius & Cell-Based Architecture →](./cell-architecture.md)
- [Idempotent Operations & Retries →](./idempotency-retries.md)
- [Backpressure](../10-scalability/backpressure.md)
- [Multi-Region](../10-scalability/multi-region.md)

---

*Previous:* [← SLA, SLO, SLI, Error Budgets](./sla-slo-sli.md)  |  *Next:* [Circuit Breaker Pattern →](./circuit-breaker.md)
