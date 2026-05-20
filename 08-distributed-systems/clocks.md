# Clocks (Logical, Vector, Hybrid Logical Clocks)

> **TL;DR** — Wall-clock time (NTP, system clock) is **untrustworthy** in distributed systems — clocks drift, jump, lie, and disagree across nodes. To order events correctly, you need **logical clocks** that track **causality** rather than time. **Lamport clocks** are integer counters that establish a partial order — "if A could have caused B, A's clock < B's clock." **Vector clocks** extend this with one counter per node, enabling detection of **concurrent** events. **Hybrid Logical Clocks (HLC)** combine wall-clock time with a logical counter so timestamps are mostly meaningful for humans AND correct for causality. Google's **TrueTime** (Spanner) goes a different direction — bounds the uncertainty in physical time using atomic clocks and GPS — paying for clock infrastructure to get globally-ordered transactions. The choice depends on what you need: causality only (vector / Lamport), human-readable + causal (HLC), or strict total order with global consistency (TrueTime-style).

---

## 1. Why Physical Clocks Fail in Distributed Systems

Every machine has a clock. They're all wrong to varying degrees.

### Sources of error
- **Quartz drift** — typical server clock drifts ~10 ppm = ~1 sec/day untreated.
- **NTP corrections** — periodic syncs to NTP servers. Can jump forward or backward.
- **Leap seconds** — astronomical adjustments. The 2012 Linux leap second bug crashed major services worldwide.
- **VM clock jitter** — virtualized clocks can pause when the host is busy.
- **Clock skew across nodes** — typical ~10 ms, can be much worse.

### Famous failure mode
```
node A's clock: 14:00:00.500
node B's clock: 13:59:59.700

at "A's 14:00:00.500": A writes X=1
at "B's 14:00:00.300" (actually after A's write): B writes X=2

if you order by wall-clock timestamp:
  X=2 looks like it happened BEFORE X=1
  but the causality is the other direction
```

If you depend on wall-clock ordering for correctness — last-write-wins, log timestamps, transaction order — you have a correctness bug waiting for clock skew.

**Don't trust wall clocks for ordering.** They're fine for human-readable display; bad for ordering decisions.

---

## 2. The Concept of Causality

In a distributed system, two events can be:
- **Causally ordered**: A happened-before B. Either same node in order, or A sent a message that B received before doing its work.
- **Concurrent**: neither caused the other. No causal link.

Lamport's **happens-before** relation `→`:
- Within a process: events ordered by program order.
- Across processes: send(m) → receive(m).
- Transitive: if A → B and B → C, then A → C.

Two events with no `→` between them are **concurrent**. They have no defined order.

Logical clocks track this **happens-before** relation without depending on wall clocks.

---

## 3. Lamport Clocks

The classic. Each node has a counter `C`. Rules:
1. Before any local event: `C = C + 1`.
2. When sending a message, attach `C`.
3. When receiving a message with timestamp `T`: `C = max(C, T) + 1`.

```
node A: events 1, 2, 3
node B: events 1, 2, 3
A's event 2 sends message to B's event 2

A: 1 → 2 → 3
        ↓ msg with ts=2
B: 1 → max(1,2)+1=3 → 4
```

After this, B's local event 2 has Lamport timestamp 3 — strictly greater than A's event 2.

### Property
- If A → B (causally), then `LC(A) < LC(B)`.
- The converse is **not** true: `LC(A) < LC(B)` doesn't imply A → B; they could be concurrent.

### Use
Lamport clocks give a **partial order** of events. They tell you "if A's timestamp is less than B's, A might have happened first or they're concurrent — but B definitely didn't cause A."

For some applications (logging, fairness, total-order broadcast with ties broken by node ID) this is enough.

### Limitations
- Can't detect concurrency. Two unrelated events get arbitrary order.
- Counter is just an integer — no human meaning.

---

## 4. Vector Clocks

Track per-node counters. If you have N nodes, each maintains an N-element vector.

Rules:
1. Before any local event: `V[self] = V[self] + 1`.
2. When sending: attach the full vector.
3. When receiving vector `W`: `V[i] = max(V[i], W[i])` for all i; then `V[self] += 1`.

### Comparing vector clocks
- `V < W` if `V[i] ≤ W[i]` for all i, and `V[j] < W[j]` for some j.
- `V = W` if equal in all components.
- Else they're **concurrent**.

```
3 nodes: A, B, C
each has a vector [a_count, b_count, c_count]

A event 1: A's vector [1,0,0]
A sends to B
B event 1 (recv): vector [1,1,0]
B event 2: vector [1,2,0]
C event 1 (independent): vector [0,0,1]

now compare:
  B [1,2,0]   vs   C [0,0,1]
  not ≤ either way → CONCURRENT
```

### Property
- A → B iff `V(A) < V(B)`.
- A || B (concurrent) iff neither `V(A) < V(B)` nor `V(B) < V(A)`.

Vector clocks **detect concurrency**. That's their superpower.

### Use
- Detect concurrent writes in eventually-consistent stores. Riak / Dynamo style: when two updates have concurrent vector clocks, the system has a conflict to resolve.
- Implement causal consistency.
- Detect lost updates.

### Limitations
- Size grows with cluster size — N-element vectors per write.
- Garbage collection of old vectors is non-trivial.
- Hard to merge after schema changes (which nodes are in N?).

### Variants
- **Dotted version vectors** — reduce size when nodes come and go.
- **Interval tree clocks** — adapt to dynamic membership.

---

## 5. Hybrid Logical Clocks (HLC)

Logical clocks (Lamport, Vector) lose human meaning — "what time was this event?" is meaningless. Wall clocks give meaning but fail causality.

**HLC** combines:
- A wall-clock-style component (physical time, milliseconds).
- A logical counter to break ties / preserve causality.

```
HLC = (physical_time, logical_counter)
```

Rules (simplified):
1. On local event: set HLC = (max(wall_clock, prev_physical), prev_logical+1 if same physical).
2. On receive: take max of (own, received), increment logical.

The physical component **mostly tracks wall-clock time** — so HLC values are interpretable as "approximately when did this happen?" — while the logical component **guarantees causality**.

### Properties
- HLC values are within ~ε of wall-clock time (so timestamps are meaningful).
- If A → B, then HLC(A) < HLC(B).
- Concurrent events get arbitrary order (no detection like vector clocks).

### Use
- CockroachDB, YugabyteDB use HLC.
- MongoDB causal sessions use a similar concept.
- Most modern "distributed clock" systems land on HLC.

### Why HLC is the practical winner
- Simpler than vector clocks (one number per event, not N).
- Human-meaningful (close to wall-clock).
- Causally correct.
- Bounded size.

For most apps that need "more than wall-clock but less than vector," HLC is the answer.

---

## 6. TrueTime (Google Spanner)

A radically different approach: don't track logical time; **bound the uncertainty of physical time**.

Each Spanner node has access to:
- **Atomic clocks** (precise reference).
- **GPS antennas** (synchronized to UTC).

The TrueTime API returns:
- `TT.now()` returns an **interval** `[earliest, latest]` — "the actual current UTC time is somewhere in this interval."

For Spanner's transactions:
- Pick `commit_timestamp = TT.now().latest`.
- **Wait** until `TT.now().earliest > commit_timestamp` before considering the transaction visible.

This **commit-wait** guarantees that if transaction T1 finished its wait before T2 read, then T1's timestamp < T2's timestamp — giving **external consistency** (a.k.a. linearizability).

```
T1 commits at "wall time 14:00:00.005"
   TT.now() = [14:00:00.000, 14:00:00.010]
   commit_timestamp = 14:00:00.010
   wait until TT.now().earliest > 14:00:00.010 (~10 ms)
T1 visible to everyone after 14:00:00.010
```

The wait is the **cost** — Spanner pays ~7 ms (typical uncertainty) per commit globally.

### Why TrueTime is rare
- Requires deploying atomic clocks + GPS in every data center.
- Google has the infrastructure. Most don't.
- HLC + cleverness gets you most of the way for free.

### Alternatives without atomic clocks
- AWS Time Sync Service offers ±1 ms accuracy in EC2. Some systems exploit it.
- Hybrid approaches (TrueTime-light) attempt similar bounds with NTP+PTP.

For most workloads, HLC is the modern answer. TrueTime is the Cadillac.

---

## 7. Comparison Table

| Clock | What it tracks | Detect concurrency | Size | Human-readable | Used by |
|---|---|---|---|---|---|
| Wall clock | Wall-clock time | No | constant | yes | Logs, displays (only) |
| Lamport | Partial order | No | counter | no | Older systems, total-order broadcast |
| Vector | Full causality | Yes | O(N) per write | no | Riak, Dynamo, some Cassandra ops |
| HLC | Causality + wall-clock-ish | No (for that, vectors) | constant | mostly | CockroachDB, YugabyteDB, Mongo causal |
| TrueTime | Bounded uncertainty + commit wait | n/a | interval | yes | Spanner |

---

## 8. Practical Uses

### 8.1 Conflict resolution in eventually-consistent stores
Vector clocks tell you "these two writes are concurrent — resolve them." Last-write-wins by wall clock is **wrong** when clocks differ; LWW by Lamport is fine; concurrent detection requires vector clocks.

### 8.2 Causal consistency
Track each session's vector clock; ensure reads see causally-prior writes. MongoDB's causal sessions use this.

### 8.3 Distributed transactions
Spanner/Cockroach/Yugabyte use HLC or TrueTime to assign global commit timestamps. Reads at a timestamp give a consistent snapshot.

### 8.4 Event sourcing / log compaction
Logs need monotonic timestamps for correctness; HLC is a good fit.

### 8.5 Debugging
Vector clocks let you reconstruct who-saw-what-when, useful for tracing causality through async systems.

---

## 9. Worked Example: A Distributed Counter with Concurrent Writes

Nodes A, B, C maintain replicas of a counter. Each accepts increments locally; replicas reconcile.

### Naive: wall-clock LWW
```
A increments at 14:00:00, value 1.
B increments at 14:00:00 (clock skew, actually 14:00:00.300), value 1.
C reads: which wins?
LWW by wall-clock: depends on whose clock is right.
```

If A and B's clocks are skewed, you can lose one of the increments. Counter wrong.

### Better: vector clocks
```
A vector [1,0,0] value 1
B vector [0,1,0] value 1
C reads, sees concurrent vectors → conflict
Resolution: SUM the increments → counter = 2 (correct!)
```

For counters specifically, **CRDTs** generalize this elegantly. See [CRDTs →](./crdts.md).

### HLC version
```
A increment at HLC(14:00:00.000, 0)
B increment at HLC(14:00:00.000, 1) — same physical, logical breaks tie
```

HLC orders them deterministically without detecting concurrency. For LWW with no semantic merging, this is enough.

---

## 10. The Leap-Second Disaster (and Friends)

Real-world clock failure modes:

### June 30, 2012 — leap second
Linux's clock handling had a bug; many JVM-based servers locked up. Reddit, LinkedIn, FourSquare, Yelp, and others went down. Mozilla, Cloudflare suffered.

### Cloudflare's 2017 leap-second bug
Cloudflare's DNS recursor (RRDNS) had a Go bug where it didn't handle leap seconds; ~2% of DNS queries failed for hours.

### NTP jumps
If your NTP server gets confused (or you misconfigure), your clock can jump hours. Anything depending on monotonic clock progression breaks.

### Lessons
- Use **monotonic clocks** (`CLOCK_MONOTONIC` on Linux) for measuring durations, not wall clocks.
- Don't compare wall clocks across nodes for correctness.
- Subscribe to leap-second changes; test your stack against them.
- Use logical clocks for ordering.

---

## 11. Synchronization Protocols

For systems that need to keep wall clocks close (but still can't fully trust them):

- **NTP** (Network Time Protocol) — sync to ~10–100 ms across the internet, ~1 ms on LAN.
- **PTP** (Precision Time Protocol, IEEE 1588) — sub-microsecond on LAN with hardware support.
- **chrony** — modern NTP daemon, much better than legacy `ntpd`.
- **AWS Time Sync** — claims < 1 ms accuracy via dedicated PTP-grade infrastructure.
- **Atomic / GPS reference clocks** — what Spanner / Facebook's TrueTime equivalents use.

Good time sync narrows the uncertainty but doesn't eliminate it. **Smart systems assume bounded but non-zero skew.**

---

## 12. Common Mistakes

- **Trusting wall clocks for ordering decisions.** Across nodes, never. On one node, only with caveats (NTP jumps).
- **Last-write-wins by wall clock.** Clock skew → wrong winner → lost writes.
- **Lamport clocks where you needed vector.** Concurrency invisible; conflicts silently resolved.
- **Vector clocks at huge cluster size.** Vector size = N nodes per write; bloats.
- **HLC without explanation.** Looks like wall-clock; behaves slightly different; confuses readers.
- **TrueTime envy.** "Let's also do commit-wait." Without atomic clocks, your wait isn't tight enough; you just slowed everything down.
- **Comparing timestamps from different nodes literally.** Even with NTP, ~10 ms is plenty for incorrect ordering.
- **Forgetting leap seconds / NTP jumps in tests.** Bugs that only fire twice a year.
- **Storing wall-clock timestamps in event logs and assuming they're orderable.** They're not, exactly.

---

## 13. Cheat Card

```
WALL CLOCKS LIE   drift, NTP jumps, leap seconds, VM jitter
                   never trust them for cross-node ordering

LAMPORT CLOCK     single counter; partial order; no concurrency detect
                   use: simple causal ordering

VECTOR CLOCK      per-node counters; detects concurrent events
                   use: conflict detection in eventually-consistent stores

HYBRID LOGICAL    wall-clock-like + logical tiebreaker
                   use: modern distributed DBs (CockroachDB, Yugabyte)

TRUETIME          bounded uncertainty via atomic + GPS
                   commit-wait to enforce ordering
                   use: only at Google scale (Spanner)

MONOTONIC CLOCK   for durations on one node; doesn't go backward
                   never use wall clock for "elapsed"

CAUSAL ORDER      A → B if same-process order or msg-send-receive
                   logical clocks track this; wall clocks don't

CONCURRENCY       no causal link; vector clocks detect it,
                   others don't

PITFALLS          LWW by wall clock, comparing timestamps across nodes,
                   ignoring leap seconds, vector clocks at huge N

RULE              For ordering: logical clocks.
                  For display: wall clock.
                  For duration: monotonic clock.
                  Never mix.
```

---

## 14. Resources

### Papers
- "Time, Clocks, and the Ordering of Events in a Distributed System" — Leslie Lamport, 1978 (the foundational paper).
- "Virtual Time and Global States of Distributed Systems" — Mattern, 1989 (vector clocks).
- "Logical Physical Clocks and Consistent Snapshots in Globally Distributed Databases" — Kulkarni et al., 2014 (HLC).
- "Spanner: Google's Globally Distributed Database" — Corbett et al., 2012 (TrueTime).
- "Dynamo: Amazon's Highly Available Key-value Store" — DeCandia et al., 2007 (vector clocks in practice).

### Books
- *Designing Data-Intensive Applications* — Kleppmann (Ch 8: The Trouble with Distributed Systems).
- *Database Internals* — Petrov.

### Articles
- "There is no Now" — Justin Sheehy (ACM Queue).
- "Logical clocks for distributed systems" — various engineering blogs.
- "Falsehoods programmers believe about time" — Noah Sussman.
- "Hybrid Logical Clocks in CockroachDB" — Cockroach Labs blog.

### Videos
- Martin Kleppmann — distributed systems lectures.
- ByteByteGo — "Distributed Clocks Explained".
- Leslie Lamport — Time, Clocks, and Distributed Systems (talks).

### Adjacent reading
- [Consistency Models →](./consistency-models.md)
- [Consensus →](./consensus.md)
- [CAP Theorem →](./cap-theorem.md)
- [CRDTs →](./crdts.md)
- [Replication →](../04-databases/replication.md)
- [Distributed Locks →](./distributed-locks.md)

---

*Previous:* [← Distributed Locks](./distributed-locks.md)  |  *Next:* [Two-Phase Commit (2PC) and Three-Phase Commit (3PC) →](./2pc-3pc.md)
