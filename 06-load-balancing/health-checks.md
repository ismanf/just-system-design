# Health Checks & Failover

> **TL;DR** — A **health check** is the LB's way of finding out whether a backend should receive traffic. **Active checks** are periodic probes the LB initiates (`/health`, TCP connect, gRPC health protocol). **Passive checks** observe live traffic and eject misbehaving backends. The signal must be sharp enough to catch real problems quickly but stable enough to avoid flapping. A bad health check is worse than none — too sensitive and it ejects healthy backends during transient blips, too lenient and dead pods keep serving errors. Good production setups combine both: lightweight active probes for liveness, passive outlier-ejection for graceful degradation. **Failover** is what happens when health checks fail: a backend is ejected, an LB instance dies and is replaced, a region is drained. Plan failover paths explicitly — the assumed-implicit ones never work in real incidents.

---

## 1. The Job of a Health Check

The LB has one critical question: **is this backend safe to send a request to?** Wrong answers in either direction hurt:

- **False positive (backend healthy when it isn't)** — users get errors.
- **False negative (backend ejected when fine)** — capacity drops, p99 spikes.

The right signal balances:
- **Liveness** — is the process running?
- **Readiness** — can it serve requests *right now*? (DB connected, cache warm, dependencies up?)
- **Performance** — is it responding fast enough?
- **Capacity** — does it have headroom?

A naive "is port 80 open" check answers only liveness. Real health checks answer readiness and often performance.

---

## 2. Active Health Checks

The LB sends a periodic probe and decides health from the response.

### 2.1 TCP connect
Open a TCP connection. If it succeeds, the backend is alive.

```
LB → SYN → backend
LB ← SYN-ACK
LB → ACK
LB closes
```

- **Pros**: cheap, works for any TCP service.
- **Cons**: tells you the OS is responding. Says nothing about the application.

Use for: TCP/UDP services where you don't have a smarter check. Postgres, Redis, message brokers.

### 2.2 HTTP `/health` (or `/healthz`)
GET a known endpoint. Healthy = 2xx. Unhealthy = anything else.

```
LB → GET /health
backend → 200 OK
```

- **Pros**: application-aware. The endpoint can check dependencies (DB, cache, downstream APIs).
- **Cons**: needs implementation. Cost depends on what the endpoint actually checks.

Best practices for the endpoint:
- **Cheap by default** — return quickly when healthy.
- **Distinguish liveness from readiness**:
  - `/healthz/live` → just "process running". Used by Kubernetes liveness probes.
  - `/healthz/ready` → "I can serve traffic right now" (DB connected, warm caches). Used by readiness probes and LB checks.
- **Don't 200-OK while broken** — common bug. The endpoint returns 200 even when the DB is down.
- **Don't put expensive checks in a health endpoint that's hit 50× per second.**
- **Don't cascade failure** — if your `/health` checks the DB and the DB is sometimes slow, the LB ejects the pod and traffic concentrates on remaining pods, which queue up on the DB, which fails further. Decouple.

### 2.3 gRPC health protocol
A standardized gRPC service for health checking. Returns `SERVING` / `NOT_SERVING` / `UNKNOWN`.

```proto
service Health {
    rpc Check(HealthCheckRequest) returns (HealthCheckResponse);
    rpc Watch(HealthCheckRequest) returns (stream HealthCheckResponse);
}
```

Envoy, ALB, Istio, all support gRPC health checks. Use this for gRPC services rather than HTTP /health-on-a-different-port.

### 2.4 Application-specific checks
- **Database**: send `SELECT 1` and time it.
- **Redis**: `PING` and check `<= some_ms` latency.
- **Kafka**: connect and list metadata.
- **Custom**: anything that gives you a yes/no signal.

### 2.5 Active check tuning
Three knobs every LB exposes:
- **Interval** — how often to probe. 1–10 seconds typical.
- **Timeout** — max wait per probe. 1–5 seconds typical.
- **Threshold** — how many consecutive results to flip state. `unhealthy_threshold=3`, `healthy_threshold=2` is common.

Trade-offs:
- Short interval + low threshold → fast detection, more probe overhead, more flapping risk.
- Long interval + high threshold → slow detection, less overhead, more stable.

A reasonable default: probe every 5s, 2s timeout, 3 unhealthy / 2 healthy threshold.

For thousands of backends, probe traffic itself can be substantial. Probe budgets matter at scale (Google publishes papers on this — they share probes across many LB instances).

---

## 3. Passive Health Checks (Outlier Ejection)

Watch live traffic. If a backend returns errors or slow responses, eject it temporarily.

```
backend B returns 5xx 5 times in a row → eject for 30s
backend B returns errors again after re-enable → eject longer (exponential)
```

Envoy's "outlier detection" is the canonical reference. Configuration:

```yaml
outlier_detection:
  consecutive_5xx: 5
  consecutive_gateway_failure: 5
  base_ejection_time: 30s
  max_ejection_time: 300s
  max_ejection_percent: 50    # never eject more than half
  interval: 10s
  enforcing_consecutive_5xx: 100
```

The win: passive checks see what users see. Active probes hit `/health`; real traffic hits real code paths. A backend with a bug in one endpoint will be ejected by passive checks but not by `/health`.

Most modern LBs do both. Envoy, Linkerd, Istio, NGINX Plus, HAProxy (`observe` keyword), AWS ALB (with target failover) — all support it.

---

## 4. The Healthy Backend Pool: Steady State

```
        all backends
       ┌──────────────┐
   ┌───┤ pool A       │  ← receives traffic
   │   ├──────────────┤
   │   │ pool B       │  ← ejected; not in rotation
   │   │ (in penalty) │
   └───┤              │
       ├──────────────┤
       │ pool A (cont)│
       └──────────────┘

probes run on both; ejected backends get re-tested
```

Most LBs let you set `max_ejection_percent` to bound how much capacity can be ejected at once. Without that, a bug that returns 5xx everywhere ejects every backend, leaving zero capacity. Bound it.

---

## 5. Health Check Anti-Patterns

### 5.1 The "always 200" endpoint
Hardcoded `return 200`. Process up = OK. Useless for readiness.

### 5.2 The "check everything" endpoint
`/health` checks DB, Redis, Kafka, downstream services, the weather. Any dependency hiccup ejects every backend simultaneously. Cascade failure.

**Fix**: depend on *critical, locally-required* dependencies only. If DB is down and your service can't function, fine — eject. If a non-critical downstream is slow, return 200 and let the request-level retries handle it.

### 5.3 Same endpoint as application traffic
`/health` is just another endpoint. Health probes count against rate limits, share thread pool with real requests, get queued behind slow requests. Under load, the LB thinks the service is unhealthy when it's just busy.

**Fix**: dedicate a thread pool or HTTP port for health checks. Java services often use Jetty's "admin" connector. Go services run a separate `http.ServeMux` on a different port.

### 5.4 Health check piggybacks on production code
Failed health check = `Sentry: HTTP 500 on /health`. Logs flooded.

**Fix**: dedicated logging strategy for health endpoints. Or: don't 500 on health — return 503 and log at info level.

### 5.5 No probe budget
Hundreds of LBs probing thousands of backends at 1-second interval. The probe traffic dwarfs the real traffic.

**Fix**: longer intervals, shared probes (one LB probes and gossips results), or use passive checks more.

### 5.6 No `max_ejection_percent`
A bug causes 5xx everywhere; LB ejects every backend; pool is empty.

**Fix**: cap ejection (50% is typical), fail open after that.

### 5.7 Probe from outside the cluster
LB lives outside the cluster, probes pods. NAT/firewall makes the probe behave differently from real traffic.

**Fix**: probe along the same path real traffic takes. Same network, same proxy chain.

---

## 6. Failover: When Checks Fail

Health checks are inputs. **Failover** is what the LB does about them.

### 6.1 Backend failover
Eject the bad backend; route to others. The mechanical case. Bounded by `max_ejection_percent`.

### 6.2 LB instance failover
The LB itself fails. Options:
- **HA pair** (active/standby): VRRP / keepalived. Standby takes the VIP. Sub-second.
- **Active/active behind ECMP**: many LBs share the VIP; a router hashes. One LB dies, the router reroutes flows.
- **Anycast IP**: same IP advertised from many sites; BGP routes to nearest healthy.
- **Cloud-managed**: ALB/NLB/GCLB handle this transparently. Underneath: anycast + ECMP.

### 6.3 AZ failover
An entire AZ becomes unreachable. The LB should detect (zone-level health checks) and stop sending traffic. AWS ALB supports zone-aware routing; cross-zone load balancing is on/off-able.

The default: **enable cross-zone load balancing** for HTTP traffic. Skews acceptable; resilience necessary. For high-throughput intra-AZ traffic (e.g., service mesh inside a single AZ), zone-affinity might be preferred to save bandwidth costs — but always with cross-zone fallback.

### 6.4 Region failover
A whole region goes down. DNS-based GSLB shifts traffic to another region. Multi-region is its own discipline. See [GSLB →](./gslb.md) and [Multi-Region →](../10-scalability/multi-region.md).

### 6.5 Origin failover (CDN)
A CDN typically has a primary origin and one or more failover origins. On primary 5xx or timeout, traffic flips to backup. Cloudflare's "load balancing pools", Fastly's "shielding + failover", AWS CloudFront's "origin groups" all support this.

---

## 7. Draining: The Inverse of Health Check

When a backend is being deprovisioned (deploy, scale-down, maintenance), you want it to stop receiving new traffic but finish in-flight requests cleanly.

```
1. mark backend as "draining" → LB stops routing new requests to it
2. wait for in-flight requests to complete (or timeout)
3. shut down backend
```

This is **connection draining** (AWS term), also called **graceful shutdown** or **lame-duck mode**.

Implementations:
- **Kubernetes**: `terminationGracePeriodSeconds` + pod readiness probe set to fail before SIGTERM.
- **AWS Target Group**: `deregistration_delay`.
- **HAProxy**: `disabled` state via runtime socket.
- **Envoy**: drain manager.

Without drain, every deploy drops in-flight requests. Users see 502s on deploy. **Always configure drain.**

---

## 8. Detecting Slow vs Dead

A dead backend (port closed, process gone) is obvious. A slow backend is harder — and more dangerous, because it ties up requests.

### Active latency check
Probe with timeout less than your SLO. If probe takes > 500 ms when SLO is 100 ms, the backend is unhealthy.

### Outlier-based latency ejection
Envoy supports `success_rate` and `failure_percentage` based outlier detection on top of consecutive errors. You can also configure `slow_callee_failure_threshold` (in some configs) to eject backends whose tail latency exceeds peer median by N×.

### Application-side circuit breakers
Each client maintains a circuit breaker per upstream. Too many slow / failed calls → open the circuit → fail fast for a cool-down period. The LB doesn't see this directly, but it's the partner pattern to passive ejection.

See [Circuit Breaker →](../11-reliability/circuit-breaker.md).

---

## 9. Real-World Examples

### Kubernetes
Three probe types:
- **livenessProbe** — if it fails, the pod is restarted.
- **readinessProbe** — if it fails, the pod is removed from Service endpoints but kept running.
- **startupProbe** — like readiness but with grace period for slow-starting apps.

LBs reading the Service endpoints will route only to "ready" pods. Combined with PreStop hook and `terminationGracePeriodSeconds`, you get drain semantics.

### AWS ALB
- HTTP/HTTPS active checks.
- Targets are marked `unhealthy` after `UnhealthyThresholdCount` failures.
- Connection draining via target group setting.
- Cross-zone load balancing for resilience.
- Optional Outlier Detection (new feature, target-failure-based).

### Envoy / Istio
- Active health checks (HTTP, TCP, gRPC).
- Outlier detection (passive).
- Locality-weighted load balancing.
- Drain manager for graceful shutdown.

### Cloudflare Load Balancing
- Active checks every N seconds.
- Origin groups with priority / failover.
- Latency-based steering ("Geo Steering", "Dynamic Steering").

---

## 10. Worked Example: Health-Checking a Web Service

A Go service has Postgres, Redis, and a downstream auth service. K8s deployment, ALB in front.

### `/health/live`
- Returns 200 always (or 503 if process is shutting down).
- ALB and K8s liveness probe both hit this.

### `/health/ready`
- Returns 200 if:
  - DB pool has at least one healthy connection.
  - Redis is reachable (cached check, runs every 5s).
  - Process isn't in shutdown.
- Returns 503 otherwise.
- K8s readiness probe hits this; ALB also configured to hit this.

### Auth-service downstream
- **Not** included in `/health/ready`. Even if down, the request-level circuit breaker handles it gracefully. Cascading the auth-service failure into ejecting all pods is worse than degraded auth.

### Shutdown
- Pod gets SIGTERM.
- `preStop` hook flips `/health/ready` to 503 and sleeps 5s.
- ALB / K8s remove pod from rotation.
- After 5s, app cleanly closes its DB pool and exits.
- `terminationGracePeriodSeconds=30` gives in-flight requests time.

Result: zero-downtime rolling deploy. Probe storms during DB hiccup don't melt the pool.

---

## 11. Common Mistakes

- **Health endpoint that 200-OKs while broken.** Find this in your codebase today.
- **Health endpoint that checks every dependency.** Cascading ejection.
- **No readiness/liveness split.** Liveness restarts pods that just needed a moment.
- **No drain on deploy.** Every deploy drops requests.
- **No `max_ejection_percent`.** Bug causes empty pool.
- **Active checks only, no passive.** Real-traffic problems hide.
- **Probes from outside the data path.** Probes succeed; real requests fail.
- **Same threshold for liveness and readiness.** Liveness needs to be loose (don't restart for transient hiccups); readiness needs to be tight (don't route to flaky pod).
- **Aggressive intervals at huge scale.** Probe traffic eats CPU.
- **Health check secret-token misuse.** Anyone can hit `/health` and see internal status. Lock it down if it leaks.

---

## 12. Cheat Card

```
ACTIVE CHECKS    TCP, HTTP, gRPC health protocol
                 interval 5s, timeout 2s, threshold 2/3

PASSIVE CHECKS   observe live traffic; eject on consecutive 5xx
                 base_ejection_time 30s; cap at 50% ejected

LIVENESS         "process alive" → restart if fails
READINESS        "can serve now" → remove from LB if fails
                 always have both

HEALTH ENDPOINT  cheap, local, async; distinguish live vs ready
                 do NOT check every downstream

DRAIN            stop new traffic → finish in-flight → shut down
                 K8s preStop + terminationGracePeriodSeconds

FAILOVER         backend (eject) · LB (HA/anycast) · AZ · region

PITFALLS         always-200, cascade-failure /health, no drain,
                 no max_ejection_percent, probes off-path

RULE             A health check that lies is worse than none.
                 A health check that flaps is almost as bad.
```

---

## 13. Resources

### Books
- *Site Reliability Engineering* — Google SRE Book, chapters on load balancing and managing critical state.
- *Release It!* — Michael Nygard (circuit breakers, bulkheads, drain).

### Documentation
- **Envoy outlier detection**: <https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/outlier>
- **Envoy health checking**: <https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/health_checking>
- **AWS ALB health checks**: <https://docs.aws.amazon.com/elasticloadbalancing/latest/application/target-group-health-checks.html>
- **Kubernetes probes**: <https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/>
- **gRPC health checking protocol**: <https://github.com/grpc/grpc/blob/master/doc/health-checking.md>

### Articles
- "Avoiding cascading failure with health checks" — Stripe Engineering Blog.
- "Probing at Google scale" — various GCP / SRE talks.
- "Liveness vs Readiness in Kubernetes" — Google Cloud blog.
- "Health checks at Netflix" — Netflix Engineering.

### Videos
- ByteByteGo — "Health Checks & Failover".
- Hussein Nasser — "Why your health checks are wrong".

### Tools
- **Kubernetes probes** built in.
- **Consul** — service discovery with health checks.
- **HAProxy / NGINX / Envoy** — health-check configs.
- **kubectl describe pod** — shows probe status.

### Adjacent reading
- [Load Balancer Fundamentals →](./load-balancer-basics.md)
- [Algorithms →](./algorithms.md)
- [Sticky Sessions →](./sticky-sessions.md)
- [GSLB →](./gslb.md)
- [Circuit Breaker →](../11-reliability/circuit-breaker.md)
- [Retry, Timeout, Backoff →](../11-reliability/retry-timeout-backoff.md)
- [Failover & Disaster Recovery →](../11-reliability/failover-dr.md)

---

*Previous:* [← Load Balancing Algorithms](./algorithms.md)  |  *Next:* [Sticky Sessions →](./sticky-sessions.md)
