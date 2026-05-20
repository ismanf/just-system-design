# Cache Stampede, Thundering Herd, Hot Keys

> **TL;DR** — The three cache pathologies that take systems down: **cache stampede** (a popular key expires, N concurrent requests miss together and all rebuild it, melting the origin), **thundering herd** (a wake-up event — restart, deploy, mass invalidation — sends a synchronized burst of misses to the origin), and **hot keys** (one or a few keys absorb a disproportionate share of traffic, saturating the shard that owns them). The fixes are the same family: **single-flight** loads (only one rebuild per key), **probabilistic early refresh**, **randomized TTL jitter**, **stale-while-revalidate**, **request coalescing**, **negative caching** with bounded TTL, **L1 in front of L2**, and for hot keys specifically — **key sharding** and **replica reads**. The root failure mode is the same: a cache that was protecting the origin briefly stops, and the origin sees full unfiltered traffic.

---

## 1. The Pathologies, Visualized

```
NORMAL                       hit rate 99%
   ┌──────────┐
   │  cache   │── 99% ─► clients
   └──────────┘
         │
         └── 1% misses ───► origin (light load)

CACHE STAMPEDE              one key expires → N misses on the same key
   ┌──────────┐
   │  cache   │  X (expired)
   └──────────┘
         ↑           ↑           ↑           ↑
       1000 simultaneous misses on the same key
                       │
                       └── 1000 concurrent origin queries

THUNDERING HERD            shared trigger → many keys miss at once
   ┌──────────┐
   │  cache   │  X X X X X X X X X X
   └──────────┘
         │
   thousands of synchronized origin queries

HOT KEY                    one key dominates → one shard saturated
   ┌─────────┬─────────┬─────────┐
   │ shard A │ shard B │ shard C │
   │ 99% RPS │ <1% RPS │ <1% RPS │
   └─────────┴─────────┴─────────┘
```

All three end the same way: the database — sized for the protected, cached traffic rate — gets hit with 10–1000× normal load, slows down, queues up, and either dies or causes upstream timeouts.

---

## 2. Cache Stampede (a.k.a. Cache-Miss Dogpile)

A popular cached value expires. Before the first miss completes its origin fetch and `SET`, 999 other requests miss the same key and all execute the same expensive origin query. The origin is now serving 1000× the work for one item.

```
t=0     value expires (TTL hits)
t=0+ε   req#1 misses → starts origin query (10ms)
t=0+ε   req#2..req#1000 miss → each starts origin query
t=10ms  req#1 completes, sets cache
t=10ms+ req#2..#1000 also complete, each writes the same value
```

You paid 1000× origin cost for one cached value.

### Fix 1 — Single-flight / Mutex
First requester acquires a lock and fetches; others wait for the result.

```python
def get_with_singleflight(k, loader, ttl=300):
    v = cache.get(k)
    if v is not None:
        return v
    lock_key = f"lock:{k}"
    if cache.set(lock_key, 1, nx=True, ex=10):   # SETNX with TTL
        try:
            v = loader()
            cache.setex(k, ttl, v)
            return v
        finally:
            cache.delete(lock_key)
    # someone else is loading; wait briefly and retry / return stale
    for _ in range(10):
        v = cache.get(k)
        if v is not None:
            return v
        time.sleep(0.05)
    return loader()  # safety: fall back
```

Caffeine, Ristretto, Go's `singleflight` package, Python's `aiocache` all expose this directly.

### Fix 2 — Probabilistic Early Refresh (XFetch)
Refresh the cache **before** TTL with probability that grows as expiry approaches. By the time the TTL hits, the value has already been refreshed by one randomly-chosen request.

The math (Vattani et al., "Optimal Probabilistic Cache Stampede Prevention"):

```python
import random, math

def should_refresh(expiry_time, delta_avg_recompute, beta=1.0):
    now = time.time()
    return now - delta_avg_recompute * beta * math.log(random.random()) >= expiry_time
```

If true on a given request, that request triggers the refresh. As `expiry_time - now` shrinks, the probability grows. Exactly one request (statistically) wins the race well before TTL.

### Fix 3 — Stale-While-Revalidate
Serve the slightly-stale value while a single background refresh happens.

```
soft_ttl = 60s   (logically expired, but still served)
hard_ttl = 600s  (actual eviction)

if soft_expired and not someone_refreshing:
    schedule background refresh
return current_value
```

This is the HTTP `Cache-Control: stale-while-revalidate` semantic. Nginx's `proxy_cache_use_stale ... updating` + `proxy_cache_background_update on` implements it. Varnish has it built in.

### Fix 4 — Async Recomputation
The cache is never populated by the request path. A separate worker recomputes hot values on schedule. Reads always hit cache.

Common for expensive aggregates (top-N lists, daily counters). The trade: bounded staleness, but no stampede ever.

---

## 3. Thundering Herd

A *synchronized* burst of misses, often across many keys. Causes:

- **Mass deploy / restart** — all pods restart, empty caches, slam Redis and the DB simultaneously.
- **Cache failure / flush** — Redis cluster fails over and warm cache vanishes. Everything misses.
- **Mass invalidation** — a tag-based purge clears 10k keys; the next minute is a flood.
- **Synchronized TTLs** — everything cached together expires together.

```
t=0   cache cleared / pods restart
t=ε   ALL traffic misses
t=ε+  origin sees full unfiltered load
```

This is cache stampede × many keys.

### Fix 1 — TTL Jitter
Randomize TTLs so they don't all expire together.

```python
cache.setex(k, base_ttl + random.randint(0, jitter), v)
# e.g., base=300, jitter=60 → 300–360s per key
```

Cheap, effective, no-brainer. **Every TTL in your system should have jitter.**

### Fix 2 — Cache Warming on Deploy
Before traffic shifts to a new pod, populate its caches. Either via a warm-up endpoint, replay of common requests, or by pulling a snapshot from a warmer cache tier (Redis).

Some teams ship a "top 1000 hot keys" file with each deploy; the new pod loads it on startup before joining the LB pool.

### Fix 3 — Rolling Deploys + Connection Drain
Never restart all pods at once. Roll deploys at < N% concurrent — even a 10% rolling deploy lets the remaining 90% of pods serve hot cache while the new 10% warm up.

### Fix 4 — Origin Concurrency Limit / Circuit Breaker
The origin (DB) should never trust the cache. Put a concurrency limit in front of every expensive origin query:

```python
@with_semaphore(max_concurrent=50)
def expensive_query(): ...
```

When the herd arrives, requests queue. The origin survives. Some requests time out, but the system doesn't melt down. **The principle: failure modes that hurt are better than failure modes that crash.**

### Fix 5 — Two-Tier Cache
A Redis L2 in front of the DB. A per-pod L1 in front of Redis. When pods restart, L1 is empty but L2 still warm — most traffic terminates at L2. Restart-time herd is absorbed without DB load.

### Fix 6 — Lazy Filling After Flush
On a mass flush, don't repopulate everything immediately. Let cache fill organically as traffic hits each key. Combined with single-flight, this turns the herd into a manageable wave.

---

## 4. Hot Keys

A single key gets disproportionate traffic.

Real examples:
- Trending tweet ID with 100k QPS reads. All on one shard.
- A "global counter" or "feature flag" key every request reads.
- The Justin Bieber problem at Twitter circa 2009 — a user with millions of followers melted the per-user shard.
- A celebrity product page on Amazon during a launch.

```
                shard ownership of "hot-key"
                       ▼
                ┌─────────────────┐
                │   shard 7       │  ←  ALL traffic
                └─────────────────┘
                ┌─────────────────┐
                │   shard 0..6,8..N│  ← idle
                └─────────────────┘
```

The shard owning the hot key saturates CPU or network. Latency spikes; other keys on that shard suffer. No amount of total cache capacity helps — the single key is the bottleneck.

### Detection
- Per-shard CPU, per-shard ops/sec on dashboards.
- `redis-cli --hotkeys` in LFU mode.
- Streaming logs, count by key prefix.
- Tail latency p999 spikes correlated with a specific shard.

### Fix 1 — In-Process L1 Cache
Each app pod caches the hot value for 1–10 seconds. The hot key now hits cache locally; Redis sees `N_pods / TTL` reads instead of `RPS`.

A pod might absorb 99% of the hot-key traffic for that key. Across 1000 pods, you get up to 1000× fan-out absorption.

This is the **single highest-leverage fix.** Use Caffeine / Ristretto.

### Fix 2 — Key Sharding (a.k.a. Sharded Counter / Replicated Hot Key)
Write the same value to N keys (`key:0`, `key:1`, ..., `key:N`). Reads pick one at random; loads spread.

```python
def get_hot(k, fanout=16):
    shard = random.randint(0, fanout-1)
    return cache.get(f"{k}:{shard}")
```

Writes have to broadcast to all N keys, which costs more. Best for read-heavy hot keys (trending content, feature flags).

For counters that need to be aggregated (write-heavy hot keys), invert: writes go to `counter:N` randomly, reads sum all shards. AWS calls this the "sharded counter" pattern.

### Fix 3 — Read Replicas
Redis Cluster `READONLY` reads from replicas. With a few replicas of each shard, hot-key reads scale linearly.

```python
client = Redis(readonly=True, prefer_replica=True)
client.get("hot:key")
```

Replicas serve stale-ish data; usually fine for trending content.

### Fix 4 — Don't Cache the Hot Key in Redis at All
Push it up the stack:
- Static HTML at the CDN.
- Pre-rendered fragments served at the edge.
- App-tier in-memory cache only, no Redis trip.

Hot keys are a sign the cache is not as close to the user as it should be. Move it.

### Fix 5 — Larger Shard Sizes
If a single hot key still saturates a shard, that shard is too small. Vertical scaling buys you headroom. Not the elegant fix, but sometimes the right one.

---

## 5. Cache Penetration

A pattern related to but distinct from stampede: clients query for keys that **don't exist** in the DB. Each query misses cache → hits DB → returns nothing → caches nothing → next request misses again. The cache provides no protection.

Attackers exploit this by enumerating random IDs.

### Fix 1 — Negative Caching
Cache the "not found" result with a short TTL.

```python
def get_user(uid):
    v = cache.get(k := f"user:{uid}")
    if v == SENTINEL_MISSING:
        return None
    if v is not None:
        return v
    row = db.find(uid)
    if row is None:
        cache.setex(k, 30, SENTINEL_MISSING)   # short TTL!
        return None
    cache.setex(k, 300, row)
    return row
```

Short TTL avoids the "user just got created → invisible for 5 minutes" trap.

### Fix 2 — Bloom Filter
A Bloom filter holds "the set of IDs that exist" with a small false-positive rate. Before hitting cache/DB, check the filter; if it says "definitely not in set," return 404 immediately.

```
client → Bloom: contains(id)?
       NO  → 404
       YES → cache → DB
```

Memory-efficient (~9 bits per ID for 1% false-positive rate). Used by Bigtable, Cassandra, Discord, and almost every system with a "definitely doesn't exist" fast path.

See [Bloom Filters →](../08-distributed-systems/bloom-filters.md).

### Fix 3 — Rate-Limit Cold Lookups
Track per-IP/per-user requests for nonexistent IDs. Block the abusive client. Combined with negative caching, this kills enumeration attacks.

---

## 6. Cache Avalanche

When the cache layer itself fails entirely. Redis cluster outage, network partition, OOM kill.

Symptoms:
- Cache hit rate drops to 0.
- All read traffic goes to the DB.
- DB saturates → queries queue → app threads block → cascading timeout failures.

### Survival design
- **Multi-AZ / failover** — Redis Sentinel or Cluster recovers in seconds to minutes.
- **Tight client timeouts** — fail fast on slow Redis.
- **Circuit breakers** — open the breaker after N failures; fall back to direct DB reads with rate limit.
- **DB rate limiting** — protect the origin from absorbing 100% of unfiltered load.
- **Graceful degradation** — return cached-stale, default, or partial responses rather than 500s.
- **Multi-tier cache** — in-process L1 keeps absorbing some traffic even with L2 down.
- **Capacity to handle cache loss** — uncomfortable but real. Some teams scale the DB to handle full traffic without cache; expensive but resilient.

The honest take: most systems can't fully survive cache loss. Aim to **degrade**, not crash. A 10× slower endpoint that returns correct data beats a 500.

---

## 7. The Single-Flight Pattern in Detail

Single-flight is the underlying primitive that solves most stampede problems. The mental model: "only one in-flight request per key at a time; all others piggyback on its result."

### In-process (Go)
```go
import "golang.org/x/sync/singleflight"
var g singleflight.Group

v, err, _ := g.Do(key, func() (any, error) {
    return expensiveFetch(key)
})
```

The Go `singleflight` package is canonical. All concurrent calls to `g.Do(key, fn)` for the same key dedupe to one execution.

### Distributed (across pods, via Redis)
Use a `SETNX` lock:
```redis
SET lockkey owner-id NX PX 10000
```

The winner fetches and `SET`s the value. The losers either wait briefly or return stale.

### With async refresh + serve stale
The reasonable compromise: serve stale while one background task refreshes. Lock-free for readers; only the refresh path is serialized.

Single-flight is so universally useful that any production cache abstraction should expose it. Caffeine, Ristretto, Go `singleflight`, Java `LoadingCache`, Python `aiocache.lock` — all have it.

---

## 8. Worked Example: Stampede Crater

A real anti-pattern that took down a service:

- Cache TTL: 300s, no jitter.
- Cache deployed Tuesday at noon; all keys created within a 1-minute window.
- Hit rate steady at 99% for a week.
- The following Tuesday at noon, all the hot keys expired within the same 1-minute window.
- The system melted.

Mitigations applied:
- TTL jitter `±20%` — keys naturally distribute their expirations.
- Single-flight on hot keys.
- Probabilistic early refresh for top 1% by traffic.
- L1 cache in each pod.

Outcome: the same "expire everything at once" event became a smooth refresh wave instead of a cliff.

---

## 9. Common Mistakes

- **No jitter on TTL.** The synchronized-expiry trap. Cheap to fix; expensive to suffer.
- **No single-flight on misses.** Even with jitter, popular keys will eventually expire and cause spikes.
- **No L1 in front of L2.** Every read goes to Redis. For hot keys, this is fatal.
- **No origin protection.** The DB trusts the cache to absorb load and has no concurrency limit. When the cache flinches, the DB dies.
- **No circuit breaker on the cache path.** A slow Redis blocks request threads; the app OOMs before the cache recovers.
- **Caching null without TTL.** Negative cache permanently hides new records.
- **Ignoring hot-key warnings.** "We saw a spike on shard 3 — anyway, what about the new feature?" That spike is your incident in three weeks.
- **Random salts in cache keys.** "We added a UUID per request to the key for safety!" Now every request misses; the cache is useless. (Yes, this happens.)
- **Cache-aside without considering the race window.** Reader populates a stale value after the writer's invalidation. Use [versioned keys or short TTLs](./cache-invalidation.md).
- **Stale-while-revalidate without bounding.** Stale served forever because nothing triggers the refresh. Always combine SWR with a hard TTL.

---

## 10. Decision Rules

```
Popular key with high read concurrency?
  → single-flight + L1 cache

Many keys with the same TTL?
  → add jitter (always)

Hot key on one shard?
  → L1 cache → key sharding → replica reads → push to CDN

Nonexistent IDs flooding the DB?
  → negative caching + Bloom filter

Cache restart / deploy = origin spike?
  → rolling deploy + warm-up + two-tier cache + origin rate limit

Refresh is slow but staleness is OK?
  → stale-while-revalidate + soft/hard TTL

Cache cluster outage = full origin load?
  → circuit breaker + degrade + origin limit + maybe DB-capacity-for-cold
```

---

## 11. Cheat Card

```
STAMPEDE       one expired key + many readers → N origin queries
  fixes        single-flight, probabilistic early refresh,
               stale-while-revalidate

THUNDERING HERD shared event → mass misses
  fixes        TTL jitter, rolling deploy, warm-up,
               two-tier cache, origin rate limit

HOT KEYS       one key saturates one shard
  fixes        L1 cache, key sharding, replica reads,
               push higher in stack

PENETRATION    misses on nonexistent IDs
  fixes        negative caching (short TTL), Bloom filter

AVALANCHE      cache layer fails entirely
  fixes        circuit breaker, degrade, origin rate limit,
               multi-tier

RULES
  every TTL has jitter
  every popular load is single-flight
  every L2 has an L1 in front of it
  every origin has a concurrency limit
  every cache outage is a degrade, not a crash
```

---

## 12. Resources

### Papers
- "Optimal Probabilistic Cache Stampede Prevention" — Vattani, Chierichetti, Lowenstein (2015). The XFetch algorithm.
- "Scaling Memcache at Facebook" — Nishtala et al., NSDI 2013. Hot keys, leases, gutter pools.

### Articles
- "Asynchronous Computation Caching" — Brad Fitzpatrick (LiveJournal-era foundational).
- "The cache stampede problem and how to solve it" — DoorDash Engineering Blog.
- "How we built our cache layer at Stripe" — Stripe Engineering.
- "Hot-key problem in Redis Cluster" — various engineering blogs.
- "Mcrouter pools and shadow reads" — Facebook Engineering.

### Documentation
- **Go `singleflight`**: <https://pkg.go.dev/golang.org/x/sync/singleflight>
- **Caffeine refresh-after-write**: <https://github.com/ben-manes/caffeine/wiki/Refresh>
- **Nginx `proxy_cache_use_stale`**: <https://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_cache_use_stale>
- **HTTP stale-while-revalidate (RFC 5861)**.

### Videos
- ByteByteGo — "Cache Stampede, Hot Keys, and Thundering Herd".
- Hussein Nasser — "Cache breakdown patterns".

### Tools
- **Caffeine** / **Ristretto** / **moka** — single-flight loaders.
- **redis-cli --hotkeys** — find hot keys in LFU mode.
- **vegeta / wrk / ghz** — load test stampede scenarios.
- **Chaos toolkit / Toxiproxy** — simulate cache outages.

### Adjacent reading
- [Cache Strategies →](./cache-strategies.md)
- [Cache Invalidation →](./cache-invalidation.md)
- [Eviction Policies →](./eviction-policies.md)
- [Distributed Caching →](./distributed-caching.md)
- [Bloom Filters →](../08-distributed-systems/bloom-filters.md)
- [Circuit Breaker →](../11-reliability/circuit-breaker.md)
- [Retry, Timeout, Backoff →](../11-reliability/retry-timeout-backoff.md)

---

*Previous:* [← Redis Deep Dive](./redis-deep-dive.md)  |  *Next:* [CDN — Content Delivery Networks →](./cdn.md)
