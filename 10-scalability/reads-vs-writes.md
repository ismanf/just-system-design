# Scaling Reads vs Scaling Writes

> **TL;DR** — Reads and writes scale through fundamentally different mechanisms. **Reads** scale almost arbitrarily by adding replicas, caches, CDNs, and read-only copies — the data is immutable from the reader's perspective, so duplication is cheap. **Writes** scale only by *partitioning* — splitting the write surface across independent failure/throughput domains (sharding). Most systems hit a read ceiling first and a write ceiling later; when you do hit the write ceiling, the cost of the fix (sharding, write-path rearchitecture, eventual-consistency models) is orders of magnitude greater than scaling reads. Knowing which side you're constrained on — and *which side you'll be constrained on next* — is the central question of capacity planning.

---

## 1. The Asymmetry

A simplified law of scaling distributed data:

```
Scaling reads:    add copies.
Scaling writes:   split the data.
```

Reads scale **horizontally and cheaply** because the data is the same everywhere; a replica is a clone. Writes scale only when you can divide work so that each shard owns an independent piece — and then *every cross-shard operation gets harder.*

This asymmetry has two consequences:
1. **A single primary** can serve far more reads (via replicas) than writes (against itself).
2. **Sharding is irreversible architecture.** It changes everything downstream — transactions, joins, indexes, backups, schema migrations. Defer it as long as honesty allows. Then do it.

---

## 2. Why Reads Are Easy

```
       Primary
          │
   writes │
          ▼
   ┌─────────────┐
   │  Database   │
   └─────┬───────┘
         │ replicates (async/sync) to N replicas
         ▼
   ┌─────┬─────┬─────┬─────┬─────┐
   │ R1  │ R2  │ R3  │ R4  │ R5  │
   └─────┴─────┴─────┴─────┴─────┘
   ▲     ▲     ▲     ▲     ▲
   │     │     │     │     │
   ├── reads served from any replica ──┤
```

For a read-heavy workload, throughput grows roughly linearly with replica count. Add a replica → add ~X reads/sec capacity. The bottleneck shifts from CPU/IO to:
- **Network egress** from the primary (replication stream).
- **Replication lag** when consumers need fresh reads.
- **Load balancer capacity** distributing reads.

Even those caps are pushed back by:
- **Cascading replicas** (read replica replicates from another read replica).
- **Caches** — Redis, Memcached, application caches.
- **CDNs** for content at the edge.
- **Materialized views** denormalized for hot read paths.

The dominant cloud OLTP architecture for the last 15 years has been "one writer, many read replicas." It scales reads to millions/sec without sharding.

---

## 3. Why Writes Are Hard

Writes can't be cloned. Two clients writing to the same row need to agree on order, atomicity, and durability. Even with eventual consistency, *somebody* has to be the source of truth for that row.

The single primary write path bottlenecks on:
- **WAL fsync throughput** — usually 10k–100k commits/sec on good hardware. See [WAL →](../09-storage/wal.md).
- **Buffer pool churn** — random writes outrunning the page cache.
- **Lock contention** — concurrent writes to the same rows / indexes.
- **Replication fan-out** — primary spends bandwidth feeding replicas.
- **Single CPU socket** — many DB engines bottleneck on a single writer thread or per-partition queue.

You can buy bigger hardware for a while — vertical scaling. Eventually:
- One machine isn't big enough.
- The single point of failure becomes unacceptable.
- IO contention from the workload itself dominates.

Then you shard. There is no other answer.

---

## 4. Read-Scaling Techniques (Cheap → Less Cheap)

A practical ladder, roughly in the order most teams climb it:

### 4.1 Application-level caching
- In-process LRU caches for hot keys.
- Sub-millisecond, but per-instance — no cross-instance coherence.
- Use for: lookups that are read 100×+ for every write, where staleness is OK.

### 4.2 Shared cache layer
- Redis or Memcached in front of the DB.
- Cache-aside is the default pattern. See [Cache Strategies →](../05-caching/cache-strategies.md).
- Adds an external dependency but flattens DB read load dramatically.
- Watch for: cache stampedes, hot keys, invalidation correctness.

### 4.3 Read replicas
- One primary, N replicas. Reads routed to replicas; writes to primary.
- Common in Postgres, MySQL, MongoDB, etc.
- Trade-off: **replication lag**. Writes may be invisible to replica reads for ms–seconds (sync replication tighter but slower).
- Implementation: app-aware routing, proxy (PgBouncer, ProxySQL), or DB-native (Aurora Reader endpoint).

### 4.4 Materialized views / read models
- Pre-compute the join, denormalized for read.
- CQRS pattern: writes update a normalized model; an async pipeline projects into a read-optimized view. See [CQRS →](../07-messaging/cqrs.md).
- Trade-off: eventual consistency between write and read models.

### 4.5 CDN / edge caching
- For HTTP-cacheable resources, CDN turns reads into something that never touches your origin.
- Static assets, public API responses, anything with high cache hit rate.
- See [CDN →](../05-caching/cdn.md).

### 4.6 Search / analytical secondary indexes
- Elasticsearch / OpenSearch for full-text reads.
- ClickHouse / Druid / Pinot for analytical reads on hot recent data.
- DynamoDB GSIs or Postgres logical replicas for alternative read paths.
- Pattern: source of truth in OLTP DB; CDC pipeline feeds the read store.

### 4.7 Geo-distributed reads
- Read replicas in multiple regions. Reads served locally; writes still funnel to primary region.
- Common for global SaaS. See [Multi-Region →](./multi-region.md).

These tools compose. A high-traffic web app might serve 99% of reads from CDN/cache/app cache, 0.9% from replicas, 0.1% from the primary.

---

## 5. Write-Scaling Techniques

Far fewer options. They are bigger commitments.

### 5.1 Batch writes
- Group multiple writes into one commit. Single transaction, single WAL fsync.
- Drastically reduces commit overhead.
- Works when the application can buffer or accumulate.
- Used everywhere from analytics ingest (Kafka batching) to bulk operations.

### 5.2 Write-behind / async writes
- Application writes to a fast queue or cache; a background worker drains to the DB.
- Trade durability for throughput.
- Risky for money / orders / inventory — fine for activity feeds, logs, metrics.

### 5.3 Vertical scaling
- Bigger machine, more cores, NVMe, more RAM. Postgres on a 96-core box with 1 TB RAM is incredibly fast.
- Easy and cheap (until it isn't). Bridges most growth curves.
- Hits ceilings around 30–100k writes/sec sustained for OLTP RDBMSs.

### 5.4 Functional partitioning
- Split by *table*: users on DB A, orders on DB B, sessions on DB C.
- Each domain owns its writes. Works when domains are loosely coupled.
- Hits limits when one domain's writes still outgrow one machine — then you re-shard that one.

### 5.5 Multi-leader (active-active) replication
- Multiple primaries accept writes. Conflicts must be detected and resolved.
- Mostly used geographically (one primary per region).
- CRDTs (see [CRDTs →](../08-distributed-systems/crdts.md)) make conflict resolution principled.
- Postgres-BDR, CockroachDB, Google Spanner, Cassandra LWT.

### 5.6 Sharding
- Split the write surface across N independent shards.
- Each shard takes 1/N of the writes; each has its own primary, WAL, replicas.
- The big-hammer answer; see [Database Sharding Strategies →](./sharding-strategies.md) and [Sharding & Partitioning →](../04-databases/sharding-partitioning.md).
- Trade-offs: cross-shard transactions, cross-shard queries, rebalancing, hot partitions.

### 5.7 Specialized write-optimized engines
- LSM-tree engines (Cassandra, ScyllaDB, RocksDB, BigTable) are designed for write throughput. They trade read amp for write amp.
- Use when the workload is fundamentally write-heavy (time-series, telemetry, audit logs).
- See [Storage Engines →](../09-storage/storage-engines.md).

### 5.8 Move writes off the OLTP DB
- Often the cheapest fix: stop writing logs / events / metrics into the transactional DB.
- Route to Kafka / Kinesis / object storage instead, and feed the OLTP DB only what truly needs ACID.
- The classic "wait, why was 80% of our write load just `INSERT INTO events`?" moment.

---

## 6. Worked Example — A SaaS Growth Curve

A typical B2B SaaS scales roughly like this. Postgres throughout.

```
0–10k users        single Postgres, 4 cores, 16 GB RAM
                   reads: 200/sec, writes: 30/sec
                   no caching, no replicas

10k–100k users     add 1 read replica + Redis cache
                   reads: 5k/sec (95% cache hit), writes: 300/sec
                   primary on 16-core, 64 GB RAM
                   80% reads served outside primary

100k–500k users    3 read replicas, multi-region CDN, ElasticSearch
                   for full-text, OLAP in Snowflake via CDC
                   reads: 50k/sec total, writes: 3k/sec on primary
                   primary on 64-core, 256 GB RAM, NVMe
                   ~70% writes still on primary; secondary stores
                   (events to Kafka, files to S3) take the rest

500k–5M users      functional partitioning: identity DB, billing DB,
                   product DB, audit log goes to Kafka/ClickHouse
                   each domain has its own primary + replicas
                   writes: 15k/sec aggregate, no single primary >5k/sec

5M+ users          shard the heavy domains (e.g., per-tenant in product
                   DB via Vitess / Citus); identity stays single-primary
                   reads: 1M/sec aggregate, writes: 200k/sec aggregate
```

Observations:
- Read scaling never required architectural change — just more cheap copies.
- Write scaling required *moving writes off the DB* before requiring sharding.
- Sharding came only when one functional domain still hit ceilings.

This is the typical shape. Different workloads (consumer social, IoT, fintech) shift the inflection points, but the order is similar.

---

## 7. The Read/Write Ratio Trap

Most workloads are **wildly read-heavy** — common ratios:

| Workload | R:W ratio |
|---|---|
| Public web pages | 1000:1 or more |
| Consumer social feeds | 100:1 |
| B2B SaaS dashboards | 50:1 |
| Email / messaging | 10:1 |
| OLTP transactions | 5:1 to 10:1 |
| Logs / telemetry | 1:10 (write-heavy) |
| IoT / sensor data | 1:100 (very write-heavy) |
| Time-series databases | varies |

The trap: read-heavy systems get used to "throw a cache at it" and assume that scales forever. It doesn't — when traffic 10×s, writes 10× too, and the absolute write rate eventually breaches what a single primary can sustain even though the *ratio* never changed.

The other trap: write-heavy systems often pretend they're read-heavy because the read path is so optimized (cached, indexed) that users only see writes as slow. In reality the write path is on a knife edge and any uptick will hurt.

Always measure the **absolute** rates. Ratios deceive.

---

## 8. CAP Implications

Scaling reads and writes interacts brutally with consistency. See [CAP Theorem →](../08-distributed-systems/cap-theorem.md) and [PACELC →](../08-distributed-systems/pacelc.md).

- **Read replicas are eventually consistent** by default. A write on the primary isn't visible on replicas for some lag window.
- **Read-your-writes** consistency requires routing the reader's reads to the primary (or a synchronously-replicated replica) for a short window after their write.
- **Geo-distributed writes** force a choice: either funnel writes to one region (high latency for far users) or accept conflicts (with multi-leader or CRDTs).
- **Sharded writes** preserve per-shard ACID but break across-shard atomicity unless you adopt 2PC / sagas (see [Saga Pattern →](../07-messaging/saga-pattern.md)).

The cost of scaling writes is paid in *consistency complexity* as well as engineering effort.

---

## 9. Read Scaling Pitfalls

- **Replication lag amnesia.** App reads from a replica immediately after writing → reads stale data → bug report. Route critical reads to primary, or use session consistency.
- **Cache staleness.** Cached lookups return data that was just updated. Solve with invalidation, short TTLs, or write-through.
- **Hot keys** in a shared cache. One key serves 10× more reads than the next; you're throttled at that key's bandwidth, not the cluster's. See [Cache Pitfalls →](../05-caching/cache-pitfalls.md).
- **Thundering herd on cache miss.** Cache expires → 10k requests stampede to the DB. Use single-flight / request coalescing.
- **Read replicas not load-balanced.** App randomly picks one → some replicas idle, one is overloaded.
- **Forgetting CDN cache-control headers.** Origin serves traffic that should have been edge-cached.
- **CDN cache pollution from auth/cookie variance.** Unique cache key per user → 0% hit rate.

---

## 10. Write Scaling Pitfalls

- **Sharding too early.** Operational pain for capacity you don't yet need. Vertical scale + functional partitioning solves a lot.
- **Sharding too late.** Now you have a 4 TB primary with no obvious shard key. The migration costs you a quarter.
- **Wrong shard key.** Hot partitions, cross-shard joins, painful rebalancing. The shard key is the single most consequential schema decision. See [Sharding Strategies →](./sharding-strategies.md).
- **Asynchronous writes for things that mustn't be lost.** Inventory, payments, orders — sync commit only.
- **Batching with unbounded delay.** Writes accumulate, latency degrades, then a burst hits all at once.
- **Storing log / event data in OLTP.** The single biggest source of "our writes are exploding" is high-volume events landing in the wrong place.
- **Index proliferation.** Every secondary index multiplies write IO. The B-tree write costs 5× when you have 4 indexes.
- **Ignoring tail writes.** p99 commit latency at 200ms doesn't mean p50 is fine — it means slow commits are queueing.
- **Multi-leader without conflict resolution strategy.** You will have conflicts. You will resolve them wrongly without a plan.

---

## 11. Diagnostic Questions

When asked "scale our system," start here:

```
1. What's the absolute read/sec and write/sec?
2. What's the ratio? Is it changing over time?
3. Where do reads hit? (Cache layer hit rate? Replica? Primary?)
4. Where do writes hit? (Single primary? Functional partitions? Sharded?)
5. What's the current bottleneck? (CPU, IO, fsync, lock contention, network)
6. What's the p99 and p999 latency for reads? For writes?
7. What's the replica lag? Under what load does it grow?
8. What % of writes are "must be durable" (orders, money) vs "best effort" (events, logs)?
9. Can any current writes be moved off the OLTP path?
10. If we shard tomorrow, what's the natural shard key? What breaks?
```

The answers determine which lever you pull first. Almost always the cheapest leverage is *moving the wrong writes off the OLTP path*, then *more aggressive read caching*, then *replicas/CDN*, then — and only then — sharding.

---

## 12. Decision Tree

```
Are reads the bottleneck?
   YES → cache → replicate → CDN → materialized views → search index
                                                      → secondary stores
   Almost no architectural commitment required.

Are writes the bottleneck?
   YES, mild   → batch · vertical scale · move logs/events off OLTP
   YES, severe → functional partition · write-optimized engine
   YES, structural → SHARD. Pick the shard key carefully.

Are both?
   You have a workload problem disguised as a database problem.
   Profile harder. Find the 10% of queries doing 90% of the work.

Is consistency the real constraint (not throughput)?
   → CAP/PACELC trade-offs. Pick consistency level per use case.

Is latency the real constraint (p99/p999)?
   → tail latency is a separate beast (see Performance section).
```

---

## 13. Cheat Card

```
PURPOSE     Reads scale by copying. Writes scale by partitioning.
            The cost gap between the two defines architecture choices.

READS                            WRITES
─────                            ──────
copy is cheap                    split is expensive
caches · replicas · CDN          shards · functional partitions
denormalize for speed            keep hot table small
millions/sec on commodity HW     10k–100k/sec per primary
horizontally elastic             discretely elastic (shard count)

READ LADDER (cheap → expensive)
  in-process cache → shared cache → read replicas →
  materialized views → CDN → secondary stores

WRITE LADDER (cheap → expensive)
  batching → vertical scale → move writes off OLTP →
  functional partition → write-optimized engine → SHARD

RATIOS      Read-heavy ≠ small absolute writes. Measure both.
            R/W ratio doesn't tell you when you'll hit the
            write ceiling — absolute write rate does.

PITFALLS    Replica lag amnesia · cache stampede · sharding too
            early or too late · wrong shard key · async writes
            for durable data · log volume into OLTP · index
            proliferation

RULE        Default to vertical + read scaling for as long as
            honest. Then carve writes off the OLTP path. Shard
            only when nothing else works. The shard key is the
            decision you live with for years.
```

---

## 14. Resources

### Books
- *Designing Data-Intensive Applications* — Martin Kleppmann. Chapters 5 (Replication) and 6 (Partitioning) are foundational.
- *Database Internals* — Alex Petrov. Engine internals that explain why writes scale differently.
- *Building Microservices* — Sam Newman. Functional partitioning as architecture.
- *Patterns of Enterprise Application Architecture* — Martin Fowler. CQRS, read models.

### Articles
- "How Stripe Built a Distributed Database" — Stripe engineering on functional partitioning.
- "How Discord Stores Billions of Messages" — write-heavy storage architecture.
- "Vitess at Slack" — sharding MySQL at scale.
- "Sharding Pinterest" — early sharding write-ups (still excellent).
- "Citus and Multi-Tenant Sharding" — Citus team blog.
- "Aurora: Scaling Reads with Cluster Endpoints" — AWS docs.

### Videos
- ByteByteGo — "Scaling Databases."
- Hussein Nasser — multi-part series on replication and sharding.
- CMU 15-721 — Andy Pavlo's distributed databases lectures.
- Jepsen analyses — see how multi-leader systems handle write conflicts.

### Tools
- **pg_stat_statements** — figure out what's actually writing on Postgres.
- **pgbench / sysbench / HammerDB** — write throughput benchmarks.
- **k6 / Locust / wrk** — generate read load.
- **Vitess, Citus** — production sharding for MySQL/Postgres.

### Adjacent reading
- [Database Sharding Strategies →](./sharding-strategies.md)
- [Hot Partition Problem →](./hot-partitions.md)
- [Capacity Planning →](./capacity-planning.md)
- [Sharding & Partitioning](../04-databases/sharding-partitioning.md)
- [Replication](../04-databases/replication.md)
- [Cache Strategies](../05-caching/cache-strategies.md)
- [CQRS](../07-messaging/cqrs.md)
- [Multi-Region](./multi-region.md)

---

*Up:* [README ↑](../README.md)  |  *Next:* [Database Sharding Strategies (Range, Hash, Geo, Directory) →](./sharding-strategies.md)
