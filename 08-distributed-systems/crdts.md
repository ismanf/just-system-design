# CRDTs — Conflict-free Replicated Data Types

> **TL;DR** — **CRDTs** are data types that can be **replicated across many nodes, updated independently, and merged automatically** without coordination — and they're **guaranteed to converge** to the same state on all nodes. They sidestep the entire "what if two writes conflict?" problem by **designing data types where there's no such thing as a conflicting write**. Two flavors: **state-based (CvRDT)** — replicas exchange full state, merged via a commutative join function; **operation-based (CmRDT)** — replicas broadcast operations that commute. Examples: **G-Counter** (grow-only counter), **PN-Counter** (positive-negative), **G-Set** (grow-only set), **OR-Set** (observed-remove set), **LWW-Register** (last-writer-wins), **Vector Clocks** (causal metadata), **RGA / Yjs** (collaborative text). Used in **Riak**, **Redis Active-Active**, **Automerge**, **Yjs / Y.js** (Google Docs-style collaboration), **Figma**, **Roblox**, **Akka Distributed Data**. CRDTs are the closest thing distributed systems have to "magic" — but the magic only applies to operations that genuinely commute.

---

## 1. The Idea

Consider a counter replicated across 3 nodes. Both A and B add 1 concurrently. They sync.

- **Naive**: last-write-wins. A sets count=1; B sets count=1. After sync: count=1. Lost an increment.
- **Lock**: only one can write at a time. Defeats availability.
- **CRDT**: each node tracks its own contribution. Merge = sum. Both increments preserved.

```
G-Counter (grow-only):
  node A: [A=1, B=0, C=0]   counter = 1
  node B: [A=0, B=1, C=0]   counter = 1
  merge:  [A=1, B=1, C=0]   counter = 2  ✓
```

The trick: each node owns its slot; merge is **element-wise max** (or sum, depending on type). Concurrent writes don't conflict because they touch different parts of the state.

CRDTs generalize this to many data types.

---

## 2. The Two Flavors

### 2.1 State-based (Convergent — CvRDT)
Replicas store the full state. To sync, send the full state to peers. Peers apply a **commutative, associative, idempotent merge function**.

```
A.merge(B.state)
B.merge(A.state)
# both now have same state regardless of order
```

Bandwidth-heavy (full state every sync) but bandwidth-cheap if state is small or gossip rare.

### 2.2 Operation-based (Commutative — CmRDT)
Replicas broadcast **operations** (e.g., `inc 1`, `add "x"`). Operations must commute when applied in any order on any state.

Requires **reliable causal delivery** of operations (no missing or out-of-order ops).

Bandwidth-light (only ops) but assumes a reliable broadcast layer.

Modern frameworks (Yjs, Automerge) use a hybrid.

---

## 3. The Canonical CRDTs

### 3.1 G-Counter (Grow-only Counter)
Map of `{node_id → count}`. Increment = increment your own slot. Value = sum of all slots. Merge = element-wise max.

```
A increments 3 times: A=[A:3, B:0, C:0]
B increments 2 times: B=[A:0, B:2, C:0]
merge: max([A:3,B:0,C:0], [A:0,B:2,C:0]) = [A:3,B:2,C:0]
value: 3+2 = 5 ✓
```

Used for: page views, likes (when undo isn't supported), distributed counters.

### 3.2 PN-Counter (Positive-Negative Counter)
Two G-counters: one for increments, one for decrements. Value = increments − decrements.

```
PN-counter = {P: G-Counter, N: G-Counter}
inc(node, k) = P.inc(node, k)
dec(node, k) = N.inc(node, k)
value = P.value() - N.value()
```

Supports both increment and decrement. Used for: stock counters, account balances (if you accept eventual consistency).

### 3.3 G-Set (Grow-only Set)
Just a set. Union on merge. Cannot remove.

```
A: {alice, bob}
B: {bob, carol}
merge: {alice, bob, carol}
```

Used for: append-only logs, "ever-seen" lists.

### 3.4 2P-Set (Two-Phase Set)
Two G-sets: `added` and `removed`. An element is in the set iff in `added` and not in `removed`. Cannot re-add a removed element.

### 3.5 OR-Set (Observed-Remove Set)
Each add tags the element with a unique ID. To remove: remove specific ID. Allows re-adding.

```
A adds "x" with tag t1: {(x,t1)}
B adds "x" with tag t2: {(x,t2)}
A removes "x": removes only t1
merge: {(x,t2)} — x still present, only A's add was removed
```

This is what real "set" semantics need under concurrent ops. Used in Riak.

### 3.6 LWW-Register (Last-Writer-Wins Register)
Single value with timestamp. Write attaches a timestamp; merge keeps the one with higher timestamp.

```
A writes "alice" at t=10
B writes "bob"   at t=12
merge: "bob" (higher t)
```

Simple but **clock-dependent** — bad clocks → wrong winner. Use HLC or vector clocks if possible.

### 3.7 Multi-Value Register (MV-Register)
Stores multiple concurrent values with vector clocks. App resolves conflicts.

Used in Dynamo / Riak when LWW isn't acceptable.

### 3.8 Sequence CRDTs
For ordered lists / strings — collaborative text. Variants: **RGA**, **WOOT**, **Logoot**, **CRDT-Sequence**, **Yata** (Yjs).

The hard part: maintaining a consistent insertion order despite concurrent inserts and deletes. Underlying tech for collaborative editors.

### 3.9 Maps
A CRDT map maps keys to CRDT values. Each value is independently a CRDT.

Used in Yjs (Y.Map), Automerge (Doc).

---

## 4. The Math: Semilattices

Formal underpinning: **state-based CRDTs require their state to form a join semilattice**. That is:
- States have a partial order.
- There's a **least upper bound** (join) for any two states.
- The join is **commutative, associative, idempotent**.

Merging two states = computing the join.

```
merge(A, B) = merge(B, A)            commutative
merge(merge(A,B), C) = merge(A, merge(B,C))   associative
merge(A, A) = A                       idempotent
```

These three properties ensure that no matter what order operations or merges happen, **all replicas converge to the same state** once they've received the same set of updates.

This is the **mathematical magic** that lets CRDTs avoid conflict.

---

## 5. Operation-Based CRDT Requirements

For CmRDTs, operations must commute on the abstract state. Plus, the system must guarantee:
- **At-least-once delivery** of operations.
- **Causal delivery** — operations causally dependent are delivered in order; concurrent ops can be delivered in any order.

In practice this means using **vector clocks** or similar causal mechanisms. See [Clocks →](./clocks.md).

If the system can't guarantee causal delivery, operations may produce incorrect state. This is why state-based is simpler in some networks — it self-corrects.

---

## 6. Real-World Use Cases

### 6.1 Collaborative editing
**Google Docs** uses Operational Transformation (OT) historically; many newer tools (Figma, Linear, Notion's recent versions, Roblox Studio collaboration) use CRDTs.

Libraries:
- **Yjs** (Y.js) — popular JS CRDT library. Used by Notion, Linear, etc.
- **Automerge** — JSON-like CRDT. Used by Microsoft Office for some experimental features.
- **Sharedb** — older OT-based.

CRDTs gracefully handle two users typing simultaneously, paste from clipboard, undo, etc.

### 6.2 Multi-master databases
- **Riak**: G-Set, OR-Set, PN-Counter built in.
- **Redis Active-Active** (Redis Enterprise): CRDT-based replication across regions.
- **Azure Cosmos DB**: CRDT-based multi-region writes.
- **Couchbase**: similar.

### 6.3 Mobile / offline-first apps
Sync engine where mobile devices and server replicate state. CRDTs handle "I edited offline; server also got edits; let's merge."

Used by: many note-taking apps, todo apps, calendar apps with offline support.

### 6.4 Distributed configuration
- **Consul, etcd, Zookeeper**: typically use consensus, not CRDTs. But sidecar agents may CRDT-merge state.
- **Akka Distributed Data**: ORSet, ORMap for Akka clusters.

### 6.5 Counters at scale
- View counts on highly-replicated content (Cloudflare-style).
- Multi-region "active users" metrics.

### 6.6 Federated systems
- ActivityPub / Mastodon-style federation (some uses).
- IPFS, blockchains (some patterns).

---

## 7. Where CRDTs Don't Fit

### Operations that don't commute
"Charge $100 if balance > $100" — depends on read-then-write. Two replicas could each see balance=$150 and both charge → balance = -$50. CRDTs alone can't prevent this; need consensus or other coordination.

### Strict ordering required
"Print the operations in the order they happened" — without a total order, you can't.

### Unique constraints
"Username must be unique across all replicas." Two replicas could both accept the same username. Consensus needed.

### Cap-fixed sets (with eviction)
LRU cache where size is bounded — eviction order isn't commutative in general.

For these, use consensus (etcd, Spanner), sagas, or other coordination patterns.

---

## 8. CRDT Trade-offs

### Strengths
- **No coordination needed** — high availability, low latency, offline-friendly.
- **Guaranteed convergence** — math says so.
- **Naturally partition-tolerant** — survive arbitrary network conditions.
- **Composable** — map of CRDT, set of CRDTs, etc.

### Weaknesses
- **Memory overhead** — track per-node state (G-counter is `O(N)` per counter).
- **Tombstone bloat** — OR-Set with many removes accumulates ghost entries.
- **Operation semantics constrained** — only commutative ops.
- **Garbage collection** — need to prune old metadata; non-trivial in distributed setting.
- **Complexity** — choosing the right CRDT and reasoning about behavior takes work.

---

## 9. Worked Example: Distributed Counter

You want a global "page view" counter, replicated across 5 regions, updated by 1000 servers, queryable from any region.

### Approach A: Consensus (Raft)
- All increments go through a leader.
- Strong consistency.
- Latency: cross-region RTT per increment (~100 ms).
- Bottleneck: leader throughput.

### Approach B: Sharded counter + periodic flush
- Each server increments its own local counter.
- Periodic flush to a central store.
- Lower latency; eventual consistency; occasional lost increments on crash.

### Approach C: G-Counter CRDT
- Each server maintains its slot.
- Gossip / async replicate the state.
- Anyone reads: sum of all slots.
- Eventual consistency; no lost increments; no leader; no cross-region writes on hot path.

```
server_1 in us-east: [..., us_east_1: 5, ...]
server_2 in eu-west: [..., eu_west_2: 3, ...]
server_3 in ap-south: ...

state replicated via gossip; merge = element-wise max
total = sum of all entries
```

Approach C scales linearly with no coordination. Trade: per-server state grows with cluster size; some metadata overhead.

For "approximate global counter," this is the modern best practice.

---

## 10. Worked Example: Collaborative Text Editor

Two users type concurrently into a shared document.

### Naive
- User A types "Hello" at position 0.
- User B types "World" at position 0.
- Result: depends on who sees whose write first. Conflicts.

### OT (Operational Transformation, Google Docs)
- Each op is transformed against concurrent ops.
- Server is canonical.
- Complex; tricky edge cases over the years.

### CRDT (Yjs / Automerge)
- Each character has a stable unique ID (`{client_id, sequence_id}`).
- Inserts go between specific IDs.
- No central server needed.

```
User A inserts "Hello" — each char has IDs {A, 1}, {A, 2}, ...
User B inserts "World" at the same logical position — IDs {B, 1}, {B, 2}, ...
when sync:
  merge produces a deterministic order based on IDs
  both users see same final text — maybe "HelloWorld" or "WorldHello"
  the order is consistent across replicas
```

Modern collaborative editors (Linear, Figma) lean on CRDTs because they survive flaky networks and offline scenarios cleanly.

---

## 11. Garbage Collection

OR-Set tombstones, G-Counter metadata for departed nodes, vector clock entries — all accumulate. Real CRDT systems must prune:

- **Stable history**: once all replicas have seen an op, its tombstone can be removed.
- **Causal history**: GC entries older than a "lower bound" timestamp.
- **Versioned re-keying**: periodically compact state.

Yjs and Automerge both have explicit GC strategies. Riak does background compaction.

Without GC, CRDTs grow forever — not a long-term solution.

---

## 12. Hybrid Approaches

Real systems often combine:
- **CRDTs for collaborative data** (notes, cursors, comments).
- **Consensus for critical state** (auth, billing, schema).
- **Operational transformation** for some legacy use cases.

The choice per data type — not the whole system.

---

## 13. Common Mistakes

- **CRDTs for non-commutative operations.** "Reserve seat" can't be a CRDT — need uniqueness.
- **LWW for everything.** Clock skew loses writes. Use proper CRDT or HLC.
- **No garbage collection.** State grows unbounded.
- **Building from scratch.** Use Yjs / Automerge / Riak's primitives. Subtle correctness bugs.
- **Confusing eventual convergence with eventual correctness.** They converge to a value; whether it's the value you wanted is up to you.
- **Operation-based without reliable causal delivery.** Operations applied out of order → divergent state.
- **State-based with huge state.** Full-state sync is expensive; use deltas or operation-based.
- **Expecting linearizable behavior.** CRDTs give eventual, not strong. If you need strong, use consensus.

---

## 14. Cheat Card

```
CRDT          replicated data type with conflict-free merge
              concurrent updates converge to same state
              no coordination needed

TWO FLAVORS
  state-based (CvRDT)  exchange full state; merge function
  operation-based (CmRDT) broadcast ops; assume causal delivery

KEY MATH      merge is commutative, associative, idempotent

TYPES
  G-Counter     grow-only counter (per-node slots, sum)
  PN-Counter    pos + neg counter
  G-Set         grow-only set (union)
  OR-Set        observed-remove set (add/remove with tags)
  LWW-Register  last-writer-wins value
  MV-Register   multi-value with vector clocks
  Sequence      ordered list/text (RGA, Yjs)
  Map           keys → CRDT values

USE CASES     collaborative editing (Yjs, Automerge),
              multi-master DBs (Riak, Redis Active-Active),
              offline-first apps, distributed counters,
              sync engines

NOT FOR       operations that don't commute,
              unique constraints, strict ordering,
              strong consistency

PITFALLS      LWW with bad clocks, op-based without causal delivery,
              no GC of tombstones / metadata,
              CRDT for inherently coordinative ops

RULE          CRDTs are magic for commutative ops.
              For everything else, use consensus.
```

---

## 15. Resources

### Papers
- "A comprehensive study of Convergent and Commutative Replicated Data Types" — Shapiro et al., 2011 (the bible).
- "Conflict-free Replicated Data Types" — Shapiro et al., 2011 (concise version).
- "Real Differences between OT and CRDT for Co-Editors" — paper analyzing trade-offs.
- "RGA: Replicated Growable Array" — Roh, Jeon, Kim, Lee.
- "Yjs Internals" — Kevin Jahns documentation.

### Books
- *Designing Data-Intensive Applications* — Kleppmann (mentions CRDTs in replication chapter).
- *Local-First Web Development* (online) — Martin Kleppmann's course (sister of his research).

### Articles
- "An introduction to CRDTs" — Martin Kleppmann blog series.
- "How Figma's multiplayer technology works" — Figma blog.
- "Why CRDTs aren't a substitute for OT in Google Docs" — old Google docs blog.
- "How we built realtime collaboration in Linear" — Linear engineering.
- "Local-first software" — Kleppmann, Wiggins, van Hardenberg, McGregor.

### Videos
- Martin Kleppmann — CRDT lectures (Cambridge).
- Strange Loop talks on Yjs, Automerge.
- ByteByteGo — "CRDTs Explained".

### Tools / Libraries
- **Yjs** — JavaScript CRDT library, widely used.
- **Automerge** — JSON-like CRDT (JS, Rust).
- **Riak** — built-in CRDTs.
- **Redis Enterprise Active-Active**.
- **Akka Distributed Data** — Scala/Java.
- **rust-crdt** — Rust library.

### Adjacent reading
- [Consistency Models →](./consistency-models.md)
- [Clocks →](./clocks.md)
- [Consensus →](./consensus.md)
- [Replication →](../04-databases/replication.md)
- [CAP Theorem →](./cap-theorem.md)
- [PACELC →](./pacelc.md)

---

*Previous:* [← Merkle Trees](./merkle-trees.md)  |  *Next:* [Byzantine Fault Tolerance →](./bft.md)
