# Why Cache? Cache Hierarchy

> **TL;DR** — A **cache** is a smaller, faster store that holds copies of data so you don't have to recompute or refetch from the slow path. You cache because the speed gap between layers (CPU register → L1 → RAM → SSD → network → disk → cross-region) is enormous, and because most workloads are **skewed**: a tiny fraction of items get the majority of requests. Caching is the single highest-leverage performance tool in distributed systems, but every cache adds two new problems — **staleness** and **invalidation**. The art is putting the cache as close to the consumer as possible without lying to them for too long.

---

## 1. Why Caches Exist

Computers are built on a stack of progressively slower, larger stores. Each layer caches the layer below it.

```
CPU register     ~0.3 ns
L1 cache         ~1 ns
L2 cache         ~3 ns
L3 cache         ~10 ns
RAM              ~100 ns
NVMe SSD         ~25 µs
SATA SSD         ~100 µs
Same-DC network  ~500 µs
Spinning disk    ~5 ms
Cross-region RTT ~80 ms
```

The numbers span eight orders of magnitude. Every layer that exists, exists because the cost of going to the next layer is unacceptable in the common case. A **cache** is just a faster store, with less capacity, holding a hot subset of a slower one.

The same pattern repeats above the OS:
- The CPU caches RAM.
- The OS page cache caches disk.
- Your application caches the database.
- A CDN caches your origin.
- The browser caches the CDN.

You don't escape caching by going up the stack. You add layers.

---

## 2. Why Caching Works: Locality

Caches work because workloads are **not uniform**. Three flavors of locality:

- **Temporal locality** — if you accessed something, you'll probably access it again soon. The tweet you just posted will be read 10,000 times in the next minute.
- **Spatial locality** — if you accessed `X`, you'll probably access `X+1`. The next row in the table, the next byte of the file.
- **Skew** — a small fraction of keys account for a huge fraction of accesses. The 80/20 rule, often closer to 99/1. Top YouTube videos. Trending hashtags. The home page of `nytimes.com`.

Without skew, caches don't help. A perfectly uniform random workload over a billion keys hits a cache holding a million keys at a 0.1% hit rate. With Zipfian skew, the same cache hits 90%+. This is why caching is so effective in practice — real workloads are *extremely* skewed.

```
                    request frequency
                    ▲
                    │  ╲
                    │   ╲
                    │    ╲___
                    │        ╲___
                    │            ╲___________
                    └─────────────────────────►  keys, sorted
                       hot tail (the 1%)
```

If your workload doesn't look like this, caching is the wrong tool. Use indexing, denormalization, or compression instead.

---

## 3. What Caching Buys You

Four real, measurable wins:

| Benefit | What it means |
|---|---|
| **Lower latency** | RAM hits in 100 ns, your DB in 1 ms. 10,000× faster reads. |
| **Higher throughput** | A Redis node handles ~100k ops/sec on commodity hardware; a typical Postgres query handles ~5–10k QPS. |
| **Lower load on origin** | Your DB sees only the misses. A 95% cache hit rate reduces DB load by 20×. |
| **Lower cost** | RAM is more expensive per GB than disk, but a small RAM cache offloading huge DB cost wins easily. |

And the secondary wins:
- **Availability** — if the cache holds the hot set, you can survive origin downtime for a while.
- **Decoupling** — the cache absorbs spikes the origin can't.
- **Cost predictability** — pay-per-request databases (DynamoDB, Spanner) get expensive fast without a cache.

---

## 4. The Hierarchy

Caches form a hierarchy from the user inward. Each layer should be hit *before* the layer beneath.

```
┌────────────────────────────────────────────┐
│ 1. Browser cache         (per user)        │  ms RTT
│ 2. Service Worker        (offline-first)   │
├────────────────────────────────────────────┤
│ 3. CDN edge              (per POP)         │  10–50 ms RTT
│ 4. Reverse proxy / Varnish                 │
├────────────────────────────────────────────┤
│ 5. App in-process cache  (per pod)         │  µs
│ 6. Distributed cache     (Redis/Memcached) │  0.3–1 ms
├────────────────────────────────────────────┤
│ 7. Database buffer pool  (per DB node)     │  100 µs
│ 8. OS page cache         (per host)        │  100 ns
└────────────────────────────────────────────┘
         ↓ origin: SQL, object store, compute
```

A request that hits layer 1 never even leaves the user's machine. A request that misses every layer pays the full cost — origin DB, possibly cross-region. The job of a cache architect is to maximize the fraction of requests that terminate as high in this stack as the freshness contract allows.

See the per-layer details in [Cache Layers →](./cache-layers.md).

---

## 5. The Cache Contract: Freshness vs Speed

Every cache trades **freshness** for **speed**. The contract is set by three knobs:

- **TTL** — time the cached value is allowed to live before being checked.
- **Invalidation policy** — what causes a value to be considered stale.
- **Consistency expectations** — does the consumer accept stale reads, and for how long?

A real-world example: Stripe's dashboard caches account state for ~5 seconds. Slack caches channel metadata for minutes. Cloudflare's free CDN tier caches static assets for 4 hours by default. Each chose a TTL based on *how much staleness their users tolerate.*

```
         ┌──────────────────────────────────┐
         │  freshness   ◄────────►   speed  │
         │       ↑                    ↑     │
         │   no cache             tight TTL │
         │                       short TTL  │
         │                       long TTL   │
         │                     cache forever│
         └──────────────────────────────────┘
```

There are exactly two hard problems in caching:
1. **Naming things** (the cache key).
2. **Invalidation** (knowing when to evict).

Phil Karlton was right about this in the 90s and he's still right.

See [Cache Invalidation →](./cache-invalidation.md) for the full story.

---

## 6. Cache Math: Hit Rate, Miss Penalty, Effective Latency

The effective latency of a cached system is:

```
L_effective = (hit_rate * L_cache) + ((1 - hit_rate) * (L_cache + L_origin))
            ≈ L_origin * (1 - hit_rate)            // when L_cache << L_origin
```

Some quick intuitions:

| Hit rate | L_origin = 50 ms | L_origin = 5 ms |
|---|---|---|
| 50% | 25 ms | 2.5 ms |
| 80% | 10 ms | 1 ms |
| 90% | 5 ms | 0.5 ms |
| 95% | 2.5 ms | 0.25 ms |
| 99% | 0.5 ms | 0.05 ms |
| 99.9% | 50 µs | 5 µs |

Two non-obvious consequences:

1. **Going from 95% to 99% halves your effective latency.** The marginal hit rate matters more than the absolute one. A 90% cache is *not* "almost as good" as a 99% cache — it's 10× worse for tail latency.
2. **Tail latency is dominated by misses.** Your p99 is essentially your miss latency. If misses cascade to a slow path (cross-region, cold DB), your p99 will be terrible even with a great hit rate.

This is why CDNs obsess about hit rates and why Netflix measures every cache by **miss cost** in addition to hit rate.

---

## 7. What to Cache

Not everything benefits from caching. The cache-worthiness of a workload is roughly:

```
cache_value = (read_frequency * miss_cost * skew) / (write_frequency * staleness_cost)
```

In English:
- Read-heavy ✓
- Expensive to compute ✓ (e.g., aggregations, fan-out queries)
- Skewed access patterns ✓
- Stable values (low write rate) ✓
- High tolerance for staleness ✓

Things that cache well:
- User profile data.
- Product catalog entries.
- Search results for common queries.
- Configuration / feature flags.
- Auth tokens, sessions.
- Rendered HTML fragments.
- API responses for slow downstream services.
- Computed aggregates (top-10 lists, leaderboards).
- Geo lookups, DNS records.

Things that cache poorly:
- Per-user real-time data (inbox, notifications) — unless you can shard by user.
- Strongly-consistent state (bank balance during a transfer).
- Write-mostly logs.
- Random uniform-distribution lookups.
- Anything where staleness causes correctness bugs (locking, idempotency state).

---

## 8. Where to Put It

The decision tree:

```
Is the data ~the same for every user, and slow-changing?
  ├─ YES → cache it at the CDN.
  └─ NO  → Is it the same across pods of one service?
            ├─ YES → distributed cache (Redis/Memcached).
            └─ NO  → in-process / per-pod cache.
                     (Then a distributed cache behind it.)
```

You almost never want only one layer. A real production stack typically has:
- CDN for static + cacheable API responses.
- A per-pod in-process LRU for hot-of-hot lookups (sub-µs hit).
- A Redis cluster as the shared L2.
- Database buffer pool as L3.
- Object storage / cold storage at the bottom.

See [Cache Layers →](./cache-layers.md).

---

## 9. The Costs You're Buying

Caches are not free. They add:

- **Operational complexity** — a new service to monitor, scale, fail over.
- **Inconsistency** — two sources of truth. Stale reads.
- **Memory cost** — RAM is ~10× the price of disk per GB.
- **Cold-start risk** — empty cache after deploy or restart = origin meltdown ("cold cache stampede"). See [Cache Pitfalls →](./cache-pitfalls.md).
- **Debugging difficulty** — "but I just updated it!" / "no you didn't, you wrote to the DB and not the cache" / "ah, the bug is now mine forever."
- **Capacity-planning failure mode** — when the cache layer breaks, the origin sees 20× normal load and dies.

Two specific traps worth naming:
- **Caching pre-mature** — caching a query that runs at 10 RPS when your DB does 50k RPS just adds operational surface area for no benefit.
- **Caching to hide a bad query** — fix the query first. A cached bad query is still a bad query when it misses.

---

## 10. A Concrete Stack Example

How a typical request to a "view product page" endpoint flows through a modern stack:

```
1. User → Browser cache hit?         (HTTP cache, ETag)
   ├─ HIT → render. done.
   └─ miss → continue
2. Browser → CDN edge (e.g., Cloudflare)
   ├─ HIT → respond from POP, ~20 ms
   └─ miss → continue, register pull
3. CDN → Origin LB → service pod
4. Service: in-process L1 cache (Caffeine, etc.)
   ├─ HIT → ~5 µs, respond
   └─ miss → continue
5. Service → Redis L2
   ├─ HIT → ~0.5 ms, respond, populate L1
   └─ miss → continue
6. Service → Postgres
   ├─ buffer pool HIT → ~100 µs
   └─ miss → SSD read → ~50 µs
7. Service populates Redis (with TTL or write-through)
   → returns to CDN → CDN caches → user.
```

A single popular product page might survive entirely on layers 1–3 for hours and never touch layers 5–7. That's the entire game.

---

## 11. Common Mistakes

- **No TTL** — relying on explicit invalidation only. One bug and stale data lives forever.
- **TTL of 0 or 1 second on a "cache"** — congratulations, you've built a slow direct read.
- **Caching errors** — caching a 500 response for an hour. Always have a separate, short TTL for non-2xx responses, or don't cache them.
- **Caching auth-sensitive data without varying the key by user** — leaking one user's data to everyone. Cardinal sin.
- **No metrics on hit rate** — you're flying blind. Every cache needs hit rate, miss latency, and eviction rate dashboards.
- **Cold deploys** — restarting all pods at once with no warm-up = thundering herd on the origin. Roll deploys + cache priming or rely on a tier of stable distributed cache.
- **One huge cache key with a giant value** — one hot key kills you. Split into smaller keys.
- **Caching the wrong granularity** — caching the whole HTML page when only the user-specific bit changes. Cache fragments instead.

---

## 12. Cheat Card

```
WHY CACHE     reads >> writes, skewed access, slow origin, expensive compute

HIERARCHY     browser → CDN → reverse proxy → app L1 → distributed L2
              → DB buffer pool → OS page cache → disk

MATH          L_eff ≈ L_origin * (1 − hit_rate)
              95% → 99% halves effective latency
              p99 is dominated by misses

CACHE WORTH   high read freq * miss cost * skew
              / write freq * staleness cost

WHEN TO USE   skewed read workloads, slow origin, expensive compute,
              tolerable staleness
WHEN NOT TO   write-heavy data, strong consistency, uniform access,
              hiding a bug

PITFALLS      no TTL, no metrics, cold-start stampedes, key leakage,
              caching errors, premature caching

RULE          A cache is a promise to be a little wrong, very quickly.
```

---

## 13. Resources

### Books
- *Designing Data-Intensive Applications* — Martin Kleppmann (caching considerations throughout, especially Ch 5).
- *Systems Performance* — Brendan Gregg (the cache hierarchy from CPU to disk in painful, useful detail).

### Articles
- "Caching at Reddit" — Reddit Engineering Blog.
- "How Discord stores billions of messages" — Discord Engineering (caching layers around Cassandra).
- "Memcache at Facebook" — Nishtala et al., NSDI 2013 (the classic).
- "The latency numbers every engineer should know" — Jeff Dean (Stanford talk).

### Videos
- ByteByteGo — "Top 5 Cache Strategies".
- Hussein Nasser — "Caching Layers Explained".

### Tools
- **Redis**, **Memcached**, **Varnish**, **Caffeine** (Java in-process), **Ristretto** (Go).
- **Cloudflare**, **Fastly**, **Akamai**, **AWS CloudFront** (CDN).
- **prometheus-redis-exporter** for hit-rate / latency metrics.

### Adjacent reading
- [Cache Layers →](./cache-layers.md)
- [Cache Strategies →](./cache-strategies.md)
- [Cache Invalidation →](./cache-invalidation.md)
- [Cache Pitfalls →](./cache-pitfalls.md)
- [Latency Numbers Every Engineer Should Know →](../01-foundations/latency-numbers.md)

---

*Up:* [README ↑](../README.md)  |  *Next:* [Cache Layers →](./cache-layers.md)
