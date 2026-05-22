# Skip Lists

> **TL;DR** — A **skip list** is a layered linked list that gives you **O(log N) search, insert, and delete** in expected time, without the rebalancing machinery of a balanced binary search tree. Each element appears in level 0 (the base list); with probability 1/2 it also appears in level 1, then level 2, and so on — building a sparse "express lane" structure. Conceived by William Pugh in 1990, skip lists became famous because they're **simpler to implement than red-black or AVL trees** while delivering comparable performance, and they're **lock-friendly** for concurrent variants. They're the backbone of **Redis sorted sets (ZSET)**, **LevelDB / RocksDB memtables**, **Cassandra's memtable**, and many in-memory ranked structures. The honest take: **you'll almost never write a skip list by hand, but understanding the structure makes Redis ZSETs, range queries in LSM-trees, and lock-free in-memory indexes legible.**

---

## 1. The big picture

```
Level 3:  HEAD ──────────────────────────────────────────► 8 ──────────► NIL
Level 2:  HEAD ──────────► 3 ────────────────────────────► 8 ──────────► NIL
Level 1:  HEAD ─► 1 ─────► 3 ─────────► 5 ──────────────► 8 ──────────► NIL
Level 0:  HEAD ─► 1 ─► 2 ─► 3 ─► 4 ─► 5 ─► 6 ─► 7 ─► 8 ─► 9 ─► 10 ─► NIL
```

Every element lives in level 0, in sorted order. Each element also independently lands in level 1 with probability 1/2, level 2 with 1/4, level 3 with 1/8, etc. The higher you go, the sparser the level — and the bigger the jumps.

Search: start at the top-left. At each step:

- If the next pointer at this level skips past the target, drop down one level.
- Otherwise, follow it forward.
- Eventually you reach level 0 and either find the target or fall off the end.

Each level has expected length N/2^k, so the total work is **O(log N)** expected — the same as a balanced tree, but with no rotations and no rebalancing.

---

## 2. Why skip lists got famous

In 1990, the conventional way to get O(log N) search-and-update was a **red-black tree** or **AVL tree**. Both work; both require tricky rotation logic that's easy to get wrong. Pugh's paper, "Skip Lists: A Probabilistic Alternative to Balanced Trees," showed:

- **Simpler code.** A few dozen lines of clear, testable logic — no rotation cases.
- **Same expected complexity.** O(log N) for search, insert, delete.
- **Better cache behavior** than pointer-heavy trees in some workloads.
- **Easier concurrency.** Inserts touch a bounded set of pointers; lock-free implementations exist.

For systems software, "simpler and equivalent" is a real win. Skip lists became one of the standard "ordered map" implementations in concurrent data-structure libraries (`java.util.concurrent.ConcurrentSkipListMap`, Lua's debug tools, etc.) and the engine behind many in-memory indexes.

---

## 3. The construction algorithm

```python
import random

P = 0.5            # promotion probability
MAX_LEVEL = 32     # safe upper bound

class Node:
    __slots__ = ("key", "value", "forward")
    def __init__(self, key, value, level):
        self.key = key
        self.value = value
        self.forward = [None] * (level + 1)

def random_level():
    lvl = 0
    while random.random() < P and lvl < MAX_LEVEL:
        lvl += 1
    return lvl

class SkipList:
    def __init__(self):
        self.head = Node(None, None, MAX_LEVEL)
        self.level = 0

    def search(self, key):
        node = self.head
        for i in range(self.level, -1, -1):
            while node.forward[i] and node.forward[i].key < key:
                node = node.forward[i]
        node = node.forward[0]
        return node.value if node and node.key == key else None

    def insert(self, key, value):
        update = [None] * (MAX_LEVEL + 1)
        node = self.head
        for i in range(self.level, -1, -1):
            while node.forward[i] and node.forward[i].key < key:
                node = node.forward[i]
            update[i] = node

        if node.forward[0] and node.forward[0].key == key:
            node.forward[0].value = value
            return

        lvl = random_level()
        if lvl > self.level:
            for i in range(self.level + 1, lvl + 1):
                update[i] = self.head
            self.level = lvl

        new_node = Node(key, value, lvl)
        for i in range(lvl + 1):
            new_node.forward[i] = update[i].forward[i]
            update[i].forward[i] = new_node
```

That's it. Sixty lines. No rotations. No balance invariants to violate. A red-black tree implementation of the same operations is closer to 300 lines.

---

## 4. Why the probabilities work

The expected length at level `k` is `N · (1/2)^k`. The expected highest level is roughly `log₂ N`. A search walks at most O(log N) horizontal hops per level on average, with at most O(log N) levels — so total expected work is O(log N).

The catch: it's *expected*, not worst-case. Bad luck (a string of high random promotions, all near the same key) can produce a degenerate structure. In practice, with `MAX_LEVEL` capped (e.g., at 32, which handles up to 2^32 items comfortably) and a balanced promotion probability, the variance is small and worst cases are vanishingly rare.

For adversarial inputs (an attacker who can control random promotions), you need a different RNG seed or a different structure. For non-adversarial workloads — which is almost everything — skip lists are robust.

---

## 5. Range queries — the natural superpower

Skip lists keep level 0 in sorted order, with each node pointing to its successor. **Range queries are linear in the result size**, with no rebalancing concerns:

```python
def range_query(self, lo, hi):
    # walk down levels to find lo
    node = self.head
    for i in range(self.level, -1, -1):
        while node.forward[i] and node.forward[i].key < lo:
            node = node.forward[i]
    # walk level 0 collecting matches
    node = node.forward[0]
    while node and node.key <= hi:
        yield (node.key, node.value)
        node = node.forward[0]
```

This is the operation that makes skip lists shine for **ordered key-value stores**. Compare to a tree: in-order traversal also works, but you've got pointers in three directions to manage and possibly need successor logic. Skip list's level 0 is just a singly-linked list — trivial to walk.

---

## 6. Skip list vs balanced BST

| | Skip list | Red-black / AVL tree |
|---|---|---|
| Search / Insert / Delete | O(log N) expected | O(log N) worst case |
| Code complexity | Low | Higher (rotations) |
| Cache behavior | Often better (linked levels) | Pointer-heavy |
| Range queries | Trivial (linked level 0) | Possible, more pointer work |
| Concurrent variants | Easier (CAS-friendly) | Harder (atomic rotations) |
| Determinism | Probabilistic | Deterministic |
| Worst-case guarantees | Probabilistic; rare degradation | Hard bound |
| Memory overhead | Roughly 2× pointers per element (expected) | 2 child pointers + balance bits |

For most software where probabilistic guarantees are fine, skip lists win on implementation simplicity and concurrency. For real-time systems where every operation must have a hard upper bound (avionics, trading), balanced trees may be preferred.

In practice the trade-off has shifted: B+ trees dominate on disk; skip lists and balanced BSTs dominate in memory; lock-free hash tables dominate where ordering isn't needed. Skip list lives in the "in-memory ordered map" niche.

---

## 7. Where skip lists actually appear in production

### Redis sorted sets (ZSET)

Redis ZSET is one of the most-used data structures in modern infrastructure (leaderboards, rate limiters, range queries on timestamps, secondary indexes). Internally it's a **combination** of a hash table (member → score) and a **skip list** (sorted by score).

When you `ZRANGEBYSCORE`, `ZRANGE`, or `ZRANK`, the skip list does the work. Insert (`ZADD`) updates both structures. Redis chose skip lists specifically because the code was simple, concurrent ops (under Redis's single-threaded model, "atomic") are easy, and range scans are first-class.

### LevelDB / RocksDB memtable

The in-memory write buffer ("memtable") of an LSM-tree (see [Storage Engines →](../09-storage/storage-engines.md)) is typically a skip list. Reasons:
- Concurrent inserts during heavy writes.
- Efficient ordered iteration for flushing to SSTables.
- Easy concurrent reads while writes happen.

### Cassandra memtable

Same pattern: in-memory sorted store with concurrent writes.

### `ConcurrentSkipListMap` in Java

`java.util.concurrent.ConcurrentSkipListMap` is the standard concurrent ordered map. Used heavily in JVM systems where you need both sortedness and high write concurrency.

### Apache Lucene's in-memory postings

Skip lists also appear in inverted indexes for fast "next greater" traversal on long postings lists. See [Inverted Indexes →](./inverted-index.md).

The pattern: **anywhere you need an in-memory ordered map with concurrent updates, skip lists are a top choice.**

---

## 8. Concurrent skip lists

The famous Maged Michael / Doug Lea / Pugh papers give lock-free and fine-grained-locking skip list algorithms. Two important properties:

- **Inserts touch a bounded set of nodes** (the predecessors at each level). This makes per-node CAS-based insertion practical.
- **Marking** (logical deletion) lets you separate "marked for delete" from physical removal — enabling concurrent readers to never see a half-deleted node.

For most engineers, the right move is to use a battle-tested library (`ConcurrentSkipListMap` in Java, `crossbeam-skiplist` in Rust) rather than implement your own. Lock-free data structures are notoriously hard to get right; bugs there look like data corruption months later.

---

## 9. The persistent / on-disk question

Skip lists are intrinsically in-memory structures. Their pointer-per-level layout doesn't play well with disk pages.

For ordered on-disk storage you want **B+ trees** (page-based, cache-friendly) or **LSM-trees** (write-friendly, with in-memory memtables that *are* skip lists). See [Database Indexing →](../04-databases/indexing.md), [Storage Engines →](../09-storage/storage-engines.md).

A skip list on disk *can* work (jump-list-style external memory layouts exist), but for almost any disk workload, B+ trees and LSMs are better engineered.

---

## 10. Common Mistakes / Anti-Patterns

- **Implementing your own concurrent skip list.** Use a library. Lock-free correctness is hard.
- **Wrong promotion probability.** P = 0.5 is the standard; experimenting with 0.25 or 0.75 rarely helps and often breaks expected guarantees.
- **No MAX_LEVEL cap.** A bad RNG run can promote one node to level 100. Cap it.
- **Trying to use skip lists for disk-resident data.** Use B+ trees or LSM. Skip lists belong in RAM.
- **Storing very small payloads with massive overhead.** Each node carries pointers per level; for a node at level 5 you have 6 pointers (~48 bytes on 64-bit) for maybe an 8-byte value. Use a packed structure if memory is tight.
- **Forgetting deterministic seed for testing.** Randomized structures need reproducible tests. Seed the RNG.
- **Assuming worst-case bounds.** Skip lists are *expected* O(log N). Bad luck can produce O(N) in theory; cap MAX_LEVEL and use cryptographic randomness to defeat adversarial inputs.
- **Hand-rolling a skip list when a B-tree or hash table is appropriate.** If you don't need ordered iteration, use a hash table. If you need disk persistence, use a B-tree.
- **Treating the data structure as the API.** Wrap it. Expose `get / set / range`, not `forward[i]`.

---

## 11. Cheat Card

```
PURPOSE   Ordered in-memory map with O(log N) expected search,
          insert, delete, and cheap range scans.

STRUCTURE
  Level 0: sorted linked list of all elements
  Level k: each element from level k-1 with probability P (=0.5)
  Max level cap (e.g., 32) prevents pathological promotions

OPERATIONS
  Search    O(log N) expected; descend levels, walk right
  Insert    O(log N) expected; bounded pointer updates
  Delete    O(log N) expected
  Range     O(log N + K); walk level 0 between bounds

WHY USED
  Simpler than red-black / AVL trees
  Naturally concurrent-friendly
  Excellent for ordered iteration / range queries

WHERE
  Redis sorted sets (ZSET)
  LevelDB / RocksDB / Cassandra memtables
  java.util.concurrent.ConcurrentSkipListMap
  Lucene postings skip pointers
  Crossbeam (Rust), other concurrent libraries

VS BST
  Probabilistic vs worst-case bounds
  Simpler code, easier concurrency
  In memory only — disk wants B+ tree / LSM

PITFALLS
  Rolling your own concurrent skip list
  No MAX_LEVEL cap → unbounded promotion
  Tweaking probability away from 0.5
  Using skip lists for on-disk storage
  Tiny values + lots of pointer overhead
  Worst-case guarantees in adversarial workloads

RULE   In-memory + ordered + concurrent = skip list (or use the
       standard library version). Don't fight it for disk.
```

---

## 12. Resources

### Papers
- "Skip Lists: A Probabilistic Alternative to Balanced Trees" — William Pugh, 1990. The original.
- "A Provably Correct Scalable Concurrent Skip List" — Maged Michael, 2008.
- "Concurrent Maintenance of Skip Lists" — Pugh, 1990.

### Documentation
- **Java `ConcurrentSkipListMap`** — <https://docs.oracle.com/en/java/javase/21/docs/api/java.base/java/util/concurrent/ConcurrentSkipListMap.html>
- **Redis ZSET internals** — <https://redis.io/docs/data-types/sorted-sets/>
- **RocksDB memtable** — <https://github.com/facebook/rocksdb/wiki/MemTable>

### Articles
- "Skip lists are awesome" — Igor Ostrovsky / SQL Server team blog.
- "Implementing skip lists in Go / Rust / C++" — many tutorial-grade write-ups.
- "Why Redis chose skip lists" — antirez's old notes (Salvatore Sanfilippo).
- "Lock-free skip lists" — Maged Michael & Doug Lea engineering posts.

### Videos
- *Skip Lists* — MIT 6.046 / Stanford CS166 lectures.
- *Inside Redis ZSET* — Redis conference talks.
- ByteByteGo — "Skip List Explained."

### Tools
- **`java.util.concurrent.ConcurrentSkipListMap`** (Java).
- **`crossbeam-skiplist`** (Rust).
- **Folly `ConcurrentSkipList`** (Facebook C++).
- **`pugh-skiplist`**, **`pyskiplist`** (Python tutorials).
- **Redis** — production-grade ZSET.

### Adjacent reading
- [Inverted Indexes →](./inverted-index.md)
- [Database Indexing (B-Tree, Hash, LSM-Tree) →](../04-databases/indexing.md)
- [Storage Engines (LSM-Trees vs B-Trees) →](../09-storage/storage-engines.md)
- [Redis Deep Dive →](../05-caching/redis-deep-dive.md)
- [Wide-Column Stores (Cassandra, HBase, ScyllaDB) →](../04-databases/wide-column-stores.md)
- [Concurrency Control →](../04-databases/concurrency-control.md)

---

*Previous:* [← Trie Data Structure for Autocomplete](./trie.md)  |  *Next:* [Inverted Indexes →](./inverted-index.md)
