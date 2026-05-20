# Storage Engines (LSM-Trees vs B-Trees)

> **TL;DR** — A **storage engine** is the layer of a database that physically lays out bytes on disk and serves reads and writes against them. Two architectures dominate. **B-Trees** (Postgres, MySQL/InnoDB, Oracle, SQL Server) update pages in place — excellent for reads, range scans, and mixed workloads; harder to scale write throughput; rely on a write-ahead log for crash safety. **LSM-Trees** (RocksDB, Cassandra, ScyllaDB, LevelDB, HBase, BigTable, DynamoDB internals) buffer writes in memory and flush to immutable sorted files, periodically merging them via compaction — excellent for write-heavy workloads, but trades read amplification, write amplification, and compaction overhead for it. Most modern OLTP still picks B-Trees; most modern wide-column, KV, and time-series workloads pick LSMs. Understanding the trade-off is the difference between picking the right database and wondering why yours is slow.

---

## 1. What a Storage Engine Does

The database engine sits between the SQL/API layer and the disk:

```
   ┌────────────────────────────────────────────┐
   │ Query layer (parser, planner, executor)    │
   └──────────────────┬─────────────────────────┘
                      │ "fetch row id=42"
                      ▼
   ┌────────────────────────────────────────────┐
   │ Storage engine                             │
   │   - index lookup                           │
   │   - buffer pool / page cache               │
   │   - read pages from disk                   │
   │   - on write: WAL append + page update     │
   │   - on commit: durability fsync            │
   └──────────────────┬─────────────────────────┘
                      ▼
                    Disk
```

The engine answers four hard questions:
1. **How are records laid out** on disk so we can find them quickly?
2. **How do writes become durable** without losing data on crash?
3. **How do concurrent readers and writers** see consistent data?
4. **How do we reclaim space** from deleted/updated data?

Two dominant answers exist: **B-Trees** and **LSM-Trees**. Almost every disk-based database is one, the other, or a hybrid.

---

## 2. The B-Tree Family

A **B-Tree** is a balanced n-ary search tree. The disk-friendly variant in databases is the **B+ tree**: all data lives in leaves, leaves are linked in a doubly-linked list for range scans, internal nodes are pure routing.

```
                  ┌───── [50, 100] ──────┐
                  │                       │
              ┌── ▼ ──┐               ┌── ▼ ──┐
            [20, 35] [60, 80]       [120, 180]
             │  │  │   │  │  │        │  │  │
            ...leaf pages with rows...

   Leaf pages are linked:  [3,4,7] ↔ [10,15,18] ↔ [20,...]
```

### Properties
- **Pages** are fixed-size (typically 4–16 KiB). The engine reads and writes pages, not bytes.
- **Updates happen in place** — to modify a row, find its leaf page, update the bytes, write the page back.
- **Crash safety** comes from a **write-ahead log (WAL)**: every change is appended to the log before the page is rewritten. See [WAL →](./wal.md).
- **Balanced height** — fanout of 100–500 gives ≤4 levels for billions of rows.
- **Range scans** are linked-list walks of leaf pages.
- **Concurrency** uses page-level latches plus row-level locks plus MVCC. See [MVCC](../04-databases/mvcc.md).

### Page splits and merges
Inserting into a full leaf forces a **split**: create a new page, redistribute entries, insert a routing key in the parent. Deletes that drop a page below half-fill may trigger a **merge**. Splits and merges propagate upward, occasionally all the way to the root.

This is the source of B-Tree write amplification: a single tuple update can rewrite multiple pages.

### Real implementations
- **InnoDB** (MySQL, MariaDB) — B+ tree, clustered index by primary key.
- **PostgreSQL** — B+ tree heap + indexes; heap is not clustered.
- **SQLite** — B-tree per table; everything in one file.
- **Oracle / SQL Server** — B+ tree, decades of refinements.
- **LMDB / BoltDB** — memory-mapped B+ tree, copy-on-write.
- **WiredTiger** (MongoDB) — B+ tree with checkpoint-and-WAL crash recovery.

### Strengths
- **Read-optimized.** Logarithmic lookups; minimal read amplification.
- **Range scans are fast** thanks to linked leaves.
- **Predictable performance** under mixed workloads.
- **In-place updates** keep storage close to logical size.
- **Mature concurrency control** — decades of refinement.

### Weaknesses
- **Random writes hurt.** Updating widely-scattered rows pulls many pages into cache and writes many pages back.
- **Write amplification from page splits and full-page rewrites** — to change one row you rewrite the whole page.
- **Fragmentation** — repeated inserts/deletes leave half-empty pages; `VACUUM` (Postgres) or `OPTIMIZE TABLE` (MySQL) reclaims.
- **Index maintenance cost** — secondary indexes multiply per-write IO.

---

## 3. The LSM-Tree Family

A **Log-Structured Merge Tree** flips the trade-off: writes are easy, reads work harder.

```
   ┌───────────────────────────────┐
   │ MemTable (in RAM)             │   sorted map (e.g., skiplist)
   │   writes go here              │
   └────────┬──────────────────────┘
            │ when full, flush to disk
            ▼
   ┌───────────────────────────────┐
   │ Level 0: SSTables (sorted)    │   immutable, possibly overlapping
   ├───────────────────────────────┤
   │ Level 1                       │
   ├───────────────────────────────┤
   │ Level 2                       │
   │   ...                         │
   ├───────────────────────────────┤
   │ Level N (largest, oldest)     │
   └───────────────────────────────┘
        ▲
        │ compaction merges levels
```

### Properties
- **Writes are sequential.** Append to a WAL + insert into an in-memory sorted structure (skiplist, red-black tree). When the memtable fills, flush it as an **SSTable** (sorted string table) — an immutable, sorted, blocky file.
- **Files are immutable.** Updates and deletes don't rewrite anything; they write a new record with a higher sequence number, or a **tombstone** for deletes.
- **Reads check memtable + all SSTables that could contain the key.** Bloom filters and per-file key ranges prune candidates.
- **Compaction** periodically merges SSTables to reclaim space and reduce per-key fragmentation.

### Levels and compaction styles
- **Leveled compaction** (RocksDB default): each level is ~10× the size of the previous. Keeps per-key duplicates low at the cost of higher write amplification.
- **Size-tiered compaction** (Cassandra historical default): merge SSTables of similar size. Fewer compactions, higher read amplification.
- **Universal compaction** (RocksDB, default in some flavors): merges all files at the top until conditions favor splitting. Balances the two.
- **Tiered + leveled hybrids** (Cassandra `UnifiedCompactionStrategy`, ScyllaDB ICS).

### Real implementations
- **RocksDB** — Facebook fork of LevelDB; the embedded LSM library everyone uses. Powers MyRocks, CockroachDB, TiDB, YugabyteDB, Kafka Streams state stores, many others.
- **LevelDB** — Google's original.
- **Cassandra / ScyllaDB** — distributed LSM-based wide-column stores.
- **HBase / Bigtable** — LSM under the hood with HDFS or Colossus.
- **InfluxDB (TSI)**, **VictoriaMetrics**, **TimescaleDB (hyperchunks for compressed columnar)** — LSM-flavored time-series engines.
- **DynamoDB / Aurora storage** — internally LSM-derived.
- **Pebble** — CockroachDB's Go RocksDB rewrite.

### Strengths
- **High write throughput.** Sequential disk IO is 10–100× faster than random writes, both on SSD and HDD.
- **Compression friendly.** Sorted blocks compress brilliantly (50–80% reduction with Zstd/Snappy is common).
- **Small writes don't trigger page rewrites.**
- **Easy snapshotting.** Immutable files = file-level snapshots are nearly free.
- **Good for time-series and append-mostly data.**

### Weaknesses
- **Read amplification.** A point lookup may touch the memtable + N SSTables across levels. Bloom filters help but don't eliminate.
- **Write amplification from compaction.** Data is rewritten multiple times as it migrates down levels (factor of 10–30× is normal).
- **Space amplification.** Deletes are logical (tombstones); old versions linger until compaction.
- **Tail latency spikes.** Big compactions can stall reads/writes.
- **Tombstone hell.** Deletes that aren't fully compacted away can poison read paths — Cassandra users know the pain.
- **Tuning is hard.** Compaction strategies, level multipliers, bloom-filter sizing, block cache ratios — production tuning is an art.

---

## 4. Side-by-Side Comparison

| Property | B-Tree | LSM-Tree |
|---|---|---|
| **Write pattern** | Random page updates | Sequential append + memtable + compaction |
| **Read pattern** | Single (or few) page reads | Memtable + multiple SSTables (Bloom-filtered) |
| **Update semantics** | In-place | Append + tombstone; cleaned up by compaction |
| **Range scan** | Linked-list walk of leaves; fast | Merge-iterate across levels; OK |
| **Write throughput** | Limited by random IO | Very high (sequential IO) |
| **Read latency (point)** | Predictable, low | Variable; can spike on cold cache |
| **Write amplification** | Low–moderate (page rewrites) | Moderate–high (compaction) |
| **Read amplification** | Low | Moderate |
| **Space amplification** | Low–moderate (fragmentation) | Moderate (tombstones, multi-version) |
| **Compression** | Page-level, modest | Block-level, excellent |
| **Crash recovery** | WAL replay → repair B-tree | WAL replay → rebuild memtable |
| **Snapshot** | Page-level CoW or backup | File-level (immutable SSTables) |
| **Best for** | OLTP, mixed reads/writes, range scans | Write-heavy, time-series, large datasets, log-structured |
| **Canonical engines** | InnoDB, Postgres, WiredTiger, LMDB | RocksDB, LevelDB, Cassandra, HBase, ScyllaDB |

---

## 5. The Three Amplifications

Every storage engine has three amplification factors, and tuning is largely about balancing them:

```
Write amplification (WA):
  bytes written to disk / bytes the user asked to write

Read amplification (RA):
  pages/files read to serve a query / pages strictly required

Space amplification (SA):
  bytes on disk / bytes the dataset logically occupies
```

You can't minimize all three simultaneously — it's a triangle. The choice of engine and tuning picks a point on it.

| Engine | WA | RA | SA |
|---|---|---|---|
| B-Tree | medium | low | low–medium |
| LSM (leveled) | high | low–medium | low |
| LSM (size-tiered) | medium | medium–high | medium–high |
| Append-only log + compaction (Bitcask) | low | low (in-memory index) | high until compacted |

The **RUM conjecture** (Athanassoulis, 2016) formalizes this: you can be good at Read, Update, or Memory — pick two.

---

## 6. The Anatomy of a Write

### B-Tree write (e.g., Postgres)
```
1. BEGIN
2. Locate the row's page using indexes
3. Pin the page in the buffer pool
4. Append WAL record (insert/update/delete with before+after image)
5. Modify the page in memory
6. Mark page dirty
7. COMMIT → flush WAL up to commit LSN (fsync)
8. Background writer eventually writes dirty pages to disk
9. Periodically: checkpoint (fsync all dirty pages, advance recovery point)
```

The page might not hit disk for minutes. Durability comes from the WAL.

### LSM write (e.g., RocksDB)
```
1. Append to WAL
2. Insert into in-memory MemTable (sorted)
3. ACK to client
4. When MemTable fills:
     - Switch to a new MemTable
     - Flush old MemTable to disk as a Level-0 SSTable
5. Compaction picks SSTables to merge, writes new SSTables, deletes old ones
6. Periodically: drop WAL segments whose data is safely flushed
```

The write path is mostly memory + one append. The cost is paid later in compaction.

---

## 7. The Anatomy of a Read

### B-Tree read
```
1. Walk index from root → leaf (≤4 page reads typically)
2. Read leaf page (or follow pointer to heap, in Postgres)
3. Apply MVCC visibility check
4. Return row
```

A handful of page reads — most often from the buffer cache, sometimes from disk.

### LSM read
```
1. Check MemTable
2. For each immutable MemTable, check it
3. For each Level-0 SSTable (newest first):
     a. Check Bloom filter
     b. If positive, binary-search the index block, then read data block
4. For each higher level (L1, L2, ...):
     a. Binary-search the level's per-SSTable key ranges
     b. Check Bloom filter
     c. Read the relevant SSTable's data block
5. Merge results by sequence number, honor tombstones
6. Return latest value (or "not found")
```

Bloom filters and the block cache make this fast in practice — but you can see why read amplification is the cost.

---

## 8. Crash Recovery

Both families converge on the same answer: **write-ahead log (WAL)**. Every change is durably written to a sequential append-only log before the in-memory state is acked.

On crash:
- **B-Tree** — replay WAL from the last checkpoint, redoing every change against the pages.
- **LSM** — replay WAL into a fresh MemTable; SSTables are immutable so they're already safe.

See [Write-Ahead Logging →](./wal.md) for the full mechanic.

---

## 9. Worked Example — Same Workload, Different Engines

**Workload:** insert 10 billion key-value pairs, average 200 bytes each, then do mostly random point reads + occasional range scans.

### On a B-Tree (e.g., InnoDB)
- Each insert finds its position, splits pages occasionally, modifies the buffer pool.
- After 2 TB of data, the B-tree is ~4 levels deep with ~500 fanout.
- Random reads: typically 0–2 disk IOs per read (most internal nodes cached).
- Random inserts: page-splits + buffer pool churn cause random write IO. Throughput plateaus.
- Storage: ~2.3 TB on disk (fill factor < 100%).
- WA ≈ 4–8× depending on tuning.

### On an LSM (e.g., RocksDB leveled)
- Inserts hit memtable + WAL — sustained millions/sec on modern hardware.
- Compaction runs continuously; WA accumulates to 20–30× across levels.
- Random reads: Bloom filters catch most negatives; positive lookups touch 1–3 SSTables.
- Range scans: merge-iterate across levels; slower than B-tree for tight ranges, similar for big scans.
- Storage: ~1.5 TB on disk (Zstd compression of sorted blocks).
- Write throughput: 5–10× the B-tree under sustained load.

Same workload, very different cost shapes. The choice is workload-driven.

---

## 10. Hybrids and Variants

Real systems rarely live in pure-B-tree or pure-LSM land:

- **MyRocks** — MySQL with RocksDB as the storage engine. Same SQL, LSM internals.
- **CockroachDB / TiDB / YugabyteDB** — distributed SQL on Pebble/RocksDB.
- **Postgres + pg_lsm / Citus columnar** — columnar / LSM-like extensions.
- **WiredTiger LSM mode** — MongoDB's engine supports both B-tree and LSM trees.
- **Bw-Tree** (Microsoft Hekaton, FoundationDB Redwood) — lock-free B-tree variants for in-memory + flash.
- **Fractal trees** (TokuDB) — B-tree variant with buffered messages, somewhere between B-tree and LSM.
- **Copy-on-Write B-Trees** (LMDB, BoltDB, ZFS-style) — immutable pages, MVCC by construction.
- **Bitcask / WiscKey** — separate key index from value log; reduces compaction overhead for large values.
- **Columnar engines** (DuckDB, Parquet-backed, Snowflake) — not in the B-tree/LSM rivalry at all; oriented for analytical scans.

The lines blur. Pebble is "LSM" but borrows ideas from B-trees. Postgres is "B-tree" but uses LSM-flavored ideas for some indexes (BRIN).

---

## 11. When to Pick Each

### Pick a B-Tree engine when
- You're doing mixed OLTP — reads, writes, range queries, joins.
- Latency matters more than throughput.
- You need mature MVCC and transaction semantics.
- You're using Postgres / MySQL / SQL Server and the engine ships with the DB. (You don't actually pick.)
- Range scans are common (analytics over recent data, leaderboards, secondary index sweeps).
- Data set fits comfortably in disk, or write throughput isn't your bottleneck.

### Pick an LSM engine when
- Writes massively outpace reads (logging, metrics, event ingest).
- Sustained ingestion rates of 100k–1M+ ops/sec per node.
- Data set is much larger than RAM and compression matters.
- Compaction stalls are acceptable for your latency budget.
- You want easy time-travel / snapshotting.
- You're using Cassandra / HBase / RocksDB-based DBs. (Again, you don't actually pick.)

### Honest defaults
- **OLTP + relational + mixed workload?** Postgres. (B-tree.)
- **Time-series + ingest-heavy + simple queries?** TimescaleDB / VictoriaMetrics / InfluxDB. (Hybrid or LSM.)
- **Distributed KV with massive write throughput?** Cassandra or ScyllaDB. (LSM.)
- **Embedded high-throughput KV?** RocksDB. (LSM.)
- **Embedded read-heavy KV?** LMDB. (CoW B-tree.)

---

## 12. Operational Reality

### B-Tree
- **VACUUM matters.** Postgres dead tuples accumulate; autovacuum keeps the tree healthy. Misconfigure it and tables bloat 10×.
- **Fragmentation** is real; `REINDEX` and `pg_repack` are tools, not academic curiosities.
- **Buffer pool sizing** is the single most important knob (`shared_buffers`, `innodb_buffer_pool_size`). Default to ~25% of RAM for Postgres, ~70% for InnoDB on a dedicated host.
- **Index bloat** from heavy updates causes performance cliffs. Monitor.

### LSM
- **Compaction tuning** is everything. Misconfigured leveled vs tiered compaction can double or triple your IO.
- **Bloom-filter sizing** trades RAM for read amp. Don't skimp.
- **Tombstone purge windows** (Cassandra `gc_grace_seconds`) prevent zombie deletes from coming back in a multi-replica system. Delete-heavy workloads need careful tuning.
- **Wide rows** (in Cassandra) can break things — keep partitions under tens of MB.
- **Compaction stalls** cause p99 latency spikes. Many production systems oversize disks and parallelism to hide them.

---

## 13. The RUM Conjecture, Restated

The 2016 paper by Athanassoulis et al. proved (in the spirit of CAP) that no storage engine can simultaneously minimize:

- **R**ead overhead
- **U**pdate overhead
- **M**emory (or space) overhead

You always trade away one. B-trees give up Update overhead. LSMs give up Read overhead. Append-only logs give up Memory. Choose with eyes open.

---

## 14. Common Mistakes / Anti-Patterns

- **Using an LSM engine for read-latency-critical workloads** without sizing the Bloom filters and block cache appropriately. The first compaction stall will hurt.
- **Heavy deletes on LSM systems** — tombstones cost you at read time and at compaction time. Prefer TTL-based expiry.
- **Hot-partition writes on LSM** — concentrated writes on one partition key can starve compaction or create giant SSTables. Spread the key space.
- **Updating wide rows in B-trees** — repeated UPDATEs of large TEXT/BLOB columns cause page splits and bloat. TOAST in Postgres mitigates but doesn't eliminate.
- **No vacuum / no autovacuum** in Postgres. The single most common reason for B-tree-based databases dying in production.
- **Tiny block sizes on LSM** — too-small SSTable blocks hurt compression and read efficiency. Use the default (64 KB) unless you have a reason.
- **Mixing OLTP and OLAP on the same engine** — neither family is great at both. Use a separate analytical engine or read replica.
- **Trusting default compaction strategy** for unusual workloads. Time-series + Cassandra default `SizeTieredCompactionStrategy` is famously suboptimal; use TWCS instead.
- **Ignoring the WAL** — both families lose data if you tune `fsync` away. See [WAL](./wal.md).
- **Assuming the engine is irrelevant** because "the DB does the right thing." The engine choice baked into your DB defines the shape of your throughput / latency / cost curves.

---

## 15. Decision Tree

```
Is your workload write-bound (millions of writes/sec, ingest-heavy)?
   YES → LSM (RocksDB / Cassandra / ScyllaDB / time-series engine)
   NO  → continue
Are reads + range scans the dominant operations?
   YES → B-Tree (Postgres / MySQL / SQLite)
   NO  → continue
Do you need ACID with complex multi-row transactions?
   YES → B-Tree-backed RDBMS
   NO  → continue
Are your values much larger than your keys (multi-KB blobs)?
   YES → consider WiscKey-style (key-value-separated) LSM
   NO  → either is fine; pick by ecosystem fit
```

---

## 16. Cheat Card

```
PURPOSE     The bottom layer of a database: how bytes go to disk
            and how queries find them again.

B-TREE      In-place updates on fixed-size pages. Logarithmic
            lookups. Range scans via linked leaves. WAL + checkpoints.
            Used by: Postgres, MySQL/InnoDB, Oracle, SQL Server,
                     SQLite, LMDB, WiredTiger.

LSM-TREE    Writes go to memtable + WAL; flushed as immutable
            sorted SSTables; compaction merges levels.
            Tombstones for deletes. Bloom filters for reads.
            Used by: RocksDB, LevelDB, Cassandra, ScyllaDB,
                     HBase, Bigtable, Pebble, DynamoDB internals.

AMPLIFICATION    B-Tree:  low RA · medium WA · low SA
                 LSM:     medium RA · high WA · low–medium SA
                 You can't minimize all three (RUM conjecture).

WHEN B-TREE    Mixed OLTP · range scans · low-latency · ACID
WHEN LSM       Write-heavy · ingest · time-series · large datasets

PITFALLS    B-Tree: bloat, no vacuum, undersized buffer pool
            LSM: compaction stalls, tombstones, hot partitions,
                 misconfigured compaction strategy

RULE        OLTP → B-tree. Write-mostly → LSM. The engine
            choice is the shape of your throughput curve.
```

---

## 17. Resources

### Books
- *Designing Data-Intensive Applications* — Martin Kleppmann. Chapter 3 is the canonical introduction to both families.
- *Database Internals* — Alex Petrov. Best technical deep-dive; covers B-trees, LSM, WAL, MVCC at engine level.
- *Database Management Systems* — Ramakrishnan & Gehrke. Classic textbook on B-trees and indexing.
- *Transaction Processing: Concepts and Techniques* — Gray & Reuter. Old but still indispensable for the durability story.

### Papers
- "The Log-Structured Merge-Tree (LSM-Tree)" — O'Neil et al., 1996.
- "Bigtable: A Distributed Storage System for Structured Data" — Chang et al., 2006. (LSM in production.)
- "RocksDB: Evolution of Development Priorities in a Key-Value Store Serving Large-Scale Applications" — Facebook, 2020.
- "WiscKey: Separating Keys from Values in SSD-conscious Storage" — Lu et al., FAST 2016.
- "Designing Access Methods: The RUM Conjecture" — Athanassoulis et al., EDBT 2016.
- "Optimal Column Layout for Hybrid Workloads" — Athanassoulis et al., VLDB 2019.

### Articles
- "How does a relational database work" — Coding Geek. Long, excellent overview.
- Mark Callaghan's blog (smalldatum.blogspot.com) — production B-tree vs LSM comparisons.
- Markus Winand — *Use The Index, Luke!* — index internals in B-trees.
- Pebble, RocksDB engineering blogs — recent compaction strategy work.
- "Cassandra Tombstones" — DataStax operational guides.

### Videos
- CMU 15-445 / 15-721 — Andy Pavlo's database courses. The lectures on storage layers and indexing are excellent.
- ByteByteGo — "B-Trees vs LSM-Trees" overview.
- "Building a Storage Engine" — RocksDB devs at various conferences.

### Tools
- **rocksdb-bench / db_bench** — benchmarks RocksDB.
- **sysbench / HammerDB / pgbench** — benchmark B-tree-based RDBMSs.
- **pg_stat_statements / pg_stat_user_tables** — observe Postgres engine internals.
- **nodetool tablestats / sstabledump** — observe Cassandra LSM internals.

### Adjacent reading
- [Database Indexing (B-Tree, Hash, LSM-Tree)](../04-databases/indexing.md)
- [MVCC — Multi-Version Concurrency Control](../04-databases/mvcc.md)
- [Write-Ahead Logging →](./wal.md)
- [Compaction & Tiered Storage →](./compaction.md)
- [Wide-Column Stores](../04-databases/wide-column-stores.md)
- [Relational Databases](../04-databases/relational-databases.md)

---

*Previous:* [← Object Storage (S3, GCS, Azure Blob)](./object-storage.md)  |  *Next:* [WAL — Write-Ahead Logging →](./wal.md)
