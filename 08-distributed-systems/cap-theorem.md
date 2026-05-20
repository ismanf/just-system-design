# CAP Theorem

> **TL;DR** — In a distributed system, you can have at most two of **C**onsistency, **A**vailability, and **P**artition tolerance — and since **partitions are inevitable**, the real choice is between consistency and availability **during a partition**. CAP is famously misunderstood: it does not say "pick two." It says, "when the network partitions (which it will), you must choose between refusing requests to stay consistent (**CP**) or answering them with possibly-stale data (**AP**)." Everyone trades. **Spanner / etcd / Zookeeper / Postgres-primary** are CP — they sacrifice availability under partition to preserve consistency. **DynamoDB / Cassandra / Riak / DNS** are AP — they keep answering, accept divergence, reconcile later. CAP says nothing about latency, throughput, or the normal (non-partitioned) case — which is where most performance trade-offs actually live. For that, see [PACELC →](./pacelc.md). CAP is a starting framework; reality is more nuanced.

---

## 1. The Theorem

Eric Brewer's 2000 conjecture, formalized by Gilbert & Lynch in 2002:

> **In any distributed data system, you cannot simultaneously guarantee all three of:**
> - **Consistency**: every read returns the most recent write.
> - **Availability**: every request gets a (non-error) response.
> - **Partition tolerance**: the system continues to operate despite arbitrary message loss between nodes.

In normal operation (no partition), you can have C + A + P. **The trade-off only forces itself during a partition.**

```
    Consistency
        ▲
        │
      CP│       (Spanner, etcd, Zookeeper, traditional RDBMS)
        │
        ├──────►  Availability
       /│
      / │
  AP /  │       (DynamoDB, Cassandra, Riak, CouchDB, DNS)
        │
   Partition tolerance (always required in a real distributed system)
```

Real distributed systems must tolerate partitions. So the *practical* CAP question is: when a partition happens, do you favor **C** (refuse requests, return errors) or **A** (answer anyway, possibly with stale data)?

---

## 2. What Each Letter Actually Means

### Consistency (C)
**Linearizability**: every read sees the most recent successful write, as if there were a single global copy. Strong consistency.

This is the *strict* CAP definition. In informal use, "consistency" might mean "eventual consistency" or "session consistency" — but CAP's C is the strongest one.

### Availability (A)
Every non-failing node returns a non-error response, within some bounded time, for every request.

Importantly: "available" in CAP doesn't mean "fast" or "high uptime." It's a binary — either every request gets answered, or you're not A.

### Partition tolerance (P)
The system keeps working when messages between nodes are dropped or delayed arbitrarily.

Networks fail. Switches reboot. Cables break. AZs disconnect. **P isn't optional in a real distributed system** — you must tolerate the possibility, even if it's rare.

This is why CAP-during-partition is the real decision.

---

## 3. The Famous Misunderstanding

You'll see diagrams showing three corners and "pick two." This is misleading.

In a single-node system: no partition possible; you can have C + A trivially.

In a multi-node system: partitions are possible; you must choose what to do during them.

CAP is **not** a permanent design label like "we're a CA database." It's **a behavioral choice during partition**.

A more accurate statement:
> When (not if) a partition happens, you must give up either consistency or availability. In the absence of a partition, you can have both.

---

## 4. CP Systems: Refuse Rather Than Lie

CP systems prioritize correctness. During a partition, the side of the partition that can't ensure consistency stops serving (or only serves reads if they can be guaranteed fresh).

Examples:
- **Apache Zookeeper / etcd / Consul** — quorum-based; minority side stops accepting writes.
- **Google Spanner** — synchronous Paxos across replicas; loses availability if quorum lost.
- **CockroachDB / TiDB** — Raft-based; minority stalls.
- **Postgres / MySQL primary** — single writer; if you can't reach it, you can't write.
- **HBase** — region servers with single writer; region offline if its server is partitioned.

Behavior under partition:
- **Minority side**: reject writes, sometimes reject reads.
- **Majority side**: continue normally.
- **After partition heals**: minority catches up.

Used when **correctness > uptime** — banking, identity, leader election, locks.

---

## 5. AP Systems: Answer, Reconcile Later

AP systems prioritize availability. During a partition, both sides accept writes; conflicts are resolved later.

Examples:
- **DynamoDB** (default eventually-consistent reads).
- **Cassandra** with low consistency level (e.g., `LOCAL_ONE`).
- **Riak** (legacy).
- **CouchDB**.
- **DNS** — caches serve stale data when authoritative servers are unreachable.
- **Git** (yes, git is AP).

Behavior under partition:
- Both sides accept writes.
- Writes may conflict (same key, different values).
- Conflicts resolved on read or via background process (last-write-wins, CRDTs, vector clocks).
- After partition heals: replicas reconcile.

Used when **uptime > correctness** — shopping carts, social timelines, sensor data, caches, content delivery.

---

## 6. Concrete Scenarios

### Scenario 1: Banking transfer
Strong consistency essential. Two debits cannot happen on the same balance. **Choose CP.** Refuse the transfer if you can't confirm.

### Scenario 2: Shopping cart
Better to let the user add an item than to error out. Multiple writes can be merged (union of carts). **Choose AP** with conflict resolution.

### Scenario 3: Leader election
Two leaders is worse than no leader. **Choose CP** — refuse to act without quorum.

### Scenario 4: DNS resolution
Better to give a slightly stale IP than no IP. **Choose AP** with TTL-bounded staleness.

### Scenario 5: Inventory
"Do we have stock?" is read-mostly; if a partition causes overselling by 1 occasionally, refund and move on. **Choose AP** for read scale, CP for the actual decrement. (Most real systems split.)

### Scenario 6: Distributed lock
Two holders is catastrophic. **Choose CP.** No quorum, no lock.

---

## 7. The Famous Critique: CAP Is Too Coarse

CAP gives you a binary view. Real systems are nuanced:

- **Latency matters even without partitions.** A "consistent" system that takes 50 ms per write because it synchronously replicates is making a trade-off CAP doesn't capture.
- **Tunable consistency.** Cassandra lets you choose ONE / QUORUM / ALL per query. Same DB, different CAP behavior per call.
- **Partial availability.** Reads can be available while writes are not; some keys available, others not.
- **Eventually consistent isn't lawless.** It has rules (causal consistency, read-your-writes, monotonic reads).
- **Partitions aren't just "yes/no".** They can be slow networks, asymmetric routes, packet loss percentages.

Hence **PACELC** — Daniel Abadi's extension that adds the latency-vs-consistency trade-off in the non-partitioned case. See [PACELC →](./pacelc.md).

---

## 8. The "CA" Myth

You'll see "CA" listed as a category — "consistent, available, not partition-tolerant." This describes a single-node system. **It's not a category for distributed systems** because P isn't optional.

Marketing materials sometimes claim "CA" for systems that "never partition" (e.g., on a private LAN). Reality: networks always partition, even briefly. "We never partition" means "we haven't yet" or "we accept downtime when we do."

Don't fall for it.

---

## 9. Sliding Consistency

Many real systems let you slide along the spectrum per operation.

### Cassandra
```sql
INSERT INTO users (...) USING CONSISTENCY QUORUM;
SELECT ... USING CONSISTENCY ONE;
```

Per-query consistency level. Choose what you need.

### DynamoDB
- Default eventually consistent reads.
- Optional strongly consistent reads (2× cost).

### Postgres with read replicas
- Writes go to primary.
- Reads from replicas (eventually consistent) or primary (strongly consistent).

### Kafka
- `acks=0` (best effort) → AP-leaning.
- `acks=all` + `min.insync.replicas` → CP-leaning.

A modern view: **CAP isn't a property of the database. It's a property of the access pattern.**

---

## 10. The Math (Briefly)

Gilbert & Lynch's 2002 proof:

Assume system claims C + A + P. Two nodes N1, N2 hold value v. Partition them. Write `v=1` to N1; read from N2. Under A, N2 must respond. Under P, N1 and N2 can't communicate. Under C, N2 must return 1 — but it has no way to know N1 was written. Contradiction.

The proof formalizes the obvious. The value of CAP is the **vocabulary** for discussing distributed trade-offs, not the proof itself.

---

## 11. Failure Mode: "We're CP" Without a Plan

A common pattern: "we built on Spanner, so we're CP." Then a regional outage hits, the system stops serving 30% of traffic, and the team is shocked.

CP doesn't mean "always works." It means "refuses to serve incorrect data." That's correct behavior. But your **operational plan** must account for it:
- What's the SLA for write availability?
- How quickly does the system re-elect leaders / form new quorums?
- What's the user experience when a partition happens? (Show "service unavailable" or auto-retry?)
- Do you have a degraded read-only mode?

A "CP system" with no failover plan is a system that goes dark periodically and surprises everyone.

---

## 12. Failure Mode: "We're AP" and Conflicts Pile Up

"AP" doesn't mean "anything goes." Conflicts must be resolved.

Strategies:
- **Last-write-wins (LWW)** — newer timestamp wins. Simple; loses data on clock skew.
- **Vector clocks** — track causal history; let app resolve conflicting versions.
- **CRDTs** — types that converge deterministically (counters, sets, registers). See [CRDTs →](./crdts.md).
- **App-level resolution** — return conflicts to the caller; let them choose.
- **Operational transformation** — used in Google Docs / collaborative editing.

If you're AP without a conflict-resolution strategy, you have a divergence-by-default system that quietly corrupts state.

---

## 13. Where Real Systems Sit

| System | CAP class | Notes |
|---|---|---|
| **Spanner** | CP (with great availability via global Paxos) | Massive infra masks the CP downside |
| **CockroachDB / TiDB / YugabyteDB** | CP | Raft-based; minority stalls |
| **Postgres / MySQL (primary)** | CP | Single writer |
| **Etcd / Zookeeper / Consul KV** | CP | Quorum-based |
| **DynamoDB** | tunable (default AP) | Strong consistency optional |
| **Cassandra** | tunable | Per-query CL |
| **Riak** | AP | Classic Dynamo descendant |
| **MongoDB** | tunable; depends on read/write concern | Often CP with `w=majority` |
| **Couchbase** | AP | Eventually consistent across nodes |
| **DNS** | AP | TTL-bounded eventual consistency |
| **S3** | Strong read-after-write (since 2020); CP-leaning | Object-level |
| **Redis Cluster** | AP-leaning; CP with consensus features off | Replication async |
| **Kafka** | tunable | `acks` config |

---

## 14. The Honest Modern View

CAP was right in 2000 but oversimplified for 2026. Better mental model:

1. **Partitions are rare; latency is constant.** Most systems care more about latency than partition behavior (see [PACELC →](./pacelc.md)).
2. **Most modern systems are tunable.** Per-call consistency choice.
3. **"Consistency" isn't binary.** A spectrum of models — strong, sequential, causal, eventual, etc. See [Consistency Models →](./consistency-models.md).
4. **Partition tolerance is a property of the system; the trade-off is per-operation.**
5. **The real engineering question is: what should this specific operation do when consistency and latency conflict?**

CAP gives you a vocabulary. PACELC gives you more. Consistency-model lattices give you the most precision.

---

## 15. Worked Example: Designing a Reservation System

A booking platform: hotels, flights. The system needs to allocate seats / rooms reliably.

### Reservation creation
- **CP**: must not double-book. Use Spanner / CockroachDB / Postgres with serializable isolation. Refuse the booking if quorum is lost.

### Browsing inventory
- **AP**: showing slightly stale availability is fine. Caches at every layer; Cassandra / DynamoDB for catalog. Better to load a page with last-known counts than to error.

### Booking confirmations
- **CP**: confirmation must persist. Use the transactional store.

### User-facing search
- **AP**: search index (Elasticsearch). Eventually consistent. Even if a booking was just made, ~10s lag is acceptable for the search UX.

### Notifications
- **AP**: best effort. Loss of a notification is regrettable but not catastrophic.

The system as a whole is a **hybrid** — CP where correctness matters, AP where availability matters. CAP didn't decide; you did, per workload.

---

## 16. Common Mistakes

- **Treating CAP as design philosophy.** "We're an AP shop." No — CAP is per-operation behavior.
- **Believing in "CA" databases.** Marketing speak. Real distributed systems must tolerate partitions.
- **Choosing AP without conflict resolution.** Divergence by default = corruption.
- **Choosing CP without HA plan.** Partition → outage → angry users.
- **Confusing CAP-C with eventual consistency.** Strong consistency (linearizability) is the C in CAP. EC is "no C."
- **Designing the whole system around one CAP choice.** Real systems blend.
- **Ignoring latency.** CAP says nothing about it. Most users feel latency, not partition behavior. See PACELC.
- **Mistaking "high availability" for CAP-A.** They're related but different — CAP-A is per-request availability under partition.
- **Assuming partitions are rare so CAP doesn't matter.** They happen, and when they do, the design holds up — or doesn't.

---

## 17. Cheat Card

```
CAP        in a partition, choose between Consistency and Availability
            P is mandatory in real distributed systems

C          linearizability — most recent write visible everywhere
A          every non-failing node responds (no errors)
P          system continues despite arbitrary message loss

CP         refuse rather than lie
            Spanner, etcd, Zookeeper, Postgres primary, RDBMS
            best for: financial, locks, leader election, identity

AP         answer with possibly-stale data, reconcile later
            DynamoDB, Cassandra, Riak, DNS, S3 (mostly)
            best for: catalogs, carts, timelines, sensors

NOT A CHOICE  partitioning isn't optional; "CA" systems are single-node

TUNABLE     Cassandra/DynamoDB/Mongo let you slide per-query

CRITIQUE    CAP ignores latency; PACELC adds the L=Latency dimension
            most real systems blend CP and AP per workload

PITFALLS    "we're AP" without conflict resolution → divergence
            "we're CP" without HA plan → outage = surprise
            confusing CAP-A with high availability
            assuming CAP labels are permanent design choices

RULE        CAP is per-operation. Choose the right behavior
            for the partition-time response, then engineer around it.
```

---

## 18. Resources

### Papers
- "Brewer's Conjecture and the Feasibility of Consistent, Available, Partition-Tolerant Web Services" — Gilbert & Lynch, 2002.
- "Towards Robust Distributed Systems" — Eric Brewer, PODC 2000 (the original).
- "CAP Twelve Years Later: How the 'Rules' Have Changed" — Eric Brewer.
- "Consistency Tradeoffs in Modern Distributed Database System Design: CAP is Only Part of the Story" — Daniel Abadi (the PACELC paper).

### Books
- *Designing Data-Intensive Applications* — Martin Kleppmann (Ch 9 covers CAP, isolation, consensus).
- *Database Internals* — Alex Petrov.

### Articles
- "You Can't Sacrifice Partition Tolerance" — Coda Hale.
- "Please stop calling databases CP or AP" — Martin Kleppmann.
- "Jepsen analyses" — Kyle Kingsbury (every major DB tested under partition).

### Videos
- ByteByteGo — "CAP Theorem Explained".
- Martin Kleppmann — "Distributed Systems lectures" (Cambridge).
- Eric Brewer — original PODC keynote.

### Adjacent reading
- [PACELC →](./pacelc.md)
- [Consistency Models →](./consistency-models.md)
- [Consensus →](./consensus.md)
- [Quorum-Based Replication →](./quorum.md)
- [Split-Brain Problem →](./split-brain.md)
- [ACID vs BASE →](../04-databases/acid-vs-base.md)
- [Replication →](../04-databases/replication.md)

---

*Up:* [README ↑](../README.md)  |  *Next:* [PACELC Theorem →](./pacelc.md)
