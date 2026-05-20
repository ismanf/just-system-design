# Consensus Algorithms (Paxos, Raft, ZAB)

> **TL;DR** — A **consensus algorithm** lets a set of distributed processes agree on a single value (or sequence of values) even when some of them fail or messages are delayed. Consensus is the foundation of every system that needs **leader election**, **strong consistency**, **distributed locks**, **distributed databases**, or **state-machine replication**. The three canonical algorithms: **Paxos** (Leslie Lamport, 1989) — mathematically elegant, brutally hard to understand, the granddaddy; **Raft** (Ongaro & Ousterhout, 2014) — designed for understandability, dominates modern systems (etcd, Consul, CockroachDB, TiKV); **ZAB** (Zookeeper Atomic Broadcast) — bespoke variant powering Zookeeper. All require a **majority** (quorum) of nodes to be alive and communicating, so they're **CP** under CAP. The FLP impossibility theorem says you can't have all three of safety, liveness, and asynchrony — real systems pick eventual liveness via timeouts. Understand Raft well; reach for Paxos when you must; use ZAB only via Zookeeper.

---

## 1. The Problem

A set of N processes. Each has a value or wants to decide one. Some may crash. Messages may be delayed or reordered. The network may partition briefly.

**Goal**: all non-failing processes agree on the same value (or sequence of values).

Sounds simple. The famous **FLP impossibility theorem** (Fischer-Lynch-Paterson, 1985) proves you cannot guarantee consensus in a fully asynchronous network where even one process can crash — you can have **safety** (no two correct processes disagree) or **liveness** (eventually a decision is reached), but not both with certainty.

Real systems sidestep FLP by:
- Using timeouts (assume timely-enough network).
- Sacrificing strict asynchrony.
- Failing to make progress during pathological partitions (acceptable — better to wait than lie).

---

## 2. What Consensus Buys You

A consensus protocol lets you build:

- **A replicated state machine** — every node applies operations in the same order, so the state is consistent.
- **Leader election** — exactly one leader at a time.
- **Distributed lock** — agree on who holds the lock.
- **Configuration** — Kubernetes, Consul, Zookeeper all store config via consensus.
- **Distributed transactions** — consensus on commit/abort.

The pattern: any time you need "everyone agrees on X across N machines, despite failures," you reach for consensus.

---

## 3. Quorum: The Core Mechanic

Consensus algorithms require a **majority** (quorum) of nodes to be reachable.

```
N=3 nodes → quorum = 2
N=5 nodes → quorum = 3
N=7 nodes → quorum = 4
N nodes → quorum = floor(N/2) + 1
```

Why majority? Because any two majorities **must overlap** in at least one node. That overlap is how new leaders learn what previous leaders committed.

```
                old quorum: {A, B, C}
                new quorum: {C, D, E}
                overlap:    {C}      ← C knows the previous decision
```

If you used a minority, two disjoint groups could both make decisions; no overlap, no agreement.

For N=2, quorum=2 — both must be up to make progress. Lose one and the cluster is unavailable. **Always run an odd number** (3, 5, 7) to balance fault tolerance and quorum size.

---

## 4. Paxos

Leslie Lamport's 1989 invention (paper "The Part-Time Parliament" published 1998). The foundational consensus algorithm. Mathematically rigorous; legendarily hard to implement correctly.

### Roles
- **Proposer** — proposes a value.
- **Acceptor** — votes on proposals.
- **Learner** — learns the decided value.

In practice, every node is all three. Roles are conceptual.

### Phases
1. **Prepare** — proposer picks a number `n`, sends `PREPARE(n)` to acceptors. Acceptors promise not to accept proposals < n, and reply with the highest-numbered proposal they've already accepted.
2. **Accept** — proposer sees responses. If any acceptor reported a previous proposal, the proposer reuses that value. Sends `ACCEPT(n, value)` to acceptors.
3. **Decide** — when a majority of acceptors accept, the value is decided. Learners are informed.

### Why it's hard
- The paper is famously cryptic ("a Greek parliament" allegory).
- Edge cases: dueling proposers can prevent progress.
- "Multi-Paxos" (the practical version) is a separate problem.
- Implementations are error-prone (Google's Paxos code is famously complex).

### Multi-Paxos
Single-decree Paxos chooses one value. Real systems need to choose **a sequence of values** (a log). Multi-Paxos: elect a stable leader, who proposes values without the full prepare phase (steady state is just accept).

### Real-world Paxos
- **Google Chubby** (lock service): Paxos.
- **Google Spanner**: Paxos per shard.
- **Cassandra Lightweight Transactions**: a Paxos variant.

### When to use Paxos directly
Almost never. Use Raft or a library that has already done Paxos right. Paxos is the topic of academic papers; Raft is the topic of production systems.

---

## 5. Raft

Diego Ongaro and John Ousterhout, 2014. Paper title: "In Search of an Understandable Consensus Algorithm." Mission accomplished.

Raft makes consensus comprehensible by separating it into three subproblems:
1. **Leader election**.
2. **Log replication**.
3. **Safety** (constraints on who can become leader).

### Roles
- **Follower** — default state. Listens.
- **Candidate** — running for leader.
- **Leader** — handles all client requests; replicates log to followers.

```
   [Follower] ── timeout ──► [Candidate] ── majority votes ──► [Leader]
       ▲                          │                                │
       └──────  higher term  ──── │                                │
                                                                  └── heartbeats keep leader role
```

### Terms
Time is divided into **terms** (logical, monotonically increasing). Each term has at most one leader. A new election begins a new term.

### Leader election
- A follower hasn't heard from a leader in `election_timeout` (randomized 150–300 ms typical).
- It becomes a candidate, increments its term, votes for itself, requests votes from peers.
- If it gets majority votes (and its log is up-to-date) → leader.
- Else: election timeout fires again, term increments, retry.

Randomized timeouts avoid split votes (where 2+ candidates split).

### Log replication
The leader receives client commands and appends them to its log. It sends `AppendEntries` RPC to followers in parallel. Once a majority has the entry replicated, the leader **commits** it and applies to its state machine. Then it returns to the client.

```
client → leader
   leader appends to its log
   leader sends AppendEntries to followers
   followers append + ack
   when majority acks → leader commits, applies
   returns success to client
   next heartbeat tells followers to commit
```

### Safety
Raft's most important invariant: **a leader's log is the source of truth**. New leaders must have **all committed entries** in their log. This is enforced by:
- Only candidates whose log is at least as up-to-date as the voter's can win the vote.
- A leader never overwrites or deletes entries; only appends.

This eliminates a whole class of bugs that plagued Paxos implementations.

### Where Raft runs in production
- **etcd** (Kubernetes' config store).
- **Consul**.
- **CockroachDB** — per-range Raft groups.
- **TiKV / TiDB** — Raft.
- **MongoDB** (since 4.0) — Raft for replica set elections.
- **Redis Sentinel** uses a Raft-like protocol.
- **Kafka KRaft** (replacing ZooKeeper, GA in 3.3).
- **Apache Cassandra** (since 4.1, Accord transactions use a variant).

If you need consensus in 2026, you almost certainly want Raft.

---

## 6. ZAB — ZooKeeper Atomic Broadcast

ZAB is the consensus protocol inside Apache Zookeeper. Designed for primary-backup state-machine replication.

### Differences from Raft
- ZAB has explicit phases for **discovery**, **synchronization**, and **broadcast**.
- Designed for total-order broadcast specifically — Zookeeper's API guarantees a sequential consistency for clients.
- The leader proposes; followers ack; majority ack triggers commit.

In practice, ZAB and Raft are similar at the high level: a leader, terms (called "epochs" in ZAB), majority quorum, log replication.

ZAB is bespoke; you encounter it only via Zookeeper. New systems use Raft.

---

## 7. The FLP Impossibility (and How We Cope)

Fischer, Lynch, Paterson (1985): **In an asynchronous distributed system with even one crash-prone process, no deterministic consensus algorithm can guarantee both safety and liveness.**

Either:
- The algorithm can hang forever (no liveness), or
- It can produce wrong answers (no safety).

The result is profound but not paralyzing. Real systems sacrifice **strict asynchrony**:
- Add timeouts. If timeout expires, treat as failure.
- Accept that during pathological network conditions, progress halts (liveness violated) — but safety holds.

Raft and Paxos guarantee **safety always** (never inconsistent) and **liveness in practice** (assuming bounded message delays).

This is why etcd / Spanner / Consul can stall during partitions but never disagree on what was committed. CP under CAP.

---

## 8. Comparison Table

| Aspect | Paxos | Raft | ZAB |
|---|---|---|---|
| Year | 1989 / 1998 | 2014 | 2007 |
| Author | Leslie Lamport | Ongaro, Ousterhout | Reed, Junqueira |
| Goal | Theoretical foundation | Understandability | Zookeeper-specific |
| Safety | Yes | Yes | Yes |
| Liveness | With timeouts | With timeouts | With timeouts |
| Quorum | Majority | Majority | Majority |
| Roles | Proposer, Acceptor, Learner | Follower, Candidate, Leader | Leader, Follower, Observer |
| Multi-Paxos / Multi-Raft | Sequence via Multi-Paxos | Native | Native |
| Industrial use | Spanner, Chubby, Cassandra LWT | Etcd, Consul, CockroachDB, Kafka KRaft, Mongo | Zookeeper |
| Implementation complexity | Very high | Moderate | High |
| Read difficulty | Notoriously hard | Approachable | Moderate |

---

## 9. Practical Concerns

### Cluster size
- N=3 tolerates 1 failure. Default for most production clusters.
- N=5 tolerates 2 failures. For higher availability or geo-distribution.
- N=7+ rare. Costs more quorum coordination per write.

Even numbers are wasteful (N=4 still tolerates 1, like N=3, but needs 3 of 4 to commit).

### Latency
Every write needs a round trip to a majority. If members are in different regions:
- Single AZ: < 1 ms.
- Same region multi-AZ: ~1 ms.
- Cross-region: 50–200 ms.
- Global: 100+ ms.

This is why Spanner uses TrueTime and Paxos commit-wait — to amortize the global RTT.

### Throughput
A single Raft group's throughput is bounded by:
- Leader's write speed.
- Followers' ack latency.
- Network bandwidth.

For more throughput, **shard** across multiple Raft groups (Multi-Raft):
- CockroachDB / TiKV: each key range has its own Raft group.
- Spanner: similar.
- Cassandra Accord (recent): per-shard.

Multi-Raft is how you scale consensus past one-leader's throughput.

### Snapshotting
Raft logs grow forever; eventually you must snapshot:
- Take a checkpoint of the state machine.
- Truncate the log up to the snapshot.
- Lagging followers may receive a snapshot transfer instead of replaying from log.

Bad snapshotting → unbounded log → disks fill.

### Leader stability
Frequent re-elections kill throughput. Tune timeouts:
- Network typical RTT × 10 = election_timeout.
- Heartbeat interval = election_timeout / 10.
- Randomize election_timeout to avoid split votes.

Stable network: leader stays leader for hours/days. Flapping network: elections constantly; throughput collapses.

---

## 10. Worked Example: Building a Replicated KV Store

Let's say you want a 3-node KV store with linearizable reads/writes.

### Architecture
- 3 Raft nodes (A, B, C).
- One is leader at a time.
- Each holds a log + state machine.
- Clients connect to any node; non-leader forwards to leader (or sends `MOVED` response).

### Write path
```
1. Client → leader: SET key=42 value=100
2. Leader appends to its log.
3. Leader sends AppendEntries to A and B's followers.
4. When majority (leader + 1 follower) confirms, leader commits.
5. Leader applies to state machine: kv[42] = 100.
6. Leader responds OK to client.
7. Next heartbeat tells followers to commit.
```

### Read path (linearizable)
```
1. Client → leader: GET key=42
2. Leader confirms it's still leader (heartbeat round-trip to majority).
3. Leader returns the value from its state machine.
```

Alternative: follower reads with a "leader-lease" mechanism for performance. Most systems do this.

### Failure modes
- **Leader dies**: followers election timeout fires; election begins; new leader elected within ~150–300 ms.
- **Network partition**: minority side becomes unable to elect/serve. Majority continues.
- **Slow follower**: leader retries AppendEntries; eventually catches up or snapshot transfer.
- **All three down**: total outage. Cluster cannot recover until majority comes back.

### What you don't write
- The Raft library (use `hashicorp/raft`, `etcd/raft`, `tikv/raft-rs`).
- The membership-change protocol (libraries handle it).
- The leader-lease mechanism (libraries handle it).

You write the **state-machine logic** (apply commands), **command serialization**, and **API**.

---

## 11. Membership Changes

A common subtlety: how do you change the cluster's members (add/remove a node) without losing safety?

Raft's solution: **joint consensus** — temporarily require quorum from both old and new configs. The cluster transitions through a hybrid state.

Etcd / Consul / Cockroach all implement this. Don't reinvent it.

Bad implementations of membership change have caused real outages (split-brain on bad reconfig).

---

## 12. Witnesses and Reduced Quorums

Some setups use:
- 2 voting members in different AZs.
- 1 lightweight "witness" (just for quorum tie-breaking).

Used by Azure Cosmos DB, some MongoDB setups. Trade: cheaper than 3 full replicas, slightly more brittle.

---

## 13. Common Mistakes

- **Implementing Paxos from scratch.** Don't.
- **Implementing Raft from scratch.** Use a library.
- **N=2 cluster.** Quorum=2; lose one node, total outage. Use 3 or 5.
- **Even-numbered cluster.** Wastes a vote. Use 3, 5, 7.
- **Cross-region without latency planning.** Every write is RTT.
- **No snapshotting.** Log grows forever.
- **Frequent leader elections.** Tune timeouts; flapping kills throughput.
- **Linearizable reads via stale follower.** Reads from non-leader return stale data unless lease/quorum-read used.
- **Assuming consensus = "always available".** It's CP. Minority side stops.
- **Thinking consensus solves all distributed problems.** It solves agreement; you still need partitioning, sharding, replication strategies above it.

---

## 14. Cheat Card

```
CONSENSUS        agree on a value/sequence despite failures and delays

QUORUM           majority (N/2 + 1); ensures any two quorums overlap

PAXOS            1989 Lamport; foundational; hard to implement
                  use via Spanner, Chubby, Cassandra LWT
                  almost never roll your own

RAFT             2014; understandable; modern default
                  used by etcd, Consul, CockroachDB, TiKV,
                  MongoDB, Kafka KRaft
                  always reach for this

ZAB              ZooKeeper's protocol; bespoke
                  encountered only via Zookeeper

KEY PROPS
  safety         never disagree on committed value (always)
  liveness       eventually decide (with timeouts)
  CAP            CP — minority side stops

CLUSTER SIZE     3 (default), 5 (HA), 7+ (rare)
                  always odd

THROUGHPUT       one Raft group bounded by leader+RTT
                  shard via Multi-Raft for scale

FLP              with crashes + async, can't have safety+liveness+async
                  real systems use timeouts; sacrifice strict async

USES             leader election, distributed locks, replicated state machines,
                  distributed transactions, config stores

PITFALLS         even cluster size, no snapshots, N=2,
                  cross-region without latency awareness,
                  rolling your own Paxos/Raft,
                  stale reads from followers

RULE             Don't write Paxos. Use Raft. Use a library.
                  Plan for the minority side to be unavailable.
```

---

## 15. Resources

### Papers
- "The Part-Time Parliament" — Leslie Lamport, 1998 (Paxos).
- "Paxos Made Simple" — Lamport, 2001 (less impenetrable).
- "In Search of an Understandable Consensus Algorithm" — Ongaro & Ousterhout, 2014 (Raft).
- "ZooKeeper: Wait-free coordination" — Hunt et al., 2010.
- "Impossibility of Distributed Consensus with One Faulty Process" — Fischer, Lynch, Paterson, 1985 (FLP).

### Books
- *Designing Data-Intensive Applications* — Kleppmann (Ch 9: Consistency and Consensus).
- *Database Internals* — Petrov.

### Articles
- "Raft Visualization" — http://thesecretlivesofdata.com/raft/ (best teaching tool).
- "Raft Consensus Algorithm" — raft.github.io.
- "Paxos Made Live" — Google's experience implementing Paxos.

### Videos
- John Ousterhout — "Designing for Understandability: The Raft Consensus Algorithm".
- Diego Ongaro — Raft Stanford lecture.
- ByteByteGo — "Paxos vs Raft Explained".

### Tools
- **etcd**, **Consul**, **Zookeeper**.
- **hashicorp/raft** (Go), **etcd/raft** (Go), **tikv/raft-rs** (Rust), **Apache Ratis** (Java).
- **Apache BookKeeper** for replicated logs.

### Adjacent reading
- [CAP Theorem →](./cap-theorem.md)
- [Consistency Models →](./consistency-models.md)
- [Leader Election →](./leader-election.md)
- [Quorum-Based Replication →](./quorum.md)
- [Distributed Locks →](./distributed-locks.md)
- [Two-Phase Commit →](./2pc-3pc.md)
- [Split-Brain →](./split-brain.md)

---

*Previous:* [← Consistency Models](./consistency-models.md)  |  *Next:* [Leader Election →](./leader-election.md)
