# ETL vs ELT

> **TL;DR** — **ETL** (Extract → Transform → Load) does the work *before* loading data into the warehouse: read from sources, reshape and clean in a separate compute layer, then write final tables. **ELT** (Extract → Load → Transform) loads raw data first and lets the warehouse do the transforms in-place via SQL. The shift from ETL to ELT is the defining architectural change of the modern data stack: when warehouses got cheap, fast, and elastic (Snowflake, BigQuery, Databricks SQL, Redshift), pulling data *into* them and transforming with SQL beat pulling data *out*, transforming in a separate engine, and pushing it back. ELT plus a transformation framework like **dbt** is the consensus 2026 default. ETL is still right when transforms must happen before storage for **compliance, PII redaction, schema enforcement, or sheer cost reasons**. The honest take: **most teams should default to ELT for warehouse work; ETL still rules for streaming, edge, and any pipeline where the load is the expensive part**.

---

## 1. The big picture

```
ETL (the old default)
─────────────────────
  ┌────────┐    ┌────────┐    ┌────────┐    ┌──────────┐
  │ Source │ ─► │Extract │ ─► │ Trans- │ ─► │ Warehouse│
  │  (DB,  │    │ (CDC,  │    │ form   │    │  (final  │
  │ files, │    │ batch) │    │ engine │    │  tables) │
  │ APIs)  │    │        │    │ (Spark │    │          │
  └────────┘    └────────┘    │ /Beam) │    └──────────┘
                              └────────┘

ELT (the new default)
─────────────────────
  ┌────────┐    ┌────────┐    ┌──────────┐    ┌─────────┐
  │ Source │ ─► │Extract │ ─► │ Warehouse│ ─► │  SQL    │
  │        │    │ + Load │    │  (raw    │    │trans-   │
  │        │    │ (Fivetran,  │  tables) │    │forms    │
  │        │    │ Airbyte)│   │          │    │(dbt)    │
  └────────┘    └────────┘    └──────────┘    └─────────┘
                                    │              │
                                    └─► final tables in same warehouse
```

The interesting difference is **where the transform runs**:

- ETL — in a separate compute engine (Spark, Informatica, Talend, custom code).
- ELT — inside the warehouse using SQL.

Everything else follows from that decision.

---

## 2. Why ELT won the warehouse

A list of changes that made ELT the better default in roughly the order they hit:

- **Cheap, separated storage and compute.** Snowflake (2014+), BigQuery (2010+), Redshift / Athena, Databricks SQL — all let you store huge raw data cheaply and scale compute on demand. Earlier warehouses (Teradata, classic Oracle DW) were priced and provisioned in a way that made "load everything raw" suicidal.
- **Columnar storage and vectorized SQL.** Modern warehouses scan TB in seconds. Transforming with SQL is genuinely fast.
- **Managed ingestion.** Fivetran, Airbyte, Stitch, Hevo turned "extract + load" into a connector configuration. No more bespoke Python scripts that break when an API changes.
- **dbt.** Made SQL transforms testable, version-controlled, modular, and dependency-managed. Turned the warehouse into a software-engineering target.
- **Schema-on-read tolerance.** Modern warehouses handle semi-structured (JSON, VARIANT) natively. You can load messy raw data without committing to a schema.
- **Analyst empowerment.** The people who know the business — analysts, analytics engineers — write SQL. Forcing transformations into Spark/Java filtered them out.

The result: most companies that started building data platforms in 2018+ skipped Spark ETL entirely for warehouse work and went straight to Fivetran/Airbyte + Snowflake/BigQuery + dbt.

---

## 3. The two architectures, side by side

| Dimension | ETL | ELT |
|---|---|---|
| Where transform runs | Separate compute (Spark, Informatica, custom) | Inside warehouse (SQL) |
| Storage of raw data | Often discarded or kept briefly | Persisted as a "raw" layer |
| Schema | Enforced before load | Inferred / evolved over time |
| Cost shape | Compute-heavy in the transform engine | Storage + warehouse compute |
| Latency to load | Slower (transforms first) | Faster (raw loads quickly) |
| Reprocessing | Re-run upstream, expensive | Re-run downstream SQL, cheap |
| Schema changes | Pipeline brittle | Warehouse columnar + JSON tolerant |
| Best for | Streaming, edge, PII redaction at edge, IoT, tight cost control on cheap warehouses | Analytics, BI, ML feature stores backed by warehouses |
| Tooling | Spark, Beam, Informatica, custom code | Fivetran/Airbyte + Snowflake/BigQuery/Databricks + dbt |

The clearest practical signal: **does your warehouse cost grow linearly with stored bytes, or with scanned bytes?** If with scanned bytes (Snowflake, BigQuery), ELT is cheaper than it looks. If with stored bytes (older or self-hosted), the calculus shifts.

---

## 4. ETL — when to choose it

ETL still wins for specific shapes:

### Streaming and real-time

You can't ELT a stream. The transform has to happen as events flow. Use Flink, Spark Structured Streaming, Kafka Streams, or Beam. See [Apache Flink →](./flink.md), [Stream Processing →](../07-messaging/stream-processing.md).

### PII and compliance

Some data **must not land in the warehouse in raw form**. Credit card numbers, social security numbers, medical identifiers — masked, tokenized, or aggregated before they touch the analytical store.

ETL gives you a transformation step that runs *before* the warehouse sees the data. That's the only safe place to hash/redact under many compliance regimes (GDPR, HIPAA, PCI-DSS).

### Edge and IoT

Devices stream sensor data over expensive or unreliable links. Pre-aggregate at the edge before shipping to the warehouse. Send 1 KB summaries, not 1 GB of raw frames.

### Sheer cost

If your warehouse charges by scan ($5–10 per TB) and your raw data is mostly noise, materializing only the clean subset is cheaper. Same logic applies to ML feature stores with bounded budgets.

### Legacy warehouse environments

Teradata, classic Oracle DW, on-prem Vertica — all priced and provisioned for "load only what you need." ELT was a bad idea on those.

### Strict typing requirements

Some regulated industries demand that data conform to a contract before it's persisted. ETL enforces that boundary at the transform step.

---

## 5. ELT — when to choose it

For analytics work in 2026, default to ELT. Specifically:

- **Analytics, BI, dashboards.** Warehouse-native; SQL is the lingua franca.
- **Data science exploration.** Analysts and DS folks need raw data to figure out what to ask.
- **Frequent schema changes upstream.** ELT absorbs them; ETL pipelines break.
- **Time-travel and audit needs.** Raw layer is permanent record; transforms are reproducible from it.
- **Backfills.** Re-deriving downstream tables is one SQL refresh, not a multi-week Spark job.
- **Multi-team consumption.** Each team can build their own transforms off the shared raw layer.

The corresponding cost: you store more (raw + transformed), and the warehouse query bill grows. For most companies, that bill is far smaller than the ETL engineering team that would otherwise be required.

---

## 6. The modern data stack — the canonical ELT shape

A typical 2026 setup:

```
   Sources                Extract+Load         Warehouse        Transform        Serve
   ────────               ─────────────       ───────────      ───────────      ─────
   ┌──────────┐           ┌───────────┐       ┌─────────┐      ┌───────┐       ┌──────┐
   │ Postgres │ ─CDC────►│  Fivetran │ ─────►│         │      │       │       │      │
   │ MySQL    │           │  Airbyte  │       │ Snow-   │ SQL  │ dbt   │ SQL   │ BI:  │
   │ Stripe   │ ─API────►│  Stitch   │ ─────►│ flake / │ ────►│       │ ────► │Looker│
   │ Salesforce│          │  Hevo     │       │ BQ /    │      │ models│       │Tableau│
   │ Kafka    │ ─Sink───►│           │ ─────►│ Databri │      │       │       │Hex   │
   │ Files    │           └───────────┘       │ -cks SQL│      │       │       │      │
   └──────────┘                               └─────────┘      └───────┘       └──────┘

                                            ↑
                                  Raw layer    Stage layer    Mart layer
```

Three logical layers inside the warehouse:

- **Raw** (`raw_postgres_orders`, `raw_stripe_charges`) — exactly what landed, untransformed. Append-only ideally.
- **Stage** (`stg_orders`, `stg_charges`) — cleaned, typed, renamed. Each source table maps 1:1 to a stage model.
- **Mart** (`fct_revenue_daily`, `dim_customer`) — business-facing fact and dimension tables. What BI consumes.

dbt enforces this pattern with explicit references between models. The DAG is encoded in code; the warehouse builds it on demand.

For details on the table shapes, see [Data Modeling for Analytics →](./dimensional-modeling.md).

---

## 7. The role of dbt

**dbt** (data build tool) is the de facto standard for ELT transforms. Its model:

- Each transform is a SQL `SELECT` in a `.sql` file.
- dbt wraps it in `CREATE TABLE` / `CREATE VIEW` / `INSERT` based on materialization config.
- Models reference each other with `{{ ref('other_model') }}`.
- dbt builds a DAG, runs models in dependency order, in parallel where possible.
- Tests (`unique`, `not_null`, `accepted_values`, custom) run against materialized data.
- Documentation auto-generated from YAML annotations + SQL.

```sql
-- models/marts/fct_revenue_daily.sql
{{ config(
  materialized='incremental',
  unique_key='dt',
  incremental_strategy='delete+insert'
) }}

SELECT
  DATE(order_at) AS dt,
  merchant_id,
  SUM(amount_cents) AS revenue_cents,
  COUNT(*) AS orders
FROM {{ ref('stg_orders') }}
WHERE 1=1
  {% if is_incremental() %}
    AND order_at > (SELECT MAX(dt) FROM {{ this }})
  {% endif %}
GROUP BY 1, 2
```

What this PR-able SQL provides:
- Version control (git diff on transforms).
- Tests (every model can declare assertions).
- Lineage (dbt knows what depends on what).
- Reproducibility (same SQL + same data = same output).

dbt put data engineering on a software-engineering footing. It's not optional in 2026 for most ELT shops.

---

## 8. CDC — keeping the raw layer fresh

**Change Data Capture** (CDC) is how transactional sources stream into the warehouse without batching huge daily exports.

- **Debezium** reads DB transaction logs (Postgres WAL, MySQL binlog) and emits row-level changes to Kafka.
- **Fivetran / Airbyte CDC** handles the plumbing.
- **AWS DMS** / **GCP Datastream** — cloud-managed CDC.
- The result: raw tables in the warehouse update within seconds to minutes of source changes.

CDC is the bridge between transactional reality and analytical convenience. Without it, you're stuck with batched dumps that go stale fast.

See [Change Data Capture →](../04-databases/cdc.md).

---

## 9. Reverse ETL — closing the loop

The newer wrinkle: once you have nice tables in the warehouse, you often want to push that data **back into operational systems**. "Reverse ETL":

- Sync customer segments from the warehouse to Salesforce or HubSpot.
- Push computed propensity scores into the product DB.
- Update Stripe customer metadata from analytical models.

Tools: **Hightouch**, **Census**, **Polytomic**. The pattern: the warehouse becomes the source of truth for operational data, not just a destination.

This is the full closed loop of the modern data stack: operational systems → warehouse → analytics → reverse ETL → operational systems.

---

## 10. Tooling landscape

A non-exhaustive taxonomy:

### Extract + Load (E + L)
- **Fivetran** — enterprise, broad connector library, expensive.
- **Airbyte** — open source + cloud, growing connector set.
- **Stitch / Talend Stitch** — early entrant, owned by Talend.
- **Hevo** — managed, similar to Fivetran.
- **Meltano** — open-source Singer-based, less polished.
- **AWS Glue / Data Pipeline** — AWS-native; complex.
- **Google Cloud Dataflow** — Beam-based; capable.
- **Debezium + Kafka Connect** — CDC + ingestion, DIY.

### Warehouses
- **Snowflake** — multi-cloud, default for many.
- **BigQuery** — GCP-native, serverless, generous free tier for small.
- **Databricks SQL** — lakehouse-native.
- **Amazon Redshift** — older but improving.
- **ClickHouse Cloud / Trino + Iceberg** — open-source-friendly.

### Transformation (T)
- **dbt** — the standard.
- **SQLMesh** — newer competitor; emphasizes blue-green deploys.
- **Dataform** — Google's dbt alternative, now part of BigQuery.

### Orchestration
- See [Data Pipelines & Orchestration →](./data-pipelines.md).

### Reverse ETL
- **Hightouch**, **Census**, **Polytomic**.

### Streaming ETL
- **Flink**, **Spark Structured Streaming**, **Kafka Streams**, **Beam**, **Materialize**, **RisingWave**.

For a green-field setup in 2026: **Fivetran/Airbyte + Snowflake/BigQuery + dbt + Airflow/Dagster + Hightouch** is the boring, well-tooled, broadly-staffed shape.

---

## 11. Operational concerns

### Idempotency

Every pipeline run should be safely re-runnable. Use:

- Upserts keyed on natural / surrogate keys.
- `MERGE` statements.
- Partition-overwrite patterns.
- dbt `incremental` models with proper unique keys.

A pipeline you can re-run is a pipeline you can debug. One you can't is one you fear.

### Backfills

Schema changed; you need to reprocess history. ELT makes this cheap — re-run the affected SQL. ETL makes it expensive — re-extract, re-transform.

Build with backfill in mind: parameterize date ranges, make all transforms deterministic, log everything.

### Data quality

dbt tests (`unique`, `not_null`, `relationships`, `accepted_values`) catch routine issues. **Great Expectations** and **Soda** add richer assertions. Run on a schedule; alert on regressions.

The hard truth: most data outages aren't about transformation logic — they're about *source data changing shape*. Detection is more valuable than prevention.

### Observability

- **Lineage**: dbt's auto-generated graph + tools like Monte Carlo, Acceldata, Lightup.
- **Freshness**: how stale is each table? Alert on stale critical tables.
- **Volume anomalies**: did 10× as many rows arrive today? Or 0?
- **Schema drift**: source added a column → did anything break?

A modern data stack is mature when "is the data correct?" can be answered without manually running queries.

### Cost

Snowflake / BigQuery bills can grow alarming. Common levers:

- **Limit scans**: clustered tables, partition pruning, materialized views.
- **Right-size warehouses**: small for routine, scale up for big jobs.
- **Auto-suspend** unused warehouses.
- **Pre-aggregate** high-fanout queries.
- **Monitor cost per model** (dbt + Snowflake info schema).

The big-data version of "premature optimization" is **too many materialized full-history tables**. Materialize only what's used; incremental-ize the rest.

---

## 12. Common Mistakes / Anti-Patterns

- **Choosing ETL by reflex because that's what you learned in 2015.** Default ELT for warehouse analytics now.
- **Choosing ELT for streaming.** Doesn't fit — streams need transforms inline.
- **No raw layer.** All transforms baked into ingestion → can't reprocess history without re-ingest.
- **Schemas in the raw layer.** Make raw permissive; enforce types in stage.
- **dbt models with no tests.** Quality regressions ship silently.
- **One giant dbt model.** Break it up. dbt's lineage is the value.
- **Source-to-mart direct, no stage.** Stage layer is where source quirks die. Skipping it leaks them everywhere.
- **Full-refresh of huge tables every run.** Use incremental materializations.
- **Reverse ETL writing to operational systems without idempotency.** Customer's plan downgraded then upgraded twice.
- **PII in the raw layer with permissive access.** Compliance violation waiting to happen.
- **No CDC, daily full dumps.** Latency stuck at "yesterday."
- **Spark ETL kept around because "we already have it."** Migration cost is real, but new pipelines should start in dbt.
- **Putting business logic in BI tool calculations.** Logic should live in the warehouse (dbt), not in Looker / Tableau / Hex calculated fields. The "single source of truth" only works if it's actually one source.
- **No cost dashboards.** Warehouse bills surprise CFOs at quarter-end.
- **Treating Fivetran/Airbyte as set-and-forget.** Connectors break when source schemas change. Monitor.
- **Streaming and batch versions of the same logic, drifting apart.** Lambda architecture pain. Use Flink / Materialize / Spark Structured Streaming for unified models, or commit to one path.

---

## 13. Cheat Card

```
PURPOSE   Move data from sources to a place where it can be
          analyzed, modeled, and acted on — with the transform
          step before or after the load.

ETL    transform before load (separate engine)
ELT    transform after load (in-warehouse SQL)

ELT WINS WHEN
  Warehouse is cheap and elastic (Snowflake, BQ, Databricks)
  Analytics, BI, ML feature engineering
  Schema drift common upstream
  Reprocessing / backfills are routine
  Multiple teams consume the same raw

ETL WINS WHEN
  Streaming — transforms happen as events flow
  PII / compliance demands pre-load redaction
  Edge / IoT — pre-aggregate before shipping
  Bandwidth or warehouse-scan costs dominate
  Strict schema enforcement before storage

MODERN STACK SHAPE
  Sources → Fivetran / Airbyte / Debezium → Warehouse → dbt → BI
  Three layers in the warehouse: raw → stage → mart
  Reverse ETL (Hightouch / Census) closes the loop

DBT NON-NEGOTIABLES
  Versioned SQL transforms
  ref() + DAG → dependency-aware runs
  Tests (unique, not_null, relationships, etc.)
  Incremental materializations for big tables
  Docs and lineage generated

OPERATIONAL ESSENTIALS
  Idempotent pipelines (re-run safely)
  Backfill plans baked in
  Data quality tests on schedule
  Lineage + freshness + volume anomaly alerts
  Cost dashboards per model / per warehouse

PITFALLS
  ETL by reflex for warehouse work
  ELT for streaming
  No raw layer → can't reprocess history
  Skipping stage layer
  No tests; no incremental models; no monitoring
  PII in raw with permissive access
  Business logic in BI calculations, not dbt
  Drift between streaming and batch versions

RULE   Default ELT for warehouse analytics; ETL for streaming
       and pre-storage transforms. Raw + stage + mart layers.
       dbt for SQL transforms; orchestrator for everything else.
```

---

## 14. Resources

### Documentation
- **dbt** — <https://docs.getdbt.com>
- **Fivetran** — <https://fivetran.com/docs>
- **Airbyte** — <https://docs.airbyte.com>
- **Snowflake** — <https://docs.snowflake.com>
- **BigQuery** — <https://cloud.google.com/bigquery/docs>
- **Databricks SQL** — <https://docs.databricks.com/sql/>
- **Debezium** — <https://debezium.io>

### Books
- *The Data Warehouse Toolkit* — Ralph Kimball. The dimensional modeling bible (applies to ELT too).
- *Fundamentals of Data Engineering* — Reis & Housley.
- *The Self-Service Data Roadmap* — Sandeep Uttamchandani.

### Articles
- "ELT vs ETL: a modern take" — dbt Labs / Fivetran / Snowflake blogs.
- "The modern data stack" — Tristan Handy (dbt) essays.
- "What is reverse ETL?" — Hightouch / Census engineering.
- "Why we replaced our ETL with dbt" — many teams' war stories.

### Videos
- *Coalesce* (dbt's annual conference) — recordings.
- *Snowflake Summit*, *Data + AI Summit* — sessions on ELT patterns.
- ByteByteGo — "ETL vs ELT Explained."

### Tools
- **Extract + Load**: Fivetran, Airbyte, Stitch, Hevo, Meltano, Debezium, AWS DMS, GCP Datastream.
- **Warehouses**: Snowflake, BigQuery, Databricks SQL, Redshift, ClickHouse Cloud, Trino + Iceberg.
- **Transform**: dbt, SQLMesh, Dataform.
- **Orchestrate**: Airflow, Dagster, Prefect, Mage.
- **Reverse ETL**: Hightouch, Census, Polytomic.
- **Quality**: Great Expectations, Soda, Monte Carlo, Acceldata.

### Adjacent reading
- [MapReduce →](./mapreduce.md)
- [Hadoop Ecosystem →](./hadoop.md)
- [Apache Spark →](./spark.md)
- [Apache Flink →](./flink.md)
- [Data Pipelines & Orchestration →](./data-pipelines.md)
- [Data Modeling for Analytics →](./dimensional-modeling.md)
- [Change Data Capture →](../04-databases/cdc.md)
- [OLTP vs OLAP →](../04-databases/oltp-vs-olap.md)
- [Data Warehouses & Data Lakes →](../04-databases/warehouses-lakes.md)
- [Lakehouse Architecture →](../04-databases/lakehouse.md)
- [Stream Processing →](../07-messaging/stream-processing.md)
- [Batch vs Stream Processing →](../07-messaging/batch-vs-stream.md)

---

*Previous:* [← Apache Flink](./flink.md)  |  *Next:* [Data Pipelines & Orchestration →](./data-pipelines.md)
