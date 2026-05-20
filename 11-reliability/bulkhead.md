# Bulkhead Pattern

> **TL;DR** — A **bulkhead** isolates resources (threads, connections, memory, queues) into compartments so the failure of one compartment doesn't sink the whole ship. Named after ship hull bulkheads, the pattern bounds **concurrent work per dependency, per tenant, per request class** — typically with a fixed-size thread pool or semaphore around each downstream call. When the downstream is slow, only its dedicated bulkhead fills; calls to other downstreams continue normally. Without bulkheads, one slow dependency exhausts the global thread pool and every endpoint hangs. The pattern is cheap, well-understood, and almost always missing in early-stage services. Production-grade systems use bulkheads at multiple layers: thread/semaphore bulkheads inside processes, connection-pool bulkheads to databases, separate fleets for different request classes, and full **cell-based architectures** at the macro scale. This page covers the in-process / per-dependency form; the larger scale is in [Cell-Based Architecture →](./cell-architecture.md).

---

## 1. The Idea — Why "Bulkhead"

A ship's hull is divided by **transverse bulkheads** — vertical walls that prevent a single hole from flooding the whole vessel. The Titanic sank because the iceberg ripped open enough adjacent compartments that the water spilled over the tops of the bulkheads, but the principle was sound: **partition the failure domain**.

In software:

```
Without bulkheads                    With bulkheads
─────────────────                    ──────────────

  shared pool of 200 threads          ┌─────────┐
  ┌─────────────────────┐             │ pool A  │ 50 threads → dep A
  │                     │             ├─────────┤
  │ all calls compete   │             │ pool B  │ 50 threads → dep B
  │                     │             ├─────────┤
  │ slow dep A → pool   │             │ pool C  │ 50 threads → dep C
  │ exhausted →         │             ├─────────┤
  │ B, C also fail      │             │ pool D  │ 50 threads → dep D
  │                     │             └─────────┘
  └─────────────────────┘
                                       slow dep A fills pool A.
  one bad dependency →                 B, C, D unaffected.
  whole service down.
```

Bulkheads convert "everything is broken" into "this one dependency is broken." The user sees graceful degradation for the affected feature instead of total outage.

---

## 2. The Failure Mode It Prevents

Without bulkheads, the classic cascade:

```
1. Dependency D becomes slow (p99: 200 ms → 30 s)
2. Threads calling D start hanging on those slow calls
3. Caller's thread pool fills with threads stuck on D
4. New requests (even for unrelated work) can't get a thread
5. Caller appears completely down
6. Caller's health checks fail
7. Load balancer pulls caller from rotation
8. Load shifts to remaining instances
9. Remaining instances repeat 1–7
10. CASCADE — full outage
```

The fundamental issue: **a single resource (thread pool) is shared across all dependencies**. Any slow dependency starves all others.

Bulkheads solve this by **giving each dependency its own bounded resource budget**. When dep D is slow, only the D bulkhead fills. The rest of the service hums along.

---

## 3. The Two Flavors

Bulkheads come in two main implementations:

### Thread-pool bulkhead
Each dependency gets its own thread pool of fixed size.

```
┌───────────────────────────────────────────────┐
│ caller process                                │
│                                               │
│  HTTP request                                 │
│       │                                       │
│       ▼                                       │
│  ┌──────────────────────────────────────┐    │
│  │  per-dependency thread pools          │    │
│  │  ┌──────┐  ┌──────┐  ┌──────┐         │    │
│  │  │ pool │  │ pool │  │ pool │  ...    │    │
│  │  │  A   │  │  B   │  │  C   │         │    │
│  │  └──────┘  └──────┘  └──────┘         │    │
│  └──────────────────────────────────────┘    │
│                                               │
└───────────────────────────────────────────────┘
```

- **Pro**: full isolation — even synchronous blocking calls are contained.
- **Con**: more threads → more memory, more context switching.
- **When**: synchronous code paths with blocking downstream calls.
- **Used by**: Hystrix (classic implementation), some resilience4j configurations.

### Semaphore (counting) bulkhead
Each dependency gets a counter (e.g., 50) of allowed concurrent calls. Calls beyond the limit fail fast (or queue briefly).

```
sem = Semaphore(50)
def call_dep_A():
    if not sem.try_acquire():
        raise BulkheadFullException
    try:
        return downstream_call()
    finally:
        sem.release()
```

- **Pro**: lightweight, no thread overhead, fits naturally with async/non-blocking code.
- **Con**: blocking calls inside the semaphore still hold the caller's thread.
- **When**: async / event-loop / non-blocking code; or tight memory budgets.
- **Used by**: resilience4j default bulkhead, most modern Go/Rust/JS implementations.

For a JVM service with synchronous code, **thread-pool bulkheads** are the canonical Hystrix-style choice. For modern async runtimes (Netty, Tokio, Node.js, asyncio), **semaphore bulkheads** are the right tool.

---

## 4. Where to Place Bulkheads

A bulkhead can go around any resource that can become exhausted under failure. The common placements:

### Per remote dependency
The classic. One bulkhead per downstream service / database / cache. Sizes typically 10–100 depending on RTT and call rate.

### Per database / connection pool
Database connection pools are bulkheads. A separate pool per use case prevents one slow query from starving everything else.

```
PostgreSQL:   max_connections = 200
   App pool A (read replica):     50
   App pool B (primary, writes):  30
   App pool C (analytics queries): 20
   Headroom for ops / migrations:  100
```

If pool B (writes) is slow, pool A (reads) still works.

### Per request class
Critical user-facing requests get their own thread pool; background batch jobs get another; admin endpoints another. A slow batch job can't drown the user-facing path.

```
Web tier
  ┌──────────────────┐
  │ user-facing pool │ priority 1
  ├──────────────────┤
  │ admin pool       │ priority 2
  ├──────────────────┤
  │ background pool  │ priority 3
  └──────────────────┘
```

### Per tenant (multi-tenant)
Per-customer quotas / pools in SaaS systems. One customer's hot keys or runaway query doesn't starve everyone else.

### Per microservice (process boundary)
The strongest bulkhead: separate processes / containers / fleets. A bug or memory leak in service A can't crash service B.

### Per cell / region
Bulkheads at the architectural level: cell-based architecture. See [Cell-Based Architecture →](./cell-architecture.md).

---

## 5. Sizing the Bulkhead

The hardest part. Two approaches:

### Little's Law
```
   concurrency = throughput × latency
```

If a downstream serves 100 req/s and each call takes 50 ms, the working concurrency is `100 × 0.05 = 5`. Bulkhead size 10–20 gives 2–4× headroom for burst and tail.

If downstream serves 50 req/s at p99 = 2 s, working concurrency at p99 is `50 × 2 = 100`. Bulkhead size of 50 will throttle. Size 150 won't.

### Measured from load tests
Run a load test; observe concurrent in-flight calls at peak. Set bulkhead size at p99 of that + headroom.

### Default starting points
For per-dependency bulkheads on an HTTP service:
- **Fast, local dep** (cache, internal RPC): 100–200 concurrent.
- **Medium dep** (internal DB): 50–100.
- **External slow dep** (third-party API): 20–50.

Tune based on actual capacity. The wrong size is either everywhere (too small, false rejections) or nowhere (too large, doesn't trigger).

### What happens when the bulkhead is full?

Three options, picked at configuration time:

1. **Reject immediately** (fail-fast) → caller gets `BulkheadFullException`, can fall back.
2. **Wait briefly** (bounded queue) → caller waits up to N ms for capacity, then rejects.
3. **Queue unboundedly** → defeats the purpose; never do this.

Default to **reject immediately** with a short queue wait (e.g., 50 ms). Combined with a fallback or circuit breaker, this gives clean degradation under overload.

---

## 6. Bulkheads vs Other Patterns

Easy to confuse. Different jobs:

| Pattern | Job |
|---|---|
| Bulkhead | Bound concurrent in-flight requests per resource |
| Rate limiter | Bound request rate per client/key |
| Circuit breaker | Stop calling broken downstream entirely |
| Timeout | Bound wall-clock per call |
| Backpressure | Producer slows down when consumer can't keep up |
| Queue | Decouple producer and consumer |

Bulkheads bound **in-flight concurrency**. Rate limiters bound **arrival rate**. Both protect, but against different shapes of overload.

The combination:
```
Rate limiter (entry)         50 req/s per client
   │
   ▼
Bulkhead (per dependency)    50 concurrent calls to downstream
   │
   ▼
Timeout (per call)           1 s wall-clock
   │
   ▼
Circuit breaker              trip on sustained failure
   │
   ▼
Fallback                     graceful degradation when full
```

Each pattern catches a different failure mode. None is a substitute for any other.

---

## 7. Worked Example — Hystrix-style Thread Pool

A canonical Hystrix configuration for calling a payment API:

```yaml
payment_api:
  thread_pool:
    core_size: 30                # active threads
    max_queue_size: 10           # short queue for bursts
    queue_size_rejection_threshold: 10
  command:
    execution:
      isolation:
        strategy: THREAD          # vs SEMAPHORE
        thread:
          timeout_in_ms: 2000
      fallback:
        enabled: true
```

What this gives us:
- Up to 30 concurrent calls to the payment API.
- 11th–40th call: queues briefly (up to 10 in queue).
- 41st+ call: rejected immediately, falls back.
- Each call capped at 2 s wall-clock.
- If the API is slow, only 30 threads are stuck — the rest of the service is unaffected.

Without this, a payment API hanging at p99 = 30 s with normal traffic of 50 req/s would consume `50 × 30 = 1500` threads. Your default web server pool of 200 dies almost instantly.

---

## 8. Worked Example — Per-Tenant Bulkhead

A multi-tenant B2B SaaS receives requests with `X-Tenant-Id`. A noisy tenant can starve others without per-tenant limits:

```python
class TenantBulkhead:
    def __init__(self, max_concurrent_per_tenant=20):
        self.semaphores = defaultdict(
            lambda: Semaphore(max_concurrent_per_tenant)
        )

    @contextmanager
    def acquire(self, tenant_id):
        sem = self.semaphores[tenant_id]
        if not sem.try_acquire(timeout=0.05):
            raise TenantBulkheadFullException(tenant_id)
        try:
            yield
        finally:
            sem.release()

# usage
def handle_request(request):
    with tenant_bulkhead.acquire(request.tenant_id):
        return process(request)
```

Now no single tenant can hold more than 20 concurrent requests across the fleet (extend with a distributed counter for fleet-wide enforcement). Other tenants get unimpeded service even when one customer is misbehaving.

This is the kind of isolation that makes "one bad customer takes down the platform" impossible.

---

## 9. Operational Reality

### Metrics to expose
- **Active count** per bulkhead.
- **Available capacity** per bulkhead.
- **Rejection count** (capacity full).
- **Queue length** if queue-enabled.
- **Average wait time** when queued.

Without these, the bulkhead is invisible — and an invisible bulkhead is a bug factory.

### Alerting
- **Sustained high utilization** of a bulkhead (>80% for 5+ min): downstream getting slow, bulkhead is doing its job, but you should know.
- **Rejection spike**: many calls rejected — probably an outage in progress.
- **Bulkhead never close to full**: oversized; could free capacity for other pools.

### Tuning
- Adjust by p99 latency of downstream × throughput.
- After incidents, review bulkhead utilization in postmortems.
- Default sizes are starting points; production tuning is iterative.

### Per-instance vs fleet-wide
A bulkhead inside a single process is local — 100 instances × 20-per-bulkhead = 2000 total concurrent. For per-tenant quotas across a fleet, you need a distributed counter (Redis, central rate-limiter service). Many teams use per-instance bulkheads + per-tenant rate limits as a layered defense.

### Bulkheads and timeouts together
A bulkhead that doesn't time out is a leak waiting to happen. Always pair: every call inside a bulkhead has a timeout, so slow calls release their slot promptly.

### Bulkheads in event loops
For async code (Tokio, asyncio, Node), the "thread" is a logical task on the event loop. Bulkheads here are usually semaphores, and the cost of an exhausted bulkhead is rejecting the call — not blocking a thread. Semantics differ; correctness is the same.

### Mesh-level bulkheads
Istio / Envoy / Linkerd allow you to configure circuit-breaker-style outlier detection and **per-cluster connection pool limits**, which act as bulkheads at the sidecar level. Useful when your application can't easily add app-level bulkheads.

```yaml
# Envoy
clusters:
  - name: payment_api
    connect_timeout: 1s
    circuit_breakers:
      thresholds:
        - priority: DEFAULT
          max_connections: 50
          max_pending_requests: 10
          max_requests: 50
          max_retries: 3
```

This is a bulkhead in mesh dialect.

---

## 10. Real-World Examples

### Netflix Hystrix
The pattern's coming-of-age. Every microservice call wrapped in a Hystrix command with its own thread pool. Dashboard showed pool utilization in real time. Internally, Netflix moved away from thread-pool isolation toward semaphore-based (with resilience4j and Mantis) due to memory overhead, but the principle stuck.

### Resilience4j bulkheads
Two implementations: `ThreadPoolBulkhead` and `SemaphoreBulkhead`. Composable with other resilience4j patterns (retry, circuit breaker, time limiter).

### Database connection pools as bulkheads
HikariCP, c3p0, pgBouncer, PgCat, RDS Proxy — all enforce per-application or per-pool concurrent connection limits. The default JDBC `BasicDataSource` of `maxActive=8` is itself a small bulkhead.

### Multi-tenant SaaS quotas
Stripe, Shopify, GitHub all maintain per-tenant rate limits + per-tenant resource quotas (effectively bulkheads). A Cyber Monday merchant can't starve every other shop.

### AWS account-level quotas
The "account limit on Lambda concurrent executions" is a bulkhead at the AWS-account level. Hit it and Lambda invocations throttle, preserving capacity for the rest of the account.

---

## 11. When Bulkheads Help — and When They Don't

### Help
- Multiple dependencies on different criticality / latency profiles.
- Mix of fast and slow downstreams.
- Multi-tenant systems.
- Mix of user-facing and background work in one service.
- Anywhere a thread pool / connection pool could be exhausted by one bad path.

### Don't help much
- A single dependency that handles all traffic (no isolation gain unless you split logically).
- A purely event-driven, non-blocking single-loop server that doesn't have a "thread pool" to exhaust (you still want semaphore bulkheads to bound memory).
- A service where every request hits the same downstream identically (size the one pool correctly, you're done).

### Often missed
- Connection pools to **the same database** for different use cases. Splitting `read pool` / `write pool` / `analytics pool` is a bulkhead.
- Separate **fleets** for critical paths and best-effort paths. Sometimes the cleanest bulkhead is two separate deployments.

---

## 12. Common Mistakes / Anti-Patterns

- **No bulkhead at all.** Default behavior: shared thread pool, one slow dep → cascade.
- **Bulkhead too large.** Never rejects; doesn't isolate; just wastes memory.
- **Bulkhead too small.** Rejects legitimate traffic; false bottleneck.
- **No timeout inside the bulkhead.** Hung calls hold slots forever; bulkhead leaks.
- **Bulkhead without metrics.** Invisible failure mode.
- **Unbounded queue in front of bulkhead.** Defeats the purpose; memory grows; latency grows.
- **One bulkhead for all downstreams.** That's just a thread pool with extra config.
- **Bulkhead per HTTP endpoint instead of per downstream.** Isolates the wrong thing.
- **No fallback when bulkhead rejects.** User sees an error that could have been a degraded response.
- **Thread-pool bulkhead in async code.** Adds thread overhead for no isolation gain; use a semaphore.
- **Static sizing copied from a blog post.** Sizes depend on your downstream's latency × throughput.
- **No per-tenant bulkhead in multi-tenant systems.** Noisy neighbors take everyone down.
- **Forgetting connection pool sizing.** DB connections are a bulkhead too — don't oversize them and overwhelm the DB.

---

## 13. Decision Rule

```
For each external dependency in your service:
  ✓ Bound concurrent calls (thread pool or semaphore)
  ✓ Size = working concurrency × 2–4 headroom
  ✓ Fail-fast or short queue when full
  ✓ Timeout inside the bulkhead
  ✓ Metrics on utilization, rejections, queue length

For each request class (user-facing vs batch vs admin):
  ✓ Separate pool / fleet / queue
  ✓ Higher priority = larger / first-served bulkhead

For each tenant in a multi-tenant service:
  ✓ Per-tenant quota or per-tenant rate limit
  ✓ Plus per-tenant bulkhead if some tenants are large

For databases:
  ✓ Separate connection pools per use case
  ✓ Bound total DB connections in the cluster
  ✓ Use pgBouncer / RDS Proxy / PgCat to share efficiently
```

---

## 14. Cheat Card

```
PURPOSE     Compartmentalize resources so one bad dependency or
            noisy tenant can't sink the whole service.

FAILURE     Without bulkheads, a slow dep saturates the shared
            thread pool → all endpoints hang → cascade.

WITH        Bulkhead full = that dep's calls fail fast; everything
            else keeps working → graceful degradation.

FORMS
  Thread-pool bulkhead       per-dep thread pool (synchronous)
  Semaphore bulkhead         per-dep counter (async / non-blocking)
  Connection pool            per-DB / per-use-case (always)
  Per-tenant quota           multi-tenant systems
  Per-request-class pool     user-facing vs batch vs admin
  Per-cell / per-fleet       macro bulkhead (cell architecture)

SIZING      working concurrency ≈ throughput × latency
            bulkhead size = 2–4× working concurrency
            measure under real load; tune iteratively

WHEN FULL   Reject immediately (preferred) + short queue wait +
            fallback / circuit breaker decision

COMPOSE WITH
  Rate limit  bounds arrival rate
  Bulkhead    bounds in-flight per dep
  Timeout     bounds wall-clock per call
  Breaker     fails fast on sustained failure
  Fallback    graceful degradation

PITFALLS    No bulkhead · too large · too small · no timeout inside ·
            no fallback · unbounded queue · one big shared pool ·
            no per-tenant in multi-tenant systems · static sizes ·
            no metrics

RULE        Around every dependency: a bounded resource budget.
            Around every tenant: a quota. Around every request
            class: a pool. One bad path stays one bad path.
```

---

## 15. Resources

### Books
- *Release It!* — Michael Nygard. Bulkhead chapter is the modern reference.
- *Reactive Design Patterns* — Roland Kuhn. Isolation patterns at depth.
- *Microservices Patterns* — Chris Richardson. Bulkhead among the resilience patterns.

### Articles
- "Bulkhead Pattern" — Microsoft Cloud Design Patterns: <https://learn.microsoft.com/azure/architecture/patterns/bulkhead>
- "Hystrix Wiki — Isolation" — archived but excellent.
- "Resilience4j Bulkhead" — <https://resilience4j.readme.io/docs/bulkhead>
- "Polly Bulkhead Isolation" — Polly docs.
- "How Netflix Avoids Cascading Failures" — Netflix tech blog.
- "PgBouncer / RDS Proxy" deep dives — connection pool sizing.

### Videos
- "Defend Your Code Against Failure" — Ben Christensen (Netflix).
- "Bulkhead Pattern Explained" — ByteByteGo / various.
- "Resilience4j: A Lightweight Fault-Tolerance Library" — Robert Winkler.

### Tools
- **resilience4j** — `ThreadPoolBulkhead`, `SemaphoreBulkhead`.
- **Polly** — Bulkhead policy.
- **Hystrix (archived)** — original thread-pool isolation.
- **Envoy / Istio / Linkerd** — circuit_breakers config = bulkhead at the sidecar.
- **HikariCP / pgBouncer / PgCat / RDS Proxy** — DB connection bulkheads.
- **Kubernetes ResourceQuotas / LimitRanges** — cluster-level bulkheads.

### Adjacent reading
- [Fault Tolerance Patterns](./fault-tolerance.md)
- [Circuit Breaker Pattern](./circuit-breaker.md)
- [Retry, Timeout, and Exponential Backoff](./retry-timeout-backoff.md)
- [Graceful Degradation →](./graceful-degradation.md)
- [Backpressure](../10-scalability/backpressure.md)
- [Hot Partition Problem](../10-scalability/hot-partitions.md)
- [Blast Radius & Cell-Based Architecture →](./cell-architecture.md)
- [Connection Pooling](../04-databases/connection-pooling.md)

---

*Previous:* [← Retry, Timeout, and Exponential Backoff](./retry-timeout-backoff.md)  |  *Next:* [Graceful Degradation →](./graceful-degradation.md)
