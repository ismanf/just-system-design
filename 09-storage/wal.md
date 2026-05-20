# WAL — Write-Ahead Logging

> **TL;DR** — A **write-ahead log (WAL)** is the sequential, append-only journal that databases write *before* they touch the actual data pages. The rule is simple and absolute: **log the intent first, change the state later.** If the process crashes, replay the log to recover. WAL is what makes the **D** in ACID possible without forcing every page write to be a synchronous random IO. It's also the foundation of replication, point-in-time recovery, and change data capture. Every serious storage engine has one — Postgres WAL, MySQL redo log, SQLite journal, RocksDB WAL, Kafka log segments, etcd Raft log. Understand the WAL and you understand how databases survive power failures, why `fsync` is sacred, and where the throughput / durability trade-offs live.

---

## 1. The Idea in One Sentence

> Don't tell the user the write succeeded until the log says it did.

The dance every write goes through:

```
1. Client: INSERT INTO orders ...
2. Engine appends a WAL record describing the change
3. Engine fsyncs the WAL
4. Engine ACKs the client
5. Engine (later, lazily) updates the actual data pages
6. Engine (later, asynchronously) flushes pages to disk
```

Until step 3 happens, the engine has not promised anything. Once step 3 happens, the engine has made an unconditional promise — even if the machine catches fire at step 4, the change is recoverable from the log.

Everything else in this page is detail on how that simple rule plays out at production scale.

---

## 2. Why It Exists — The Problem WAL Solves

Suppose you have a 16 KiB B-tree page and a single 100-byte row update.

Without WAL, you must either:
1. **Write the whole page synchronously** for every modification — random IO, awful throughput.
2. **Hope nothing crashes** before the page eventually gets written — data loss on power failure.

Neither is acceptable. A WAL gives you a third option:

3. **Append a small record describing the change to a sequential log**, then update the page in memory. Pages drift to disk on their own schedule. On crash, replay the log against the (possibly stale) page contents.

Sequential append is 10–100× faster than random page writes. That throughput differential is the entire economic justification for WAL.

```
Without WAL                With WAL
───────────────            ─────────────────────────────
random page write          sequential log append
~150 µs (SSD)              ~5–20 µs
~10 ms (HDD)               ~1 ms (HDD, batched)
no replay needed           replay on recovery
1 IO per write             ≪1 IO per write (group commit)
```

---

## 3. ARIES — The Algorithm Underneath

The standard recipe for WAL-based recovery is **ARIES** (Algorithms for Recovery and Isolation Exploiting Semantics), published by Mohan et al. at IBM in 1992. Almost every modern DB implements a flavor of it.

ARIES has three principles:

1. **Write-Ahead Logging**: log record on durable storage **before** the affected page is flushed.
2. **Repeating history during redo**: on recovery, redo *all* committed changes (including ones that crashed before being flushed) — then undo uncommitted ones.
3. **Logging changes during undo**: undo operations themselves are logged (as compensation log records) so that crashes during recovery are also recoverable.

The three phases of ARIES recovery:

```
┌─────────────────────────────────────────────────────────────┐
│ 1. ANALYSIS                                                 │
│    Scan WAL from last checkpoint → end.                     │
│    Determine which transactions were active at crash.       │
│    Determine which pages were dirty.                        │
├─────────────────────────────────────────────────────────────┤
│ 2. REDO                                                     │
│    Replay every log record from the earliest dirty-page LSN.│
│    Idempotent: each page knows its own LSN, skips records   │
│    already applied.                                         │
├─────────────────────────────────────────────────────────────┤
│ 3. UNDO                                                     │
│    For transactions that were active (not committed) at     │
│    crash time, walk their log records backwards, apply      │
│    compensation log records (CLRs) to roll them back.       │
└─────────────────────────────────────────────────────────────┘
```

A clean shutdown skips phases 2 and 3 because the checkpoint already proved everything was flushed.

---

## 4. Anatomy of a WAL Record

Most engines structure each WAL record like this:

```
+------------+-----------+--------+--------+--------+--------+-----------+
| LSN        | prev LSN  | tx ID  | type   | page#  | offset | payload   |
+------------+-----------+--------+--------+--------+--------+-----------+

LSN       Log Sequence Number — monotonically increasing position in the log
prev LSN  Pointer to this transaction's previous log record (for undo)
tx ID     Transaction identifier
type      INSERT / UPDATE / DELETE / COMMIT / ABORT / CHECKPOINT / CLR
page#     Which data page is affected
offset    Where in the page
payload   Old value, new value, or both, depending on type
```

The **LSN** is the heartbeat of the system. Every page header records the highest LSN that has touched it. During redo, the engine skips log records whose LSN ≤ the page's LSN — that change has already been applied. This makes redo idempotent and crash-during-recovery safe.

---

## 5. Physical vs Logical vs Physiological Logging

Three flavors of WAL records — most engines mix them:

| Type | What it logs | Example | Pro | Con |
|---|---|---|---|---|
| **Physical** | "On page P at offset O, change bytes X to Y" | InnoDB redo log | Simple, fast replay | Bound to exact page layout |
| **Logical** | "Insert row (id=42, name='Alice') into table T" | MySQL binlog, Postgres logical replication | Portable across versions / schemas | Requires re-execution; non-deterministic ops are hard |
| **Physiological** | Physical per-page, logical within a page | Postgres WAL, ARIES standard | Idempotent + compact | Most complex to implement |

Postgres famously uses **physiological logging** for crash recovery (its main WAL) and **logical logging** for replication (via logical decoding). Same log, two views.

---

## 6. The Three Sacred Rules

Three invariants make WAL correct. Violate any one and you have a database that lies about durability.

### Rule 1 — Log Before Data

The log record describing a change is durable on disk *before* the modified page is durable on disk.

This is why it's called *write-ahead* logging.

### Rule 2 — Commit ⇒ Log Durable

Before returning success to the client, the WAL up to and including the commit record must be `fsync`'d.

Without this, a "successful" commit can vanish on power loss. This is also the single biggest source of fake durability — engines that lie about fsync (network filesystems, certain SSDs with volatile caches, virtualization layers) silently destroy this guarantee.

### Rule 3 — Idempotent Redo

Replaying the WAL from any point must produce the same final state. This is achieved by storing the LSN of the last applied change in each page header.

If you understand only these three rules, you understand 80% of WAL.

---

## 7. Checkpoints

The WAL grows forever unless bounded. Two problems:
- **Replay time** grows linearly with log length.
- **Disk space** doesn't survive without bound.

A **checkpoint** is the engine's promise: "Everything before this LSN is durable on data pages."

```
1. Engine quiesces (or flushes in-flight) dirty pages up to the current LSN.
2. Engine writes a CHECKPOINT record into the WAL recording the new
   "redo start" LSN.
3. WAL segments older than the redo-start LSN can be archived or discarded.
```

After a checkpoint, recovery only needs to replay from the redo-start LSN — much faster.

### Fuzzy checkpoints
The naive checkpoint (stop the world, flush all pages, write checkpoint marker) is too disruptive. Real systems use **fuzzy checkpoints**: a checkpoint marker is written, dirty pages drain to disk in the background, and a follow-up record finalizes the checkpoint when drain is done. The system never blocks.

Postgres `checkpoint_timeout`, MySQL InnoDB's adaptive flushing, RocksDB's flush — all variants of fuzzy checkpointing.

---

## 8. Group Commit

The naive WAL costs you one fsync per commit. fsync is expensive (microseconds on NVMe, milliseconds on slower disks). At 10k commits/second with 1 ms fsync, you can't keep up.

**Group commit** batches multiple commits into a single fsync:

```
T1 commits at time t0 → buffered, waits
T2 commits at time t0+50µs → buffered, waits
T3 commits at time t0+90µs → buffered, waits
At time t0+100µs (or buffer full), fsync once for all three.
All three ACK simultaneously.
```

Trade-off: a small per-commit latency for vastly higher throughput. Postgres `commit_delay`, MySQL `innodb_flush_log_at_timeout`, Kafka's batch settings — all group commit.

Modern engines use **pipelined group commit** (Mariadb's leader-follower variant, Postgres 9.2's parallel commit) — the first committer takes the fsync, latecomers piggyback.

---

## 9. fsync — The Sacred Syscall

`fsync` (or `fdatasync`, or `FlushFileBuffers` on Windows) is the syscall that forces the OS to push buffered writes to physically durable media. The WAL contract collapses without it.

### The fsync corruption saga
For decades, engines assumed: "if fsync returns success, my data is durable." In 2018, Postgres developers discovered that on Linux, if a buffered write fails *between* fsync calls, the kernel discards the dirty pages but returns success on the next fsync. The infamous *fsync-gate* led to changes in how Postgres handles fsync failures (PANIC instead of retry).

Lessons:
- **fsync can lie** if the underlying storage lies (consumer SSDs with volatile cache, network filesystems with poor implementation, qemu virtio in certain modes).
- **fsync failure must be treated as fatal**, not retried.
- **Test your storage** with `diskchecker.pl`, `pgperffarm`, or `fio --fsync` to validate it actually honors durability.

### Linux `O_DIRECT` and `O_DSYNC`
- `O_DIRECT` — bypass the page cache; write goes straight to the storage layer. Used by InnoDB.
- `O_DSYNC` — every write is implicitly fsync'd before returning.
- Most engines use `O_DIRECT` + explicit `fsync` for maximum control.

### How fast is fsync?
Approximate latency budget (single fsync, well-tuned hardware):

```
NVMe SSD (enterprise, power-loss protected)   ~10–50 µs
NVMe SSD (consumer, FUA flush)                ~100–500 µs
SATA SSD (enterprise)                         ~200–500 µs
SATA SSD (consumer)                           ~1–10 ms
HDD                                           ~5–15 ms
Network filesystem                            varies wildly
```

If you need 100k commits/sec, the math is unforgiving without group commit.

---

## 10. WAL in the Wild — Real Implementations

### PostgreSQL WAL
- 16 MB segment files in `pg_wal/`.
- Physiological logging.
- `wal_level`: `minimal` / `replica` / `logical` controls how much is logged (extra detail needed for replication).
- Streaming replication ships WAL records to replicas in real time.
- Archive mode (`archive_mode = on`, `archive_command = ...`) ships completed WAL segments to S3 / cold storage for point-in-time recovery.
- Logical decoding plugins (`pgoutput`, `wal2json`) translate physiological records back into logical events for CDC.

### MySQL InnoDB
- **Redo log** (`ib_logfile0`, `ib_logfile1`) — physical, used for crash recovery.
- **Undo log** — separate, used for transaction rollback and MVCC.
- **Binlog** — logical, used for replication and CDC.
- Two-phase commit between redo log and binlog ensures consistency across both (XA).

### SQLite
- **Rollback journal** (default) — old page contents copied to a journal file before modification. Roll back by copying back.
- **WAL mode** (since 3.7) — append-only log of changes; readers don't block writers. Set `PRAGMA journal_mode = WAL`. The standard recommendation for most embedded uses.

### RocksDB / LevelDB
- WAL on every write before insertion into the memtable.
- When memtable flushes to SSTable, the WAL segment is no longer needed and can be deleted.
- `WAL_recycling` reuses log files to avoid filesystem overhead.

### Kafka
- The log **is** the database. Each partition is a WAL.
- Configurable durability (`acks=0/1/all` and `flush.messages`/`flush.ms`).
- No checkpoints — log retention bounds the size instead.

### etcd / Consul / Raft-based systems
- The Raft log is the WAL; consensus is durable replication of WAL entries.
- `wal_segment_size` controls rotation.

### S3 (object storage internals)
- Even object storage has internal WAL-like journals for metadata mutations.

---

## 11. WAL Beyond Recovery

WAL is not just for crash safety. Once you have a sequential, ordered, complete record of every change, you get powerful capabilities almost for free:

### Replication
Ship the WAL to a replica; the replica replays it. **Logical streaming replication** in Postgres, MySQL binlog replication, MongoDB oplog, Kafka mirror, etcd Raft replication — all WAL-based.

### Point-in-time recovery (PITR)
Restore the last base backup, then replay WAL up to a target LSN or timestamp. You can recover to "Tuesday at 3:42 PM, just before that DELETE."

### Change Data Capture (CDC)
Read the WAL as a stream of business events; ship to Kafka / a search index / a warehouse. Debezium, Maxwell, AWS DMS — all tail the WAL.

See [CDC](../04-databases/cdc.md).

### Hot standby reads
Replicas applying the WAL can serve read-only queries. Postgres hot standby, MySQL read replicas.

### Forensics
The WAL is a tamper-evident audit trail (especially with archive). When data goes missing at 03:14 AM, the WAL knows who did it.

---

## 12. Operational Reality

### Sizing
- **Postgres**: `max_wal_size` (default 1 GB) caps active WAL; archive growth needs disk monitoring.
- **MySQL**: `innodb_log_file_size` (now `innodb_redo_log_capacity`) at 1–4 GB for production. Small = frequent checkpoints = bad throughput.
- **RocksDB**: WAL size shrinks as memtables flush; usually small.

### WAL retention
WAL needed for:
- Recovery (until checkpoint clears it)
- Replication (until all replicas have consumed it)
- Archive / PITR (according to retention policy)

A stuck replica or failed archive command **freezes WAL deletion**. The most common Postgres outage cause: archive command failing silently → WAL fills the disk → primary stops.

Always monitor `pg_replication_slots.confirmed_flush_lsn`, `pg_stat_archiver`, and equivalent in your engine.

### Compression and encryption
- Postgres `wal_compression` compresses full-page images.
- Most engines encrypt WAL when encryption-at-rest is enabled — encrypt both pages and journal.

### WAL on a separate disk
Classic optimization: put the WAL on a dedicated, fast device (NVMe), data files on capacity disk (HDD or larger SSD). The WAL gets sequential writes optimized for the fast device; data gets random IO on bulk storage. Less common in cloud (separate EBS volumes), more common on-prem.

### Async vs sync commit
- **Sync commit** (default): wait for WAL fsync before ACK. Safest, slower.
- **Async commit** (`synchronous_commit = off` in Postgres, `innodb_flush_log_at_trx_commit = 0/2` in InnoDB): WAL flushed in batches; small window of data loss on crash.

Use async commit only when you know the cost of losing a few hundred ms of writes on crash. Many high-throughput analytics workloads accept this; banking does not.

### Replication mode flavors
- **Async replication**: primary commits, ships WAL afterwards. Replica lags; data loss possible on primary failure.
- **Semi-sync** (MySQL): primary waits for at least one replica to ack WAL receipt.
- **Sync replication**: primary waits for replica to ack WAL durable. Best durability, worst latency.
- **Quorum sync** (Postgres `synchronous_standby_names = ANY 2 (s1, s2, s3)`): wait for N of M to ack.

See [Replication](../04-databases/replication.md).

---

## 13. Common Mistakes / Anti-Patterns

- **Disabling fsync "for performance."** Postgres `fsync = off` and MySQL `innodb_flush_log_at_trx_commit = 0` will absolutely lose committed data on crash. Use only on throwaway dev databases.
- **Putting WAL on storage that lies about fsync.** Consumer SSDs with volatile cache, certain network filesystems, qemu without proper cache mode. Validate before betting on durability.
- **No monitoring of WAL disk fill rate.** WAL piles up due to failed archive / stuck replication slot → disk full → DB stops. Always alert at 80%.
- **`wal_keep_segments` too small** for replica recovery. Replicas falling behind hit "WAL gone" and must re-base from scratch.
- **Treating binlog as the source of truth in MySQL.** The redo log is what makes the engine crash-safe; binlog is for replication. Lose binlog → lose CDC; lose redo log → lose data.
- **Long-running transactions blocking checkpoint.** ARIES / Postgres can't truncate WAL past the oldest active xact. A 12-hour query holds back the world.
- **Logical replication on Postgres without a replication slot management strategy.** Inactive slots hold WAL forever — disk fills.
- **Mixing high-volume `UNLOGGED` tables with replication.** Unlogged tables skip WAL → not replicated → silent divergence.
- **Replication lag treated as a number, not a window.** "We're 30 seconds behind" can mean "if the primary dies, you lose the last 30 seconds of commits." Quantify the loss budget.
- **Trusting cloud-managed durability without testing failover.** Many "managed" databases have surprising defaults; chaos-test before production.

---

## 14. Worked Example — A Crash Walk-Through

```
T1 BEGIN
T1 INSERT row A (LSN 100, written to WAL, page in buffer pool)
T2 BEGIN
T1 UPDATE row B (LSN 101, written to WAL)
T2 INSERT row C (LSN 102, written to WAL)
T1 COMMIT (LSN 103, written to WAL, fsync'd → ACK to client)
*** CRASH ***  pages for A, B, C not yet flushed to disk
```

Recovery:

```
1. ANALYSIS
   - Read WAL backwards from end.
   - T1 committed at LSN 103 → "committed transactions: {T1}"
   - T2 did not commit → "active transactions: {T2}"
   - Dirty pages identified.

2. REDO
   - Replay from earliest dirty-page LSN.
   - Apply LSN 100 (T1 INSERT A) — page LSN was 0, so apply.
   - Apply LSN 101 (T1 UPDATE B) — apply.
   - Apply LSN 102 (T2 INSERT C) — apply. (We redo even uncommitted T2!)
   - Apply LSN 103 (T1 COMMIT) — record T1 final state.
   - End of redo.

3. UNDO
   - T2 active at crash → roll back.
   - Walk T2's records backwards: LSN 102 (INSERT C).
   - Apply compensation: DELETE C, write CLR at LSN 104.
   - T2 fully rolled back. Done.
```

After recovery: T1's changes durable, T2's changes gone, database consistent. The client who saw T1's ACK had a real promise.

---

## 15. Decision Rule

```
Are you building a system that takes writes and must survive crashes?
   YES → you need a WAL. Period.

Can you tolerate losing the last few hundred ms of writes on crash?
   YES → async commit is OK for throughput.
   NO  → sync commit + fsync mandatory; group commit for throughput.

Need replication or CDC?
   The WAL is your friend. Use it.

Need point-in-time recovery?
   Archive WAL to durable storage; design retention by RPO.

Don't fight WAL. It's the cheapest insurance you'll ever buy.
```

---

## 16. Cheat Card

```
PURPOSE     Append-only journal of every change, fsync'd before
            commit. Recovers crash-state, drives replication and CDC.

RULES       1. Log record durable before data page.
            2. Commit ⇒ WAL fsync before ACK.
            3. Redo idempotent (page LSN gates it).

PHASES      Analysis → Redo (everyone) → Undo (uncommitted txns)

KEY CONCEPTS
            LSN         monotonic position in the log
            checkpoint  "all changes <= LSN_X are on disk"
            group commit batch fsyncs to amortize cost
            fuzzy ckpt  background drain, no stop-the-world

ENGINES     Postgres pg_wal  ·  InnoDB redo log + binlog  ·
            SQLite WAL mode  ·  RocksDB WAL  ·  Kafka log  ·
            etcd Raft log

USES        Crash recovery · Replication · PITR · CDC · Audit

PITFALLS    fsync lies · disabled fsync · WAL disk fills (failed
            archive / stuck slot) · long-running txns blocking
            checkpoint · async commit without budget

RULE        "Write the intent first, change the state later."
            That sentence is half of database engineering.
```

---

## 17. Resources

### Papers
- "ARIES: A Transaction Recovery Method Supporting Fine-Granularity Locking and Partial Rollbacks Using Write-Ahead Logging" — Mohan et al., IBM, 1992. The foundational paper.
- "Aether: A Scalable Approach to Logging" — Johnson et al., VLDB 2010.
- "Foundations of Recovery" — Härder & Reuter, 1983. Defines the ACID acronym; sets the scene for WAL.

### Books
- *Transaction Processing: Concepts and Techniques* — Gray & Reuter. The bible of recovery.
- *Database Internals* — Alex Petrov. Chapters on WAL, recovery, and B-tree integration are excellent.
- *Designing Data-Intensive Applications* — Martin Kleppmann. Chapters 3 and 5 set the context.
- *The Internals of PostgreSQL* — Suzuki Hironobu. Free online. Chapter 9 is a magnificent WAL walk-through.

### Articles
- "PostgreSQL's WAL" — interdb.jp: <https://www.interdb.jp/pg/pgsql09.html>
- "Fsync-gate" — Postgres devs unpack Linux fsync semantics (2018 mailing list thread, then LWN coverage).
- "MySQL InnoDB Redo Log Deep Dive" — Percona engineering blog.
- "SQLite WAL mode" — official docs: <https://www.sqlite.org/wal.html>
- "How RocksDB's WAL Works" — Facebook engineering blog.
- "Don't Forget About fsync()" — Daniel Stenberg, Andrew Morton commentary.

### Videos
- CMU 15-445 — Andy Pavlo's lecture on logging and recovery. Excellent ARIES walk-through.
- ByteByteGo — "Write-Ahead Logging" overview.
- Talks from Postgres Vision / PGCon on WAL internals.

### Tools
- **pg_waldump** — inspect Postgres WAL records.
- **mysqlbinlog** — read MySQL binlog.
- **wal2json**, **pgoutput** — Postgres logical decoding plugins.
- **Debezium** — CDC engine reading WALs across many DBs.
- **diskchecker.pl** — test whether your hardware honors fsync.

### Adjacent reading
- [Storage Engines (LSM-Trees vs B-Trees)](./storage-engines.md)
- [Compaction & Tiered Storage →](./compaction.md)
- [Database Transactions & Isolation Levels](../04-databases/transactions-isolation.md)
- [MVCC — Multi-Version Concurrency Control](../04-databases/mvcc.md)
- [Replication](../04-databases/replication.md)
- [Change Data Capture (CDC)](../04-databases/cdc.md)
- [Failover & Disaster Recovery](../11-reliability/failover-dr.md)

---

*Previous:* [← Storage Engines (LSM-Trees vs B-Trees)](./storage-engines.md)  |  *Next:* [Compaction & Tiered Storage →](./compaction.md)
