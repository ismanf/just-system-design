# Leader Election

> **TL;DR** — **Leader election** is the process by which a cluster of nodes agrees that exactly one of them is the **leader** — the node responsible for coordinating writes, serializing decisions, or handling tasks that can't be safely done by multiple nodes simultaneously. Leaders are everywhere: replicated databases (Postgres primary, Cassandra coordinator), distributed KV stores (etcd, Zookeeper), schedulers (Kubernetes controllers), partitioned services (Kafka partition leader), and almost any system that needs a single point of authority within a group. The two main approaches: **consensus-based** (Raft, Paxos, ZAB — the modern default, used by etcd, Zookeeper, Consul) — provably correct, requires quorum; and **lease-based** (renew a lease in a shared store) — simpler but relies on clock assumptions. The hard problems are **avoiding two leaders** (split-brain), **bounding leader failover time** (seconds matter), and **ensuring fencing** (an old leader recovers and thinks it's still leader — must be stopped from acting).

---

## 1. Why You Need a Leader

A cluster of N peers can act independently for most things. But some operations require **single-writer semantics**:

- **Linearizable writes** to a replicated state machine.
- **Coordinator** of distributed transactions.
- **Scheduler** that assigns jobs.
- **Owner** of a shard or partition.
- **Failure detector** acting on a peer.
- **Garbage collector** of a shared resource.

If two nodes simultaneously try to do these, you get conflicts, duplicate work, or corruption.

A **leader** is a node currently designated to perform these single-writer duties on behalf of the group. Everyone else (followers) defers to it.

---

## 2. What Makes Leader Election Hard

Two desiderata, in tension:

- **Safety**: never two leaders at the same time.
- **Liveness**: a new leader is elected promptly after the old one fails.

If you can't tell whether the old leader is dead or just slow (the **partial failure** problem), you can't safely elect a new one without risking two leaders.

The textbook constraint: **safety must always hold; liveness can be compromised during pathological conditions.** This means under bad partitions, the cluster may temporarily have no leader rather than two.

---

## 3. Approaches

### 3.1 Consensus-Based Election (Raft, Paxos, ZAB)
Use a consensus algorithm. The leader is whoever wins a majority vote.

```
1. Follower detects no heartbeats from leader.
2. Becomes candidate; increments term.
3. Requests votes from peers.
4. If majority votes for it → leader.
5. Sends heartbeats to maintain leadership.
```

Used by: etcd, Consul, Zookeeper (via ZAB), CockroachDB (per shard), Kafka KRaft, MongoDB replica sets, Kubernetes (via etcd).

**Pros**
- Provably correct (with the algorithm's assumptions).
- Tolerates minority failures.
- Built into proven libraries.

**Cons**
- Requires quorum — minority side can't elect.
- Latency: leader election takes 150 ms – 1 sec typically.
- Complex if you implement it yourself.

This is the **default for serious systems**. See [Consensus →](./consensus.md).

### 3.2 Lease-Based Election
The "leader" is whoever holds a lease (a timed lock) in some shared store. Renewing the lease keeps you leader.

```
SET leader_lock = my-id NX EX 30   # Redis: set if not exists, 30s TTL
# every 10s: extend the lease

if lease expires → another node can grab it
```

Used by: Kubernetes' "leader election" library (writes to a ConfigMap / Lease object in etcd), various background-job workers, Sidekiq Enterprise.

**Pros**
- Simple to implement.
- Doesn't require running a consensus cluster.
- Works with any shared store that supports atomic operations.

**Cons**
- Relies on **clocks** — if a node's clock pauses (GC, kernel preemption), it may think it still holds the lease while another node took over.
- Needs **fencing tokens** (see below).
- Often built atop a consensus store underneath (Kubernetes leases live in etcd).

Lease-based works *if* you implement fencing. Without it, you have a subtle correctness hole.

### 3.3 Bully Algorithm
A classic from textbook distributed systems: each node has a priority/ID. On detection of leader failure, nodes with higher IDs "bully" lower ones into recognizing them.

Rarely used in production now; consensus is better.

### 3.4 Static Configuration
Hardcode the leader. Simplest possible. No election — just config. Failover requires human intervention.

Used for: simple master-replica DBs in legacy setups.

---

## 4. Fencing: The Critical Detail

The classic correctness bug:

```
node A is leader, holds lease until t=30
at t=29: node A starts a stop-the-world GC pause (10s)
at t=31: lease expires (A doesn't know — paused)
at t=32: node B acquires lease, becomes leader
at t=33: B does work, writes to DB
at t=39: A resumes GC; doesn't know it lost lease
at t=39+: A also tries to write to DB
```

A and B both think they're leader. Both write. Corruption.

**Fencing** prevents this: every operation A wants to perform requires presenting a **token** that the resource can validate.

```
when A acquires lease: gets token "10"
when B acquires lease later: gets token "11"

A writes with token 10
DB compares: latest seen token is 11 → reject A's write
```

The fencing token is a monotonically increasing number. Resources track the highest token they've seen and reject older ones. This is the **only safe** way to do lease-based leader election.

Martin Kleppmann's famous blog post: "How to do distributed locking" explains this in detail.

---

## 5. Real-World Examples

### Kubernetes
Components like the controller-manager and scheduler use the **leader-election library** (`k8s.io/client-go/tools/leaderelection`). It writes to a `Lease` object in etcd. Multiple replicas of a controller can run; only the leader does work.

```yaml
apiVersion: coordination.k8s.io/v1
kind: Lease
metadata:
  name: my-controller
spec:
  holderIdentity: "controller-pod-abc"
  leaseDurationSeconds: 15
  renewTime: "2026-05-19T14:00:00Z"
```

If the lease isn't renewed in 15s, another pod grabs it.

### Postgres
Single primary, multiple standbys. Failover via:
- Manual (`pg_ctl promote`).
- Patroni / repmgr / Stolon — these use Zookeeper / etcd / Consul for consensus-backed election.

### Redis
- **Sentinel**: separate cluster monitors primary; on failure, Sentinel quorum elects a new primary.
- **Cluster mode**: per-shard primary; cluster gossip + voting elects new primary.

### Kafka
Each partition has a leader. ZooKeeper (old) / KRaft (new) coordinates election.

### Etcd / Zookeeper / Consul
The consensus cluster IS the election mechanism. Other services use it as a coordination substrate.

### MongoDB
Replica sets do Raft-based election when the primary fails. Takes ~10–30 seconds.

---

## 6. How Long Does Election Take?

Raft / etcd-style: ~150–300 ms for election timeout + ~one RTT for vote collection. Practical leader change: 0.3–1 sec under good conditions.

Lease-based: lease duration + retry interval. Often 5–30 sec.

Sentinel: tens of seconds for failure detection + election + DNS / discovery update.

The hard part isn't election speed; it's **detecting that the old leader is dead**. Heartbeat-based detection has a fundamental trade-off:
- Short intervals → fast detection, more network traffic, more false positives.
- Long intervals → slow detection, less traffic, more lag during real failure.

Most systems land at 5–30 sec for production. Hyperscalers tune lower with sophisticated detection.

---

## 7. Split-Brain: The Worst Outcome

Two leaders acting simultaneously. Each accepts writes. State diverges. Reconciliation is painful or impossible.

Causes:
- Network partition + no quorum check.
- Slow GC pause + no fencing.
- Misconfigured cluster (e.g., even-size, no quorum).
- Manual intervention promoting a replica without demoting old primary.

Avoidance:
- **Always require quorum** for leadership.
- **Always use fencing tokens** for shared resources.
- **Always run odd-sized clusters** (3, 5, 7) for clean majority.
- **Standby fencing**: when promoting a new primary, ensure the old one cannot still write (STONITH-style: "shoot the other node in the head").

See [Split-Brain →](./split-brain.md).

---

## 8. Leader Lease Heart-Beat Tuning

The two timeouts:
- **Lease duration** — how long the leader's lease is valid without renewal.
- **Renewal interval** — how often the leader extends it.

Renewal interval should be **< lease duration / 2** to allow for retries and clock skew.

Example (Kubernetes default):
- Lease duration: 15s.
- Renewal interval: 10s.
- Retry on failure to renew.

If renewal fails consistently, the leader steps down. Another pod acquires the (now-expired) lease.

---

## 9. Multi-Leader Patterns

Sometimes you don't want a single leader. Alternatives:

### Per-shard leaders
Each partition has its own leader. Spread leadership across nodes. Used by Cassandra, CockroachDB, Kafka.

```
shard 0 leader: node A
shard 1 leader: node B
shard 2 leader: node C
```

Failure of A only affects shard 0. Each node leads ~1/N of shards in steady state.

### Multi-master replication
Every node accepts writes; conflicts resolved via CRDT / vector clocks. No "leader" in the classical sense. See [CRDTs →](./crdts.md).

### Leaderless replication (Dynamo-style)
Cassandra, DynamoDB, Riak: no leader; any replica accepts writes; coordination via quorums on read/write.

These are alternatives to leader election — for workloads that can tolerate the consistency trade-offs.

---

## 10. Worked Example: Background Job Scheduler

You have 3 worker pods. You want exactly one to run a periodic cleanup task every hour. Others sleep.

### Naive: cron in each pod
All three pods run the cleanup. Duplicates. Wasted work or worse, corruption.

### Naive: one designated pod
Hardcoded. If that pod dies, cleanup stops. No failover.

### Leader election via Kubernetes Lease
```go
import "k8s.io/client-go/tools/leaderelection"

leaderelection.RunOrDie(ctx, leaderelection.LeaderElectionConfig{
    Lock: &resourcelock.LeaseLock{
        LeaseMeta: metav1.ObjectMeta{Name: "cleanup-leader", Namespace: "default"},
        Client: client.CoordinationV1(),
        LockConfig: resourcelock.ResourceLockConfig{Identity: podName},
    },
    LeaseDuration: 15 * time.Second,
    RenewDeadline: 10 * time.Second,
    RetryPeriod:   2 * time.Second,
    Callbacks: leaderelection.LeaderCallbacks{
        OnStartedLeading: func(ctx context.Context) {
            startCleanupCron()  // only the leader runs this
        },
        OnStoppedLeading: func() {
            stopCleanupCron()
        },
    },
})
```

Behavior:
- All 3 pods race to acquire the lease.
- One wins; the others wait.
- The leader runs `cleanupCron`.
- If the leader pod dies, lease expires; another grabs it within ~15 sec.

For the cleanup operation itself, since it touches the DB, also include a fencing check: store the lease's `resourceVersion` and pass it to any operation that could be racey.

---

## 11. Common Mistakes

- **No quorum requirement.** Two leaders possible during partition.
- **Lease-based without fencing.** Clock pauses → split-brain.
- **Even-numbered cluster.** Tied votes, indecision.
- **N=2 cluster.** Lose one node, total outage.
- **Heartbeat-only detection without grace.** Network blip = unnecessary failover.
- **Leadership not respected by callers.** Followers also write because the logic isn't centralized.
- **No leader-step-down on stale lease.** Old leader keeps acting.
- **Custom election protocol.** Use a library.
- **Election that takes minutes.** Half your SLA, gone.
- **No alert on prolonged "no leader" state.** Cluster effectively down, nobody notices.

---

## 12. Cheat Card

```
LEADER         the single node currently coordinating writes / decisions

WHY            single-writer semantics: writes to a state machine,
                schedulers, partition owners, distributed locks

APPROACHES
  consensus-based (Raft/Paxos)  modern default; provably safe
                                  used by etcd, K8s, Cassandra, CockroachDB
  lease-based                    simpler; needs fencing
                                  used by K8s leader election, Sidekiq
  static                         hardcode + manual failover
  bully                          textbook; rarely production

QUORUM         majority required to elect; minority side has no leader

FENCING        monotonic token presented on every operation
                resource rejects stale tokens
                MANDATORY for lease-based correctness

CLUSTER SIZE   odd numbers (3, 5, 7); never 2

ELECTION TIME  consensus: 150 ms – 1 s
                lease: 5–30 s
                Sentinel: tens of seconds

SPLIT-BRAIN    avoid via quorum + fencing + odd cluster size

MULTI-LEADER   per-shard leaders for scale (Cassandra, CockroachDB)
                multi-master for availability (Dynamo, CRDTs)

PITFALLS       no fencing, even cluster size, custom protocol,
                no quorum check, lease without GC tolerance,
                forgotten step-down

RULE           Use a library. Use a quorum-based protocol.
                Always fence. Test failover.
```

---

## 13. Resources

### Papers
- "Paxos Made Simple" — Leslie Lamport, 2001.
- "In Search of an Understandable Consensus Algorithm" — Ongaro & Ousterhout, 2014 (Raft).
- "ZAB: A Simple Totally Ordered Broadcast Protocol" — Reed & Junqueira.

### Books
- *Designing Data-Intensive Applications* — Kleppmann (the consensus chapter).
- *Database Internals* — Petrov.

### Articles
- "How to do distributed locking" — Martin Kleppmann (fencing tokens).
- "Is Redlock safe?" — antirez response.
- "Leader election in Kubernetes" — Kubernetes blog.
- "Raft visualization" — http://thesecretlivesofdata.com/raft/.

### Videos
- Diego Ongaro — Raft Stanford lecture.
- ByteByteGo — "Leader Election Explained".

### Tools
- **etcd**, **Consul**, **Zookeeper** for consensus-backed coordination.
- **Kubernetes client-go leader-elector**.
- **hashicorp/raft**, **etcd/raft**, **tikv/raft-rs** for embedding consensus.

### Adjacent reading
- [Consensus →](./consensus.md)
- [Distributed Locks →](./distributed-locks.md)
- [Split-Brain Problem →](./split-brain.md)
- [Quorum-Based Replication →](./quorum.md)
- [Clocks →](./clocks.md)
- [Failover & Disaster Recovery →](../11-reliability/failover-dr.md)

---

*Previous:* [← Consensus](./consensus.md)  |  *Next:* [Distributed Locks →](./distributed-locks.md)
