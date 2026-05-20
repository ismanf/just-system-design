# Consistent Hashing

> **TL;DR** — **Consistent hashing** maps both **keys** and **nodes** onto a circular hash space ("the ring"). Each key is owned by the next node going clockwise. Adding or removing a node moves **only ~1/N of the keys**, instead of nearly all of them as with naive modulo (`hash(key) % N`). Adding **virtual nodes** smooths out distribution. It's the foundational trick behind Dynamo, Cassandra, ScyllaDB, Memcached/Ketama, Riak, Couchbase, CDN routing, load balancers, and distributed caches.

---

## 1. The Problem It Solves

Suppose you shard keys across N nodes with `hash(key) % N`.

```
N = 4
hash(key) % 4 → node
add node, N = 5
hash(key) % 5 → almost all keys move
```

Adding **one node** invalidates almost every key. For a cache, **every key misses simultaneously** → thundering herd against your database. Disaster.

Consistent hashing fixes this so that **adding or removing one node moves only ~1/N of the keys**.

---

## 2. The Ring

```
                  ┌──── 0 / 2^32 ────┐
                  │                  │
           hash space wraps around
                  │                  │
                  └────── 2^31 ──────┘
```

- Hash keys to a position on `[0, 2^32)` (or some ring size).
- Hash node identifiers to positions on the **same** ring.
- A key is owned by the **first node** found going clockwise from the key's position.

```
           ●  node A (hash=10)
       ╱       ╲
      ●         ●  node B (hash=80)
   key X
   (hash=70)  → owned by B
                      ╲
                       ●  node C (hash=180)
key Y (hash=90) → owned by C
```

When you **add node D** at hash=120, it takes over only the arc between **80 and 120** — everything else stays put.

---

## 3. The Naive Version Has a Problem: Skew

If you only hash the **node IDs** once, you can land them unevenly on the ring. One node might own a huge arc; another might own a tiny one. Distribution is lumpy.

The fix: **virtual nodes (vnodes)**.

Each physical node is represented by **many points** on the ring (e.g., 100 or 256). With enough virtual placements, the law of large numbers smooths things out and every physical node owns roughly an equal share.

```
node A → 200 vnodes scattered around the ring
node B → 200 vnodes
node C → 200 vnodes
```

Now ownership is uniform, *and* removing a node redistributes its load evenly across the rest.

---

## 4. Algorithm in Pseudocode

```python
class ConsistentHashRing:
    def __init__(self, vnodes_per_node=128):
        self.ring = SortedDict()   # hash → node
        self.vnodes = vnodes_per_node

    def add_node(self, node):
        for i in range(self.vnodes):
            h = hash(f"{node}#{i}") % RING_SIZE
            self.ring[h] = node

    def remove_node(self, node):
        for i in range(self.vnodes):
            h = hash(f"{node}#{i}") % RING_SIZE
            del self.ring[h]

    def get_node(self, key):
        h = hash(key) % RING_SIZE
        # find the smallest position >= h, wrapping around
        i = self.ring.bisect_left(h)
        if i == len(self.ring):
            i = 0
        return self.ring.values()[i]
```

Lookups are `O(log N × vnodes)` — a binary search on the ring.

---

## 5. Replication on the Ring

For **redundancy**, place each key on the **next K nodes clockwise** (the "preference list"). Each key has K replicas; reads/writes can quorum across them.

```
key X (hash=70)
  primary  → first node clockwise
  replica  → next node clockwise
  replica  → next node clockwise
```

Cassandra's `RF=3` is literally this: three replicas are the next three *physical* nodes on the ring (skipping additional vnodes from the same physical box).

---

## 6. Where It's Used

| System | How |
| --- | --- |
| **Amazon Dynamo** (the 2007 paper) | Founded the modern technique. |
| **Cassandra / ScyllaDB** | Token ring, vnodes per node, RF replicas clockwise. |
| **Riak / Couchbase / Voldemort** | Dynamo-style ring. |
| **DynamoDB** | Internal partition routing. |
| **Memcached (Ketama)** | Client-side consistent hashing of cache nodes. |
| **Redis Cluster** | Fixed 16384 hash slots distributed among nodes (a variant). |
| **CDN edge selection** | Map IPs / users to edge POPs consistently. |
| **Load balancers** | "ring hash" (Envoy / HAProxy / NGINX) for session affinity. |
| **Distributed task queues** | Assign tasks to workers consistently for locality. |
| **Sticky sessions in stateless tiers** | Same user → same backend in a way that survives node changes. |

If you see "**ring hash**", "**ketama**", "**hash ring**", "**token ring**" — you're looking at consistent hashing.

---

## 7. Variants and Refinements

### Jump Consistent Hash (Google, 2014)
A compact algorithm that picks a bucket in **O(log N)** time with no ring data structure. Returns "which of N buckets does this key go to?" Adding a bucket moves the right fraction without storing anything.
- Pros: tiny memory, fast.
- Cons: buckets are numbered 0..N-1 — works best when N changes by adding at the end (i.e., you can't easily remove an arbitrary bucket).

### Rendezvous (HRW) Hashing
For each key, compute `hash(key, node)` for every node; the node with the highest score wins.
- Pros: no ring, no vnodes, automatic balance.
- Cons: O(N) per lookup; not great with many nodes.
- Used by some CDNs and weighted load balancers.

### Bounded-load Consistent Hashing
Adds a "capacity" cap per node — if the natural owner is full, the key spills to the next node. Prevents hot keys from overloading a node. Used by Vimeo, Google's load balancers.

### Maglev Hashing (Google)
A consistent hashing scheme with a precomputed lookup table for very low per-lookup cost. Used in Google's L4 LBs.

---

## 8. Failure & Rebalancing

**Adding a node**: ~1/N of keys move *only from neighbors* (with vnodes, from many neighbors).
**Removing a node**: its keys redistribute to the next nodes clockwise.

This is the headline property: **no global reshuffle**.

In practice you also need to **stream the actual data** to the new owners (Cassandra bootstrap, Redis Cluster slot migration). That can take hours for big shards; rate-limit the streaming so it doesn't crush the cluster.

---

## 9. Hot Keys vs Hot Partitions

Consistent hashing helps with **uniform distribution of keys** to nodes. It doesn't help when **one key** is far hotter than the rest:

- The single key always hashes to the same node — it's a hot **node**.
- Solution: split the key into N synthetic sub-keys (`counter#0..#N-1`) and aggregate on read.
- Or: replicate the hot key to all nodes (cache layer) and read locally.

See [Hot Partition Problem](../10-scalability/hot-partitions.md).

---

## 10. Consistent Hashing in Caches

Use case: a Memcached/Redis cluster of 10 nodes with **client-side** hashing. Without consistent hashing, adding the 11th node invalidates ~90% of the cache → DB stampede.

With consistent hashing (Ketama-style):
- Adding the 11th node invalidates ~1/11 of keys (and only those keys move).
- The DB sees a small bump, not a flood.
- Almost every modern cache client supports this out of the box.

**Hash tags** (e.g., Redis `{user:42}:profile`) let you force keys to the **same slot** so multi-key ops (transactions, MGET, scripts) work across them.

---

## 11. Trade-offs

- **Lookup cost**: O(log N × vnodes). Fast in practice; the ring is small in RAM.
- **Memory**: 100–256 vnodes × N nodes × (hash + pointer) bytes. Negligible for hundreds of nodes.
- **Ring maintenance**: when nodes come and go, every client/router must learn the new ring. Use gossip (Cassandra), a config service (etcd), or push the slot map (Redis Cluster).
- **Heterogeneous nodes**: give bigger nodes more vnodes; they own a bigger arc.
- **Stateful migration**: actually moving data takes time — plan for backpressure.

---

## 12. Where It Doesn't Help

- When you need **range queries** that span keys. Consistent hashing scatters them.
- When you have **massive skew** in one key. A ring of nodes still all hash one key to one node.
- When you need **strong cross-shard consistency**. Use NewSQL / consensus.
- When the workload is dominated by **co-located joins** that consistent hashing won't preserve. Use a *composite* shard key with hash tags to force locality (`{tenant}/...`).

---

## 13. Implementations to Read

- **`hash-ring`** (Node, Python, Java implementations).
- **`libketama`** — original C reference for memcached.
- **HAProxy** — `hash-type consistent` mode.
- **NGINX** — `hash $key consistent`.
- **Envoy** — `RING_HASH` and `MAGLEV` load-balancing policies.
- **Cassandra** — see `Murmur3Partitioner` + token ring.
- **Redis Cluster** — fixed 16384 slot map (a quantized variant of consistent hashing).
- **Vitess** — `lookup` and `hash` vindexes.

Reading one of these is the fastest way to internalize the idea.

---

## 14. Cheat Card

```
PROBLEM      hash(key) % N moves almost every key when N changes.

SOLUTION     hash both keys AND nodes onto a ring.
              key is owned by the next node clockwise.
              ~1/N of keys move when a node is added/removed.

SKEW FIX     virtual nodes (vnodes).
              each physical node owns many random arcs → uniform.

REPLICATION  next K nodes clockwise = replicas (Dynamo / Cassandra).

VARIANTS
  Jump hash             tiny, fast, fixed-N-ish.
  Rendezvous (HRW)      no ring; O(N) lookup; great for small N.
  Bounded-load          cap per node; spill if hot.
  Maglev                lookup-table based; very fast L4 LB.

WHERE
  Caches (Memcached/Redis client-side, Ketama).
  Distributed DBs (Dynamo, Cassandra, Riak, Couchbase).
  CDNs / L4 LBs (Envoy ring_hash / Maglev).
  Sticky sessions / locality routing.

DOESN'T HELP
  Hot single key → split synthetic sub-keys.
  Range queries → use range partitioning.
  Cross-shard ACID → use NewSQL.

ALWAYS
  Use vnodes for smooth distribution.
  Plan for slot migration time when adding nodes.
```

---

## 15. Resources

### Papers
- **"Consistent Hashing and Random Trees: Distributed Caching Protocols for Relieving Hot Spots on the World Wide Web"** — Karger et al., 1997 (the original paper): <http://www.cs.princeton.edu/courses/archive/fall09/cos518/papers/chash.pdf>
- **"Dynamo: Amazon's Highly Available Key-value Store"** — DeCandia et al., 2007.
- **"A Fast, Minimal Memory, Consistent Hash Algorithm"** — Lamping & Veach (Google, 2014) — Jump Consistent Hash.
- **"Maglev: A Fast and Reliable Software Network Load Balancer"** — Google NSDI 2016.

### Articles
- "Consistent hashing explained" — Tom White: <http://www.tom-e-white.com/2007/11/consistent-hashing.html>
- "Consistent Hashing with Bounded Loads" — Google Research blog.
- "How consistent hashing works" — many blog posts; the Cloudflare and AWS Builders' Library writeups are clear.
- "Implementing a hash ring in 30 lines" — common tutorial style.

### Books
- *Designing Data-Intensive Applications* — Kleppmann; Ch. 6.
- *Cassandra: The Definitive Guide* (3rd ed.) — Carpenter & Hewitt.
- *Database Internals* — Alex Petrov.

### Videos
- ByteByteGo: "Consistent Hashing Explained" — <https://www.youtube.com/@ByteByteGo>
- Hussein Nasser consistent hashing videos — <https://www.youtube.com/@hnasr>
- Tom Scott / Computerphile explainer videos.

### Code references
- **libketama** (memcached's client).
- **Envoy ring_hash / Maglev** — <https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/upstream/load_balancing/load_balancers>
- **Cassandra Murmur3Partitioner** source.
- **Redis Cluster slot map**.

### Adjacent reading
- [Sharding & Partitioning](./sharding-partitioning.md)
- [Hot Partition Problem](../10-scalability/hot-partitions.md)
- [Load Balancing Algorithms](../06-load-balancing/algorithms.md)
- [Key-Value Stores](./key-value-stores.md)
- [Wide-Column Stores](./wide-column-stores.md)

---

*Previous:* [← Sharding & Partitioning](./sharding-partitioning.md)  |  *Next:* [Database Federation →](./federation.md)
