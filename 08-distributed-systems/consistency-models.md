# Consistency Models (Strong, Eventual, Causal, Read-Your-Writes, Monotonic)

> **TL;DR** — A **consistency model** is a contract between a system and its users about what reads can return after writes. The strongest is **linearizability** ("strong consistency") — every read sees the most recent write, as if there were a single global copy. The weakest practical level is **eventual consistency** — replicas converge eventually but reads can return any value in-between. Between these live the *useful* models: **sequential consistency**, **causal consistency**, **read-your-writes**, **monotonic reads**, **monotonic writes**, **writes-follow-reads**, and various **session guarantees**. Stronger models are easier to reason about but cost availability and latency; weaker models scale better but burden applications with conflict handling. The real-world dial isn't binary — it's a **lattice** of options, each appropriate for some workload. Pick deliberately, document explicitly, and assume your colleagues won't remember which one you chose.

---

## 1. Why Consistency Models Matter

Imagine two clients writing to a distributed store with replicas:

```
   client A: write X = 1
   client A: write X = 2
   client B: read X → ?
```

What should B see? "Whatever was most recent" is a fine answer for a single server. With multiple replicas and async replication, B might see 0, 1, or 2 depending on which replica it talks to and when.

The **consistency model** is the rulebook saying which of those answers is allowed.

If you don't pick one, you've chosen "anything goes" — which is the developer's nightmare.

---

## 2. The Lattice (Roughly Stronger → Weaker)

```
LINEARIZABILITY        every read sees most recent write across the system
  ↑
SEQUENTIAL CONSISTENCY all clients see operations in the same order
  ↑
CAUSAL CONSISTENCY     operations preserve causal order
  ↑
SESSION GUARANTEES     read-your-writes, monotonic reads, monotonic writes,
                        writes-follow-reads
  ↑
EVENTUAL CONSISTENCY   replicas converge eventually
```

Each level is **strictly weaker** than the ones above; what's allowed at a lower level may be disallowed at a higher one.

Real-world systems pick one or expose tunable controls.

---

## 3. Linearizability ("Strong Consistency")

**Definition**: there exists a single total order of all operations consistent with real time. Every read returns the value of the most recent write **as of the moment the read began**.

```
                              real time ─────────►
   A: write X=1 ──────────────┤
   B:               read X ───┤  must see X=1 or later
```

Linearizability is what your single-node intuition expects. If you wrote, then read, you see the write. If two readers issue at the same time, they may differ — but only at the level of overlap; once a read returns, every later read sees at least as recent state.

### Where you see it
- Single-node databases.
- Etcd, Zookeeper, Consul (for KV ops on the leader).
- Spanner (TrueTime + Paxos commit-wait).
- Postgres / MySQL primary.
- DynamoDB strongly-consistent reads.

### Cost
- Requires consensus (Paxos / Raft).
- Latency: synchronous replication, possibly cross-region.
- Availability: minority side stops on partition (CAP-CP).

### When you need it
- Locks, leader election, financial transactions, identity.
- Anywhere "the most recent value" is essential to correctness.

---

## 4. Sequential Consistency

**Definition**: all clients see operations in **the same order**, but that order doesn't need to match real time.

Weaker than linearizability — clients can be reading "old" data globally, as long as they all agree on the sequence.

```
real time:      A: write X=1, then B: read X
sequential OK:  some clients might see X=1 immediately, others later,
                but they all see the same total order
```

Used in CPU memory models (the JVM's sequential consistency for `volatile`); rare in databases. The distinction from linearizability mostly matters for memory models, not for designing databases.

---

## 5. Causal Consistency

**Definition**: operations preserve **causal order**. If A happens before B (e.g., A's read fed into B's write), all clients see A before B. Operations not causally related can be seen in any order.

```
real time:   A: post "Storm coming"   ──► B: reply "Stay safe"
                                            │
                                            causally depends on A

  all clients must see A before B if they see B at all
  but might not see them at the same time as another concurrent post
```

This is what most users actually expect. "I posted, you replied; nobody should see your reply without my post."

### Where you see it
- Riak with vector clocks.
- COPS, Eiger (research).
- MongoDB's "causal consistency" sessions.
- Some social networks for feeds.
- Bayou (early distributed sync system).

### Cost
- Needs per-operation metadata (vector clocks or similar).
- Cheaper than linearizability; doesn't require global agreement.

### When you need it
- Social timelines / messages where "reply before post" is wrong.
- Collaborative editing partial guarantees.
- Anywhere causality matters but global order doesn't.

See [Clocks →](./clocks.md) for causal tracking.

---

## 6. The Session Guarantees

A practical bundle, named in the **Bayou** paper (1995). These are guarantees within a **single client's session**.

### 6.1 Read-Your-Writes
After you write, subsequent reads (from the same client) return at least that write.

```
client A: write X=1; read X → 1 (not stale 0)
```

Easy to implement at the client level: route the client's reads to the primary or to a replica caught up past their last write timestamp.

### 6.2 Monotonic Reads
Successive reads from the same client never go backward in time.

```
client A: read X → 5; read X → 3      ← WRONG (went backward)
client A: read X → 5; read X → 5 or 7  ← OK
```

If a user refreshes a page and sees fewer messages than before, you've violated monotonic reads.

### 6.3 Monotonic Writes
Writes from the same client are applied in the order issued.

```
client A: write X=1; write X=2; another client reads X → 2
not allowed: see X=1 after seeing X=2 from same source
```

### 6.4 Writes-Follow-Reads
If you read X and then write Y based on it, anyone seeing your Y will see (at least) the X you read.

Models like Riak and Cassandra typically offer these as session-level guarantees through libraries that track per-session metadata.

---

## 7. Eventual Consistency

**Definition**: if no new writes happen, eventually all replicas converge to the same value.

That's it. Nothing about ordering, nothing about lag.

```
write X=1 on replica R1
write X=2 on replica R2 (concurrent)

eventually: all replicas have X=2 (or X=1, depending on resolution)

before "eventually" arrives: reads can return 1, 2, neither, both
```

### Where you see it
- DNS (TTL-bounded).
- Cassandra default.
- DynamoDB default reads.
- S3 (before 2020 it was weak; now strong read-after-write).
- Most CDN caches.
- Git (eventually consistent across clones).

### Strengths
- Highest availability (no need to reach quorum).
- Lowest latency (any replica answers).
- Tolerates arbitrary partitions.

### Weaknesses
- Reads can return wildly different values.
- Application must handle inconsistency (conflict resolution, retries).
- "Eventually" might be milliseconds or weeks.

### When you can use it
- Read-heavy with stale-tolerance.
- Truly independent writes.
- Pre-aggregated data.
- CDN-style caches.

The honest take: **eventual consistency is the default for the cheapest scale**, but applications often want at least session guarantees on top.

---

## 8. Comparison Table

| Model | Guarantee | Cost | Where |
|---|---|---|---|
| **Linearizability** | Most-recent-write visible globally | High latency, CP | Etcd, Spanner, RDBMS primary |
| **Sequential** | All clients see same order | Medium | Memory models, rare in DBs |
| **Causal** | Causally-related ops in order | Vector clocks | Riak, Mongo causal sessions |
| **Read-your-writes** | Your writes visible to you | Routing trick | Most modern DBs offer it |
| **Monotonic reads** | No going backward | Sticky replica | Same |
| **Monotonic writes** | Your writes in order | Easy | Same |
| **Writes-follow-reads** | Causality from reads | Tracking | Less common |
| **Eventual** | Converge if no new writes | None upfront | Cassandra, S3, DNS |

---

## 9. What Most Apps Actually Want

The most common pragmatic combination: **read-your-writes + monotonic reads + bounded staleness**. This gives users a coherent experience without the cost of full linearizability.

```
strong-write, eventual-read-with-RYW:
  - writes go to primary, ack on quorum
  - reads usually go to replicas (eventual)
  - but a user's reads after their own writes are routed
    to the primary OR to a replica caught up past their
    write timestamp
```

This pattern works because most user complaints about consistency are "I did X, why don't I see X?" Read-your-writes solves that without paying for linearizability everywhere.

---

## 10. Implementing Session Guarantees

### Sticky session routing
Pin client to a specific replica that has the writes they need. Simple but doesn't scale well.

### Write-tracking tokens
After a write, the server returns a token (often a log offset / timestamp). Client passes it on subsequent reads. Server routes to a replica caught up past the token.

```
POST /write → response includes "token: 12345"
GET /read?after=12345 → routed to replica with offset >= 12345
```

Mongo's `clientReadConcern` and Cassandra's `LightweightTransactions` work this way under the hood. DynamoDB's "Consistent Read" forces the strong path.

### Causal context
Mongo causal sessions: every operation includes the client's last seen cluster time. Reads wait for replicas to catch up to that time.

These all exist because **session guarantees are what users actually feel**.

---

## 11. Strong vs Eventual: A Concrete Example

User updates their profile photo. The system caches the photo URL.

### Strong consistency
```
POST /profile (photo=new.jpg) → updates DB
GET /profile → returns new.jpg

every other reader → eventually consistent? No, strong: new.jpg everywhere.

latency: write 50ms (sync replicate), read 5ms
availability: minority side fails on partition
```

### Eventual consistency
```
POST /profile → updates one replica, ack
GET /profile (different replica) → still old.jpg

write latency: 5ms
read latency: 1ms (any replica)
user sees old.jpg briefly after update

mitigations: read-your-writes (route their reads to written replica)
            CDN cache invalidation by tag
```

The same operation, two trade-offs, both legitimate.

---

## 12. Consistency vs Isolation

Often confused. Two different concepts:

- **Consistency** (CAP / consistency models) — about **replicas and reads after writes**. Distributed systems concept.
- **Isolation** (ACID's I) — about **concurrent transactions** and what they see of each other. Single-database concept (mostly).

A linearizable DB can have weak isolation (read committed); a non-linearizable DB can have strong isolation (serializable).

In Postgres, "serializable isolation" prevents transactional anomalies. In etcd, "linearizable" means a single operation sees the latest committed value. Different axes.

See [Transactions & Isolation Levels →](../04-databases/transactions-isolation.md).

---

## 13. Real-World System Choices

### Linearizable
- Etcd, Zookeeper, Consul KV.
- Spanner.
- CockroachDB, TiDB (linearizable within key).
- Postgres / MySQL primary.

### Causal+
- MongoDB (with causal session).
- Riak with vector clocks (legacy).

### Eventual
- Cassandra default.
- DynamoDB default reads.
- DNS.
- S3 (now strong after 2020; was eventual).
- CDN caches.
- Most key-value caches.

Most modern systems are **tunable** — you pick the model per operation.

---

## 14. Common Anti-Patterns

### "We have replicas, so we have HA."
Replicas != consistency model. Replicas of an eventually-consistent system are eventually-consistent. Don't conflate availability with consistency.

### "Our cache is fine; we'll just invalidate on write."
Without read-your-writes guarantees and invalidation timing, the cache can show stale data immediately after the user's own write. Frustrating.

### "We're eventually consistent" with no convergence strategy
Eventually doesn't mean "eventually correct"; it means "eventually agreed on something". Without conflict resolution, the "something" might be wrong (last-write-wins on skewed clocks loses data).

### "Strong consistency everywhere, slow is the user's problem"
Synchronous global replication makes every write 100+ ms. Half the use cases don't need it.

### Ignoring session guarantees
Users notice "I did X, I don't see X" before they notice anything else.

---

## 15. Worked Example: Comment System

A user posts a comment on a popular video. Other users want to see it.

### Approach A: Linearizable everywhere
- Write goes through consensus (Paxos/Raft).
- Every read globally sees it within milliseconds.
- Latency: 50–200 ms per write.
- Scale: limited by consensus throughput.

### Approach B: Eventual, with read-your-writes
- Write goes to primary, async replicates.
- The poster's reads route to the primary (or a tagged replica) — sees their own comment instantly.
- Everyone else: eventually consistent, lag ~100 ms.
- Higher throughput, lower write latency.

### Approach C: Strong write, eventual fan-out, causal session
- Write through transactional primary (linearizable within the comment-thread aggregate).
- Fan-out to feed projections (eventual).
- Causal session ensures users see comments in order.

Most large systems (Reddit, Twitter, YouTube) pick something like B or C, not A. The user experience is identical for the author (RYW); other users tolerate small lag.

---

## 16. Common Mistakes

- **Treating "consistency" as one thing.** It's a spectrum.
- **Choosing the strongest model by default.** Latency cost is real.
- **Choosing eventual without conflict strategy.** Divergence-by-default.
- **Confusing CAP-C (linearizability) with consistency model lattice.** They're related but different.
- **Confusing consistency with isolation.** Different axes.
- **Promising "strong reads" without quorum.** A single replica read isn't strong.
- **Forgetting session guarantees.** Users notice these first.
- **Assuming a DB labeled "consistent" gives you what you think.** Read the docs — it often only applies to specific operations.

---

## 17. Cheat Card

```
LINEARIZABILITY  most recent write visible globally
                  cost: consensus, latency, CP under partition
                  use: locks, leader election, money, identity

SEQUENTIAL       global order, not necessarily real-time
                  mostly memory models

CAUSAL           causally-related ops in order
                  use: social timelines, collab tools, "happened-before"

SESSION:
  READ-YOUR-WRITES   see your own writes
  MONOTONIC READS    no going backward
  MONOTONIC WRITES   your writes in order
  WRITES-FOLLOW-READS  what you read informs what others see of your writes

EVENTUAL         converge eventually; reads can return anything
                  use: caches, CDN, telemetry, anything stale-tolerant

PRACTICAL PICK   eventual + read-your-writes + monotonic reads
                  covers ~95% of user-facing cases

CHOICE BY OP
  payment/lock         linearizable
  comment/timeline     causal or eventual + RYW
  cart                 eventual + RYW
  product page         eventual + bounded staleness

NOT THE SAME     consistency (replicas) ≠ isolation (transactions)

PITFALLS         strong-everywhere is slow;
                  eventual-everywhere is wrong;
                  forgetting session guarantees;
                  conflating CAP-C and weaker forms

RULE             Pick the weakest model that the workload can tolerate.
                 Then promise it loudly and code to it.
```

---

## 18. Resources

### Papers
- "Linearizability: A Correctness Condition for Concurrent Objects" — Herlihy & Wing, 1990.
- "Session Guarantees for Weakly Consistent Replicated Data" — Terry et al., 1994 (the Bayou paper).
- "Don't Settle for Eventual: Scalable Causal Consistency for Wide-Area Storage with COPS" — Lloyd et al., SOSP 2011.
- "Consistency in Non-Transactional Distributed Storage Systems" — Viotti & Vukolić (taxonomy of dozens of models).

### Books
- *Designing Data-Intensive Applications* — Martin Kleppmann (Ch 5 and 9).
- *Database Internals* — Alex Petrov.

### Articles
- "Strong consistency in Cassandra is not what you think" — various.
- "Causal consistency" — Peter Bailis blog posts.
- Jepsen analyses for every major DB.

### Videos
- Kyle Kingsbury / Jepsen — Strange Loop talks.
- Martin Kleppmann — distributed systems lectures.
- ByteByteGo — "Consistency Models".

### Adjacent reading
- [CAP Theorem →](./cap-theorem.md)
- [PACELC →](./pacelc.md)
- [Clocks →](./clocks.md)
- [Consensus →](./consensus.md)
- [Quorum-Based Replication →](./quorum.md)
- [Transactions & Isolation Levels →](../04-databases/transactions-isolation.md)
- [CRDTs →](./crdts.md)

---

*Previous:* [← PACELC Theorem](./pacelc.md)  |  *Next:* [Consensus Algorithms →](./consensus.md)
