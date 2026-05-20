# Quorum-Based Replication

> **TL;DR** — **Quorum-based replication** requires writes and reads to touch enough replicas that **any pair of operations overlaps in at least one node**. With **N** replicas, **W** writes, **R** reads, the key invariant is `W + R > N`. This guarantees that any read intersects with the most recent write. Choosing W and R lets you tune **consistency vs availability vs performance**: high W (strong durability, slower writes), high R (consistency, slower reads), low both (fastest, weakest). **Dynamo-style** databases (Cassandra, DynamoDB, Riak) make W and R configurable per request. **Consensus systems** (etcd, Spanner) effectively use majority quorums (`W = R = (N/2)+1`). Quorum reads + read-repair + anti-entropy keep replicas consistent over time. The pitfalls: **clock skew breaking last-write-wins**, **sloppy quorums under partition**, **stale reads in default configurations**, and **misunderstanding "QUORUM"** in Cassandra's tunable consistency.

---

## 1. The Basic Idea

Suppose you have N replicas of a key-value store. A write goes to W replicas before acknowledgment; a read goes to R replicas and picks the freshest answer.

```
N = 5 replicas

W = 3, R = 3:   W + R = 6 > 5  →  any read sees at least one node
                                   from the latest write quorum

write to nodes:  A B C
read from nodes:     C D E    ← C overlaps; latest value visible
```

The overlap guarantees consistency. **Quorum intersection** is the property that makes this work.

---

## 2. The Invariant: W + R > N

If `W + R > N`, every read quorum and write quorum share at least one replica. That shared replica has the latest write, so the read sees it.

```
N = 5
W = 4, R = 2:  W+R=6 > 5 ✓
W = 3, R = 3:  W+R=6 > 5 ✓  (the canonical "majority quorum")
W = 5, R = 1:  W+R=6 > 5 ✓  (read-mostly: fast reads, durable writes)
W = 1, R = 5:  W+R=6 > 5 ✓  (write-mostly: fast writes, slow reads)
W = 2, R = 2:  W+R=4 ≤ 5 ✗  no overlap guarantee — possible stale read
```

The arithmetic decides the trade-off:
- **Large W** → durable writes; high write latency.
- **Large R** → consistent reads; high read latency.
- **Low W + low R** → fast both; possible stale reads (eventually consistent).
- **W + R > N** → strong consistency.

For Dynamo-style databases, **W = R = quorum = (N/2)+1** is the standard "majority quorum" — best balance.

---

## 3. What "Touches" Means

Different databases interpret "write to W replicas" differently:

- **Strict quorum**: W replicas must **ack** the write before client sees success. Failure to reach W = write fails or pending.
- **Sloppy quorum**: writes can go to any W reachable nodes — not necessarily the canonical ones. Used in Dynamo to maintain write availability during partitions. Trade-off: temporary inconsistency.
- **Async replication after ack**: write ack'd at primary; others catch up later. NOT a quorum; it's classic master-replica.

The "quorum" label can hide important distinctions. Read the docs.

---

## 4. Tunable Consistency in Cassandra

Cassandra's per-query consistency level — the canonical example:

| Level | Replicas required |
|---|---|
| **ANY** | At least one node accepted the write (hinted handoff OK) |
| **ONE** | One replica ack'd |
| **TWO** | Two replicas |
| **THREE** | Three replicas |
| **QUORUM** | `(N/2)+1` |
| **LOCAL_QUORUM** | `(N/2)+1` in the local datacenter |
| **EACH_QUORUM** | Quorum in EVERY datacenter |
| **ALL** | All replicas |
| **SERIAL** / **LOCAL_SERIAL** | For lightweight transactions (Paxos-based) |

A single Cassandra cluster can serve linearizable writes for some queries (SERIAL) and at-least-one writes for others (ONE), per call. Powerful and dangerous.

The typical defaults:
- `LOCAL_QUORUM` for writes and reads inside a region.
- `EACH_QUORUM` for global-strong reads (slow).
- `ONE` for high-throughput, eventually-consistent paths.

---

## 5. DynamoDB Quorum Behavior

DynamoDB internally uses quorum reads/writes but exposes it as a simple toggle:
- **Eventually consistent read** (default) — reads from any replica. Fast, possibly stale.
- **Strongly consistent read** — quorum read. Slower (~50% more), no stale data.
- Writes are quorum (W = majority of 3 replicas).

DynamoDB hides the W/R mechanics. The exposed knob is "consistent read: yes/no."

---

## 6. Consensus Quorums

Raft / Paxos / ZAB use **strict majority quorums** for both writes and "linearizable reads":
- Cluster size N = 3, 5, 7.
- Writes require `(N/2)+1` acks.
- Linearizable reads require contacting the leader (which proves it's still leader by heartbeating majority, or holding a lease).

In consensus systems, you don't choose W and R — they're fixed at majority. The flexibility is at the schema/architecture level (which keys use this consensus group).

See [Consensus →](./consensus.md).

---

## 7. Read Repair and Anti-Entropy

Quorum reads can detect divergence:

```
read from 3 replicas: A returns v=5, B returns v=5, C returns v=3
quorum agrees on v=5; C is stale
→ trigger read repair: push v=5 to C asynchronously
```

This is **read repair**. Cassandra, Riak, DynamoDB all do this.

Additionally:
- **Background anti-entropy**: scheduled processes compare replica states (via Merkle trees, see [Merkle Trees →](./merkle-trees.md)) and reconcile differences. Cassandra calls this "repair"; Dynamo calls it "anti-entropy."

Quorum + read repair + anti-entropy = eventually consistent with bounded divergence.

---

## 8. Sloppy Quorums and Hinted Handoff

If the canonical replicas for a key are unreachable, what then?

- **Strict quorum**: write fails.
- **Sloppy quorum** (Dynamo): write to any W nodes, even non-canonical ones. The receiving node holds a "hint" — when the canonical replica comes back, transfer the write. Called **hinted handoff**.

```
canonical replicas for key K: A, B, C
A is down

write goes to B, C, D (sloppy quorum)
D writes locally + records a hint: "this is for A; deliver later"

when A returns, D pushes the data; hint cleared
```

Sloppy quorum preserves write availability under partition at the cost of momentary inconsistency. After partition heals, hints converge.

This is the **AP** end of CAP applied to quorum systems.

---

## 9. The Last-Write-Wins Trap

When two writes happen concurrently to different replicas, how do you decide which wins?

### Last-write-wins (LWW) by wall-clock timestamp
```
Replica A receives write at 14:00:00.100, value 5
Replica B receives write at 14:00:00.050, value 3
LWW: A's write wins (later timestamp)
```

Problem: **wall clocks lie**. If A's clock is 100 ms ahead, A always wins even when the writes happened simultaneously. **You can silently lose writes.**

### Better: vector clocks
Detect concurrency; let the application resolve. Riak's original model.

### Better still: CRDTs
Operations that converge correctly regardless of order. See [CRDTs →](./crdts.md).

### Acceptable LWW
Only if writes to the same key are rare or you genuinely don't care about ties.

The default in many quorum DBs is LWW by wall-clock. Know this and decide if it's OK for your data.

---

## 10. The Sloppy-Quorum Hidden Cost

Even with `W + R > N`, sloppy quorums can violate consistency:

```
canonical: A, B, C
partition: A unreachable; D substitutes
write to B, C, D (sloppy quorum, W=3)

read with R=3: read from A, B, C? A is back, doesn't have the data.
              two of three have the data, the read returns it ✓ (or not?)

if read sees A, B (no D, A doesn't have it):
   only 1 of 3 has the new data → conflict resolution
```

In practice, sloppy quorums mean "consistency until partitions, then best-effort." Most real systems accept this with eventual reconciliation.

---

## 11. Quorum and Latency

Larger quorums = higher latency.

- **W = 1**: fast (only need primary); risk of loss.
- **W = quorum**: medium (wait for majority); durable.
- **W = ALL**: slowest (wait for all); maximum durability.

Cross-region replicas amplify this. A cross-AZ quorum adds ~1 ms. Cross-region adds 50–200 ms.

Best practice: keep the "quorum members" geographically close (in one region). Use cross-region replicas for disaster recovery, not for primary write path.

---

## 12. Practical Configurations

### Small-scale: N=3
- W=2, R=2 (W+R=4>3) — majority quorum, classic.
- Tolerates 1 replica down for write.
- Tolerates 1 replica down for read.

### Larger: N=5
- W=3, R=3 — majority quorum.
- Tolerates 2 down.
- More replicas = more durability, more latency.

### Read-heavy: N=5
- W=4, R=2.
- Faster reads (R=2), slower writes (W=4).
- Same consistency guarantee.

### Write-heavy: N=5
- W=2, R=4.
- Faster writes, slower reads.

### Eventually consistent: N=5
- W=1, R=1.
- Maximum availability and speed. No consistency guarantee. Use only for stale-tolerant data.

### Multi-region: N=5 across 3 regions
- LOCAL_QUORUM (in one region) for fast.
- EACH_QUORUM for global-strong but slow.

---

## 13. Worked Example: Cassandra Tuning

A user-profile service runs on a 3-node Cassandra cluster.

### Default (LOCAL_QUORUM both)
- Writes: 2 of 3 acks.
- Reads: 2 of 3 acks. Read repair if mismatched.
- Latency: ~5–10 ms per query.
- Consistency: strong (within the region).

### High availability: ONE both
- Writes: 1 of 3 acks.
- Reads: 1 of 3 (first to respond).
- Latency: ~2 ms.
- Consistency: best-effort eventually consistent.

### Critical writes: QUORUM both, ALL writes for super-critical
- Writes: 3 of 3.
- Slower but everything is durable everywhere.
- One node down → write fails.

The point: tune per query, per workload. Same database, multiple consistency contracts.

---

## 14. Common Mistakes

- **Forgetting `W + R > N`.** Reads can return stale data.
- **Sloppy quorum default with no awareness.** "Consistent" until partition; then surprises.
- **LWW with wall-clock timestamps.** Lost writes on clock skew.
- **Treating "QUORUM" as a fixed property.** It's per-query.
- **Reads from ONE everywhere.** Stale data; users notice; debugging hell.
- **Writes to ALL** without considering write availability — one node down, no writes.
- **Cross-region EACH_QUORUM on the hot path.** 100+ ms per write.
- **Misunderstanding `LOCAL_QUORUM` semantics across DCs.** Local-only consistency.
- **No background anti-entropy.** Replicas drift slowly forever.
- **Replication factor 2.** Quorum=2; lose any one → can't write. Use RF=3.
- **Replication factor too high.** Latency, storage cost. 3 is the sweet spot for most.

---

## 15. Quorum vs Other Replication Models

| Model | How it replicates | Latency | Consistency | Examples |
|---|---|---|---|---|
| **Single primary, async** | Primary acks; replicates after | Lowest | Eventually consistent | Postgres async replica, MySQL classic |
| **Single primary, sync** | Primary waits for replicas | Higher | Strong | Postgres sync standby |
| **Multi-master** | All nodes accept writes; conflict resolve | Lowest | Eventually consistent (CRDT-friendly) | Riak, Dynamo |
| **Quorum** | Need W/R acks | Medium | Tunable | Cassandra, DynamoDB |
| **Consensus** | Majority via Raft/Paxos | Medium | Linearizable | etcd, Spanner |
| **Chain replication** | Sequential through nodes | Higher | Strong | CRAQ, original chain rep |

Quorum sits in the middle — flexible, tunable, the default of many modern eventually-consistent DBs.

---

## 16. Cheat Card

```
QUORUM         enough replicas to overlap any two operations

INVARIANT      W + R > N → reads always see latest writes

PARAMS
  N            total replicas
  W            replicas ack'd on write
  R            replicas read on read

CHOOSE
  W=R=majority    classic balanced (N/2+1 each)
  W=N, R=1        write-once-read-many (durable, fast reads)
  W=1, R=N        write-many-read-rare
  W=R=1           eventually consistent / max availability

OVERLAP        guaranteed only with W+R>N
SLOPPY QUORUM   substitute nodes when canonical replicas down
                + hinted handoff to converge later

CONFLICTS      LWW (risky with wall clocks) or vector clocks or CRDTs

READ REPAIR    fix stale replicas during read
ANTI-ENTROPY   background reconciliation (Merkle trees)

CASSANDRA      per-query: ANY/ONE/TWO/QUORUM/LOCAL_QUORUM/EACH_QUORUM/ALL

CONSENSUS      uses majority quorum implicitly (Raft/Paxos)

REPLICATION    rf=3 typical; rf=5 for HA; never rf=2

PITFALLS       W+R≤N silent inconsistency, LWW by wall clock,
                cross-region EACH_QUORUM on hot path,
                no anti-entropy, sloppy-quorum surprises

RULE           Choose W and R together to match your
                durability, consistency, and latency targets.
```

---

## 17. Resources

### Papers
- "Dynamo: Amazon's Highly Available Key-Value Store" — DeCandia et al., 2007 (the foundational quorum paper).
- "Cassandra — A Decentralized Structured Storage System" — Lakshman & Malik, 2010.
- "Quorum-Based Replication" — various texts (the math).

### Books
- *Designing Data-Intensive Applications* — Kleppmann (Ch 5).
- *Cassandra: The Definitive Guide* — Carpenter & Hewitt.

### Documentation
- **Cassandra consistency**: <https://cassandra.apache.org/doc/latest/cassandra/architecture/dynamo.html>
- **DynamoDB consistency**: <https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/HowItWorks.ReadConsistency.html>

### Articles
- "Probabilistically Bounded Staleness" — Peter Bailis (how quorum tradeoffs play out in practice).
- "Eventually consistent Cassandra has surprising consistency holes" — Jepsen analyses.

### Videos
- ByteByteGo — "Quorum Replication".
- Tim Berglund — Cassandra fundamentals.

### Adjacent reading
- [CAP Theorem →](./cap-theorem.md)
- [PACELC →](./pacelc.md)
- [Consensus →](./consensus.md)
- [Consistency Models →](./consistency-models.md)
- [Replication →](../04-databases/replication.md)
- [Merkle Trees →](./merkle-trees.md)
- [Clocks →](./clocks.md)
- [CRDTs →](./crdts.md)

---

*Previous:* [← Two-Phase Commit (2PC) and Three-Phase Commit (3PC)](./2pc-3pc.md)  |  *Next:* [Gossip Protocol →](./gossip-protocol.md)
