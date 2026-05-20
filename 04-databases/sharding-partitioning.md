# Sharding & Partitioning

> **TL;DR** — **Partitioning** = splitting one big table into chunks so it scales. **Sharding** = distributing those chunks across multiple physical servers. The two terms are often used interchangeably; "partitioning" usually means logical/per-node, "sharding" means horizontal across nodes. The three common strategies are **range**, **hash**, and **directory (lookup)**. Picking the right **shard key** is the single most consequential decision — you'll live with it for years. Watch for **hot partitions**, **cross-shard queries**, and **rebalancing costs**.

---

## 1. Why Shard

You shard when one machine can't hold the data, can't serve the QPS, or can't afford to fail alone:

- **Storage** — data exceeds a single node's disk.
- **Writes** — single primary write throughput is saturated.
- **Reads** — even with replicas, a hot path overloads each replica.
- **Operational blast radius** — keep a per-shard failure to a fraction of users.
- **Tenant isolation** — one tenant per shard (or shard group).
- **Geographic locality** — shard by region.

```
Before:     [    ONE BIG DB    ]   ← one writer, one disk, one failure domain.
After:      [shard 1][shard 2][shard 3][shard 4]    ← N writers, N disks.
```

Sharding multiplies capacity. It also multiplies operational complexity. Don't shard before you must.

---

## 2. Partitioning vs Sharding (Terminology)

- **Partitioning** — logical division of a table into smaller pieces. Postgres native partitioning, MySQL partitions, Oracle partitioning. Pieces may still live on the same node.
- **Sharding** — horizontal split across **physical machines**.
- **Vertical partitioning** — splitting a wide table into multiple narrower tables (rarely useful at scale; see denormalization).

In casual conversation people say "shard" or "partition" interchangeably. In this doc:
- **Partition** = a piece of a table (logical).
- **Shard** = a physical home for one or more partitions.

---

## 3. The Three Sharding Strategies

### 3.1 Range Sharding
Each shard owns a contiguous range of the partition key.

```
shard A: user_id [0  – 10M)
shard B: user_id [10M – 20M)
shard C: user_id [20M – 30M)
...
```

**Pros**
- Efficient range queries (`WHERE id BETWEEN ...`) — single shard.
- Easy to understand.
- Works well when key has natural ordering (time, ID).

**Cons**
- **Hot partitions** if writes cluster on one end (e.g., monotonic IDs → newest shard takes all writes).
- Rebalancing requires splitting ranges, which can be tricky.

Used by: HBase, Bigtable, CockroachDB (range-based KV), TiKV.

### 3.2 Hash Sharding
Hash the partition key; pick a shard by hashed value modulo N, or via **consistent hashing**.

```
shard = hash(user_id) % N
```

**Pros**
- Even distribution by default.
- No hot spot from monotonic IDs.
- Simple math.

**Cons**
- Range queries cross all shards.
- Modulo schemes shuffle the world when N changes — use **consistent hashing**.

Used by: Cassandra, DynamoDB (hash partition key), Vitess hash vindexes, Redis Cluster, sharded Mongo.

### 3.3 Directory (Lookup) Sharding
A separate lookup table maps `key → shard`.

```
lookup:
  user_id=42  → shard 7
  user_id=88  → shard 3
```

**Pros**
- Full flexibility: arbitrary placement, easy migrations, tenant-level pinning.
- Can move data without changing the algorithm.

**Cons**
- The lookup is a new dependency and a potential SPOF.
- Extra hop on every query.
- Caching the lookup is essential at scale.

Used by: Slack (with Vitess), GitHub (Spokes), many B2B SaaS with per-tenant sharding.

### Mix-and-match
Real systems combine. Example: **hash by tenant_id**, then **range by created_at** within the tenant — keeps each tenant's data co-located, with time order inside.

---

## 4. Picking a Shard Key (This Is The Decision)

The shard key determines distribution, locality, hot spots, joinability, and migration pain. **Choose like your career depends on it — because it might.**

### Good shard-key qualities
- **High cardinality** — many distinct values.
- **Even distribution** — no single value dominates.
- **Co-locates the queries you run together** — single-shard reads where possible.
- **Stable** — values don't change for a row (you'd have to move data).

### Bad shard-key smells
- Low cardinality (`country`, `status`).
- Heavy skew (one tenant > 80% of data → "noisy neighbor" shard).
- Highly correlated with time + monotonic IDs (write hotspot on newest).
- Cross-cuts every business query (every query touches every shard).

### Common keys
- **`user_id`** / **`tenant_id`** — most multi-tenant SaaS.
- **`(tenant_id, entity_id)`** — composite.
- **`account_id`** — banking-style.
- **`hash(uuid)`** — when no natural key.

### Anti-pattern: sequential IDs
`AUTO_INCREMENT` PK + range sharding = every new row hits the newest shard. The classic "we sharded and it didn't help" outcome. Use a non-monotonic key, **`hash`-shard**, or use the CockroachDB-style "hashed sharded indexes" trick.

---

## 5. Cross-Shard Queries

What you can't do efficiently in a sharded system:
- `JOIN` across shards (hugely expensive).
- `ORDER BY` + `LIMIT` across all shards (must fetch top-N from each, merge).
- Global `COUNT(*)`.
- Transactions across shards (need 2PC or sagas).

What you can:
- Single-shard queries (90%+ of OLTP if you picked the key well).
- Scatter-gather to N shards in parallel and merge (`fan-out` queries).
- Async aggregations into a warehouse / cache.

Design your data model so the hot 90% of queries live on **one shard**. Accept that the long tail will need scatter-gather or denormalized derived stores.

---

## 6. Rebalancing — The Operational Cost

When you add a shard you need to move some data. Three strategies:

### Naive modulo (`hash(key) % N`)
Adding a shard changes N, so **almost every row** needs to move. Disaster at scale.

### Consistent hashing
Hashes both keys and shard IDs onto a ring. Adding a shard moves ~`1/N` of data, not all of it.

```
ring positions
    [.....shard A.....shard B.....shard C.....]
   add shard D between B and C → only the B–D arc moves.
```

See [Consistent Hashing](./consistent-hashing.md).

### Virtual nodes (vnodes / tokens / slots)
Pre-allocate **many** logical partitions (e.g., 16384 hash slots in Redis Cluster, 256 vnodes per Cassandra node). Each physical shard owns a subset; moving slots between nodes is the unit of rebalancing. Smooth and parallel.

Almost every modern sharded store uses some form of this.

---

## 7. Hot Partitions — The Recurring Nightmare

Even with good distribution, traffic can concentrate on one partition:
- Celebrity user with millions of followers.
- One tenant doing a bulk import.
- A timestamp prefix that everyone reads "now."
- A single key (a counter, a feature flag, a "Trending Now" entry).

Symptoms: one shard is at 100% CPU/IO; the others coast.

Mitigations:
- **Split the hot key**: shard `counter:popular_post` into `counter:popular_post:bucket0..127`; sum on read.
- **Add randomness**: prefix keys with a hash bucket to spread.
- **Read replicas of the hot shard**.
- **Cache the hot data** in Redis / CDN.
- **Move the hot tenant to its own shard** (directory routing).
- **Rate-limit / queue** the hot writer.

See [Hot Partition Problem](../10-scalability/hot-partitions.md).

---

## 8. Tooling

### Postgres
- **Native partitioning** (range / list / hash) since PG10 — same node.
- **Citus** extension — sharded Postgres with distributed planner.
- **pg_partman** — automated partition lifecycle.
- **pgcat / PgPool** — connection pooling + routing helpers.

### MySQL
- **Native partitioning** (per-table).
- **Vitess** — battle-tested at YouTube, Slack, GitHub, PlanetScale.
- **MySQL Cluster (NDB)** — niche.

### MongoDB
- Built-in **sharded cluster** (config servers + mongos + shards).
- Pick **hashed** or **ranged** shard key carefully.

### Cassandra / Scylla
- Naturally sharded by partition key + token ring.

### DynamoDB
- Hidden partitions; you provide the partition key.
- Hot partition throttling is a real concern.

### NewSQL (CockroachDB, Spanner, TiDB)
- Sharding is automatic and dynamic.
- You declare locality and indexes; the engine handles ranges.

### Redis Cluster
- 16384 hash slots distributed across nodes. Multi-key ops within one slot only (use **hash tags** `{tenant}` to force co-location).

### Kafka
- Topic partitions are shards. Producer chooses partition (default: hash of the key).

---

## 9. Postgres Native Partitioning — A Quick Tour

```sql
CREATE TABLE events (
  id bigserial,
  tenant_id text NOT NULL,
  occurred_at timestamptz NOT NULL,
  payload jsonb
) PARTITION BY RANGE (occurred_at);

CREATE TABLE events_2026_05 PARTITION OF events
  FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

CREATE INDEX ON events_2026_05 (tenant_id);
```

Benefits:
- **Drop old partitions instantly** for retention (just `DROP TABLE` the month).
- Smaller indexes per partition.
- Better autovacuum behavior on hot vs cold partitions.
- Partition pruning at plan time avoids scanning irrelevant partitions.

Still **one Postgres node**. For horizontal scale, add Citus or Vitess on top.

---

## 10. Sharding in Practice — A Migration Story

A typical journey:

1. **Single Postgres**. Works fine for years.
2. **Add read replicas**. Reads scale; writes still saturating.
3. **Cache hot reads** in Redis. Bought another year.
4. **Federate** by service: each microservice gets its own DB. Easy isolation, no real sharding.
5. **Logical partitioning** of one giant table by tenant — same DB, smaller indexes.
6. **Vertical split** (move tables to dedicated DBs).
7. **Horizontal shard** the biggest table:
   - Pick a shard key.
   - Build a **routing layer**.
   - **Dual-write** during migration (write to both old and new).
   - Backfill historical data.
   - **Cutover** reads, then writes.
   - Decommission the old.

This is a multi-month operation in any non-trivial codebase. Tooling helps but doesn't make it easy.

---

## 11. Cross-Shard Transactions

If you must update across shards atomically:
- **2-phase commit (2PC)** — possible (Vitess, XA) but brittle and slow.
- **Saga pattern** — sequence of local transactions with compensations. The standard for service-to-service.
- **NewSQL** (Spanner, Cockroach) — built-in cross-shard ACID, paid for with consensus latency.
- **Idempotent operations** — make each step retry-safe; rebuild via reconciliation when needed.

See [Saga Pattern](../07-messaging/saga-pattern.md) and [Two-Phase Commit](../08-distributed-systems/2pc-3pc.md).

---

## 12. Routing Layer

Every sharded system needs to know "where does this query go?"

- **Smart client** — driver knows the shard map (Cassandra, Redis Cluster, DynamoDB).
- **Proxy/router** — a service in front (Vitess vtgate, Citus coordinator, ProxySQL, sharded MySQL).
- **Application-side routing** — the app picks the connection itself.

Trade-offs:
- Smart clients = no extra hop; harder to upgrade.
- Proxies = central control; one more thing to operate; potential SPOF (run multiple).

Proxies dominate in MySQL / Postgres land; smart clients in NoSQL.

---

## 13. Common Mistakes

- **Sharding too early.** Most teams shard at 10× the scale they ever reach. Vertically scale first.
- **Sequential ID + range sharding** → permanent hot shard.
- **Low-cardinality shard key** → tiny number of shards in use.
- **Hot tenant on shared shard** → noisy neighbor.
- **No plan for rebalancing** → painful future operations.
- **Cross-shard joins on the hot path** → death by scatter-gather.
- **No idempotency / retries** in the routing layer → duplicate writes during failover.
- **Backups per shard, restore strategy untested** → DR is a fiction.
- **Schema changes ignored** at scale — every shard must migrate identically.
- **Dual-write bugs** during migration — one side gets ahead, data diverges.

---

## 14. Cheat Card

```
PARTITION = logical chunk of a table.
SHARD     = physical home for one or more partitions.

STRATEGIES
  Range      ★ range queries efficient    ✗ hot partitions w/ monotonic IDs
  Hash       ★ even distribution           ✗ range queries fan out
  Directory  ★ flexible, per-tenant        ✗ extra lookup + dependency

SHARD-KEY TESTS
  high cardinality?    even distribution?
  co-locates queries?  stable per row?

REBALANCING
  modulo = BAD (everything moves).
  consistent hashing or VIRTUAL NODES = good (only ~1/N moves).

HOT PARTITIONS
  split keys · randomize · cache · move tenant · rate-limit · add replicas.

CROSS-SHARD
  joins / global ORDER BY / global COUNT = expensive.
  Sagas for distributed business actions.
  NewSQL gives you cross-shard ACID at consensus cost.

TOOLS
  Postgres: native partitioning · Citus · pg_partman · pgcat
  MySQL:    Vitess · ProxySQL · PlanetScale
  Mongo:    sharded clusters
  Redis:    Cluster + hash tags
  NewSQL:   Cockroach / Spanner / TiDB (automatic)

WHEN TO SHARD
  vertical maxed out · writes saturate · single-AZ failure unacceptable.
  Otherwise: bigger box + replicas + cache + federation.

RULES
  Don't pick the shard key on day one.
  When you must: high cardinality, no monotonic, query-aligned.
  Drill rebalancing before you need it in anger.
```

---

## 15. Resources

### Books
- *Designing Data-Intensive Applications* — Kleppmann (Ch. 6 — partitioning is essential reading).
- *Database Internals* — Alex Petrov.
- *Vitess: A Database Clustering System* — Sugu Sougoumarane (papers / talks).
- *Building Microservices* — Sam Newman (federation patterns).

### Articles
- "Pattern: Sharding" — microservices.io: <https://microservices.io/patterns/data/sharding.html>
- "How GitHub uses Vitess" — engineering blog.
- "How Slack scaled with Vitess" — engineering blog.
- "Sharding & IDs at Instagram" — Instagram engineering: <https://instagram-engineering.com/sharding-ids-at-instagram-1cf5a71e5a5c>
- "DynamoDB hot partitions" — AWS blog and re:Invent talks.
- "Cassandra data modeling" — DataStax / Scylla.

### Documentation
- **Postgres partitioning** — <https://www.postgresql.org/docs/current/ddl-partitioning.html>
- **Citus** — <https://docs.citusdata.com/>
- **Vitess** — <https://vitess.io/docs/>
- **MongoDB sharding** — <https://www.mongodb.com/docs/manual/sharding/>
- **DynamoDB partitioning** — AWS docs.
- **Cassandra data modeling** — <https://cassandra.apache.org/doc/latest/cassandra/data_modeling/>

### Videos
- ByteByteGo: "Database sharding" — <https://www.youtube.com/@ByteByteGo>
- Hussein Nasser sharding deep dives — <https://www.youtube.com/@hnasr>
- "Vitess at YouTube" — Sugu Sougoumarane conference talks.
- Andy Pavlo CMU lectures on distributed databases.

### Tools
- **Vitess**, **Citus** — production sharding tools.
- **ProxySQL** — routing for MySQL.
- **PgBouncer / pgcat** — pooling + light routing for Postgres.
- **pg_partman** — Postgres partition automation.
- **Debezium / Maxwell** — CDC during migrations.

### Adjacent reading
- [Consistent Hashing](./consistent-hashing.md)
- [Hot Partition Problem](../10-scalability/hot-partitions.md)
- [Replication](./replication.md)
- [Database Federation](./federation.md)
- [Two-Phase Commit (2PC) and Three-Phase Commit (3PC)](../08-distributed-systems/2pc-3pc.md)
- [Saga Pattern](../07-messaging/saga-pattern.md)
- [Multi-Region](../10-scalability/multi-region.md)

---

*Previous:* [← Replication](./replication.md)  |  *Next:* [Consistent Hashing →](./consistent-hashing.md)
