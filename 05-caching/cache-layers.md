# Client-Side, CDN, Server-Side, Database Caching

> **TL;DR** — A real production system caches at **every layer** between the user and the source of truth: in the **browser** (HTTP cache, Service Worker), at the **CDN edge**, at a **reverse proxy** (Varnish, Nginx), in your **application process** (Caffeine, Ristretto), in a **distributed cache** (Redis, Memcached), in the **database buffer pool**, and in the **OS page cache**. Each layer trades freshness for proximity. The further from the origin a request terminates, the faster it is — and the more users it serves with one cached copy. The skill is sizing each layer's TTL and capacity so the working set fits, the hit rates compound, and a single layer's failure doesn't melt the next.

---

## 1. The Stack

```
  ┌─────────────────────────────────────────────────────┐
  │                  USER'S BROWSER                     │
  │  L0: Browser HTTP cache (memory + disk)             │
  │  L0: Service Worker (programmable cache)            │
  └─────────────────────────────────────────────────────┘
                          │
                          ▼
  ┌─────────────────────────────────────────────────────┐
  │                    CDN EDGE                         │
  │  L1: POP cache (per region)                         │
  │  L1: Tiered cache → shield → origin                 │
  └─────────────────────────────────────────────────────┘
                          │
                          ▼
  ┌─────────────────────────────────────────────────────┐
  │              REVERSE PROXY / GATEWAY                │
  │  L2: Varnish / Nginx microcache                     │
  └─────────────────────────────────────────────────────┘
                          │
                          ▼
  ┌─────────────────────────────────────────────────────┐
  │                APPLICATION SERVER                   │
  │  L3: In-process cache (Caffeine, Ristretto, LRU)    │
  │  L4: Distributed cache (Redis, Memcached)           │
  └─────────────────────────────────────────────────────┘
                          │
                          ▼
  ┌─────────────────────────────────────────────────────┐
  │                   DATABASE                          │
  │  L5: Buffer pool                                    │
  │  L5: Query result cache (rare nowadays)             │
  └─────────────────────────────────────────────────────┘
                          │
                          ▼
  ┌─────────────────────────────────────────────────────┐
  │              OPERATING SYSTEM                       │
  │  L6: Page cache (kernel)                            │
  └─────────────────────────────────────────────────────┘
                          │
                          ▼
                       DISK / S3
```

Every cached layer above is a request that *didn't happen* below. Compounding matters: 80% × 80% × 80% means only 0.8% of requests reach origin.

---

## 2. Client-Side: Browser HTTP Cache

Browsers cache resources locally based on **HTTP cache headers**. The cache lives in two tiers (memory + disk) and is per-origin.

The headers that drive it:

| Header | What it does |
|---|---|
| `Cache-Control: max-age=N` | Cache for N seconds (the modern way). |
| `Cache-Control: s-maxage=N` | Same, but only for shared caches (CDN). |
| `Cache-Control: public/private` | Allow shared caches or not. |
| `Cache-Control: no-cache` | Cache it, but revalidate every time. |
| `Cache-Control: no-store` | Never cache. For sensitive data. |
| `Cache-Control: immutable` | Don't revalidate. The file at this URL will never change. |
| `Cache-Control: stale-while-revalidate=N` | Serve stale for N seconds while refetching in background. |
| `ETag: "abc123"` | Opaque version tag. Browser sends back as `If-None-Match`. |
| `Last-Modified` | Timestamp version. Browser sends back as `If-Modified-Since`. |
| `Vary: Accept-Encoding` | Cache differently based on these request headers. |
| `Age` | How long the response has been in caches. |

### Two cache validations
- **Strong (fresh)** — `max-age` not expired → return from cache, no network.
- **Conditional revalidation** — `max-age` expired but `ETag`/`Last-Modified` present → send `If-None-Match` → server replies `304 Not Modified` if unchanged. Saves bandwidth but not the round trip.

### Cache-busting via URL fingerprinting
Modern frontends bundle files with hashed names: `app.f3a9b2.js`. The URL is unique to the content. Set:

```
Cache-Control: public, max-age=31536000, immutable
```

The browser caches the file for a year and never revalidates. When the app changes, the hash changes, so the URL changes, so it's a new cache key. **This is the single most effective web caching pattern.** Use it for every static asset.

### Versioning hazard
Setting an asset to `max-age=86400` and then changing it without changing the URL means some users will see the new HTML referring to old JS for a day. Result: white-screen production incidents. Either fingerprint your URLs *or* keep short TTLs on un-fingerprinted assets.

### Service Workers
A Service Worker is a programmable proxy inside the browser. You write JS that intercepts `fetch` events and decides whether to hit the network, the Cache API, or IndexedDB. Used for:
- Offline-first PWAs (Twitter Lite, Pinterest).
- Custom cache strategies (stale-while-revalidate is built in).
- Background sync.
- Push notifications.

Service Workers give you the same patterns as server-side caching (cache-aside, write-through, etc.) on the client.

---

## 3. CDN Edge

A **CDN** (Content Delivery Network) is a distributed reverse-proxy fleet with POPs (Points of Presence) in dozens to hundreds of cities. The user's request hits the nearest POP, which either serves from cache or pulls from origin.

What CDNs do, in priority order:
1. **Cache static assets** — JS, CSS, images, fonts. The bread-and-butter.
2. **Cache HTML pages** — for sites with anonymous traffic, this is enormous.
3. **Cache API responses** — increasingly common, especially with edge functions.
4. **TLS termination** — keep the expensive handshake near the user.
5. **DDoS absorption** — anycast + huge capacity.
6. **Edge compute** — Cloudflare Workers, Fastly Compute@Edge, Lambda@Edge.

The deep dive is in [CDN →](./cdn.md). For this page, the key fact:

> The CDN is the most important cache layer for any internet-facing system that serves more than a handful of users.

If you ignore everything else on this page, putting CloudFront / Cloudflare / Fastly in front of your origin is the highest-ROI change you can make.

Cache hierarchy within a CDN: **tiered caching** routes regional POP misses to a smaller set of "shield" or "tier-2" caches before going to origin. This reduces origin load dramatically because most regional misses are not global misses.

```
user → POP-london → POP-london miss
                 → shield-eu-west → shield hit (someone in Berlin asked already)
                 → respond, populate POP-london
```

Without tiered caching, every regional POP is a separate cache and your origin sees `N_pops × miss_rate` worth of pulls for the same object.

---

## 4. Reverse Proxy / Microcache

Between the CDN and your application sits a layer most people forget: the **reverse proxy** — Nginx, Varnish, HAProxy, Envoy. These can hold a short-TTL cache called a **microcache**.

Use case: an authenticated personalized page that you can't push to the CDN. Cache it at the reverse proxy for 1–5 seconds. At 1000 RPS, that cuts origin load by 1000–5000×. Slack does this. Reddit does this.

```nginx
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=microcache:10m
                 max_size=1g inactive=60s use_temp_path=off;

location /feed/ {
    proxy_cache microcache;
    proxy_cache_valid 200 1s;
    proxy_cache_use_stale error timeout updating;
    proxy_cache_background_update on;
    proxy_cache_key "$scheme$host$request_uri$cookie_userid";
    proxy_pass http://backend;
}
```

`proxy_cache_use_stale ... updating` + `proxy_cache_background_update on` give you single-flight refresh under load — only one backend request at a time per cache key. The others get a slightly-stale response. This kills the [thundering herd →](./cache-pitfalls.md).

Varnish takes this further with **VCL** (Varnish Configuration Language) — you can write rules to:
- Strip cookies that prevent caching.
- Normalize the cache key (case-fold, sort query params).
- Cache different variants for logged-in vs anonymous users.
- Implement stale-while-revalidate and grace-mode.

Varnish is what most large media sites (NYT, BBC) put in front of their origin app servers. It's brutally fast (single-host throughput in the hundreds of thousands of req/s).

---

## 5. Application-Level (In-Process) Cache

An **in-process cache** lives inside your app's memory. Sub-microsecond hits. No network. No serialization. The fastest tier you control.

Examples:
- **Caffeine** (Java) — high-throughput Window TinyLFU.
- **Ristretto** (Go) — TinyLFU, contention-free.
- **lru-cache** (Node).
- **functools.lru_cache** (Python).
- **moka** (Rust).

When to use:
- The data is **read by every request** in the pod (hot config, feature flags).
- Latency budget is tight (microseconds matter).
- The same value can be served to multiple users (no per-user key).
- Eventual consistency is acceptable across pods.

Pitfalls:
- **Stale across pods** — pod A updates value, pod B has old copy for the TTL duration. Use short TTLs.
- **Memory pressure** — overfilling the cache GCs your service. Set a cap.
- **Cold start** — new pod has empty cache. Big rollouts can cripple the origin. Either prime the cache, deploy gradually, or rely on the L2 (Redis) to absorb the miss.
- **Per-pod skew** — if traffic isn't load-balanced perfectly, some pods cache certain keys more than others. Usually fine.

Caffeine has become the default for new Java systems. It gives ~80–90% of an LRU's hit rate at ~10× the throughput and supports refresh-after-write, async loading, and stats out of the box.

```java
LoadingCache<String, User> users = Caffeine.newBuilder()
    .maximumSize(10_000)
    .expireAfterWrite(Duration.ofMinutes(5))
    .refreshAfterWrite(Duration.ofMinutes(1))   // single-flight refresh
    .recordStats()
    .build(key -> userRepo.findById(key));
```

That's a production-grade cache in 7 lines. It handles single-flight, async refresh, eviction, and stats.

---

## 6. Distributed Cache

A **distributed cache** is a shared L2 across all your app pods. The canonical implementations:
- **Redis** — single-threaded, rich data types, persistence options, cluster mode.
- **Memcached** — multi-threaded, dumb-but-fast, no persistence, simpler.
- **DragonflyDB** — Redis-compatible, multi-threaded.
- **KeyDB** — multi-threaded Redis fork.
- **Hazelcast** — JVM-native, distributed by design.

Why a shared cache?
- Pod A's miss populates the cache; pods B–Z benefit.
- One cache invalidation reaches all pods at once.
- The cache survives pod restarts and rolling deploys.
- You can put more memory in one Redis cluster than in 100 pods combined.

Why not always?
- 0.3–1 ms network hop per call. Trivial for most APIs, painful for tight loops.
- Serialization cost (JSON vs MessagePack vs Protobuf matters).
- Single hot key can swamp one Redis node.

**Combine, don't choose.** A typical stack:
- **L1**: in-process Caffeine, 10s TTL, ~10k entries.
- **L2**: shared Redis, 5min TTL, ~10M entries.
- **L3**: origin.

The deep dive lives in [Distributed Caching →](./distributed-caching.md) and [Redis Deep Dive →](./redis-deep-dive.md).

---

## 7. Database-Level Caching

Two distinct things often called "database caching":

### 7.1 Buffer pool / page cache
Every relational DB keeps a **buffer pool** of recently-read pages in RAM. Postgres calls it `shared_buffers`. MySQL/InnoDB calls it `innodb_buffer_pool_size`. SQL Server has the **buffer pool manager**.

This is the database's own cache. Properly sized — typically **25–75% of host RAM** — it absorbs the hot working set of your data. If your queries are slow because of cold reads, your buffer pool is the first knob to turn. A 1 GB buffer pool over a 100 GB table is a problem; a 100 GB buffer pool over a 100 GB table makes you immortal.

```sql
-- Postgres
SHOW shared_buffers;        -- typically 25% of RAM
SHOW effective_cache_size;  -- planner's estimate of total cache (~75% of RAM)

-- Cache hit ratio (Postgres)
SELECT sum(blks_hit)::float / NULLIF(sum(blks_hit + blks_read), 0)
FROM pg_stat_database;
-- aim for 0.99+
```

### 7.2 Query result cache
A separate cache of materialized query results. Less common today:
- MySQL's query cache existed but was removed in 8.0 — it was a serialization bottleneck.
- ProxySQL and Vitess can do query-level caching.
- Postgres has no native query result cache; the closest thing is materialized views.

If you need query-level caching, do it in your application or with Redis. Don't try to retrofit it onto your DB.

### 7.3 Adjacent: materialized views & pre-aggregations
A **materialized view** is a stored-and-refreshable query result. It's a cache too, just one the database manages. Postgres supports them (`REFRESH MATERIALIZED VIEW`), as does Snowflake (auto-refresh), BigQuery, ClickHouse, etc. For expensive aggregations queried often (dashboards, leaderboards), they're often the right tool.

---

## 8. OS Page Cache

The kernel keeps recently-read file pages in RAM. Every disk read populates it; every disk write hits it first (then async flushes). For most databases and file-backed systems, the OS page cache is the actual reason reads are fast.

This is *why* databases recommend leaving headroom for the OS page cache rather than allocating all RAM to the DB process. Postgres explicitly relies on the OS page cache; that's why `effective_cache_size` includes it.

For object stores (S3) and disk-based systems, you mostly don't see the page cache; it just works. The two times you'll care:
- You're running a custom storage engine and want to disable it (`O_DIRECT` for raw I/O).
- A container's memory limit shrinks the page cache and your IOPS tank.

---

## 9. A Realistic Hit-Rate Compounding Example

For a Netflix-style catalog endpoint:

| Layer | Hit rate (of incoming) | What reaches the next layer |
|---|---|---|
| Browser cache | 30% | 70% of traffic |
| CDN edge | 90% of 70% = 63% | 7% |
| App L1 (in-process) | 50% of 7% = 3.5% | 3.5% |
| Redis L2 | 90% of 3.5% = 3.15% | 0.35% |
| DB buffer pool | 99% of 0.35% = 0.346% | 0.0035% |

Of 1M requests/sec, the origin DB sees ~35 disk reads/sec. That's the magic of compounding caches.

This is also why a single layer failing is dangerous: if Redis vanishes, the origin suddenly sees 3.5% of 1M = 35,000 RPS. Better hope your DB can handle it. Most can't.

---

## 10. Sizing Each Layer

A rule-of-thumb sizing approach:

1. **Identify the working set** — the volume of data accessed in a typical hot window (last 5 min, last hour, last day).
2. **Browser** — set TTLs based on staleness tolerance per resource type:
   - Hashed static assets: 1 year, immutable.
   - HTML: short (seconds–minutes).
   - API data: as long as the contract allows.
3. **CDN** — capacity is the CDN's problem. You tune TTL and cache-key normalization. Aim for >90% hit rate on cacheable URLs.
4. **In-process** — `min(working_set, 10–20% of pod RAM)`. Smaller is fine; you have an L2 below.
5. **Redis** — `working_set * (1 + buffer)`. Plan for `maxmemory` with `allkeys-lru` eviction. Watch eviction rate as the *true* sizing signal — if it's nonzero, the working set is bigger than the cache.
6. **DB buffer pool** — 25–75% of host RAM. Postgres conventional: 25% `shared_buffers` + leave the rest for the OS page cache. MySQL: 50–80% `innodb_buffer_pool_size` since InnoDB does the buffering itself.

---

## 11. Cache Key Design Across Layers

A cache key is the contract between a request and a stored value. Get it wrong and you serve the wrong data.

**Things that should be in the key:**
- The resource ID (`user:123:profile`).
- The variant axis (`user:123:profile:v=mobile`).
- The version of the schema / serializer (`v2`).
- The locale, currency, tenant — anything that changes the response.

**Things that should NOT be in the key:**
- Tracking params (`utm_*`, `gclid`) — normalize them out at the CDN.
- Session IDs (unless you really need per-session caching, which mostly defeats the cache).
- Random query strings.

**Use `Vary` correctly at the CDN:**
```
Vary: Accept-Encoding, Accept-Language
```
This tells the CDN to cache different copies based on these request headers. Be careful: `Vary: User-Agent` shatters your cache into thousands of fragments.

A canonical key naming convention helps:

```
service:entity:id:variant:version
products:detail:42:en-US:v3
users:settings:123:v1
```

---

## 12. Failure-Mode Drills

What happens when each layer fails?

| Layer down | Effect |
|---|---|
| Browser cache | None at first; users see slightly more traffic. Refetch ETag. |
| CDN | Catastrophe. Origin gets full traffic. Multi-CDN or origin shields required. |
| Reverse-proxy microcache | Brief origin spike. Usually survivable. |
| In-process | Pod hits Redis. Fine. |
| Redis | Origin DB gets ~10× normal RPS. Have a circuit breaker + fallback. |
| DB | Application breaks. Cache cannot save you. |

Two things every team eventually learns the hard way:
- **The CDN must work.** If it doesn't, you don't have a working website. Multi-CDN setups (Cloudflare + Fastly with health-checked DNS) exist precisely for this reason.
- **Redis is a load-bearing wall.** If your service can't tolerate Redis being down, design for it: graceful degradation, in-process fallback, or treat Redis loss as a P0 incident.

---

## 13. Common Mistakes

- **Caching only one layer.** All-or-nothing thinking. Each layer has different costs/benefits.
- **Conflicting TTLs.** Browser caches for 1 hour, CDN caches for 1 day. User won't see new content for a day even after a flush. Coordinate.
- **No `Vary` header for language/encoding** — users get wrong-language responses.
- **Caching errors and 5xx responses.** Brief outage → cached errors for hours.
- **`Cache-Control: no-cache` confused with `no-store`.** `no-cache` *does* cache; it just revalidates every time.
- **Not normalizing query strings.** `?utm_source=x` and `?utm_source=y` are separate cache keys; cache fragmented.
- **Forgetting `s-maxage`** — you wanted browser to revalidate quickly but CDN to hold for a day; only `s-maxage` distinguishes.
- **Caching personalized content at the CDN without varying by user.** Severe security risk.
- **In-process cache without max size.** OOM on the first hot key surge.

---

## 14. Cheat Card

```
LAYERS         browser → CDN → reverse proxy → app L1 → Redis L2
               → DB buffer pool → OS page cache → disk

BROWSER        Cache-Control + ETag + fingerprinted URLs
               immutable + max-age=31536000 for static assets

CDN            tiered caching, ~90% hit rate target,
               edge compute for personalization

REVERSE PROXY  microcache (1–5s) for personalized pages,
               proxy_cache_use_stale for single-flight

APP L1         Caffeine/Ristretto, ~10k entries, sub-µs hit
APP L2         Redis/Memcached, ~10M entries, ~0.5ms hit

DB             buffer pool sized to working set, hit ratio > 99%

PITFALL        TTLs not aligned across layers
PITFALL        no Vary header → wrong content delivered
PITFALL        cache personalized data at CDN

RULE           Cache compounds. A 90% hit rate at every layer
               means only 0.01% of traffic reaches origin.
```

---

## 15. Resources

### Books
- *High Performance Browser Networking* — Ilya Grigorik (the bible for browser/HTTP caching).
- *Designing Data-Intensive Applications* — Martin Kleppmann.

### Documentation
- **MDN — HTTP caching**: <https://developer.mozilla.org/en-US/docs/Web/HTTP/Caching>
- **Varnish Book**: <https://book.varnish-software.com>
- **Nginx HTTP cache module**: <https://nginx.org/en/docs/http/ngx_http_proxy_module.html>
- **Postgres `shared_buffers`**: <https://www.postgresql.org/docs/current/runtime-config-resource.html>

### Articles
- "Caching tutorial for web authors" — Mark Nottingham: <https://www.mnot.net/cache_docs/>
- "RFC 9111: HTTP Caching" — IETF.
- "Fastly tiered caching" — Fastly Engineering blog.

### Videos
- ByteByteGo — "How CDN works".
- Hussein Nasser — "Nginx caching deep dive".

### Tools
- **Caffeine** (Java), **Ristretto** (Go), **moka** (Rust), **lru-cache** (Node).
- **Varnish**, **Nginx**, **Envoy**.
- **Redis**, **Memcached**, **Dragonfly**.

### Adjacent reading
- [Why Cache? Cache Hierarchy →](./caching-overview.md)
- [Cache Strategies →](./cache-strategies.md)
- [Distributed Caching →](./distributed-caching.md)
- [CDN →](./cdn.md)
- [Cache Pitfalls →](./cache-pitfalls.md)

---

*Previous:* [← Why Cache? Cache Hierarchy](./caching-overview.md)  |  *Next:* [Cache Strategies →](./cache-strategies.md)
