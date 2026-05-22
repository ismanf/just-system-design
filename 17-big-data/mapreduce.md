# MapReduce

> **TL;DR** — **MapReduce** is a programming model for batch-processing huge datasets across a cluster of commodity machines. You write two functions — **map** (transform each input record into key/value pairs) and **reduce** (combine all values sharing a key) — and the framework handles parallelism, data movement, fault tolerance, and recovery. Introduced by Google in 2004 and popularized by open-source **Hadoop**, MapReduce is what made "big data" mean something — it let companies analyze terabytes on cheap commodity hardware when prior systems demanded specialty supercomputers. Today, **almost nobody writes raw MapReduce jobs anymore** — Spark, Flink, BigQuery, Snowflake, and Trino all do the same work faster and with less code. But the *conceptual model* is alive everywhere: every modern distributed analytics engine is MapReduce + a smarter execution layer. Understanding MapReduce is understanding how the entire data-processing ecosystem thinks.

---

## 1. The big picture

```
Input files (HDFS / S3)
   │
   ▼
┌────────────────────────────────────────┐
│       MAP                              │
│  (input record) → [(k1, v1), (k2, v2)] │
│  parallel across N mappers             │
└────────────────────────────────────────┘
   │
   ▼  (shuffle: group all values by key, send to right reducer)
┌────────────────────────────────────────┐
│       REDUCE                           │
│  (k, [v1, v2, ...]) → output           │
│  parallel across M reducers            │
└────────────────────────────────────────┘
   │
   ▼
Output files
```

You write `map(record)` and `reduce(key, values)`. The framework does everything else:

- Splits input into chunks (typically the HDFS block size — 128 MB).
- Schedules mapper tasks on machines that already have the data (data locality).
- Runs them in parallel.
- Collects key/value pairs, sorts and groups by key.
- Sends each key's values to one reducer (the shuffle).
- Runs reducer tasks in parallel.
- Detects failed tasks, reschedules them on healthy nodes.
- Writes output back to distributed storage.

The model's brilliance is its simplicity. Two functions, embarrassingly parallel, with a well-defined coordination step in the middle. That's enough to process petabytes.

---

## 2. The canonical example — word count

The "Hello World" of MapReduce:

```python
# Map: emit (word, 1) for every word in every line
def mapper(line):
    for word in line.split():
        yield (word.lower(), 1)

# Reduce: sum all 1s for each word
def reducer(word, counts):
    yield (word, sum(counts))
```

Input: billions of lines across thousands of files.
Output: one entry per unique word with total count.

The same code runs on 10 GB of text on 5 machines or 10 TB on 5000 machines. **You don't change the code; you change the cluster.** That's the magic.

---

## 3. The five phases in detail

### 3.1 Input split

The framework divides input into *splits* — chunks roughly the size of one HDFS block. Each split becomes one mapper task.

```
file1.gz (5 GB)  ─►  ~40 splits of 128 MB each
file2.gz (3 GB)  ─►  ~24 splits of 128 MB each
                     ──────────────────────────
                     64 mapper tasks total
```

Why 128 MB? It's the sweet spot — large enough that task scheduling overhead is amortized, small enough that a single failed task is cheap to retry.

### 3.2 Map

Each mapper reads its split, runs `map()` on every record, and emits key/value pairs to local disk. Crucially, **mappers run on the same node that stores the data when possible** ("data locality" — moving compute to data, not data to compute).

### 3.3 Combine (optional)

A *combiner* is a mini-reducer that runs on the mapper's output before shuffle. For word count, that's `reduce(word, counts) → (word, sum)` applied locally. The shuffle then ships one `(the, 3247)` pair instead of 3247 individual `(the, 1)` pairs. Network savings are enormous; this is the single biggest practical optimization in MapReduce.

Combiners must be **associative and commutative** — the framework may call them zero, one, or many times.

### 3.4 Shuffle

The hard part. Every reducer gets all values for "its" keys (determined by a partition function — usually `hash(key) % num_reducers`). The framework:

- Sorts mapper output by key.
- Transfers it across the network to the right reducer.
- Merges multiple sorted streams arriving at the reducer.

Shuffle is the **most expensive phase of any MapReduce job**. It's also where every failure mode lives — slow nodes, network blips, skewed keys, OOMs on the reducer side. Most "my Hadoop job is slow" tuning is shuffle tuning.

### 3.5 Reduce

Each reducer receives `(key, [v1, v2, ...])` for its assigned keys, runs `reduce()`, writes output. Outputs typically go back to HDFS / S3 as one file per reducer.

---

## 4. Fault tolerance — why MapReduce won

The brilliance isn't the map/reduce idea (functional programmers had it for decades). It's the *runtime* that makes batch jobs reliable on flaky commodity hardware.

- **Task failures**: a mapper or reducer crashes → the master reschedules it on another node. Inputs are immutable; the retry produces the same output.
- **Slow tasks (stragglers)**: a task taking too long → run a *speculative copy* on another node. Use whichever finishes first.
- **Node failures**: lose a node → all in-flight tasks get rescheduled. Map output stored on local disk is lost; affected mappers are re-run.
- **Master failure**: classic Hadoop had a single master (a real weakness, later mitigated). Job state is persisted; on restart, the master resumes.

What makes this work: **inputs are immutable, tasks are deterministic, outputs are versioned**. Retries are safe because the same input always yields the same output. Functional purity is operational power.

For huge clusters, this matters enormously. A 1000-node job has many failures *per run*. Without runtime fault tolerance, the job would never finish. With it, the user doesn't even notice.

---

## 5. Data locality — the throughput trick

Networks are slow compared to disks. MapReduce knows this and schedules computation on the node that stores the data. The hierarchy:

1. **Node-local** — the mapper runs on a machine that holds the block. Read from local disk. Fast.
2. **Rack-local** — same rack, different machine. Read over the top-of-rack switch. Medium.
3. **Off-rack** — different rack. Read over the data center spine. Slow.

A well-tuned Hadoop cluster sees >90% node-local reads. That's why these systems scale to petabytes: you're streaming from disks at the *full aggregate disk bandwidth of the cluster*, not bottlenecked on cross-rack network.

The lesson generalizes: in big-data systems, **move the code, not the data**. Spark, Presto/Trino, Snowflake, BigQuery — all internalize this rule, even where they hide the mechanics.

---

## 6. What MapReduce is bad at

The model is restrictive in deliberate ways. That restriction is what makes it parallelize. It also makes some workloads awkward:

- **Iterative algorithms** — PageRank, gradient descent, graph traversals. Each iteration writes output to disk, the next reads it back. The disk round-trip dominates. Spark's in-memory model came specifically to fix this.
- **Interactive queries** — startup overhead (tens of seconds), shuffle on disk. Forget sub-second response.
- **Streaming / real-time** — MapReduce is batch by design. Bounded inputs, batch outputs.
- **Joins of comparable-sized tables** — possible but painful; the user must encode the join as map-then-reduce manually.
- **Complex multi-stage workflows** — chaining MR jobs gets tedious. Higher-level languages (Pig, Hive) and successors (Spark) hide this.
- **Small data** — the startup overhead of even a tiny MR job is measured in seconds. Don't use it for anything that fits in memory.

By 2014, Spark had displaced MapReduce for most analytics. By 2020, Spark itself was being displaced by SQL engines (Snowflake, BigQuery, Trino) for queries and by streaming engines (Flink) for real-time. MapReduce remains the conceptual ancestor — but the actual code is mostly retired.

---

## 7. Common MapReduce patterns

Even though you may never write a raw MR job, the patterns appear everywhere — in Spark, in SQL plans, in Beam pipelines. Recognize them:

### Filtering and projection

`map` emits a filtered/projected subset. No reduce stage needed (often called *map-only* jobs).

### Counting / aggregation

The word-count shape. `map → combine → reduce → sum/count/avg/max/min`.

### Inverted index

```
map: (doc_id, text) → [(word, doc_id), (word, doc_id), ...]
reduce: (word, [doc_id, ...]) → (word, sorted unique doc_ids)
```

This is the literal algorithm Google built to index the web. See [Inverted Indexes →](../19-advanced/inverted-index.md).

### Reduce-side join

```
map: emit (join_key, ("A", record)) for table A
     emit (join_key, ("B", record)) for table B
reduce: for join_key, separate A's and B's, emit cross product
```

Works for joins of any size but does a full shuffle of both tables.

### Map-side join (broadcast join)

If one table is small (fits in memory), distribute it to every mapper and join in the map phase. No shuffle. **Foundational** in modern SQL engines (Spark, Trino — both call this "broadcast join").

### Composite key + secondary sort

Sort within a reducer's input by something beyond the key (e.g., timestamp). Pack the secondary into the key, write a custom partitioner. Manual but powerful.

### Top-K per group

`map` emits per-record values, `reduce` keeps a heap of top K per key. With a combiner, the network cost is bounded.

### Iterative algorithm (with multiple MR rounds)

```
PageRank pass 1 → output to HDFS
PageRank pass 2 → reads pass 1 output, writes to HDFS
...
```

This is the *exact* shape that motivated Spark — keeping state in memory across iterations.

---

## 8. Worked example — Stripe-style transaction analytics

Imagine 1 TB of payment events per day. Two daily questions:

1. Top 10 merchants by transaction count.
2. Median transaction amount per country.

### Top 10 merchants

```python
def mapper(event):
    yield (event.merchant_id, 1)

def combiner(merchant, counts):
    yield (merchant, sum(counts))

def reducer(merchant, counts):
    yield (merchant, sum(counts))

# Then a second pass: sort/limit-10 (often a single reducer)
def top_k_mapper(merchant, count):
    yield ("__all__", (merchant, count))

def top_k_reducer(_, pairs):
    yield sorted(pairs, key=lambda x: -x[1])[:10]
```

Two MR jobs chained.

### Median per country

Medians are hard for MapReduce because they aren't decomposable (you need the full sorted list). Two practical approaches:

- **Brute force**: emit `(country, amount)` from map; reducer sorts all values for its country and picks the middle. Works if no country is too big to fit in one reducer.
- **Approximate**: use t-digest or HDR Histogram, mergeable structures. Each mapper builds a sketch; combiner merges; reducer extracts the percentile. Constant memory, approximate answer.

This illustrates a general truth: **MapReduce favors operations that are associative and decomposable**. When they aren't, you either pay the cost (shuffle huge per-key data) or approximate.

---

## 9. Higher-level languages on top

Writing raw map/reduce is tedious. Two languages won at hiding it:

- **Pig Latin** — a procedural dataflow language: `LOAD → FILTER → JOIN → GROUP → STORE`. Compiles to MR jobs. Still occasionally seen at large Hadoop shops.
- **Hive (HiveQL)** — SQL on Hadoop. Compiles to MR (later Tez, then Spark). Made data-warehouse-style queries possible on HDFS at scale. **Still widely used**, though usually on engines that aren't classical MR anymore.

Both are case studies in the general lesson: **most users want SQL, not lambdas**. The raw API is a primitive; the SQL on top is the product.

---

## 10. From MapReduce to DAGs

MapReduce is two stages: map → reduce. Real jobs need more — joins, multi-stage aggregations, iteration. Classical Hadoop chained jobs by reading and writing HDFS between them. That meant disk I/O between every stage.

The successors generalized to **directed acyclic graphs (DAGs)** of stages:

- **Apache Tez** — DAG executor on top of YARN, faster successor to classic MR.
- **Spark** — in-memory DAG engine; stages connected via memory or shuffle, recompute on failure via lineage.
- **Flink** — DAGs for streaming and batch unified.
- **BigQuery / Dremel** — pipeline of leaf workers and mixers, similar shape under the hood.

The vocabulary survives: every modern engine's execution plan looks like a DAG of "stages" with "shuffles" between them. **MapReduce was the two-node version of this graph.**

---

## 11. When MapReduce is still the right tool

A short list — increasingly short, but real:

- **Massive one-shot batch over object storage** where startup time and disk shuffle are fine. Cheap to operate; mature tooling.
- **Mature Hadoop estates** with running pipelines, on-prem, in a stable state. Migration cost > value.
- **Workloads with extreme input sizes (PB+)** that need to stream from disk at full aggregate bandwidth and have no in-memory option.
- **Learning** — the model itself is worth knowing because every successor borrows from it.

For new analytics work in 2026: pick **Spark**, **Flink**, **BigQuery**, **Snowflake**, **Databricks**, or **Trino**. Use Hive metastore + Iceberg/Delta tables on object storage. Skip writing MR jobs in the dialects of yore.

---

## 12. Common Mistakes / Anti-Patterns

- **Skewed keys.** One reducer gets 80% of the data; the rest sit idle. Use salted keys, custom partitioners, or two-stage aggregation.
- **No combiner where one applies.** Shuffle cost 100×; jobs take 10× longer.
- **Tiny files.** 10 million 50KB files → 10 million mappers → scheduler dies. Compact small files first.
- **One giant reducer.** "Order by global sort" on terabytes is a single reducer. Use TotalOrderPartitioner or sample-based partitioning.
- **Writing per-record output in the mapper.** Output buffering matters; the framework expects emit-and-go.
- **Side-effects in map/reduce.** The framework retries tasks; side effects fire twice. Make them idempotent or put them outside the job.
- **MapReduce for small data.** Startup overhead dwarfs the work.
- **MapReduce for iterative algorithms.** Disk on every iteration; Spark is the answer.
- **Confusing combiner with reducer.** Combiner must be associative/commutative; reducer doesn't have to be in general but does in practice if you also use it as a combiner.
- **Forgetting data locality.** Cross-rack reads ruin throughput.
- **Ignoring serialization.** Avro / Parquet beat plain text. Compression beats none.
- **Treating MR as a real-time pipeline.** It's not; the latency floor is seconds to minutes.
- **Custom partitioners without testing.** Get the hash wrong, the whole job is a no-op.
- **Reducer that holds all values in memory.** Reducers should stream their input, not buffer it.

---

## 13. Cheat Card

```
PURPOSE   Batch-process huge datasets across many machines by
          writing only map() and reduce() — the runtime does
          parallelism, shuffle, locality, and fault tolerance.

CORE MODEL
  map(record)        → [(k, v), ...]
  combine(k, [v])    → [(k, v')]            (optional; assoc + commutative)
  shuffle             → group all v's by k across the cluster
  reduce(k, [v])     → output

THE FIVE PHASES
  1. input split   (~128 MB chunks, one task each)
  2. map           (data-local; emit k/v)
  3. combine       (mapper-side mini-reduce; CRUCIAL for shuffle cost)
  4. shuffle       (sort, partition, send to reducers — the hot phase)
  5. reduce        (per-key aggregate; write output)

PATTERNS
  Filter / project / count / aggregate
  Inverted index, top-K, reduce-side join
  Broadcast (map-side) join when one table fits in memory
  Iterative algorithms — switch to Spark

WHY IT WON (HISTORICALLY)
  Data locality (compute moves to data)
  Fault tolerance (retry on failure, speculative on stragglers)
  Simple model — runs on commodity hardware
  Linear scaling to thousands of nodes

WHY IT'S RETIRED (NOW)
  Disk shuffle every stage → slow
  No real iteration or streaming
  Higher-level engines (Spark, Flink, BigQuery, Snowflake)
    do the same job 10–100× faster

PITFALLS
  Skewed keys (1 reducer eats everyone)
  No combiner where one applies
  Tiny-files problem
  One giant final reducer
  Iterative algorithms in pure MR
  Side effects in map/reduce
  Reducer that buffers all values in RAM
  Small data → MR overhead > the work itself

RULE   Understand the model; rarely write the code. Every modern
       engine is a smarter execution layer on top of these ideas.
```

---

## 14. Resources

### Papers
- "MapReduce: Simplified Data Processing on Large Clusters" — Jeffrey Dean and Sanjay Ghemawat (2004): <https://research.google/pubs/pub62/> (the seminal paper)
- "The Google File System" — Ghemawat, Gobioff, Leung (2003).
- "Bigtable: A Distributed Storage System for Structured Data" — Google (2006).

### Books
- *Hadoop: The Definitive Guide* (4th ed.) — Tom White. The MapReduce-on-Hadoop reference.
- *Designing Data-Intensive Applications* — Martin Kleppmann. Chapter 10 covers the model and its evolution.
- *Data-Intensive Text Processing with MapReduce* — Lin & Dyer (free PDF).

### Documentation
- **Apache Hadoop MapReduce** — <https://hadoop.apache.org/docs/stable/hadoop-mapreduce-client/hadoop-mapreduce-client-core/MapReduceTutorial.html>
- **Apache Hive** — <https://hive.apache.org/>
- **Apache Pig** — <https://pig.apache.org/>

### Articles
- "Why MapReduce is dead" — many post-2014 critiques; pair with "what replaced it."
- "Anatomy of a MapReduce job" — Cloudera engineering blog series.
- "MapReduce design patterns" — book by Miner & Shook.

### Videos
- *MapReduce: The Programming Model* — Jeff Dean (many talks online).
- *Hadoop ecosystem overview* — Cloudera / Hortonworks legacy talks.
- ByteByteGo — "MapReduce Explained."

### Adjacent reading
- [Hadoop Ecosystem →](./hadoop.md)
- [Apache Spark →](./spark.md)
- [Apache Flink →](./flink.md)
- [ETL vs ELT →](./etl-vs-elt.md)
- [Data Pipelines & Orchestration →](./data-pipelines.md)
- [Data Modeling for Analytics →](./dimensional-modeling.md)
- [Distributed File Systems →](../09-storage/distributed-file-systems.md)
- [OLTP vs OLAP →](../04-databases/oltp-vs-olap.md)
- [Data Warehouses & Data Lakes →](../04-databases/warehouses-lakes.md)
- [Lakehouse Architecture →](../04-databases/lakehouse.md)
- [Batch vs Stream Processing →](../07-messaging/batch-vs-stream.md)

---

*Previous:* [← Tail Latency & p99](../16-performance/tail-latency.md)  |  *Next:* [Hadoop Ecosystem →](./hadoop.md)
