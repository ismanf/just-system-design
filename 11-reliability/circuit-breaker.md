# Circuit Breaker Pattern

> **TL;DR** — A **circuit breaker** is a small state machine wrapping a remote call. It monitors failures and **trips open** when the downstream is failing, so subsequent calls **fail fast** without hammering the broken service. After a cooling-off period it transitions to **half-open**, lets a few probe requests through, and either closes (recovered) or re-opens (still broken). The pattern is the difference between "downstream slow → callers pile up → entire fleet exhausts threads → cascading collapse" and "downstream slow → breaker trips → callers fail fast → graceful degradation kicks in." Made famous by Netflix's Hystrix (now archived); modern implementations include **resilience4j** (JVM), **Polly** (.NET), and built-in mesh circuit breakers in **Envoy / Istio / Linkerd**. The pattern is cheap, effective, and almost always misconfigured the first time — thresholds, windows, and half-open behavior all need tuning to the workload.

---

## 1. The Idea

```
                  closed ──── failures cross threshold ───►  open
                    ▲                                          │
                    │ probes succeed              after timeout │
                    │                                          │
                  half-open ◄───────────────────────────────────┘
                    │
                    │ probes fail
                    ▼
                  open
```

Three states. One state machine. One job: don't keep calling a downstream that's clearly broken.

```
CLOSED      Normal operation. Calls flow through. Counters track failures.

OPEN        Trip thrown. All calls fail fast (no remote attempt).
            Stays open for a cooldown window (e.g., 30 s).

HALF-OPEN   Cooldown over. Allow a few probe calls. If they succeed,
            close the circuit. If any fail, re-open and reset the timer.
```

The breaker is a **circuit breaker** in the electrical sense — when current flow becomes dangerous, the breaker opens the circuit to protect the downstream wiring.

---

## 2. Why It Exists

The motivating failure shape — without a breaker, a slow downstream destroys the caller:

```
  downstream gets slow (p99 → 30 s)
       │
       ▼
  caller threads block on calls
       │
       ▼
  thread pool exhausts
       │
       ▼
  caller can't serve OTHER requests either
       │
       ▼
  caller's health checks fail
       │
       ▼
  caller is pulled from rotation
       │
       ▼
  load shifts to remaining nodes
       │
       ▼
  CASCADE — entire fleet falls over
```

A circuit breaker breaks this chain by saying: "downstream is broken; I will fail fast on this call without using a thread for 30 seconds." Threads stay free. The caller's own SLO degrades but doesn't crash.

Netflix learned this the hard way in the early 2010s. Their analysis of cascading failures led to Hystrix; the resulting pattern became standard.

---

## 3. State Transitions in Detail

```
START in CLOSED
   - request → call downstream
   - on success → reset / decrement failure counter
   - on failure or slow call → increment failure counter
   - if (failure rate > threshold AND volume > min_calls)
        → transition to OPEN

OPEN
   - request → immediately fail (no downstream call)
   - after open_timeout (e.g., 30 s) → transition to HALF-OPEN

HALF-OPEN
   - allow N probe requests through (e.g., 5)
   - if all succeed → CLOSED
   - if any fail → OPEN (reset timer)
   - other concurrent requests during half-open → fail fast
```

Knobs the implementation gives you:
- **Failure threshold** — error rate or count at which to trip.
- **Volume threshold** — minimum sample size; don't trip on 2 calls.
- **Sliding window** — count window (last N calls) vs time window (last T seconds).
- **Open duration** — cooldown before half-open.
- **Half-open probe count** — how many calls to test recovery.
- **What counts as a failure** — exceptions, timeouts, certain HTTP codes, slow calls.

Default starting points for a typical RPC call (these will need tuning):

```yaml
failure_rate_threshold: 50%          # trip at 50% errors
slow_call_rate_threshold: 50%        # also trip at 50% slow calls
slow_call_duration: 2s               # what "slow" means
sliding_window:
  type: COUNT_BASED
  size: 100
minimum_number_of_calls: 20
wait_duration_in_open_state: 30s
permitted_calls_in_half_open: 5
```

These numbers are not gospel. Tune from production data.

---

## 4. What Counts as a Failure

A surprisingly subtle decision. Common categories:

| Event | Usually counts? |
|---|---|
| Network exception (connection refused, reset) | Yes |
| Timeout | Yes |
| HTTP 5xx | Yes |
| HTTP 4xx | **No** (client's fault, not downstream's) |
| HTTP 429 | Maybe (downstream backpressure, not failure) |
| Slow but successful call | Yes (separately) |
| Application-level error code in response body | Often yes |

Get this wrong and you'll:
- Trip on legitimate 404s (bad).
- Not trip on slow downstream that returns 200 OK after 10 s (bad).
- Trip on rate-limit 429s when the downstream is fine (bad).

**Most production systems have a separate "slow call" counter alongside the failure counter** — the breaker should trip on either condition. A downstream returning 200 OK after 30 s is functionally as dangerous as one returning 500 in 50 ms.

---

## 5. The Three Window Strategies

How you count failures matters.

### Count-based window
"Last N calls." Once N calls have happened, compute the failure rate over those N.
- **Pro**: Stable threshold regardless of traffic rate.
- **Con**: At low traffic, recent failures count for a long time.

### Time-based window
"Last T seconds." Compute the failure rate over the last T seconds.
- **Pro**: Recency-correct under varying load.
- **Con**: Low-traffic periods give noisy small samples.

### Rolling / sliding bucket window
Split the window into buckets; expire the oldest bucket every interval.
- The hybrid most production implementations use.

For real traffic, **prefer count-based with a minimum-volume threshold**. It avoids the "1 call out of 1 failed, 100% failure rate, trip!" pathology and adapts well from low to high QPS.

---

## 6. Per-Instance vs Per-Service Breakers

Where you place the breaker matters as much as how you tune it.

```
                ┌──────────┐   ┌──────────┐
                │  caller  │   │  caller  │
                └────┬─────┘   └────┬─────┘
                     │              │
            ┌────────┴──────────────┴────────┐
            │     load balancer / mesh        │
            └────────┬──────────────┬────────┘
                     │              │
                ┌────▼────┐   ┌─────▼─────┐
                │ inst 1  │   │  inst 2   │
                └─────────┘   └───────────┘
```

Three places to put a circuit breaker:

### Per-downstream-instance
Each caller maintains a breaker per downstream IP/instance. Pulls bad instances out of rotation without affecting healthy ones.

- **Where**: Envoy, gRPC client-side, Finagle.
- **Pro**: Granular; one bad pod doesn't trip the whole service.
- **Con**: Each caller must coordinate state per instance.

### Per-downstream-service
One breaker per dependency, regardless of instance. Trips the whole downstream's calls.

- **Where**: most application-level libraries (Hystrix, resilience4j).
- **Pro**: Simple, fewer counters.
- **Con**: One bad instance can trip the service-level breaker even if others are healthy. Mitigation: layer with per-instance load balancing.

### Service-mesh / sidecar
Envoy / Linkerd / Istio implement outlier detection per upstream host. Bad hosts are ejected; the breaker behavior is configured in the mesh, not in app code.

- **Where**: any service mesh.
- **Pro**: Polyglot; no app-level changes.
- **Con**: Less context for retry/fallback decisions; tuning lives in the mesh config.

Modern production stacks combine **mesh-level outlier detection** (eject bad instances) with **application-level circuit breakers** (degrade or fallback when the dependency is unhealthy).

---

## 7. Worked Example — A REST Call

```python
breaker = CircuitBreaker(
    name="payments_api",
    failure_rate_threshold=50,        # %
    slow_call_rate_threshold=50,
    slow_call_duration_threshold=2.0, # seconds
    minimum_number_of_calls=20,
    sliding_window_size=100,          # count-based
    wait_duration_in_open_state=30.0,
    permitted_calls_in_half_open=5,
)

def charge(amount, card):
    try:
        return breaker.call(payments_api.charge, amount, card)
    except CircuitBreakerOpenError:
        # downstream is broken; fall back
        return queue_charge_for_retry(amount, card)
    except PaymentsError as e:
        log.warning("payment failed: %s", e)
        raise
```

Three observable behaviors:
1. When the payments API is healthy: every call goes through normally. Counter is ~0% failures.
2. When the payments API is slow: slow calls accumulate; breaker trips; subsequent calls go to the fallback queue immediately.
3. After 30 s: 5 probe calls go through. If the API recovered, the breaker closes; otherwise it opens for another 30 s.

The fallback queue is what makes this useful — a tripped breaker without a fallback is just a faster failure. See [Graceful Degradation →](./graceful-degradation.md).

---

## 8. Implementations

### Hystrix (Netflix, archived 2018)
The grandfather. Introduced thread-pool isolation + circuit breaker + metrics. Officially deprecated; replaced by resilience4j and service mesh.

### resilience4j (JVM)
The modern JVM choice. Lightweight, functional-style, integrates with Reactor / RxJava / Kotlin coroutines. Provides circuit breaker, retry, rate limiter, bulkhead, time limiter — composable.

### Polly (.NET)
The de facto .NET resilience library. Same patterns: retry, circuit breaker, fallback, bulkhead, timeout.

### gobreaker (Go)
Sony's simple, clean Go implementation. Sub-100-line state machine.

### Envoy / Istio / Linkerd
Mesh-native outlier detection. Strips bad endpoints from upstream clusters based on consecutive 5xx, gateway failures, slow responses.

### Cloud-native and language-specific
- **AWS App Mesh / AWS Service Mesh** — Envoy-based.
- **Cloudflare Workers / Lambda layer libraries** — application-level wrappers.
- **Custom in-house** — many big companies still build their own for tight integration with metrics and feature flags.

---

## 9. Half-Open: The Tricky State

Half-open is where breakers misbehave under load. The questions:

- **How many probes?** Too few → flap; too many → hammer a still-broken downstream.
- **What if the probes are slow?** Some implementations count slow probes as failures; others time them out and re-open.
- **What about concurrent traffic during half-open?** Most implementations fail-fast everything except the N probes. Some allow proportional traffic through.
- **How fast does the breaker re-close?** Some require N consecutive successes; others reset after one.

Anti-pattern: a downstream that's flaky (50/50 on each call) puts the breaker in a permanent open/half-open/open cycle. The remedy: smooth the half-open transition by requiring multiple consecutive successes.

---

## 10. Composition with Retries, Timeouts, Bulkheads

A circuit breaker rarely sits alone. The typical resilience stack around a downstream call:

```
   ┌────────────────────────────────────────────────────────┐
   │  (outer to inner)                                       │
   │                                                          │
   │  Fallback     ─ what to do if all of the below fail    │
   │     │                                                    │
   │     ▼                                                    │
   │  CircuitBreaker  ─ fail fast if downstream is broken   │
   │     │                                                    │
   │     ▼                                                    │
   │  Retry         ─ try a few times with backoff          │
   │     │                                                    │
   │     ▼                                                    │
   │  TimeLimiter   ─ cap wall-clock on each attempt        │
   │     │                                                    │
   │     ▼                                                    │
   │  Bulkhead      ─ cap concurrent in-flight calls        │
   │     │                                                    │
   │     ▼                                                    │
   │  Actual remote call                                     │
   │                                                          │
   └────────────────────────────────────────────────────────┘
```

The order matters:
- Bulkhead inside time limiter: bulkhead-blocked calls don't count against time budget unfairly.
- Retry inside circuit breaker: if circuit's open, no point retrying.
- Fallback outermost: catches every failure mode.

resilience4j composes these as decorators; Polly does the same with `Policy.Wrap`. The composition is a small DSL each library encourages you to use uniformly.

---

## 11. Operational Reality

### Metrics to expose
- Breaker state (CLOSED/OPEN/HALF_OPEN) as a gauge.
- State transitions per minute.
- Failure rate.
- Slow call rate.
- Call latency percentiles.
- Calls rejected because circuit was open.

Without these, the breaker is invisible — and an invisible breaker is impossible to debug.

### Alerting
- Page when breakers stay open longer than X minutes.
- Page when a breaker flaps (>N transitions per minute).
- Track total "circuit-open rejections" as a top-level SLI input.

### Dashboards
- One row per dependency: state over time, failure rate, latency.
- Comparison view across breakers in a service.

### Logs and tracing
- Log every state transition with reason.
- Inject breaker state into traces so you can see "circuit open" in the request span.

### Forcing state
Most libraries let you forcibly **disable** (always closed), **force open** (kill switch), or **force closed** (override). Use sparingly; document; alert when set.

---

## 12. Real-World Examples

### Netflix Hystrix (2012–2018)
The pattern's coming-of-age. Thread-pool-isolated, breaker-per-dependency, dashboards in real time. Hystrix is archived because the team felt the pattern was understood and the library wasn't where the innovation was; modern Netflix uses resilience4j and mesh-level breakers.

### Stripe API client behavior
When Stripe's API returns 5xx, the official client backs off with jitter. Under sustained outage, integrations should circuit-break and queue retries to avoid amplifying load.

### AWS SDK retry + breaker (in some flavors)
The AWS SDK has built-in retry; some services (DynamoDB) recommend application-level breakers for sustained ThrottlingException windows.

### gRPC client-side load balancing
gRPC's xDS-based load balancing supports outlier detection — instances with high error rates are removed from the round-robin pool. This is effectively a per-instance breaker at the transport layer.

### Service mesh ejection
Istio + Envoy outlier_detection config:
```yaml
outlierDetection:
  consecutive5xxErrors: 5
  interval: 30s
  baseEjectionTime: 30s
  maxEjectionPercent: 50
```
A pod that returns 5 consecutive 5xx is ejected for 30 s. After ejection, it's slowly probed back. Same state machine; lives in the sidecar.

---

## 13. When to Use — and When Not To

### Use when
- Calling a remote service that can fail or slow down independently.
- Slow responses can starve your thread/connection pool.
- A clear fallback exists (cached data, stale value, partial response, queue for later).
- You can express "failure" clearly (codes, exceptions, latency).

### Don't bother when
- The call is in-process / lock-free (no remote hop).
- The call is essential (no fallback possible). Use timeouts + bulkheads instead; circuit breaker would just convert "user sees error after 30 s" to "user sees error immediately" — sometimes desirable, sometimes not.
- The downstream is so fast that timeouts are cheap and pools are huge.
- A single slow call can't damage the caller (true for some async, fire-and-forget paths).

In doubt: add it. The cost is low and the benefit during incidents is high.

---

## 14. Common Mistakes / Anti-Patterns

- **Tripping on too-small samples.** Two failures in a row trip the breaker; you flap. Use a minimum-volume threshold.
- **Counting 4xx as failures.** A wave of 404s trips a healthy downstream.
- **Counting 429 as failures.** Backpressure isn't a downstream failure; it's a signal to slow down.
- **No "slow call" detection.** Downstream returns 200 OK after 30 s; breaker stays closed; threads exhausted.
- **Breaker per service-level method instead of per downstream.** Tripping one method doesn't help if you call the same downstream by another path.
- **No fallback.** Open circuit just fails faster — sometimes worse for the user than a slow success.
- **Probing too aggressively in half-open.** 100 concurrent probes hammer a still-broken downstream and re-trip immediately.
- **No visibility.** State transitions invisible; can't debug incidents.
- **Default Hystrix settings copy-pasted everywhere.** Defaults are starting points; tune.
- **Forgetting timeouts.** A breaker that opens after 30 s of failures still hung 30 s of threads first. Pair with aggressive timeouts.
- **Using the breaker to mask bugs.** "It trips sometimes; we'll deal with it" → permanent silent failure mode.
- **Ignoring half-open flap.** A 50/50 downstream causes open ↔ half-open ↔ open loops; require N consecutive successes before closing.
- **Cascading open circuits.** Service A breaker opens → service A returns errors → service B breaker around A opens → ... → whole stack tripped. Mitigate with bulkheads, fallbacks, and clear failure semantics.

---

## 15. Decision Rule

```
Around every cross-service call:
  ✓ Bulkhead    bound concurrency
  ✓ Timeout     bound wall-clock
  ✓ Retry       bounded count, backoff, jitter, idempotent only
  ✓ Circuit Breaker  fail fast on sustained failure
  ✓ Fallback    do something useful when broken

The breaker is one knob in a chain. Tune as a group:
  - Timeout < (caller's deadline)
  - Retry count × (timeout + backoff) < (caller's deadline)
  - Breaker trips before retries amplify load
  - Half-open recovers gracefully (require N consecutive successes)
```

---

## 16. Cheat Card

```
PURPOSE     Fail fast when a downstream is broken or slow. Protect
            the caller's threads, pools, and SLO. Probe for recovery
            without hammering.

STATES
  CLOSED      normal; counters track failures
  OPEN        fail immediately; cool down for N seconds
  HALF-OPEN   allow K probe calls; close on all-success, re-open
              on any failure

KNOBS
  failure_rate_threshold      e.g., 50% errors
  slow_call_rate_threshold    e.g., 50% above slow_threshold
  slow_call_duration          e.g., 2 s
  sliding_window              count-based or time-based
  minimum_number_of_calls     prevent tripping on tiny samples
  wait_duration_in_open       cooldown before half-open
  permitted_calls_in_half_open  probe count

WHAT COUNTS AS FAILURE
  ✓ network exception      ✓ timeout      ✓ HTTP 5xx
  ✓ slow success (separately)
  ✗ HTTP 4xx (client's fault)
  ~ HTTP 429 (depends; usually no)

PLACEMENT   per-instance · per-service · service-mesh outlier
            production stacks usually combine mesh + app-level

COMPOSE WITH  Bulkhead → Timeout → Retry → Breaker → Fallback

PITFALLS    No min-sample threshold · counting 4xx · no slow-call
            detection · no fallback · aggressive half-open probes ·
            no metrics · forgotten timeouts · default tuning ·
            cascading breaker chains

RULE        Around every cross-service call: bulkhead, timeout,
            retry-with-backoff, breaker, fallback. Tune as a group.
            Make state and transitions observable.
```

---

## 17. Resources

### Books
- *Release It!* — Michael Nygard. The pattern's modern formulation.
- *Reactive Design Patterns* — Roland Kuhn. Circuit breaker in reactive systems.
- *Microservices Patterns* — Chris Richardson. Circuit breaker chapter.

### Articles
- "Making the Netflix API More Resilient" — Netflix tech blog, the original Hystrix essays.
- "Fault Tolerance in a High Volume, Distributed System" — Ben Christensen, Netflix.
- "Circuit Breaker" — Martin Fowler: <https://martinfowler.com/bliki/CircuitBreaker.html>
- "resilience4j Circuit Breaker" docs: <https://resilience4j.readme.io/docs/circuitbreaker>
- "Polly Circuit Breaker" docs: <https://www.pollydocs.org/>
- "Envoy Outlier Detection" docs: <https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/outlier>
- "Hystrix Wiki" — archived but still informative.

### Videos
- "Defend Your Code Against Failure" — Ben Christensen (Netflix).
- "Hystrix and Cascade Failures" — various Netflix talks.
- "Resilience4j: A Lightweight Fault-Tolerance Library" — Robert Winkler.
- ByteByteGo — "Circuit Breaker Pattern" overview.

### Tools
- **resilience4j** — JVM resilience library.
- **Polly** — .NET resilience library.
- **gobreaker** — simple Go circuit breaker.
- **opossum** — Node.js circuit breaker.
- **Envoy / Istio / Linkerd** — mesh-level outlier detection.
- **failsafe-js / Cockatiel** — JS/TS resilience libraries.

### Adjacent reading
- [Fault Tolerance Patterns](./fault-tolerance.md)
- [Retry, Timeout, and Exponential Backoff →](./retry-timeout-backoff.md)
- [Bulkhead Pattern →](./bulkhead.md)
- [Graceful Degradation →](./graceful-degradation.md)
- [Backpressure](../10-scalability/backpressure.md)
- [Service Mesh (Istio, Linkerd)](../03-apis/service-mesh.md)
- [Tail Latency & p99](../16-performance/tail-latency.md)

---

*Previous:* [← Fault Tolerance Patterns](./fault-tolerance.md)  |  *Next:* [Retry, Timeout, and Exponential Backoff →](./retry-timeout-backoff.md)
