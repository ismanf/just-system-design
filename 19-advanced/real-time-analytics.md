# Real-Time Analytics (Apache Druid, Pinot, ClickHouse)

> **TL;DR** — **Real-time analytics** means **sub-second OLAP queries on freshly ingested data** — typically aggregations over millions to billions of rows updated within seconds. Traditional warehouses (Snowflake, BigQuery, Redshift) are too slow (seconds to minutes) for user-facing dashboards and interactive product analytics. **Apache Druid**, **Apache Pinot**, and **ClickHouse** are the three engines that filled that gap, each with a slightly different philosophy: **Druid** for time-series and event analytics with rich segmentation; **Pinot** for upsert-friendly real-time aggregations driven by LinkedIn/Uber; **ClickHouse** for general-purpose blazing-fast OLAP with a simpler operational model. The honest take: **all three solve the same problem** — sub-second queries over high-cardinality, frequently-updated event data — and the right pick is mostly determined by **team familiarity and operational preference**. Modern hosted variants (**Tinybird**, **Apache Druid managed**, **StarTree for Pinot**, **ClickHouse Cloud**) reduce the operational cost dramatically.

---

## 1. The big picture

```
Events                    Real-time analytics engine
   │                          ┌────────────────────────┐
   ▼                          │  Indexing tier          │
┌─────────┐   stream    ──►   │  - real-time ingest     │
│ Kafka / │                   │  - in-memory store      │
│ Kinesis │                   │  - flush to segments    │
└─────────┘                   ├────────────────────────┤
   │                          │  Historical tier        │
   ▼                          │  - columnar segments    │
┌─────────┐    batch   ──►    │  - on disk / object     │
│ S3 / DW │                   │    storage              │
└─────────┘                   ├────────────────────────┤
                              │  Query tier             │
                              │  - scatter / gather     │
                              │  - aggregation push-down│
                              └───────────┬────────────┘
                                          │
                                          ▼
                                   Dashboard / API
                                   (sub-second response)
```

The shape:
- **Ingest streaming and batch** sources.
- **Index columnar** for fast aggregations.
- **Partition by time** so old data lives cheaply on object storage, recent data in RAM/SSD.
- **Push aggregations down** to data nodes; coordinator merges.
- **Return p99 sub-second** for the dashboard.

If a classical warehouse is a freight train (high throughput, slow start), real-time analytics engines are bullet trains: small, lots of them, sub-second arrivals.

---

## 2. Why warehouses aren't enough

Snowflake, BigQuery, Redshift, Databricks SQL are excellent for **data engineering and BI**: ETL, dbt models, exploratory analyses, ad-hoc reporting. But they have constraints:

- **Cold-start latency** measured in seconds (BigQuery's slot start, Snowflake warehouse spin-up).
- **Query latency** typically 1–30 s for medium queries.
- **Concurrency limits** — Snowflake virtual warehouses serialize per warehouse; queue depth becomes painful past a few concurrent users.
- **Ingestion latency** — minutes to hours for "fresh" data.
- **Cost shape** — query-billed (BigQuery) or warehouse-billed (Snowflake). User-facing dashboards firing 1000 QPS quickly become a five-figure monthly bill.

For *user-facing* analytics — product dashboards, real-time leaderboards, ad-server impressions, fraud signals — a different engine is needed. That's what Druid / Pinot / ClickHouse do.

| | Warehouse (Snowflake/BQ) | Real-time analytics (Druid/Pinot/CH) |
|---|---|---|
| Typical query latency | 1–30 s | 50–500 ms |
| Concurrency | 10s of QPS | 1000s of QPS |
| Ingest freshness | Minutes-hours | Seconds |
| Best for | Ad-hoc, ETL, BI | User-facing dashboards, telemetry |
| Cost shape | Per-query / warehouse hour | Cluster-hosted |
| Storage | Object storage | Cluster disk + object storage |

---

## 3. Apache Druid

Druid (2011, Metamarkets → Imply) is one of the original time-series analytics engines.

### Architecture

```
┌──────────────────────────────────────────────┐
│ Master tier                                   │
│  Coordinator (segment placement)              │
│  Overlord (ingestion tasks)                   │
├──────────────────────────────────────────────┤
│ Data tier                                     │
│  MiddleManager (real-time ingest)             │
│  Historical (immutable segments)              │
├──────────────────────────────────────────────┤
│ Query tier                                    │
│  Broker (query routing + scatter/gather)      │
│  Router (UX-facing)                           │
└──────────────────────────────────────────────┘
       │                          │
       ▼                          ▼
   Metadata DB              Deep storage
   (Postgres / MySQL)       (S3 / HDFS / GCS)
```

Druid's distinctive features:
- **Segments** are immutable, columnar files indexed by time. Old segments live on object storage; recent ones on Historical nodes' local disk.
- **Real-time ingest** via Kafka or Kinesis; data is queryable within seconds.
- **Bitmap indexes** on dimensions for fast filtering.
- **Roll-up** at ingest — aggregate raw events into pre-aggregated rows (e.g., per-minute counts) to shrink the index.
- **Approximate algorithms** built in (HLL for cardinality, theta sketches for unique counts, t-digest for quantiles).
- **Time partitioning** is first-class; every query is bounded by a time range.

Strengths: time-series aggregations, faceted dimensions (dropdown filters), funnel/cohort analysis, real-time ingest from Kafka, mature.

Weaknesses: operational complexity (many node types), schema-on-ingest (changes require re-ingest), updates / deletes are awkward, joins are limited.

Used by: Netflix, Airbnb, Walmart, eBay, Confluent, Imply (the commercial vendor).

---

## 4. Apache Pinot

Pinot (LinkedIn 2014, now Apache + StarTree) is in the same family. The original use case: LinkedIn's "Who Viewed Your Profile" — billions of events, hundreds of millions of users, must answer instantly.

### Architecture

```
┌──────────────────────────────────────────────┐
│ Controllers (cluster mgmt, segment assign)    │
├──────────────────────────────────────────────┤
│ Brokers (query routing, scatter/gather)       │
├──────────────────────────────────────────────┤
│ Servers (segment storage + execution)         │
│  - Real-time servers (Kafka consumption)      │
│  - Offline servers (batch ingest)             │
├──────────────────────────────────────────────┤
│ Minions (compaction, segment ops)             │
└──────────────────────────────────────────────┘
```

Pinot's distinctive features:
- **Upsert tables** — modify rows by primary key in real time. Few real-time OLAP engines support this; Pinot is the leader. Critical for LinkedIn-style "current state per user" queries.
- **Star-tree index** — pre-aggregated cube for very fast aggregations on common dimension combinations.
- **Lots of index types** — bloom filter, sorted, inverted, range, geospatial, text, JSON. Tunable per column.
- **Tiered storage** — recent data on SSD, older on slower storage / S3.
- **Real-time ingest from Kafka** with exactly-once semantics.

Strengths: upserts in real time, very high concurrency (LinkedIn runs > 100K QPS), rich indexing, multi-tenant.

Weaknesses: heavy operationally, learning curve for index choices, joins are limited (use sub-query / lookup patterns), schema management is involved.

Used by: LinkedIn (origin), Uber (UberEats analytics), Walmart, Stripe, Slack.

---

## 5. ClickHouse

ClickHouse (Yandex 2009, open-sourced 2016, ClickHouse Inc. 2021) is the simpler-feeling member of the trio. SQL-first, MergeTree storage, exceptional single-node performance.

### Architecture

```
┌──────────────────────────────────────────────┐
│ ClickHouse nodes (homogeneous)               │
│  - MergeTree tables (columnar, partitioned)  │
│  - Replicated tables (ZK / KeeperCoordinated)│
│  - Distributed table (routes / aggregates)   │
└──────────────────────────────────────────────┘
        │                          │
        ▼                          ▼
   Local disk / SSD           Optional remote tier
                              (S3 / GCS object storage)
```

ClickHouse's distinctive features:
- **MergeTree engine** family — column-oriented, sorted by primary key, partitioned by time (or any expression), with background merges. Read-and-write throughput off the charts.
- **Vectorized query execution** — SIMD, batched operations. Very fast scans.
- **Materialized views** — pre-aggregated tables that update incrementally on insert.
- **SQL-first** with Postgres-flavor syntax.
- **Streaming via Kafka engine, ReplacingMergeTree for "upserts" (via background dedup)**.
- **Dictionaries** — small in-memory tables for fast lookups in queries.
- **Distributed table** for horizontal scale; replicas via ReplicatedMergeTree.

Strengths: simplicity, raw speed on a single node (no sharding required for many workloads), excellent SQL, strong ecosystem (Grafana, dbt-clickhouse, Materialize, Tinybird).

Weaknesses: real updates are limited (background dedup, not transactional), joins are slower than dedicated joins (use IN / dictionary patterns), exactly-once is harder than Pinot's, ZooKeeper / Keeper operational burden for replicated clusters (mitigated in recent versions).

Used by: Cloudflare (DNS analytics — billions of events/day), Uber, Yandex, Spotify, Mux, Tinybird.

---

## 6. The three side by side

| | Druid | Pinot | ClickHouse |
|---|---|---|---|
| Origin | Metamarkets (2011) | LinkedIn (2014) | Yandex (2009) |
| Primary model | Time-series segments | Real-time + batch tables | MergeTree columnar |
| Ingest streaming | Kafka, Kinesis | Kafka, Kinesis | Kafka engine, materialized views |
| Sub-second queries | Yes | Yes | Yes |
| Real-time updates | Awkward (re-ingest) | First-class (upserts) | ReplacingMergeTree (eventual) |
| Joins | Limited | Limited | Real but slow; prefer dictionary lookups |
| Approximate aggregations | HLL, theta sketches built in | HLL, sketches | uniqHLL12, quantiles, all built-in |
| Operational complexity | High (5+ node types) | High | Medium (homogeneous + ZK/Keeper) |
| Learning curve | Steep | Steep | Moderate (SQL helps) |
| Best at | Time-series with many filters | Real-time upserts, high QPS | General-purpose blazing OLAP |
| Hosted | Imply, AWS managed | StarTree | ClickHouse Cloud, Tinybird, Altinity |

There is no universal winner. Pick by team and use case:

- "We're already a Java shop, need real-time per-user current state, > 50K QPS" → **Pinot**.
- "We want time-series segmentation, deep funnel analytics, many roll-up patterns" → **Druid**.
- "We want SQL, simpler ops, fast single-node, integrating with the rest of our stack" → **ClickHouse**.

In 2026, **ClickHouse has grown the fastest** of the three. Tinybird (managed ClickHouse) and ClickHouse Cloud have lowered the activation energy substantially. Many new builds start there.

---

## 7. The non-OSS landscape

Several adjacent tools you'll encounter:

- **Apache Doris** / **Apache Kylin** — Chinese-origin OLAP engines; growing in international use.
- **Rockset** (acquired by OpenAI) — fully managed, designed for product analytics. Still operates as a brand.
- **Materialize** — streaming materialized views (Timely Dataflow). Standing SQL queries.
- **RisingWave** — newer streaming SQL database.
- **StarRocks** — open-source MPP SQL engine, lakehouse-friendly.
- **Apache Doris** — MPP OLAP from China, similar feel to ClickHouse.
- **Tinybird** — managed ClickHouse + an API layer for shipping analytics endpoints.
- **Cube** — semantic layer + caching often used in front of ClickHouse/Snowflake.
- **Apache Druid (Imply managed)**, **StarTree (Pinot managed)**, **Altinity (ClickHouse managed)**.

For most teams: pick one of {ClickHouse, Druid, Pinot} based on use case + managed-vs-self trade-off, then standardize.

---

## 8. Typical use cases

### Product analytics dashboard

"Show me clicks, signups, conversion by country, device, plan, over the last 24 hours / 7 days / 90 days."

Common shape: dozens of dashboards, hundreds of users, queries every few seconds, data freshness in seconds. All three engines fit. ClickHouse is the modern default.

### Real-time observability

Metrics + logs + traces at scale. **ClickHouse** powers Cloudflare, Uber's M3 + ClickHouse, and many internal observability stacks. Druid is used at Airbnb for the same. Recent observability tools (SigNoz, Hyperdx) are ClickHouse-based.

### Fraud / risk scoring

Streams of transactions; aggregated stats per account, IP, device, country in real time. Pinot's upsert capability shines here.

### Ad tech

Impressions / clicks / conversions for hundreds of thousands of campaigns. Sub-second response, billions of events. Druid is the canonical choice; ClickHouse is increasingly used.

### Embedded analytics for SaaS

You're a SaaS company; customers see analytics dashboards. The dashboard hits your real-time analytics engine on every page load. ClickHouse + Tinybird is a popular shape.

### Time-series at high cardinality

When Prometheus or InfluxDB hit cardinality walls, teams migrate to ClickHouse or Pinot for huge label spaces.

---

## 9. Operational patterns

### Pre-aggregation / roll-up

Raw events at ingest time can be aggregated by minute / hour. A million events per hour become a few thousand pre-aggregated rows. Query is much faster; storage is much smaller. Trade-off: lose row-level inspection past the retention of raw data.

### Time partitioning

Every fact table partitioned by event timestamp. Old partitions move to cheaper storage (S3) or get dropped on retention. Recent partitions stay hot.

### Approximate algorithms

For unique counts, percentiles, top-K, exact computation over billions of rows is wasteful. **HyperLogLog**, **theta sketches**, **t-digest**, **count-min sketch** — all are first-class in modern OLAP engines. See [Count-Min Sketch & HyperLogLog →](../08-distributed-systems/probabilistic-data-structures.md).

### Materialized views

Pre-compute common aggregations; auto-update on inserts. ClickHouse's MV is mature; Druid's data-cube model is similar.

### Tiered storage

Hot data on local NVMe; warm on EBS / cluster SSD; cold on S3 / GCS. Druid and Pinot do this natively; ClickHouse has S3-backed storage tiers since 2022.

### Hybrid OLTP + OLAP

Some teams replicate Postgres / MySQL into a real-time analytics engine via CDC (Debezium → Kafka → engine). Live transactional reads stay in the OLTP; analytics queries hit the OLAP replica. See [Change Data Capture →](../04-databases/cdc.md).

---

## 10. Common Mistakes / Anti-Patterns

- **Using a data warehouse for user-facing dashboards.** First production traffic spike → query queue → users wait 10s. Real-time analytics engine fixes it.
- **Using a real-time analytics engine for everything.** Ad-hoc data engineering and ETL belong in the warehouse / lakehouse. Both engines have a role.
- **No time partitioning.** Queries scan all history. Always partition by time on event tables.
- **Updates that conflict with engine model.** Druid doesn't really do row updates. ClickHouse's "updates" are async background. Pinot is your friend for true upserts.
- **Joins.** All three handle joins poorly compared to a warehouse. Denormalize, use dictionary lookups, or move joins to ETL upstream.
- **Cardinality explosion.** High-cardinality dimensions (UUID per row) blow up bitmap indexes. Bucket or hash.
- **No retention plan.** Tables grow forever; performance degrades; storage cost explodes. Set TTLs.
- **No data quality enforcement at ingest.** Garbage event → undetected for weeks → wrong dashboards.
- **One giant cluster shared by every tenant.** Noisy-neighbor problems. Use multi-tenant patterns or per-customer clusters at high scale.
- **No materialized views for hot queries.** Same aggregation runs every second. Materialize it.
- **Treating sub-second p99 as automatic.** Achievable but tuning-intensive. Plan for it.
- **Ignoring approximate algorithms.** Exact `COUNT(DISTINCT user_id)` over 10B rows is needlessly expensive. HLL is usually within 1% accuracy.
- **Schema-on-read in Druid / Pinot.** They're schema-on-write. Get the schema right early, or plan for reindexing.
- **No backups.** "It's just a cache." It's the production analytics datastore. Back it up.

---

## 11. Cheat Card

```
PURPOSE   Sub-second OLAP on fresh, high-volume event data —
          for user-facing dashboards and real-time signals.

CHARACTERISTICS
  Columnar, distributed, time-partitioned
  Streaming + batch ingest
  Pre-aggregation / materialized views
  Approximate algorithms first-class (HLL, t-digest, sketches)
  Sub-second p99 at thousands of QPS

THE BIG THREE
  Druid       time-series segments; bitmap-rich; mature
  Pinot       real-time upserts; star-tree; LinkedIn-grade QPS
  ClickHouse  SQL-first; MergeTree; simpler ops; growing fastest

WHEN TO USE EACH (DEFAULT TAKES)
  ClickHouse — general-purpose, SQL-friendly, modern default
  Pinot      — real-time upserts, very high QPS
  Druid      — time-series segmentation, deep funnels

PATTERNS
  Time partition all fact tables; TTL old partitions
  Pre-aggregate at ingest (roll-up) when granularity allows
  Materialized views for hot queries
  Approximate algorithms for COUNT(DISTINCT) and quantiles
  Denormalize; avoid joins; use dictionary lookups
  CDC from OLTP → engine for combined transactional + analytical

WHEN NOT TO USE
  Ad-hoc data engineering / heavy ETL → warehouse
  Joins of comparable big tables → warehouse
  Low-latency single-row reads → OLTP

PITFALLS
  Using data warehouse for user-facing dashboards
  No time partitioning → full-history scans
  Joins ignoring the engine's limits
  Cardinality bombs (UUID per row in a dimension)
  No retention / TTL plan
  Schema-on-write engines treated as schema-on-read
  No backups because "it's just a cache"

RULE   Real-time analytics complements the warehouse — it doesn't
       replace it. Pick by team comfort and update semantics:
       ClickHouse first, Pinot for upsert-heavy, Druid for time-
       series cubes.
```

---

## 12. Resources

### Documentation
- **Apache Druid** — <https://druid.apache.org/docs/latest/>
- **Apache Pinot** — <https://docs.pinot.apache.org>
- **ClickHouse** — <https://clickhouse.com/docs>
- **ClickHouse Cloud** — <https://clickhouse.com/cloud>
- **Tinybird** — <https://www.tinybird.co/docs>
- **Apache Doris / StarRocks** — <https://doris.apache.org> / <https://www.starrocks.io>

### Papers
- "Druid: A Real-time Analytical Data Store" — Yang, Tschetter, et al., 2014.
- "Pinot: Realtime OLAP for 530 Million Users" — Im et al. (LinkedIn), 2018.
- "ClickHouse: A Modern OLAP DBMS" — Schulze et al., 2024 (overview paper).

### Articles
- "Real-time analytics at LinkedIn" — LinkedIn engineering blog (Pinot origin).
- "How Netflix uses Druid" — Netflix Tech Blog.
- "Cloudflare's DNS analytics: 10 trillion queries on ClickHouse" — Cloudflare engineering.
- "How Discord uses ClickHouse" — Discord engineering.
- "Apache Druid vs. Apache Pinot vs. ClickHouse" — multiple comparison pieces.

### Videos
- *Apache Druid: A Real-time Analytical Data Store* — early talks.
- *Apache Pinot at LinkedIn / Uber* — engineering talks.
- *ClickHouse Meetups* — quarterly.
- ByteByteGo — "Real-time OLAP Explained."

### Tools
- **Druid** (Imply Cloud, AWS Managed).
- **Pinot** (StarTree Cloud).
- **ClickHouse** (ClickHouse Cloud, Tinybird, Altinity).
- **Materialize / RisingWave** — streaming SQL.
- **Apache Doris / StarRocks** — alternatives.
- **Cube** — semantic + caching layer.
- **dbt-clickhouse** — dbt for ClickHouse.

### Adjacent reading
- [Apache Spark →](../17-big-data/spark.md)
- [Apache Flink →](../17-big-data/flink.md)
- [Stream Processing →](../07-messaging/stream-processing.md)
- [Batch vs Stream Processing →](../07-messaging/batch-vs-stream.md)
- [OLTP vs OLAP →](../04-databases/oltp-vs-olap.md)
- [Data Warehouses & Data Lakes →](../04-databases/warehouses-lakes.md)
- [Lakehouse Architecture →](../04-databases/lakehouse.md)
- [Time-Series Databases →](../04-databases/time-series-databases.md)
- [Count-Min Sketch & HyperLogLog →](../08-distributed-systems/probabilistic-data-structures.md)
- [Change Data Capture →](../04-databases/cdc.md)
- [Design a Metrics & Monitoring System →](../18-case-studies/monitoring-system.md)
- [Design an Ad Click Aggregator →](../18-case-studies/ad-click-aggregator.md)

---

*Previous:* [← Embedding-Based Retrieval](./embedding-retrieval.md)  |  *Next:* [Edge Computing →](./edge-computing.md)
