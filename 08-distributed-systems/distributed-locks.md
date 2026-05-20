# Distributed Locks (Redlock, ZooKeeper, etcd)

> **TL;DR** — A **distributed lock** ensures that only one process at a time holds exclusive access to a resource across a distributed system. The classic implementations: **ZooKeeper / etcd** (consensus-backed; correct and the safer choice for "must not fail" cases), **Postgres advisory locks** (transactional, surprisingly powerful for non-distributed cases), and **Redis-based locks** (fast, simple, with **Redlock** as the multi-instance variant). Distributed locks are famously hard to get right because of **clock drift**, **GC pauses**, **network partitions**, and **dual leaders**. The canonical fix is **fencing tokens** — a monotonically increasing number presented on every operation so the resource can reject stale lock holders. Use distributed locks sparingly: prefer **single-writer leader election** when possible, **optimistic concurrency** when correctness allows, and **idempotency** to make double-execution harmless. When you do need a lock, ZooKeeper / etcd-backed locks with proper fencing are the boring correct answer.

---

## 1. Why Distributed Locks Exist

A single-machine `mutex` works because the OS scheduler is the arbiter. In a distributed system, no shared scheduler — and the participants might:

- Be on different machines.
- Have arbitrarily skewed clocks.
- Get GC-paused.
- Become unreachable mid-operation.
- Be killed and replaced.

You still want: "only one client holds the lock at a time."

The lock prevents:
- Concurrent updates to a shared resource (double-charge, double-email).
- Multiple workers doing the same job.
- Multiple leaders.
- Conflicting writes to non-transactional stores.

---

## 2. What a Distributed Lock Must Promise

Two properties:
- **Mutual exclusion (safety)** — at most one client holds the lock at any time.
- **Deadlock freedom (liveness)** — the lock is eventually released (timeouts, lease expiry).

And the famous third (often overlooked):
- **Fault tolerance** — the system survives node failures.

Plus the practical:
- **Bounded staleness** — if a client crashes holding the lock, others can take it within a known time.

---

## 3. Implementations

### 3.1 Redis SETNX with TTL (single-instance)

```redis
SET resource:lock <unique_token> NX PX 30000
```

- `NX` = only if not exists.
- `PX 30000` = expire in 30 sec (auto-release if holder dies).
- `<unique_token>` = a random value the holder generated.

To release, use Lua to atomically check ownership:

```lua
if redis.call("GET", KEYS[1]) == ARGV[1] then
    return redis.call("DEL", KEYS[1])
end
return 0
```

**Pros**: very fast (~1 ms), simple, no dedicated infrastructure.

**Cons**: single Redis = SPOF. Failover loses unreplicated locks. **No fencing tokens** unless you add them. Famous critique by Martin Kleppmann: under GC pause + lease expiry, two clients can hold the lock simultaneously.

**Use for**: best-effort coordination (de-dupe background jobs, single-flight cache repopulation). NOT for safety-critical exclusivity.

### 3.2 Redlock (multi-instance Redis)

Designed by antirez (Redis creator) to address single-Redis SPOF. Acquire the lock across N (typically 5) independent Redis instances. Hold counts if majority acks within a time bound.

```
N=5 Redis instances
client tries SET on all 5 in parallel
counts successes within T_acquire
if successes >= 3 AND elapsed < lease_validity → got lock
otherwise → release all, fail
```

**Pros**: tolerates instance failures.

**Cons**: Kleppmann's critique stands — distributed locks fundamentally need consensus + fencing. Redlock makes assumptions about clocks and GC pauses that don't always hold. The Redis author (antirez) has rebutted, but the consensus view is: **for true safety, prefer consensus-backed locks (ZK/etcd) over Redlock.**

**Use for**: still best-effort + replicated, not for things where double-execution is catastrophic.

### 3.3 ZooKeeper Distributed Lock

ZooKeeper provides ephemeral sequential znodes that are perfect for locks.

```
client creates: /locks/resource/lock-<seq>
sees: /locks/resource/lock-0001, lock-0002, lock-0003
becomes leader if it has the lowest sequence number
otherwise watches the znode just below itself

if leader's session ends → znode disappears (ephemeral)
next-lowest becomes leader
```

**Pros**: provably correct under partition (CP). Sessions are explicit; ephemeral znodes auto-delete on session loss. Fencing via the znode sequence number.

**Cons**: ZooKeeper is a separate service to operate. Latency ~5–50 ms per op. Session timing is its own subtle business.

**Use for**: production-grade locks where correctness is critical. Kafka uses (used) ZK for partition leadership; Hadoop's YARN uses ZK; many enterprise systems.

### 3.4 etcd Distributed Lock

Similar to ZooKeeper. etcd's `lease` + `transaction` primitives let you build correct locks. Most clients (Go's `clientv3/concurrency`) wrap it as a clean API.

```go
session, _ := concurrency.NewSession(client)
mutex := concurrency.NewMutex(session, "/mylock")
mutex.Lock(ctx)
// critical section
mutex.Unlock(ctx)
```

**Pros**: same correctness as ZK. JSON API, gRPC. Standard for Kubernetes (which uses etcd).

**Cons**: same as ZK — needs the cluster.

**Use for**: Kubernetes-native systems, modern Go services.

### 3.5 Postgres Advisory Locks

A surprisingly good "distributed lock" if your processes all talk to one Postgres.

```sql
-- session-level lock
SELECT pg_advisory_lock(hash('resource_name'));
-- ... critical section ...
SELECT pg_advisory_unlock(hash('resource_name'));

-- transaction-level (auto-released on commit/rollback)
SELECT pg_advisory_xact_lock(hash('resource_name'));
```

**Pros**: zero new infrastructure. Transactional integration. Auto-release on disconnect.

**Cons**: requires that all participants connect to the same Postgres. Not truly "distributed" in the WAN sense.

**Use for**: services sharing a Postgres; quick coordination without standing up ZK/etcd.

### 3.6 Database Row Locks

`SELECT ... FOR UPDATE` is a lock. Works fine for short-lived locks against database resources.

```sql
BEGIN;
SELECT * FROM jobs WHERE id = 42 FOR UPDATE SKIP LOCKED;
-- do work
COMMIT;
```

`SKIP LOCKED` is great for work queues. The classic pattern for "claim a job, finish it, commit."

---

## 4. Comparison Table

| Implementation | Safety | Latency | Setup cost | Fencing built in | When to use |
|---|---|---|---|---|---|
| Redis SETNX | Weak | 1 ms | Trivial | No | Best-effort |
| Redlock | Medium (debated) | ~5 ms | Multi-Redis | No (need add) | Don't, in 2026 |
| ZooKeeper | Strong | 5–50 ms | Operate ZK cluster | Yes (sequence) | Production-grade |
| etcd | Strong | 5–50 ms | Operate etcd | Yes (revision) | Kubernetes ecosystem |
| Postgres advisory | Strong (single DB) | < 5 ms | None new | Implicit | Shared-DB coordination |
| Postgres row locks | Strong | < 5 ms | None new | Implicit | Resource-tied locks |

---

## 5. Fencing Tokens (Critical)

Without fencing, a distributed lock can have **two simultaneous holders** even in correctly-designed systems, because:
- Process A acquires the lock.
- A GC-pauses for 30 sec; lease expires.
- B acquires the lock.
- A wakes up; still thinks it's holding.

If A now writes to the resource, both A and B are writing. Disaster.

**Fencing fixes this**: every lock acquisition gets a **monotonically increasing token**. The protected resource checks the token on every operation:

```
A acquires lock → gets token 17
B acquires lock later → gets token 18

A's write to storage carries token 17
B's write carries token 18
storage tracks "latest accepted = 18"
A's request comes in: token 17 < 18 → REJECTED
```

The fencing token is the **only** thing that makes distributed locks truly safe.

- ZooKeeper: znode sequence number.
- etcd: revision number from the transaction.
- Redis: you must add this layer yourself (a counter incremented atomically; pass to resource).
- Postgres: usually implicit in transaction order.

If your lock implementation can't give you a fencing token and the resource doesn't validate it: **you don't have a safe lock**. You have a fast probabilistic mechanism.

---

## 6. The Kleppmann vs antirez Debate

Famous 2016 exchange:

- **Martin Kleppmann** argued Redlock isn't safe because GC pauses + clock skew can violate mutual exclusion. Concluded: for correctness, use ZooKeeper / etcd with fencing tokens. For best-effort, single-Redis is fine.
- **antirez (Redis author)** responded that Kleppmann's failure scenarios are extreme and Redlock works "in practice."

The consensus that emerged: **both are right within their respective scopes.** Redlock is fine when occasional double-acquire is acceptable; for true safety, use consensus + fencing.

Don't get into theological arguments — pick based on your correctness requirement.

---

## 7. Common Patterns

### 7.1 Singleton background job
```python
def hourly_job():
    if not acquire_lock("hourly_job", ttl=3600):
        return
    try:
        run_job()
    finally:
        release_lock("hourly_job")
```

Multiple workers; only one runs the job. If a worker dies, lock expires; another takes it next hour.

### 7.2 Single-flight cache populate
```python
def get_or_populate(key):
    v = cache.get(key)
    if v: return v
    if cache.set_nx(f"lock:{key}", 1, ex=10):
        try:
            v = expensive_load(key)
            cache.set(key, v, ex=300)
            return v
        finally:
            cache.delete(f"lock:{key}")
    else:
        # someone else is loading; wait and retry
        time.sleep(0.05)
        return get_or_populate(key)
```

Cache stampede protection. See [Cache Pitfalls →](../05-caching/cache-pitfalls.md).

### 7.3 Job queue claim
```sql
-- Postgres
BEGIN;
SELECT * FROM jobs WHERE status='pending'
  ORDER BY created_at
  LIMIT 1
  FOR UPDATE SKIP LOCKED;
UPDATE jobs SET status='processing' WHERE id=...;
COMMIT;
```

Multiple workers; each claims a different job atomically.

### 7.4 Resource exclusivity
A user is editing a document; lock it.
- Acquire lock with TTL = expected edit duration.
- Renew (heartbeat) while editing.
- Release on save.

Used by Google Docs (with finer-grained operational transformation underneath).

### 7.5 Leader election (degenerate case)
A leader is effectively a long-lived lock holder. See [Leader Election →](./leader-election.md).

---

## 8. Avoiding Distributed Locks

Distributed locks are expensive and bug-prone. Often you can avoid them:

### 8.1 Idempotency
Make the operation safe to run twice. Then locking isn't required for correctness — only efficiency.

```python
# bad: lock to prevent double-charge
acquire_lock(charge_id)
charge_card()
release_lock()

# good: pass idempotency key; Stripe dedupes
charge_card(idempotency_key=charge_id)
```

See [Idempotency →](../03-apis/idempotency.md).

### 8.2 Optimistic Concurrency Control
Read version, write with version-check, retry on conflict.

```sql
UPDATE accounts SET balance=80, version=5
WHERE id=42 AND version=4
-- if 0 rows affected, conflict; retry
```

No locks needed. Works for low-contention cases.

### 8.3 Conflict-free data types (CRDTs)
For some operations (counters, sets), CRDTs eliminate locks entirely. See [CRDTs →](./crdts.md).

### 8.4 Single-writer leader
One leader does the writes; followers forward. No lock — natural exclusivity. See [Leader Election →](./leader-election.md).

### 8.5 Partition the work
Shard by key. Each shard has one owner. No cross-shard contention. See [Sharding →](../04-databases/sharding-partitioning.md).

The takeaway: **the best distributed lock is the one you didn't need.**

---

## 9. Operational Concerns

### 9.1 Lock TTL tuning
- Too short: holder hasn't finished; lock auto-released; someone else takes it.
- Too long: holder dies, lock unavailable forever-ish.

Pick TTL > expected operation duration × 2. Renew (heartbeat) for long operations.

### 9.2 Renewal / heartbeat
A long-running operation should periodically extend its lease. Stop extending when done.

If the network glitches and renewal fails, the holder should **stop the operation** (it may have lost the lock).

### 9.3 Lock contention
If many clients fight for the same lock, throughput drops. Symptoms:
- High p99 latency.
- Many timeouts on acquire.
- One operation at a time vs. multi-threaded ideal.

Mitigations:
- Shard the lock by key (`lock:account:42` vs `lock:account:*`).
- Use optimistic CC instead.
- Use a different pattern (queue, leader, sharding).

### 9.4 Lock leakage
Holder dies without releasing → TTL expires eventually. But if you forgot the TTL, the lock is held forever. Always set a TTL.

### 9.5 Cross-region locks
Cross-region consensus is slow (50–200 ms per op). Locks across regions are usually a bad idea — better to scope to a region.

---

## 10. Worked Example: Process-Payment Lock

A worker processes pending payments. Each payment must be processed by exactly one worker, even if workers crash.

### Naive (race condition)
```python
payment = db.find_pending()
process(payment)
db.mark_done(payment)
```

Two workers can both grab the same payment.

### With row lock
```sql
SELECT * FROM payments WHERE status='pending'
  ORDER BY id
  LIMIT 1
  FOR UPDATE SKIP LOCKED;
```

Inside a transaction. Atomically claims the row. Other workers skip it.

### With distributed lock (different table)
```python
payment_id = peek_pending()  # outside any tx
if etcd.lock(f"payment:{payment_id}"):
    try:
        process(payment_id)
        db.mark_done(payment_id)
    finally:
        etcd.unlock(f"payment:{payment_id}")
```

Works if the DB isn't shared. The row-lock version is usually simpler if it is shared.

### With idempotency + no lock
```python
payment_id = peek_pending()
result = stripe.create_charge(idempotency_key=payment_id, ...)
db.mark_done(payment_id, result)
```

No lock needed; Stripe dedupes. The DB update at the end is idempotent (`mark_done` if not already).

---

## 11. Common Mistakes

- **No TTL.** Holder dies, lock held forever.
- **No fencing token.** Lock holder pauses → two writers.
- **TTL too short.** Operation aborted mid-flight.
- **Trying to use Redis as a "safe" lock for money.** Use ZK/etcd, or just idempotency.
- **Lock and modify on different stores without fencing.** Race.
- **Holding a lock across an RPC.** Lock duration dominated by network latency; contention amplified.
- **Forgetting to release on exception.** Use `try/finally` or `with`.
- **Renewal without "did I still hold it?" check.** Token / version verifies.
- **Reinventing distributed locks.** Use a library or proven primitive.
- **Distributed lock when idempotency would do.** Almost always the simpler answer.

---

## 12. Cheat Card

```
DISTRIBUTED LOCK    only one client at a time across the cluster

PROMISES
  mutual exclusion (safety) — at most one holder
  deadlock free (liveness) — eventually released
  fault tolerant — survive holder death

IMPLEMENTATIONS
  Redis SETNX       fast, simple, best-effort; NO safety guarantee
  Redlock           multi-Redis; debated safety
  ZooKeeper         CP, correct, fencing via sequence znodes
  etcd              CP, correct, fencing via revision
  Postgres advisory same-DB only; transactional
  Postgres row lock FOR UPDATE SKIP LOCKED; work queues

FENCING TOKEN       monotonically increasing; resource validates
                    MANDATORY for safety against GC pauses

PRACTICAL ORDER
  1. Idempotency (avoid the lock)
  2. Optimistic CC (version checks)
  3. Single-writer leader (one owner)
  4. Postgres advisory or row lock (if shared DB)
  5. etcd/ZooKeeper (cross-system, safety-critical)
  6. Redis (best-effort only)

TTL                 > 2× expected operation duration
RENEWAL             for long operations; verify token
RELEASE             in try/finally; idempotent

PITFALLS            no TTL, no fencing, holding across RPC,
                    Redlock for money, reinventing locks

RULE                Don't reach for a distributed lock first.
                    Reach for idempotency. Then optimism.
                    Then a real coordination primitive.
```

---

## 13. Resources

### Articles
- "How to do distributed locking" — Martin Kleppmann (the canonical critique).
- "Is Redlock safe?" — antirez (the rebuttal).
- "Note on distributed computing" — Waldo et al. (foundational).
- "Distributed locks with Redis" — Redis docs.

### Books
- *Designing Data-Intensive Applications* — Kleppmann.
- *Database Internals* — Petrov.

### Videos
- ByteByteGo — "Distributed Locks".
- Hussein Nasser — "Why distributed locks are hard".

### Documentation
- **etcd `clientv3/concurrency`**: <https://pkg.go.dev/go.etcd.io/etcd/client/v3/concurrency>
- **Apache Curator (ZooKeeper)** locks: <https://curator.apache.org/curator-recipes/index.html>
- **Postgres advisory locks**: <https://www.postgresql.org/docs/current/explicit-locking.html#ADVISORY-LOCKS>
- **Redis distributed locks**: <https://redis.io/docs/latest/develop/use/patterns/distributed-locks/>

### Tools
- **etcd** + concurrency package.
- **ZooKeeper** + Apache Curator.
- **Consul** sessions and locks.
- **Redis** + Redlock library (with caveats).
- **Postgres**.

### Adjacent reading
- [Consensus →](./consensus.md)
- [Leader Election →](./leader-election.md)
- [Split-Brain →](./split-brain.md)
- [Idempotency →](../03-apis/idempotency.md)
- [Concurrency Control →](../04-databases/concurrency-control.md)
- [Redis Deep Dive →](../05-caching/redis-deep-dive.md)
- [Cache Pitfalls →](../05-caching/cache-pitfalls.md)

---

*Previous:* [← Leader Election](./leader-election.md)  |  *Next:* [Clocks →](./clocks.md)
