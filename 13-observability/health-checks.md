# Health Checks & Heartbeats

> **TL;DR** — A **health check** is an endpoint or signal a service exposes to say "I'm alive and ready to serve." Used correctly, they drive load balancer pool membership, container restarts, autoscaler decisions, and deployment gating. Used naively, they cause cascading failures: dependency outage → health checks fail → instances removed → remaining instances overwhelmed → everything collapses. The right design distinguishes **liveness** (am I broken — restart me), **readiness** (am I ready to serve traffic — drain me), and **startup** (give me time to warm up). **Heartbeats** are the inverse pattern — services or workers periodically signal they're alive to a central watchdog; absence of heartbeat triggers action. Get health checks right and most reliability problems get easier. Get them wrong and they amplify outages.

---

## 1. The Three Kinds of Health Check

Kubernetes formalized the distinction, and it's the right model whether or not you use K8s.

| Probe | Question | Action on fail |
| --- | --- | --- |
| **Liveness** | Is the process broken? | **Restart** the process / pod |
| **Readiness** | Should I receive traffic? | **Remove** from load balancer pool |
| **Startup** | Has it finished initializing? | Hold off other probes until pass |

A failing liveness probe is a **terminal** verdict — kill and restart. A failing readiness probe is **temporary** — pause sending traffic until it recovers. Conflating them is the most common health-check bug in production.

---

## 2. Liveness — "Restart Me"

A liveness probe should fail **only if restarting the process would help**.

Good liveness signals:
- Process is deadlocked (a watchdog goroutine/thread can't tick).
- Out of memory, leaking, GC-thrashing.
- Stuck in an unrecoverable state.

Bad liveness signals (= will cause cascading restarts):
- "Can't reach the database" — restarting won't fix the database.
- "External API is timing out" — restarting won't fix the external API.
- "Garbage collector ran long once."

The bar for failing liveness should be high. **When in doubt, don't fail it.** Restart loops are worse than degraded service.

### Implementation pattern

```python
@app.get("/healthz/live")
def liveness():
    # Did our internal "still alive" thread tick recently?
    if time.time() - last_internal_tick > 30:
        return "deadlock", 500
    return "ok", 200
```

The check verifies the *process's internal heartbeat*, not external dependencies. A truly broken process can't tick its own heartbeat.

---

## 3. Readiness — "Send Me Traffic"

A readiness probe should fail when the instance **can't usefully serve a request right now**.

Good readiness signals:
- DB connection pool is exhausted (won't be able to serve).
- Critical dependency unreachable (orders service down → can't checkout).
- Still loading data on startup.
- Draining for shutdown.
- Memory pressure approaching critical.

Bad readiness signals (= will cause cascading failure):
- "Some single backend is slow" — if all your replicas check the same backend and it's slow, they all fail readiness, all get removed, no instances left.
- "Any error happened in the last 30s."

Readiness must be **safe to depool**. If failing it would take all instances out of the pool, it shouldn't fail.

### Implementation pattern

```python
@app.get("/healthz/ready")
def readiness():
    if shutting_down:
        return "draining", 503
    if db_pool.in_use / db_pool.max > 0.95:
        return "saturated", 503
    if not critical_cache_loaded:
        return "warming", 503
    return "ok", 200
```

### Readiness during deploys

Readiness is what makes **zero-downtime deploys** work:

```
1. New pod starts.
2. Liveness passes once process is up.
3. Readiness fails until cache warms / pool ready.
4. LB doesn't route to it yet.
5. Readiness passes → LB adds it.
6. Old pod gets SIGTERM → flips readiness to fail (drain).
7. LB drains old pod's traffic over a grace period.
8. Old pod exits.
```

Without readiness, you serve traffic to a half-initialized process. With it, deploys are invisible.

---

## 4. Startup — "Give Me Time"

Some processes need minutes to be ready (load 50 GB index, warm cache, build connection pool, run migrations). With only liveness, the process gets killed before it can finish starting.

K8s' `startupProbe`:
- Runs **before** liveness and readiness.
- Fails are tolerated for up to N seconds.
- Once it passes, normal liveness/readiness take over.

Without K8s, the pattern is `max_init_seconds`: don't run liveness in the first N seconds.

---

## 5. What a Health Endpoint Should Look Like

A minimal HTTP health endpoint:

```
GET /healthz/live      → 200 OK / 500
GET /healthz/ready     → 200 OK / 503
GET /healthz/startup   → 200 OK / 503  (optional)
```

Return codes only — no body needed for the LB. For human consumption, a `/health/detail` endpoint can list each subsystem:

```json
{
  "status": "degraded",
  "checks": {
    "process":     {"status": "ok"},
    "db":          {"status": "ok",  "latency_ms": 3},
    "cache":       {"status": "warn", "msg": "85% memory"},
    "kafka":       {"status": "ok"},
    "downstream_x":{"status": "ok"}
  },
  "version":  "2026.05.18-3a9c1",
  "uptime_s": 9871
}
```

This is useful for dashboards and debugging, **not** for load balancers. LBs see only the binary version.

### Don't expose `/health` publicly

It leaks operational data and is a DDoS target. Restrict to internal/private networks or require auth, or use a separate internal port.

---

## 6. Cascading Failures: The Fatal Pitfall

The most dangerous health-check anti-pattern:

> All instances check the **same shared backend** in readiness. The backend gets slow. All instances fail readiness. LB removes all of them. No instances serve traffic. Total outage.

Real example: a microservice's readiness probe checks "can I query the user-DB?" The DB has a brief blip. All 50 instances of the service fail readiness simultaneously. The LB removes all 50. The DB recovers in 5 seconds, but the service is unreachable for the full LB sync interval.

### Mitigations

- **Check only local state in readiness.** External dependencies are checked **per request** with circuit breakers, not in readiness.
- **Tolerate downstream errors.** If a non-critical dep is down, log it, return what you can.
- **Don't depool the whole fleet.** Even if checks legitimately fail, the LB should keep some minimum quorum live (or fail-open).
- **Stagger readiness signals.** Random jitter avoids all-fail-at-once when many instances see the same blip.

The rule: **readiness checks dependencies whose failure you cannot mitigate**. Everything else is handled per-request in code.

---

## 7. Heartbeats — The Inverse

A heartbeat is the active version: the service or job **periodically sends** "I'm alive" to a central watchdog. Absence → action.

Use cases:
- **Workers / consumers** (Kafka consumer groups, Celery workers) — broker considers them dead after no heartbeat.
- **Leader election** — leader sends heartbeats; followers trigger election on missing heartbeats.
- **Distributed locks** — lock holder must keep heartbeating to extend lease.
- **Cron / scheduled jobs** — emit a "last ran at" heartbeat; alert if absent.
- **External monitors** (Better Stack Heartbeat, Healthchecks.io, Dead Man's Snitch) — useful for cron, batch, and "we expect this every day."

Heartbeat parameters:
- **Interval** — how often (1–30s typical).
- **Timeout / TTL** — how long after the last heartbeat we consider dead (~2-3× interval).
- **Action** — depool, restart, alert, fail-over.

### Failure modes

- **Heartbeat from a deadlocked process** — the heartbeating goroutine still runs while the main loop is stuck. Heartbeat from the *work*, not from a background timer.
- **Heartbeat too aggressive** — network blips falsely trigger fail-over. Tune TTL.
- **Heartbeat too lax** — long dead-time before action.

For consumers/workers, modern frameworks (Kafka, BullMQ, Sidekiq, Temporal) handle this for you. For bespoke systems, build the heartbeat to gate on actual progress.

---

## 8. Synthetic / External Monitoring

Health checks the service runs **on itself** can't detect "I'm running fine, but DNS doesn't resolve me" or "the load balancer thinks I'm healthy but routes traffic into a void."

Complementary signal: **external synthetic probes**:

- Probe from an external location (multiple regions).
- Hit a real production endpoint with a typical request.
- Measure success, latency, content.

Tools: Pingdom, Datadog Synthetics, AWS CloudWatch Synthetics, GCP Uptime Checks, StatusCake, Better Stack Uptime.

Synthetic checks catch:
- DNS / TLS / CDN problems invisible to internal probes.
- Routing issues.
- Regional brownouts.

Don't overdo it — synthetic monitors that hit expensive endpoints constantly become a load source.

---

## 9. Health Checks and Load Balancers

Each LB has health-check options:

| LB | Health-check setup |
| --- | --- |
| **AWS ALB / NLB** | Target group health check: HTTP/HTTPS/TCP path, interval, threshold |
| **GCP Load Balancer** | Backend service health check |
| **Nginx, HAProxy, Envoy** | Per-backend HTTP/TCP probes |
| **Kubernetes Service / Ingress** | Pod readiness drives Endpoints |
| **Service Mesh (Istio, Linkerd)** | Service-mesh sidecar probes |

Parameters that matter:
- **Interval** — how often (5–30 s typical).
- **Threshold** — how many consecutive failures before marking unhealthy (2–3 typical).
- **Timeout** — how long to wait per check (1–5 s).
- **Path** — exactly the readiness endpoint.

Misconfigurations:
- Threshold = 1 → flaky. One slow check depools the instance.
- Threshold = 10 → slow to react. Real failures keep getting traffic.
- Interval too short → probe traffic dominates real traffic on small fleets.

---

## 10. Health Checks in Service Mesh / Sidecars

Sidecars (Envoy in Istio/Linkerd) handle health checks for the app:

- **Outlier detection** — sidecar tracks success rate to each upstream; ejects bad endpoints.
- **Active checks** — sidecar pings endpoints periodically.
- **Passive checks** — sidecar uses real request outcomes.

The advantage: language-agnostic, configured centrally, decoupled from app code.

See [Service Mesh →](../03-apis/service-mesh.md).

---

## 11. Health Checks for Databases and Other Stateful Services

Databases need health checks too — but with extra care:

- **Postgres / MySQL** — `SELECT 1` is the canonical liveness; `pg_isready` is built-in.
- **Redis** — `PING`.
- **Kafka brokers** — JMX metrics + ZooKeeper/KRaft state.
- **Replicas** — health = "primary + replica lag below threshold". A replica that's way behind is healthy but useless.

For HA setups (Patroni, Vitess, AWS RDS Multi-AZ), the orchestrator's health checks drive failover. Tune thresholds to balance "fail over too eagerly" (false alarms, split-brain risk) vs "fail over too slowly" (extended outage). See [Failover & Disaster Recovery →](../11-reliability/failover-dr.md).

---

## 12. Graceful Shutdown

A complete picture pairs readiness with **graceful shutdown**:

```
SIGTERM received
  → set readiness = false (so LB drains)
  → wait for in-flight requests to complete (with timeout)
  → close DB pools, message broker, etc.
  → exit
```

In Kubernetes:
- `preStop` hook calls `/healthz/shutdown` or sleeps.
- `terminationGracePeriodSeconds` sized to fit your longest request.
- App listens for SIGTERM and starts draining.

Without graceful shutdown, deploys drop in-flight requests on the floor. With it, deploys are invisible to users.

---

## 13. Worked Example — A Solid Health Setup

A Go service, K8s deployment:

```go
var (
    ready       atomic.Bool
    shuttingDown atomic.Bool
    lastTick    atomic.Int64
)

// Background heartbeat from main work loop
go func() {
    for {
        lastTick.Store(time.Now().Unix())
        time.Sleep(10 * time.Second)
    }
}()

http.HandleFunc("/healthz/live", func(w http.ResponseWriter, r *http.Request) {
    if time.Now().Unix() - lastTick.Load() > 30 {
        http.Error(w, "deadlock", 500)
        return
    }
    w.WriteHeader(200)
})

http.HandleFunc("/healthz/ready", func(w http.ResponseWriter, r *http.Request) {
    if shuttingDown.Load() || !ready.Load() {
        http.Error(w, "not ready", 503)
        return
    }
    if dbPool.InUse() > 0.95 * dbPool.Max() {
        http.Error(w, "saturated", 503)
        return
    }
    w.WriteHeader(200)
})

// SIGTERM handling
sigCh := make(chan os.Signal, 1)
signal.Notify(sigCh, syscall.SIGTERM)
go func() {
    <-sigCh
    shuttingDown.Store(true) // start draining
    time.Sleep(20 * time.Second) // allow LB to depool
    server.Shutdown(ctx)
}()
```

```yaml
# K8s pod spec
livenessProbe:
  httpGet: { path: /healthz/live, port: 8080 }
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3
readinessProbe:
  httpGet: { path: /healthz/ready, port: 8080 }
  periodSeconds: 5
  failureThreshold: 2
startupProbe:
  httpGet: { path: /healthz/ready, port: 8080 }
  failureThreshold: 30
  periodSeconds: 5
terminationGracePeriodSeconds: 30
```

This handles: deadlock detection (liveness), saturation depool (readiness), startup grace, and graceful drain.

---

## 14. Common Mistakes / Anti-Patterns

- **Conflating liveness and readiness.** Liveness restarts; readiness depools. Different actions, different signals.
- **Checking external dependencies in liveness.** Dependency blip → restart loop → outage amplification.
- **Checking shared backends in readiness across all instances.** Backend blip → entire fleet depooled.
- **Single shared health endpoint for everything** — `/health` does both probes, plus reports detailed status to anyone. Pick one purpose per endpoint.
- **Heavy health-check work.** Running a join query on every probe = self-DDoS.
- **No startup probe (or no grace).** Slow-starting services killed before they're up.
- **Health checks unauthenticated and public.** Information leak + DDoS target.
- **Health checks bypassing real code path** — probe passes, real requests fail. Probe should at least touch the same path.
- **No graceful shutdown.** Deploys drop in-flight requests.
- **Threshold = 1.** Flaky probes flap the LB.
- **Threshold = 10 with 30s interval.** 5-minute MTTR before a bad instance is removed.
- **Heartbeat from a background timer rather than the working thread.** Deadlocked main loop still heartbeats.
- **Probes more frequent than the work itself.** On a small fleet, probes dominate traffic.
- **Forgetting health checks on dependencies in Service Mesh sidecars.** Outlier detection unused.
- **No synthetic external monitoring.** Missing failures at the LB / DNS / TLS layer.

---

## 15. Cheat Card

```
THREE PROBES — different actions
  LIVENESS   "broken?"  fail → RESTART      |  check INTERNAL state only
  READINESS  "ready?"   fail → DEPOOL       |  fail only if SAFE to depool
  STARTUP    "warming?" tolerate fails N s  |  hold off other probes

RULE: don't fail liveness on external deps.  Don't fail readiness on shared deps.

ENDPOINT
  /healthz/live    /healthz/ready    /healthz/startup
  return 200 / 5xx only.  detail page separate, auth-required.

HEARTBEATS
  inverse — service pings watchdog; absence = action
  use cases: workers, leader election, locks, cron monitors
  emit from the WORK, not a side timer

GRACEFUL SHUTDOWN
  SIGTERM → readiness=false → drain (wait) → close pools → exit
  K8s: preStop + terminationGracePeriodSeconds

LB CONFIG
  interval 5–30s · timeout 1–5s · threshold 2–3
  not 1 (flaky), not 10 (slow)

SYNTHETIC MONITORS
  external probes from multiple regions
  catch DNS/TLS/CDN/routing — invisible to internal probes

CASCADING FAILURE WATCHOUT
  if 100% of instances check the same dep, all depool together
  check local state in readiness; per-request circuit-break externals

RULE: liveness = "kill me if true", readiness = "skip me if true",
      and never let a probe take out the whole fleet.
```

---

## 16. Resources

### Documentation
- **Kubernetes Probes** — <https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/>
- **AWS ALB Health Checks** — <https://docs.aws.amazon.com/elasticloadbalancing/>
- **Envoy Health Checking** — <https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/health_checking>
- **GCP Health Checks** — <https://cloud.google.com/load-balancing/docs/health-checks>

### Articles
- "Health checks and graceful shutdown in K8s" — Cindy Sridharan / various.
- "How readiness probes can cause cascading failures" — Datadog / classic blog posts.
- "Heartbeats and timeouts in distributed systems" — Marc Brooker (AWS).
- "The Tail at Scale" — Dean & Barroso (Google) — related: how slow nodes propagate.
- AWS Builders' Library — "Implementing health checks": <https://aws.amazon.com/builders-library/>

### Books
- *Site Reliability Engineering* — Google.
- *Release It!* — Michael Nygard. Stability patterns, including health/heartbeat.

### Videos
- "Probes in Kubernetes" — Liz Rice / KubeCon.
- ByteByteGo — "Health checks and load balancing".

### Tools
- **External monitoring:** Better Stack Uptime, Datadog Synthetics, Pingdom, UptimeRobot, StatusCake.
- **Cron monitors:** Healthchecks.io, Dead Man's Snitch, Better Stack Heartbeat.
- **Service mesh:** Istio / Linkerd / Envoy outlier detection.
- **Watchdog libraries:** kubernetes/client-go, sidekiq lifecycle, Temporal SDK.

### Adjacent reading
- [Load Balancer Fundamentals →](../06-load-balancing/load-balancer-basics.md)
- [Health Checks & Failover →](../06-load-balancing/health-checks.md)
- [Circuit Breaker Pattern →](../11-reliability/circuit-breaker.md)
- [Failover & Disaster Recovery →](../11-reliability/failover-dr.md)
- [Graceful Degradation →](../11-reliability/graceful-degradation.md)
- [Logging Best Practices →](./logging.md)
- [Metrics & Time-Series →](./metrics.md)
- [Alerting & On-Call →](./alerting.md)

---

*Previous:* [← Centralized Log Aggregation](./log-aggregation.md)  |  *Up:* [README ↑](../README.md)
