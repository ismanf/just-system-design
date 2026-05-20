# Concurrency Control (Optimistic vs Pessimistic Locking)

> **TL;DR** — Two strategies to keep concurrent transactions from corrupting shared data:
> - **Pessimistic locking** — *acquire a lock first*; others wait. Safe under contention; can deadlock and limit throughput.
> - **Optimistic concurrency control (OCC)** — *don't lock*; on commit, check that nothing changed since you read; retry on conflict. Great under low contention; falls apart under high contention.
>
> Modern engines use **MVCC** which gives you OCC-like behavior for reads for free; explicit row locks (`SELECT FOR UPDATE`) and version columns add explicit pessimistic/optimistic semantics on top. The choice isn't "one or the other" — it's *which pattern fits this exact piece of state*.

---

## 1. The Problem They Solve

When two transactions touch the same data, three things can go wrong:

- **Lost update** — Both read X, both modify locally, both write back. One write silently wins.
- **Write skew** — Both read a set, both decide independently, the combined effect breaks an invariant.
- **Dirty read / inconsistent state** — One transaction sees half of another's changes.

Concurrency control is the broader name for everything that prevents these — including isolation levels, locking, MVCC, OCC, and consensus.

---

## 2. Pessimistic Locking

> *"Assume conflicts will happen. Lock first."*

```sql
BEGIN;
  SELECT * FROM accounts WHERE id = 'A' FOR UPDATE;  -- ← lock the row
  -- (now no other tx can update / lock row 'A')
  UPDATE accounts SET balance = balance - 100 WHERE id = 'A';
COMMIT;
```

`FOR UPDATE` takes an **exclusive lock** on the row(s) until the transaction commits or rolls back. Concurrent transactions wanting to read-or-write the same row **wait**.

### Variants
| SQL | Meaning |
| --- | --- |
| `SELECT … FOR UPDATE` | Exclusive lock; blocks other reads-for-update and writes. |
| `SELECT … FOR NO KEY UPDATE` (PG) | Lighter; doesn't block FK lookups. |
| `SELECT … FOR SHARE` | Shared lock; multiple readers OK, writers blocked. |
| `SELECT … FOR UPDATE NOWAIT` | Fail immediately if the row is locked. |
| `SELECT … FOR UPDATE SKIP LOCKED` | Skip already-locked rows. Brilliant for **work queues**. |

### Pros
- **Correct by construction** — once you hold the lock, you know nothing will change underneath you.
- **Predictable** — no retries, no surprise aborts.
- **Easy to reason about** — single linear sequence of events.

### Cons
- **Throughput suffers under contention** — every conflicting transaction waits.
- **Deadlocks possible** — two transactions waiting on each other's locks; the DB kills one.
- **Long transactions = long-held locks** — a slow downstream call inside a transaction blocks everyone.
- **Hard to scale across services** — distributed locks are their own beast.

### When to use it
- Operations where contention is *common* (a popular row, hot inventory item, account balance).
- Operations where retries are expensive (sending email, charging cards).
- Work-queue dispatch (`FOR UPDATE SKIP LOCKED`).
- Anywhere correctness > throughput.

---

## 3. Optimistic Concurrency Control (OCC)

> *"Assume conflicts are rare. Detect them at commit and retry."*

The pattern with a version column:

```sql
-- Read
SELECT balance, version FROM accounts WHERE id = 'A';
-- balance = 500, version = 17

-- Compute new value in app
new_balance = 500 - 100

-- Write only if no one else has touched the row
UPDATE accounts
   SET balance = $1, version = version + 1
 WHERE id = 'A' AND version = 17;
-- check rows_affected:
--   1 → success
--   0 → conflict; re-read, retry
```

Same pattern with a `last_updated` timestamp or with an `ETag`-style condition. HTTP-level: `If-Match: "v17"` → server enforces.

### Pros
- **No locks held** during the read/think/write window. Other transactions can keep working.
- **High throughput** under low contention.
- **No deadlocks** (no locks!).
- **Composes well** across services and HTTP.

### Cons
- **Retries** under contention — wasted work that piles up. Hot rows under high write rate are *terrible* for OCC.
- **Caller must handle conflict** — retry logic, idempotency, max retry caps.
- **Subtle bugs** if the version check is forgotten.

### When to use it
- User-edited records (low rate of concurrent edits per row).
- Web APIs with optimistic-update semantics (`If-Match`).
- Caches and BFFs.
- Distributed systems where holding a lock is impractical.
- Most "save my profile" / "save my draft" use cases.

---

## 4. MVCC — The Hybrid You Get For Free

Most modern engines (Postgres, MySQL InnoDB, Oracle, CockroachDB, Spanner) use **MVCC**. Every write creates a new row version; every transaction reads from a consistent snapshot.

- **Readers never block writers; writers never block readers.**
- For most workloads this is "free" — you didn't ask for it; it just happens.
- Lost updates are *still* possible if you read-modify-write outside the DB; MVCC doesn't fix that.
- `SERIALIZABLE` in Postgres builds on MVCC + dependency tracking (SSI) to catch the remaining anomalies — at the cost of occasional `serialization_failure` aborts (which is essentially OCC at the engine level).

See [MVCC](./mvcc.md).

---

## 5. Concrete Comparison

| | Pessimistic | Optimistic | MVCC |
| --- | --- | --- | --- |
| Holds locks during read/think | ✅ | ❌ | ❌ |
| Other tx wait | ✅ | ❌ | ❌ |
| Retry on commit conflict | ❌ | ✅ | sometimes (SSI) |
| Throughput under low contention | OK | High | High |
| Throughput under high contention | OK (queued) | Bad (retry storm) | OK |
| Deadlocks possible | ✅ | ❌ | ✅ (with locks) |
| Easy to reason about | ✅ | Mostly | Mostly |
| Cross-service | Hard | Easy | n/a |

A useful heuristic:

- **Hot key with high write rate** → pessimistic. (Don't retry-storm an inventory counter.)
- **Editable user record** → optimistic with version column.
- **Bulk dispatch queue** → pessimistic with `SKIP LOCKED`.
- **Multi-row invariant** → `SERIALIZABLE` (or `FOR UPDATE` on the materialized set).
- **Cache → DB write** → optimistic; cache may be stale; reconcile via OCC.

---

## 6. The Patterns You'll Use Most

### Pattern A: Atomic UPDATE expression (the cleanest)
```sql
UPDATE inventory
   SET qty = qty - 1
 WHERE sku = $1 AND qty > 0;
```
Single SQL statement; the DB does the math atomically. **Inventory decrement done right.** No app-side read-modify-write. If `rows_affected = 0`, the item was out of stock.

### Pattern B: Optimistic version column
```sql
UPDATE users SET email = $1, version = version + 1
 WHERE id = $2 AND version = $3;
```
Returns rows_affected. App retries on 0.

### Pattern C: Pessimistic lock for a hot row
```sql
BEGIN;
  SELECT * FROM seats WHERE id = $1 FOR UPDATE;
  -- check availability, decide, etc.
  INSERT INTO bookings ...
COMMIT;
```

### Pattern D: Work queue with SKIP LOCKED
```sql
SELECT id, payload
  FROM jobs
 WHERE status = 'pending'
 ORDER BY created_at
 LIMIT 10
 FOR UPDATE SKIP LOCKED;

-- mark them in progress in the same transaction
UPDATE jobs SET status = 'running' WHERE id = ANY($ids);
COMMIT;
```
Multiple workers grab disjoint batches without colliding. Beautiful.

### Pattern E: Advisory locks (Postgres)
```sql
SELECT pg_advisory_lock(hashtext('migrate-users'));
-- only one process holds this lock cluster-wide
SELECT pg_advisory_unlock(hashtext('migrate-users'));
```
For *application-level* exclusivity that doesn't map to a row. Useful for distributed cron jobs, migrations, leader election within an app.

### Pattern F: Compare-and-swap / conditional write
```python
# Redis
ok = redis.set("flag", "new", xx=True, ex=60)   # only if exists
```
NoSQL stores expose similar primitives. Same OCC mental model.

---

## 7. Deadlocks: Causes and Prevention

A deadlock = two transactions waiting on each other's locks forever. The DB detects and kills one.

**Top causes:**
- Different access order. T1 locks A then B; T2 locks B then A. → deadlock.
- Many small `SELECT … FOR UPDATE` calls inside one transaction.
- Implicit foreign-key locks at child-row inserts.
- Index range locks (MySQL InnoDB gap locks especially).

**Prevention:**
- **Always lock rows in the same order** across the entire codebase. (Sort IDs ascending before locking.)
- **Short transactions**. Less time to deadlock.
- **Touch fewer rows per transaction**.
- **Retry the killed transaction** — that's what your app must do; the DB will not do it for you.
- **Use timeouts** (`SET lock_timeout = '2s'`) to fail fast.

You'll never eliminate deadlocks completely; design for the retry.

---

## 8. Distributed Locks (Different Beast)

Locks across services need a coordinator. Options:

- **Redis Redlock** — `SET key val NX EX ttl`. Single-node is usually enough; Redlock attempts multi-node correctness but is debated.
- **ZooKeeper / etcd** — strongly-consistent ephemeral nodes. The right tool for leader election, distributed cron.
- **DB advisory locks** — Postgres advisory locks for "cluster-wide" exclusivity if everyone shares the DB.
- **Database row** — `INSERT … ON CONFLICT DO NOTHING RETURNING` works as a lightweight lease.

Gotchas:
- **Always set a TTL** — a process holding a lock can die.
- **Fencing tokens** — give each lock-holder a monotonically increasing token; later writers must include it. Prevents zombie process from corrupting after its lock expired.

See [Distributed Locks](../08-distributed-systems/distributed-locks.md).

---

## 9. OCC and Idempotency Are Friends

OCC + idempotent operations is a powerful combo:
- The client retries because conflicts may happen.
- The server treats each (idempotency-key, request) as idempotent.
- Conflicts that lose retry without double-applying changes.

This is why credit-card APIs, e-commerce checkouts, and signup flows usually use **OCC for state changes** + **idempotency keys** for retries.

See [Idempotency](../03-apis/idempotency.md).

---

## 10. Common Mistakes

- **Reading, computing, writing in app code** without `FOR UPDATE` or a version check → silent lost updates.
- **Holding locks during network calls** → cascades into outages.
- **Not handling `serialization_failure` retries** under `SERIALIZABLE` or OCC → 500s.
- **Lock ordering inconsistencies** → frequent deadlocks.
- **Polling a work queue without `SKIP LOCKED`** → contention and duplicate work.
- **Using OCC on a hot key** → retry storm; switch to pessimistic for that row.
- **Distributed locks without TTLs / fencing** → zombie holders corrupt data.
- **Forgetting MVCC means *reads* are free** — putting `FOR SHARE` everywhere "just in case" hurts more than it helps.

---

## 11. Diagnosing Concurrency Problems

Symptoms → causes:

| Symptom | Likely cause |
| --- | --- |
| Random lost updates | App-side read-modify-write without locking or version |
| Deadlock errors | Inconsistent lock order |
| Bursts of `serialization_failure` | `SERIALIZABLE` under contention; you must retry |
| Slow transactions piling up | Long-held locks; transactions doing I/O while open |
| Throughput plateau under load | Hot-row contention; consider sharding or pessimistic queue |
| Double-charging customers | Missing idempotency / version checks |
| Duplicate work in queue | No `SKIP LOCKED` / no claim mechanism |
| Phantom rows in a check | Need `SERIALIZABLE` or set-level lock |

Tools: `pg_locks`, `pg_stat_activity`, `SHOW ENGINE INNODB STATUS`, `log_lock_waits`.

---

## 12. Cheat Card

```
PESSIMISTIC  lock first. SELECT … FOR UPDATE / FOR SHARE / SKIP LOCKED.
  ★ hot rows, queues, correctness-first paths.
  ✗ deadlocks · long-held locks block everyone.

OPTIMISTIC   no lock; version-check on commit; retry on conflict.
  ★ user edits, web APIs (If-Match), cross-service.
  ✗ retry-storm under contention.

MVCC         engine gives you snapshot reads for free.
              SSI (Postgres SERIALIZABLE) = engine-level OCC.

PATTERNS
  atomic SQL  →  UPDATE … SET x = x - 1 WHERE id = $1.
  version OCC →  UPDATE … SET …, version = version+1 WHERE id=$1 AND version=$old.
  pessimistic →  SELECT … FOR UPDATE.
  queue       →  FOR UPDATE SKIP LOCKED.
  cluster lock →  pg_advisory_lock / etcd / ZooKeeper with TTL + fencing.

DEADLOCKS
  fix lock order. keep tx short. retry on the error code.

DISTRIBUTED
  TTL on every lock. Fencing tokens. Heart-beats. Beware zombies.

WITH IDEMPOTENCY
  OCC + idempotency keys = safe retries end-to-end.
```

---

## 13. Resources

### Books
- *Designing Data-Intensive Applications* — Kleppmann (Ch. 7 — transactions, Ch. 11 — stream processing).
- *Database Internals* — Alex Petrov.
- *Transaction Processing: Concepts and Techniques* — Jim Gray & Andreas Reuter (the canonical reference).

### Articles
- "Optimistic vs Pessimistic Locking" — many engineering blog posts.
- "A Critique of ANSI SQL Isolation Levels" — Berenson et al., 1995.
- "How Postgres makes transactions atomic" — Brandur Leach.
- "Solving the Lost Update Problem" — Vertabelo blog.
- "FOR UPDATE SKIP LOCKED in PostgreSQL" — Crunchy Data / EnterpriseDB blogs.
- "The trouble with distributed locks" — Martin Kleppmann (counter-Redlock): <https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html>

### Documentation
- **PostgreSQL Explicit Locking** — <https://www.postgresql.org/docs/current/explicit-locking.html>
- **MySQL InnoDB Locks** — <https://dev.mysql.com/doc/refman/en/innodb-locking.html>
- **Spanner / CockroachDB locking** — vendor docs.
- **etcd lock recipe** — <https://etcd.io/docs/v3.5/learning/lock/>
- **ZooKeeper recipes** — <https://zookeeper.apache.org/doc/current/recipes.html>

### Videos
- ByteByteGo: "Optimistic vs Pessimistic Locking" — <https://www.youtube.com/@ByteByteGo>
- Hussein Nasser locking deep dives — <https://www.youtube.com/@hnasr>
- CMU 15-445 concurrency lectures.

### Adjacent reading
- [Transactions & Isolation Levels](./transactions-isolation.md)
- [MVCC](./mvcc.md)
- [Distributed Locks](../08-distributed-systems/distributed-locks.md)
- [Idempotency](../03-apis/idempotency.md)
- [Saga Pattern](../07-messaging/saga-pattern.md)

---

*Previous:* [← Transactions & Isolation Levels](./transactions-isolation.md)  |  *Next:* [MVCC →](./mvcc.md)
