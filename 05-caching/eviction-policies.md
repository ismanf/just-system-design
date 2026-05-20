# Cache Eviction Policies (LRU, LFU, FIFO, ARC, TTL)

> **TL;DR** — When the cache is full and a new item needs space, **eviction** decides what to throw out. The classic policies are **LRU** (evict least recently used), **LFU** (evict least frequently used), **FIFO** (evict oldest insertion), **TTL** (time-based expiry, often combined with another policy), and **ARC** (Adaptive Replacement Cache — balances recency and frequency). Modern caches (Caffeine, Ristretto) use **TinyLFU** or **W-TinyLFU**, which combine a frequency sketch with a small LRU admission window — they routinely beat plain LRU by 10–30% on real workloads. Pick policy based on whether your workload is **recency-biased** (newest items dominate), **frequency-biased** (some items are perennial favorites), or both.

---

## 1. Why Eviction Exists

A cache has finite memory. Sooner or later you fill it. Every new item you insert past the limit forces an existing item out. The eviction policy is the rule that picks which one.

The goal: **maximize hit rate** for the workload. A perfect oracle would evict whichever item won't be touched again soonest (Bélády's algorithm — provably optimal, impossible to implement). Real policies are heuristics that approximate the oracle using either:
- **Recency** — items used recently are likely to be used again (temporal locality).
- **Frequency** — items used a lot are likely to be used a lot more.
- **Both** — most modern policies.

There's a separate axis to evict on: **age** (TTL). TTL evicts items that have lived too long regardless of usage. Most production caches combine: TTL bounds staleness, and a policy like LRU bounds memory.

---

## 2. The Classic Policies

### 2.1 FIFO (First-In, First-Out)
Evict whichever item was inserted longest ago. No matter how often it's used.

```
[A B C D E F G]   ← insertion order
 ↑ evict next if full
```

**Pros**
- Trivial to implement. A queue.
- O(1) for everything.

**Cons**
- Ignores access pattern entirely. If the oldest item is the most popular, you just evicted your hottest cached value.
- Bélády's anomaly: more cache can give *worse* hit rate.

**When to use** — almost never as a primary cache policy. Common as a memory-management heuristic in OS contexts. Sometimes useful for time-window caches where insertion order *is* the freshness signal (e.g., a feed cache where old items are no longer interesting).

### 2.2 LRU (Least Recently Used)
Evict the item that hasn't been accessed for the longest time.

```
access:  A B C A D A B E F   (most recent access first)
queue:   F E B A D ... → evict the tail
```

The standard implementation is a hash map + doubly-linked list:
- Hash map: `key → node`. O(1) lookup.
- Doubly-linked list: most-recent at head, least-recent at tail. O(1) move-to-head, O(1) evict-tail.

```python
class LRU:
    def __init__(self, capacity):
        self.cap = capacity
        self.cache = OrderedDict()

    def get(self, k):
        if k not in self.cache:
            return None
        self.cache.move_to_end(k)
        return self.cache[k]

    def put(self, k, v):
        if k in self.cache:
            self.cache.move_to_end(k)
        self.cache[k] = v
        if len(self.cache) > self.cap:
            self.cache.popitem(last=False)
```

**Pros**
- Captures temporal locality, which most workloads have.
- O(1) ops, low overhead.
- Easy to reason about.

**Cons**
- **Scan-resistant? No.** A single big scan of cold data evicts your entire hot working set. Catastrophic for OLTP DBs scanning logs.
- Ignores frequency. A trending tweet is no different from a one-time visitor.
- Contention in concurrent implementations — every read mutates the list head. Locking hurts throughput.

**When to use** — the default for most application caches. Redis with `allkeys-lru` is fine for 90% of cases.

### 2.3 LFU (Least Frequently Used)
Evict the item with the fewest accesses.

```
key  accesses
 A      120
 B       80
 C        4   ← evict
 D       45
```

**Pros**
- Keeps perennial favorites (homepage, top trending products) regardless of when they were last touched.
- Better than LRU on heavy-tailed Zipfian workloads.

**Cons**
- **Stale popularity** — an item that was hot a month ago still has a huge count and won't be evicted, blocking newer hot items. Solution: count decay (see [TinyLFU](#26-tinylfu--w-tinylfu-modern-default)).
- More expensive: a counter per entry, and eviction requires finding the min.
- The first access of a new item starts at 1; popular new items can be evicted before they have time to accumulate counts.

**When to use** — workloads with stable popularity distributions: catalog data, popular content, search-query suggestions.

### 2.4 LRU-K (LRU with K-th-most-recent reference)
Track the time of the *K-th* most recent access (not just the last). Evict the item with the oldest K-th access. K=1 is LRU; K=2 is the most common.

This handles "tourists" — items accessed once and never again — by giving more weight to repeated access.

**Pros**
- Scan-resistant: a one-time scan only sets the 1st-access time; the 2nd-access time stays old, so scanning items don't displace hot items.
- Better hit rate than LRU on mixed workloads.

**Cons**
- More memory per entry (K timestamps).
- Mostly historical — superseded by TinyLFU in modern systems.

### 2.5 ARC — Adaptive Replacement Cache
Two LRU lists: **T1** (recently-added, seen once) and **T2** (frequently-accessed, seen multiple times). Plus two "ghost" lists (B1, B2) that remember recently-evicted keys but not their values. Hits on ghost entries adapt the boundary between T1 and T2.

```
| T1 (recency) | T2 (frequency) |    ← actual cache
| B1 (ghost)   | B2 (ghost)     |    ← recently evicted keys only
```

Effectively: ARC self-tunes the recency/frequency mix based on what would have helped recently.

**Pros**
- Scan-resistant.
- Adapts to workload changes.
- Patented by IBM (originally) — which is why it's not in everything. The patent has since expired.

**Cons**
- More complex than LRU. Higher per-op cost.
- Not universally better than W-TinyLFU on real workloads.

**Used by** — ZFS ARC (where the policy gets its name), various enterprise storage systems.

### 2.6 TinyLFU / W-TinyLFU (Modern Default)
Maintain a small **frequency sketch** (a count-min sketch with periodic aging) over all keys, plus a small LRU **admission window**. New items enter the window first; only if their frequency is competitive do they get admitted to the main cache.

```
new key → [ Window LRU ] → admission filter (TinyLFU sketch) → [ Main cache ]
                                                                ↑ probabilistic
                                                                  decision
```

The sketch ages by halving counters periodically. This handles the "stale popularity" problem of pure LFU.

**Pros**
- 10–30% higher hit rate than LRU on most real workloads.
- Scan-resistant by design.
- O(1) ops, low memory overhead per key.
- Concurrent-friendly.

**Cons**
- More complex implementation.
- The sketch is probabilistic — can occasionally make wrong calls.

**Used by** — Caffeine (Java), Ristretto (Go), moka (Rust). If you're starting today and your language has a Caffeine-equivalent, use it.

---

## 3. TTL (Time To Live)

TTL is not really an eviction *policy* in the LRU sense — it's an **expiry** rule. Items expire at a wall-clock deadline regardless of usage.

```
SET user:42  {...}  EX 300       # Redis: expire in 300 seconds
```

You almost always combine TTL with an eviction policy:
- TTL bounds **staleness** (max time before refresh).
- Eviction bounds **memory** (max items in cache).

Redis policies that combine the two:
- `volatile-lru` — evict by LRU, but only keys with a TTL set.
- `volatile-lfu` — same, LFU.
- `volatile-ttl` — evict whichever has the soonest expiry.
- `allkeys-lru` / `allkeys-lfu` — evict by LRU/LFU regardless of TTL.
- `noeviction` — reject writes when memory is full (safest, scariest).

### TTL design patterns
- **Fixed TTL** — simplest. All keys live N seconds. Easy reasoning, but stampedes possible if many keys expire together.
- **Jittered TTL** — `ttl = base + random(0, jitter)`. Avoids synchronized expiry. Cheap insurance.
- **Sliding TTL** — TTL renews on access. Useful for "active session" data. Risk: keys never expire, no freshness floor. Combine with a hard cap.
- **Two-tier TTL** — soft TTL triggers async refresh, hard TTL evicts. Underpins "stale-while-revalidate." Excellent for hot keys.

```python
# soft + hard TTL pattern
def get_with_swr(k):
    v = cache.get(k)
    if v is None:
        return load_and_set(k)
    if v.age > soft_ttl:
        # serve current value; refresh in background
        schedule_refresh(k)
    return v.value
```

---

## 4. Comparison Table

| Policy | Captures recency | Captures frequency | Scan-resistant | Adaptive | Memory overhead | Real-world fit |
|---|---|---|---|---|---|---|
| FIFO | ✗ | ✗ | ✗ | ✗ | None | Niche |
| LRU | ✓ | ✗ | ✗ | ✗ | 2 pointers/entry | Default |
| LFU | ✗ | ✓ | partial | ✗ | counter/entry | Stable popularity |
| LRU-K | ✓ | partial | ✓ | ✗ | K timestamps | Mixed reads |
| ARC | ✓ | ✓ | ✓ | ✓ | 2 lists + ghosts | Storage / mature |
| TinyLFU | partial | ✓ | ✓ | partial | sketch + window | Modern default |
| W-TinyLFU | ✓ | ✓ | ✓ | ✓ | sketch + window | Caffeine/Ristretto |
| TTL (alone) | — | — | — | — | timestamp/entry | Always combine |

---

## 5. Picking a Policy

A simple decision tree:

```
Is your library Caffeine / Ristretto / moka?
  └─ YES → W-TinyLFU. Done.

Is your cache Redis?
  ├─ Default workload      → allkeys-lru
  ├─ Stable popularity     → allkeys-lfu (Redis 4+)
  ├─ All keys have TTL     → volatile-ttl or volatile-lru
  └─ Critical, no eviction → noeviction (and ALARM on OOM)

Is your cache Memcached?
  └─ Slab-LRU with hot/warm/cold tiers (default since 1.5.x).
     Tune slab classes if you have wide key-size distribution.

Is your cache an OS/DB buffer pool?
  └─ Postgres: clock-sweep (LRU approximation, scan-resistant).
     InnoDB: midpoint LRU (similar idea — newly-loaded pages
     enter mid-list, scan pages don't poison the hot end).
```

The strongest signal for choosing is the **workload's access distribution**. Plot the access frequency of your keys for 24h. If it's:
- Heavy-tail / Zipfian → LFU or TinyLFU wins.
- Recency-dominant (timeline data, news) → LRU is fine.
- Mixed / changing → ARC or W-TinyLFU.
- One scan periodically wipes you out → anything scan-resistant.

---

## 6. Hit-Rate Curves: The Picture

For a given workload, plotting hit rate vs cache size shows two things:
- The **knee** — where additional memory stops paying back.
- The **gap** between policies.

```
hit rate
   1.0 ┤              ┌──── W-TinyLFU
       │           ┌──┘ ┌────  LRU
   0.8 ┤        ┌──┘ ┌──┘
       │     ┌──┘ ┌──┘
   0.6 ┤   ┌─┘ ┌──┘
       │ ┌─┘┌──┘
   0.4 ┤─┘┌─┘
       │┌─┘
   0.2 ┤┘
       └─────────────────────────► cache size (relative to working set)
       0    25%   50%   75%   100%
```

Two takeaways:
- Doubling cache size has rapidly diminishing returns past the knee.
- The right policy can be worth more than 2× the memory.

Caffeine ships with benchmarks against several real-world traces (search, OLTP, web). On the search trace, W-TinyLFU at 1% of working set roughly matches LRU at 4–5%. Big practical difference.

---

## 7. Worked Example: A Simulated Trace

Imagine 10 distinct keys, accesses follow Zipf (skewed). Cache size: 3.

```
Access trace: A A B A C B A D A B C E A B F A B C D A ...
              (A and B dominate; rare misses on C/D/E/F)

FIFO        evicts A first when full (anomaly: A is hottest)
LRU         keeps {A, B, C}; thrashes on D/E/F
LFU         locks in {A, B} forever; third slot churns
ARC         splits: {A, B in T2}, {recent in T1}; smooth
W-TinyLFU   admits via window; A/B dominate quickly; smooth
```

The pattern: FIFO is bad; LRU is good-with-thrashing; LFU is good-but-rigid; ARC and W-TinyLFU are best.

---

## 8. Cost: CPU, Memory, Concurrency

Eviction policies have ops-side cost.

| Policy | Per-op CPU | Memory/entry | Lock contention |
|---|---|---|---|
| FIFO | ~constant | 1 pointer | Low |
| LRU | small | 2 pointers | **High** (every read mutates head) |
| LFU | small | counter | Medium |
| ARC | medium | several | Medium |
| W-TinyLFU | small | sketch (amortized) | **Low** (Caffeine: lock-free reads) |

The "every read mutates the list head" problem is the reason plain LRU caches struggle in highly concurrent code. Caffeine and Ristretto avoid this by using **per-thread ring buffers** that batch the access events and apply them in bulk. The result: read throughput ~80× higher than naive LRU.

If you've ever profiled a Java app and seen `ConcurrentHashMap.compute` lighting up under your LRU implementation, that's the contention cost. Move to Caffeine.

---

## 9. Redis Eviction in Practice

Redis is the most commonly-tuned cache. Key facts:

- `maxmemory N` — sets the limit. Without this, Redis grows until it OOMs.
- `maxmemory-policy P` — the policy.
- Default is `noeviction`. **This is a footgun**: writes start failing once full. Always set this to `allkeys-lru` (or LFU) for a cache.
- Redis 4+ added LFU with proper counter aging.
- Redis 6+ improved the LRU sample size (`maxmemory-samples`, default 5). Higher values approximate true LRU better; default is usually fine.

Redis's LRU is *approximate*: it samples N keys randomly, picks the least-recently-used among them, evicts. Faster than true LRU at almost identical hit rate.

```redis
> CONFIG SET maxmemory 4gb
> CONFIG SET maxmemory-policy allkeys-lru
> CONFIG SET maxmemory-samples 10        # better quality LRU
```

---

## 10. Memcached Slabs and LRU

Memcached uses **slab allocation**: memory is pre-divided into fixed-size slab classes (chunks). Each slab class has its own LRU.

Consequence: eviction is *per-slab-class*, not global. If your application stores wildly different key/value sizes, you can have one slab class evicting hot items while another sits half-empty. Mitigation: rebalance slabs (`memcached -L`, `lru_crawler`).

Modern Memcached (1.5+) introduced **segmented LRU**: each slab has HOT / WARM / COLD segments. New items enter HOT; items demoted from HOT enter WARM; items in COLD get evicted. Scan-resistant and concurrency-friendly.

---

## 11. Common Mistakes

- **Using `noeviction` for a cache.** It's not a cache; it's a memory store waiting to OOM. Use a real policy unless you want eviction to be a manual operation.
- **One TTL fits all.** Hot keys get a longer effective lifetime through reuse; cold keys waste cache slots. Tune TTLs per data type, not globally.
- **No jitter on TTLs.** Synchronized expiries → thundering herd. Add `±10–30%` jitter.
- **LFU without aging.** Yesterday's hot key dominates forever. Make sure your LFU decays counters.
- **Ignoring scan workloads.** A nightly batch job that scans 50% of your DB will obliterate LRU. Either run scans through a separate connection / cache, or use a scan-resistant policy.
- **Caching big values alongside small ones in slab-based systems.** Memcached's slab classes will fight you. Either split caches or move to Redis.
- **Hit rate is great but tail latency is bad.** Miss latency dominates p99. Sometimes you need to *lower* hit rate variance, not raise the average.
- **No eviction metrics.** "How many keys did we evict in the last hour?" should be a dashboard. Spikes mean working set just exceeded capacity.

---

## 12. Cheat Card

```
POLICIES
  FIFO       insertion order; usually wrong
  LRU        last-touched; default; not scan-resistant
  LFU        most-touched; needs aging; stable popularity
  LRU-K      K-th touch; scan-resistant
  ARC        adaptive recency+frequency; ZFS-style
  W-TinyLFU  modern default; Caffeine/Ristretto

REDIS
  allkeys-lru        most caches
  allkeys-lfu        catalog-like popularity
  volatile-ttl       when keys have meaningful TTLs
  noeviction         NOT a cache. Avoid.

TTL
  bound staleness, not memory
  jitter (±10–30%) avoids synchronized expiry
  soft + hard TTL → stale-while-revalidate

PITFALLS
  no jitter; no eviction metrics; scan poisons LRU;
  LFU without aging; slab fragmentation; no max-memory

RULE  Use W-TinyLFU if your lib supports it.
      Otherwise: allkeys-lru with TTL + jitter.
```

---

## 13. Resources

### Books
- *Database Internals* — Alex Petrov (buffer pool eviction strategies in DBs).
- *Designing Data-Intensive Applications* — Martin Kleppmann.

### Papers
- "TinyLFU: A Highly Efficient Cache Admission Policy" — Einziger, Friedman, Manes (2017). The foundational paper for the modern default.
- "ARC: A Self-Tuning, Low Overhead Replacement Cache" — Megiddo & Modha, FAST 2003.
- "LRU-K Page Replacement" — O'Neil, O'Neil, Weikum, SIGMOD 1993.

### Documentation
- **Redis eviction**: <https://redis.io/docs/latest/develop/reference/eviction/>
- **Caffeine design**: <https://github.com/ben-manes/caffeine/wiki/Design>
- **Memcached slab management**: <https://github.com/memcached/memcached/wiki/UserInternals>

### Articles
- "Caffeine: A high performance caching library" — Ben Manes blog posts.
- "The cache replacement zoo" — various benchmark write-ups.

### Videos
- ByteByteGo — "LRU, LFU, ARC".
- CMU 15-721 — Buffer pool internals.

### Tools
- **Caffeine** (Java, W-TinyLFU).
- **Ristretto** (Go, TinyLFU).
- **moka** (Rust, W-TinyLFU).
- **Redis** with `--maxmemory-policy`.
- **Memcached** with segmented LRU.

### Adjacent reading
- [Cache Strategies →](./cache-strategies.md)
- [Cache Invalidation →](./cache-invalidation.md)
- [Cache Pitfalls →](./cache-pitfalls.md)
- [Redis Deep Dive →](./redis-deep-dive.md)
- [Database Indexing →](../04-databases/indexing.md)

---

*Previous:* [← Cache Strategies](./cache-strategies.md)  |  *Next:* [Cache Invalidation Patterns →](./cache-invalidation.md)
