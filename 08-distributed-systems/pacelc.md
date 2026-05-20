# PACELC Theorem

> **TL;DR** — **PACELC** extends [CAP →](./cap-theorem.md) with the trade-off that actually dominates daily operations: **latency vs consistency**, even when there is no partition. It says: **if Partitioned, choose Availability or Consistency; Else, choose Latency or Consistency.** PACELC gives every system a two-letter label like **PA/EL** (Cassandra: AP under partition, low latency normally) or **PC/EC** (Spanner: CP under partition, consistent normally with synchronous replication). CAP captures the rare partition case; PACELC captures the **other 99% of the time** — where adding synchronous replication, quorum reads, or cross-region replicas trades latency for stronger consistency. Daniel Abadi introduced it in 2012 because most engineering decisions are about latency, not partition behavior. Modern databases are best described in PACELC terms.

---

## 1. The Question CAP Doesn't Answer

CAP tells you what happens during a partition: choose C or A. But:

```
   99.99% of the time:    no partition       ← CAP says nothing
   0.01% of the time:     partition          ← CAP applies
```

In the 99.99% case, you still face a trade-off: **do you replicate synchronously (slower, more consistent) or asynchronously (faster, eventually consistent)?**

Synchronous replication = strong consistency = higher latency.
Asynchronous replication = lower latency = weaker consistency.

This is what dominates engineering decisions in practice. CAP is silent on it. **PACELC fills the gap.**

---

## 2. The Theorem

Daniel Abadi, 2010:

> **If there is a partition (P), how does the system trade off Availability and Consistency (A vs C); ELse (E), when the system is running normally, how does it trade off Latency and Consistency (L vs C)?**

Acronym: **P-A-C / E-L-C** → **PACELC**.

Every distributed system has a two-letter classification:

| Label | Meaning |
|---|---|
| **PA/EL** | Partition: choose Availability. Else: choose Latency. |
| **PA/EC** | Partition: A. Else: Consistency. |
| **PC/EL** | Partition: Consistency. Else: Latency. (Rare.) |
| **PC/EC** | Partition: Consistency. Else: Consistency. |

Most systems are **PA/EL** (Cassandra-like) or **PC/EC** (Spanner-like). The other combinations exist but are less common.

---

## 3. The Latency-Consistency Trade-off

In normal operation, to be strongly consistent across replicas, a write must wait for acknowledgement from enough replicas. That takes time — especially across AZs or regions.

```
ASYNC replication (PA/EL): write to primary, ack immediately, replicate in background
   write latency: 1 ms
   consistency: eventual (replicas lag)

SYNC replication (PC/EC): write to primary AND replicas, ack only when N of M confirm
   write latency: 5–50 ms (depending on N and geography)
   consistency: strong
```

If your writes happen 1000× per second and each adds 10 ms, you've added 10 seconds of cumulative latency. That trade matters.

PACELC names this trade-off explicitly.

---

## 4. Classifying Real Systems

### PA/EL — Always lean toward availability and latency

- **DynamoDB** (default eventually-consistent reads).
- **Cassandra** with default consistency levels.
- **Riak**.
- **Voldemort**.
- **CouchDB**.

These are "Dynamo-style" systems. Multi-master writes, async replication, eventual consistency by default. Maximize uptime and speed.

### PC/EC — Always lean toward consistency

- **Spanner** — synchronous Paxos globally.
- **CockroachDB** — Raft synchronous replication.
- **YugabyteDB**.
- **VoltDB**.
- **HBase** (kinda — see notes).
- **Postgres / MySQL primary** with synchronous standby.
- **etcd / Zookeeper**.

These are "consensus-style" systems. Quorum writes, synchronous consistency, accept higher latency.

### PA/EC — Available on partition, consistent in normal ops

- **MongoDB** (with `w=majority`) — partition: minority can keep serving stale reads (A); else: synchronous to majority (C).

Less common but real.

### PC/EL — Consistent under partition, latency-optimized normally

- A theoretical category; few clean examples. Some systems with weakly-consistent reads + strongly-consistent writes during partition fall here.

---

## 5. Why PACELC Beats CAP for Engineering

CAP forces a single label. PACELC names two situations separately, matching reality:

```
SCENARIO                   CAP says           PACELC says
write to global DB         AP or CP?          synchronous or async normally?

  partition happens        CP refuses         PC: refuse, preserve C
  no partition             ?                  EC: pay latency for C

scenario:                  CAP says           PACELC says
shopping cart writes       AP                 PA: stay available
  no partition             ?                  EL: low latency, eventually consistent
```

Every architecture decision is more meaningful when you name both halves.

---

## 6. Concrete PACELC Examples

### Cassandra (PA/EL)
- Partition: minority replicas keep serving with weakened consistency. Available.
- Normal: writes return after `ONE` ack by default. Replication async. Low latency.

To get EC behavior in Cassandra: use `QUORUM` or `ALL` consistency levels. You're trading latency for consistency on demand.

### Spanner (PC/EC)
- Partition: minority side can't reach quorum, refuses writes. Consistent.
- Normal: TrueTime + Paxos commit waits for majority across regions. Higher latency, strong consistency.

### DynamoDB (PA/EL by default)
- Partition: writes accepted; conflicts resolved later.
- Normal: writes ack after one replica's ack; reads eventually consistent.
- Option: strongly-consistent reads = EC for that read, with 2× cost.

### MongoDB with `w=majority`, `readConcern=majority` (PA/EC)
- Partition: minority can serve stale reads → available.
- Normal: writes wait for majority. Consistent in normal ops.

### Kafka
- `acks=all` + `min.insync.replicas=2`: PC/EC for those writes (refuse if can't ack quorum; wait for replicas normally).
- `acks=1`: PA/EL — faster, possible loss on broker failure.

PACELC isn't a fixed system property — it's per-config.

---

## 7. The Geography Dimension

Latency dominates PACELC most when replicas are far apart.

```
single-AZ replication      < 1 ms RTT          sync-vs-async ≈ trivial diff
single-region (cross-AZ)   ~ 1 ms RTT          minor
cross-region (US-EU)       ~ 80 ms RTT         major decision
global (worldwide)         100+ ms RTT         massive
```

A "PC/EC" system in one region is fine. The same system across continents adds 100+ ms per write. Spanner's "global ACID" comes with the round-trip cost; that's why Google built TrueTime and Paxos commit-waits — to hide it as much as possible.

This is the **reason** most large systems have a regional primary with async cross-region replicas. The PC/EC dream is great until the latency bill arrives.

---

## 8. PACELC as Architectural Lens

When making distributed-storage decisions, ask both questions:

### 8.1 "What happens under partition?"
- Refuse writes (PC) or accept them (PA)?
- Does the application tolerate stale reads during partition?
- How long is acceptable downtime vs how much divergence?

### 8.2 "What happens in normal ops?"
- Synchronous (EC) — higher latency, stronger consistency.
- Asynchronous (EL) — lower latency, eventual consistency.
- What latency does the user-facing path require?
- What consistency does the business logic require?

The answers can differ per operation:
- Reservation write: PC/EC (no choice).
- User profile read: PA/EL (cache is fine).
- Cart update: PA/EL (last-write-wins fine).
- Payment authorization: PC/EC.

Most systems are mixtures.

---

## 9. Tuning per Operation

Modern databases let you tune PACELC per query:

```sql
-- Cassandra: tune E side
SELECT * FROM users USING CONSISTENCY LOCAL_QUORUM;   -- EC-leaning
SELECT * FROM users USING CONSISTENCY ONE;             -- EL-leaning

-- MongoDB
db.users.find().readConcern("majority");        -- EC
db.users.find().readConcern("local");            -- EL

-- DynamoDB
get_item(consistent_read=True);    -- EC
get_item(consistent_read=False);    -- EL (default)
```

This is great. It means PACELC isn't a label you stamp on the DB; it's a knob you turn per call.

---

## 10. PACELC vs CAP Side-by-Side

| Aspect | CAP | PACELC |
|---|---|---|
| Captures partition behavior | Yes | Yes (PA / PC) |
| Captures normal-ops behavior | No | Yes (EL / EC) |
| Two-letter labels | C, A, P | PA/EL, PA/EC, PC/EL, PC/EC |
| Useful for engineering choices | Sometimes | Often |
| Granular | No | Better |
| Famous | Yes | Less so, but more correct |

PACELC is **more useful**. CAP is **more famous**. In conversation, use CAP for the framework, PACELC for the trade-off.

---

## 11. Worked Example: Multi-Region Database Choice

Goal: a global B2B SaaS, write latency target < 100 ms for 95% of writes, strong read-after-write for users.

### Option A: Spanner (PC/EC)
- Partition: refuses writes on minority. Acceptable; SLA forgives brief outages.
- Normal: synchronous quorum across continents. p99 write ~50–150 ms. Reads ~10–50 ms.
- Cost: $$$$.
- Operational simplicity: very high; managed.
- Consistency: strong, globally.

### Option B: Cassandra (PA/EL with QUORUM for critical writes)
- Partition: writes accepted on any side; conflicts via LWW.
- Normal: with `LOCAL_QUORUM`, writes ack in 1–5 ms locally. With `EACH_QUORUM`, cross-region: 80–200 ms.
- Cost: lower, self-host or managed.
- Consistency: tunable per query.

### Option C: Postgres regional + async cross-region replication
- One primary per region (or one global primary).
- Reads from local replicas (eventual).
- Cross-region replication is async — fast writes locally, but cross-region reads can be stale.
- Partition between regions: regional primary stays available.

### Decision drivers
- If strong global consistency essential (banking) → Spanner.
- If high write throughput per region + tunable consistency → Cassandra.
- If team familiar with Postgres + regions mostly independent → C.

PACELC frames this clearly: "we want PC/EC because the writes must reconcile globally" vs "we want PA/EL because users tolerate seconds of lag."

---

## 12. The "Everything is Tunable" View

PACELC's labels are useful but not absolute. Modern systems:
- Tunable per query (Cassandra, Mongo, DynamoDB).
- Tunable per replica set (Mongo, Cassandra).
- Tunable per region (read replicas).
- Tunable per workload (read-heavy vs write-heavy).

A more sophisticated frame: **what consistency, what latency, what availability, for this specific operation under this specific failure scenario?** PACELC is the vocabulary; the answer is per-op.

---

## 13. Common Mistakes

- **Labeling a whole system PA or PC.** Real systems tune per-call.
- **Ignoring the E-side trade-off.** "We're CP" means little if writes take 200ms in normal ops.
- **Choosing EC for everything.** Latency catastrophic.
- **Choosing EL for everything.** Inconsistencies pile up.
- **Assuming async replication = "fast and fine".** It's fast and *eventually consistent*; build for that.
- **Synchronous cross-region replication "for safety".** Often the wrong trade. Use regional primaries + async.
- **Forgetting that partitions are rare; latency is constant.** The E-side bites you daily.
- **Skipping PACELC because CAP is more famous.** PACELC is more accurate.

---

## 14. Cheat Card

```
PACELC      If Partition, choose A or C; Else, choose L or C

PA/EL       partition: available; normal: low latency
             Cassandra, DynamoDB (default), Riak, CouchDB

PA/EC       partition: available; normal: consistent
             MongoDB with w=majority

PC/EL       partition: consistent; normal: low latency (rare)

PC/EC       partition: consistent; normal: consistent
             Spanner, CockroachDB, etcd, Zookeeper,
             Postgres+sync standby

THE INSIGHT  CAP captures rare cases; PACELC captures the daily trade-off
             (synchronous vs async replication)

GEOGRAPHY    far replicas amplify the EC-vs-EL cost
             cross-region sync adds 50–200 ms per write

TUNABLE      modern DBs let you choose per query
             not a system-wide label

CHOOSE PER OP
  payment, lock, reservation       PC/EC
  cart, timeline, dashboard         PA/EL
  user profile                       PA/EL with EC reads when needed
  global ledger                      PC/EC

RULE         CAP names the partition trade-off.
             PACELC also names the latency trade-off.
             You feel the L cost every day; design for it.
```

---

## 15. Resources

### Papers
- "Consistency Tradeoffs in Modern Distributed Database System Design: CAP is Only Part of the Story" — Daniel Abadi, IEEE Computer 2012.
- "Brewer's Conjecture and Feasibility of Consistent, Available, Partition-Tolerant Web Services" — Gilbert & Lynch (CAP proof).

### Books
- *Designing Data-Intensive Applications* — Martin Kleppmann.
- *Database Internals* — Alex Petrov.

### Articles
- "Problems with CAP, and Yahoo's little known NoSQL system" — Daniel Abadi (the PACELC origin).
- "PACELC: an updated CAP" — Eric Brewer remarks.
- "Jepsen analyses" — practical PACELC tests in the wild.

### Videos
- Daniel Abadi — lectures on PACELC.
- Martin Kleppmann — distributed systems lectures.

### Adjacent reading
- [CAP Theorem →](./cap-theorem.md)
- [Consistency Models →](./consistency-models.md)
- [Consensus →](./consensus.md)
- [Quorum-Based Replication →](./quorum.md)
- [Replication →](../04-databases/replication.md)
- [ACID vs BASE →](../04-databases/acid-vs-base.md)

---

*Previous:* [← CAP Theorem](./cap-theorem.md)  |  *Next:* [Consistency Models →](./consistency-models.md)
