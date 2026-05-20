# MVCC — Multi-Version Concurrency Control

> **TL;DR** — **MVCC** is the technique that lets modern databases serve concurrent transactions without making readers and writers fight for locks. Every row update creates a new **version** rather than overwriting in place; each transaction sees a consistent **snapshot** of the database as of a chosen moment. The headline win: **readers never block writers, writers never block readers.** The cost: lots of dead row versions to clean up, which is why Postgres has `VACUUM` and MySQL InnoDB has the undo log. Postgres, Oracle, MySQL InnoDB, SQL Server snapshot isolation, CockroachDB, Spanner, SQLite WAL — all use MVCC.

---

## 1. Why MVCC Exists

Pre-MVCC databases relied on **2PL (two-phase locking)**: every read took a shared lock, every write took an exclusive lock, and you held them until commit. Concurrent readers and writers blocked each other. Throughput cratered under any contention.

MVCC takes a different angle:

> Each transaction sees a *consistent view* of the database — a **snapshot** — regardless of what other transactions are doing. Writes don't overwrite; they add a new version with a visibility window.

Now readers don't need shared locks at all. The DB serves whichever version was visible at the reader's snapshot.

---

## 2. The Core Idea

```
Initial state:
  row id=1   version v0   balance=100

T1 starts (snapshot s1). T1 reads → sees v0.
T2 starts (snapshot s2 > s1).
T2 UPDATE: row id=1 balance=200 → creates v1.
  Internally:
    v0: balance=100   created=t0  deleted=t2
    v1: balance=200   created=t2  deleted=∞

T1 (still running) reads row 1 → still sees v0 = 100 (its snapshot).
T2 commits.

T3 starts (snapshot s3 > t2). T3 reads → sees v1 = 200.

Eventually, no transaction can see v0 anymore. VACUUM reclaims it.
```

A row has a *creation timestamp* (the transaction that wrote it) and an *expiration timestamp* (the transaction that deleted/replaced it). A snapshot picks rows whose creation ≤ snapshot < expiration.

---

## 3. Implementation Flavors

### Postgres — version-per-row in the heap
- Each row in the heap has hidden `xmin` (creator txn) and `xmax` (deleter txn) columns.
- Updates create a new row in the heap and mark the old one with `xmax = updater`.
- The visibility check at read time consults the **transaction status** to decide which row versions you can see.
- Dead versions accumulate; `VACUUM` reclaims them. **Bloat** is an operational concern.

### MySQL InnoDB — undo log
- Updates *overwrite* the row in place but keep an **undo record** describing how to roll back.
- Older transactions construct their snapshot by walking undo records back from the current row.
- Less bloat in the heap; the undo log can grow huge if a long transaction holds an old snapshot open.

### Oracle — undo segments
- Similar to InnoDB. Undo segments grow with long-running transactions.

### SQL Server — version store in `tempdb`
- Row versions for snapshot isolation live in `tempdb`. Misconfigured `tempdb` causes outages.

### CockroachDB / Spanner — MVCC over KV
- Storage layer (RocksDB / Pebble) holds versioned keys.
- Each write encodes its timestamp; reads filter by a read timestamp.
- Garbage collection runs continuously, bounded by a configurable retention window.

All are versions of the same idea: keep old data around long enough to satisfy concurrent snapshots.

---

## 4. Snapshot Isolation — What MVCC Gives You

Most engines implement **Snapshot Isolation (SI)** as a side effect of MVCC:

- A transaction sees a consistent snapshot as of its start (or its first statement).
- Other commits during your transaction are invisible to you.
- Your own writes are visible to you (read-your-writes).
- On commit, the engine checks for *write-write* conflicts (lost-update style); aborts if needed.

SI prevents **dirty reads**, **non-repeatable reads**, and **phantoms in single-table queries**. It does **not** prevent **write skew** by itself — that requires `SERIALIZABLE` (Postgres adds SSI on top of SI to handle it).

See [Transactions & Isolation Levels](./transactions-isolation.md).

---

## 5. The Visibility Algorithm (Postgres-flavor)

For each candidate row version:
1. Is the creator transaction (`xmin`) **committed at or before my snapshot**?
   - If no → invisible.
2. Was it deleted/replaced by a transaction (`xmax`) **committed at or before my snapshot**?
   - If yes → invisible.
3. Otherwise → visible.

The engine consults an in-memory snapshot of "which transaction IDs were committed at what time" to answer these questions cheaply.

---

## 6. Dead Versions, VACUUM, and Bloat

A row version is **dead** when no live transaction's snapshot can possibly see it. Dead versions:
- Take disk space.
- Pollute the buffer pool.
- Slow sequential scans.
- Hide live rows from quick `count(*)`-style queries.

Postgres `VACUUM` reclaims them; `VACUUM FULL` rewrites the whole table; `autovacuum` runs them automatically.

A **long-running transaction** holds back the snapshot horizon. Until it commits, *no* dead version newer than its snapshot can be removed → bloat balloons. The classic Postgres operational bug: a forgotten idle-in-transaction connection causing the DB to grow uncontrollably.

```
SELECT pid, state, age(now(), xact_start) AS tx_age
FROM pg_stat_activity
WHERE state = 'idle in transaction'
ORDER BY tx_age DESC;
```

Hunt these. Set `idle_in_transaction_session_timeout` to fail-fast.

---

## 7. What MVCC Does and Doesn't Solve

✅ Readers don't block writers (and vice versa).
✅ Consistent snapshots → no dirty reads.
✅ Predictable read performance under load.
✅ Cheap, fast, parallel reads at scale.

❌ Does *not* prevent **lost updates** if you read-modify-write outside the DB. Use explicit locks (`FOR UPDATE`), OCC version columns, or `SERIALIZABLE`.
❌ Does *not* prevent **write skew** under SI. Need `SERIALIZABLE` (SSI in Postgres).
❌ Does *not* eliminate locks entirely — writers still take row locks to serialize themselves.
❌ Comes with **bloat** and **vacuum** as operational concerns.

---

## 8. MVCC and Reads Beyond Single Rows

A snapshot is **database-wide** in many engines. You can run a long analytical query that takes minutes and see consistent data across many tables — because every read is filtered by your snapshot.

That's why dumping data with `pg_dump` or running an offline analytics query in transaction mode produces a coherent backup, even while OLTP traffic continues. MVCC's snapshot is the unsung hero of online operations.

---

## 9. MVCC Pitfalls

### Long transactions
The single biggest operational risk. Long transactions keep snapshots alive → vacuum can't reclaim space → bloat → slow scans → more bloat → outage.

Mitigations:
- `idle_in_transaction_session_timeout`.
- Code reviews to ban "open tx then network call".
- Monitoring for `pg_stat_activity.state = 'idle in transaction'`.
- Run analytics on a replica.

### Autovacuum tuning
Default settings work for small tables but lag on huge ones. Watch for:
- `n_dead_tup` growing in `pg_stat_user_tables`.
- `last_autovacuum` falling behind.
- High `bloat` reported by extensions like `pgstattuple`.

Tune autovacuum to be more aggressive on hot tables.

### HOT updates (Postgres)
If an update doesn't change any indexed column **and** there's room on the same page, Postgres does an in-place version chain ("HOT update") without updating indexes. This saves a lot of I/O — but is defeated if your hot table has too many indexes or you've changed indexed columns.

### Transaction ID wraparound (Postgres)
Transaction IDs are 32-bit. After ~2 billion, they wrap. Vacuum *must* run periodically to "freeze" old rows; if not, the DB will refuse new writes to protect itself. This has bitten famous outages (Sentry, MailChimp, etc.). Modern Postgres warns aggressively, but it's still a real risk in poorly-vacuumed systems.

---

## 10. MVCC in NoSQL and Distributed Databases

- **DynamoDB** — versions records internally; conditional writes (`expected version`) implement OCC at the API.
- **Cassandra** — last-write-wins by **timestamp**; old versions get reconciled via compaction; tombstones replace deletes.
- **CockroachDB / Spanner / TiDB** — multi-version KV underneath with a **closed-timestamp / TrueTime** notion of snapshots. Garbage collection horizon configurable.
- **MongoDB WiredTiger** — MVCC + snapshots, with checkpoint-based persistence.

The same pattern recurs: keep multiple versions, expose a snapshot view, garbage-collect old versions when safe.

---

## 11. Worked Example (Postgres flavor)

```
T1: BEGIN; SELECT balance FROM accounts WHERE id='A';   -- snapshot s1; sees 100
T2: BEGIN; UPDATE accounts SET balance=200 WHERE id='A'; COMMIT;   -- creates v1=200
T1: SELECT balance FROM accounts WHERE id='A';           -- still sees 100 (snapshot s1)
T1: UPDATE accounts SET balance = balance - 10 WHERE id='A';
    -- BLOCKS (T2's exclusive row lock no longer held — but the row is *now* version v1)
    -- T1's snapshot saw 100, but the physical update locks the latest row.
    -- Postgres: if the latest row version was committed AFTER T1's snapshot,
    -- and T1 tries to update it, T1 sees a serialization failure under REPEATABLE READ.
    -- Under READ COMMITTED, T1's UPDATE re-reads the latest row → 200, applies -10 → 190.

T1: COMMIT;
```

Two different outcomes depending on isolation level — that's MVCC's tunable nature. **Read Committed** transparently updates the latest version (last-write-wins-ish; lost-update risk). **Repeatable Read** errors out and forces you to retry.

The lesson: even with MVCC, app-side read-modify-write is unsafe. Use atomic SQL, `FOR UPDATE`, or a version column.

---

## 12. Tools to See MVCC

Postgres:
```sql
-- See dead tuples and last vacuum
SELECT relname, n_live_tup, n_dead_tup, last_autovacuum
FROM pg_stat_user_tables ORDER BY n_dead_tup DESC LIMIT 10;

-- See transaction visibility horizons
SELECT * FROM pg_stat_activity WHERE state = 'active';

-- Inspect hidden columns
SELECT xmin, xmax, ctid, * FROM accounts;
```

MySQL InnoDB:
```sql
SHOW ENGINE INNODB STATUS;
SELECT * FROM information_schema.innodb_trx;
```

Monitoring: track `xact_commit`, `n_dead_tup`, autovacuum frequency, the oldest live snapshot.

---

## 13. Common Mistakes

- **Idle in transaction** sessions piling up → catastrophic bloat.
- **Disabling autovacuum** without a manual replacement → eventual outage.
- **Treating `count(*)` as cheap** on MVCC engines — Postgres has to scan to confirm visibility. Cache or use approximate counts.
- **Reading huge result sets in long transactions** (analytics on the OLTP). Use replicas.
- **Assuming Snapshot Isolation = Serializable**. It isn't; write skew is real.
- **Forgetting that updates create new rows in Postgres** — heavy update workloads + many indexes = write amplification.
- **Ignoring `txid wraparound`** monitoring. Test that autovacuum keeps `relfrozenxid` healthy.

---

## 14. Cheat Card

```
MVCC  = each transaction sees a consistent snapshot;
        every update creates a new row version with visibility metadata.

WIN   readers ↔ writers don't block each other.
COST  dead row versions → bloat → vacuum.

ISOLATION on top of MVCC
  Postgres READ COMMITTED         per-statement snapshot
  Postgres REPEATABLE READ        whole-transaction snapshot (Snapshot Isolation)
  Postgres SERIALIZABLE           SSI: SI + dependency tracking + abort on conflict
  MySQL InnoDB REPEATABLE READ    SI + gap locks → prevents phantoms

WHAT MVCC FIXES   dirty reads, non-repeatable reads, blocking.
WHAT IT DOESN'T   lost updates (app-side r-m-w), write skew (need SERIALIZABLE).

OPS
  Postgres: VACUUM / autovacuum / freezing / wraparound monitoring.
  MySQL:    undo log growth with long transactions.
  Watch for: idle-in-transaction sessions, heavy update workloads with many indexes.

NoSQL/DISTRIBUTED
  Cassandra: last-write-wins by timestamp; tombstones.
  Cockroach/Spanner: versioned KV with snapshot reads.
  Dynamo / Mongo: conditional / version-based writes.

RULE
  Use the engine's snapshots. Don't fight them.
  Atomic SQL > read-modify-write in app.
  Short transactions. Always.
```

---

## 15. Resources

### Books
- *Designing Data-Intensive Applications* — Kleppmann (Ch. 7 — transactions, MVCC).
- *Database Internals* — Alex Petrov.
- *PostgreSQL Internals* — Hironobu Suzuki (free): <https://www.interdb.jp/pg/>
- *Transaction Processing: Concepts and Techniques* — Gray & Reuter.

### Articles
- "PostgreSQL MVCC concurrency" — official docs: <https://www.postgresql.org/docs/current/mvcc.html>
- "How Postgres makes transactions atomic" — Brandur Leach: <https://brandur.org/postgres-atomicity>
- "Inside PostgreSQL's MVCC" — Tomas Vondra blog posts.
- "Beware the read-modify-write hazard" — Marc Brooker / AWS Builders' Library.
- "Serializable Snapshot Isolation (SSI)" — Cahill et al., the academic paper.
- Jepsen analyses — <https://jepsen.io/>

### Documentation
- Postgres MVCC, vacuum, autovacuum.
- MySQL InnoDB undo log.
- CockroachDB MVCC: <https://www.cockroachlabs.com/docs/stable/architecture/storage-layer.html>
- Spanner external consistency.

### Videos
- Hussein Nasser MVCC deep dives — <https://www.youtube.com/@hnasr>
- ByteByteGo: "What is MVCC?" — <https://www.youtube.com/@ByteByteGo>
- CMU 15-445 lectures (Andy Pavlo) on concurrency control.
- Martin Kleppmann lectures — <https://www.youtube.com/@kleppmann>

### Tools
- `VACUUM (VERBOSE, ANALYZE)` and `auto_explain` in Postgres.
- **pgstattuple** extension — bloat introspection.
- `SHOW ENGINE INNODB STATUS` in MySQL.
- **pganalyze**, **Datadog DB monitoring**.

### Adjacent reading
- [Transactions & Isolation Levels](./transactions-isolation.md)
- [Concurrency Control](./concurrency-control.md)
- [Relational Databases Deep Dive](./relational-databases.md)
- [Replication](./replication.md)

---

*Previous:* [← Concurrency Control](./concurrency-control.md)  |  *Next:* [Replication →](./replication.md)
