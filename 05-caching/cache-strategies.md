# Cache Strategies (Cache-Aside, Read-Through, Write-Through, Write-Back, Write-Around)

> **TL;DR** — There are five canonical caching patterns: **cache-aside** (the app manages the cache), **read-through** (the cache manages reads), **write-through** (writes go to cache *and* DB synchronously), **write-back / write-behind** (writes go to cache, DB updated async), and **write-around** (writes skip the cache, only reads populate). **Cache-aside is the default** — it covers 80% of real systems and is simple. Write-through is what you want when consistency matters more than write latency. Write-back is for write-heavy systems where you can tolerate small windows of data loss on cache failure. Mixing strategies for reads and writes is normal; pick per-workload.

---

## 1. The Five Patterns at a Glance

```
                READ PATH                     WRITE PATH
                ─────────                     ──────────
CACHE-ASIDE     app reads cache; on miss      app writes DB; invalidates cache
                app reads DB and populates
READ-THROUGH    app reads cache; cache        (independent)
                fetches DB on miss
WRITE-THROUGH   (independent)                 app writes cache; cache writes DB
                                              both sync
WRITE-BACK      (independent)                 app writes cache; cache writes DB
                                              async (later)
WRITE-AROUND    (independent)                 app writes DB; cache untouched
```

In practice you combine: e.g., **cache-aside read + write-around** (the most common pattern at Stripe-scale services).

---

## 2. Cache-Aside (a.k.a. Lazy Loading)

The default. The application is in charge of reading from the cache, falling back to the DB on miss, and populating the cache after fetching.

```
                READ
   ┌─────┐ 1. get   ┌───────┐
   │ App ├─────────►│ Cache │
   └──┬──┘          └───┬───┘
      │                 │ miss
      │ 2. read         │
      ▼                 │
   ┌─────┐              │
   │  DB │              │
   └──┬──┘              │
      │ 3. set          │
      └─────────────────►
```

```python
def get_user(user_id):
    key = f"user:{user_id}"
    user = cache.get(key)
    if user is not None:
        return user
    user = db.query("SELECT * FROM users WHERE id = %s", user_id)
    if user is not None:
        cache.set(key, user, ttl=300)
    return user
```

### Pros
- **Simple** — just a `get` then a fallback. No magic.
- **Resilient to cache failures** — if Redis is down, the app falls back to the DB. (Though see [pitfalls](./cache-pitfalls.md): the DB may not survive.)
- **Only caches what's actually read** — no wasted memory on writes that nobody reads.
- **Library-agnostic** — works with any cache and any DB.

### Cons
- **Cache miss penalty** — first read of every key pays the full DB cost.
- **Inconsistency window** — between DB write and cache invalidation, readers see stale data.
- **Double-roundtrip on miss** — cache → DB → cache → response.
- **Cache stampede risk** — N concurrent misses on the same key → N DB queries. See [Cache Pitfalls →](./cache-pitfalls.md).

### When to use
The default. Anything mostly-read with non-trivial fanout. Postgres + Redis with cache-aside reads is the workhorse of web engineering.

### When not to use
- Tight loops where the extra `cache.get` round trip is too expensive (use in-process L1).
- Cases where missed reads should *not* hit the DB (e.g., heavy origin protection — use [read-through with negative caching](./cache-pitfalls.md)).

---

## 3. Read-Through

Same shape as cache-aside, but the **cache** owns the read fallback, not the app. The app sees a single interface: "give me this key."

```
   ┌─────┐ 1. get   ┌───────┐ 2. miss  ┌─────┐
   │ App ├─────────►│ Cache ├─────────►│  DB │
   └─────┘ 4. value └───┬───┘ 3. value └─────┘
                        │ ↑
                       set on miss
```

In practice this is implemented as a *loader function* registered with the cache library:

```java
LoadingCache<String, User> users = Caffeine.newBuilder()
    .maximumSize(10_000)
    .expireAfterWrite(Duration.ofMinutes(5))
    .build(userId -> db.findUser(userId));   // ← read-through loader

User u = users.get("abc");  // single call, miss handled by loader
```

### Pros
- **Cleaner application code** — call sites stop branching on hit/miss.
- **Loader can centralize single-flight semantics** — only one thread loads per key.
- **Refresh-ahead** — many libraries (Caffeine, EHCache) can refresh in the background before TTL expires.

### Cons
- **Library lock-in** — only works if your cache lib supports it. Redis itself doesn't (Redis is a dumb store; you'd put your loader in front of it).
- **Same stampede risk** as cache-aside, unless the loader is single-flight.
- **Errors are awkward** — what does the loader return when the DB is down? Throw? Return cached error? Return stale?

### When to use
- JVM apps using Caffeine, Guava Cache, or EHCache.
- Anywhere you want the cache library to handle async refresh and single-flight.

### When not to use
- Polyglot environments where the cache is plain Redis. Just write cache-aside.

---

## 4. Write-Through

Writes go to the cache *and* the DB synchronously. The cache is always up to date because every write went through it.

```
   ┌─────┐ 1. write ┌───────┐ 2. write ┌─────┐
   │ App ├─────────►│ Cache ├─────────►│  DB │
   └─────┘ 4. ok    └───┬───┘ 3. ok    └─────┘
                        │
                        └── ack to app
```

Typical implementation: the application does both writes itself, in order:

```python
def update_user(user_id, data):
    db.execute("UPDATE users SET ... WHERE id = %s", user_id)
    cache.set(f"user:{user_id}", data, ttl=300)  # write-through
```

Or — less common — the cache library is configured to write through to a registered "writer" function.

### Pros
- **Cache stays consistent with DB** for written keys. No stale reads of just-updated data.
- **Reads after writes are fast** — value is already hot.
- **Simple mental model** — both writes happen on the same request.

### Cons
- **Higher write latency** — pays both the DB and the cache cost per write.
- **More writes to cache** — including values that may never be read.
- **Partial failures are ugly** — DB succeeded, cache failed (or vice versa). You need to decide who's the source of truth and what to do.
- **Doesn't help cold reads** — items never written through this app still cause misses.

### Order matters
The classic mistake is to **write to cache first, then DB**: if DB fails, your cache holds a value that doesn't exist in the source of truth. Always: **DB first, then cache.** If the cache write fails, evict the key so the next reader populates from DB.

### When to use
- Writes are infrequent enough that the extra latency doesn't hurt.
- Reads-after-writes are common and you can't tolerate the stale window.
- You're doing cache-aside reads anyway and want fewer misses.

### When not to use
- Write-heavy workloads (logging, telemetry, hot counters).
- The cache may not see most reads — wasted writes.

---

## 5. Write-Back (Write-Behind)

Writes go to the cache and acknowledge immediately. The cache (or a worker behind it) flushes to the DB asynchronously, batched or queued.

```
   ┌─────┐ 1. write ┌───────┐
   │ App ├─────────►│ Cache │── async ──►  ┌─────┐
   └─────┘ 2. ok    └───────┘              │  DB │
                                           └─────┘
                                 (batched, eventually)
```

This is how:
- CPU L1/L2 caches work (write-back to DRAM).
- The OS page cache works (`fsync` is what forces the flush).
- Many high-throughput counters work (write to Redis, flush to Postgres every N seconds).
- Kafka consumers in commit-async mode.

### Pros
- **Lowest write latency** — RAM speed.
- **Batches and coalesces writes** — N writes to the same key in 1s become one DB write.
- **Absorbs bursts** — bursty load goes to cache; DB sees a smooth rate.

### Cons
- **Data loss risk** — cache crash before flush = lost writes. This is the headline tradeoff.
- **Complexity** — durability, ordering, retries, dead-letter queues, idempotency on flush.
- **Stale reads from DB** — other consumers reading directly from the DB see old data until the flush.
- **Hard to reason about consistency** — what counts as "committed"?

### Mitigations
- Persist the cache to disk (Redis AOF) so a crash recovers most writes.
- Use Redis replicas as the durability bound.
- Make the flush idempotent (every record has a stable ID) so retries are safe.
- Have a max-buffer / max-age bound so the flush queue can't grow unbounded.

### When to use
- Counters, metrics, leaderboards (write rate dwarfs read rate per key).
- Activity feeds, "last seen" timestamps.
- Anything where occasional small data loss is tolerable.
- High-throughput ingest where DB throughput is the bottleneck.

### When not to use
- Money. Anything financial. The CFO will not accept "we cached it."
- Audit logs / compliance.
- Strong consistency requirements.

### Real-world examples
- **Discord** uses Redis as a write-back layer in front of Cassandra for some hot counters.
- **Twitter** uses similar patterns for engagement counts; the displayed count is an estimate, the durable count is the DB.

---

## 6. Write-Around

Writes go directly to the DB, skipping the cache entirely. The cache is populated only on read misses (i.e., cache-aside reads).

```
                READ                   WRITE
   ┌─────┐ get cache   ┌───────┐         ┌─────┐ write
   │ App ├────────────►│ Cache │         │ App ├──────► DB
   └──┬──┘             └───────┘         └─────┘
      │ on miss
      ▼
   ┌─────┐
   │  DB │
   └─────┘
```

### Pros
- **Write-once-read-rarely** keys never pollute the cache. Logs, telemetry, big blobs.
- **No write-side cache failure modes** — DB is the only writer.
- **Lower cache memory pressure** — only "really read" items live there.

### Cons
- **First read after a write is always a miss.** Trader's nightmare: you wrote a value, immediately reread it, got a stale cache hit. With write-around + cache-aside, you'd actually get the right answer (cache miss → DB read), but if you'd cached the *old* value before the write and haven't invalidated, you get the wrong answer. You must invalidate on write.

### When to use
- Bulk imports / writes the app won't read back soon.
- Write-once data (immutable logs, archives).
- When the cost of populating the cache from a write is wasted (writes don't predict reads).

### When not to use
- Read-after-write workflows where the writer is also the next reader (e.g., user updates profile then immediately views it).

---

## 7. Cache-Aside + Write-Around: The Default Recipe

The most common pattern in real systems:

```
READ:    cache-aside (app reads cache, falls back to DB, populates)
WRITE:   write-around with explicit invalidation
         (app writes DB, then deletes the cache key)
```

```python
def get_user(uid):
    key = f"user:{uid}"
    u = cache.get(key)
    if u is not None:
        return u
    u = db.find(uid)
    cache.set(key, u, ttl=300)
    return u

def update_user(uid, data):
    db.update(uid, data)
    cache.delete(f"user:{uid}")  # invalidate, don't write-through
```

Why this combo wins:
- **Reads are fast and lazy** — only popular items live in cache.
- **Writes are simple** — one transactional DB write, then a `DEL`.
- **Failure modes are clean** — worst case the cache holds a stale value until the next reader.
- **No partial-write inconsistencies** in the cache.

The one subtle issue is **delete-then-read race**: reader misses cache *before* the writer deletes, then populates with the old DB row *after* the writer committed. The reader briefly holds a stale value. Mitigations:
- Short TTL.
- Delete-on-write *plus* short TTL — bounds the staleness.
- Versioned keys (include a row version) — see [Cache Invalidation →](./cache-invalidation.md).

---

## 8. Comparison Table

| Pattern | Read flow | Write flow | Strengths | Weaknesses | Typical use |
|---|---|---|---|---|---|
| Cache-aside | App: cache → DB on miss | App: DB write, invalidate | Simple, resilient | Stampede risk, first-read miss | The default |
| Read-through | App: cache (loader inside) | (any) | Clean code, library refresh | Library lock-in | JVM apps |
| Write-through | (any) | App: cache + DB sync | Cache always fresh | Write latency, wasted writes | Read-after-write critical |
| Write-back | (any) | App: cache, async DB flush | Lowest write latency | Data loss risk, complexity | Counters, metrics |
| Write-around | App: cache → DB on miss | App: DB, skip cache | No write pollution | First-read miss after write | Bulk writes, logs |

---

## 9. Mixing Strategies (Real Systems Do This)

Pick read and write strategies independently. A typical large-scale stack:

- **Hot user data** — cache-aside read, write-through (low write rate, read-after-write matters).
- **Product catalog** — cache-aside read, write-around (catalog updates are rare; readers are many; staleness measured in seconds is fine).
- **View counts** — read directly from cache (Redis `INCR`), write-back flush every 30s to Postgres.
- **Audit log** — no cache, write-around (durability is the entire point).
- **Search results** — cache-aside read, no writes (read-only derived data).

There is no single right answer. The question is: per-workload, what are the cost of staleness, cost of latency, and cost of complexity? Pick accordingly.

---

## 10. Failure Modes and Recovery

Each pattern fails differently. Drill the failures.

### Cache-aside
- **Cache miss storm** — N concurrent misses → N DB queries. Mitigate with single-flight or `SETNX` lock keys.
- **Stale read** — bounded by TTL. Accept it or shorten the TTL.
- **Cache down** — degrades to direct DB reads. DB sees the original rate × hit-rate-ratio more load. Have a circuit breaker.

### Read-through
- **Loader thundering herd** — handled by the loader's single-flight if it's good. Verify with chaos testing.
- **Loader error** — what's cached? An exception? A null? A stale value? Pick a policy and document it.

### Write-through
- **DB write fails** — don't update cache; raise an error.
- **Cache write fails** — DB is updated, cache is stale. Invalidate the key (`DEL`), let the next reader populate.

### Write-back
- **Cache crash before flush** — pending writes lost. Bound the buffer to limit blast radius.
- **DB flush fails repeatedly** — buffer grows. Have a max size + dead-letter queue.
- **Ordering** — two writes to the same key in cache, flushed out of order. Use timestamps or sequence numbers on flush.

### Write-around
- **Read after write returns stale** — only if you forgot to invalidate. Always invalidate (or use short TTLs).

---

## 11. Worked Example: A "Read User Profile" Endpoint

Let's trace four variants of the same endpoint.

### Variant A: pure cache-aside

```python
def handle(uid):
    user = cache.get(f"user:{uid}")
    if not user:
        user = db.find(uid)
        cache.setex(f"user:{uid}", 300, user)
    return user
```

Reads: 95% cache hit → 0.5 ms. 5% miss → 5 ms.
Effective: 0.725 ms.

### Variant B: read-through with loader and refresh

```python
@cache.read_through(ttl=300, refresh_after=60)
def get_user(uid):
    return db.find(uid)
```

Reads: 95% hit → 0.5 ms. 5% miss → 5 ms (single-flight, so concurrent misses share one DB query).
Effective: 0.725 ms; tail latency is better under load.

### Variant C: write-through

```python
def update(uid, patch):
    user = db.update(uid, patch)
    cache.setex(f"user:{uid}", 300, user)

def handle(uid):
    return cache.get_or_load(f"user:{uid}", lambda: db.find(uid))
```

Reads: identical to A. Writes: pay both costs. Useful if read-after-write of the same key is on the critical path.

### Variant D: cache-aside read + write-around with version tag

```python
def update(uid, patch):
    new_version = db.update(uid, patch).version
    cache.delete(f"user:{uid}")  # invalidate; next read repopulates

def handle(uid):
    return cache.get_or_load(f"user:{uid}", lambda: db.find(uid))
```

Variant D is the production default for 90% of services. Simple, fast, hard to misuse.

---

## 12. Common Mistakes

- **Writing to cache before DB.** If DB fails, your cache is now wrong, and there's no good recovery.
- **Updating cache values on write instead of deleting them.** A concurrent reader may overwrite your update with stale DB data after your set. **Delete is safer than update.**
- **No TTL with cache-aside.** A subtle bug in the invalidate path means stale data forever.
- **Write-back for money / orders / inventory.** Don't.
- **No single-flight on misses.** First popular key wakes up → DB melted.
- **Caching `None` / "not found" without thinking.** Saves DB load on missing keys (good!) but a misbehaving writer may not invalidate. Pick a short TTL for negative entries.
- **Choosing a strategy globally instead of per-workload.** "We use write-through everywhere" — for what reason?
- **Forgetting that read-through libraries cache exceptions.** Caffeine by default doesn't, but check your stack.

---

## 13. Cheat Card

```
CACHE-ASIDE     read-default; app branches on miss; safe and simple
READ-THROUGH    cache library owns the loader; single-flight built-in
WRITE-THROUGH   sync to cache + DB; cache stays fresh; slow writes
WRITE-BACK      async to DB; fast writes; data-loss risk
WRITE-AROUND    skip cache on write; invalidate; reads stay lazy

DEFAULT RECIPE  cache-aside read + write-around + delete-on-write + TTL

ORDER           always DB-then-cache. Never cache-then-DB.

OPERATIONS      delete > update for cache invalidation
NEGATIVES       cache "not found" with a SHORT TTL

PITFALLS        stale on race, stampede on cold key, partial failure,
                using write-back for durable data

RULE            Cache-aside until you have a reason. Then justify.
```

---

## 14. Resources

### Books
- *Designing Data-Intensive Applications* — Martin Kleppmann (Ch 11 talks about derived data and caches).
- *Building Microservices* — Sam Newman (chapter on service-level caching).

### Documentation
- **AWS — Caching Strategies**: <https://docs.aws.amazon.com/AmazonElastiCache/latest/mem-ug/Strategies.html>
- **Caffeine wiki — Population & loaders**: <https://github.com/ben-manes/caffeine/wiki>
- **Redis patterns — caching**: <https://redis.io/learn/howtos/solutions/caching-architecture>

### Articles
- "Caching at Reddit" — Reddit Engineering.
- "Scaling Memcache at Facebook" — Nishtala et al., NSDI 2013 (cache-aside in the wild).
- "How we built read-through caching in our Java services" — Various Netflix tech blog posts.

### Videos
- ByteByteGo — "Top 5 Caching Strategies".
- Hussein Nasser — "Cache patterns".

### Tools
- **Caffeine** (Java), **Ristretto** (Go), **moka** (Rust).
- **Redis** (with Lua scripts for atomic cache-aside).
- **AWS ElastiCache**, **Azure Cache for Redis**, **Memorystore** (GCP).

### Adjacent reading
- [Cache Invalidation →](./cache-invalidation.md)
- [Cache Pitfalls →](./cache-pitfalls.md)
- [Distributed Caching →](./distributed-caching.md)
- [Eviction Policies →](./eviction-policies.md)
- [Idempotency →](../03-apis/idempotency.md)

---

*Previous:* [← Cache Layers](./cache-layers.md)  |  *Next:* [Eviction Policies →](./eviction-policies.md)
