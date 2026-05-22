# Hadoop Ecosystem

> **TL;DR** — **Hadoop** is the open-source project that turned Google's MapReduce + GFS papers into a stack any company could run on commodity hardware. The core pieces are **HDFS** (distributed file system), **YARN** (cluster resource manager), and **MapReduce** (the original compute engine). Around that core grew an entire ecosystem — **Hive**, **Pig**, **HBase**, **ZooKeeper**, **Sqoop**, **Oozie**, **Tez**, **Impala**, **Spark**, **Flink** — collectively known as "the Hadoop ecosystem." From 2008–2016 it was the consensus answer to "big data." Today, **most of that core has lost the war to cloud-native equivalents**: HDFS to S3/GCS, YARN to Kubernetes, MapReduce to Spark/Flink/Trino, Hive to Snowflake/BigQuery + Iceberg/Delta. The remaining bits — **Hive Metastore**, **Avro**, **Parquet**, **ORC**, and the *patterns* of distributed file-based analytics — became foundational primitives of the modern data platform. The honest take: **don't start new Hadoop clusters in 2026**. Use object storage, a lakehouse table format, and a modern compute engine. But understand Hadoop because *every* modern data platform inherits its shape.

---

## 1. The big picture

Hadoop's original architecture was a three-layer stack:

```
┌────────────────────────────────────────────────────┐
│ Compute frameworks                                 │
│  MapReduce · Tez · Spark · Hive · Pig · Impala     │
├────────────────────────────────────────────────────┤
│ Cluster resource manager                           │
│  YARN (CPU, memory, container scheduling)          │
├────────────────────────────────────────────────────┤
│ Storage                                            │
│  HDFS (replicated block storage across nodes)      │
└────────────────────────────────────────────────────┘
```

The pitch in 2010 was simple and revolutionary: **store everything as files on cheap commodity hardware, replicated 3×, on a system that scales horizontally**. Throw any compute framework at the same data. No specialty storage appliance, no proprietary hardware, no per-CPU licensing.

That pitch worked. From ~2008 to ~2016, every Fortune 500 had a Hadoop project. The vendors — Cloudera, Hortonworks (merged in 2019), MapR (sold to HPE in 2019) — were unicorns.

What's changed: **object storage is now better than HDFS**, **Kubernetes is now better than YARN**, **Spark/Flink/Trino are better than MapReduce**, and **lakehouse table formats** (Iceberg, Delta Lake, Hudi — see [Lakehouse →](../04-databases/lakehouse.md)) solved the things HDFS couldn't. The ecosystem disaggregated.

This page documents what was, what survives, and how to read modern data infrastructure through that lineage.

---

## 2. HDFS — the file system that started it

**HDFS** (Hadoop Distributed File System) was the open-source clone of Google's GFS. Properties:

- **Write-once, read-many** — files are immutable. Append-only at best.
- **Big blocks** — default 128 MB (was 64 MB originally). Designed for streaming reads, not random access.
- **3× replication** — every block lives on three nodes, ideally one per rack. Cheap durability.
- **NameNode + DataNodes** — one central metadata server (NameNode) plus many storage workers (DataNodes).

### HDFS architecture

```
                   ┌──────────────────┐
                   │   NameNode       │  in-memory directory tree,
                   │   (master)       │  block-to-node map
                   └────────┬─────────┘
                            │
        ┌───────────────────┼──────────────────────┐
        ▼                   ▼                      ▼
   ┌─────────┐         ┌─────────┐            ┌─────────┐
   │DataNode │         │DataNode │            │DataNode │
   │ blocks: │         │ blocks: │            │ blocks: │
   │ b3, b7  │         │ b1, b3  │            │ b2, b7  │
   └─────────┘         └─────────┘            └─────────┘
```

A client asks the NameNode "where's `/orders/2026-05.parquet`?" The NameNode replies with block locations. The client streams from the DataNodes directly. Network bandwidth scales horizontally.

### What HDFS got right

- **Linear scalability to petabytes** on cheap disks.
- **Data locality** — co-locate compute with storage. See [MapReduce →](./mapreduce.md).
- **Fault tolerance** built into replication.
- **Open format** — anything that can read a file can read HDFS.

### What HDFS got wrong (in hindsight)

- **NameNode as a single point of failure and capacity wall.** All metadata in RAM. ~150 bytes per file/block. 500 million files needed 75 GB RAM. HA NameNode helped; the federation model never quite worked.
- **Tightly coupled compute and storage.** You bought disks to grow storage; you got compute "for free." In the cloud, you want them independent — one is cheap, the other is expensive.
- **Small files problem.** Each file takes a NameNode entry; tens of millions of small files crash performance. Whole pipelines became "compact small files into big files" jobs.
- **No multi-tenancy beyond crude quotas.** Sharing HDFS clusters across teams was painful.

### The cloud killed HDFS

When S3, GCS, Azure Blob came in, they did all the things HDFS promised, plus:

- **Independent scaling** of storage and compute.
- **11 nines of durability** without you running anything.
- **No NameNode** — the cloud handles metadata at planet scale.
- **Pay-per-byte**, not per-disk.
- **Native multi-tenancy and IAM.**

Modern data platforms run on object storage. HDFS lives on in legacy estates, and as the inspiration for storage engines built on top of object stores (Alluxio, JuiceFS, MinIO).

---

## 3. YARN — the resource manager

**YARN** (Yet Another Resource Negotiator) was Hadoop 2.0's split of the original MapReduce master into a generic resource manager. It opened Hadoop to other frameworks (Spark, Tez, Flink).

```
┌──────────────────────────┐
│  ResourceManager (RM)    │
│  - scheduler             │
│  - apps manager          │
└────────────┬─────────────┘
             │
   ┌─────────┼─────────┐
   ▼         ▼         ▼
┌───────┐ ┌───────┐ ┌───────┐
│ Node  │ │ Node  │ │ Node  │
│Manager│ │Manager│ │Manager│
│       │ │       │ │       │
│contain│ │contain│ │contain│
│ -ers  │ │ -ers  │ │ -ers  │
└───────┘ └───────┘ └───────┘
```

A job submits to the RM, which negotiates a fixed slice of CPU+memory (a *container*) on some NodeManager. Inside that container runs an *ApplicationMaster*, which then negotiates more containers for the actual work.

YARN was useful for years, but in retrospect it competed with the same problem Kubernetes solved better. Today, **Spark on Kubernetes** and **Flink on Kubernetes** are the dominant deployment models for big-data compute. YARN survives in legacy Hadoop estates and in some EMR / Dataproc configurations.

---

## 4. MapReduce — the compute engine

The original Hadoop compute framework. Covered in detail in [MapReduce →](./mapreduce.md). For the purposes of this page, the relevant facts:

- Disk-bound shuffle made it slow for anything multi-stage or iterative.
- Tedious to program directly; most users went through Hive or Pig.
- Replaced by Tez (a DAG engine, faster MR), then by Spark (in-memory DAGs).

Nobody writes new MapReduce in 2026.

---

## 5. Hive — SQL on Hadoop

**Apache Hive** was the system that made big data usable. Facebook built it in 2008 to let analysts write SQL ("HiveQL") instead of Java MapReduce.

Hive's three components:

- **HiveQL** — SQL-like language. Compiles to MapReduce / Tez / Spark execution plans.
- **Metastore** — a relational DB (typically MySQL/Postgres) that stores table definitions, schemas, partitions, locations.
- **Execution engine** — pluggable. Originally MR, later Tez, later Spark.

The defining contribution: **the Hive Metastore became the lingua franca of the data lake**. Every modern engine — Spark, Trino, Presto, Athena, Dremio, Snowflake's external tables — speaks to a Hive Metastore (or its successor, AWS Glue Catalog / Iceberg REST Catalog).

Hive itself as an execution engine is now legacy. Hive *tables* and the *Metastore* live on, and have shaped lakehouse table formats. The new world is **Iceberg / Delta Lake / Hudi tables registered in a metastore, queried by Trino / Spark / Snowflake** — Hive's idea, modernized.

---

## 6. Pig — the dataflow language

**Apache Pig** ("Pig Latin") was a procedural dataflow language: `LOAD → FILTER → JOIN → GROUP → STORE`. Compiled to MR. Yahoo built it; it competed with Hive throughout the early 2010s.

The winner was SQL. Pig is largely retired. Mentioned here because old Hadoop estates still have Pig jobs lying around, and because the dataflow model (`LOAD ... STORE`) survives in Beam, dbt, and Spark's DataFrame API.

---

## 7. HBase — wide-column store on HDFS

**Apache HBase** is the open-source clone of Google's Bigtable. Key/value + wide-column store sitting on top of HDFS. Strong consistency per row, automatic sharding, billion-row scale.

Used by Facebook (HBase was a Facebook product for years), Pinterest, Adobe, telecoms. Pinned to HDFS for storage.

In the cloud, the equivalent role is filled by **DynamoDB**, **Bigtable**, **Cassandra/ScyllaDB**, or **CockroachDB**. HBase is rarely the new-build choice; existing HBase estates are migrating off.

See [Wide-Column Stores →](../04-databases/wide-column-stores.md).

---

## 8. The supporting cast

A taxonomy of ecosystem tools you'll encounter — some still relevant, many archaeological:

| Tool | What it did | 2026 status |
|---|---|---|
| **ZooKeeper** | Coordination, leader election, configuration | Still used (Kafka, HBase). Many systems moving to etcd / Raft built-in. |
| **Sqoop** | Bulk import/export between RDBMS and HDFS | Retired; replaced by CDC tools (Debezium), Airbyte, Fivetran. |
| **Flume** | Log ingestion into HDFS | Retired; Kafka + Kafka Connect, Fluent Bit, Vector. |
| **Oozie** | Workflow scheduler for MR / Pig / Hive jobs | Retired; Airflow, Dagster, Prefect — see [Data Pipelines →](./data-pipelines.md). |
| **Ambari** / **Cloudera Manager** | Cluster GUI | Vendor-specific; obsolete for cloud users. |
| **Mahout** | Scalable ML on MR | Retired; Spark MLlib, then PyTorch/TensorFlow. |
| **Tez** | Faster DAG-based successor to MR | Used by Hive; mostly displaced by Spark. |
| **Impala** | MPP SQL engine on HDFS / S3 (Cloudera) | Niche; competes with Trino / Presto. |
| **Apache Kudu** | Columnar storage with row-level updates | Niche; eclipsed by lakehouse table formats. |
| **Apache Drill** | Schema-on-read SQL | Niche; mostly Trino/Presto territory. |
| **Apache Storm** | Streaming, pre-Flink | Retired; Flink, Spark Structured Streaming. |
| **Apache Samza** | LinkedIn's streaming | Niche; Flink/Kafka Streams. |

The pattern: tools tightly coupled to HDFS or YARN are mostly gone. Tools that lived above the storage layer — Spark, Flink, Trino, dbt, Iceberg — survive and thrive.

---

## 9. The file formats that survived

If anything from Hadoop deserves a victory parade, it's the **columnar file formats**:

### Parquet

Apache Parquet (Twitter + Cloudera, 2013) is the de facto columnar format for analytics. Columnar layout + per-column compression + statistics + predicate pushdown. **Spark, Trino, Snowflake, BigQuery (federated), DuckDB, Polars** — all read Parquet natively.

A query like `SELECT amount FROM orders WHERE country = 'US' AND year = 2026`:
- Reads only the `amount` and `country` columns (column pruning).
- Skips row groups whose stats show `country != 'US'` (predicate pushdown).
- Reads compressed column chunks (zstd / Snappy / GZIP).

10–100× faster than reading row-oriented CSV / JSON. **For any analytics workload over object storage, Parquet is the default.**

### ORC

Apache ORC (Hortonworks, 2013) is Parquet's sibling. Same idea, slightly different layout. ORC tends to win for Hive workloads, Parquet for everyone else. They're interchangeable for most modern engines.

### Avro

Apache Avro (Hadoop project, 2009) is *row*-oriented binary with embedded schema. Bad for analytics; great for streaming and Kafka. Schema evolution rules are particularly good. See [Serialization Formats →](../16-performance/serialization.md).

### SequenceFile / RCFile / Text

Earlier Hadoop formats. Historical interest only.

The fact that Parquet, ORC, and Avro became cross-platform standards is one of Hadoop's biggest legacies. They outlived the platform that birthed them.

---

## 10. The Hadoop business — what happened

A short economic history, because it explains why things are the way they are:

- **2006**: Doug Cutting starts Hadoop at Yahoo, based on Google's papers.
- **2008–2010**: Hadoop adopted by Yahoo, Facebook, Twitter at scale.
- **2009**: Cloudera founded. Hortonworks (2011), MapR (2009) follow.
- **2012–2016**: "Big data" hype peak. Every enterprise has a Hadoop project. Conference circuit booms.
- **2014**: Spark goes mainstream. The first sign that the MR/HDFS coupling is loosening.
- **2017**: S3/EMR pattern displaces on-prem Hadoop at most cloud-first companies.
- **2019**: Cloudera and Hortonworks merge. MapR sold to HPE. The party's over.
- **2020+**: Lakehouse formats (Iceberg, Delta, Hudi) decouple "data warehouse semantics" from any specific storage or engine. The Hadoop value prop is now table-stakes everywhere.

The lesson generalizes: **infrastructure innovations get absorbed into clouds and become commodity**. The interesting work moves up the stack — to query engines, table formats, and tooling. Don't fight the gravity.

---

## 11. Modern equivalents — Hadoop → cloud

| Hadoop component | Modern equivalent |
|---|---|
| HDFS | S3, GCS, Azure Blob, OCI Object Storage |
| YARN | Kubernetes (Spark on K8s, Flink on K8s), Databricks/EMR/Dataproc managed compute |
| MapReduce | Spark, Flink |
| Hive | Iceberg/Delta tables + Trino/Spark/Snowflake/BigQuery |
| HBase | DynamoDB, Bigtable, Cassandra, CockroachDB |
| Pig | dbt, Spark DataFrames |
| Sqoop | Debezium, Airbyte, Fivetran |
| Flume | Kafka + Kafka Connect, Vector, Fluent Bit |
| Oozie | Airflow, Dagster, Prefect, Argo Workflows |
| Mahout | Spark MLlib, then PyTorch / TensorFlow / scikit-learn |
| Tez | Spark, Trino |
| Hive Metastore | Glue, Hive Metastore (still!), Iceberg REST Catalog, Unity Catalog |
| Parquet / ORC / Avro | **Same** — they won |

The mental model that survives: **storage layer (immutable files, columnar formats, table metadata) decoupled from compute layer (Spark, Trino, Snowflake, BigQuery) via a catalog**. That's modern data architecture. Hadoop pioneered every piece, even if the named software is gone.

---

## 12. When to actually use Hadoop today

A short, honest list:

- **You already run a healthy Hadoop estate** and migration cost > value. Maintain it, pay down debt incrementally.
- **Government / regulatory / air-gapped** environments where cloud isn't an option and you need a known-stable on-prem stack.
- **Custom data sovereignty** requirements with massive throughput that on-prem solves cheaply.
- **Some EMR / Dataproc workloads** that still call themselves "Hadoop" but are really Spark + S3 + Hive Metastore in a managed wrapper.

For new builds elsewhere: start with **object storage + Iceberg or Delta + Spark/Trino/Snowflake + Airflow/Dagster**. Skip the Hadoop word; you'll get most of its goodness anyway.

---

## 13. Common Mistakes / Anti-Patterns

- **Starting a new Hadoop cluster in 2026.** Use a managed cloud equivalent or a lakehouse on object storage.
- **Tightly coupling storage and compute.** "We need 100 more disks" should never mean "we need 100 more compute nodes."
- **One big shared HDFS cluster across the whole company.** Multi-tenancy is harder than it looks; small isolation problems become huge.
- **Storing tens of millions of tiny files.** NameNode dies, jobs crawl. Compact regularly into larger Parquet files.
- **CSV/JSON as the storage format.** 10–100× slower than Parquet at query time, no schema evolution.
- **Hive Metastore as a Postgres single point of failure** with no HA or backups.
- **Pig for new work.** Use SQL or DataFrames.
- **MR for new work.** Spark or Flink for compute; let your SQL engine compile it.
- **Sqoop for new ingestion.** CDC (Debezium) or managed ingestion (Airbyte/Fivetran) are far better.
- **Oozie for new orchestration.** Airflow / Dagster / Prefect are the modern picks.
- **Ignoring the small-files problem until it's a crisis.** Build compaction into pipelines from day one.
- **Treating HDFS replicas as backups.** Replication isn't backup; backup is for human error and corruption.
- **Running Mahout for new ML work.** It's archaeology; use PyTorch / TensorFlow / scikit-learn / Spark MLlib.
- **Optimizing Hive on MR.** Move to Hive on Spark/Tez, or just to Trino/Spark + Iceberg.

---

## 14. Cheat Card

```
PURPOSE   Open-source platform for batch analytics on commodity
          hardware: HDFS + YARN + MapReduce, plus a sprawling
          ecosystem of tools that grew up around it.

CORE TRIPLET
  HDFS         distributed file system, 3× replicated blocks
  YARN         cluster resource manager (CPU, memory, containers)
  MapReduce    original batch compute engine

WHAT STILL MATTERS
  Parquet, ORC, Avro      columnar / row file formats
  Hive Metastore           metadata catalog (modernized as Iceberg REST, Glue, Unity)
  The mental model of compute over immutable files
  The data-locality and DAG-execution patterns
  HBase (legacy where it's installed)

WHAT'S RETIRED
  HDFS          → S3 / GCS / Azure Blob
  YARN          → Kubernetes (Spark on K8s, Flink on K8s)
  MapReduce     → Spark, Flink
  Hive engine   → Iceberg/Delta + Trino/Spark + a metastore
  Pig, Oozie, Sqoop, Flume, Mahout, Storm

WHY IT LOST
  Coupled storage and compute
  NameNode metadata wall
  Operational complexity vs cloud-managed alternatives
  Object storage caught up and surpassed HDFS

WHEN HADOOP TODAY
  Healthy legacy estate, migrate when ready
  Air-gapped / regulatory on-prem builds
  EMR / Dataproc managed clusters (really Spark + S3)

PITFALLS
  Starting a new Hadoop cluster in 2026
  Treating HDFS replicas as backups
  Millions of tiny files
  CSV/JSON instead of Parquet
  Hive Metastore Postgres without HA
  New pipelines on Pig / Oozie / Sqoop / Flume

RULE   Don't say "Hadoop" — say "object storage + lakehouse +
       Spark/Trino/Snowflake + Airflow." You'll be more accurate
       and you'll skip 10 years of legacy debt.
```

---

## 15. Resources

### Books
- *Hadoop: The Definitive Guide* (4th ed.) — Tom White. The classic, mostly historical now.
- *Hadoop Operations* — Eric Sammer. Operational realities.
- *Designing Data-Intensive Applications* — Martin Kleppmann. Modern lens on the same problem space.

### Papers
- "MapReduce" — Dean & Ghemawat (2004).
- "Google File System" — Ghemawat et al. (2003).
- "Bigtable" — Chang et al. (2006).

### Documentation
- **Apache Hadoop** — <https://hadoop.apache.org>
- **Apache Hive** — <https://hive.apache.org>
- **Apache HBase** — <https://hbase.apache.org>
- **Apache Parquet** — <https://parquet.apache.org>
- **Apache ORC** — <https://orc.apache.org>
- **Apache Avro** — <https://avro.apache.org>

### Articles
- "Hadoop is dead. Long live Hadoop." — many takes; pick a recent one.
- "The end of Hadoop?" — Bessemer / VC retrospective pieces.
- "Why we left Hadoop" — Discord, Spotify, Pinterest engineering blogs.
- "From Hadoop to Lakehouse" — Databricks engineering posts.

### Videos
- *Hadoop Distributed File System* — original Yahoo talks (historical).
- *Hadoop Summit* / DataWorks recordings.
- *Migrating off Hadoop* — recent Spark Summit / Data+AI talks.
- ByteByteGo — "Hadoop Ecosystem Explained."

### Adjacent reading
- [MapReduce →](./mapreduce.md)
- [Apache Spark →](./spark.md)
- [Apache Flink →](./flink.md)
- [ETL vs ELT →](./etl-vs-elt.md)
- [Data Pipelines & Orchestration →](./data-pipelines.md)
- [Data Modeling for Analytics →](./dimensional-modeling.md)
- [Distributed File Systems (HDFS, GFS) →](../09-storage/distributed-file-systems.md)
- [Object Storage (S3, GCS, Azure Blob) →](../09-storage/object-storage.md)
- [Data Warehouses & Data Lakes →](../04-databases/warehouses-lakes.md)
- [Lakehouse Architecture →](../04-databases/lakehouse.md)
- [Wide-Column Stores (Cassandra, HBase, ScyllaDB) →](../04-databases/wide-column-stores.md)

---

*Previous:* [← MapReduce](./mapreduce.md)  |  *Next:* [Apache Spark →](./spark.md)
