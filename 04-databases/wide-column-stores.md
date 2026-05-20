# Wide-Column Stores (Cassandra, HBase, ScyllaDB)

> **TL;DR** — A **wide-column store** is a distributed, partition-tolerant database that looks like a sparse, sorted map: `(partition_key, clustering_key, column_name) → value`. Rows within a partition are stored together, sorted, and read efficiently in ranges. The model — pioneered by Google **Bigtable** and Amazon **Dynamo** — gives you **massive write throughput, linear horizontal scale, multi-DC replication**, at the cost of a **constrained query model** (you must design tables around the queries you'll run). Players: **Cassandra**, **ScyllaDB**, **HBase**, **Google Bigtable**, **DynamoDB** (kind of — it overlaps with KV).

---

## 1. The Data Model in One Picture

```
┌────────────────────────────────────────────────────────────────┐
│  Table: messages                                                │
│                                                                 │
│  partition key:  room_id                                        │
│  clustering key: (sent_at, msg_id)        ← sort within partn   │
│                                                                 │
│  ┌──────────────── partition: "room_42" ─────────────────────┐  │
│  │ (sent_at, msg_id) -> { user_id, body, attachments }       │  │
│  │ 2026-05-19T10:00Z m_1 → ada,  "hi"          ,  [...]      │  │
│  │ 2026-05-19T10:01Z m_2 → bob,  "hello"       ,  null       │  │
│  │ 2026-05-19T10:02Z m_3 → ada,  "going home"  ,  null       │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                 │
│  ┌──────────────── partition: "room_77" ─────────────────────┐  │
│  │ ...                                                       │  │
│  └────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────┘
```

Two key ideas:
- **Partition key** = which physical shard / node holds this row.
- **Clustering key** = the sort order *within* a partition.

A single partition can hold *millions* of rows. Reads within a partition are cheap; cross-partition reads are expensive.

---

## 2. The Players

| Engine | Lineage | Notes |
| --- | --- | --- |
| **Apache Cassandra** | Dynamo + Bigtable hybrid | Most popular OSS wide-column. Multi-DC built in. CQL query language. |
| **ScyllaDB** | Cassandra-compatible | C++ rewrite. 10× faster on the same hardware. Shard-per-core architecture. |
| **HBase** | Bigtable open-source clone | Strong single-row consistency. Pairs with HDFS / Hadoop. |
| **Google Bigtable** | The original (2006 paper) | Managed on GCP. Powers many Google products. |
| **DynamoDB** | Dynamo lineage | Managed AWS. KV + wide-column hybrid (hash + range key). |
| **Apache Accumulo** | NSA fork of Bigtable | Cell-level security. |
| **CockroachDB** (kind of) | Spanner-like, not pure wide-column | But the lowest layer uses RocksDB key-value much like Bigtable's tablets. |

For most people the practical choice is **Cassandra or ScyllaDB** (self-managed) or **Bigtable / DynamoDB** (managed).

---

## 3. Where They Came From

- **Bigtable (2006)** — Google's storage for indexed web pages, Earth, Analytics. Single-row strong consistency, multi-tablet, MapReduce friendly.
- **Dynamo (2007)** — Amazon's shopping-cart store. Multi-master, AP, eventually consistent, consistent-hashing ring.
- **Cassandra (2008)** — Facebook merged the two ideas: Dynamo-style ring + Bigtable-style data model.
- **HBase (2008)** — Apache OSS reimplementation of Bigtable on HDFS.
- **ScyllaDB (2015)** — Cassandra-compatible C++ rewrite optimizing every byte of CPU.

You'll find Bigtable's DNA in almost every modern distributed KV/column store (RocksDB, LevelDB, Pebble, ScyllaDB).

---

## 4. The Query Model — Design Tables For Your Queries

You can NOT query wide-column stores like SQL. The fundamental access patterns are:

- **By partition key**: very fast — go to the node, read the partition.
- **By partition key + clustering key range**: cheap range scan within a partition.
- **Across partitions**: requires scatter-gather across nodes — slow, expensive, sometimes refused.

So the data-modeling principle becomes:

> **One table per access pattern. Denormalize to make every query a partition-key lookup.**

Want messages by room ordered by time? → `(room_id) clustering (sent_at, id)`.
Want a user's recent activity across rooms? → a separate denormalized table keyed on `user_id`.
Want the latest 10 messages in any room visited by a user? → yet another table.

**You write the same fact multiple times** to satisfy multiple read patterns. Storage is cheap; reads must be fast.

### A concrete example
```sql
-- Cassandra CQL

CREATE TABLE messages_by_room (
  room_id     uuid,
  sent_at     timestamp,
  msg_id      uuid,
  user_id     uuid,
  body        text,
  PRIMARY KEY ((room_id), sent_at, msg_id)
) WITH CLUSTERING ORDER BY (sent_at DESC, msg_id DESC);

CREATE TABLE messages_by_user (
  user_id     uuid,
  sent_at     timestamp,
  msg_id      uuid,
  room_id     uuid,
  body        text,
  PRIMARY KEY ((user_id), sent_at, msg_id)
) WITH CLUSTERING ORDER BY (sent_at DESC, msg_id DESC);
```

Same logical data, two physical tables, each tuned for one read.

---

## 5. The Storage Engine: LSM-Trees

Wide-column stores use **log-structured merge trees (LSM)** under the hood.

```
Writes go to:
  1. WAL (commit log on disk)            ← durability
  2. Memtable (in-memory sorted map)     ← hot writes
When memtable fills:
  3. Flushed to immutable SSTables on disk
Periodically:
  4. Compaction merges SSTables to keep read costs low
```

Compared to B-trees (Postgres / MySQL):
- **Writes are nearly sequential** → much higher throughput.
- **Updates** are out-of-place (write a new tombstone or version, compaction reconciles).
- **Reads may touch multiple SSTables** → bloom filters and indexes make this fast.
- **Compaction** is the operational headache — runs in the background, can spike I/O.

This is the same family of storage as RocksDB, LevelDB, ScyllaDB, MyRocks. Knowing LSM-tree behavior is essential for tuning these systems.

See: [Storage Engines (LSM vs B-tree)](../09-storage/storage-engines.md).

---

## 6. The Ring (Cassandra / Scylla)

Cassandra arranges nodes in a **token ring**. Each partition key hashes to a token; the token points to one (or more) replicas around the ring.

```
                node A   (tokens 0  – 25)
              /         \
        node D            node B   (tokens 26 – 50)
              \         /
                node C   (tokens 76 – 100)
                          ←  replicas chosen by walking clockwise
```

- **Replication factor (RF)**: how many copies per partition. Typically 3.
- **Consistency level (CL)**: how many replicas must respond to a read/write — `ONE`, `QUORUM`, `LOCAL_QUORUM`, `ALL`.
- **Coordinator**: any node can serve any request; it forwards to the right replicas.

Tunable consistency = your choice per query:
- `CL = ONE` — fastest, may read stale data.
- `CL = QUORUM` — strong-ish (majority).
- `CL = ALL` — slow but linearizable-ish across replicas.

For multi-DC: `LOCAL_QUORUM` reads inside one DC; cross-DC is async.

---

## 7. Wide-Column vs Key-Value vs Document

| | Wide-column | Key-value | Document |
| --- | --- | --- | --- |
| Access pattern | Partition key + clustering range | One key | One key, sub-paths |
| Query language | CQL / HQL / Bigtable API | `GET/SET` | JSON queries |
| Range scans | Within partition | No (mostly) | Limited |
| Indexes | Per-table secondary, limited | Usually none | Rich secondary |
| Best for | Time-series, messaging, large-scale event data | Caches, sessions, leaderboards | Variable-shape app data |
| Schema | Strict columns, flexible values | None | Flexible JSON |

A wide-column store is "key-value with an *ordered sub-key*." That ordered sub-key is the killer feature: efficient range reads inside a partition.

See: [Key-Value Stores](./key-value-stores.md) · [Document Stores](./document-stores.md).

---

## 8. When Wide-Column Shines

- **Time-series at scale** — IoT sensors, metrics archives, financial ticks. Partition by entity, cluster by time.
- **Messaging / chat history** — partition by conversation, cluster by sent_at.
- **Event sourcing / audit logs** — append-only by entity.
- **High-write workloads** (10k+ writes/sec sustained per node).
- **Multi-region, multi-master** with eventual consistency.
- **Predictable access patterns** — the table layout matches every query.

Big users: **Discord** (Cassandra → ScyllaDB), **Apple**, **Netflix**, **Uber** (in places), **Spotify**, **Instagram** (parts).

---

## 9. When It Hurts

- **Ad-hoc queries** — you can't just "SELECT … WHERE foo=bar" if you didn't model for it.
- **Joins / aggregations** across partitions — slow scatter-gather.
- **Strong consistency** is possible (`QUORUM`) but the system is designed for AP, not CP. Use a different store if you need linearizability everywhere.
- **Few nodes, small data** — operational complexity is unjustified. Postgres is friendlier.
- **Relational integrity** — there are no foreign keys.
- **Hot partitions** — a single partition under heavy load melts one node.
- **Operations** — Cassandra is famously *not* low-touch. ScyllaDB and managed Bigtable/DynamoDB ease this.

---

## 10. Common Anti-Patterns

- **Querying with `ALLOW FILTERING`** in Cassandra. Almost always means a missing table for that query.
- **Unbounded partitions** — partition by `user_id` for a chat-message table where one user has 100M messages = doomed.
- **Wide rows that exceed practical limits** (~100 MB per partition for Cassandra is the rough guidance; ScyllaDB tolerates larger).
- **Frequent updates and deletes** — tombstones accumulate, reads get slow, compaction can't keep up.
- **Secondary indexes used like in SQL** — they exist but are *local* per node and don't scale; materialized views/derived tables are better.
- **Using wide-column as a queue** — tombstones, hot partitions, bad fit.
- **Multi-table writes "transactionally"** — there are no multi-row transactions across partitions. Lightweight transactions (`IF NOT EXISTS`) exist but are slow.

---

## 11. Replication & Multi-DC

Native multi-DC is one of wide-column's superpowers. You configure:
- **Replication strategy**: `NetworkTopologyStrategy` with RF per DC.
- **Consistency level**: `LOCAL_QUORUM` for low-latency local reads; cross-DC replication is async.
- **Snitch**: tells the cluster what rack / DC each node lives in for replica placement.

You can stretch a single Cassandra/ScyllaDB cluster across continents, with local quorum reads in each region and asynchronous replication closing the gap. **Single-region writes with global eventual reads** is a typical pattern.

---

## 12. Operations: The Tax You Pay

- **Compaction** strategies (Size-Tiered, Leveled, Time-Window) — pick based on workload. Wrong choice = read amplification or disk explosion.
- **Repair** — anti-entropy process to reconcile replicas; must run regularly.
- **Tombstones** — expire correctly with `gc_grace_seconds` else you get the famous "tombstone overwhelmed" errors.
- **Hot nodes** — monitor partition-size and request distribution.
- **Bootstrapping new nodes** — streaming TBs of data takes hours.
- **JVM tuning** (Cassandra) — Scylla avoids this by being native code.
- **Upgrades** — rolling, careful, version-locked.

Tools: **nodetool**, **DataStax OpsCenter**, **Reaper** (repair management), **ScyllaDB Manager**, **JMX/Prometheus exporters**.

---

## 13. Time-Series in a Wide-Column Store

The most natural fit. Pattern:
```sql
PRIMARY KEY ((device_id, bucket), ts)
WITH CLUSTERING ORDER BY (ts DESC);
```
- `device_id, bucket` = partition (e.g., bucket = year-month).
- `ts` = clustering key, sorted by time.

Why bucket? To avoid unbounded partitions over time. A device with 10 years of data shouldn't share one giant partition; bucketing by month keeps each partition small.

Read: "give me the latest 100 readings for device X in May 2026" = a fast range scan inside one partition.

---

## 14. ScyllaDB — The Modern Drop-in

ScyllaDB:
- C++ instead of JVM.
- **Shard-per-core** — one thread pinned per core, lock-free.
- **DPDK / Seastar** networking — bypasses the kernel.
- **Cassandra-compatible** wire protocol → same drivers, same CQL.
- Often **5–10× the throughput** at the same hardware.

Many shops migrate Cassandra → ScyllaDB to slash node count and bill. Most code changes are zero; the tuning knobs differ.

---

## 15. HBase & Bigtable

- **HBase** lives on **HDFS**, scales to petabytes, integrates with Hadoop / Spark. Strong single-row consistency. Pairs with **OpenTSDB** for time-series. Operationally heavy.
- **Bigtable** (GCP) — the *original* model, managed. Used by Google Search, Maps, Analytics. Single-row strong consistency. No SQL but supports HBase API and a streamlined client.

Both are great when you're already in their ecosystem (Hadoop / GCP). Outside it, Cassandra/Scylla usually win.

---

## 16. Cheat Card

```
WIDE-COLUMN = sparse, sorted, distributed map.
  Key:  partition_key + clustering_key + column.
  Row:  many columns, may be sparse.

PLAYERS    Cassandra · ScyllaDB · HBase · Bigtable · DynamoDB-ish.

PROS       linear write scale, multi-DC, time-series friendly,
            tunable consistency, mature replication.
CONS       constrained query model, ops heavy, no joins,
            tombstone / compaction headaches, hot partitions.

DESIGN RULE
  one table per access pattern. denormalize.
  pick partition key for distribution + query locality.
  clustering key = order within partition.

LSM-TREE storage = sequential writes + background compaction.

REPLICATION  RF=3 typical · CL = LOCAL_QUORUM common.
RING        consistent hashing places replicas around the cluster.

WATCH OUT FOR
  unbounded partitions   wide-row growth
  ALLOW FILTERING        secondary indexes used like SQL
  tombstone storms       wrong compaction strategy
  multi-partition txns   ad-hoc analytics

USE WHEN
  Discord / Netflix / Uber-style high write throughput,
  time-series, messaging history, audit logs, multi-DC active-active.

DON'T USE
  Small data, relational, ad-hoc queries → use Postgres.
```

---

## 17. Resources

### Books
- *Cassandra: The Definitive Guide* (3rd ed.) — Carpenter & Hewitt.
- *Designing Data-Intensive Applications* — Kleppmann; Ch. 3 on storage and Ch. 5/6 on replication & partitioning.
- *HBase: The Definitive Guide* — Lars George.
- *Database Internals* — Alex Petrov.

### Papers (the canon)
- *Bigtable: A Distributed Storage System for Structured Data* — Chang et al. (Google, 2006).
- *Dynamo: Amazon's Highly Available Key-value Store* — DeCandia et al. (2007).
- *Cassandra: A Decentralized Structured Storage System* — Lakshman & Malik (Facebook, 2010).

### Docs
- **Cassandra docs** — <https://cassandra.apache.org/doc/latest/>
- **ScyllaDB docs** — <https://docs.scylladb.com/>
- **HBase docs** — <https://hbase.apache.org/book.html>
- **Google Bigtable** — <https://cloud.google.com/bigtable/docs>

### Articles
- "How Discord stores trillions of messages" — Discord engineering (Cassandra → ScyllaDB migration): <https://discord.com/blog/how-discord-stores-trillions-of-messages>
- "Cassandra data modeling best practices" — DataStax academy.
- "Scylla University" — free courses: <https://university.scylladb.com/>
- "Cassandra anti-patterns" — DataStax engineering blog.

### Videos
- ByteByteGo: Cassandra explained — <https://www.youtube.com/@ByteByteGo>
- Hussein Nasser Cassandra / Scylla videos — <https://www.youtube.com/@hnasr>
- ScyllaDB Summit talks — production stories.
- "Building a Database for a Trillion-Row Workload" — Discord at conference talks.

### Tools
- **nodetool** (Cassandra) — operations CLI.
- **cqlsh** — interactive shell.
- **DataStax Studio**, **DBeaver** — clients.
- **Reaper** — repair scheduler.
- **ScyllaDB Manager** — managed ops.
- **Stargate** — REST / GraphQL / Document API on top of Cassandra.

### Adjacent reading
- [SQL vs NoSQL](./sql-vs-nosql.md)
- [Key-Value Stores](./key-value-stores.md)
- [Consistent Hashing](./consistent-hashing.md)
- [Replication](./replication.md)
- [Sharding & Partitioning](./sharding-partitioning.md)
- [Storage Engines (LSM-tree)](../09-storage/storage-engines.md)
- [Time-Series Databases](./time-series-databases.md)

---

*Previous:* [← Document Stores](./document-stores.md)  |  *Next:* [Graph Databases →](./graph-databases.md)
