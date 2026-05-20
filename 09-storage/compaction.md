# Compaction & Tiered Storage

> **TL;DR** — **Compaction** is the background process that rewrites storage to reclaim space, drop deleted/old records, and reorganize data for cheaper reads. Most prominent in **LSM-tree engines** (RocksDB, Cassandra, ScyllaDB, HBase) where it's load-bearing — without compaction, reads grind to a halt and disks fill with garbage. **Tiered storage** is the complementary idea of moving data between fast/expensive media (RAM, NVMe) and slow/cheap media (HDD, object storage, archive) based on access frequency. Together they govern the cost/performance shape of any large-scale storage system. Pick the wrong compaction strategy for your workload and you'll burn 10× the IO; pick the wrong tier for your hot data and you'll burn 100× the money.

---

## 1. The Big Picture

```
Compaction (vertical):
    Many small files / fragmented pages
              │
              ▼
    Merged / cleaned / rewritten output
              │
              ▼
    Less garbage, fewer files, better read locality

Tiering (horizontal):
    Hot data    →  RAM, NVMe SSD            ($$ per GB, fast)
    Warm data   →  HDD, slower SSD          ($$ per GB)
    Cold data   →  Object storage           ($ per GB)
    Frozen data →  Glacier / tape           (¢ per GB, slow restore)
```

These two ideas show up everywhere storage has to scale. They are how databases keep performing as data grows, and how cloud storage stays cheap.

---

## 2. Why Compaction Exists

Append-only systems (LSM-trees, WALs, immutable log structures) write fast precisely because they never modify old data — they just append new records. That's also their problem: deletes and updates pile up as duplicates and tombstones, and reads must merge across more and more files over time.

Compaction is the GC of append-only storage. It rewrites a subset of the data to:

1. **Drop tombstones** — once a delete is guaranteed visible to everyone, the tombstone and the original record can both go.
2. **Drop old versions** — under MVCC or LSM update semantics, only the newest value matters once older readers are gone.
3. **Merge sorted files** — fewer files at read time means fewer Bloom-filter checks and fewer seeks.
4. **Reclaim space** — physically free disk blocks.
5. **Improve locality** — adjacent keys move into adjacent disk blocks, helping range scans.

Without compaction:
- LSM trees devolve into "scan thousands of files per read."
- Postgres tables bloat indefinitely (dead tuples never reclaimed).
- Log archives turn into petabytes of dupes.
- Free disk space disappears one tombstone at a time.

---

## 3. LSM Compaction — The Main Event

In an LSM tree, every write creates new files (SSTables); none of them update in place. Without compaction the read path looks at every file ever written. The compaction strategy decides *which* files to merge, *when*, and *into what*.

The three classic strategies:

### 3.1 Size-Tiered Compaction (STCS)

```
Level 0:  [S0][S1][S2][S3]                   ~256 MB each
            │   │   │   │
            └───┴───┴───┘
                  ▼ merge when 4 of similar size exist
Level 1:        [S0+1+2+3]                   ~1 GB
                 ...
                 ▼
Level 2:    [merged big one]                 ~4 GB
                 ...
```

When you have K SSTables of roughly the same size, merge them into one larger SSTable. Default in older Cassandra.

**Pros**
- Cheap write amplification (~log_K(N) merges).
- Simple to implement.

**Cons**
- **Space amplification** is bad — until the final merge happens, you have multiple overlapping copies on disk. Often need 2–3× the logical data size.
- **Read amplification** is moderate — many overlapping SSTables in lower levels.
- Big merges (the final ones) are huge IO storms — kilometers of compaction queue then nothing.

Use when writes vastly dominate and disk is plentiful.

### 3.2 Leveled Compaction (LCS)

```
Level 0:  Small files from memtable flushes (may overlap)
Level 1:  Non-overlapping files, total size = 10× MemTable
Level 2:  Non-overlapping files, total size = 10× L1
Level 3:  Non-overlapping files, total size = 10× L2
   ...
```

Each level holds non-overlapping SSTables totaling roughly 10× the level above. When a level exceeds its target, a file is picked and merged with overlapping files in the next level.

**Pros**
- **Low space amplification** (~1.1×) — minimal duplication.
- **Low read amplification** — at most one SSTable per level can match a key (after L0).
- Predictable size growth.

**Cons**
- **High write amplification** — same data is rewritten log10(dataset/level1) times, often 20–30× total.
- Compaction must keep up with ingest or L0 fills and writes stall.

Default in RocksDB. Used by Cassandra with `LeveledCompactionStrategy` for read-heavy workloads.

### 3.3 Time-Window Compaction (TWCS)

```
Per-time-window buckets:
  [2026-05-19 00:00–06:00]   →   compaction within window only
  [2026-05-19 06:00–12:00]   →   ...
  [2026-05-19 12:00–18:00]   →   ...
```

For time-series workloads, data within a window is compacted together; old windows are eventually dropped wholesale (often via TTL).

**Pros**
- Excellent for time-series — old data dropped efficiently as whole files.
- Avoids cross-window IO storms.
- Reads for a time range hit a small number of files.

**Cons**
- Out-of-order writes (late-arriving data) break the model.
- Useless for non-time-ordered workloads.

Cassandra TWCS, ScyllaDB ICS time-window variants, InfluxDB and VictoriaMetrics use TWCS-style compaction internally.

### 3.4 Universal / Tiered+Leveled Hybrids

Modern engines blur the lines. **RocksDB Universal**, **Cassandra UnifiedCompactionStrategy**, **ScyllaDB Incremental Compaction** all try to balance the write amplification of leveled with the simplicity of size-tiered. They typically start tiered for fast writes, transition to leveled-like behavior at higher levels, and bound the compaction queue dynamically.

If you're not sure which to pick: **leveled for read-heavy, time-windowed for time-series, hybrid for everything else.**

---

## 4. The Amplification Trade-Off

The three amplifications appear again here:

```
                 Write     Read    Space
                 Amp       Amp     Amp
                 -----     ----    -----
STCS             low       med     HIGH (2–3×)
LCS              HIGH      low     low (~1.1×)
TWCS             low       low     low (per window)
Universal        medium    med     medium
```

Compaction policy is choosing which amplification to overspend on. SSDs forgive write amp better than HDDs (sequential write throughput is cheaper than capacity in NVMe land). HDDs forgive write amp poorly. Cloud charges you for everything.

---

## 5. Postgres VACUUM — B-Tree's Cousin to Compaction

Postgres isn't an LSM, but its MVCC model still produces garbage that needs cleaning. Every UPDATE creates a new tuple version; DELETE marks the old one dead but doesn't remove it. Without cleanup, tables bloat and indexes balloon.

**VACUUM** does for Postgres what compaction does for LSMs:
- Reclaim space from dead tuples.
- Update visibility map so the planner can skip pages.
- Avoid transaction-ID wraparound (must vacuum every table within ~2 billion transactions).

**Autovacuum** runs continuously, but only kicks in by default after a fraction of the table changes. Heavy-write tables outpace autovacuum and bloat fast.

```
VACUUM           remove dead tuples, mark space reusable
VACUUM FULL      rewrite entire table without dead tuples (LOCKS!)
VACUUM ANALYZE   also refresh planner statistics
pg_repack        non-blocking VACUUM FULL replacement
```

Production lesson: autovacuum tuning is a real job. Watch `pg_stat_user_tables.n_dead_tup`, set per-table `autovacuum_vacuum_scale_factor`, and don't be afraid to use `pg_repack` after a bulk update.

InnoDB has its own version — the **purge thread** removes undo logs and dead row versions. Same problem, same solution shape.

---

## 6. Compaction Costs You Should Measure

```
1. Write IO consumed by compaction (per second, sustained)
2. Space amplification (disk used / logical data size)
3. Compaction queue depth (number of pending tasks)
4. Read amp (avg files touched per read)
5. Tail latency during big compactions (p99, p999)
6. CPU spent compacting (often the bottleneck on modern hardware)
```

When something is wrong, one of these will be screaming. RocksDB exposes them as stats; Cassandra via `nodetool compactionstats`; Postgres via `pg_stat_progress_vacuum`.

---

## 7. Tiered Storage — The Idea

Not all data deserves the same hardware. The 80/20 rule applies aggressively to storage: a small fraction of data is hot, most is rarely accessed, and a long tail is essentially archival.

```
Cost per GB-month (cloud rough order):

  RAM cache                ~$10
  Local NVMe              ~$1
  Network SSD (io2)       ~$0.20
  Network SSD (gp3)       ~$0.08
  HDD-backed              ~$0.04
  Object storage (hot)    ~$0.023
  Object IA               ~$0.012
  Object archive          ~$0.004
  Glacier Deep Archive    ~$0.001
```

A 1000× cost gap exists between hot RAM and cold archive. Tiering policies decide which slice lives where.

---

## 8. Tiered Storage in Databases

### Postgres + extensions
Postgres core doesn't tier natively, but you can:
- Use partitioning (`pg_partman`) to keep hot partitions on fast disk, archive older partitions to slower storage or detach to S3 via foreign tables.
- Use `pg_tier`, `Citus`, or commercial extensions for explicit tier management.

### Cassandra / ScyllaDB
- Native tiered storage in Cassandra 4.0+ — write to fast disk, compact down to slow disk per strategy.
- ScyllaDB DC-aware tiering for hot/cold separation.

### Snowflake / BigQuery
- Storage is object-storage-native; compute is ephemeral. Tiering is *automatic and invisible*.
- Snowflake's micro-partitions + result cache + warehouse cache make multi-tier behavior implicit.

### ClickHouse
- Native multi-disk volumes with TTL policies — "move parts older than 30 days to cold disk."
- Common architecture: NVMe for last week, HDD for last quarter, S3 for everything older.

### OpenSearch / Elasticsearch
- **Hot-warm-cold-frozen** tiers. Hot nodes (NVMe) take ingest and recent searches; warm and cold demote to slower disk; frozen tier uses **searchable snapshots** in S3.
- Lifecycle policies (ILM) automate the migration.

### Druid / Pinot / Real-time analytics
- Streaming ingest into hot tier; segments age out to deep storage on object stores; queries fan out across tiers.

### Kafka tiered storage
- KIP-405 (Kafka 3.6+) — tiered storage moves older log segments to object storage. Keeps Kafka brokers small and disks cheap while retaining longer history.

---

## 9. Object Storage as the Cold Tier

The dominant cold tier in modern systems is object storage. The pattern:

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  Compute / DB                                                │
│      │                                                       │
│      ├── recent data on local NVMe   (fast tier)             │
│      ├── older data on EBS / HDD     (warm tier)             │
│      └── archive on S3 / GCS         (cold tier)             │
│                                                              │
│  Lifecycle policies move data between tiers automatically.   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

Variants:
- **Tiered with cache** (Snowflake, Databricks): all data on object storage, hot data cached on local SSD. Compute is elastic; storage is durable and cheap.
- **Tiered with sync** (Elasticsearch hot-warm-cold): explicit tiers, data moves on schedule, queries dispatched to right tier.
- **Tiered with restore** (Glacier-style archive): data in archive must be restored before reads — slow, used for compliance-only data.

The economic gravity is toward "compute + object storage." Modern OLAP and time-series systems are increasingly built this way.

---

## 10. Lifecycle Policies — The Operating Manual

In cloud object storage:

```yaml
Rule "log archive":
  Prefix: logs/
  Transition:
    - After 30 days  → STANDARD_IA
    - After 90 days  → GLACIER_IR
    - After 365 days → DEEP_ARCHIVE
  Expire after: 7 years

Rule "incomplete uploads":
  Filter: any multipart upload
  Abort after: 7 days

Rule "old versions":
  Filter: any noncurrent version
  Transition: 30 days → IA
  Expire: 90 days
```

If you don't write these rules, you eat the cost. Buckets without lifecycle policies are how cloud bills surprise people.

---

## 11. Compaction & Tiering Gotchas

### Compaction stalls writes
LSM engines back-pressure writers when compaction can't keep up. Symptoms: write latency spikes from microseconds to seconds, then settles. Causes: too few compaction threads, slow disk, oversized memtables.

Mitigations:
- Throttle ingest under back-pressure (e.g., Kafka producer with `batch.size` and `linger.ms`).
- Provision more compaction parallelism (`max_background_compactions` in RocksDB).
- Pre-split hot keyspaces to spread compaction load.

### Cassandra tombstone hell
Tombstones aren't removed until `gc_grace_seconds` (default 10 days) to prevent zombie deletes across replicas. A delete-heavy workload can accumulate so many tombstones that reads time out. Symptoms: `tombstone_warn_threshold` exceeded; `TombstoneOverwhelmingException`.

Mitigations:
- Use TTL-based expiry instead of explicit DELETEs.
- Use TWCS for time-series workloads — whole files dropped.
- Don't write null values that create tombstones.

### Big compactions in production
Major compactions (merging all data into one file) used to be a thing in Cassandra. They take days, hit IO ceilings, and starve normal operations. Modern advice: **never run a major compaction manually**; let the strategy do its thing.

### Read amp in tiered systems
Object storage cold tier reads are ~50 ms first-byte. If queries touch cold tier files often, latency goes sideways. Mitigate with:
- Aggressive metadata caching.
- Lazy promotion (pull cold data back to hot tier on access).
- Range-prefetch / multi-range reads.

### Restore time from archive
Glacier Deep Archive restore is **12 hours**. Compliance archives are fine. Operational data is not. Pick the right tier.

### Tiering with mutable data
Tiering works best with immutable / append-only data. Mutable data (an OLTP row that updates daily) doesn't tier well — it never gets cold.

### Snapshot semantics across tiers
Snapshots that span tiered storage are tricky. Verify your DR strategy includes tier-spanning consistent backups.

---

## 12. Real-World Compaction Architectures

### RocksDB at Meta
- Default leveled compaction.
- Subcompaction (parallel within a single compaction job).
- Tier-mode for write-heavy use cases.
- Trivial moves (rename, no rewrite) when SSTables can be promoted between levels without overlap.

### Cassandra at Apple
- Mix of LCS for read-heavy column families and TWCS for time-series.
- Aggressive monitoring of compaction queue.

### Cassandra → ScyllaDB at Discord
- Migrated from Cassandra to ScyllaDB, partly because compaction efficiency was better. Same data shape, half the nodes.

### ClickHouse at Cloudflare
- Multi-disk volumes with NVMe + HDD.
- TTL-based segment migration.
- S3-backed cold tier for historical analytics.

### Elasticsearch at GitHub
- Hot-warm-cold ILM for logs and audit events.
- Frozen tier with searchable snapshots in S3 for compliance retention.

### Snowflake
- Micro-partitions on S3 only.
- Result cache (24 hr), warehouse cache, metadata service make tiering invisible.

---

## 13. Common Mistakes / Anti-Patterns

- **Default compaction strategy for an atypical workload.** Cassandra time-series on STCS is famously bad; switch to TWCS.
- **Disabling compaction "temporarily."** It's never temporary — data piles up, future compaction becomes IO-storm bait.
- **Mass deletes on tombstone-heavy systems.** Use TTL or drop partitions instead.
- **No autovacuum tuning on Postgres.** Bloat outpaces defaults on heavy-update tables.
- **Tiering policies copy-pasted from a blog post.** Validate against actual access patterns. Lifecycle rules are easy to get wrong.
- **Forgetting incomplete-multipart cleanup** in S3 lifecycle — silent cost growth.
- **Putting OLTP indexes on cold tier.** Indexes need to be hot or the engine grinds.
- **Restoring from Glacier for an outage** without realizing it's a 12-hour SLA. Plan DR to a hot tier.
- **Not monitoring compaction queue depth.** When it's growing, you're an hour from write stalls.
- **Treating tiered storage as free.** Cold-tier reads still cost retrieval; archive tiers have minimum durations.
- **Mutable data on cold tier.** Updates re-promote; you pay double.
- **One compaction thread on a 64-core machine.** Provision compaction parallelism for the hardware.

---

## 14. Operational Playbook

When compaction is misbehaving:

```
1. Check compaction queue / pending tasks.
   - Growing? Compaction can't keep up. Either throttle ingest
     or increase compaction parallelism.
2. Check space amplification.
   - 2× or more? Switch from size-tiered to leveled or hybrid.
3. Check read amplification.
   - Multiple SSTables touched per read? Strategy mismatch or
     too many L0 files.
4. Check IO utilization.
   - Disk saturated? Either faster disks or a different strategy.
5. Look at largest SSTables.
   - One enormous file? Major-compaction relic; consider rewriting.
6. Watch tail latency during compactions.
   - p99 spikes? Smaller per-job size, throttled compactions, or
     dedicated compaction queue.
```

When tiering is misbehaving:

```
1. Check tier residency vs access pattern.
   - Hot data on cold tier? Lifecycle policy is wrong.
2. Check restore latency for the use case.
   - Are users waiting for Glacier? Promote one tier hotter.
3. Check cost per tier.
   - Cold tier 10% of bill? Often a sign you can be more aggressive.
4. Check egress / API cost.
   - Reads from cold tier crossing regions? Add caching or replicate.
```

---

## 15. Cheat Card

```
PURPOSE     COMPACTION reclaims space, drops garbage, reorganizes
            files in append-only / MVCC systems.
            TIERED STORAGE moves data between fast/expensive and
            slow/cheap media based on access patterns.

COMPACTION STRATEGIES
  STCS       merge K similar-sized SSTables. Cheap writes,
             high space amp.
  LCS        non-overlapping levels, 10× growth.
             Low space + read amp, high write amp.
  TWCS       per-time-window. Best for time-series.
  Universal  hybrid; balances all three amplifications.

POSTGRES    VACUUM / autovacuum / pg_repack — same job, B-tree style.
INNODB      purge thread — same job, different name.

TIERS       RAM > NVMe > SSD > HDD > Object > Archive > Tape
            Each step is 3–10× cheaper and 10–100× slower.

LIFECYCLE   Rules in object storage automate tier transitions.
            Forgetting them is the #1 silent cost mistake.

PITFALLS    Strategy mismatch · tombstone hell · disabled compaction ·
            no autovacuum · mutable data on cold tier · default
            policies · long-running txns blocking VACUUM · cold-tier
            indexes · restore-from-archive in an outage

RULE        Compaction is rent. You pay it continuously or
            catastrophically. Tiering is leverage — most data is
            cold, most cost should follow.
```

---

## 16. Resources

### Papers
- "The Log-Structured Merge-Tree (LSM-Tree)" — O'Neil et al., 1996.
- "bLSM: A General-Purpose Log-Structured Merge Tree" — Sears & Ramakrishnan, SIGMOD 2012.
- "Optimal Bloom Filters and Adaptive Merging for LSM-Trees" — Dayan et al., 2018.
- "Monkey: Optimal Navigable Key-Value Store" — Dayan & Idreos, SIGMOD 2017. Tuning the LSM amplification space.

### Books
- *Database Internals* — Alex Petrov. Chapters on LSM and SSTable management.
- *Designing Data-Intensive Applications* — Martin Kleppmann.
- *Cassandra: The Definitive Guide* — Eben Hewitt et al. Detailed compaction strategy chapters.

### Documentation
- **RocksDB Wiki** — leveled, universal, FIFO compaction docs: <https://github.com/facebook/rocksdb/wiki>
- **Cassandra Compaction Strategies** — DataStax docs.
- **ClickHouse Storage Policies** — <https://clickhouse.com/docs/en/operations/storage-policy>
- **Elasticsearch Index Lifecycle Management (ILM)** — official docs.
- **PostgreSQL VACUUM** — <https://www.postgresql.org/docs/current/sql-vacuum.html>

### Articles
- Mark Callaghan's blog — sustained LSM benchmarks for years.
- "Cassandra Tombstones in the Real World" — DataStax engineering.
- "Time-Window Compaction Strategy in Practice" — Apache Cassandra mailing list archives.
- "Snowflake's Architecture" — Snowflake engineering whitepapers.
- "How Discord Stores Billions of Messages" — Discord engineering blog.
- "Kafka Tiered Storage" — KIP-405 design doc and Confluent posts.

### Videos
- ByteByteGo — "Compaction in LSM Trees" overview.
- "Cassandra Internals" — DataStax conference talks.
- "Snowflake Architecture" — talks from Marcin Zukowski and others.
- CMU 15-721 — Andy Pavlo's storage layer lectures.

### Tools
- **db_bench** (RocksDB) — measures compaction throughput.
- **nodetool compactionstats / tablestats** (Cassandra).
- **pg_repack**, **pg_squeeze** — Postgres bloat removal.
- **rocksdb-tools** — sst_dump, ldb_tool inspect SSTables.
- **AWS S3 Storage Lens** — visualize tier residency.
- **clickhouse-client** with system.parts/system.disks — inspect tier residency.

### Adjacent reading
- [Storage Engines (LSM-Trees vs B-Trees)](./storage-engines.md)
- [WAL — Write-Ahead Logging](./wal.md)
- [Erasure Coding vs Replication →](./erasure-coding.md)
- [Wide-Column Stores](../04-databases/wide-column-stores.md)
- [Time-Series Databases](../04-databases/time-series-databases.md)
- [Cache Eviction Policies](../05-caching/eviction-policies.md)
- [Object Storage](./object-storage.md)

---

*Previous:* [← WAL — Write-Ahead Logging](./wal.md)  |  *Next:* [Erasure Coding vs Replication →](./erasure-coding.md)
