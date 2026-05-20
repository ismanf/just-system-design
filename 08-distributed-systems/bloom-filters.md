# Bloom Filters

> **TL;DR** — A **Bloom filter** is a space-efficient probabilistic data structure that tells you whether an element is **definitely not** in a set or **probably is**. It can give **false positives** ("I think X is there" when X isn't) but never **false negatives** ("I'm sure X isn't there" — and it really isn't). The structure: a bit array of size `m`, with `k` independent hash functions. Insert by setting `k` bits; query by checking all `k` bits are set. Trade-offs: tiny memory (~10 bits per element for 1% false-positive rate), constant-time operations, and no deletions (without variants). Used everywhere: **databases** (skip disk reads for missing keys — Cassandra, RocksDB, Bigtable), **caches** (avoid origin lookups for definitely-missing items), **distributed systems** (membership tests, deduplication), **web** (URL safe-browsing checks). When you need "is X definitely not here?" cheaply, Bloom is the answer.

---

## 1. The Idea

You have a huge set (millions to billions of elements). You want to ask "is X in the set?" without storing the set explicitly.

A Bloom filter says:
- **"Definitely not"** — and means it.
- **"Probably yes"** — but might be wrong (false positive).

That asymmetry is the whole point. If "definitely not" is what you need to short-circuit expensive work (a disk read, a network call, an origin fetch), Bloom is gold.

```
filter = bit array of m bits, all 0 initially
insert(x): set bits at positions h1(x), h2(x), ..., hk(x)
contains(x): check all k positions are 1
```

If any bit at `hi(x)` is 0, x was never inserted — return false.
If all bits are 1, x was probably inserted — return true (might be false positive).

---

## 2. A Visual Example

`m = 16`, `k = 3` hashes.

```
bit array (initial):
[0 0 0 0 0 0 0 0 0 0 0 0 0 0 0 0]

insert "alice": hashes to bits 2, 7, 11
[0 0 1 0 0 0 0 1 0 0 0 1 0 0 0 0]

insert "bob": hashes to bits 3, 7, 13
[0 0 1 1 0 0 0 1 0 0 0 1 0 1 0 0]   ← bit 7 already 1 (shared with alice)

query "alice": check bits 2, 7, 11 → all 1 → "probably yes" ✓
query "carol": say carol hashes to 5, 9, 11 → bit 5 is 0 → "definitely no" ✓
query "dave":  say dave hashes to 3, 7, 11 → all 1 → "probably yes" ✗ (false positive)
```

`dave` was never inserted, but his hash bits collide with bits set by alice and bob. False positive.

---

## 3. False Positive Rate

For a Bloom filter with `m` bits, `n` inserted elements, `k` hash functions:

```
FP rate ≈ (1 - e^(-kn/m))^k
```

For a given desired FP rate `p`, the optimal:
- `m/n = -log(p) / (log 2)²` ≈ `1.44 * log2(1/p)` bits per element.
- `k = (m/n) * ln(2)` ≈ `0.7 * (m/n)` hash functions.

Rules of thumb:

| FP rate | Bits per element | Hash functions |
|---|---|---|
| 10% | 5 | 4 |
| 1% | 10 | 7 |
| 0.1% | 14 | 10 |
| 0.01% | 19 | 14 |

**1% FP rate at 10 bits per element** is the canonical operating point. For a billion items at 1% FP: ~1.25 GB of memory. Tiny compared to storing the actual data.

---

## 4. No False Negatives

This is the magic. A Bloom filter can never report "X is not in the set" if X was inserted, because:
- When X was inserted, k bits were set to 1.
- Those bits can only flip back to 0 by ... deletion, which Bloom doesn't support.

So if the filter says "definitely no," it's right. **This is what makes Bloom useful for short-circuiting.**

---

## 5. Why You Can't Delete

To delete X, you'd want to clear the k bits at X's hash positions. But those bits might also be set by other elements. Clearing them gives false negatives for those.

```
alice: bits 2, 7, 11
bob:   bits 3, 7, 13
delete "alice" → clear 2, 7, 11
now query "bob": bits 3, 7, 13 → 7 is 0 → "definitely no" ❌ WRONG
```

Bloom filters are **insert-only**.

To support deletes, use variants (next section).

---

## 6. Variants

### 6.1 Counting Bloom Filter
Use small counters (4 bits) instead of single bits. Increment on insert, decrement on delete.

- **Pros**: supports deletes.
- **Cons**: 4× memory; counter overflow possible (rare).

### 6.2 Scalable Bloom Filter
Start with a small filter; when it fills up, allocate a new larger one with tighter FP rate. Query checks all layers.

- **Pros**: doesn't need to size N upfront.
- **Cons**: query cost grows; memory layered.

### 6.3 Cuckoo Filter
Different structure (uses cuckoo hashing) that supports deletes and has lower memory at the same FP rate.

- **Pros**: deletes, sometimes better than Bloom.
- **Cons**: more complex; insert can fail (table full).

### 6.4 Bloom Filter with Concurrent Updates
Modify to allow atomic updates (CAS on words instead of single bits) for concurrent use.

### 6.5 Quotient Filter, Xor Filter
Newer designs with different trade-offs. Xor filters are 3× faster lookup than Bloom and use 25% less space at the same FP rate. Catch: bulk-built (rebuild on changes).

### 6.6 Layered / Spectral Bloom Filters
For multi-set membership or counts.

For most uses, **standard Bloom filter** or **Counting Bloom filter** is what you need.

---

## 7. Where Bloom Filters Are Used

### 7.1 Databases — skip disk reads
Cassandra, HBase, RocksDB, LevelDB, Bigtable: keep a Bloom filter per SSTable. When looking up a key, check the filter first. If "definitely not," skip the disk read entirely.

```
read key X:
   for each SSTable:
      if bloom.contains(X):
         read SSTable to confirm
      else:
         skip — definitely not here
   if no SSTable said "probably" → not found

millions of negative lookups per second without touching disk.
```

This is **the** killer use case for Bloom in production.

### 7.2 Caches — avoid origin lookups
Want to know "does this user exist?" without hitting the DB on every nonexistent ID:
- Bloom filter of all user IDs.
- Check filter first.
- "Definitely no" → return 404 immediately.
- "Probably yes" → check DB.

Protects DB from enumeration attacks and pollution from cache penetration. See [Cache Pitfalls →](../05-caching/cache-pitfalls.md).

### 7.3 Distributed deduplication
Stream processing: have I seen this event before? Bloom filter of seen IDs. False positives mean rare double-process; idempotent consumer handles it.

### 7.4 Browser safe-browsing
Chrome / Firefox keep a local Bloom filter of malicious URLs. On every navigation, check locally; if "probably yes," query Google's full list. Avoids querying for every URL.

### 7.5 Network: Bitcoin SPV clients
Light wallets ask full nodes to filter transactions by a Bloom filter — privacy + bandwidth savings.

### 7.6 CDN / proxy negative caching
"Is this URL in the catalog?" — Bloom filter prevents flood of origin requests for random URLs.

### 7.7 Spell checkers (historical)
Dictionary as Bloom filter; words not in dictionary identified immediately.

---

## 8. Cassandra/RocksDB Detailed Example

Modern LSM-tree databases (Cassandra, RocksDB, LevelDB, ScyllaDB) maintain many SSTables on disk. A read for key X may need to check all of them.

```
SSTables: 50 files on disk
naive lookup: read 50 files → 50 disk seeks per read
with Bloom filters: check 50 in-memory filters; expected to read ~1 file
```

Each SSTable has a Bloom filter sized to its contents (typically 10 bits/key, 1% FP). Result: **massive** read-amplification reduction.

Without Bloom filters, LSM-tree databases would be unusable for point lookups.

---

## 9. Sizing in Practice

You need:
- Expected number of elements `n`.
- Desired false-positive rate `p`.

```python
import math

n = 10_000_000  # 10M elements
p = 0.01        # 1% FP rate

m = int(- (n * math.log(p)) / (math.log(2) ** 2))  # ~95.85M bits
k = int((m / n) * math.log(2))                      # ~7 hash functions

# memory: 95.85M bits ≈ 12 MB
```

For 10M items at 1% FP: 12 MB and 7 hash functions per query. Trivial.

---

## 10. Hash Functions

Bloom filters need `k` independent hash functions. In practice:
- Use one fast hash function (xxHash, MurmurHash3) and derive k variants by `h1 + i*h2` (Kirsch-Mitzenmacher trick).
- This is mathematically equivalent (asymptotically) and much faster.

Avoid cryptographic hashes — too slow. Bloom values speed.

---

## 11. Worked Example: Avoiding Cache Penetration

Scenario: 100M user IDs in DB. Attackers enumerate random IDs to slam DB.

### Naive
- Cache key `user:<id>` with miss for nonexistent.
- Cache penetration: random IDs always miss, hit DB.

### Negative caching alone
- Cache "not found" for short TTL. Helps repeat attacks.
- Doesn't help random attacks (each random ID is a fresh miss).

### Bloom filter at the front
```python
bloom = BloomFilter(capacity=200_000_000, error_rate=0.001)
# load all user IDs into the filter at startup (or maintain on insert)

def get_user(uid):
    if not bloom.contains(uid):
        return None     # cheap path: definitely doesn't exist
    v = cache.get(uid)
    if v: return v
    v = db.find(uid)
    if v: cache.set(uid, v)
    return v
```

Now even random attacks short-circuit at the Bloom filter — no cache hit, no DB query. 99.9% of random nonexistent IDs blocked at memory speed.

Cost: maintain the filter. Add users → insert into filter. Delete users → can't delete from Bloom (use counting Bloom or rebuild periodically).

---

## 12. Worked Example: SSTable Lookup

Cassandra-style. Read key `K`.

```
1. Check memtable (in-memory recent writes) — fast.
2. Check row cache (if enabled) — fast.
3. For each SSTable on disk (potentially dozens):
   a. Check Bloom filter — in memory, ~1 µs.
   b. If "definitely not", skip this SSTable.
   c. If "probably", read the index, then the data.
4. Return earliest non-nil result.
```

Without Bloom: dozens of disk seeks for keys that don't exist. With Bloom: typically 0–1 disk seeks for nonexistent keys.

Bloom filters added ~5–15% memory overhead to Cassandra in exchange for **orders of magnitude** read throughput on negative lookups.

---

## 13. Limitations

- **No deletes** (without variants).
- **Fixed size**: you must know `n` upfront. Scalable Bloom mitigates.
- **False positives can't be 0**: scale `m` for low FP, but never 0.
- **Bit-level data structure**: not human-readable; hard to debug.
- **Lookups are O(k)**: not bad, but k can be 10+ for low FP. Cuckoo / Xor filters can be faster.
- **Cache locality**: random hashes touch random bits → cache-unfriendly. Blocked Bloom filters partition into cache-line-sized chunks for better speed.

---

## 14. Common Mistakes

- **Treating "probably yes" as definite.** Always verify with the real source.
- **Sizing too small for actual n.** FP rate explodes once you exceed planned capacity.
- **Adding deletes naively.** Causes false negatives. Use Counting Bloom or rebuild.
- **Using cryptographic hashes.** Too slow.
- **Sharing one Bloom for many sets.** False positive rate accumulates.
- **No periodic rebuild.** Long-running filters can degrade if data churns.
- **Not measuring actual FP rate.** Always log "Bloom said yes, verification said no" so you can verify your error rate matches assumptions.
- **Bloom filter on the wrong path.** Use it for "definitely not" short-circuits; useless if you must check the source anyway.

---

## 15. Cheat Card

```
WHAT          probabilistic set membership; false positives, no false negatives

WHEN          checking "is X definitely NOT in the set?" cheaply
              short-circuit expensive lookups (disk, network, DB)

STRUCTURE     bit array of m bits, k hash functions
              insert: set k bits; query: check all k bits

MATH          ~10 bits/element for 1% FP, 7 hash functions
              ~14 bits/element for 0.1% FP, 10 hash functions
              m = -n·ln(p)/(ln 2)²;  k = (m/n)·ln 2

LIMITS        no deletes (without Counting Bloom)
              fixed size (need n upfront, or use Scalable Bloom)

VARIANTS      Counting Bloom (deletes), Scalable Bloom (grows),
              Cuckoo Filter (deletes + better), Xor Filter (faster lookup)

USED BY       Cassandra, HBase, RocksDB, LevelDB (skip SSTables)
              Bigtable, ScyllaDB
              CDN / cache (prevent penetration)
              Chrome safe-browsing
              Bitcoin SPV

DESIGN        size for max n; pick FP rate; use fast hash

PITFALLS      treating "probably" as definite; under-sized filter;
              naive deletes; not measuring actual FP

RULE          "Definitely not" is a powerful short-circuit.
              Always verify the "probably yes" against ground truth.
```

---

## 16. Resources

### Papers
- "Space/time trade-offs in hash coding with allowable errors" — Burton Bloom, 1970 (the original).
- "Less hashing, same performance: Building a better Bloom filter" — Kirsch & Mitzenmacher, 2008 (the two-hash trick).
- "Cuckoo Filter: Practically Better than Bloom" — Fan et al., 2014.
- "Xor Filters: Faster and Smaller Than Bloom and Cuckoo Filters" — Graf & Lemire, 2020.

### Books
- *Database Internals* — Alex Petrov (Bloom filters in storage engines).
- *Designing Data-Intensive Applications* — Kleppmann.

### Articles
- "Bloom filters by example" — Bill Mill (visual explanation).
- "How Cassandra uses Bloom filters" — DataStax blog.
- "RocksDB's Bloom filter implementation" — RocksDB blog.

### Videos
- ByteByteGo — "Bloom Filters Explained".
- Brilliant.org animated explainers.

### Tools / Libraries
- **Python**: `bloom-filter2`, `pybloom-live`.
- **Go**: `github.com/willf/bloom`, `github.com/seiflotfy/cuckoofilter`.
- **Java**: Guava's `BloomFilter`.
- **Rust**: `bloomfilter`, `growable-bloom-filter`.
- **Redis**: `RedisBloom` module.

### Adjacent reading
- [Probabilistic Data Structures →](./probabilistic-data-structures.md)
- [Storage Engines →](../09-storage/storage-engines.md)
- [Database Indexing →](../04-databases/indexing.md)
- [Cache Pitfalls →](../05-caching/cache-pitfalls.md)
- [Wide-Column Stores →](../04-databases/wide-column-stores.md)
- [Merkle Trees →](./merkle-trees.md)

---

*Previous:* [← Gossip Protocol](./gossip-protocol.md)  |  *Next:* [Count-Min Sketch & HyperLogLog →](./probabilistic-data-structures.md)
