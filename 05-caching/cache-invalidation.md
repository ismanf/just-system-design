# Cache Invalidation Patterns

> **TL;DR** — Cache invalidation is the act of telling a cache "this entry is no longer trustworthy." It's famously hard because it's a **distributed consistency problem in disguise**: you have two stores (DB + cache) and need to keep them in agreement under concurrent writes, partial failures, and replicas. The five real strategies are **TTL expiry** (always set one), **delete-on-write** (the default), **write-through** (cache participates in the write), **versioned keys** (no invalidation — change the key), and **event-driven invalidation** (CDC / pub-sub fans out updates). Most production systems combine TTL + delete-on-write + versioned keys. Watch out for the read-modify-cache race, the herd that follows a flush, and the cross-region inconsistency window.

---

## 1. Why It's Hard

Cache invalidation isn't hard because the *operation* is hard. `DEL key` is one syscall. It's hard because:

1. **Two stores must agree** — the cache and the source of truth. Any non-atomic operation between them has a failure window.
2. **Multiple readers and writers race** — a stale read can repopulate the cache after a delete.
3. **Replicas and edges lag** — a cache near London may have the old value when New York already invalidated.
4. **Negative caches and TTLs interact** — cached "not found" entries can hide newly-created data.
5. **The cost of being wrong is workload-dependent** — sometimes 5 minutes of staleness is fine; sometimes 50 ms is unacceptable.

Phil Karlton: "There are only two hard things in Computer Science: cache invalidation and naming things." He wasn't being cute.

---

## 2. The Strategies

### 2.1 TTL Expiry (always have one)
Set an expiry. The cache evicts the value when time runs out. The cache may serve stale data up to that bound, then naturally repopulates from the source.

```
SET user:42 {...} EX 300   # invalidated 300s after this set
```

This is not an "invalidation" in the sense of an explicit action — it's *passive* invalidation. The point is: **every cached value should have a TTL**, even if you also delete-on-write. The TTL is your insurance against forgotten or failed invalidations.

A common production failure: deploy a new code path that updates the DB but forgets to delete the cache key. The TTL bounds the bug. Without TTL, the wrong value lives forever.

Tradeoffs:
- Long TTL → less origin load, more staleness.
- Short TTL → more origin load, less staleness.
- Pick based on the staleness contract your users tolerate.

### 2.2 Delete-on-Write (the default)
On every write to the DB, delete the corresponding cache key. The next read repopulates with fresh data.

```python
def update_user(uid, patch):
    db.update(uid, patch)            # 1) write source of truth
    cache.delete(f"user:{uid}")      # 2) invalidate cache
```

This is the "cache-aside + write-around" pattern (see [Cache Strategies →](./cache-strategies.md)).

**Why delete, not update?**
- A concurrent reader may have just fetched the *old* DB row and is about to `SET` it after your `DEL`. You can't prevent this race, but delete is safer than update because if it happens, the cache holds an *old* value — same as before your write. If you had instead `SET` a new value, a stale reader could overwrite it.
- Delete is idempotent. Multiple invalidations are cheap.
- Delete makes the next read an explicit miss, which surfaces in metrics — you see your cache miss rate, and that's useful.

**Order matters: DB-then-cache.** If the DB write fails, you didn't touch the cache. If the cache delete fails (and DB succeeded), the cache holds a stale value bounded by TTL.

### 2.3 Write-Through
The write goes to the cache *and* the DB synchronously. See [Cache Strategies →](./cache-strategies.md). From an invalidation perspective: the cache is updated, not invalidated.

Trade-off: the next read is faster (cache fresh), but if two writers race, you can end up with a value in cache that doesn't match the DB. Usually solved by writing the DB first, then the cache, with the DB write returning the final value.

### 2.4 Versioned Keys (skip invalidation entirely)
Embed a version or hash in the cache key. When the underlying object changes, *the key changes* — old key is orphaned, new key is fresh.

```
user:42:v17        # current
user:42:v18        # after update
```

The old `v17` will simply expire by TTL and disappear. No `DEL` needed.

The version comes from:
- A monotonic version column in the DB (`UPDATE ... SET version = version + 1`).
- A content hash (`sha256(serialized_value)`).
- A logical timestamp.

```python
def get_user(uid):
    version = db.scalar("SELECT version FROM users WHERE id = %s", uid)
    key = f"user:{uid}:v{version}"
    u = cache.get(key)
    if u is None:
        u = db.find(uid)
        cache.setex(key, 300, u)
    return u
```

**Pros**
- No race condition between writer and reader. They use different keys.
- No need for explicit invalidation in code.
- Multiple versions can coexist briefly (great for blue-green or canary deploys).
- Works across regions naturally — each region populates its current version.

**Cons**
- Need the version on the read path, which usually means a small DB lookup or a separate version cache.
- Old versions linger until TTL. Memory cost.
- Doesn't handle "delete this entity" cleanly — you can't version "not exists."

**When to use**
- Content that's expensive to serialize.
- Configuration / feature flags.
- Anything you want zero-stale-read guarantees on.

Real-world example: web asset fingerprinting (`app.f3a9b2.js`) is versioned-keys applied to HTTP caches. The technique works the same way at the data layer.

### 2.5 Event-Driven Invalidation (CDC / Pub-Sub)
Writes to the DB emit events. A subscriber listens and invalidates the corresponding cache keys.

```
   ┌─────┐
   │ App │── write ──► DB ──► CDC stream ──► invalidator ──► cache.del(...)
   └─────┘                    (Debezium,                     (everywhere)
                               Kafka)
```

Real-world wiring options:
- **Debezium + Kafka** — captures Postgres/MySQL changes, emits to Kafka, a consumer invalidates the cache.
- **DynamoDB Streams** — built-in CDC, drives a Lambda that invalidates.
- **Postgres `LISTEN/NOTIFY`** — simpler but limited.
- **Application-level pub-sub** — each writer publishes to Redis Pub/Sub or Kafka.

**Pros**
- The DB is the single source of truth; cache state is derived. Decoupled.
- Handles writes that bypass the cache invalidation code (a one-off SQL migration, an admin tool).
- Works across regions: replicate the event stream, each region invalidates locally.
- Fans out to many caches (CDN purge, app-tier cache, search index update).

**Cons**
- Real complexity: schemas, partitioning, ordering, replay, dead letters.
- Latency: there's a small window between commit and invalidation. Usually 10–500 ms.
- The invalidator can become a critical dependency. Plan for it being slow or down.

**When to use**
- You have multiple readers/writers across services/teams.
- You already run Kafka or have a CDC pipeline.
- Cache lives at a different location than the writer (CDN, edge, geographically distant region).

This is the pattern Stripe, Shopify, and large media sites use for keeping CDN caches in sync with backend changes.

---

## 3. The Read-Modify-Cache Race

The most common silent-bug source. Two threads:

```
Thread A:                 Thread B (writer):
  GET cache → miss          UPDATE DB
  SELECT FROM db ── reads pre-update row
                            DEL cache  ←── invalidation
  SET cache(old_value)  ←── stale write
```

Result: cache holds the *old* value after the writer's invalidation. Bug.

### Fixes
- **Short TTL** — bounds how long the stale value lives. Cheapest fix; usually sufficient.
- **Delete twice (double-delete)** — writer deletes the cache before the DB write *and* after. Reduces the race window. Stripe and Meituan have written about this pattern.
   ```python
   cache.delete(k)
   db.update(...)
   cache.delete(k)
   # optionally: time.sleep(small_delay); cache.delete(k) again
   ```
- **Versioned keys** — race becomes harmless because the slow reader writes under the old version key, which nobody reads.
- **Lock the cache key during repopulation** — `SETNX cache_lock:k 1 EX 5`. Only one repopulator at a time. Same primitive solves [thundering herd](./cache-pitfalls.md).
- **Read-through with single-flight** — Caffeine/Ristretto's loader handles this for in-process caches.

### When you really need fresh-after-write
Use the DB transaction's returning value to populate the cache atomically with the commit:

```python
with db.transaction() as tx:
    new_row = tx.update(uid, patch).returning("*")
cache.setex(f"user:{uid}", 300, new_row)
```

The DB returned the post-update row in the same call. The window between commit and set is tiny. Combine with delete-on-write before the update for safety.

---

## 4. Cross-Region and Edge Invalidation

When caches live in many regions or at the CDN edge, invalidation is harder.

### Patterns

**Broadcast purge**
Send an invalidation to every cache node. CDN APIs do this:
```
POST https://api.cloudflare.com/.../purge_cache
{"files": ["https://example.com/p/42"]}
```
Or for tags:
```
{"tags": ["product-42"]}
```
Latency: 100 ms to a few seconds across global POPs. Cloudflare claims ~1s, Fastly claims ~150 ms (their selling point).

**Tag-based invalidation**
Tag each cached value with one or more labels. To invalidate, purge by tag — clears all cached objects with that tag.

```
cache.setex("user_profile:42", 300, value, tags={"user:42"})
...
cache.invalidate_tag("user:42")  # clears anything tagged with this user
```

Fastly's "surrogate keys" and Cloudflare's "cache tags" both implement this. Saves you from listing every concrete URL — one tag invalidates fan-outs.

**Event-driven fan-out**
A DB write emits an event. A consumer per region invalidates locally. Works for both Redis and CDN. Latency: replicas-lag + processing time.

**TTL with stale-while-revalidate**
Bound staleness universally with a short TTL. After expiry, serve stale + revalidate in background. The simplest scheme, the most robust failure mode.

```
Cache-Control: max-age=60, stale-while-revalidate=600
```

### Multi-region consistency window
There's no way to make a global cache 100% consistent without paying enormous latency cost. The honest answer: pick a staleness budget (e.g., "users must see updates within 5 seconds globally") and engineer to that.

Stripe's approach (as written about by Brandur Leach): they tolerate small windows of staleness, instrument the lag, and alarm when it exceeds the SLA. The world doesn't end.

---

## 5. Negative Caching and "Not Found"

If you cache the *absence* of data, you also need to invalidate it.

```python
def get_user(uid):
    sentinel = "__missing__"
    v = cache.get(k := f"user:{uid}")
    if v == sentinel:
        return None
    if v is not None:
        return v
    row = db.find(uid)
    if row is None:
        cache.setex(k, 30, sentinel)  # short TTL for negatives
        return None
    cache.setex(k, 300, row)
    return row
```

Use a **shorter TTL for negatives** than for positives. If a user is created after a "not found" cache populates, the new user is invisible until the negative entry expires.

For high-cardinality negative spaces (random IDs that don't exist), consider a **Bloom filter** in front of the DB instead — see [Bloom Filters →](../08-distributed-systems/bloom-filters.md).

---

## 6. Cache Coherence Across Multiple Caches

Two layers of cache (e.g., in-process L1 + Redis L2) make invalidation more interesting. Deleting in Redis doesn't invalidate the L1 cache in each pod.

### Patterns

**Short L1 TTL**
Set the in-process cache TTL to seconds (5–60). Tolerate that staleness window. By far the simplest and most common.

**Pub-sub invalidation**
Publish "key X invalidated" to a Redis Pub/Sub channel. Every pod subscribes and removes from L1.

```python
# writer
db.update(...)
cache.delete(k)
redis.publish("invalidations", k)

# reader pods
async for msg in redis.subscribe("invalidations"):
    l1_cache.delete(msg)
```

Caveat: Pub/Sub is **at-most-once**. A disconnected subscriber misses the message. Combine with short TTL on L1 as a safety net.

**Redis Client-Side Caching (RESP3 tracking)**
Redis 6+ supports server-assisted client-side caching: the server tracks which keys each client has cached, and pushes invalidations when those keys change. Used in Lettuce, redis-py, jedis.

```python
# pseudo
redis = Redis(client_tracking=True)
v = redis.get("k")   # cached in client
# when "k" changes server-side, client gets push notification, evicts locally
```

This is the cleanest pattern for L1+L2 with Redis as L2 and is a recent feature worth knowing about.

---

## 7. CDN Invalidation Specifically

CDN cache invalidation is its own subdiscipline because:
- The cache is *spread* across hundreds of POPs.
- Network latency makes synchronous invalidation impractical.
- Many CDNs charge for purges (per-request fees), so you don't want to purge constantly.

### Three main approaches
- **TTL-only** — let things expire. Set short TTLs (seconds–minutes) for things that change often. Long TTLs for immutable assets (fingerprinted JS). Cheapest, simplest.
- **Explicit purge by URL** — `POST /purge {url}`. Use for specific paths after a deploy or edit.
- **Tag-based / surrogate-key purge** — tag responses with labels; purge by label. Most flexible at scale.

```
# Fastly surrogate key
Surrogate-Key: product-42 category-shoes brand-nike

# Later, purge everything tagged product-42:
POST /service/{id}/purge/product-42
```

### Soft purge
Mark cached entry as stale rather than removing it. The CDN serves stale-with-revalidation. Survives origin downtime. Fastly's default purge is soft purge.

### When you must guarantee global freshness
You can't truly. Choose a strategy:
- Use very short TTL with stale-while-revalidate.
- Bypass cache for the small set of authenticated/personalized requests where freshness matters most.
- Use server-push (WebSockets / SSE) for live updates and treat the cache as a starting snapshot.

---

## 8. Worked Example: Updating a Product Page

User edits product 42's price. We want all caches refreshed.

```
1. Admin saves change → POST /admin/products/42
2. App writes DB:
     UPDATE products SET price = 19.99, version = version + 1 WHERE id = 42

3. Invalidate app-tier cache:
     redis.del("product:42")        # cache-aside key

4. Publish change event:
     kafka.produce("products.changed", key=42, value={id:42, version:18})

5. CDN purge by tag:
     cdn.purge_tag("product-42")    # invalidates /p/42, /p/42/reviews,
                                    # category pages tagged with product-42

6. Cross-region: each region's app subscribes to "products.changed",
   invalidates its own Redis copy.

7. L1 caches per-pod: short TTL (30s) auto-expires. Or pub-sub.

8. Re-render path:
     - First reader misses Redis → reads DB (v=18) → populates cache.
     - First request through CDN misses → fetches from origin → caches.
     - Steady-state restored.
```

End-to-end, users see the new price within seconds. The price is bounded by the longest cache layer's stale window.

---

## 9. Idempotent Invalidations and Retries

Invalidation must be **idempotent**. Deleting the same key twice should be safe. This matters because:
- Event streams may deliver duplicates.
- Failed invalidations should be retried.
- Multiple writers may invalidate concurrently.

Bad: cache update that's an increment, or a delete that requires the value to be a specific version. Good: `DEL k`, `SET k v` (with TTL), CDN tag purge.

For multi-step invalidations (DEL Redis + CDN purge + emit event), use either:
- A workflow / outbox pattern — the invalidation is a record in your DB, written transactionally with the update. A worker reads the outbox and performs the invalidations. Failures retry. See [Outbox Pattern →](../07-messaging/outbox-pattern.md).
- A CDC pipeline that derives invalidations from the DB log itself, so even bypassing the app still triggers invalidation.

---

## 10. Common Mistakes

- **No TTL at all.** If anything goes wrong with explicit invalidation, the bug is permanent.
- **Update cache instead of delete.** Races leave you with stale values you wrote yourself.
- **Cache-first, DB-second writes.** DB fails → cache lies forever.
- **Forgetting negative caches.** New records invisible until the negative TTL expires.
- **Not invalidating fan-outs.** Updating a product invalidates `product:42` but not `category:shoes:top10`. Tag-based invalidation or explicit cascade lists fix this.
- **Synchronous invalidation across regions on the hot path.** Adds 100s of ms to a write. Almost never worth it.
- **Synchronizing all TTLs.** Mass expiry = thundering herd. Add jitter.
- **Treating Pub/Sub as reliable.** It's at-most-once. Layer a TTL underneath.
- **Hand-listing every URL to purge on the CDN.** Becomes unmaintainable. Use tags from day one.
- **Ignoring the CDC stream when bypassing the app.** A bulk SQL migration that skips the cache invalidation path leaves cache stale until TTL.

---

## 11. Cheat Card

```
PATTERNS
  TTL              always have one; jitter ±10–30%
  delete-on-write  default; DB then cache; safer than update
  write-through    cache + DB sync; freshest but races
  versioned keys   key changes on update; race-free
  event-driven     CDC / pub-sub; cross-region, fan-out

THE RACE
  reader and writer race → reader can repopulate stale
  fixes: short TTL, double-delete, versioned keys, lock-on-load

CROSS-REGION
  broadcast purge        explicit, milliseconds–seconds
  tag-based purge        scales to fan-out; CDN surrogate keys
  TTL + SWR              simplest; pick a staleness budget
  Redis tracking         server-pushed invalidations to clients

PITFALLS
  no TTL; cache-first writes; cached errors;
  negative caches blocking new data; sync cross-region invalidations;
  CDC bypass; pub-sub-as-only-mechanism

RULE  Delete on write. TTL on everything. Treat cache as
      eventually-consistent and bound the lag.
```

---

## 12. Resources

### Books
- *Designing Data-Intensive Applications* — Martin Kleppmann (derived data, change data capture).
- *Database Internals* — Alex Petrov.

### Documentation
- **Redis client-side caching**: <https://redis.io/docs/latest/develop/use/client-side-caching/>
- **Fastly surrogate keys**: <https://docs.fastly.com/en/guides/getting-started-with-surrogate-keys>
- **Cloudflare cache tags**: <https://developers.cloudflare.com/cache/how-to/purge-cache/purge-by-tags/>
- **AWS CloudFront invalidations**: <https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/Invalidation.html>

### Articles
- "Online migrations at scale" — Jacqueline Xu, Stripe Engineering Blog.
- "How we ship code fast and safely" — Stripe (uses versioned keys broadly).
- "Cache invalidation really is one of the hardest things" — Marc Brooker.
- "Improving site stability with Surrogate Keys" — Fastly.

### Videos
- ByteByteGo — "Cache Invalidation Patterns".
- Martin Kleppmann — "Turning the database inside-out" (CDC + derived state).

### Tools
- **Debezium** — CDC for Postgres, MySQL, MongoDB.
- **Kafka** + Connect — invalidation event bus.
- **Fastly**, **Cloudflare**, **CloudFront** — tag-based purge.

### Adjacent reading
- [Cache Strategies →](./cache-strategies.md)
- [Cache Pitfalls →](./cache-pitfalls.md)
- [Change Data Capture (CDC) →](../04-databases/cdc.md)
- [Outbox Pattern →](../07-messaging/outbox-pattern.md)
- [Bloom Filters →](../08-distributed-systems/bloom-filters.md)
- [Consistency Models →](../08-distributed-systems/consistency-models.md)

---

*Previous:* [← Eviction Policies](./eviction-policies.md)  |  *Next:* [Distributed Caching →](./distributed-caching.md)
