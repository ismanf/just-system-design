# Two-Phase Commit (2PC) and Three-Phase Commit (3PC)

> **TL;DR** — **Two-Phase Commit (2PC)** is a protocol for committing a transaction across multiple participants atomically — all commit, or all abort. A **coordinator** runs two phases: **prepare** (ask everyone if they can commit) and **commit/abort** (tell everyone the outcome). 2PC is the canonical way distributed transactions work in DB-XA / JTA, but it's **fragile**: a coordinator crash leaves participants stuck holding locks indefinitely. **Three-Phase Commit (3PC)** adds an intermediate "pre-commit" phase that lets participants make progress without the coordinator, but **assumes synchronous networks** — an assumption rarely safe on the real internet. **In modern microservice architectures, 2PC and 3PC are largely abandoned** in favor of **sagas** (asynchronous, with compensations) for cross-service workflows, or single-system distributed transactions backed by **consensus** (Spanner, CockroachDB). Understand 2PC for the vocabulary; reach for sagas in practice.

---

## 1. The Problem

Multiple participants (databases, services) need to apply a change atomically. Either all of them succeed or none do.

```
Service A: deduct $100 from account_x
Service B: add $100 to account_y

If A succeeds and B fails → money lost.
If A fails and B succeeds → money created.
If both succeed → correct.
If both fail → correct (the user retries).
```

Single-database transactions solve this with ACID inside one process. Across multiple participants, you need a coordination protocol.

---

## 2. Two-Phase Commit (2PC)

A **coordinator** drives the protocol. Each participant is a resource manager (DB, service) capable of "preparing" a transaction.

### Phase 1: Prepare
```
Coordinator → all participants: "Can you commit?"
Each participant:
   - Performs the operation locally (without committing).
   - Writes to its WAL (durable).
   - Replies "Yes, prepared" or "No, abort".
```

A participant that says "prepared" promises:
- The change is durable in its WAL.
- It will not commit unilaterally.
- It will not abort unilaterally.
- It will wait for the coordinator's decision.

### Phase 2: Commit or Abort
```
If all said "prepared":
   Coordinator decides COMMIT.
   Coordinator → all: "Commit."
   Each participant commits and acks.

If any said "abort":
   Coordinator decides ABORT.
   Coordinator → all: "Abort."
   Each participant rolls back.
```

After acks, the transaction is fully committed (or aborted).

### Properties
- **Atomicity**: all commit or all abort.
- **Durability**: each participant has prepared state in WAL; coordinator's decision is also durable.
- **Blocking**: participants who said "prepared" hold locks until they hear from the coordinator.

---

## 3. The Famous 2PC Failure Mode

```
Coordinator → A: prepare → A prepares (locks resources)
Coordinator → B: prepare → B prepares (locks resources)
Coordinator → C: prepare → C prepares (locks resources)

A, B, C all say "prepared."

Coordinator writes "decision: commit" to its WAL.

Coordinator → A: commit (acked)
Coordinator CRASHES before telling B and C.
```

Now B and C are stuck: they prepared, can't unilaterally commit or abort. They hold locks until coordinator recovers.

This is the **2PC blocking problem**. Coordinator's WAL eventually allows recovery, but until then, B and C are paused. If coordinator never recovers (rare but real), operators must intervene.

**2PC is not partition-tolerant in the strong sense.** It's safe (no inconsistency) but not live (can block forever).

---

## 4. Three-Phase Commit (3PC)

Addresses 2PC's blocking by adding a phase between prepare and commit: **pre-commit**.

### Phases
1. **CanCommit?** — coordinator asks participants if they could commit. They reply Yes/No.
2. **PreCommit** — if everyone said Yes, coordinator sends "pre-commit." Participants ack. They're now in a state where they can decide on their own if the coordinator dies.
3. **DoCommit** — coordinator sends final commit. Participants commit.

### Key idea
If the coordinator crashes after step 2 (pre-commit acks received), participants can **timeout and self-commit**, since everyone agreed to pre-commit. No more indefinite blocking.

### Why 3PC isn't actually used
- It assumes a **synchronous network** with bounded message delays. The real internet doesn't provide this.
- Under network partitions or asymmetric failures, 3PC can produce incorrect outcomes.
- Complex to implement correctly.
- In practice, 2PC + a robust coordinator (replicated coordinator state, recovery procedures) is preferred.

3PC is mostly a textbook curiosity. Real systems use 2PC (when they use distributed transactions at all) or sagas.

---

## 5. Where 2PC Is Used

### XA transactions / JTA
The **X/Open XA** standard defines an interface for resource managers (DBs, message queues) to participate in 2PC. Java's JTA implements XA. JBoss / Tomcat / WebSphere transaction managers do XA.

Used in enterprise Java for spanning a DB + a message broker:
```java
@Transactional
void method() {
    db.update();
    queue.send();
    // both commit or both rollback as one XA tx
}
```

Heavy machinery. Slow. Limited adoption outside enterprise Java.

### Database internals
Some distributed databases use 2PC internally for cross-shard transactions:
- **CockroachDB**: 2PC over Raft groups.
- **TiDB**: 2PC variant (Percolator).
- **Spanner**: 2PC over Paxos groups.
- **VoltDB**: optimized cross-partition commits.

These wrap 2PC inside consensus, which mitigates the coordinator-failure problem (because the coordinator's state is replicated).

### File systems
Distributed file systems (HDFS, Lustre) use 2PC-like protocols for metadata changes.

### Microservices? Mostly no.
Theoretically, you can XA across services. In practice:
- Most services don't speak XA.
- 2PC's blocking + complexity is hard for microservices to absorb.
- Sagas are the modern alternative. See [Saga Pattern →](../07-messaging/saga-pattern.md).

---

## 6. 2PC vs Consensus vs Sagas

These three patterns serve different needs.

| Pattern | What it gives | What it costs |
|---|---|---|
| **2PC** | Atomic commit across participants | Blocking on coordinator failure |
| **Consensus (Raft/Paxos)** | Agreement on a sequence of values; replicated state machine | Latency of quorum |
| **Saga** | Atomic outcome across services via compensations | Eventual consistency; non-trivial compensations |

### When 2PC is the right answer
- You're inside one database / DBMS family that supports it (cross-shard transactions internally).
- All participants are reachable; coordinator is replicated.
- You're willing to accept the blocking failure mode.

### When consensus is the right answer
- You're building a replicated state machine.
- You need linearizable ordering of operations.
- Examples: etcd, Spanner, CockroachDB.

### When sagas are the right answer
- You're coordinating across services that have their own DBs.
- You can tolerate eventual consistency between steps.
- You can design compensations.

Most modern microservice flows are sagas. 2PC survives inside databases.

---

## 7. The Coordinator Problem

A 2PC coordinator is itself a single point of failure. Production deployments mitigate by:

- **Replicating the coordinator's state** via consensus. The coordinator's WAL is itself a Raft log. So when the coordinator "crashes," another instance picks up.
- **Persistent state with quick recovery**. Coordinator writes its decision durably before sending it; on restart, it can resume.
- **Heuristic completion** — if coordinator is unreachable for a long time, operators may force participants to commit or abort. Dangerous: can violate atomicity if you guess wrong.

A single-instance coordinator is the textbook 2PC's biggest flaw. Real implementations replicate.

---

## 8. Worked Example: Cross-Service Transfer

A bank transfers money: debit `account_x` (Service A) and credit `account_y` (Service B).

### Naive (broken)
```python
service_a.debit("x", 100)
service_b.credit("y", 100)
```

If `credit` fails, money is lost.

### 2PC version
```
coordinator:
  send prepare(debit x 100) to A
  send prepare(credit y 100) to B
  await both
  if both prepared:
     send commit to both
     await acks
  else:
     send abort to both
```

If A crashes after prepare: B is stuck holding the credit prepared.

If coordinator crashes after sending commit to A but before B: A committed, B prepared, no progress until coordinator recovers.

### Saga version
```
1. service_a.debit_async("x", 100, txn_id=T1)
2. on TxnDebited event:
   service_b.credit_async("y", 100, txn_id=T1)
3. on TxnCredited event:
   mark txn T1 complete

if step 2 fails (after step 1):
   service_a.compensate_debit("x", 100, txn_id=T1)
   mark txn T1 failed
```

Eventual consistency: there's a brief window where the debit is committed but the credit isn't. The user might see an intermediate state. Add UX honesty ("pending") to handle.

### Spanner-style
Spanner uses 2PC over Paxos groups internally. From the application's perspective:
```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 'x';
UPDATE accounts SET balance = balance + 100 WHERE id = 'y';
COMMIT;
```

Underneath, Spanner runs 2PC across shards, with Paxos replication for durability. The blocking problem is mitigated by replicated coordinators.

The shape varies. The trade-offs are real.

---

## 9. Practical 2PC Hardening

If you must use 2PC:

- **Replicate the coordinator's state.** Use Raft / Paxos for the coordinator's WAL.
- **Bound timeouts.** Participants and coordinator both should time out and not wait forever.
- **Heuristic completion procedures** for stuck transactions. Document them; rehearse them.
- **Monitor stuck transactions.** "How many transactions have been in prepared state for > 60 sec?" should be a dashboard.
- **Test partition scenarios.** Use chaos engineering to simulate coordinator failures.
- **Idempotency on every step.** A retry shouldn't be catastrophic.

---

## 10. Why Most Systems Don't Use 2PC for Microservices

Three reasons it's mostly avoided:

### 10.1 Latency
Each phase = network round trip. With participants in multiple AZs / regions: every transaction = several RTTs of latency. Often unacceptable.

### 10.2 Coupling
Every participant must implement the 2PC protocol (XA or equivalent). For microservices with diverse stacks, painful.

### 10.3 Blocking
The fundamental problem. Even with mitigations, you're one bad day from stuck transactions.

Saga compensations are easier than coping with stuck 2PC transactions in the long run.

---

## 11. Common Mistakes

- **Implementing 2PC from scratch.** Hard; the edge cases are subtle.
- **Single-instance coordinator.** SPOF. Replicate via consensus.
- **No timeout on participants.** Forever-stuck on coordinator failure.
- **Heuristic completion without operator playbook.** Wrong decisions cause inconsistency.
- **Using XA where you could use idempotency + retries.** Often the simpler answer.
- **Trying 3PC because "it's better."** It assumes synchronous networks, which the internet isn't. Mostly textbook.
- **Mixing 2PC and async retries.** Confusion about who's responsible for retrying.
- **Using 2PC for cross-service flows.** Sagas almost always better.

---

## 12. Decision Rules

```
Within one DB system (e.g., CockroachDB)?
  → Built-in 2PC (over consensus). Use the transaction API.

Cross-service workflow, eventual consistency OK?
  → Saga (orchestration or choreography).

Cross-service, need ACID-like?
  → Rethink. Often you can:
     - Combine services for that path.
     - Use an outbox + saga.
     - Accept eventual consistency with monitoring.

Legacy XA stack (Java EE)?
  → Use what's there; don't roll your own.

New design considering 2PC?
  → Strong default: NO. Use saga or built-in transactional DB.
```

---

## 13. Cheat Card

```
2PC          atomic commit across participants
              phase 1: prepare; phase 2: commit/abort

PROPERTIES   atomic (all or none), durable (WAL), BLOCKING

FAILURE MODE  coordinator crashes mid-commit
              participants stuck in "prepared" state holding locks

3PC          adds pre-commit phase; non-blocking in theory
              assumes synchronous network — unsafe in practice
              mostly textbook

USED BY      XA / JTA (enterprise Java)
              CockroachDB, TiDB, Spanner (over consensus)
              database internals

NOT USED FOR  microservice cross-service workflows
              use sagas instead

MODERN VIEW   2PC inside one DB system (over consensus): fine
              2PC across independent services: avoid

HARDENING    replicate coordinator via consensus
              bound timeouts
              monitor stuck transactions
              idempotent participants

PITFALLS     single-instance coordinator,
              no timeout, heuristic decisions without docs,
              implementing from scratch,
              rolling 3PC for internet networks

RULE         For distributed transactions across services,
              use sagas. For ACID within one DB system, let
              the DB handle 2PC over consensus internally.
```

---

## 14. Resources

### Papers
- "Notes on Data Base Operating Systems" — Jim Gray, 1978 (the foundational 2PC paper).
- "The Three-Phase Commit Protocol" — Dale Skeen, 1981.
- "Consensus on Transaction Commit" — Gray & Lamport, 2006 (Paxos-based commit).

### Books
- *Designing Data-Intensive Applications* — Kleppmann (Ch 9: Consistency and Consensus covers 2PC well).
- *Transaction Processing: Concepts and Techniques* — Gray & Reuter (the classic).

### Articles
- "Why don't more apps use 2PC?" — various engineering blogs.
- "Distributed transactions are not all the same" — Cockroach Labs.
- "Saga pattern" — Chris Richardson microservices.io.

### Videos
- ByteByteGo — "2PC and 3PC Explained".
- Martin Kleppmann — Distributed systems lectures.

### Documentation
- **X/Open XA**: <https://en.wikipedia.org/wiki/X/Open_XA> (still the spec).
- **JTA**: <https://docs.oracle.com/javaee/7/tutorial/transactions001.htm>
- **CockroachDB transactions**: <https://www.cockroachlabs.com/docs/stable/transactions.html>

### Adjacent reading
- [Consensus →](./consensus.md)
- [Quorum-Based Replication →](./quorum.md)
- [Saga Pattern →](../07-messaging/saga-pattern.md)
- [Outbox Pattern →](../07-messaging/outbox-pattern.md)
- [Database Transactions & Isolation Levels →](../04-databases/transactions-isolation.md)
- [Delivery Guarantees →](../07-messaging/delivery-guarantees.md)

---

*Previous:* [← Clocks](./clocks.md)  |  *Next:* [Quorum-Based Replication →](./quorum.md)
