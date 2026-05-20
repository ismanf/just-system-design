# Key-Value Stores (Redis, DynamoDB)

> **TL;DR** — A **key-value store** is the simplest possible database: a giant distributed `Map<key, value>` with sub-millisecond `GET`/`PUT`/`DELETE`. The simplicity is the feature — it makes them blazing fast, easy to scale horizontally, and trivial to reason about. The catch: any access pattern beyond "look up by primary key" is awkward. They're the workhorses of caching (Redis, Memcached), feature flags, session stores, rate limiters, leaderboards, and at hyperscale, primary stores too (DynamoDB).

---

## 1. The Model

```
key  ─────► value
"user:42"    {"name":"Ada","email":"ada@example.com"}
"session:abc" {"user_id":42,"expires":1716072600}
"rl:ip:1.2.3.4:GET:/login" 7
"feed:42"    [42, 41, 40, ...]
```

Keys are usually strings. Values can be:
- **Bytes** (Memcached).
- **Strings** (basic Redis).
- **Rich types** (Redis: lists, hashes, sets, sorted sets, streams, bitmaps, hyperloglog).
- **JSON documents** (DynamoDB items, with attributes).

Operations: `GET`, `PUT` / `SET`, `DELETE`, `INCR`, `EXPIRE`, `EXISTS`, sometimes simple atomic ops (compare-and-swap, increment).

There's no SQL, no joins, no schema enforcement.

---

## 2. The Two Big Families

### 2.1 In-memory KV (cache-grade)
- **Redis** — single-threaded event loop, rich data types, optional persistence, pub/sub, streams, Lua scripting, modules (RedisJSON, RediSearch, RedisGraph).
- **Memcached** — multi-threaded, pure bytes, no persistence, simpler.
- **KeyDB** / **Dragonfly** — Redis-compatible multi-threaded forks.

Used for: **caching**, sessions, real-time data, queues, leaderboards.

### 2.2 Disk-backed distributed KV (system-of-record-grade)
- **DynamoDB** — AWS managed, hash + range keys, single-digit-ms reads at any scale.
- **Cassandra / ScyllaDB** — usually grouped as wide-column but the KV mode is common.
- **Bigtable** — Google's foundational store. Single-row strong consistency.
- **etcd / Consul / ZooKeeper** — small KVs with strong consistency for coordination.
- **RocksDB / LevelDB** — embedded LSM-tree KVs that power half the modern databases (Kafka Streams, CockroachDB, MySQL MyRocks, etc.).
- **FoundationDB** — Apple's distributed ordered KV with ACID.

Used for: **primary storage** at scale, configuration, leader election, embedded state, time-series rollups.

---

## 3. Why They're So Fast

- **One operation = one network call**.
- No query planner.
- No index choice — the key *is* the index.
- Data fits in memory (in-memory KVs) or on SSD with LSM-tree write paths.
- Partitioning is simple: hash the key → pick the shard.

A single Redis box on commodity hardware does **100k–1M ops/sec**. DynamoDB scales linearly with provisioned capacity. A Cassandra cluster handles **millions of writes/sec across nodes**.

---

## 4. Redis — The Swiss Army Knife

Redis isn't *just* a cache. The data types make it a serious tool:

| Type | What it stores | Example use |
| --- | --- | --- |
| **String** | Up to 512 MB blob | Counters, cached HTML, JSON |
| **Hash** | Field → value map | User profile, session |
| **List** | Ordered list | Job queue (`LPUSH`/`BRPOP`) |
| **Set** | Unordered unique members | Tags, online users |
| **Sorted set** | Members with scores | Leaderboards, time-ordered feeds |
| **Bitmap** | Bits at offsets | Daily active users, presence |
| **HyperLogLog** | Approx count | Unique-visit counters |
| **Stream** | Append-only log | Pub/sub with consumer groups |
| **Geo** | Lat/lon points | Nearby search |
| **JSON** (RedisJSON module) | Documents | Mini document store |
| **Vector** (RediSearch) | Embeddings | ANN search |

### Persistence
- **RDB snapshots** — periodic fork+dump to a file. Fast restart, some data loss possible.
- **AOF** (Append-Only File) — log every write. Higher durability.
- **No persistence** — pure cache; lose it all on restart.

### High availability
- **Redis Sentinel** — automated failover for a primary + replicas.
- **Redis Cluster** — sharded, multi-primary, automatic resharding.
- **Hosted**: AWS ElastiCache, GCP MemoryStore, Redis Enterprise.

### Single-threaded gotcha
Redis processes commands serially on one core (with helpers for I/O and persistence). One slow command (`KEYS *`, large `LRANGE`, big-key `DEL`) **blocks everything**. Always:
- Use `SCAN` not `KEYS`.
- `DEL` big keys with `UNLINK` (async).
- Avoid huge values; shard them.

### Pipelining and Lua
- **Pipelining** — send many commands without waiting for replies; throughput goes from ~100k to ~1M ops/sec.
- **Lua scripts** — `EVAL` runs a Lua function atomically server-side. Perfect for "check + set + increment" patterns.

---

## 5. DynamoDB — Hash-Range Modeling

DynamoDB is the canonical hosted distributed KV. Items live in **tables** with a **partition key** (hash) and optional **sort key** (range).

```
table: orders
PK = customer_id            SK = order_id
"cust_42"                   "ord_1"   { total: 4200, status: "paid" }
"cust_42"                   "ord_2"   { total: 2100, status: "open" }
"cust_88"                   "ord_5"   { total: 9000, status: "paid" }
```

Queries you can do:
- `GET (PK, SK)` — point lookup.
- `Query` `WHERE PK = ? AND SK BETWEEN ? AND ?` — range within one partition.
- `Scan` — read entire table (avoid in production).
- **Global Secondary Indexes (GSI)** — alternate (PK, SK) views, eventually consistent.

You can't `JOIN`, `GROUP BY`, or `WHERE` on arbitrary attributes efficiently. You **design your tables around your access patterns** before you write any code. This is called **single-table design** — model every entity in one table with carefully-chosen keys.

### Capacity modes
- **Provisioned** — declare read/write units; throttled above that.
- **On-demand** — pay per request, auto-scales.

DynamoDB is best for: workloads with **predictable access patterns**, very high throughput, and a need for managed operations.

---

## 6. When to Reach for a KV Store

| Use case | Tool |
| --- | --- |
| Cache layer in front of DB | Redis / Memcached |
| Session store | Redis |
| Rate-limit buckets | Redis (with Lua / Redis Cell) |
| Leaderboards | Redis sorted sets |
| Job queue | Redis lists / streams; SQS |
| Real-time pub/sub | Redis pub/sub / streams; NATS |
| Distributed lock | Redis (Redlock) / etcd |
| Feature flags | Redis / ConfigCat / LaunchDarkly |
| Hot product lookups | DynamoDB / Redis |
| Shopping cart | DynamoDB / Redis |
| Massive-scale primary storage | DynamoDB / Cassandra / Scylla |
| Service discovery / config | etcd / Consul / ZooKeeper |
| Embedded local KV | RocksDB / LevelDB / Badger |

---

## 7. Patterns

### Cache-aside
```
read:
  v = redis.get(key)
  if v: return v
  v = db.get(key)
  redis.set(key, v, ex=ttl)
  return v

write:
  db.write(key, v)
  redis.del(key)   # or redis.set(key, v)
```

Simple and the most common. See [Cache Strategies](../05-caching/cache-strategies.md).

### Write-through / Write-behind / Read-through
Variations where the cache and the DB are coordinated by a layer above. Stronger guarantees, more complex.

### Distributed lock with Redis
```lua
-- atomic: set if not exists with TTL
SET lock:resource <random_token> NX EX 30
-- release: only delete if token matches
EVAL "..."   # compare-and-delete
```
Use **Redlock** when you need multi-node locks; the algorithm is debated — for many cases a single Redis instance is enough.

### Counters
```
INCR page_views:home
EXPIRE page_views:home 86400
```
Atomic counters at millions of ops/sec.

### Leaderboards
```
ZADD leaderboard 1500 "user:42"
ZADD leaderboard 1620 "user:88"
ZREVRANGE leaderboard 0 9 WITHSCORES
```
Top-10 in O(log N + 10).

### Time-bucketed rate limiting
```lua
local key = "rl:" .. ip .. ":" .. minute
local n = redis.call("INCR", key)
if n == 1 then redis.call("EXPIRE", key, 60) end
return n
```

---

## 8. Consistency

In-memory KVs (Redis):
- Single primary → strong consistency for one client.
- Replicas are async → reads from replicas can be stale.
- Cluster mode shards by key; cross-shard transactions are limited.

Distributed KVs (DynamoDB, Cassandra):
- **Tunable**. DynamoDB lets you choose **strongly** or **eventually** consistent reads per call. Strong reads cost more and are routed to the partition leader.
- Cassandra tunes via `ONE` / `QUORUM` / `ALL` consistency levels per query.

Coordination KVs (etcd, ZooKeeper):
- **Strict serialization** via consensus (Raft / Zab). Slow for high throughput, perfect for small "source of truth" data (cluster state, leader election).

---

## 9. Scaling

### Vertical
- One Redis on a big box scales further than people think (hundreds of GB RAM, 1M ops/sec with pipelining).

### Sharding (Redis Cluster, DynamoDB partitions, Cassandra tokens)
- Hash the key → pick a partition.
- Adding nodes rebalances. **Consistent hashing** keeps reshuffling small.
- Hot keys = hot partitions. Mitigate by adding randomness or read-replicas of the hot data.

### Replication
- Async replicas for read scaling.
- Sync replicas (or quorum writes) for durability.
- Multi-region: Active-active in Cassandra, Global Tables in DynamoDB, Geo-Replication in Redis Enterprise.

See: [Consistent Hashing](./consistent-hashing.md) · [Sharding & Partitioning](./sharding-partitioning.md).

---

## 10. The Failure Modes That Bite

- **Hot key** — one key takes all the QPS. Latency on that shard blows up.
- **Big key** — a hash/list/set with millions of members. Single-threaded engines stall during big-key ops.
- **Eviction surprises** — cache under memory pressure evicts hot keys you assumed would persist.
- **Stale cache** — TTL too long; users see old data after writes.
- **Cache stampede / thundering herd** — TTL expires for a hot key, 10k clients miss simultaneously and hammer the DB.
- **Persistence pause** — Redis `BGSAVE` fork on a huge dataset causes a multi-second latency spike.
- **Cross-shard transactions** — Redis MULTI/EXEC works only within one slot; transactions across slots aren't supported.
- **DynamoDB hot partition throttling** — a misdesigned partition key serializes traffic to one partition.
- **Network outages** for distributed KVs cause split-brain or stale reads if config is wrong.

Each of these has known mitigations — see Section 12.

---

## 11. Sizing Reality (rough)

| Tool | Latency p50 | Throughput / node | Footprint |
| --- | --- | --- | --- |
| Memcached | 0.1–0.3 ms | ~1M ops/sec | RAM-bounded |
| Redis | 0.2–0.5 ms | 100k–1M ops/sec with pipelining | RAM-bounded |
| DynamoDB | 1–10 ms | Linear with capacity | Cloud-managed |
| Cassandra / Scylla | 1–10 ms | 10k–100k+ writes/sec/node | Disk-bounded |
| etcd | 1–5 ms writes | Hundreds–few-k writes/sec | Strong consistency cost |

Always benchmark with your real workload before sizing.

---

## 12. Best-Practice Recipes

- **Always set a TTL** on cache keys to avoid forever-growth.
- **Avoid `KEYS *`** in Redis. Use `SCAN`.
- **Cap value size**. Multi-MB blobs in Redis = trouble.
- **Pipeline** when you have many small ops to the same Redis.
- **Use `UNLINK`** instead of `DEL` on big keys.
- **Set memory eviction policy** (`allkeys-lru`, `volatile-lfu`) to match your workload.
- **Pre-warm caches** on deploy if cold-start is unacceptable.
- **Add jitter to TTLs** to prevent stampedes.
- **Use SETNX-with-jitter or `request coalescing`** for stampede defense.
- **Provision for peak, not average** with DynamoDB on-demand or autoscaling.
- **Watch partition heat** — DynamoDB CloudWatch metrics by partition.
- **Test backups** — for Redis, restore from RDB/AOF in a staging env regularly.

---

## 13. KV as Primary Store — When It Works

DynamoDB, Cassandra, Bigtable can all hold the **system of record**. To make that work you must:

- **Model access patterns first** (single-table design for DynamoDB).
- **Accept lack of joins.** Denormalize ruthlessly.
- **Build indexes** for any non-PK access (GSIs in Dynamo, materialized views in Cassandra).
- **Plan idempotent writes** because retries are part of the model.
- **Embrace eventual consistency** for analytics; use strong reads for critical paths.
- **Use Streams / CDC** to feed search, analytics, and downstream services from changes.

Plenty of large SaaS run primary on DynamoDB or Cassandra (Snap, Discord, Apple iCloud parts, Netflix). Just *don't* drift into it from a relational schema without redesigning.

---

## 14. Picking the Right KV

```
Need a cache or short-lived state?
  → Redis (rich types) or Memcached (simpler).

Need coordination / locks / leader election / small strong-consistent state?
  → etcd / ZooKeeper / Consul.

Need a primary store at internet scale, hosted, no ops?
  → DynamoDB.

Need a primary store at internet scale, self-hosted, multi-DC?
  → Cassandra / ScyllaDB.

Need embedded fast local store inside your service?
  → RocksDB / Badger / LevelDB.

Need vector + KV in one box?
  → Redis (RediSearch) or pgvector + Redis.
```

---

## 15. Common Mistakes

- Treating Redis as a *durable* store without tuning persistence.
- Storing the only copy of important data in Memcached.
- Using `KEYS *` in production.
- Multi-MB values; single-threaded Redis stalls.
- Hot keys / partitions undetected until they melt.
- TTL = forever → memory grows until OOM.
- Cache stampedes after deploys when caches are cold.
- No connection pool from app to Redis.
- DynamoDB single-table design done after launch (extremely costly to retrofit).
- Reading from cache without DB fallback if cache is down → outage when cache reboots.

---

## 16. Cheat Card

```
KEY-VALUE = the simplest DB. GET / PUT / DELETE by key.

IN-MEMORY      Redis (rich types), Memcached (bytes).
                ★ cache, sessions, rate limits, leaderboards, queues.

DISTRIBUTED    DynamoDB, Cassandra, Scylla, Bigtable.
                ★ primary store at scale, key-driven workloads.

COORDINATION   etcd, ZooKeeper, Consul.
                ★ small strong-consistent state, locks, service discovery.

EMBEDDED       RocksDB, LevelDB, Badger.
                ★ local state inside a service.

DESIGN AROUND  the key. Access patterns first.
                No joins. Denormalize. Build per-pattern indexes.

REDIS GOTCHAS
  single-threaded — slow commands block everyone
  KEYS *, big keys, no TTL, no persistence config, hot keys, stampedes

DYNAMODB GOTCHAS
  partition heat, hot partition throttling, GSI cost, single-table design

SCALING
  shard by key (consistent hashing).
  add replicas for reads.
  multi-region only when you really need it.
```

---

## 17. Resources

### Books
- *Redis in Action* — Josiah Carlson.
- *Designing Data-Intensive Applications* — Kleppmann (Ch. 3 — storage engines).
- *The DynamoDB Book* — Alex DeBrie. The single best resource on DynamoDB modeling.

### Documentation
- **Redis docs** — <https://redis.io/docs/>
- **Memcached** — <https://memcached.org/>
- **DynamoDB Developer Guide** — <https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/>
- **Bigtable** — <https://cloud.google.com/bigtable/docs>
- **Cassandra** — <https://cassandra.apache.org/doc/latest/>
- **RocksDB Wiki** — <https://github.com/facebook/rocksdb/wiki>

### Articles
- "Designing for DynamoDB" — Rick Houlihan / Alex DeBrie.
- "Cache Stampede" — many blog posts; the classic problem.
- "Why is Redis fast?" — Redis blog and Hussein Nasser.
- "Redis is fast, Redis is simple" — talks by Salvatore Sanfilippo (antirez).
- "The Original Dynamo Paper" — Amazon 2007: <https://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf>

### Videos
- ByteByteGo: "Redis Explained" — <https://www.youtube.com/@ByteByteGo>
- Hussein Nasser Redis / DynamoDB deep dives — <https://www.youtube.com/@hnasr>
- AWS re:Invent talks on DynamoDB by Rick Houlihan — gold.

### Tools
- **redis-cli**, **RedisInsight** — clients / UIs.
- **Memtier benchmark** — load testing for Redis/Memcached.
- **Dynamic library**: NoSQL Workbench for DynamoDB.
- **PgBouncer / Redis-pipelining libraries** — pool / pipeline.

### Adjacent reading
- [Caching Strategies](../05-caching/cache-strategies.md)
- [Cache Pitfalls (stampede etc.)](../05-caching/cache-pitfalls.md)
- [Distributed Locks](../08-distributed-systems/distributed-locks.md)
- [Wide-Column Stores](./wide-column-stores.md)
- [Consistent Hashing](./consistent-hashing.md)

---

*Previous:* [← Relational Databases Deep Dive](./relational-databases.md)  |  *Next:* [Document Stores →](./document-stores.md)
