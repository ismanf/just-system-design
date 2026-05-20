# Count-Min Sketch & HyperLogLog

> **TL;DR** — Two more **probabilistic data structures** that trade exact answers for massive memory savings. **Count-Min Sketch (CMS)** estimates how many times each element appears in a stream — like a hash-table-of-counts but with bounded memory. It overestimates but never underestimates (frequencies-floor). Useful for **heavy hitters**, **top-k**, **frequency thresholds** at huge stream rates. **HyperLogLog (HLL)** estimates the **cardinality** (number of distinct elements) of a multiset with ~1% error using ~1.5 KB regardless of input size. Used everywhere: unique-visitor counts, distinct IPs, "how many distinct queries hit this endpoint." Like Bloom filters (see [Bloom Filters →](./bloom-filters.md)), these structures embrace bounded inaccuracy in exchange for **tiny memory** and **constant-time ops**. Use them when "approximately right is fine and exactly right would cost a fortune."

---

## 1. Why Probabilistic Structures

Counting and cardinality look trivial on small data. On streams of billions of events:

- **Exact count of each element**: a HashMap. Memory = O(N) distinct keys. For 1B distinct keys, GBs.
- **Exact cardinality**: a HashSet. Same memory.

Probabilistic alternatives:

- **CMS**: estimate frequency with ~constant memory. Error bounded.
- **HLL**: estimate distinct count with ~1.5 KB. Error ~1%.

Both are O(1) per insert and per query. Both work on streams. Both are mergeable across partitions (sum / merge their state). All these properties make them **darlings of stream processing**.

---

## 2. Count-Min Sketch (CMS)

### Structure
A 2D array of counters: **width w**, **depth d** (rows). `d` hash functions, each independent.

```
        col 0   col 1   col 2   ...   col w-1
row 0    [   ]   [   ]   [   ]         [   ]
row 1    [   ]   [   ]   [   ]         [   ]
row 2    [   ]   [   ]   [   ]         [   ]
row d-1  [   ]   [   ]   [   ]         [   ]
```

### Insert
For each item `x`:
- For each row `i`: increment `cms[i][hash_i(x) % w]`.

### Query (frequency estimate)
- For each row `i`: get `cms[i][hash_i(x) % w]`.
- Return the **minimum** of these.

### Why minimum?
Each cell holds the count of all items hashed to it. The minimum across rows is the cell *least polluted* by collisions — closest to the true count.

```
insert "alice" 5 times:
row 0, col(hash_0(alice)): +5
row 1, col(hash_1(alice)): +5
row 2, col(hash_2(alice)): +5

now "bob" collides at row 0, col(hash_0(bob)) = same as alice
that row's count for alice becomes 5 + (bob's count) — inflated

min over rows: probably picks a row where bob didn't collide → 5 (correct)
```

### Error bounds
With width `w = ceil(e / ε)` and depth `d = ceil(ln(1/δ))`:
- Estimate error ≤ `ε × N` with probability `≥ 1 - δ`.

Where `N` is the total stream size.

Examples:
- ε = 0.01, δ = 0.001: w = 272, d = 7. Memory = 1904 counters (~8 KB).
- ε = 0.001, δ = 0.0001: w = 2719, d = 10. Memory = ~110 KB.

For trillion-event streams: ~MB of memory; queries in microseconds.

### Properties
- **Over-estimates only** — frequency lower-bound is the true count.
- **No deletes** in standard CMS (deletes would break the minimum guarantee).
- **Mergeable** — sum two CMS sketches element-wise to combine.

### Variants
- **Count-Min-Log Sketch** — better for highly skewed distributions.
- **Conservative Update CMS** — only increment the minimum cell. Tighter estimates.

---

## 3. CMS Use Cases

### Heavy hitters (top-k frequent items)
Combine CMS with a min-heap of top-k candidates. Each item's CMS count determines if it's heap-worthy.

Used by: trending topics, traffic anomaly detection.

### Frequency threshold
"Has any IP made > 1000 requests in this minute?" — query CMS per IP. If estimate > threshold, alert.

Used by: DDoS detection, rate limiting at scale.

### Real-time analytics
"How often did each search query appear today?" — CMS per query. Read top-k periodically.

Used by: Twitter Heron / Storm, Apache Spark Streaming.

### Approximate counters in DBs
ClickHouse, Druid, Apache Pinot all use CMS internally for cardinality and frequency aggregations.

---

## 4. HyperLogLog (HLL)

The "how many distinct things have I seen?" structure.

### The intuition
Hash each element to a uniformly random bit string. The position of the first 1 in the hash, on average, doubles for every doubling of input distinct count.

- 1 distinct element → expect to see at most 1-bit leading zeros on average.
- 1000 distinct elements → expect maximum ~10 leading zeros.
- 10⁹ → expect ~30 leading zeros.

So: **the longest run of leading zeros seen in the hashes** is approximately `log2(distinct count)`.

This is the **probabilistic counting** core. HLL refines it.

### HLL structure
- m registers (buckets), each storing a small value (typically 6 bits = max value 63).
- Total memory: `m × 6 bits`. Standard HLL: m = 16384 = 1.5 KB. Error ~1%.

### Insert
For element `x`:
1. Compute `h(x)` (64-bit hash).
2. Use the first `log2(m)` bits to pick a register.
3. Count leading zeros in the rest; +1.
4. Update register if this leading-zero count > current value.

```
m = 16, h(x) = 0010 1101 ...
bucket: first 4 bits = 0010 = 2
remaining bits: leading zeros = ?
register[2] = max(register[2], leading_zeros + 1)
```

### Query (cardinality)
Combine all registers via a corrected harmonic mean.

```
estimate = α_m × m² / sum(2^(-register[i]))
```

`α_m` is a bias correction constant.

### Error
With m = 16384 registers (1.5 KB), standard error is ~0.81%. Smaller m gives less memory at higher error. Larger m goes down to ~0.1% error for ~12 KB.

The error is **relative** — counting 1 million distinct ± 1%. Independent of actual cardinality.

### Properties
- **Mergeable** — register-wise max → union cardinality of two HLLs.
- **No deletes** — registers only grow.
- **Constant memory regardless of input size.**
- **Probabilistic** — variance, not bias; over- or under-count by small margin.

---

## 5. HLL Use Cases

### Unique visitor count
Redis's `PFCOUNT`. 1.5 KB per "uniques" key.

```redis
PFADD uniques:home user1 user2 user3
PFADD uniques:home user1   # duplicate, doesn't grow much
PFCOUNT uniques:home       # ≈ 3
```

### Unique IPs / queries / events
DDoS detection: "how many unique IPs hit this URL in 5 min?" — HLL per URL per 5-min window.

### Aggregation across shards
Per-shard HLL → merge for global count.

```
shard_0: HLL_A
shard_1: HLL_B
shard_2: HLL_C
global: merge(A, B, C) → cardinality estimate of total set
```

This **mergeability** is HLL's killer feature for distributed systems.

### Analytics
Google BigQuery's `APPROX_COUNT_DISTINCT` = HLL.
Snowflake `APPROX_COUNT_DISTINCT` = HLL.
Postgres `count(DISTINCT)` is exact and slow on large data; HLL extension gives approx fast.
Apache Druid, ClickHouse, Apache Pinot, Presto all use HLL.

### Set operations on cardinalities
- **Union** — merge HLLs. Exact.
- **Intersection** — `|A ∩ B| = |A| + |B| - |A ∪ B|`. Inclusion-exclusion. Less accurate when sets overlap heavily.
- **Difference** — similar.

---

## 6. HyperLogLog++ and Variants

Refinements over the years:
- **HyperLogLog++** (Google, 2013) — bias-corrected at small cardinalities; sparse representation for low counts (smaller memory until you hit a threshold).
- **HLL-TailCut+** — better register encoding for further savings.
- **Compressed HLL** — varint encoding.

Most production HLLs are HLL++. Redis, BigQuery, Druid implementations all use it.

---

## 7. Other Probabilistic Structures (Briefly)

A small zoo. Each solves a specific approximate problem.

### Top-K (CMS + heap)
Find heavy hitters in a stream. CMS for counts; heap for the top items.

### MinHash
Estimate **Jaccard similarity** between sets. Used in plagiarism detection, near-duplicate detection (Google's web crawl, news clustering).

### t-Digest
Estimate quantiles (p99, p999) on streams with high accuracy. Used in monitoring, metrics aggregation.

### KLL Sketch
Newer streaming quantile sketch — provable bounds, simpler than t-digest.

### Skip Lists
Probabilistic ordered structures. Redis sorted sets, LevelDB use skip lists. See [Skip Lists →](../19-advanced/skip-lists.md).

### Bloom Filter
Set membership. See [Bloom Filters →](./bloom-filters.md).

### Quotient Filter, Cuckoo Filter, Xor Filter
Bloom alternatives with deletes / better memory.

### Reservoir Sampling
Sample k items from a stream of unknown size, uniformly at random.

These structures share a theme: **bounded inaccuracy + constant or sub-linear memory**.

---

## 8. Combining Sketches

Real systems combine sketches for richer queries.

### CMS + HLL
- CMS: per-element frequency estimate.
- HLL: total distinct count.

Together: "what fraction of distinct visitors are returning?"

### MinHash + LSH (Locality-Sensitive Hashing)
Find near-duplicates in huge datasets. Used in plagiarism detection, web crawl deduplication, recommendation systems.

### CRDTs + counters
G-Counter, PN-Counter for replicated approximations.

---

## 9. When to Use Sketches

```
Need exact answer + dataset fits in memory?
  → HashMap / HashSet / counter table.

Stream / huge dataset, ε% error OK?
  → CMS for frequencies, HLL for cardinality.

Distributed (shards across machines), need global aggregate?
  → Mergeable sketches (HLL, CMS) — sketch per shard, merge centrally.

Real-time + low memory?
  → Sketches always shine here.

User-facing exact counters (followers, likes)?
  → Exact storage (databases, atomic counters).

Internal analytics, top-k, "approximately right is fine"?
  → Sketches.
```

The rule: **users care about exact for their own data; analytics tolerates approximation for system-wide stats**.

---

## 10. Worked Example: Real-Time Dashboard at Scale

Service: 10M events/sec, need "unique users per minute" and "top-100 most active users."

### Without sketches
- HashSet for unique users: hundreds of MB.
- HashMap for per-user counts: GBs.
- Memory dominates; aggregation across pods expensive.

### With sketches
- **HLL per 1-min bucket**: 1.5 KB × 60 minutes = 90 KB. Unique counts within 1% error.
- **CMS per 1-min bucket**: ~50 KB each. Per-user frequency estimates.
- **Top-100 heap**: 100 entries.

Total memory: ~5 MB per minute. Trivial. Mergeable across pods.

```
each pod:
  on event:
    hll.add(user_id)
    cms.add(user_id)
    top_k.consider(user_id, cms.estimate(user_id))

every minute, send hll+cms to central aggregator
aggregator: union all HLLs → global unique count
            sum all CMSs → global frequencies
            merge top-k candidates
```

This is what Twitter Heron, Apache Storm, Apache Flink running real-time stream pipelines actually do.

---

## 11. Operational Considerations

### Choosing parameters
- HLL: 16384 registers (1.5 KB) for ~1% error — standard default.
- CMS: width 2000, depth 5 typical for ε=0.001, δ=0.007.

### Serialization
Sketches need to serialize for storage / network. Most implementations have compact formats. Don't roll your own.

### Merging
Mergeability is the killer feature. Each sketch supports a merge operation. Plan your topology so sketches accumulate at edges and merge upstream.

### Periodic reset
For windowed counts: reset every minute / hour. Time-bucketed sketches.

### Eviction in HLL
HLL has no eviction. To represent "uniques in last N minutes," maintain N HLLs (per minute) and union when querying.

---

## 12. Real-World Use Cases

- **Redis `PFADD`/`PFCOUNT`**: HLL. Every Redis user has had access to HLL since 2014.
- **Google Analytics**: HLL underpins much of the cardinality counting.
- **Twitter**: stream-processing pipelines use CMS + HLL for trending and active-user counts.
- **Reddit**: similar patterns for sub-counts.
- **CloudFlare**: edge analytics use HLL extensively.
- **Druid, ClickHouse, BigQuery, Snowflake**: `APPROX_COUNT_DISTINCT` everywhere.
- **HyperLogLogLog (a real paper)**: even more compressed HLL variant.

---

## 13. Common Mistakes

- **Using CMS for exact counts.** Returns over-estimates; users see inflated numbers. Use exact for user-facing.
- **HLL for very small cardinalities (< 100).** HLL is over-engineered here; use a set.
- **No bias correction.** Vanilla HLL has bias at small N; use HLL++.
- **Forgetting sketches don't delete.** "User unfollowed" can't be subtracted from a non-PN counter.
- **Merging incompatible sketches.** Same parameters required.
- **Underestimating CMS variance.** With heavy collisions, individual estimates skew badly. Top-k mitigates.
- **Treating sketches as exact in audits.** Always note approximate nature; users see error.
- **Memory overhead for many small sketches.** 1.5 KB × millions of keys = GBs. Reduce HLL precision or use sparse encoding.

---

## 14. Cheat Card

```
SKETCH        probabilistic data structure: small memory, approximate answers

COUNT-MIN SKETCH (CMS)
  estimates frequency of each element in a stream
  2D array of counters; k hash rows; min over rows on query
  over-estimates only; no deletes (standard form)
  use: heavy hitters, top-k, frequency thresholds

HYPERLOGLOG (HLL)
  estimates distinct count of a multiset
  ~1.5 KB for ~1% error, regardless of input size
  registers track max leading zeros per hash bucket
  use: unique visitors, distinct IPs, cardinality

MERGEABILITY  both CMS and HLL merge: shard-local sketches → global

ERROR
  HLL: 1% with 1.5 KB
  CMS: ε × N with width/depth tuning

ALSO          MinHash (similarity), t-Digest (quantiles), Reservoir Sampling

WHEN TO USE   massive streams, distributed aggregation, OK with ε error

NOT FOR       per-user exact counts (likes, followers, balances)

PITFALLS      exact user-facing, no deletes, mismatched merge params,
              undersized for true scale, treating estimates as facts

RULE          Sketches let you summarize universes with a teacup
              of memory. Use them where users won't audit the count.
```

---

## 15. Resources

### Papers
- "An improved data stream summary: the count-min sketch and its applications" — Cormode & Muthukrishnan, 2005 (CMS).
- "HyperLogLog: the analysis of a near-optimal cardinality estimation algorithm" — Flajolet et al., 2007.
- "HyperLogLog in Practice: Algorithmic Engineering of a State of the Art Cardinality Estimation Algorithm" — Heule, Nunkesser, Hall (Google), 2013 (HLL++).
- "Approximate Counting in a Single Pass" — various follow-ups.
- "Mining of Massive Datasets" — Leskovec/Rajaraman/Ullman (free textbook covering MinHash, LSH).

### Books
- *Designing Data-Intensive Applications* — Kleppmann.
- *Database Internals* — Petrov.
- *Mining of Massive Datasets* — Stanford.

### Articles
- "Sketching for big data" — Florian Müller and several blog posts.
- "Redis HyperLogLog" — Redis docs and antirez blog.
- "Approximate count distinct in BigQuery" — Google Cloud blog.

### Videos
- ByteByteGo — "Probabilistic Data Structures".
- Stanford CS246 course videos.
- Strange Loop talks on sketches and streams.

### Tools / Libraries
- **Redis**: `PFADD`, `PFCOUNT` (HLL).
- **Apache DataSketches**: `theta-sketches`, `hll`, `cpc`, `tuple`, `quantiles`.
- **Stream-lib** (Java): CMS, HLL, t-Digest.
- **algebird** (Scala): functional sketches at Twitter.
- **Python**: `datasketch`, `pdsa`.

### Adjacent reading
- [Bloom Filters →](./bloom-filters.md)
- [Merkle Trees →](./merkle-trees.md)
- [Gossip Protocol →](./gossip-protocol.md)
- [Stream Processing →](../07-messaging/stream-processing.md)
- [Real-Time Analytics →](../19-advanced/real-time-analytics.md)
- [Skip Lists →](../19-advanced/skip-lists.md)
- [Tail Latency & p99 →](../16-performance/tail-latency.md)

---

*Previous:* [← Bloom Filters](./bloom-filters.md)  |  *Next:* [Merkle Trees →](./merkle-trees.md)
