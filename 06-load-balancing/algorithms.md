# Load Balancing Algorithms (Round Robin, Least Connections, IP Hash, Weighted, etc.)

> **TL;DR** — Picking a backend is the LB's core job. **Round-robin** rotates through targets — simple, fair for homogeneous pools, blind to load. **Weighted round-robin** lets you weight heavier servers more. **Least-connections** picks the backend with the fewest active connections — usually the best default for variable-duration HTTP requests. **Least-response-time / EWMA** picks the fastest backend over a sliding window — adapts to actual latency. **IP-hash / consistent hashing** pins a client (or key) to a specific backend — useful for affinity, sticky-cache patterns, and avoiding cache thrash. **Power-of-two-choices** picks two random backends and routes to the less loaded — astonishingly good fairness at almost no cost. **Random** is competitive at large pool sizes and is the cheapest. The decision drivers are: **homogeneity of backends**, **variability of request cost**, **need for affinity**, and **what state the LB can track**.

---

## 1. Why Algorithm Choice Matters

The wrong algorithm doesn't usually crash. It just distributes load unevenly, so some backends are overloaded (high p99, dropped requests) while others sit idle (wasted capacity).

```
Round-robin on heterogeneous backends:
  ┌─────────┬─────────┬─────────┐
  │ pod A   │ pod B   │ pod C   │
  │ slow    │ fast    │ fast    │
  └─────────┴─────────┴─────────┘
     equal share → A queues, p99 spikes, B/C idle

Least-connections on the same pool:
  ┌─────────┬─────────┬─────────┐
  │ pod A   │ pod B   │ pod C   │
  │ slow    │ fast    │ fast    │
  └─────────┴─────────┴─────────┘
     A holds fewer concurrent (because of length)? actually MORE.
     Hmm. Wait — slow pods accumulate connections. Least-conn
     skips them, sends more to fast pods. Self-balancing. 
```

The algorithm has to match the workload.

---

## 2. The Canon: Eight Algorithms You'll See

### 2.1 Round-Robin (RR)
Pick the next backend in rotation. Order: A, B, C, A, B, C, ...

```python
def pick(self):
    self.i = (self.i + 1) % len(self.backends)
    return self.backends[self.i]
```

**Pros**
- Trivial implementation.
- Perfectly fair on average for homogeneous pools with uniform request cost.
- No state besides the index.

**Cons**
- Ignores backend health beyond up/down.
- Ignores backend load.
- Ignores backend speed.
- If one backend is slow, RR keeps sending requests to pile up.

**When to use** — homogeneous pool, requests of roughly equal cost, low cardinality. Many simple setups start here and never need more.

### 2.2 Weighted Round-Robin (WRR)
Each backend has a weight. Heavier backends get more requests.

```
weights: A=1, B=2, C=3
order: A B B C C C A B B C C C ...
```

**Pros**
- Handles heterogeneous pools.
- Simple to reason about — a 2× weight means 2× traffic.

**Cons**
- Static weights need tuning. Real backends drift.
- Doesn't adapt to instantaneous load.

**When to use** — known heterogeneous hardware (e.g., bigger EC2 instances mixed with smaller), canary deployments (5% to canary, 95% to stable), gradual rollouts.

### 2.3 Least-Connections (LC)
Pick the backend currently handling the fewest active connections.

```python
def pick(self):
    return min(self.backends, key=lambda b: b.active_connections)
```

**Pros**
- Adapts to request cost. Slow requests accumulate connections; LC routes around them.
- Excellent default for HTTP APIs with mixed-cost endpoints.

**Cons**
- Requires the LB to track connection state per backend.
- "Connections" can lie: with HTTP/2 multiplexing, one connection = N streams; LB sees one but backend is busy.
- Equal-conn backends → tie-breaking matters.

**When to use** — HTTP/1.1 APIs with variable request durations. The default in HAProxy when you want better-than-RR.

### 2.4 Weighted Least-Connections (WLC)
LC with backend weights. Picks `argmin(active_conn / weight)`.

```
B  active=2 weight=2 → 1.0
A  active=3 weight=3 → 1.0
C  active=4 weight=4 → 1.0
all tied; LB picks the first
```

**Pros**
- Combines heterogeneity handling with load-awareness.

**Cons**
- Same downsides as LC. Tuning weights is manual.

### 2.5 Least-Response-Time / EWMA
Track a moving average of recent response times per backend. Pick the fastest.

Formula (exponentially-weighted moving average):
```
ewma_new = alpha * recent_latency + (1 - alpha) * ewma_old
```

Modern variants (P2C-EWMA, used by Linkerd2-proxy): pick two random backends, choose the one with lower EWMA. Avoids herding to the same backend and the resulting overload-recovery cycle.

**Pros**
- Adapts in real time to backend speed.
- Better than LC when "active connection count" misleads.
- Self-balances around degrading backends.

**Cons**
- EWMA noise can cause flapping.
- Cold-start: a new backend has no history; how do you score it?
- More state per backend.

**When to use** — variable backend performance, mature LBs (Envoy, Linkerd), where you want hot-spot avoidance.

### 2.6 IP-Hash (Source-IP Hash)
Hash the client's source IP, modulo pool size. Same client → same backend.

```python
def pick(self, client_ip):
    return self.backends[hash(client_ip) % len(self.backends)]
```

**Pros**
- Stateless affinity. Sticky sessions without cookies.
- Reproducible across LB instances if pool is identical.

**Cons**
- **NAT/CGNAT problem** — many users behind one corporate or carrier-grade NAT IP all land on one backend. Skew.
- Adding/removing a backend reshuffles ~1/N keys → cache misses everywhere. (Use **consistent hashing** to mitigate.)
- One client = one backend = no load distribution per user.

**When to use** — when you need affinity, the user pool is internet-broad, and you accept some skew. Better alternative: cookie-based affinity at L7, or consistent hashing.

### 2.7 Consistent Hashing (CH)
Hash both backends and keys onto the same ring (or use rendezvous / Maglev hash). Each key maps to the next backend clockwise. Adding/removing a backend only moves `1/N` of keys.

```
ring: ─A─────B───────C─────D─

key=42 → hash position → next on ring = B
```

**Pros**
- Stable mappings: adding a backend disrupts ~1/N keys, not all.
- Used in CDNs (cache locality), distributed caches (sharding), and L4 LBs (Maglev).
- "Sticky by key" without cookie tracking.

**Cons**
- Skewed when keys aren't uniform.
- Need replicas / virtual nodes (vnodes) to balance hash space.
- Adding/removing a backend still affects 1/N of cached state — cold misses.

**When to use** — sharded caches, request routing where backend memory matters (e.g., cache locality), Maglev/Katran-style LBs, sticky routing without cookies.

See [Consistent Hashing →](../04-databases/consistent-hashing.md) for the full theory.

### 2.8 Power-of-Two-Choices (P2C)
Pick **two backends at random**, route to the less loaded. Done.

```python
def pick(self):
    a, b = random.sample(self.backends, 2)
    return a if a.active_connections < b.active_connections else b
```

**Pros**
- Provably almost as good as picking the *best* backend.
- O(1) per request; tiny state.
- Robust to herd effects (no thundering toward the "best" backend).
- The maximum load is `O(log log N)` worse than the optimal (Mitzenmacher's theorem).
- Used by Netflix's Ribbon, Linkerd2, Envoy's `LEAST_REQUEST`, Twitter's Finagle.

**Cons**
- Needs accurate "load" signal per backend (conn count, EWMA, or in-flight requests).
- Two random reads per pick (negligible).

**When to use** — most modern production LBs choose this as the default load-aware algorithm. Strongly recommended over plain LC.

### 2.9 Random
Pick a backend uniformly at random.

```python
def pick(self):
    return random.choice(self.backends)
```

**Pros**
- Stateless. Trivial.
- Surprisingly competitive when N is large (>10) — load variance is `O(√N)`.
- No coordination needed across LB instances.

**Cons**
- Worst-case backend gets ~`log N / log log N` × average load (classic balls-and-bins).
- Doesn't adapt.

**When to use** — when you need zero-state load balancing and pool is large enough that variance doesn't matter. Or as a comparison baseline.

---

## 3. Comparison Table

| Algorithm | State needed | Adapts to load | Adapts to speed | Sticky | Cost | Default for |
|---|---|---|---|---|---|---|
| Round-robin | counter | ✗ | ✗ | ✗ | O(1) | Simple homogeneous |
| Weighted RR | weights | ✗ | ✗ | ✗ | O(1) | Known heterogeneity |
| Least-connections | conn count | ✓ | partial | ✗ | O(N) or heap | HTTP/1.1 default |
| Weighted LC | + weights | ✓ | partial | ✗ | O(N) | Mixed hardware |
| Least response time | EWMA | ✓ | ✓ | ✗ | O(N) | Mature L7 |
| IP-hash | none | ✗ | ✗ | by IP | O(1) | Cheap affinity |
| Consistent hashing | ring | ✗ | ✗ | by key | O(log N) | Cache routing |
| Power-of-2 | conn count | ✓ | partial | ✗ | O(1) | Modern default |
| Random | none | ✗ | ✗ | ✗ | O(1) | Stateless / large N |

---

## 4. The Real Winner: Power-of-Two-Choices

P2C deserves the spotlight. The math is elegant:

> Pick two random servers and route to the less loaded. The maximum load grows as `O(log log N)`. Picking one random server gives `O(log N / log log N)`. Picking the actual minimum gives `O(1)` but requires global state.

P2C captures almost all the benefit of picking-the-best at almost the cost of random. **It's the algorithm to start with in 2026.**

Variants in the wild:
- **Envoy `LEAST_REQUEST`** — picks 2 (configurable) and routes to the one with fewer in-flight.
- **Linkerd2-proxy P2C-EWMA** — picks 2 and uses EWMA latency.
- **Finagle (Twitter)** — uses P2C with various load signals.
- **HAProxy `random`** with a count parameter — implements P2C when count=2.

The single thing P2C doesn't solve is affinity. For that, use hashing.

---

## 5. Algorithms in Real LBs

### NGINX
- `round-robin` (default)
- `least_conn`
- `ip_hash`
- `hash <key>` with `consistent` flag (consistent hashing)
- `random two` (P2C, with optional method)
- `least_time` (commercial NGINX Plus only)

### HAProxy
- `roundrobin`
- `static-rr` (weighted RR without dynamic adjustment)
- `leastconn`
- `source` (IP hash)
- `uri`, `url_param`, `hdr` (consistent hash on URI / param / header)
- `random` and `random N` (P2C when N=2)
- `first` (first available — useful for active/standby)

### Envoy
- `ROUND_ROBIN`
- `LEAST_REQUEST` (P2C by default; `choice_count` configurable)
- `RING_HASH` (consistent hashing)
- `MAGLEV` (Maglev hashing — consistent + better distribution)
- `RANDOM`

Envoy is the gold-standard modern LB; the algorithms reflect current best practice.

### AWS ALB
- Round-robin (default for HTTP)
- Least outstanding requests (newer, default on some account types)

### AWS NLB
- Flow hash (5-tuple consistent hash)

### GCLB
- Round-robin
- Least-request
- Ring hash
- Maglev hash

The default in any modern LB is moving from RR → P2C / least-request. Plain RR is fading.

---

## 6. Affinity Strategies (Affinity vs Algorithm)

Affinity = "same client / key → same backend." Several layers:

| Where | How |
|---|---|
| L4 | IP-hash, consistent hashing on 5-tuple |
| L7 — cookies | LB injects a cookie; subsequent requests carry it; LB routes by cookie value |
| L7 — header | Hash on a custom header (`X-Tenant-Id`) |
| L7 — body / param | Hash on URI path or query param |
| Client-side | Client library hashes locally (Redis Cluster, gRPC client-side LB) |

For real session affinity in HTTP, cookie-based is the winner. IP-hash is too coarse with NAT. Consistent hashing on a stable header (tenant id, user id) gives both routing intelligence and cache-locality benefits.

See [Sticky Sessions →](./sticky-sessions.md).

---

## 7. Picking by Scenario

```
homogeneous pods, uniform request cost
  → round-robin (or P2C — overkill but cheap)

heterogeneous pods (different CPU/RAM/specialized)
  → weighted RR or weighted P2C

HTTP/1.1, mixed-duration requests
  → least-connections or P2C-LR (default in many LBs)

HTTP/2 / gRPC at L7
  → P2C-LR with per-request balancing
    (not per-connection — that pins streams)

variable backend speed, want adaptive
  → P2C-EWMA (least response time)

cache-locality / shard routing
  → consistent hash on the key
    (Maglev hash if you have it; ring hash otherwise)

sticky session for stateful service (legacy)
  → cookie-based or consistent hash on user id
    (NOT IP-hash unless you've thought about NAT)

ultra-low-state, huge pool
  → random or P2C
```

---

## 8. Pitfalls Specific to Each Algorithm

### RR pitfalls
- One slow backend keeps getting requests → queue → timeout.
- Mitigation: combine with outlier ejection (auto-eject backends with high error rate or latency).

### Least-connections pitfalls
- HTTP/2 multiplexing — connection count != work in progress.
- Slow start: a fresh backend has 0 connections, gets a flood. Use slow-start config.
- Counts can stick if connections aren't closed cleanly.

### EWMA pitfalls
- Cold-start problem. Score newcomers with neutral or pool average.
- Noisy at low traffic. Add minimum sample count before trust.
- Stale samples decay slowly. Half-life matters; tune it.

### IP-hash pitfalls
- NAT skew.
- Adding backend = reshuffle.
- IPv6 addresses are per-device, so less skew there — though carrier IPv6 still NATs.

### Consistent hash pitfalls
- Non-uniform keys → skewed. Add virtual nodes.
- Hash collisions in small rings.
- Maglev hash is dramatically better than naive ring hash for evenness.

### P2C pitfalls
- The load signal matters. Conn count is okay; EWMA is better; queue depth is best if you have it.
- Lock contention if implemented naively across many threads.

---

## 9. Slow-Start

A newly added backend has zero connections and zero history. Naive LC/P2C floods it. Fix: **slow-start** — gradually ramp up the share of traffic to a new backend over N seconds. NGINX, HAProxy, and Envoy all support this.

```nginx
upstream app {
    least_conn;
    server 10.0.0.1 slow_start=30s;
}
```

Critical when:
- Backends have a JIT warmup (Java).
- Backends populate a cache from cold (any caching system).
- Backends do non-trivial initialization.

---

## 10. Outlier Ejection (Adaptive Health)

Even with a smart algorithm, a backend can become slow or fail intermittently. **Outlier ejection** (Envoy term; "automatic banning" elsewhere) ejects a backend from the pool after observing too many errors or high latency, then probes it later to reinstate.

Configuration (Envoy):
```yaml
outlier_detection:
  consecutive_5xx: 5
  base_ejection_time: 30s
  max_ejection_percent: 50
  interval: 10s
```

This is **passive health checking** — observe live traffic. Combined with active health checks, it catches problems active probes miss.

See [Health Checks →](./health-checks.md).

---

## 11. Worked Example: Choosing for a Microservice

Service: 20 stateless pods, HTTP/2 + gRPC, mixed request durations (50 µs to 200 ms).

**Bad choice**: round-robin
- Long-running requests pile up randomly on some pods → p99 spikes.

**Better**: least-connections
- HTTP/2 conn count misleads. Some pods have 10 connections × 50 streams each.

**Best**: P2C with least-request *at the stream level*
- The LB tracks active streams (not connections) per backend. Picks the less-loaded of two random.
- Add outlier ejection for slow / erroring pods.
- Add slow-start for newly-rolled pods.

Envoy default config with `LEAST_REQUEST` does exactly this. The result: smooth p99 across the pool, automatic accommodation of slow pods, clean rollouts.

---

## 12. Common Mistakes

- **Plain round-robin in production for varied workloads.** You'll discover this when one pod's p99 is 10× the others'.
- **IP-hash with NAT users.** All corporate users hash to one pod.
- **Least-connections without slow-start.** Newly-added backends get flooded.
- **No outlier ejection.** A flaky backend keeps getting traffic forever.
- **L4 LB for gRPC.** Per-stream balancing impossible — load skews.
- **Tuning weights once and forgetting.** Hardware drifts; static weights become wrong.
- **Consistent hash without virtual nodes.** Uneven distribution; one node carries 30%.
- **Choosing the "best" algorithm without observing.** Measure, then tune.

---

## 13. Cheat Card

```
ROUND-ROBIN         simplest; homogeneous; no load awareness
WEIGHTED RR         known heterogeneity; static weights
LEAST-CONN          adapts to durations; HTTP/1.1 default
LEAST-RESPONSE      EWMA; adapts to speed
IP-HASH             cheap affinity; NAT-blind
CONSISTENT HASH     cache locality; key-stable
POWER-OF-TWO        modern default; tiny cost, near-optimal
RANDOM              stateless; competitive at large N

WHEN TO USE
  RR / WRR          uniform pool, simple
  LC / WLC          variable durations, HTTP/1.1
  Least-response    mature L7 mesh, variable backends
  IP-hash           cheap affinity (rare)
  Consistent hash   sharded cache, key-stable routing
  P2C               default for new systems
  Random            stateless, very large pool

ALWAYS ADD          outlier ejection, slow-start, health checks

PITFALLS            NAT + IP-hash; HTTP/2 + LC; static weights;
                    no slow-start; no outlier ejection

RULE                Start with round-robin if your pool is uniform.
                    Move to P2C the moment it isn't.
```

---

## 14. Resources

### Books
- *Site Reliability Engineering* — Google SRE Book.
- *Probability and Computing* — Mitzenmacher & Upfal (the math behind power-of-two).

### Papers
- "The Power of Two Random Choices: A Survey of Techniques and Results" — Mitzenmacher.
- "Maglev: A Fast and Reliable Software Network Load Balancer" — Eisenbud et al., NSDI 2016 (the Maglev hash algorithm).
- "Consistent Hashing and Random Trees" — Karger et al. (the original 1997 paper).

### Documentation
- **NGINX upstream module**: <https://nginx.org/en/docs/http/ngx_http_upstream_module.html>
- **HAProxy load balancing**: <https://www.haproxy.com/documentation/haproxy-configuration-tutorials/load-balancing/>
- **Envoy load balancers**: <https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing>

### Articles
- "Beyond round-robin" — Google Cloud blog.
- "Algorithm choices in production load balancers" — Linkerd / Finagle / Envoy team writeups.
- "The 99% of consistent hashing" — by the author of jump consistent hash.

### Videos
- ByteByteGo — "Top 6 Load Balancing Algorithms".
- Hussein Nasser — "How load balancers pick servers".

### Tools
- **HAProxy**, **NGINX**, **Envoy**, **Traefik** — all the algorithms.
- **wrk**, **vegeta**, **ghz** — load-test to compare distributions.

### Adjacent reading
- [Load Balancer Fundamentals →](./load-balancer-basics.md)
- [L4 vs L7 →](./l4-vs-l7.md)
- [Health Checks →](./health-checks.md)
- [Sticky Sessions →](./sticky-sessions.md)
- [Consistent Hashing →](../04-databases/consistent-hashing.md)
- [Sharding & Partitioning →](../04-databases/sharding-partitioning.md)

---

*Previous:* [← Layer 4 vs Layer 7 Load Balancing](./l4-vs-l7.md)  |  *Next:* [Health Checks & Failover →](./health-checks.md)
