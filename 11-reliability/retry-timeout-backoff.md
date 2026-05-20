# Retry, Timeout, and Exponential Backoff

> **TL;DR** — Three small primitives that together make remote calls survivable. A **timeout** caps how long a call can hang — without one, a slow downstream blocks the caller forever. A **retry** tries again after a transient failure — without one, a brief network blip becomes a user error. **Exponential backoff with jitter** waits progressively longer (and randomly) between retries so a hundred clients don't synchronize and finish off the very downstream they're trying to reach. The combination is mandatory on every network call. The two most common bugs are **no timeout at all** and **retries without backoff or jitter** — both turn small failures into outages. Combine with **idempotency** (so retries don't double-charge), **circuit breakers** (so a broken downstream stops getting hit), **bulkheads** (so retries don't exhaust the thread pool), and **deadlines** (so the user-perceived wait is bounded end-to-end). This page is the playbook.

---

## 1. The Three Primitives

```
TIMEOUT              cap wall-clock per call
                     ─────────────────────────
                     if no response in T → fail

RETRY                try again on transient failure
                     ──────────────────────────
                     bounded count + only on retriable errors
                     + only on idempotent operations

BACKOFF + JITTER     wait between retries
                     ─────────────────────
                     exponential (1s, 2s, 4s, 8s)
                     + randomness to break sync
```

Each is useless without the others:
- **Timeout without retry**: a transient blip becomes a user error.
- **Retry without timeout**: retries pile up on a hung call.
- **Retry without backoff**: 1000 callers hammer a struggling downstream.
- **Backoff without jitter**: 1000 callers retry at exactly the same moments → synchronized waves.

Together they turn "everything is on fire" into "a few ms blip the user didn't notice."

---

## 2. Timeouts

The single most impactful, most under-applied reliability primitive.

### The rule
**Every network call has a timeout.** No exceptions. Database queries, HTTP requests, gRPC calls, DNS lookups, S3 PUTs, cache reads, lock acquisitions — every blocking call must specify *how long it's willing to wait*.

A network call without a timeout is a bug. The default behavior of "wait forever" is never what you want in production.

### How to pick a timeout

Two approaches:

**Static budgets**: a documented timeout per dependency type.
```
fast cache lookup        50 ms
DB point read            100 ms
DB complex query         1 s
internal RPC             500 ms
external HTTP            2 s
S3 single GET            5 s
S3 multipart upload      60 s
batch job                15 m
```

These should reflect the dependency's actual latency distribution (p99 + headroom), not the average. A timeout shorter than p99 spuriously fails healthy calls; a timeout 100× p99 lets sick calls hang forever.

**Deadline propagation**: the user's request comes in with a budget (e.g., 200 ms). Each downstream call gets *the remaining budget*. The deepest hop in the call graph has the least time. This is the gold standard.

```
User request          deadline = 200 ms
   │
   ▼ T = 0
Service A             timeout for next call = 190 ms (10 ms used)
   │
   ▼ T = 5 ms
Service B             timeout = 175 ms (15 ms used so far)
   │
   ▼ T = 8 ms
Database              timeout = 50 ms (or remaining, whichever is less)
```

gRPC has deadline propagation built in. HTTP requires explicit header conventions (`X-Request-Deadline`, `traceparent` etc.). Without propagation, each hop independently waits its full local timeout — and the user has already left.

### What "timeout" actually means

Several distinct timeouts on every HTTP/TCP call. Distinguish them or be surprised:
- **Connect timeout** — how long to wait for TCP/TLS handshake. Default ~30 s in many libraries; cap to 1–5 s.
- **Read timeout** (or per-call): time waiting for next byte after the connection is established.
- **Total timeout / overall deadline**: end-to-end wall clock.
- **Idle timeout**: time the connection can sit idle in a pool.
- **Pool / acquisition timeout**: how long to wait for a free connection from the pool.

The classic bug: setting only `read_timeout` and forgetting `connect_timeout` — a DNS resolution that hangs for 60 s blows your budget every time.

### Per-call vs aggregate timeouts
A retry policy that does 3 attempts at 500 ms each can take 1.5 s + backoff before failing. **The caller's deadline must be larger than the worst case**, or the deadline expires before retries finish. Always set an aggregate deadline alongside per-attempt timeouts.

---

## 3. Retries

### When to retry

A retry is only safe when **all three** are true:
1. The error is **transient** (transient = will probably succeed if tried again).
2. The operation is **idempotent** (safe to apply more than once).
3. There is **headroom** (we haven't blown the deadline; we haven't exceeded the retry budget).

### What errors are retriable?

| Error | Retry? |
|---|---|
| Network exception (connection refused, reset) | Yes |
| Timeout | Yes (if idempotent) |
| HTTP 408 (Request Timeout) | Yes |
| HTTP 425 (Too Early) | Yes |
| HTTP 429 (Too Many Requests) | Yes, with Retry-After |
| HTTP 500 (Internal Server Error) | Maybe — usually yes, careful |
| HTTP 502/503/504 (Gateway / Unavailable / Timeout) | Yes |
| HTTP 4xx (other) | **No** — client's fault, retry won't help |
| HTTP 401, 403 | No — retry won't authenticate you |
| HTTP 404 | No — the thing doesn't exist |
| gRPC UNAVAILABLE, DEADLINE_EXCEEDED, RESOURCE_EXHAUSTED | Yes |
| gRPC NOT_FOUND, PERMISSION_DENIED, INVALID_ARGUMENT | No |
| DB deadlock | Yes (commonly) |
| DB serialization failure | Yes |
| App-level "duplicate" or "validation" errors | No |

Retrying non-retriable errors is the second most common retry bug. The first is...

### Idempotency

If you POST `{ amount: 100 }` and the request times out, did the charge go through or not? Without idempotency, you can't safely retry — a retry might double-charge.

The fix: **idempotency keys**. The client generates a unique key per logical operation; the server deduplicates by key. See [Idempotency →](../03-apis/idempotency.md) and [Idempotent Operations & Retries →](./idempotency-retries.md).

Without idempotency:
- GET, HEAD, PUT, DELETE are usually safe (by HTTP semantics).
- POST and most state-changing RPCs are not safe; do not retry.

With idempotency keys, every operation becomes safe to retry. This is the unlock that makes retry policies aggressive.

### Bounded retry count

Always bound the count. Production-typical:
- **2–3 retries** for user-facing requests.
- **5–10 retries** for background jobs and async work.
- **Unbounded retries** only for fire-and-forget pipelines with backpressure and deduplication; usually a bad idea.

Each retry adds latency (timeout + backoff) — bound the count so the total budget stays reasonable.

### Retry budget (token bucket)

A retry budget protects against retry storms across the fleet. Conceptually: total retries are capped at, say, **10% of original requests over the last minute**. When the budget runs out, retries are disabled until traffic recovers.

```
if (retries_in_last_60s / requests_in_last_60s) > 0.1:
    skip retry; fail fast
```

Google's gRPC and Envoy both implement this. Without it, a fleet-wide downstream slowdown gets amplified by retries on every caller simultaneously.

---

## 4. Exponential Backoff with Jitter

The naive retry is "try, fail, try again immediately." This is the worst possible behavior under load — it amplifies the original failure.

**Exponential backoff**: wait progressively longer.

```
attempt 1: try
attempt 2: wait 1 s, try
attempt 3: wait 2 s, try
attempt 4: wait 4 s, try
attempt 5: wait 8 s, try
```

Formally: `delay = base × 2^(attempt - 1)`, optionally capped.

This alone isn't enough. If a thousand clients all hit a failure at time T, they all retry at T+1s, then T+3s, then T+7s. Synchronized waves.

**Jitter**: add randomness.

### The three canonical jitter strategies

**Full jitter** (recommended for most cases):
```
delay = random_between(0, base × 2^attempt)
```
Each client picks a random time within the window. Best at decorrelating retries.

**Equal jitter**:
```
delay = (base × 2^attempt) / 2 + random_between(0, (base × 2^attempt) / 2)
```
Half deterministic, half random. Slightly tighter clustering.

**Decorrelated jitter** (recommended by AWS for many scenarios):
```
delay = random_between(base, previous_delay × 3)
```
Self-adjusting based on previous delay; smoother distribution under sustained failures.

The AWS Architecture Blog post "[Exponential Backoff and Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)" is the canonical reference. The conclusion: **full jitter or decorrelated jitter outperforms no jitter substantially in tail latency and downstream success rate**.

### Capping the delay

Set a max delay (e.g., 30 s) so you don't wait 17 minutes on attempt 12.

```
delay = min(max_delay, random_between(0, base × 2^attempt))
```

For batch jobs where you genuinely want to wait hours, the cap can be higher.

### Honoring `Retry-After`

If the downstream tells you when to retry (`Retry-After: 30` header, `RESOURCE_EXHAUSTED` with details), honor it. The downstream knows its own state better than you do.

---

## 5. Deadlines — End-to-End Time Budget

Timeouts cap per-call wall-clock. **Deadlines** cap end-to-end wall-clock for a user request across many hops.

```
request enters: deadline = now + 200 ms

each downstream call:
  timeout = min(per-call-timeout, deadline - now)

if (deadline - now) < required_for_call:
  fail fast; don't even try

retries:
  if (deadline - now) < (timeout + backoff):
    give up; don't retry
```

Deadlines compose naturally. gRPC has first-class deadlines that propagate across hops. HTTP needs explicit header passing (custom headers, or the `Expect-Reply-By`/`X-Deadline` conventions some teams adopt).

Without deadlines: the user gives up at 5 s, but your service keeps trying for another 30 s on every retry, wasting capacity on work no one cares about. **Deadlines are how you abandon stale work.**

---

## 6. Worked Example — The Production Setup

A canonical resilience4j-style configuration for a service-to-service call:

```yaml
timeout:
  connect: 1 s
  read: 500 ms
  total: 2 s          # an attempt can't exceed this

retry:
  max_attempts: 3
  retryable_exceptions:
    - ConnectException
    - SocketTimeoutException
    - GrpcUnavailableException
    - GrpcDeadlineExceededException
  retryable_status_codes:
    - 408
    - 429
    - 500
    - 502
    - 503
    - 504
  ignore_status_codes:
    - 400
    - 401
    - 403
    - 404
    - 422
  backoff:
    type: exponential_with_jitter
    base: 100 ms
    multiplier: 2
    max_delay: 2 s
    strategy: full_jitter
  retry_budget:
    max_retries_per_minute: 10
    request_per_minute: 100   # = 10% of requests

deadline:
  inherit_from_request: true
  default: 2 s

circuit_breaker:
  failure_rate_threshold: 50%
  minimum_number_of_calls: 20
  wait_in_open_state: 30 s
```

In English:
- Each attempt is capped at 2 s (with 1 s connect + 500 ms read components).
- Up to 3 attempts; only on idempotent retriable errors.
- Backoff: 100 ms → 200 ms → 400 ms, with full jitter, capped at 2 s per wait.
- Retries don't exceed 10% of base traffic.
- If the user's deadline expires mid-retry, give up.
- A circuit breaker wraps the whole thing.

This is roughly what production looks like at AWS, Google, Stripe, Netflix, and most companies that have learned the hard way.

---

## 7. Worked Example — A Bad Retry Policy

To make the harm concrete, here's a real-world anti-pattern that has caused multiple outages at multiple companies:

```python
def call_downstream(payload):
    for attempt in range(10):                 # too many
        try:
            return http.post(URL, payload)    # no timeout
        except Exception:                     # catches everything
            continue                          # no backoff
```

When the downstream slows down to 30 s/request:
- Every request hangs 30 s, ten times, before giving up — **300 s per logical request**.
- All threads block on hung calls; pool exhausts.
- Each caller does 10 retries → downstream gets **10× normal load** while it's already drowning.
- Errors catch and swallow 4xx — the downstream's "you sent garbage" response is retried 10× too.
- No backoff — synchronized retries from 1000 clients.

This is how single slow downstreams take down entire products. The fix is the configuration above.

---

## 8. The Math of Retry Amplification

Retries multiply downstream load. If every call retries N times and 10% of calls fail:

```
Original load:        Q requests/sec
Retry rate (10% fail × N retries): 0.1 × N × Q
Effective load on downstream:      Q × (1 + 0.1 × N)

N = 3:  1.3× load
N = 5:  1.5× load
N = 10: 2× load
```

For 50% failures and N = 5: **3.5× load**. The downstream that was struggling at 1× is now serving 3.5× — guaranteed collapse.

**Retry budgets** cap the amplification. Without one, you guarantee that any downstream slowdown becomes a self-reinforcing collapse.

---

## 9. Operational Reality

### Per-attempt vs aggregate timeout
A retry policy that does 3 attempts at 500 ms = 1.5 s before backoff + retries. Always set an aggregate deadline >1.5 s, or the caller's deadline expires mid-retry. The aggregate is what the user actually waits.

### TCP retries and timeouts
TCP retransmits packets transparently. A "connection timeout" of 500 ms may include several silent retransmissions inside it. Linux defaults (`tcp_syn_retries`, `tcp_retries2`) can hold a connection in retry state for minutes. For latency-critical paths, tune kernel TCP settings or use shorter explicit timeouts.

### DNS timeouts
DNS resolution can hang for 5–30 s on a misconfigured network. Many libraries don't expose this. Use `getaddrinfo` with `AI_NUMERICSERV` where possible, or resolve via a known resolver with a short timeout, or use a caching resolver in front.

### Connection pool timeouts
Distinct from request timeouts. A request waiting for a free connection from the pool is queued. Always set `pool_acquisition_timeout`. Otherwise, hung calls keep connections occupied and new requests queue forever.

### Slow successful responses
A `200 OK` after 30 s is more dangerous than a 500 in 50 ms because the breaker doesn't trip. Set timeouts that catch slow successes.

### Retries during failover
A primary DB fails over; the first call after failover fails; your retry hits the new primary and succeeds. This is exactly what retries are for — *as long as the operation is idempotent*.

### "Retry on idempotent" trap
GET/HEAD/PUT/DELETE are idempotent by HTTP convention, but only if the server actually implements them that way. A PUT that uses `serial++` is not idempotent. A DELETE that returns 404 on second call is technically not idempotent (state observable differs). Verify your endpoints actually behave.

### Hedged requests
A latency-mitigation technique distinct from retries: send a second request *before* the first times out (e.g., after p95) to whichever of N replicas responds first. See [Tail Latency →](../16-performance/tail-latency.md). Cassandra, gRPC, and many low-latency systems use this.

### Long-running operations
For operations that take minutes (a video transcoding job, a long export), don't tie up a TCP connection. Use **polling** or **callbacks**: return a 202 Accepted + job ID; the client polls for completion. Timeouts apply only to the polling calls.

---

## 10. Composition with Other Patterns

```
Timeout    bounds wall clock per attempt
Retry      tries again on transient errors
Backoff    waits between retries
Jitter     decorrelates retries across fleet
Deadline   bounds total wall clock end-to-end
Idempotency  makes retries safe
Circuit Breaker  stops retries when downstream is broken
Bulkhead   prevents pool exhaustion from retries
Rate Limit / Retry Budget  caps total retry amplification
Fallback   what to do when all retries fail
```

The full composition (resilience4j-style):

```
fallback(
  circuitBreaker(
    bulkhead(
      timeLimiter(
        retry(
          () -> remoteCall()
        )
      )
    )
  )
)
```

This is the production resilience stack. Skip any layer and you've created a failure mode.

---

## 11. Real-World Examples

### AWS SDK exponential backoff
AWS SDKs (boto3, AWS SDK for Java) include built-in retry with exponential backoff and decorrelated jitter, configured per service. Most engineers don't realize the SDK is already retrying — disable explicit application-level retries that double up.

### Google Cloud Storage client
Auto-retries with truncated exponential backoff (1s, 2s, 4s, ..., max 32s) plus jitter. Default 6 attempts. Honors `Retry-After`.

### Kubernetes API server clients
client-go uses exponential backoff with jitter on transient errors. Critical for controllers that watch resources and reconcile state.

### Postgres `pg_retry`, application-level patterns
For deadlock and serialization failures, applications retry the transaction up to 3–5 times with backoff. Standard idiom.

### Stripe API
Documented exponential backoff with jitter (1s base, max 60s). Idempotency keys make retries safe. Server-side retry-budget prevents amplification.

---

## 12. Common Mistakes / Anti-Patterns

- **No timeout at all.** The number-one production reliability bug.
- **Timeout much larger than needed.** A 60 s timeout on a 50 ms operation lets sick calls hang forever.
- **Catch-and-retry on every exception.** Including 4xx and authentication errors. Now you retry "you forgot the API key."
- **Retry on non-idempotent operations** without idempotency keys → duplicate effects.
- **No backoff** → 1000 callers hammer the dying downstream.
- **No jitter** → synchronized waves of retries.
- **No retry budget** → retry amplification cascades.
- **Per-call timeout only, no aggregate deadline** → user gave up; service keeps retrying.
- **Connect timeout absent or set to library default** (often 30 s) → DNS / TCP issues hang.
- **Retries during a circuit-open state** → no point.
- **Application retries on top of SDK retries** → exponential multiplication.
- **Polling without backoff** → "exponential backoff" pattern violated for slow-completion operations.
- **`Retry-After` header ignored** → downstream told you when to retry; you didn't listen.
- **Retries log nothing** → can't diagnose retry storms.
- **Long-running calls inside short timeouts** → the operation succeeded but the client thinks it failed.

---

## 13. Decision Rule

```
For every network call:
  ✓ timeout (connect + read + total)
  ✓ aggregate deadline (or inherit from caller)

For every retry decision:
  ✓ idempotent? (or has idempotency key)
  ✓ error is transient?
  ✓ have budget for another attempt?
  ✓ within the deadline?

For every retry attempt:
  ✓ exponential backoff
  ✓ full jitter
  ✓ honor Retry-After if present
  ✓ count this retry toward the global budget

For every operation that retries:
  ✓ paired with circuit breaker, bulkhead, fallback
  ✓ metrics on retry rate, retry budget, deadline-expired
```

---

## 14. Cheat Card

```
PURPOSE     Survive transient remote-call failures without
            destroying the downstream or hanging the caller.

THE THREE
  TIMEOUT     cap wall-clock per attempt (connect, read, total)
  RETRY       bounded count, only on idempotent + transient errors
  BACKOFF + JITTER  exponential + random to decorrelate retries

DEADLINE    end-to-end budget for the user request; propagates
            across hops; abandons stale work

WHEN TO RETRY
  ✓ network exception · timeout · 5xx · 429 · gRPC UNAVAILABLE
  ✗ 4xx (client's fault) · 401/403/404 · validation · auth

JITTER FLAVORS
  full        rand(0, base × 2^n)        — usually best
  equal       half deterministic + half random
  decorrelated  rand(base, prev × 3)     — AWS's choice

RETRY BUDGET   cap retries at ~10% of base traffic; protects
              against fleet-wide amplification

AMPLIFICATION  N retries × P fail-rate = (1 + N×P)× load
              N=10, P=10% → 2× load on a sick downstream

COMPOSE WITH   Bulkhead → TimeLimiter → Retry → CircuitBreaker →
              Fallback. The full resilience stack.

PITFALLS    No timeout · retries on non-idempotent ops · no backoff ·
            no jitter · no retry budget · per-call timeout only ·
            DNS / TCP timeouts default-long · SDK + app double retry ·
            ignoring Retry-After · catching all exceptions

RULE        Every network call has a timeout. Every retry has
            backoff and jitter. Every retry budget is bounded.
            Idempotency makes retries safe. Without these, small
            failures become outages.
```

---

## 15. Resources

### Books
- *Release It!* — Michael Nygard. Timeouts and retries chapters are foundational.
- *Site Reliability Engineering* — Google. Addressing cascading failures and retry budgets.
- *Designing Data-Intensive Applications* — Martin Kleppmann. Chapter 8 on the trouble with distributed systems.

### Articles
- "Exponential Backoff and Jitter" — Marc Brooker, AWS Architecture Blog: <https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/>
- "Timeouts, Retries, and Backoff with Jitter" — AWS Builders' Library: <https://aws.amazon.com/builders-library/timeouts-retries-and-backoff-with-jitter/>
- "The Tail at Scale" — Dean & Barroso. Hedged requests, tail-latency mitigation.
- "Avoiding insidious failure modes" — Will Tarreau / HAProxy notes.
- "Stripe API Retries" — Stripe API docs (idempotency + retry guidance).
- "gRPC Retry Design" — gRFC A6: <https://github.com/grpc/proposal/blob/master/A6-client-retries.md>

### Videos
- "Avoiding Cascading Failures" — Google SRE talks.
- ByteByteGo — "Retry Patterns and Exponential Backoff."
- "Tail at Scale" — Jeff Dean, Strange Loop.

### Tools
- **resilience4j** — JVM retry + circuit breaker + bulkhead.
- **Polly** — .NET retry policies.
- **Tenacity** — Python retry library.
- **retry-go** — Go retry library.
- **failsafe-js / Cockatiel** — JS/TS resilience.
- **AWS SDK / gRPC / Google Cloud SDK** — built-in retry policies.
- **Envoy / Istio** — service-mesh retries with budgets.

### Adjacent reading
- [Idempotency](../03-apis/idempotency.md)
- [Idempotent Operations & Retries →](./idempotency-retries.md)
- [Circuit Breaker Pattern](./circuit-breaker.md)
- [Bulkhead Pattern →](./bulkhead.md)
- [Backpressure](../10-scalability/backpressure.md)
- [Rate Limiting](../03-apis/rate-limiting.md)
- [Tail Latency & p99](../16-performance/tail-latency.md)
- [Synchronous vs Asynchronous Communication](../03-apis/sync-vs-async.md)

---

*Previous:* [← Circuit Breaker Pattern](./circuit-breaker.md)  |  *Next:* [Bulkhead Pattern →](./bulkhead.md)
