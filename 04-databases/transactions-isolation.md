# Database Transactions & Isolation Levels

> **TL;DR** — A **transaction** is a unit of work that's **atomic** (all-or-nothing), **isolated** (concurrent transactions don't trip over each other), and **durable** once committed. The SQL standard defines four isolation levels — **READ UNCOMMITTED**, **READ COMMITTED**, **REPEATABLE READ**, **SERIALIZABLE** — that trade off concurrency anomalies (dirty reads, non-repeatable reads, phantoms, lost updates, write skew). Modern engines achieve isolation through **MVCC** + locks. Picking the right level — and writing transactions that are short, well-scoped, and idempotent — is one of the highest-leverage backend skills.

---

## 1. What a Transaction Actually Buys You

```sql
BEGIN;
  UPDATE accounts SET balance = balance - 100 WHERE id = 'A';
  UPDATE accounts SET balance = balance + 100 WHERE id = 'B';
COMMIT;
```

Four guarantees (the **ACID** ones):

- **Atomicity** — both updates apply or neither does.
- **Consistency** — constraints / FKs / app rules still hold at commit.
- **Isolation** — another transaction running at the same time doesn't see "A debited but B not yet credited."
- **Durability** — once `COMMIT` returns, surviving a power loss is the DB's problem.

The hard one is **isolation**. The DB lets you tune it.

See [ACID vs BASE](./acid-vs-base.md).

---

## 2. The Anomalies (the things isolation prevents)

When concurrent transactions run, certain bad things can happen:

| Anomaly | What it means |
| --- | --- |
| **Dirty read** | T1 reads a value T2 has written but not yet committed. T2 rolls back → T1 saw nonsense. |
| **Non-repeatable read** | T1 reads row X. T2 commits a change to X. T1 reads X again → different value. |
| **Phantom read** | T1 runs `SELECT WHERE …` getting N rows. T2 inserts a matching row & commits. T1 re-runs → N+1 rows. |
| **Lost update** | T1 and T2 both read X, modify it, write back. The last write wins; the other's change is silently lost. |
| **Write skew** | Two transactions read overlapping sets, each makes a local decision, and the combined effect violates a constraint that no single transaction broke. (E.g., "at least one doctor on call" — each sees one other on call, both go off.) |
| **Read skew** | T1 reads X and Y; between the two reads, T2 commits an update affecting both. T1 sees an inconsistent pair. |

Isolation levels are *exactly* about which of these the DB prevents.

---

## 3. The Four SQL Standard Isolation Levels

| Level | Dirty read | Non-repeatable read | Phantom | Lost update | Write skew |
| --- | --- | --- | --- | --- | --- |
| READ UNCOMMITTED | ⚠️ possible | ⚠️ | ⚠️ | ⚠️ | ⚠️ |
| READ COMMITTED   | ✅ no | ⚠️ | ⚠️ | ⚠️ | ⚠️ |
| REPEATABLE READ  | ✅ | ✅ | ⚠️ in spec (✅ in some impls) | ⚠️ (mitigated in Postgres SI) | ⚠️ |
| SERIALIZABLE     | ✅ | ✅ | ✅ | ✅ | ✅ |

Higher levels = more correctness, more contention, more aborts, lower throughput.

### What real engines actually default to
- **PostgreSQL** — **READ COMMITTED**.
- **MySQL InnoDB** — **REPEATABLE READ** (with extra MVCC tricks).
- **Oracle / SQL Server** — vary; mostly **READ COMMITTED**.
- **CockroachDB / Spanner** — **SERIALIZABLE** by default.

Important: the SQL standard is a **lower bound**. Real engines often give you stronger guarantees than the label suggests (Postgres "RR" is actually *Snapshot Isolation*, MySQL InnoDB RR adds gap locks to prevent phantoms). Always check your engine's docs.

---

## 4. How Isolation Is Implemented

Two big families:

### Locking
Acquire shared locks for reads, exclusive locks for writes. Hold until commit. Strict 2PL gives serializability.

- Simple model.
- High contention; readers can block writers and vice versa.
- Older engines (early SQL Server, DB2 default).

### MVCC (Multi-Version Concurrency Control)
Each transaction sees a **snapshot** of the DB taken at start (or per statement). Writers create new row versions; readers never wait.

- Postgres, MySQL InnoDB, Oracle, SQL Server (under snapshot), CockroachDB, Spanner.
- Readers don't block writers; writers don't block readers.
- The default since the 2000s.
- Garbage collection of old row versions is now a real ops concern (Postgres `VACUUM`).

See [MVCC](./mvcc.md).

### Serializable Snapshot Isolation (SSI)
A clever scheme used by Postgres `SERIALIZABLE`: take a snapshot like SI, but track read/write dependencies; abort transactions that would produce a non-serializable history. Gives you true serializability without holding locks. Cost: occasional `serialization_failure` errors — your app must retry.

---

## 5. Reading the Docs Carefully

The same label means different things in different engines:

### Postgres
- `READ COMMITTED` — each *statement* sees committed data at the moment it starts.
- `REPEATABLE READ` — actually **snapshot isolation**. The whole transaction sees data as of its start. Doesn't see other commits afterward. Phantoms cannot occur via single-table queries; *write skew* can.
- `SERIALIZABLE` — adds SSI dependency tracking; aborts conflicting transactions.

### MySQL InnoDB
- `READ COMMITTED` — similar to Postgres.
- `REPEATABLE READ` (default) — snapshot isolation, plus **gap locks** on indexed `SELECT … WHERE` to prevent phantoms.
- `SERIALIZABLE` — additionally promotes plain `SELECT` to `SELECT … FOR SHARE`.

### Oracle
- Doesn't expose READ UNCOMMITTED.
- `READ COMMITTED` (default) — statement-level consistency.
- `SERIALIZABLE` — snapshot-based with conflict detection.

### Spanner / CockroachDB
- `SERIALIZABLE` is the only option (essentially). Strong external consistency.

---

## 6. Explicit Row Locking

When MVCC isn't enough, you can demand locks within a transaction:

```sql
BEGIN;
  SELECT * FROM accounts WHERE id = 'A' FOR UPDATE;
  -- ... compute ...
  UPDATE accounts SET balance = ... WHERE id = 'A';
COMMIT;
```

- `FOR UPDATE` — exclusive lock until commit; others wait.
- `FOR NO KEY UPDATE` — Postgres; lighter, doesn't block FK lookups.
- `FOR SHARE` — shared lock; multiple readers OK, writers blocked.
- `FOR UPDATE SKIP LOCKED` — magic for **work-queue** patterns. Pick the next available row, skip locked ones.

Use them when you need *application-level* exclusivity that the isolation level doesn't already give you (e.g., decrementing inventory).

---

## 7. The Lost-Update Trap

Without help, this code is broken:
```python
# T1
acc = db.fetch("balance from accounts where id='A'")  # 100
acc = acc - 100
db.write("update accounts set balance=$1 where id='A'", acc)  # 0

# T2 concurrently
acc = db.fetch("...")  # 100
acc = acc - 50
db.write("update accounts set balance=$1 ...", acc)  # 50
```

Both committed; the second silently overwrote the first. **Net change: only -50 applied**, not -150.

Fixes:
- **Compute server-side**: `UPDATE accounts SET balance = balance - 100 WHERE id='A'` — atomic.
- **Pessimistic lock**: `SELECT ... FOR UPDATE` then update.
- **Optimistic concurrency**: store a version; `UPDATE ... WHERE id='A' AND version=$old` and check rows-affected.
- **Higher isolation**: `REPEATABLE READ` or `SERIALIZABLE` detects this.

See [Concurrency Control](./concurrency-control.md).

---

## 8. Write Skew and Phantoms — The Subtle Ones

Lost updates are easy to spot. Write skew is invisible until it bites you.

```
Rule: at least one doctor must be on call.

Doctors on call: [A, B].

T1: read on_call=[A,B]; decide A can go off; UPDATE A SET on_call=false;
T2: read on_call=[A,B]; decide B can go off; UPDATE B SET on_call=false;
COMMIT both.

Now zero doctors are on call. Each transaction was correct on its own snapshot.
```

`SERIALIZABLE` prevents this. `READ COMMITTED` / Snapshot Isolation does **not**.

If your business invariants depend on **the set of rows other transactions are reading**, you need either explicit locks (`SELECT ... FOR UPDATE` on the materialized set), `SERIALIZABLE`, or a centralizing pattern (e.g., one row that everyone updates so conflicts are direct).

---

## 9. Practical Default: READ COMMITTED + Smart Patterns

For most applications:

1. **Default to READ COMMITTED.** Cheap, fast, sufficient for the majority of code.
2. **Make writes idempotent.** Most retries are safe by construction.
3. **Use atomic UPDATE expressions** (`SET x = x + 1`) instead of read-modify-write.
4. **Add `FOR UPDATE`** on the specific rows where exclusivity matters.
5. **Use SERIALIZABLE** for code that mints money / inventory / unique IDs; tolerate occasional retry.
6. **Add a `version` column** + optimistic checks for user-facing edits.

This combination handles 95% of OLTP correctly.

---

## 10. Short, Focused Transactions

Long transactions are the source of most operational pain:
- Hold locks → block other transactions.
- Hold MVCC snapshots → bloat old row versions, slow vacuum.
- Vulnerable to deadlocks and timeout aborts.
- Hurt latency for everyone else.

Rules:
- **Don't open a transaction** and then do network calls.
- **Don't sleep / await long operations** inside a transaction.
- **Commit fast**; reopen if you need more steps.
- **Push business logic** that does I/O *outside* the transaction.
- **Batch only what's small**.

A transaction that's open for 100 ms in a busy DB is too long. Aim for < 10 ms in hot paths.

---

## 11. Deadlocks

Two transactions waiting on each other's locks → deadlock. The DB detects and kills one (you see `deadlock_detected` / similar). Your app must retry.

Reduce deadlocks by:
- Always acquiring locks in the **same order** across the codebase (e.g., always update lower-ID account first).
- Keeping transactions short.
- Using indexes so locks are narrow (no full-table locking surprise).
- `SELECT … FOR UPDATE` only the rows you need.

Deadlocks aren't bugs in the DB. They're bugs in lock ordering.

---

## 12. Transactions in Distributed Systems

When data lives across shards or services, single-DB transactions don't help.

- **Distributed transactions (2PC / 3PC)** — possible (CockroachDB, Spanner, FoundationDB do this internally), but expensive. Avoid across services.
- **Sagas** — long-running coordinated workflows of local transactions, with compensations. See [Saga Pattern](../07-messaging/saga-pattern.md).
- **Outbox pattern** — atomically write data + event to one DB; publish event from outbox to a queue. See [Outbox](../07-messaging/outbox-pattern.md).
- **Idempotent service calls** — see [Idempotency](../03-apis/idempotency.md).

The classic mistake: "we'll wrap the order service and payment service in a distributed transaction." You won't; you'll use a saga.

---

## 13. Long-Running Operations

If you must do something that takes seconds (export a CSV, generate a PDF):
- **Don't wrap it** in a DB transaction.
- Begin only the *small* DB writes; do the heavy work outside.
- If you must hold long-running state, model it as **rows** (a "job" row that progresses through statuses).
- Use **advisory locks** (Postgres) if you need cluster-wide exclusivity *without* table locks.

---

## 14. Common Mistakes

- Bumping isolation to SERIALIZABLE without handling **serialization_failure** retries.
- Mixing read-only and read-write in the same transaction, holding snapshots open forever.
- Reading, modifying, writing in app code (lost updates).
- Long transactions full of network calls.
- Treating "REPEATABLE READ" the same across engines (it isn't).
- Forgetting that auto-commit mode is *implicit transactions* — every statement is its own transaction.
- Holding `FOR UPDATE` longer than needed.
- No retry on serialization errors → users see 500s.
- Assuming foreign-key checks are free (they take locks).

---

## 15. Diagnosing Concurrency Issues

In Postgres:
- `pg_stat_activity` — currently running queries.
- `pg_locks` — what locks each transaction holds / waits for.
- `pg_stat_database` — `deadlocks`, `conflicts`, `temp_files`.
- `log_lock_waits`, `log_min_duration_statement` — set these in `postgresql.conf`.
- `EXPLAIN ANALYZE` to ensure indexes minimize lock scope.

In MySQL: `SHOW ENGINE INNODB STATUS`, `information_schema.innodb_locks`, `performance_schema.threads`.

Tools: **pganalyze**, **Datadog DB Monitoring**, **percona toolkit**.

---

## 16. Cheat Card

```
ACID GUARANTEES   Atomicity · Consistency · Isolation · Durability

ISOLATION LEVELS  (SQL spec)
  READ UNCOMMITTED  — pretty much never used.
  READ COMMITTED    — default in Postgres / Oracle. Cheap, sufficient often.
  REPEATABLE READ   — snapshot isolation in PG; default in MySQL InnoDB.
  SERIALIZABLE      — strongest. Postgres uses SSI; expect retries.

ANOMALIES
  dirty · non-repeatable · phantom · lost update · write skew · read skew

DEFAULTS BY ENGINE
  Postgres / Oracle  → READ COMMITTED
  MySQL InnoDB       → REPEATABLE READ
  CockroachDB / Spanner → SERIALIZABLE

LOST UPDATE FIX
  Atomic SQL (SET x = x - 1)
  FOR UPDATE row lock
  Optimistic version column
  SERIALIZABLE (with retry on conflict)

WRITE SKEW FIX
  SERIALIZABLE, or
  SELECT FOR UPDATE on the materialized set, or
  centralize the invariant in one row.

RULES OF THUMB
  Keep transactions SHORT (< 10 ms in hot paths).
  No network calls inside a transaction.
  Make writes idempotent.
  Acquire locks in CONSISTENT ORDER (deadlock prevention).
  Always retry on `serialization_failure` / `deadlock_detected`.

DISTRIBUTED
  No 2PC across services. Use SAGA + Outbox + Idempotency.
```

---

## 17. Resources

### Books
- *Designing Data-Intensive Applications* — Kleppmann (Ch. 7, transactions). The clearest treatment in print.
- *Database Internals* — Alex Petrov.
- *Transactional Information Systems* — Weikum & Vossen (deep, academic).
- *PostgreSQL Internals* — Suzuki (free online): <https://www.interdb.jp/pg/>

### Articles
- "A Critique of ANSI SQL Isolation Levels" — Berenson et al., 1995. The paper that exposed the spec's ambiguities: <https://www.microsoft.com/en-us/research/publication/a-critique-of-ansi-sql-isolation-levels/>
- "How Postgres makes transactions atomic" — Brandur Leach: <https://brandur.org/postgres-atomicity>
- "Serializable Snapshot Isolation" — Cahill, Röhm, Fekete paper.
- "Highly Available Transactions: Virtues and Limitations" — Bailis et al.
- Jepsen analyses — Kyle Kingsbury's deep tests of real DBs: <https://jepsen.io/>

### Documentation
- **Postgres concurrency control** — <https://www.postgresql.org/docs/current/mvcc.html>
- **MySQL InnoDB transaction model** — <https://dev.mysql.com/doc/refman/en/innodb-transaction-model.html>
- **CockroachDB transactions** — <https://www.cockroachlabs.com/docs/stable/transactions.html>
- **Spanner external consistency** — <https://cloud.google.com/spanner/docs/true-time-external-consistency>

### Videos
- Hussein Nasser transaction series — <https://www.youtube.com/@hnasr>
- ByteByteGo: "Database transactions and isolation levels" — <https://www.youtube.com/@ByteByteGo>
- Martin Kleppmann's transactions lecture — <https://www.youtube.com/@kleppmann>
- CMU 15-445 transactions section — <https://15445.courses.cs.cmu.edu/>

### Tools
- `pg_locks`, `pg_stat_activity`, `pg_stat_statements`.
- **Jepsen test suites** — see how vendors fare under chaos.

### Adjacent reading
- [ACID vs BASE](./acid-vs-base.md)
- [MVCC](./mvcc.md)
- [Concurrency Control (Optimistic vs Pessimistic)](./concurrency-control.md)
- [Saga Pattern →](../07-messaging/saga-pattern.md)
- [Outbox Pattern →](../07-messaging/outbox-pattern.md)
- [Idempotency](../03-apis/idempotency.md)

---

*Previous:* [← Database Normalization & Denormalization](./normalization.md)  |  *Next:* [Concurrency Control →](./concurrency-control.md)
