# Backpressure

> **TL;DR** — **Backpressure** is the mechanism by which a slow consumer tells a fast producer to slow down — directly, indirectly, or by explicit signal. Without it, fast producers overwhelm slow consumers, queues grow unbounded, memory fills, latency spirals, and the whole pipeline collapses under load it was supposed to absorb. Backpressure is the difference between a system that **degrades gracefully** under overload and one that **fails catastrophically**. The mechanisms range from **TCP flow control** (built in) through **bounded queues**, **rate limiting**, **load shedding**, and **explicit credit-based protocols** (gRPC streaming, Reactive Streams, Kafka consumer pull). The goal is never to "have no backpressure" — every system has backpressure somewhere. The goal is to make it **explicit, predictable, and applied at the right boundary**, not as an emergent collapse.

---

## 1. The Idea

```
Without backpressure:
   producer ──fire-and-forget──► [unbounded queue] ──► slow consumer
                                       │
                                       ▼ grows forever
                                  OOM / GC death / latency spike / cascade

With backpressure:
   producer ◄──slow down/wait/reject──┐
        │                              │
        ▼                              │
      [bounded queue] ──► consumer ───┘
```

Backpressure says: "the system is full; don't send more." It can be communicated four ways:
1. **Block the producer** (synchronous, in-process: a full queue makes `put` wait).
2. **Reject explicitly** (HTTP 503, 429, gRPC `UNAVAILABLE`).
3. **Drop / sample** (best-effort streams: shed least-important traffic).
4. **Pull-based / credit-based** (consumer asks for N more; producer never sends more than asked).

Every production system uses at least one of these at every boundary that buffers work.

---

## 2. Why Backpressure Matters

Without backpressure, a pipeline under sustained overload exhibits a predictable failure shape:

```
   inbound rate ── steady high ─────────────────────────────────
   capacity ─── below inbound ─────────────────────────────────

       1. Queue grows                       (memory pressure)
       2. Latency for items at queue tail grows
       3. Garbage collection thrashes        (Java/JS apps)
       4. CPU spent on overhead, not throughput
       5. Connections / file descriptors exhaust
       6. Crashes / OOM kills
       7. On restart, the queue is gone; data lost
       8. Upstream retries fire; effective load 2–10×
       9. CASCADE: failure propagates outward
```

The collapse is non-linear. A system holding 1k items handles them fine. The same system with 1M items in queue doesn't process them 1000× more slowly — it usually processes them 0× more slowly (OOM) or in the wrong order.

**A bounded queue with backpressure converts catastrophic failure into latency or rejection.** Latency you can measure. Rejection you can route around. Catastrophe you can only post-mortem.

---

## 3. Where Backpressure Already Exists (TCP)

The original backpressure mechanism in computing: **TCP's sliding window**.

```
   sender ────[window full]───► receiver
        │                          │
        │   ACKs with new window   │
        │ ◄────────────────────────┘
        │
        ▼
   send more, bounded by advertised window
```

TCP receivers advertise a **receive window** in each ACK. If the receiver's buffer fills (slow application reading from the socket), it shrinks the window and the sender stops. The application up the stack feels the slowdown as `write()` blocking or returning `EAGAIN`.

Anywhere you use TCP, this is your default backpressure mechanism — but it only stretches as far as the socket. Once data is in your application's in-process queue, TCP has no opinion.

HTTP/2 and HTTP/3 layer **stream-level** flow control on top, allowing fine-grained backpressure across multiplexed streams within one connection.

---

## 4. The Bounded Queue — Your Best Friend

```
   producer ──put(item)──► [bounded queue, capacity = K] ──► consumer

   Behavior when full:
     1. Block producer until space free
     2. Reject (return error / NACK)
     3. Drop oldest
     4. Drop newest
     5. Sample / coalesce
```

A bounded queue **forces** backpressure. The choice of what to do when full is the design decision:

| Policy | When to use | Trade-off |
|---|---|---|
| Block | In-process, low-fanout, OK to slow producer | Producer thread waits; cascades if upstream not also backpressured |
| Reject | RPC, HTTP, async APIs | Producer learns and can route around / retry |
| Drop oldest | Metric streams, dashboards | Freshness over completeness |
| Drop newest | Audit logs, ordered events (rarely) | Loss at tail; bad if order matters |
| Sample | High-volume telemetry | Statistical, not exact |
| Coalesce | Idempotent updates, latest-value sensors | Acceptable if "latest" matters most |

The single biggest backpressure bug in real systems is **unbounded queues** — `LinkedBlockingQueue` with no capacity, `Channel(Unlimited)`, `chan interface{}` without buffer cap, a Kafka topic with retention but no consumer-side bound. These are time bombs.

---

## 5. Backpressure Mechanisms in Practice

### 5.1 Synchronous blocking
A thread calls `queue.put(item)` and blocks if the queue is full. Backpressure propagates up the call stack — the caller of the producer blocks too. Pure and reliable; doesn't scale to async / many-connection workloads.

### 5.2 Rejection (429 / 503)
HTTP servers reject when overloaded:
- **429 Too Many Requests** — rate-limited; client should slow down.
- **503 Service Unavailable** — overloaded; client should retry later with backoff.
- **gRPC `RESOURCE_EXHAUSTED` / `UNAVAILABLE`** — same shapes.

Includes a `Retry-After` header to coordinate retries. See [Rate Limiting →](../03-apis/rate-limiting.md) and [Retry, Timeout, Backoff →](../11-reliability/retry-timeout-backoff.md).

### 5.3 Load shedding
At the LB or application edge: if queue depth / latency exceeds threshold, reject the lowest-priority work. Netflix's **Concurrency Limits** library and Google's **Adaptive Load Shedding** are canonical implementations.

```
   if in_flight > target_in_flight or queue_depth > limit:
       reject(request) with 503
```

The clever bit is the threshold being **dynamic** — measured from actual latency, not a hardcoded limit.

### 5.4 Credit-based / pull-based protocols
Consumer dictates pace. Producer cannot send unless granted credits.

- **Reactive Streams** (`Publisher`/`Subscriber`/`request(n)`) — the JVM standard.
- **gRPC streaming** — flow-controlled via HTTP/2.
- **Kafka consumer poll** — consumers explicitly ask for N records.
- **AMQP / RabbitMQ prefetch** — broker sends only `prefetch_count` unacked messages.

This is the gold standard for streaming systems: the consumer's actual rate is the only thing producing data flows at.

### 5.5 Token / Leaky bucket
Rate limiters in front of the producer — producers earn or are granted tokens; no tokens, no send. See [Rate Limiting →](../03-apis/rate-limiting.md).

### 5.6 Circuit breaker
After enough failures or timeouts, the upstream "trips" and rejects calls fast without trying the downstream. This is backpressure as **time** rather than space: stop sending entirely for a window. See [Circuit Breaker →](../11-reliability/circuit-breaker.md).

### 5.7 Probabilistic / priority shedding
At the LB: drop low-priority requests with increasing probability as load rises; keep high-priority traffic. Often used in conjunction with **request priority headers** or per-tenant SLOs.

---

## 6. Pull vs Push

A defining axis of streaming systems:

```
PUSH                              PULL
────                              ────
producer dictates rate            consumer dictates rate
backpressure: explicit signal     backpressure: implicit (don't ask)
faster steady-state               sometimes higher latency
risk: producer overruns consumer  risk: consumer too slow → backlog
                                  
HTTP/2 streams                    Kafka consumer.poll
WebSockets (default push)         RabbitMQ basic.get
SSE                               Reactive Streams request(n)
Webhooks                          gRPC streaming (with flow control)
```

**Push** is simpler when consumers can keep up. **Pull** is safer when they can't. **Push with explicit backpressure** (flow control, request(n), prefetch limits) is the modern compromise that gives both efficiency and safety.

Kafka's choice of pull-based consumers was a deliberate response to push-based brokers (early RabbitMQ, ActiveMQ) overrunning consumers. The trade-off: consumer-side polling overhead and tuning of `fetch.min.bytes` / `fetch.max.wait.ms`.

---

## 7. The Reactive Streams Protocol

The JVM standard for async backpressure. Four interfaces, one rule:

```
Subscriber.onSubscribe(Subscription)
Subscriber.onNext(item)
Subscriber.onError(throwable)
Subscriber.onComplete()

Subscription.request(n)    ← consumer pulls
Subscription.cancel()      ← consumer stops
```

The rule: **the publisher MUST NOT send more items than the subscriber has requested.**

Implementations: Project Reactor, RxJava, Akka Streams, Java 9 `java.util.concurrent.Flow`. Adopted by Spring WebFlux, R2DBC, MongoDB reactive driver, many others.

Same idea appears as:
- **async iterators** with backpressure in JavaScript (`AsyncIterator`, Node Streams).
- **Trio / asyncio Channels** in Python.
- **Go channels** with bounded capacity (`make(chan T, N)`).
- **Tokio mpsc channels** in Rust with bounded capacity.

---

## 8. End-to-End Backpressure

Backpressure works only if it propagates **all the way back to the source of traffic**. A break anywhere along the chain means an unbounded queue forms there.

```
   LB ──► API ──► async queue ──► worker ──► database
   ▲       ▲           ▲             ▲           ▲
   │       │           │             │           │
   └── backpressure must flow back to the load generator

   If ANY edge buffers without bound, the unboundedness lives
   there and the chain has no real backpressure.
```

The right shape:
1. **DB has bounded connections / queue.**
2. **Worker pool size is bounded; can't grab work it can't process.**
3. **Async queue is bounded; producer (API) blocks or rejects.**
4. **API enforces concurrency limits / rate limits.**
5. **LB rejects (503) when API saturates.**
6. **Client backs off on 503 / 429.**

Skip any of those and you've moved the failure from "rejected at the edge" to "blew up in the middle."

---

## 9. Worked Example — A Web Service with Async Workers

```
   ┌─────────┐         ┌─────────┐         ┌─────────┐
   │  ALB    │─── ► ──│   API   │─── ► ──│  Queue  │
   └─────────┘         └─────────┘         └─────────┘
                                              │
                                              ▼
                                          ┌─────────┐    ┌────────┐
                                          │ Workers │───►│   DB   │
                                          └─────────┘    └────────┘
```

A reasonable backpressure design:

```
ALB:        target response time SLO; reject above 200 inflight per node (503)
API:        token-bucket rate limit 100 req/s per API key
            in-flight limit per node based on Concurrency Limits library
            queue.put with timeout → returns 503 if can't enqueue in 50 ms
Queue:      bounded at 10,000 messages; SQS visibility timeout enforces order
Workers:    pool of N; each processes one message at a time
            DB connection pool of M; semaphore around DB calls
DB:         connection limit 100; statement_timeout 5 s
Clients:    on 503/429, exponential backoff with jitter
```

Result: at sustained 5× capacity, requests are rejected at the API edge with 503. Latency for accepted requests stays bounded. No queue grows without bound. The system **stays up**.

Without those bounds: queue grows → workers fall behind → API timeouts → ALB marks node unhealthy → fleet shrinks → remaining nodes overwhelmed → cascade.

---

## 10. Worked Example — Kafka Consumer Lag

Kafka consumers pull, so backpressure is implicit: if a consumer is slow, lag grows. Lag itself is the backpressure signal that other components observe.

```
   Producer ─► [partition log] ─────► Consumer
                  ▲                    │
                  │                    ▼
              retention                slow processing
              hard cap (time/bytes)    causes lag
                  │
                  └── eventually drops oldest if retention exceeded
```

Mitigations:
- **Scale consumers horizontally** (within partition count).
- **Scale processing capacity** (KEDA on consumer lag).
- **Apply outer backpressure** — if downstream of the consumer can't keep up, slow the consumer's commits or fail closed.
- **Drop-and-log** for non-critical streams (telemetry).
- **Quotas** — Kafka producer quotas limit the upstream when consumers fall behind.

The Kafka design philosophy: **lag is a measurable, recoverable form of backpressure**. Watch it, scale it, but expect to live with some.

---

## 11. Backpressure vs Buffering — The Confusion

Beginners confuse "I have a buffer, so I have backpressure." A buffer **delays** the moment of truth; it doesn't apply pressure. Backpressure happens **when the buffer is full**, by the chosen full-policy.

A common pattern that looks safe but isn't:

```
   producer ──► [unbounded async queue] ──► consumer

   "We have a queue, so spikes are smoothed."
```

Smoothed until the spike outlasts the queue's RAM. Then the entire process dies and you lose everything in the queue.

**Buffers must be bounded.** Backpressure must be defined for when the bound is hit.

---

## 12. Common Backpressure Failures

### Cascade through retry amplification
Downstream slows → upstream retries (often without backoff) → effective load 2–10× → downstream fully collapses. The fix: **bounded retries + exponential backoff + jitter + circuit breakers**. Backpressure must include "don't make it worse."

### Synchronous chain with unbounded thread pool
Java app with `ExecutorService.newCachedThreadPool()` — accepts any work, spawns threads forever. Memory dies before CPU does.

### Async chain with unbounded channels
Go `chan T` with no buffer (sync) is fine; `chan T` with `cap N` is bounded; `make(chan T)` in some patterns... read carefully. Languages with first-class channels make it easy to introduce unbounded buffers accidentally.

### Slow consumer with auto-ack
RabbitMQ / SQS with auto-ack: messages delivered to a consumer that crashes mid-process — and you've already acknowledged. Bound the in-flight set explicitly.

### Backpressure stops at the gateway, not the client
API returns 503; client retries immediately without backoff; effective load unchanged. Backpressure must reach the **load source**.

### One client consuming 80% of capacity
Per-client / per-tenant rate limits are needed. Otherwise one noisy neighbor backpressures everyone else by saturating shared capacity.

### Forgot the timeout
A `queue.put()` that blocks forever **is** backpressure — but if the producer is an HTTP request handler, the user's request stalls until the LB times out. Better: `put(timeout=200ms)` then reject with 503.

### Sampling without telling anyone
Dropping is backpressure too — but if your metric pipeline silently drops at 10× load, your dashboards lie about the spike. Always meter what was dropped.

---

## 13. Backpressure Patterns You'll See

### Token bucket / leaky bucket
Generic rate limiting; backpressure as a throttle. See [Rate Limiting →](../03-apis/rate-limiting.md).

### Concurrency limits (Little's Law)
Bound in-flight count rather than rate: `concurrency_limit = target_throughput × target_latency`. Self-adapts to actual capacity. Netflix's library is the reference implementation.

### Adaptive load shedding
Monitor response time vs target; reject proportionally as latency rises. Implemented in Envoy adaptive concurrency, Linkerd, Google internal systems.

### Bulkhead
Isolate work pools so one slow downstream doesn't drain the global thread pool. See [Bulkhead Pattern →](../11-reliability/bulkhead.md).

### Outbox / store-and-forward
Producer writes to durable storage instead of remote endpoint; a separate process drains. The durable store provides the buffer, with bounded retention.

### Priority queues
Multiple queues per priority; high-priority preempts low. Under load, low-priority gets dropped first.

### Adaptive sampling
Telemetry pipelines sample down dynamically: at 1× load, send everything; at 10× load, send 10%. Honeycomb, Datadog, OpenTelemetry samplers all do this.

---

## 14. Diagnostic Questions

When debugging an overload, ask:

```
1. Where is the unbounded buffer?
   - Find it; bound it.

2. Where does backpressure stop?
   - Trace back from the bottleneck to the load source.
   - Anywhere the signal doesn't propagate, queues grow.

3. What is each component's "full" policy?
   - Block? Reject? Drop? Sample?
   - Does it match the workload semantics?

4. Are retries amplifying load?
   - Add backoff + jitter, circuit breakers.

5. What signals do we have?
   - Queue depths, in-flight counts, latency, rejection rates.
   - You can't tune what you can't see.

6. Are clients well-behaved on rejection?
   - Honor 429/503, back off, jitter.
```

---

## 15. Common Mistakes / Anti-Patterns

- **Unbounded queues anywhere.** The single most common bug in async systems.
- **Backpressure that stops at the LB.** Clients keep hammering with no slowdown.
- **Retries without backoff or jitter.** Multiplies load on a struggling downstream.
- **Auto-ack with slow consumers.** Lose work on crashes.
- **`Thread.sleep(retry_delay)` on shared threads.** Burns the thread pool; should be async or with circuit breaker.
- **Treating buffering as backpressure.** Buffers delay; bounded buffers + full-policy is backpressure.
- **One queue, all priorities.** First overloaded thing is critical work behind log uploads.
- **Throttling on average load.** Tail latency goes wrong long before average does.
- **No per-tenant / per-key limits.** One noisy neighbor takes the whole system down.
- **Silent dropping.** Telemetry without "I dropped X" metrics misleads operators.
- **Backpressure at the wrong layer.** Slowing the worker doesn't help if the API still accepts work without bound.
- **Blocking inside async event loop.** Backpressure that blocks the wrong thread freezes the whole node (Node.js, Tokio, Netty).

---

## 16. Decision Rule

```
For every queue, channel, buffer, or "we'll catch up later" mechanism:

  - Is it bounded?           If no → bound it.
  - What happens when full?  If undefined → define it.
  - Does the signal flow     If no → fix that path.
    back to the load source?

For every retry path:

  - Bounded retry count?     Yes.
  - Backoff with jitter?     Yes.
  - Circuit breaker around   Yes.
    the downstream?

For every shared resource:

  - Per-tenant / per-key     Yes.
    limits?
  - Priority isolation?      If mixed workloads.

For every overload scenario:

  - Reject with a clear      429 / 503 + Retry-After.
    signal?
  - Meter the rejections.    Yes — both for ops and for the
                              autoscaler / SRE.
```

---

## 17. Cheat Card

```
PURPOSE     Slow / reject / drop fast producers when slow consumers
            can't keep up. Convert catastrophic overload into bounded
            latency or explicit rejection.

MECHANISMS  Block (synchronous queue)
            Reject (429, 503, gRPC UNAVAILABLE)
            Drop oldest / newest / sample / coalesce
            Pull / credit-based (Reactive Streams, Kafka, gRPC)
            Rate limit (token / leaky bucket)
            Circuit breaker (time-based shed)

LAYERS      TCP flow control     (built-in transport)
            HTTP/2-3 stream      (per-stream)
            App rate limiter     (per-client)
            App concurrency lmt  (in-flight bound)
            Bounded queue        (between stages)
            Worker pool size     (bounded fan-out)
            Connection pool      (downstream bound)

END-TO-END  Backpressure must reach the LOAD SOURCE. Any
            unbounded buffer in the chain is a time bomb.

ASYMMETRY   Latency you can measure. Rejection you can route.
            Catastrophe you can only post-mortem.

PITFALLS    Unbounded queues · retries without backoff · auto-ack
            with slow consumers · backpressure stopping at the LB ·
            no per-tenant limits · silent drops · sleep on shared
            threads · buffering ≠ backpressure

RULE        Every buffer is bounded; every bound has a policy;
            every policy is observable; every retry has limits;
            every overload has a rejection signal. Build for the
            bad day.
```

---

## 18. Resources

### Books
- *Release It!* — Michael Nygard. The book on cascading failure, backpressure, circuit breakers, and overload patterns.
- *Designing Data-Intensive Applications* — Martin Kleppmann. Chapters on streaming and reliable messaging.
- *Reactive Design Patterns* — Roland Kuhn. Reactive Streams in depth.
- *Site Reliability Engineering* — Google. Load-shedding and overload chapters.

### Articles
- "Adaptive Concurrency Limits" — Netflix tech blog: <https://netflixtechblog.medium.com/performance-under-load-3e6fa9a60581>
- "Handling Overload" — Google SRE Book chapter.
- "Reactive Streams Specification" — <https://www.reactive-streams.org/>
- "Backpressure Explained" — Jay Kreps / Confluent on Kafka pull model.
- "Hystrix How It Works" — Netflix archive.
- "Envoy Adaptive Concurrency" — Envoy docs.
- "Load Shedding at Stripe" — Stripe engineering posts.
- "Coordinated Omission" — Gil Tene, tangentially related (overload distorts measurements).

### Videos
- "Stop Rate Limiting! Capacity Management Done Right" — Jon Moore, conference talk.
- "Adaptive Concurrency Limits at Netflix" — talks by Eran Landau.
- ByteByteGo — "Backpressure" overview.
- "The Hardest Problem in Distributed Systems" series — many touch on backpressure.

### Tools
- **Netflix concurrency-limits** — JVM library for adaptive limits.
- **Envoy** — adaptive concurrency filter, rate-limit filter.
- **resilience4j** / **Hystrix (archived)** — JVM resilience patterns.
- **Project Reactor / RxJava** — Reactive Streams implementations.
- **kafka-consumer-groups** — view lag, the canonical Kafka backpressure signal.
- **Linkerd / Istio** — service mesh with retries, timeouts, circuit breakers.

### Adjacent reading
- [Rate Limiting](../03-apis/rate-limiting.md)
- [Circuit Breaker Pattern](../11-reliability/circuit-breaker.md)
- [Retry, Timeout, and Exponential Backoff](../11-reliability/retry-timeout-backoff.md)
- [Bulkhead Pattern](../11-reliability/bulkhead.md)
- [Graceful Degradation](../11-reliability/graceful-degradation.md)
- [Hot Partition Problem](./hot-partitions.md)
- [Auto-Scaling](./auto-scaling.md)
- [Message Queues vs Pub/Sub vs Streams](../07-messaging/queue-vs-pubsub-vs-stream.md)
- [Kafka Deep Dive](../07-messaging/kafka.md)

---

*Previous:* [← Auto-Scaling (Horizontal Pod Autoscaler, AWS ASG)](./auto-scaling.md)  |  *Next:* [Geographically Distributed Systems (Multi-Region) →](./multi-region.md)
