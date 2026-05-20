# Split-Brain Problem

> **TL;DR** — **Split-brain** is when a distributed system's nodes get partitioned and **both sides** keep operating independently — each thinking it's the authoritative cluster. Both accept writes; data diverges; reconciliation is painful or impossible. Split-brain is the single worst failure mode in distributed databases, replicated services, and clustered systems. The fix is **always the same recipe**: **require quorum** (only the majority side can make decisions), **fence stale leaders** (via tokens or STONITH so they can't keep writing), **use odd-numbered cluster sizes** (3, 5, 7) to guarantee a clear majority, and **prefer to be unavailable rather than divergent** (CP under CAP). Every consensus protocol (Raft, Paxos, ZAB) is fundamentally a split-brain prevention mechanism. When you hear "we had to manually reconcile two databases after the outage," that was a split-brain incident.

---

## 1. The Scenario

```
Cluster of 5 nodes, all replicas of a database.

Network partitions:
   ┌─────────────────────────────────┐
   │  partition A: nodes 1, 2        │
   │                                  │  
   │  partition B: nodes 3, 4, 5     │
   └─────────────────────────────────┘

Both sides see "I'm a cluster of N nodes; others are dead."

If both sides accept writes:
  - A writes "balance=80"  
  - B writes "balance=120"  

When partition heals:
  - Two conflicting values.
  - Some writes will be lost or require manual merge.
  - This is split-brain.
```

The damage scales with time. A 5-second partition with no writes? Trivial. A 30-minute partition during peak traffic? Hours of cleanup, possible data loss, possibly impossible to reconcile.

---

## 2. Why It Happens

Split-brain occurs when:
1. The cluster has **no clear quorum mechanism**, or
2. The mechanism is misconfigured, or
3. Both halves of a partition can each form a quorum (impossible with odd N and proper algorithm), or
4. A failed leader **doesn't actually stop writing** (no fencing).

The classic root cause: **trying to maintain availability during partition without consensus**.

---

## 3. The Core Defense: Quorum

The fundamental defense: **only a majority of nodes can make decisions.**

```
5 nodes total. Quorum = 3.

Partition into 2 + 3:
  - 2-node side: can't reach quorum → stops accepting writes.
  - 3-node side: has quorum → continues normally.

No split-brain: only one side serves writes.
```

This is why consensus algorithms (Raft, Paxos, ZAB) require majority. They **prefer unavailability over inconsistency** — a deliberate CAP-CP choice.

For this to work:
- Cluster size must be **odd** (3, 5, 7) so there's always a clear majority.
- Quorum is `(N/2) + 1`.

```
N=2:  quorum=2; either side has only 1 — both unavailable. Useless.
N=3:  quorum=2; partition 1+2 → minority/majority clear. ✓
N=4:  quorum=3; partition 2+2 → both unavailable. Wasteful.
N=5:  quorum=3; partition 2+3 → clear. ✓
```

**Always run odd-numbered clusters.** Even sizes waste a vote.

---

## 4. The Fencing Problem

Even with quorum, split-brain can occur if a **failed leader keeps writing** to shared resources.

```
Node A is leader, writing to a shared storage.
Network partition: A is on the minority side.
Quorum says: A is no longer leader. B (in majority) becomes leader.
But A doesn't know yet — keeps writing.
Now both A and B write to shared storage.
```

Two leaders, briefly. Bad.

The defense: **fencing**.

### 4.1 Fencing tokens
Each new leader gets a monotonic token. Every write carries the token. Storage rejects writes with stale tokens.

```
A leader, token=17. Writes to storage with token 17.
B becomes new leader, token=18.
Storage tracks latest seen token = 18.
A's continued writes (token 17) are rejected.
```

This is the **only fully safe** approach. See [Distributed Locks →](./distributed-locks.md) for the fencing-token discussion.

### 4.2 STONITH ("Shoot The Other Node In The Head")
On detecting failover, physically/forcibly stop the old leader: power off, kill process, network-isolate. Used in HA clustering (Pacemaker, Linux-HA).

Effective but harsh; requires out-of-band control over the node.

### 4.3 Lease-based with grace periods
Leader holds a lease with TTL. Must continuously renew. If can't renew (slow network, GC pause), it must voluntarily step down before the lease expires.

Trickier than it looks — depends on clocks and timely self-detection.

---

## 5. Without Quorum: How Split-Brain Hits

Real-world failure modes when systems don't enforce quorum:

### 5.1 Async replication primary failover
- Postgres async replica setups: if you promote a standby without demoting the primary, both write.
- DNS or load balancer continues sending some traffic to the old primary.
- Writes diverge.

### 5.2 Dual-master misconfiguration
- MySQL master-master with both accepting writes by mistake.
- Result: silently diverging schemas and rows.

### 5.3 Manual failover during partition
- Operator panics, manually promotes a replica.
- The old primary, separated by partition, is still up and serving.
- Two primaries.

### 5.4 Sentinel false alarms
- Redis Sentinel cluster has a network glitch; thinks the primary is dead; promotes a replica.
- Original primary is fine; clients still talk to it.

### 5.5 DNS / load balancer disagreement
- LB1 sends traffic to "old primary."
- LB2 sends traffic to "new primary."
- Both primaries serve.

These all stem from "no single point of truth for who the leader is" plus "no fencing."

---

## 6. CAP Lens

Split-brain is the **CAP-A overshoot** — favoring availability so aggressively that consistency breaks.

CAP-CP systems (Spanner, etcd, Zookeeper, CockroachDB) **refuse writes** on the minority side. By design. The minority is unavailable; that's the trade.

CAP-AP systems (Dynamo-style: Cassandra, Riak, DynamoDB) **accept writes on both sides**, then reconcile via vector clocks / CRDTs / last-writer-wins. They've designed conflict resolution in — so "divergence + reconcile" is the *expected* path, not split-brain in the catastrophic sense.

Real split-brain disasters happen in systems that **think they're CP but accidentally allow AP-style writes during partition**.

---

## 7. Famous Real Outages

### Knight Capital, 2012
A deployment race led to two different versions of trading code running. Wasn't strictly distributed split-brain but had the same flavor — divergent behavior across "two halves" of a system. Lost $440M in 45 minutes.

### MongoDB primary elections
Several documented incidents where MongoDB's primary election fell into split-brain when network partitions occurred during elections. Newer versions use Raft (since 4.0); older replica sets had bugs.

### GitHub MySQL outage, 2018
A 43-second partition led to split-brain in MySQL Orchestrator's master orchestration. Cleanup required 24 hours of careful manual reconciliation. Their post-mortem is a classic.

### AWS S3 / DynamoDB partial failures
S3 and DynamoDB are AP-style — they don't traditionally suffer "split-brain" but have had inconsistency windows during regional outages.

### Reddit's 2016 outage
A Kubernetes split-brain across data centers caused services to think they were primary in two places. Hours of recovery.

The pattern: **partial network failures are common; split-brain is the failure mode**.

---

## 8. Detection

Detection is hard because, from the inside, a partitioned node can't tell "I'm partitioned" vs "the others are dead." Indicators:

- **Lost heartbeats** to known peers.
- **Inability to form quorum** for new operations.
- **Conflicting timestamps / versions** detected during reconciliation.
- **Two nodes claiming "I'm the leader"** in monitoring.
- **DNS / discovery service reporting both nodes as healthy primaries.**
- **Drift in row counts or data across replicas.**

Monitoring should alert on:
- Number of "leaders" or "primaries" claimed in a cluster (should be 1).
- Replication lag spikes.
- Quorum-status changes.
- Diverging versions on the same key.

Most consensus systems publish leader status; cross-checking across nodes detects multi-leader states quickly.

---

## 9. Resolution

When split-brain has happened, options:

### 9.1 Pick a winner
One side becomes canonical. The other's writes are discarded. Painful — lost data. But sometimes necessary.

Used when: one side is much further along, or one side is known compromised.

### 9.2 Merge writes
Attempt to combine writes from both sides. Possible for some data types (sets, counters with vector clocks). Hard or impossible for others (constraints, uniqueness).

Used when: data is naturally CRDT-like, or app supports merging.

### 9.3 Restore from snapshot
Take the latest backup; replay writes from one side. Sacrifices the other side entirely.

### 9.4 Manual reconciliation
Human inspects diffs, decides per-row. Tedious; expensive; sometimes the only option.

The honest answer: **resolution is operational pain**. Prevention is the goal.

---

## 10. Prevention Checklist

### Architecture
- [ ] Use a consensus protocol (Raft, Paxos, ZAB) for any state that must be single-writer.
- [ ] Odd-numbered cluster sizes (3, 5, 7).
- [ ] Require majority quorum for writes and leadership.
- [ ] **Don't use N=2 clusters.** Lose 1 node → no quorum → cluster unavailable.

### Operations
- [ ] Never manually promote a replica without first stopping the old primary.
- [ ] Fencing tokens for shared resources.
- [ ] STONITH or similar forced-stop for failed primaries when feasible.
- [ ] Monitor "number of leaders" as a key metric.
- [ ] Alert immediately on multiple primaries detected.

### Network
- [ ] Multi-AZ clusters where partitions can't isolate a majority.
- [ ] Sufficient redundant network paths.
- [ ] Don't put all nodes in 2 AZs; use 3.

### Application
- [ ] Use CP databases for state requiring strong consistency.
- [ ] Use AP databases with CRDT-like conflict resolution where appropriate.
- [ ] Idempotent writes wherever possible — limits damage of duplicate writes.

### Testing
- [ ] Chaos test partition scenarios.
- [ ] Verify cluster behavior with N-1, N-2 nodes down.
- [ ] Simulate network partitions (Toxiproxy, iptables).

---

## 11. The N=2 Trap

A surprising amount of split-brain comes from 2-node setups.

```
2 nodes: A and B. Both replicas.
A becomes "primary"; B is standby.

A and B can't reach each other:
  Option 1: B becomes primary (split-brain if A still alive on its side).
  Option 2: B refuses to become primary (no failover; outage).

There's no good answer with N=2.
```

The fix: **3 nodes minimum**. Or use 2 nodes + a tie-breaker / witness (lightweight node that just votes). Some systems support this (Postgres with Patroni's witness mode, MongoDB arbiter — though arbiters have their own problems).

**N=2 is unsafe for any system requiring HA.** Always 3.

---

## 12. Worked Example: Postgres HA via Patroni

You run Postgres with Patroni for HA. 3 nodes across 3 AZs. etcd as the coordination store.

### Normal operation
- One Postgres is primary. Others are streaming replicas.
- Patroni writes "I'm the leader" to etcd, with a lease.
- Renews every 5 sec; lease TTL 10 sec.

### Network partition
- Primary AZ becomes isolated from the other two.
- Primary can't renew lease.
- After 10s, lease expires.
- Patroni in another AZ detects no lease → starts election.
- New primary elected (majority of the 3 Patronis).
- DNS / endpoint flipped.

### Old primary recovery
- Old primary returns to network.
- Patroni on old primary sees a new lease exists.
- Steps down voluntarily.
- Becomes replica of new primary.
- Catches up via replication.

### What prevents split-brain
- etcd is the single source of truth for leadership.
- Quorum (2 of 3 etcd nodes) required for leader change.
- Old primary checks etcd before doing anything — and finds it lost the role.
- Fencing via Patroni's voluntary step-down + etcd lease.

What if Patroni on old primary is buggy / Postgres process refuses to stop? STONITH-style intervention (kill -9 the postmaster, or shut down the node) — last resort.

---

## 13. Worked Example: A Kafka Partition Leader

Kafka partition has 3 replicas: leader on broker A, followers on B and C.

### Partition: A is isolated
- A can't reach B or C.
- A's leadership lease expires (controller no longer reachable).
- Controller (ZK or KRaft) elects new leader: B (or C).
- Producers redirect to new leader.

### Old leader A still has clients sending writes
- A appends to its log locally.
- But A can't commit those writes (no quorum among ISRs).
- When A reconnects: A's log diverges from new leader's.
- A truncates its log to match new leader (loses uncommitted writes).
- Becomes follower; catches up.

### What prevents split-brain
- Kafka's ISR (in-sync replicas) mechanism — writes are committed only when ISR confirms.
- A on its own (no other ISRs) cannot commit.
- Controller decides on failover; quorum (ZK or KRaft Raft) required.
- A's "extra" writes are abandoned; producers get an error if they had `acks=all`.

This is Kafka's CP-leaning design. Some writes can be lost (those that didn't reach ISR), but **no writes diverge**.

---

## 14. Anti-Patterns That Cause Split-Brain

- **Manual failover scripts that don't verify cluster state.**
- **DNS-based leader discovery with TTL.** Stale clients keep hitting old leader.
- **Even-numbered clusters.** N=4 → partition 2-2 → no winner.
- **N=2 with auto-failover.** No way to tell partition from real failure.
- **No fencing.** GC pause on old leader → two writers.
- **Independent monitoring agents on each side.** Each thinks it's authoritative.
- **Hand-rolled replication.** Easier to get wrong than use a proven system.
- **Trusting "ping works" as proof of health.** Doesn't mean the application is responsive or the leader is current.

---

## 15. Common Mistakes

- **Running N=2 cluster.** Either no HA or split-brain risk.
- **Even cluster sizes.** Tied votes, no quorum.
- **No fencing.** GC pause produces two leaders briefly.
- **Manual failover without stopping old primary.** Classic disaster.
- **Treating async replication failover as safe.** It isn't — there's always a divergence window.
- **DNS or LB pointing both old and new primary as healthy.** Two leaders receive traffic.
- **No "number of leaders" metric.** Split-brain undetected for hours.
- **Skipping chaos testing.** Partition scenarios surface real bugs.
- **Believing "we never partition."** Yes you do, briefly, often.
- **Postgres async standby with manual failover, no orchestrator.** Painful and risky.

---

## 16. Cheat Card

```
SPLIT-BRAIN     two sides of a partition both accept writes
                divergent state; hard to merge

CAUSES          no quorum requirement, no fencing,
                even/2-node clusters, manual failover errors,
                trusting async replication for HA

PREVENT
  consensus protocol (Raft/Paxos/ZAB)
  odd cluster size (3, 5, 7)
  majority quorum required
  fencing tokens on shared resources
  STONITH or voluntary step-down on lease loss
  single source of truth (etcd, Zookeeper) for leader role
  multi-AZ + spread for clear majority

DETECT          monitor "leaders count" = 1
                alert on dual-primary
                replication lag + divergence

RESOLVE         pick a winner (lose other side's writes)
                merge if data type allows (CRDT-like)
                manual reconciliation (painful)
                restore from snapshot (data loss)

CAP             classic CP design: refuse, don't divide
                AP done right: design for divergence + merge

N=2             DANGER. Either no HA or split-brain risk.
                Use 3 nodes or 2+witness.

PITFALLS        even sizes, no fencing, manual failover,
                async replication HA, DNS confusion,
                no chaos testing

RULE            Prevention via quorum + fencing.
                Detection via "leader count" metric.
                Resolution is always painful — don't get there.
```

---

## 17. Resources

### Books
- *Designing Data-Intensive Applications* — Kleppmann (covers split-brain and partial failures extensively).
- *Site Reliability Engineering* — Google SRE Book.

### Articles
- "GitHub's October 21 incident report" — GitHub blog (MySQL Orchestrator split-brain).
- "How to do distributed locking" — Martin Kleppmann (fencing tokens).
- "Postgres HA with Patroni" — Cybertec Postgres.
- "MongoDB replica set failover" — MongoDB docs.
- "Avoiding split-brain in Pacemaker clusters" — Linux-HA docs.

### Papers
- "The Byzantine Generals Problem" — Lamport (the source of much consensus theory).
- "In Search of an Understandable Consensus Algorithm" — Raft paper.

### Videos
- ByteByteGo — "Split-Brain Problem".
- Kyle Kingsbury / Jepsen talks on real DB failures.

### Tools
- **Patroni** — Postgres HA with leader election via etcd/Zookeeper.
- **Pacemaker + Corosync** — Linux HA cluster manager.
- **Repmgr** — Postgres replication manager.
- **Toxiproxy** — simulate network partitions for testing.
- **Jepsen** — formal partition / failure testing.

### Adjacent reading
- [Consensus →](./consensus.md)
- [Leader Election →](./leader-election.md)
- [Quorum-Based Replication →](./quorum.md)
- [Distributed Locks →](./distributed-locks.md)
- [CAP Theorem →](./cap-theorem.md)
- [PACELC →](./pacelc.md)
- [Replication →](../04-databases/replication.md)
- [Failover & Disaster Recovery →](../11-reliability/failover-dr.md)
- [Chaos Engineering →](../11-reliability/chaos-engineering.md)

---

*Previous:* [← Byzantine Fault Tolerance](./bft.md)  |  *Up:* [README ↑](../README.md)
