# Distributed Caching (Redis, Memcached)

> **TL;DR** — A **distributed cache** is a shared in-memory store that spans multiple nodes, accessed over the network by many clients. The two dominant choices are **Redis** (single-threaded per shard, rich data structures, optional persistence, cluster mode for horizontal scale) and **Memcached** (multi-threaded, dumb key/value, no persistence, simpler ops). Modern alternatives — **DragonflyDB**, **KeyDB**, **Hazelcast** — exist but the world runs on Redis and Memcached. Architecture choices that matter: **sharding** (client-side hashing vs cluster mode vs proxy), **replication** (primary/replica vs multi-primary), **failover** (Sentinel, Cluster, managed services), and **client topology** (connection pools, retries, hedging). The classical trap is treating the cache as a database — when Redis goes down, the cache disappearing must be survivable.

---

## 1. Why Distributed

A single-node cache works fine until:
- The working set doesn't fit in one host's RAM.
- A single node's throughput (~100k–1M ops/sec for Redis) is exceeded.
- You need a cache that survives any one host failing.
- Multiple application services need a shared cache.

A distributed cache solves these by:
- **Sharding** keys across N nodes → N times the memory and throughput.
- **Replicating** data → survives node loss.
- **Centralizing** the cache so every app instance hits the same data.

The cost: network hops (~0.3–1 ms per op), operational complexity, and a new tier you must operate or pay someone to operate.

---

## 2. Memcached vs Redis (The Big Comparison)

| Aspect | Memcached | Redis |
|---|---|---|
| Threading | Multi-threaded | Single-threaded per shard |
| Data model | String values only | Strings, hashes, lists, sets, sorted sets, streams, bitmaps, hyperloglog |
| Persistence | None (pure cache) | RDB snapshots, AOF append-log (optional) |
| Replication | None native (`memcached-bridge` etc.) | Built-in primary/replica |
| Sharding | Client-side only | Native Cluster mode (16384 hash slots) |
| Max value size | 1 MB (default) | 512 MB |
| Eviction | Slab LRU, segmented LRU | 8 policies (LRU/LFU/TTL × allkeys/volatile) |
| Pub/Sub | No | Yes |
| Lua scripting | No | Yes |
| Transactions | No | MULTI/EXEC + WATCH (optimistic) |
| Memory efficiency | High for small values | Slightly less; hash field overhead |
| Throughput per node | ~1M+ ops/sec (multi-core) | ~100–200k ops/sec single shard |
| Latency | Microseconds | Microseconds, but tail can suffer on hot keys |
| Operations | Stateless, simpler | Stateful, more knobs |
| When to pick | Pure key/value lookups; raw speed | Rich data, persistence, structures |

The rule of thumb:
- **Memcached** when you cache simple values (HTML fragments, serialized rows, ID lists), value sizes are small, and you want pure speed.
- **Redis** when you need data structures (sorted sets for leaderboards, hashes for per-field updates), pub/sub, persistence, or pipelines.

Most teams today default to Redis because the feature set covers more cases, and the throughput gap rarely matters at small/medium scale.

---

## 3. Sharding Topologies

The hard problem in distributed caching is "which key lives on which node?" Three approaches.

### 3.1 Client-Side Hashing (Consistent Hashing)
The client library hashes the key, maps to a node, sends the request. The cache nodes know nothing about each other.

```
key "user:42" → hash → 0xab... → node 3 (of 8)
```

Used by: Memcached (with consistent hashing in clients like `mcrouter`, `dalli`, `pymemcache`), older Redis clients before Cluster.

**Pros**
- Simple servers (just key/value).
- Linearly scalable.
- Client library is the only smart piece.

**Cons**
- Every client must agree on the ring. Misconfiguration = split brain.
- Resizing the cluster invalidates a fraction of keys. Consistent hashing minimizes this; modulo hashing doesn't.
- No automatic failover — client must drop a node and rehash.

See [Consistent Hashing →](../04-databases/consistent-hashing.md).

### 3.2 Server-Side Sharding (Redis Cluster)
Nodes form a cluster. The key space is divided into **16384 hash slots** (`CRC16(key) % 16384`). Each primary owns a contiguous range of slots. Replicas mirror a primary.

```
slots 0–5460    → primary A (+ replica)
slots 5461–10922 → primary B (+ replica)
slots 10923–16383 → primary C (+ replica)
```

The client connects to any node and sends a command. If the key's slot lives elsewhere, the server replies `MOVED` with the right node. Smart clients cache the slot map.

**Pros**
- Built-in. No client-side hashing logic.
- Automatic failover: cluster promotes replica on primary failure.
- Reshardable online.

**Cons**
- Cluster mode is **more constrained**: multi-key operations must target keys in the same slot (use **hash tags**: `{user:42}:profile` and `{user:42}:settings` hash to the same slot).
- More moving parts: gossip, slot maps, epochs.
- Some Lua scripts and transactions don't work cross-shard.

### 3.3 Proxy-Based (mcrouter, Twemproxy, Envoy)
A proxy layer sits between clients and cache nodes. Clients connect to the proxy; the proxy handles sharding and routing.

```
client ──► proxy ──► node A / B / C
              │   ──► routes by key, can fan out, hedge
              │
              └── pool of cache nodes
```

Examples:
- **mcrouter** (Facebook's Memcached proxy) — handles routing, replication, failover, prefix-based pools, and shadow reads.
- **Twemproxy / nutcracker** — older, simpler. Twitter built it.
- **Envoy** with `memcached_proxy` filter.
- **Redis Sentinel** acts as a service-discovery layer rather than a data-path proxy.

**Pros**
- Clients are dumb. Easy multi-language.
- Centralized policy: failover, replication, pool sharding.
- Sophisticated patterns: shadow reads to a second pool, prefix-based routing, hedged reads.

**Cons**
- Extra hop = +1 RTT.
- Proxy is a new SPOF; usually run as a fleet.
- Operational overhead of another tier.

---

## 4. Replication and Failover

### Redis primary/replica
A primary takes writes; one or more replicas tail an async replication stream. Reads can hit replicas for scale.

```
   writes
   ┌────────► primary ──► replica₁ (read scale)
   │                  └─► replica₂ (read scale)
   client
   ┌────────► replica? (reads only, may be stale)
```

Replication is **asynchronous by default**: writes ack at the primary, replicas catch up. A primary failure during replication lag loses unacked writes.

### Redis Sentinel
A separate Sentinel cluster (3 or more nodes) monitors primaries and replicas. On primary failure, Sentinel orchestrates promotion of a replica and updates clients via discovery API.

```
   sentinels (gossip, monitor health)
   │
   ▼
   primary  ──►  replica  (becomes new primary on failure)
```

Sentinels handle:
- Health checks.
- Leader election among themselves (quorum).
- Failover orchestration.
- Client discovery (`SENTINEL get-master-addr-by-name`).

Sentinel is the **standard HA path for non-clustered Redis**. Used widely.

### Redis Cluster
Built-in HA via shard groups. Each primary has 0+ replicas. On primary failure, replicas in the same shard elect a new primary via gossip and slot ownership transfers.

No Sentinel needed; the cluster manages itself.

### Memcached HA
Native Memcached has no replication. Strategies:
- **Twin pools**: write to two pools in parallel; reads pick one. Loses one pool, lose 50% of cache (worst case), but graceful.
- **mcrouter replicated pools**: writes go to multiple pools; reads served from one, falls over.
- **Just accept it.** Memcached is a cache; if a node dies, the keys it held repopulate from the origin. Many large deployments (Facebook included) accept the miss spike on node loss and engineer their DB to absorb it.

### Managed services
- **AWS ElastiCache** (Redis, Memcached) — handles failover, snapshots, parameter groups.
- **Azure Cache for Redis** — enterprise tiers offer multi-region, persistence, active-active.
- **GCP Memorystore**.
- **Redis Cloud** (Redis Ltd.).

These remove most of the ops burden but cost more and limit configuration.

---

## 5. Cross-Region and Active-Active

A regional Redis cluster serves a regional app. What if your app spans regions?

### Patterns

**Regional caches with central DB**
Each region has its own Redis cluster. Cache is populated from the DB on miss. No cross-region replication. Cheapest, simplest. Staleness is bounded by TTL.

**Read-only cross-region replica**
A primary in one region; replicas in others. Reads local, writes go to the primary (cross-region). Useful when writes are rare and the primary region is the canonical writer.

**Active-Active (Redis Enterprise CRDTs)**
Multiple primaries across regions; writes accepted anywhere; conflicts resolved via CRDTs (counters, sets, etc.). Enterprise feature. Stripe-level investment.

**Event-driven invalidation**
Each region has independent cache. A change event (CDC, Kafka) reaches every region and invalidates locally. The DB might be globally replicated separately. See [Cache Invalidation →](./cache-invalidation.md).

In practice: cross-region cache replication is rarely worth the complexity. Most teams replicate the DB and let regional caches populate independently.

---

## 6. Client Topology

The client library matters as much as the server. Production clients handle:

- **Connection pooling** — TCP setup is expensive; reuse connections. Typically 10–100 per app instance.
- **Pipelining** — batch many commands in one round trip. Hugely improves throughput.
- **Async I/O / non-blocking** — Lettuce (Java), aiocache (Python), ioredis (Node).
- **Retries with backoff** — short retry on transient errors. Be careful: retries can amplify load during incidents.
- **Hedging** — for read-heavy workloads, fire to two replicas, take the first response. Trades RPS for tail latency. Memcached's mcrouter does this.
- **Circuit breaker** — when Redis is hot, fall back to direct DB reads with degraded performance rather than queueing.
- **Local cache (L1)** — a small in-process cache in front of Redis. Eats 80% of hot-key load. See [Cache Layers →](./cache-layers.md).
- **Topology refresh** — for cluster mode, periodically refresh the slot map and replica list.

Bad client behavior amplifies incidents. The most common: a client that doesn't time out, blocks a request thread waiting for a slow Redis, and runs out of threads while Redis is recovering. **Set tight timeouts: 50–200 ms.** Failing fast is the right answer.

---

## 7. Memory Sizing and Hot Keys

### How much memory?
```
memory ≈ working_set × (1 + overhead) × replication_factor + headroom
```

- Working set: count × avg size.
- Overhead: Redis has per-key overhead (~50 bytes) plus structure-specific cost. A hash with many small fields is much more efficient than millions of separate keys.
- Replication factor: count primaries + replicas.
- Headroom: 25–30% to avoid running at the edge.

For Redis, monitor `used_memory_dataset` (the part that's actually data) vs `used_memory` (data + bookkeeping). The ratio gives a sense of overhead.

### Hot keys
A single key receiving disproportionate traffic — e.g., the trending tweet ID, the home page key, a "user:counts" key. In a sharded cluster, that one key lives on one node, which gets hammered.

Symptoms: one node at 90% CPU while others idle.

Mitigations:
- **Local L1 cache** absorbs the bulk before it hits Redis.
- **Replicate the key across multiple keys** (key sharding): write to `key:0`, `key:1`, ..., `key:N`. Reads pick one randomly. Works for read-only or aggregable values.
- **Read from multiple replicas** for the hot key (`READONLY` mode).
- **Move the hot key off the cache entirely** for that path — sometimes it belongs in a CDN or a precomputed file.

See [Cache Pitfalls →](./cache-pitfalls.md) for more.

---

## 8. Operational Concerns

### What you watch
- **Hit rate** — per service / per key prefix. Drops are signals.
- **Latency p50/p95/p99/p999** — Redis usually has flat p99; if p999 spikes, you have either a slow command (`KEYS`, big `SMEMBERS`) or a hot key.
- **Memory usage and eviction rate** — eviction > 0 means working set exceeds cache.
- **Replication lag** — Redis exposes `master_repl_offset` and replica offsets. Lag > seconds = warning.
- **Connection count** — clients leaking connections is a classic outage.
- **Command stats** — `INFO commandstats`. Hot or slow commands jump out.

### What kills you in production
- **`KEYS *` in prod** — single-threaded scan over all keys; blocks every other client.
- **Big `MGET` / `MSET` / `HGETALL`** — blocking ops on huge values.
- **Lua scripts that loop** — single-threaded means one slow script blocks the world.
- **`MIGRATE` during resharding** with huge keys — same problem.
- **No `maxmemory` set** — Redis runs the host OOM.
- **AOF rewrite during peak** — disk I/O storm.
- **No connection pooling** — TCP open/close per request, dies under load.
- **Network partitions** — split brain in cluster, write to wrong primary.

The single best operational rule: **never run an O(N) command on a big key in production.** Use `SCAN`, `HSCAN`, `SSCAN`, `ZSCAN` with bounded counts.

---

## 9. Persistence: Cache or Database?

Redis can persist data (RDB snapshots, AOF append-log). This makes it tempting to use as a database. **Don't.**

Why not:
- Redis was designed in-memory-first. Persistence is durable-ish, not durable.
- Snapshot frequency vs durability is a trade-off; you can lose recent writes on crash.
- Memory cost is high. SSD is far cheaper.
- Single-threaded scaling caps throughput per shard.

Where persistence IS useful:
- **Warm starts** — restart Redis without rebuilding the entire cache from origin.
- **Stateful uses inside a service** — pub/sub patterns where loss is tolerable, queue intermediates, rate-limit counters.
- **Source of truth for small things** — feature flags, session data, where the risk profile is acceptable.

But: if you'd cry if it were lost, store it in Postgres. Cache it in Redis.

---

## 10. The Alternatives

### DragonflyDB
Redis-compatible (RESP protocol), multi-threaded, claims 25× the throughput per node. Single-node replacement for Redis Cluster in many cases. Younger, but rapidly maturing.

### KeyDB
Multi-threaded fork of Redis. Now part of Snap. Drop-in compatible.

### Hazelcast / Apache Ignite
Distributed in-memory data grids, more "database-like" with SQL, transactions, near caches. Heavier than Redis; popular in enterprise Java.

### AWS DAX
DynamoDB Accelerator. A purpose-built Memcached-compatible cache for DynamoDB. Useful only if you're on DynamoDB.

### Couchbase / Aerospike
Hybrid systems that started as caches and grew into databases. Couchbase has Memcached lineage. Aerospike is built for high-throughput key-value with low-latency SSDs.

For a green-field system in 2026, Redis or DragonflyDB cover 95% of needs.

---

## 11. Worked Example: Sizing a Redis Tier

Service: e-commerce product details cache. 50M unique products, 500 bytes/product, 10:1 read:write ratio, 100k RPS reads at peak, 99% hit-rate target.

**Working set**
- Cold: all 50M products × 500B = 25 GB. Compressed (msgpack): ~12 GB.
- Hot (95% of traffic): top 5M products → 2.5 GB.
- Plan capacity for cold to support a recent-launch / catalog refresh.

**Throughput**
- 100k RPS × ~0.5 ms cache hit = manageable from a few primaries.
- Single Redis primary: ~100–200k ops/sec. With cluster of 3 shards: 300k+ ops/sec headroom.

**Topology**
- 3 primaries, each with 1 replica = 6 nodes.
- 16 GB per node × 6 = 96 GB total, 32 GB per primary (24 GB usable for data + 8 GB headroom).
- Hash slots evenly across primaries.
- Use hash tags to co-locate keys within the same product (`{product:42}:detail`, `{product:42}:reviews`).

**TTL / eviction**
- TTL 5 min on positive entries.
- `allkeys-lru` eviction.
- Jitter TTL ±60s.

**Client topology**
- App pods have in-process Caffeine L1 (10k entries, 30s TTL).
- Lettuce client, async, connection pool of 32 per pod.
- Timeout 100 ms per call. Circuit breaker opens after 50 fails / 10 sec.
- Fallback to direct DB read on circuit-open, with a 1k QPS limiter to protect the DB.

**Monitoring**
- Per-shard hit rate, p99 latency, evictions/sec.
- Slow log (`SLOWLOG`) alerted on any > 10 ms.
- Memory usage > 80% triggers capacity review.

This is a typical "small but real" Redis tier. Most production setups look like this with different numbers.

---

## 12. Common Mistakes

- **Treating Redis as a database** — losing data when a node crashes and a replica isn't current.
- **No `maxmemory`** — Redis grows until OOM.
- **Using `KEYS *`** — O(N) command in production. Use `SCAN`.
- **Storing huge values** — > 100 KB per key turns hot keys catastrophic. Split or move to S3.
- **No connection pooling** — TCP storm under load.
- **Long-running Lua scripts** — block every other client. Single-thread is unforgiving.
- **No circuit breaker** — when Redis is down or slow, the app blocks and OOMs on request threads.
- **Synchronous cross-region writes to a remote Redis** — adds 100s of ms to your write path.
- **Misconfigured Cluster** — hash tag oversight makes multi-key transactions fail in surprising ways.
- **Ignoring the L1** — every read goes to Redis even when in-process caching would absorb 90% of it.
- **Spinning up Memcached without `binary` protocol and SASL** — text protocol leaks data; no auth = open buffet.

---

## 13. Cheat Card

```
WHEN MEMCACHED   simple K/V, raw throughput, multi-thread
WHEN REDIS       data structures, persistence, pub/sub, scripts

TOPOLOGY
  Single shard           tiny scale, fits in one host
  Sentinel + replicas    HA without sharding
  Cluster (16384 slots)  HA + horizontal scale
  Proxy (mcrouter)       Memcached at scale

CLIENT
  pool 10–100 conns / pod
  timeout 50–200 ms
  circuit breaker on failure
  in-process L1 in front

OPS
  set maxmemory + allkeys-lru
  never KEYS *; use SCAN
  watch hit rate, evictions, p99, repl lag
  slow log: nothing > 10 ms

SIZING
  working_set × (1 + overhead) × replicas + 25% headroom
  hash tags to co-locate multi-key ops

NEVER  treat it as a database; store strict source of truth in Postgres
```

---

## 14. Resources

### Books
- *Redis in Action* — Josiah Carlson.
- *Designing Data-Intensive Applications* — Martin Kleppmann.

### Documentation
- **Redis Cluster spec**: <https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/>
- **Memcached wiki**: <https://github.com/memcached/memcached/wiki>
- **AWS ElastiCache**: <https://docs.aws.amazon.com/AmazonElastiCache/>
- **mcrouter**: <https://github.com/facebook/mcrouter/wiki>

### Papers and articles
- "Scaling Memcache at Facebook" — Nishtala et al., NSDI 2013.
- "Redis Cluster: A pragmatic approach to high availability and scalability" — antirez (Redis author).
- "DragonflyDB architecture" — Roman Gershman.

### Videos
- ByteByteGo — "Redis Explained".
- Hussein Nasser — "Memcached vs Redis", "Redis Internals".

### Tools
- **Redis** (OSS), **DragonflyDB**, **KeyDB**.
- **Memcached** + **mcrouter**.
- **Lettuce** (Java), **ioredis** (Node), **redis-py** (Python), **go-redis** (Go).
- **redis-cli --bigkeys**, **redis-cli --hotkeys**, **redis-cli --latency**.

### Adjacent reading
- [Redis Deep Dive →](./redis-deep-dive.md)
- [Cache Pitfalls →](./cache-pitfalls.md)
- [Cache Invalidation →](./cache-invalidation.md)
- [Consistent Hashing →](../04-databases/consistent-hashing.md)
- [Key-Value Stores →](../04-databases/key-value-stores.md)
- [Replication →](../04-databases/replication.md)

---

*Previous:* [← Cache Invalidation Patterns](./cache-invalidation.md)  |  *Next:* [Redis Deep Dive →](./redis-deep-dive.md)
