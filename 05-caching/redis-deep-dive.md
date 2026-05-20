# Redis Deep Dive (Data Structures, Pub/Sub, Streams, Persistence)

> **TL;DR** — Redis is much more than a key/value cache. It's a **single-threaded in-memory data-structure server** with a rich set of types — **strings, hashes, lists, sets, sorted sets, streams, bitmaps, HyperLogLog, geo, JSON** — plus pub/sub messaging, Lua scripting, transactions, persistence options (**RDB**, **AOF**), replication, and a built-in cluster mode. The reason Redis won the cache wars: most "I need a small custom data store" problems map cleanly to one of its primitives. Sorted sets for leaderboards, hashes for user objects, streams for event queues, HyperLogLog for cardinality, geo for location radius search. The trade-off you accept: single-threaded means O(N) commands block everyone, so design accordingly.

---

## 1. The Model

Redis runs one event loop per node, processing commands sequentially in a single thread. That sounds limiting; it's actually the source of much of Redis's behavior:

- Every command is **atomic** — no lock needed.
- No cross-CPU cache coherence cost — extremely fast for short commands.
- One slow command blocks every other client on that node. Behave.
- Throughput per node: ~100–200k ops/sec for simple commands.
- Latency: ~50–500 µs same-DC.

Redis 6 added I/O threading (only for socket reads/writes, not command processing). Redis 7 added cluster improvements and functions. Throughput grew but the single-threaded command model is unchanged.

---

## 2. Data Structures

### 2.1 Strings
The base type. Holds anything up to 512 MB: text, binary, integers, JSON, serialized objects.

```
SET user:42:name "Alice"        OK
GET user:42:name                "Alice"
SETEX user:42:name 60 "Alice"   # with TTL
SETNX lock:job:1 "owner-id"     # set-if-not-exists (locks)
INCR pageviews:home             # atomic counter
INCRBY balance:42 5             # add 5
GETRANGE k 0 4                  # substring
APPEND k " more"
```

**Use for**: counters, flags, serialized objects, simple cached values, distributed locks (with TTL).

The `INCR` / `DECR` family is the reason Redis is the default backend for rate limiters and counters. Atomic + O(1) + fast.

### 2.2 Hashes
A map of field → value. One Redis key, many fields.

```
HSET user:42 name "Alice" email "a@x" age 30
HGET user:42 name           # "Alice"
HMGET user:42 name email    # ["Alice","a@x"]
HGETALL user:42             # all fields (be careful with big hashes)
HINCRBY user:42 visits 1
HDEL user:42 age
HEXISTS user:42 email
```

**Use for**: user objects (one key per user instead of 30), config groups, per-entity metadata.

Memory-efficient: small hashes (under `hash-max-listpack-entries`, default 128) are stored as packed lists — far cheaper than many separate keys.

**Gotcha**: `HGETALL` is O(N) over fields. A user with 10,000 fields hurts.

### 2.3 Lists
Doubly-linked list of strings. O(1) at both ends.

```
LPUSH q job1 job2 job3       # push left
RPUSH q job4 job5            # push right
LPOP q                       # pop left
RPOP q                       # pop right
LRANGE q 0 9                 # first 10
LLEN q                       # length
BLPOP q 5                    # blocking pop, 5s timeout
```

**Use for**: simple queues, recent-N feeds, undo stacks.

For real queues with consumer groups, durability, retries → use [Streams](#28-streams) or Kafka.

### 2.4 Sets
Unordered collection of unique strings.

```
SADD friends:42 1 2 3 4
SISMEMBER friends:42 3      # 1 (yes)
SINTER friends:42 friends:99  # mutual friends
SUNION friends:42 friends:99
SDIFF friends:42 friends:99
SCARD friends:42            # cardinality
SRANDMEMBER friends:42 5    # 5 random
```

**Use for**: unique-id sets, tag membership, set algebra (mutuals, exclusions).

Sets are O(1) insert and membership; set operations are O(M+N).

### 2.5 Sorted Sets (ZSETs)
Like Sets, but each member has a **score** (double). Stored sorted by score. The Redis killer feature.

```
ZADD leaderboard 1500 alice 1450 bob 1600 carol
ZRANGE leaderboard 0 9 REV WITHSCORES   # top 10
ZRANK leaderboard alice                  # rank of alice
ZSCORE leaderboard alice                 # 1500
ZINCRBY leaderboard 10 alice             # alice +10
ZRANGEBYSCORE leaderboard 1000 2000      # range
ZRANGEBYLEX names "[a" "[c"              # lexical range
ZREMRANGEBYRANK leaderboard 0 -101       # keep top 100
```

**Use for**: leaderboards, ranked feeds, priority queues, time-windowed sliding indices (`score = unix_timestamp`), rate-limit sliding windows.

Implementation: skip list + hash table. Adds are O(log N), most reads are O(log N + M) where M is the result size.

### 2.6 Bitmaps (operations on string bits)
Treat a string as an array of bits. Stunningly memory-efficient for boolean per-ID data.

```
SETBIT visited:2026-05-19 42 1       # user 42 visited today
GETBIT visited:2026-05-19 42         # 1
BITCOUNT visited:2026-05-19          # how many unique visitors
BITOP AND visited:both day1 day2     # users in both days
```

**Use for**: daily active users, feature opt-ins per user ID, presence bitmaps.

1 GB of memory = ~8 billion bits = a slot for every internet user. Real systems use bitmaps to track presence at planetary scale.

### 2.7 HyperLogLog (cardinality estimation)
Probabilistic data structure that estimates the count of unique items with ~0.81% error using ~12 KB regardless of input size.

```
PFADD uniques:home user1 user2 user3
PFCOUNT uniques:home              # ~3
PFMERGE uniques:total uniques:home uniques:about
PFCOUNT uniques:total             # union cardinality
```

**Use for**: unique-visitor counts, distinct-IP counts, "how many users searched X this week" without storing every visitor.

See [Probabilistic Data Structures →](../08-distributed-systems/probabilistic-data-structures.md) for theory.

### 2.8 Streams
Append-only log with consumer groups. Like a tiny Kafka inside Redis.

```
XADD mystream * user 42 action login          # add entry; * = auto-ID
XLEN mystream
XRANGE mystream - +                            # all entries
XREAD COUNT 10 STREAMS mystream 0              # read from start

# consumer groups (durable consumption)
XGROUP CREATE mystream g1 $ MKSTREAM
XREADGROUP GROUP g1 worker1 COUNT 10 STREAMS mystream >
XACK mystream g1 1684500000000-0
```

**Use for**: in-app event log, job queues with at-least-once delivery, real-time pipelines where Kafka is overkill.

Pros over lists: durable consumer state, consumer groups, persistent message IDs, replay.

Cons vs Kafka: bounded to one node's memory, no native partitioning, smaller ecosystem.

### 2.9 Geo
Sorted-set under the hood; stores geohashes.

```
GEOADD places 13.361389 38.115556 "Palermo"
GEOADD places 15.087269 37.502669 "Catania"
GEODIST places Palermo Catania km        # 166.27
GEOSEARCH places FROMLONLAT 15 37 BYRADIUS 200 km ASC
```

**Use for**: nearby search, geo-fencing, ride-matching prefilters.

See [Geohashing & Quadtrees →](../19-advanced/geohashing-quadtrees.md).

### 2.10 JSON, Time-series, Vector (modules)
Redis Stack adds modules:
- **RedisJSON** — native JSON with path queries.
- **RedisTimeSeries** — downsampling, aggregations.
- **RediSearch** — full-text and vector search.

If you need these, look at Redis Stack or hosted Redis Cloud.

---

## 3. Choosing the Right Structure

A worked example. "Top-100 leaderboard with user names, scores, and a daily active flag."

```
ZADD lb 1500 user:42                   # score-indexed sorted set
HSET user:42 name "Alice" avatar "..."  # user profile as hash
SETBIT active:2026-05-19 42 1           # bitmap for daily actives
```

Three structures, three O(1)/O(log N) ops, ~tiny memory each. The same workload in a SQL table would be three queries with indexes; in Redis it's three commands totaling sub-millisecond.

Picking is mostly mechanical:
- One value per key → string.
- Many fields under one entity → hash.
- Ordered by something → sorted set.
- Unique membership → set.
- Per-ID booleans → bitmap.
- Approximate count → HyperLogLog.
- Append-only events → stream.
- Locations → geo.

---

## 4. Persistence

Redis can survive restarts via two complementary mechanisms.

### RDB (snapshot)
Periodic point-in-time snapshots of the dataset to disk.

```redis
SAVE         # synchronous; blocks Redis
BGSAVE       # fork child, snapshot async
```

Configured by `save N M` rules (e.g., `save 60 1000` = snapshot if 1000 keys changed in 60s).

- **Pros**: compact, fast to restore, small disk footprint.
- **Cons**: can lose recent writes since the last snapshot. Fork cost spikes memory briefly.

### AOF (append-only file)
Every write is appended to a log file. On restart, Redis replays the log to rebuild state.

```redis
appendonly yes
appendfsync everysec   # fsync once per second (recommended)
                       # alternatives: always (durable, slow), no (fast, risky)
```

- **Pros**: very durable (~1 second loss with `everysec`).
- **Cons**: file grows; needs periodic rewrite (`BGREWRITEAOF`); slower restart on large datasets.

### Hybrid (default in modern Redis)
RDB preamble + AOF append. Best of both: fast restore + durable since last snapshot.

### Should you enable it?
- **Pure cache**: no — let the cache rebuild from origin. Cheaper.
- **Cache that's expensive to warm**: snapshot only. Saves cold-start storms.
- **Storing things you'd cry over**: snapshot + AOF. But also — should this really be in Redis at all?

Persistence is your insurance against pod restarts and infrastructure events, not against data loss in the formal database sense. **Don't treat Redis as a database.**

---

## 5. Replication

Async primary/replica. A replica issues `PSYNC` on connect; the primary sends an RDB snapshot, then streams subsequent commands.

```
primary  ──► replica₁ (RDB stream)
         ──► replica₂
         ──► replica₃ (chained from replica₁? = cascade)
```

Key facts:
- **Async**: writes ack at primary before replicas have them. Failover can lose writes.
- **WAIT** command: `WAIT 2 1000` blocks until 2 replicas ack or 1s timeout. Approximates synchronous replication.
- **Replicas can read**: configure `replica-read-only yes` (default).
- **Diskless replication**: streams the snapshot from memory, no temp file. Useful with fast networks.
- **Replica lag**: monitor with `INFO replication` (`master_repl_offset` − `slave_repl_offset`).

---

## 6. Pub/Sub

Fire-and-forget messaging. Publishers send to channels; subscribers receive.

```
SUBSCRIBE channel1
PUBLISH channel1 "hello"
PSUBSCRIBE news.*           # pattern subscribe
```

- **At-most-once delivery.** Disconnected subscribers miss messages.
- **No persistence.** Messages aren't stored.
- **No backpressure.** Slow subscribers risk being disconnected.

**Use for**: lightweight fan-out, in-app event bus, cache invalidation broadcasts (with TTL safety net), real-time notifications where loss is OK.

**Don't use for**: durable work queues (use streams or a real broker), payment-event broadcasting, anything where you need exactly-once or replay.

Redis 7 added **sharded pub/sub** (`SSUBSCRIBE`) for cluster mode, which routes by shard and avoids the all-shard broadcast pain.

---

## 7. Transactions (MULTI/EXEC + WATCH)

```redis
MULTI
INCR pageviews:home
INCR pageviews:total
EXEC
```

Between `MULTI` and `EXEC`, commands are queued. `EXEC` runs them atomically and sequentially. No other client gets in between.

`WATCH` adds optimistic concurrency:

```redis
WATCH balance:42
balance = GET balance:42
if balance >= amount:
    MULTI
    DECRBY balance:42 amount
    EXEC          # fails if balance:42 changed since WATCH
```

`EXEC` returns nil if any watched key was modified — you retry the transaction.

This is **optimistic locking** in Redis. Cheap, correct, and works well at low contention. At high contention, retries dominate.

**Note**: in Redis Cluster, transactions only work on keys in the same hash slot. Use **hash tags** (`{user:42}:bal`, `{user:42}:lock`) to co-locate.

---

## 8. Lua Scripting

Run a Lua script atomically on the server. The script blocks all other commands while it runs.

```lua
-- atomic conditional decrement
local v = tonumber(redis.call('GET', KEYS[1]))
if v >= tonumber(ARGV[1]) then
    return redis.call('DECRBY', KEYS[1], ARGV[1])
end
return -1
```

```redis
EVAL "..." 1 balance:42 50
```

Use for: composite atomic ops that aren't expressible as a single command. Rate limiters, atomic compare-and-set patterns, atomic dequeue+ack.

Pitfalls:
- Scripts block the loop — keep them short.
- Loops over big collections — bad idea.
- Cluster mode: scripts must touch only keys in the same slot.

Modern alternatives:
- **Redis Functions** (7.0+) — persisted, versioned, modular. Better than `EVAL` for complex logic.
- **Server-side modules** (RedisJSON, RediSearch, etc.).

---

## 9. Cluster Mode Refresher

16384 hash slots, distributed across primaries. Each primary may have replicas. Cluster commands:

```
CLUSTER NODES
CLUSTER SLOTS
CLUSTER COUNTKEYSINSLOT 123
```

Routing:
- Smart client computes `CRC16(key) % 16384`, finds the primary owning the slot.
- If the client's slot map is stale, server replies `MOVED <slot> <ip:port>`. Client updates.
- During resharding, `ASK` redirects: "this key is migrating; ask this node now."

Multi-key constraints: keys must hash to the same slot. Use hash tags:

```
SET {user:42}:profile "..."
SET {user:42}:settings "..."
# both keys hash to the slot for "user:42"
```

Gotchas:
- `MULTI`, scripts, transactions: same-slot only.
- `MGET`, `MSET`: client must split across slots.
- A cluster of 3 primaries means a single hot key is still served by one node.

---

## 10. Performance Tuning

### Commands to avoid in prod
- `KEYS pattern` — O(N) full scan. Use `SCAN` instead.
- `FLUSHALL`, `FLUSHDB` — outside maintenance.
- `MIGRATE` of big keys.
- `SORT` on large sets.
- `SMEMBERS`, `HGETALL`, `LRANGE 0 -1` on large structures.
- `DEBUG SLEEP`. (Yes, this exists. Yes, people accidentally use it.)

### Pipelining
Batch many commands in one round trip. Cut latency for bulk ops.

```python
pipe = r.pipeline()
for k in keys:
    pipe.get(k)
results = pipe.execute()    # one network round trip
```

Pipelining can take 10k ops to a single batch. Massive throughput multiplier.

### Pubsub & client routing best practices
- Separate clients for blocking commands (`BLPOP`, `SUBSCRIBE`) from query clients.
- Use a connection pool sized for your peak concurrency.
- Health-check connections (`PING`) before reusing after long idle.

### Memory tips
- Use hashes for grouped fields — far cheaper than separate keys.
- Use shorter keys when keys are many. `u:42:n` instead of `user:42:name` saves bytes at scale.
- Compress big values client-side (msgpack, lz4) if CPU cost is acceptable.
- Watch `mem_fragmentation_ratio`. > 1.5 → consider `MEMORY PURGE` or restart.
- Disable `THP` (Transparent Huge Pages) on Linux — Redis warns about this on boot.

---

## 11. Distributed Locks (RedLock and Friends)

A lock backed by Redis is tempting and dangerous.

### Simple single-instance lock
```
SET lock:job:1 "owner-id" NX PX 5000   # set-if-not-exists, 5s TTL
```

On release, **check ownership before delete** to avoid releasing someone else's lock:

```lua
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
end
return 0
```

This is fine for **best-effort coordination** — coalescing background jobs, single-flight cache repopulation.

### RedLock (multiple instances)
Acquire the lock across N Redis instances in parallel; succeed if majority responded within a bounded time. Released similarly.

Martin Kleppmann's famous critique: RedLock is not safe under all failure modes; clock skew and GC pauses can cause two clients to think they hold the lock. The Redis author (antirez) wrote a response. Both have a point.

**Practical guidance**: for mutual exclusion that *must* be correct (financial transactions, leader election), use ZooKeeper, etcd, or a Postgres advisory lock — purpose-built consensus systems. For best-effort coordination, single-instance Redis lock with TTL is fine.

See [Distributed Locks →](../08-distributed-systems/distributed-locks.md).

---

## 12. Observability

What you watch:

- `INFO` — high-level snapshot. `INFO memory`, `INFO replication`, `INFO clients`, `INFO stats`.
- `SLOWLOG GET 10` — recent slow commands. Anything > 10 ms is suspect.
- `LATENCY DOCTOR` — Redis's self-diagnosis.
- `MEMORY DOCTOR` — same for memory.
- `MEMORY USAGE key` — per-key memory.
- `OBJECT FREQ key` (LFU mode) — access frequency.
- `redis-cli --bigkeys` — find the biggest keys per type.
- `redis-cli --hotkeys` (LFU) — find the most-accessed keys.
- `redis-cli --latency` — measures latency from this client.
- `LATENCY HISTORY event` — historical latency events.

Production dashboards every cache tier needs:
- Ops/sec by command.
- p50, p95, p99, p999 latency.
- Hit ratio (`keyspace_hits / (keyspace_hits + keyspace_misses)`).
- Memory used vs maxmemory.
- Evicted keys per second.
- Connection count and rejected connections.
- Replication lag.
- AOF fsync time.

---

## 13. Common Mistakes

- **Using lists as queues at scale.** Streams or Kafka are the right tool.
- **`KEYS *` in production.** O(N), blocks the world. Find your colleague who did this and have a chat.
- **Massive `HGETALL` / `SMEMBERS`.** Use `HSCAN` / `SSCAN`.
- **Storing 10 MB blobs.** Compress, split, or use S3 with a pointer in Redis.
- **No `maxmemory` + no eviction policy.** Crashes via OOM on the host.
- **`save` directive enabled on a pure cache.** Pointless fork pressure.
- **Pub/Sub for durable messaging.** Use streams.
- **RedLock for financial correctness.** Use a real consensus system.
- **Single Redis primary for a system that can't survive a 30-second cache outage.** Plan failover or be ready.
- **Ignoring cluster hash-tag rules.** `MGET`/transactions silently break.
- **Not pipelining bulk operations.** Throughput leaves on the table.
- **`SET` without TTL when you meant to cache.** Forever-keys accumulate. Watch your memory.

---

## 14. Cheat Card

```
DATA STRUCTURES
  string    counters, blobs, flags
  hash      one-key-per-entity with many fields
  list      simple FIFO/LIFO, recent-N
  set       unique membership, set algebra
  zset      sorted by score, leaderboards, sliding windows
  bitmap    per-ID booleans at scale
  hll       cardinality estimation
  stream    durable log with consumer groups
  geo       location radius queries

PERSISTENCE
  RDB           snapshots, fast restore, possible loss
  AOF           every-write log, ~1s loss, slower restart
  hybrid        default; both

REPLICATION   async, WAIT for synchronous-ish

PUBSUB        at-most-once, no persistence

TRANSACTIONS  MULTI/EXEC + WATCH (optimistic)

CLUSTER       16384 slots, hash tags for co-location

NEVER         KEYS *, big HGETALL, RedLock for money,
              treat Redis as your only DB

PIPELINE      batch reads/writes for throughput

WATCH METRIC  hit ratio, p99, memory, evictions, repl lag, slow log
```

---

## 15. Resources

### Books
- *Redis in Action* — Josiah Carlson.
- *Redis Definitive Guide* — work-in-progress, redis.io docs are the practical reference.
- *Designing Data-Intensive Applications* — Martin Kleppmann.

### Documentation
- **Redis docs**: <https://redis.io/docs/latest/>
- **Commands reference**: <https://redis.io/commands/>
- **Redis Cluster spec**: <https://redis.io/docs/latest/operate/oss_and_stack/reference/cluster-spec/>
- **Persistence docs**: <https://redis.io/docs/latest/operate/oss_and_stack/management/persistence/>

### Articles
- "How to do distributed locking" — Martin Kleppmann.
- "Is Redlock safe?" — antirez (response to above).
- "Redis Sorted Sets are awesome" — many engineering blogs (Pinterest, GitHub).
- "Sharded Pub/Sub in Redis 7" — redis.io.

### Videos
- Hussein Nasser — Redis internals series.
- ByteByteGo — "Redis Top 5 Use Cases".

### Tools
- **redis-cli** with `--bigkeys`, `--hotkeys`, `--latency`, `--scan`.
- **RedisInsight** — official GUI.
- **redis-benchmark** — for sizing.
- **redis-exporter** for Prometheus.

### Adjacent reading
- [Distributed Caching →](./distributed-caching.md)
- [Cache Pitfalls →](./cache-pitfalls.md)
- [Key-Value Stores →](../04-databases/key-value-stores.md)
- [Distributed Locks →](../08-distributed-systems/distributed-locks.md)
- [Probabilistic Data Structures →](../08-distributed-systems/probabilistic-data-structures.md)
- [Geohashing & Quadtrees →](../19-advanced/geohashing-quadtrees.md)

---

*Previous:* [← Distributed Caching](./distributed-caching.md)  |  *Next:* [Cache Pitfalls →](./cache-pitfalls.md)
