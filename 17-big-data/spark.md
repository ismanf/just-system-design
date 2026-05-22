# Apache Spark

> **TL;DR** — **Apache Spark** is a general-purpose distributed compute engine built around two ideas: **DAG execution** (compose operations into a graph, optimize and execute lazily) and **in-memory data sharing** (keep working sets in RAM across stages instead of going to disk between every step like MapReduce). The result is **10–100× faster batch processing**, plus first-class support for **SQL** (Spark SQL), **streaming** (Structured Streaming), **machine learning** (MLlib), and **graphs** (GraphX). Born at UC Berkeley AMPLab around 2010, it displaced MapReduce as the default big-data compute engine and became the operational core of Databricks. Today Spark sits beside Trino/Snowflake/BigQuery and Flink — each has a niche, but **Spark is the safe default for batch ETL, large-scale ML feature engineering, and lakehouse compute**. The honest take: **Spark + Iceberg/Delta on S3 + Airflow is one of the most boring, capable, and well-tooled data-platform shapes you can pick in 2026.**

---

## 1. The big picture

Spark's core model: a **driver** program describes a computation as a chain of transformations on distributed datasets. The driver builds a DAG of those transformations, optimizes it, and dispatches **stages of tasks** to **executors** that run on cluster workers.

```
┌──────────────────────────────────────────────────────────┐
│  Driver                                                  │
│  - parses user code                                      │
│  - builds DAG (transformations + actions)                │
│  - Catalyst optimizes the plan                           │
│  - DAG scheduler splits into stages                      │
│  - dispatches tasks                                      │
└────────────────┬─────────────────────────────────────────┘
                 │
        ┌────────┴────────┬───────────────┐
        ▼                 ▼               ▼
   ┌──────────┐      ┌──────────┐    ┌──────────┐
   │ Executor │      │ Executor │    │ Executor │
   │  cache   │      │  cache   │    │  cache   │
   │  tasks   │      │  tasks   │    │  tasks   │
   └──────────┘      └──────────┘    └──────────┘
        │                 │               │
        ▼                 ▼               ▼
   Object storage / HDFS / Kafka / JDBC / files
```

Operations are **lazy** — transformations build the plan; **actions** (collect, write, count) trigger execution. The optimizer (**Catalyst**) reorders, rewrites, and pushes down predicates before any task runs.

Three deployment modes you'll see:
- **Local** (one JVM) for development and unit tests.
- **Standalone / Mesos / YARN** (legacy clusters).
- **Kubernetes** (the modern way for self-managed Spark).
- **Managed**: Databricks, AWS EMR / Glue, GCP Dataproc / Serverless, Azure Synapse / HDInsight.

For new builds, Spark on Kubernetes or one of the managed services. Standalone is fine for dev; YARN is legacy Hadoop territory.

---

## 2. Why Spark beat MapReduce

Three reasons, in order of importance:

### 2.1 In-memory caching between stages

MapReduce wrote intermediate results to HDFS between every job. Iterative algorithms (PageRank, gradient descent) and multi-stage pipelines paid disk I/O on every iteration. Spark keeps working sets in RAM across stages, falling back to disk only when memory runs out.

For an iterative ML algorithm with 20 iterations, that's 20× less disk I/O. For batch ETL with 5 stages, that's 5× less.

### 2.2 DAG execution + Catalyst optimizer

You write a sequence of operations; Spark builds the full DAG and optimizes the whole plan before running anything. Predicate pushdown, projection pruning, join reordering, broadcast detection — all happen automatically. MapReduce executed exactly what you wrote.

### 2.3 Unified API

One engine that handles SQL, batch, streaming, ML, graphs. Same primitives, same cluster, same monitoring. MapReduce was just batch; the rest of the Hadoop ecosystem stitched in other tools.

This consolidation is the reason Spark became the default. You don't run five different engines; you run one.

---

## 3. RDDs, DataFrames, Datasets — the three APIs

Three APIs, layered, with progressively more semantic information for the optimizer.

### RDD (Resilient Distributed Dataset)

The original Spark API. A typed, immutable, partitioned collection of objects with lineage information for fault recovery.

```scala
val rdd = sc.textFile("s3://bucket/logs/*.log")
            .filter(line => line.contains("ERROR"))
            .map(line => (line.split(" ")(0), 1))
            .reduceByKey(_ + _)
rdd.collect()
```

Closures of arbitrary code (JVM, Python, Scala). Powerful but **opaque to Catalyst** — the optimizer can't see inside your lambdas.

Use RDDs today only when:
- You truly need arbitrary functions Catalyst can't express.
- You're doing custom partitioning or specific shuffle control.
- Backward compatibility.

Otherwise: don't. DataFrames are faster.

### DataFrame

Spark's tabular API. Rows with a schema; columnar internal representation; **Catalyst-optimized**.

```python
df = (spark.read.parquet("s3://bucket/orders/")
      .filter("country = 'US'")
      .groupBy("merchant_id")
      .agg(F.sum("amount").alias("total"))
      .orderBy(F.desc("total"))
      .limit(10))
df.show()
```

This is the default API for almost everything in modern Spark. Catalyst rewrites it into the same physical plan as the equivalent SQL.

### Dataset

A typed DataFrame in Scala/Java. Combines the type safety of RDDs with Catalyst optimization. Not available in Python (PySpark is essentially DataFrame only).

### Spark SQL

You can hand Spark a SQL string and it'll execute the same way as the DataFrame API:

```python
spark.sql("""
  SELECT merchant_id, SUM(amount) AS total
  FROM parquet.`s3://bucket/orders/`
  WHERE country = 'US'
  GROUP BY merchant_id
  ORDER BY total DESC
  LIMIT 10
""").show()
```

In production data engineering, most Spark code is a mix of DataFrame chaining and SQL strings — pick whichever reads better. Both go through Catalyst.

---

## 4. Execution: stages, shuffles, tasks

Spark splits a DAG into **stages** at shuffle boundaries. Within a stage, work is *pipelined* — operations chain without writing to disk. Across stages, data is **shuffled** between executors.

```
Stage 1                Stage 2 (after shuffle)
─────────              ────────────────────────
read parquet           re-grouped by merchant_id
filter country='US'    sum + sort
project amount, merch  return top 10
                                  ↑
                                  shuffle
```

Each stage runs as a set of **tasks** — one task per partition. If the input has 200 partitions, the stage runs 200 tasks in parallel (up to the number of executor slots).

### Shuffle — the expensive part

A shuffle rewrites data so that all rows with the same key land on the same executor. Mechanically: each task writes its output, partitioned by key, to local disk. The next stage's tasks pull those files over the network.

Shuffles cost:
- **Disk I/O** — write shuffle files locally, read on the consumer side.
- **Network** — transferring across the cluster.
- **Serialization** — encode/decode every row.
- **Memory pressure** — large shuffles spill to disk; spilling kills performance.

**Most Spark performance work is shuffle work.** Reduce shuffles (broadcast joins, partition-aligned data), or make them cheaper (smaller payloads, fewer columns, more compute per row).

### Tasks per partition

A partition is a chunk of data. The number of partitions sets the parallelism. Too few → underutilized cluster. Too many → task overhead dominates.

- Read partitions match the input format (Parquet file boundaries, Kafka partitions, JDBC predicates).
- Repartitioning (`df.repartition(N)`) reshuffles; `df.coalesce(N)` combines without shuffle (only downsize).
- Default shuffle partitions = 200. **For most pipelines this is wrong**. Set it based on data size and cluster size. AQE (Adaptive Query Execution) tunes it dynamically when enabled.

---

## 5. Adaptive Query Execution (AQE)

Spark 3.0+ added **AQE** — the optimizer re-plans during execution using observed runtime statistics. AQE does three big things:

- **Dynamic coalesce of shuffle partitions** — collapses too-small partitions after shuffle. Big win on tiny-partition jobs.
- **Switch sort-merge join to broadcast join** when one side turns out small. Catches missed broadcasts at runtime.
- **Skew handling** — splits heavy partitions and replicates the small side. Mitigates the #1 cause of Spark stragglers.

**Always enable AQE** (`spark.sql.adaptive.enabled=true`). It's off by default in older versions; for new clusters it's on.

---

## 6. Joins — the make-or-break operation

Spark's optimizer picks a join strategy based on size and stats:

| Strategy | When picked | Cost |
|---|---|---|
| **Broadcast hash join** | One side fits in `spark.sql.autoBroadcastJoinThreshold` (default 10 MB) | Fastest — no shuffle. Small side ships to every executor. |
| **Shuffle hash join** | Medium-sized | Shuffles both sides; builds a hash table on the smaller. |
| **Sort-merge join** | Both sides large | Shuffles + sorts both sides; merges. Most general. |
| **Broadcast nested loop** | Cross-join with one tiny side | Last resort; bad. |

Tuning levers:
- **`broadcast()` hint** — force a broadcast when stats are missing or wrong. `df1.join(broadcast(df2), "id")`.
- **`autoBroadcastJoinThreshold`** — raise it (e.g., 64–128 MB) when you have memory and small dimension tables.
- **Bucketing** — pre-shuffled tables co-located by hash. Joins on bucketed columns skip the shuffle.
- **Partition pruning** — predicates on partition columns avoid scanning whole tables.

Skewed joins are a tail-latency catastrophe — one task with a billion rows for `country='US'` while the others finish in seconds. AQE's skew handling, salting (add a random prefix to break up hot keys), or pre-aggregation are the cures.

---

## 7. Spark SQL and Catalyst

Catalyst is Spark's query optimizer. It rewrites your logical plan through a series of rules:

```
Unresolved logical plan
   ↓ (analysis: resolve column names, tables)
Resolved logical plan
   ↓ (rule-based optimization: predicate pushdown, constant folding, ...)
Optimized logical plan
   ↓ (physical planning: pick join strategy, partitioning)
Physical plan
   ↓ (codegen: generate JVM bytecode for hot paths)
Execution
```

What this means for users: write declarative DataFrame / SQL code, get optimized execution for free. Catalyst handles predicate pushdown into Parquet, projection pruning to only read needed columns, constant folding, and many other rewrites.

For analytics over Parquet on S3, Spark's read path is highly tuned: column pruning, predicate pushdown, page skipping using min/max stats — often reading single-digit percent of the file footprint.

---

## 8. Structured Streaming

Spark Structured Streaming runs the same DataFrame API on unbounded data. Mental model: **a streaming query is a query on a table that keeps growing**.

```python
events = (spark.readStream
          .format("kafka")
          .option("subscribe", "transactions")
          .load())

agg = (events
       .groupBy(F.window("event_time", "1 minute"), "merchant_id")
       .agg(F.sum("amount")))

(agg.writeStream
    .outputMode("update")
    .format("parquet")
    .option("path", "s3://bucket/agg/")
    .option("checkpointLocation", "s3://bucket/checkpoints/")
    .trigger(processingTime="30 seconds")
    .start())
```

Properties:
- **Micro-batches by default** — Spark accumulates events and runs them in small batches (defaults around 100ms–seconds). Lower latency than batch jobs, higher than true streaming.
- **Continuous mode** (experimental) — true per-event processing with millisecond latency. Less mature; production usage is rare.
- **Exactly-once** with checkpointing — Spark commits batch outputs atomically when the sink supports it (Delta, Iceberg, Kafka with idempotent producer, files via _SUCCESS markers).
- **Event-time windowing, watermarks, stateful operators** — same shape as Flink, slightly different ergonomics.

Spark Streaming is the right pick when:
- Your latency budget is "seconds, not milliseconds."
- You want the same engine for batch and stream.
- Your sink is a lakehouse format (Delta, Iceberg).
- Your team already runs Spark.

For pure low-latency streaming, prefer **Flink** — its event-time semantics, exactly-once guarantees on broader sinks, and per-event processing model are stronger. See [Apache Flink →](./flink.md), [Stream Processing →](../07-messaging/stream-processing.md).

---

## 9. MLlib and beyond

Spark MLlib provides distributed implementations of common algorithms:
- Linear / logistic regression, decision trees, random forests, GBT, k-means, ALS recommenders, etc.
- A Pipeline API similar to scikit-learn.

Reality check: **MLlib is fine for distributed feature engineering at scale, but no one uses it for cutting-edge ML training**. Deep learning training moved to PyTorch / TensorFlow / JAX with GPU clusters. Spark's role in modern ML is:

- Feature engineering on TB-scale data.
- ETL into ML feature stores.
- Batch inference for large datasets.
- The bridge from "data in the warehouse" to "tensors on a GPU."

For feature stores: Tecton, Feast (open source), Databricks Feature Store — most are built on Spark for batch features plus a streaming layer for fresh ones.

---

## 10. Deployment and cluster operations

A working production Spark setup needs:

### Resource sizing

- **Executor memory** — typically 4–32 GB. Larger than ~64 GB hits GC pause issues; prefer more executors than fatter ones.
- **Executor cores** — 4–8 cores per executor is a common sweet spot. Too many → scheduler contention; too few → JVM overhead dominates.
- **Driver memory** — small for ETL (4–8 GB), larger if `.collect()`-ing big results.
- **`spark.sql.shuffle.partitions`** — start near `(total data / 128 MB)` and adjust with AQE.

### Spot / preemptible instances

Spark's lineage-based recovery handles executor loss well. Mixing 50–80% spot + 20–50% on-demand for the driver and a few executors is a common cost-saving pattern. Save ~50–70% on infra cost.

### Storage formats

- **Parquet** for analytical data.
- **Delta Lake** or **Apache Iceberg** for ACID tables, time travel, schema evolution, and concurrent writes. See [Lakehouse →](../04-databases/lakehouse.md).
- **Avro** for streaming.
- **Compressed with zstd or Snappy**.

### Monitoring

- Spark UI for per-job DAGs, task timings, shuffle metrics.
- Spark history server for completed jobs.
- Metrics via JMX / Prometheus / Datadog.
- Cluster-side: Kubernetes / cloud metrics.

The biggest operational pains are: **stragglers from skewed keys**, **shuffle spills filling local disk**, **OOMs in executors from misjudged join strategies**, and **slow cloud storage on small files**.

---

## 11. Spark in 2026 — where it fits

The honest competitive landscape:

| Workload | First choice | Spark's fit |
|---|---|---|
| Batch ETL over data lake | **Spark** | Default. Plays with Iceberg/Delta, runs on K8s or managed. |
| Lakehouse interactive SQL | **Trino / Snowflake / Databricks SQL** | Spark is OK; specialists are faster for interactive. |
| Sub-second streaming | **Flink / Kafka Streams** | Spark Structured Streaming OK for seconds-level. |
| Distributed ML training | **PyTorch / TensorFlow + Ray / Horovod** | Spark for feature engineering, not training. |
| Massive batch inference | **Spark** | Great fit. |
| Ad-hoc analytics on warehouse | **dbt + Snowflake/BigQuery** | Spark wins when data is on the lake, not the warehouse. |
| Real-time analytics (sub-second) | **ClickHouse / Druid / Pinot** | Out of Spark's league for query latency. |

The trend: **Spark is the batch and feature-engineering layer**; specialists handle interactive SQL, streaming, and ML training. That's fine — Spark is genuinely excellent at what it does.

---

## 12. Worked example — a Spark ETL pipeline

A daily pipeline turning raw Kafka events into a curated table:

```python
from pyspark.sql import SparkSession, functions as F

spark = (SparkSession.builder
         .appName("daily-orders")
         .config("spark.sql.adaptive.enabled", "true")
         .config("spark.sql.adaptive.skewJoin.enabled", "true")
         .getOrCreate())

# 1. Read raw events (Parquet, partitioned by date)
events = (spark.read
          .parquet("s3://raw/events/dt=2026-05-19/")
          .filter("event_type = 'order_placed'"))

# 2. Enrich with merchants (broadcast join — small dim table)
merchants = spark.read.parquet("s3://dim/merchants/")
enriched = events.join(F.broadcast(merchants), "merchant_id", "left")

# 3. Aggregate
daily = (enriched
         .groupBy(F.to_date("event_time").alias("dt"),
                  "merchant_id", "merchant_name", "country")
         .agg(F.sum("amount_cents").alias("revenue_cents"),
              F.count("*").alias("order_count"))
         .where("revenue_cents > 0"))

# 4. Write as an Iceberg table — atomic, schema-tracked
(daily.write.format("iceberg")
      .mode("overwrite")
      .option("overwrite-partition", "dt=2026-05-19")
      .save("warehouse.orders_daily"))
```

What's good about this code:
- AQE on (skew safety).
- Broadcast join hint (deterministic — no accident).
- Reading Parquet with partition pruning (`dt=2026-05-19/`).
- Writing to Iceberg (atomic, evolvable, time-travelable).
- Idempotent (re-running overwrites the same partition).

Wire this up to **Airflow / Dagster** (see [Data Pipelines →](./data-pipelines.md)) and you have a production-grade daily pipeline in 30 lines.

---

## 13. Common Mistakes / Anti-Patterns

- **`.collect()` on a big DataFrame.** Driver OOM. Use `.write` or `.foreach` for distributed sinks.
- **Lots of small files in the output.** Use `df.coalesce()` or repartition by target file size before writing.
- **One huge shuffle partition (skew).** Enable AQE skew handling, salt keys, or pre-aggregate.
- **`autoBroadcastJoinThreshold=-1`** (broadcast disabled). Almost always wrong — small dim tables should broadcast.
- **Default `spark.sql.shuffle.partitions=200` on tiny data.** Wastes time on task overhead. AQE coalesces this when enabled.
- **Default 200 on huge data.** Tasks too big, spills everywhere. Set higher.
- **Reading CSV/JSON in production.** 10× slower than Parquet. Pay the conversion cost once.
- **No predicate pushdown** because of UDFs after `filter`. Move filters earlier so Catalyst can push them.
- **Using RDDs for new code.** DataFrames are faster and clearer.
- **Operations inside Python UDFs.** PySpark UDFs incur serialization overhead. Use built-in functions, Spark SQL expressions, or pandas UDFs (vectorized).
- **Cache everything.** Caching helps when reused; otherwise it's memory pressure. Cache deliberately.
- **`repartition()` everywhere "to fix" performance.** Each repartition is a shuffle. Often wrong tool.
- **Spark Structured Streaming with no checkpoint directory or with a bad one.** State lost on restart; exactly-once guarantees gone.
- **MLlib for cutting-edge ML.** Use it for feature engineering at scale; reach for PyTorch/TensorFlow for training.
- **One huge cluster shared by all teams.** Noisy-neighbor problems. Use Kubernetes namespaces or managed multi-tenant services.
- **Letting Spark write tiny partition files because of `spark.sql.files.maxRecordsPerFile` defaults.** Compact regularly.
- **Stale Spark version on managed services.** New versions have huge perf wins (AQE, photonization, Catalyst rules). Upgrade often.

---

## 14. Cheat Card

```
PURPOSE   General-purpose distributed compute: batch ETL, SQL,
          streaming, ML feature engineering. Lakehouse-friendly.

CORE
  Driver        plans, optimizes, schedules
  Executors     run tasks in parallel
  Stages        chunks of work between shuffles
  Tasks         one per partition
  Catalyst      query optimizer (DataFrame / SQL)
  Tungsten      codegen + columnar memory layout
  AQE           adaptive runtime re-planning

APIs (PICK DATAFRAMES)
  RDD           low-level, opaque to Catalyst — avoid for new code
  DataFrame     default; Catalyst-optimized
  Dataset       typed DF (Scala/Java)
  Spark SQL     strings, same engine as DataFrame

JOIN STRATEGIES
  Broadcast hash    one side fits; fastest
  Shuffle hash      medium
  Sort-merge        both large
  AQE can convert sort-merge → broadcast at runtime

TUNING LEVERS
  spark.sql.adaptive.enabled = true                 (always)
  spark.sql.adaptive.skewJoin.enabled = true        (skew safety)
  spark.sql.shuffle.partitions ≈ data / 128 MB
  broadcast() hint on small dims
  Partition pruning + Parquet predicate pushdown
  Bucketing for repeated joins on same key

STREAMING
  Structured Streaming, micro-batch (default seconds)
  Continuous mode = experimental
  Checkpointing required for exactly-once

DEPLOYMENT
  Spark on Kubernetes (modern self-managed)
  Databricks / EMR / Dataproc (managed)
  Spot instances + on-demand mix

WHEN SPARK WINS
  Batch ETL over data lake
  Feature engineering for ML
  Large-scale lakehouse compute
  Multi-stage iterative work

WHEN IT LOSES
  Sub-second streaming → Flink / Kafka Streams
  Interactive sub-second SQL → Trino / Snowflake / ClickHouse
  ML training → PyTorch + Ray / Horovod

PITFALLS
  .collect() on big data
  Tiny output files
  Skewed keys without salting / AQE
  RDDs for new code
  PySpark UDFs in the hot path
  Default shuffle partitions on extreme data sizes
  No checkpointing on streaming jobs
  Stale Spark version

RULE   Spark + Parquet + Iceberg/Delta on S3 + Airflow is the
       default lakehouse shape. Boring, capable, well-tooled.
```

---

## 15. Resources

### Books
- *Spark: The Definitive Guide* — Chambers & Zaharia. The reference.
- *Learning Spark* (2nd ed.) — Damji et al.
- *High Performance Spark* — Karau & Warren. Tuning-focused.
- *Designing Data-Intensive Applications* — Martin Kleppmann.

### Documentation
- **Apache Spark** — <https://spark.apache.org/docs/latest/>
- **PySpark** — <https://spark.apache.org/docs/latest/api/python/>
- **Structured Streaming** — <https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html>
- **Delta Lake** — <https://delta.io>
- **Apache Iceberg** — <https://iceberg.apache.org>

### Papers
- "Resilient Distributed Datasets" — Zaharia et al. (2012). The original Spark paper.
- "Spark SQL: Relational Data Processing in Spark" — Armbrust et al. (2015).
- "Structured Streaming: A Declarative API for Real-Time Applications in Apache Spark" — Armbrust et al. (2018).

### Articles
- "Adaptive Query Execution: speeding up Spark SQL at runtime" — Databricks blog.
- "Why Spark beat MapReduce" — multiple retrospectives.
- "Spark on Kubernetes best practices" — Databricks / Spark community.
- "Stripe's Spark journey" — Stripe engineering blog (excellent).

### Videos
- *Spark Summit / Data + AI Summit* — annual; deep technical talks.
- *Apache Spark internals* — Holden Karau talks.
- ByteByteGo — "Apache Spark Explained."

### Tools
- **Spark UI**, **Spark history server** — task-level diagnostics.
- **`spark-submit`**, **`pyspark`**, **`spark-shell`**.
- **Delta Lake / Apache Iceberg / Apache Hudi** — ACID lakehouse tables.
- **dbt-spark**, **Airflow's `SparkSubmitOperator`**, **Dagster's `dagster-spark`**.
- **Spark on Kubernetes operator** (Google's, IBM's, or vanilla).
- **Databricks**, **AWS EMR / Glue**, **GCP Dataproc / Serverless**, **Azure Synapse**.

### Adjacent reading
- [MapReduce →](./mapreduce.md)
- [Hadoop Ecosystem →](./hadoop.md)
- [Apache Flink →](./flink.md)
- [ETL vs ELT →](./etl-vs-elt.md)
- [Data Pipelines & Orchestration →](./data-pipelines.md)
- [Data Modeling for Analytics →](./dimensional-modeling.md)
- [Stream Processing →](../07-messaging/stream-processing.md)
- [Batch vs Stream Processing →](../07-messaging/batch-vs-stream.md)
- [Lakehouse Architecture →](../04-databases/lakehouse.md)
- [Data Warehouses & Data Lakes →](../04-databases/warehouses-lakes.md)
- [Object Storage →](../09-storage/object-storage.md)
- [Tail Latency & p99 →](../16-performance/tail-latency.md)

---

*Previous:* [← Hadoop Ecosystem](./hadoop.md)  |  *Next:* [Apache Flink →](./flink.md)
