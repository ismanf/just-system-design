# Data Pipelines & Orchestration (Airflow, Dagster, Prefect)

> **TL;DR** — A **data pipeline** is a chain of operations that move and transform data from source to use — extract, load, transform, validate, publish. An **orchestrator** is what runs that chain: it schedules tasks, manages dependencies, handles failures, retries, backfills, and gives you observability into what ran when. **Apache Airflow** is the elder statesman and remains the most-used choice. **Dagster** treats data assets (tables, files) as the unit of work and brings strong typing, testing, and observability. **Prefect** focuses on dynamic flows and a clean Python API. **Mage**, **Argo Workflows**, **Step Functions**, **dbt Cloud**, and **Temporal** sit nearby in adjacent shapes. The honest take: **orchestration is one of the highest-leverage investments a data team makes**, and most teams stick with the orchestrator they picked early. **Choose Airflow if you want the largest ecosystem; Dagster if you want strong typing and asset-aware modeling; Prefect if you want dynamic Pythonic flows**. The real value isn't the tool — it's the discipline of declaring dependencies, owning failures, and treating pipelines as code.

---

## 1. The big picture

A data pipeline is a DAG of tasks:

```
                       extract_orders
                          │
                          ▼
   ┌───── load_orders_raw ◄──── load_returns_raw ────┐
   │                                                  │
   ▼                                                  ▼
  stg_orders ◄─────────────── stg_returns
   │                                  │
   └─────────────┬────────────────────┘
                 ▼
        fct_revenue_daily
                 │
                 ▼
        publish_to_BI / Reverse ETL
```

The orchestrator's job:
- **Schedule** the DAG to run on a cadence (daily, hourly, event-driven, on demand).
- **Resolve dependencies** — never run a downstream task before its upstreams complete successfully.
- **Parallelize** independent branches.
- **Retry** transient failures with backoff.
- **Surface failures** — alerts, logs, dashboards.
- **Track lineage** — what produced what, when, with which code version.
- **Backfill** — rerun for past periods when something was wrong or new logic ships.
- **Maintain idempotency boundaries** — each run is repeatable safely.

Without an orchestrator, you have cron + bash + Slack hopes. With one, you have an auditable, debuggable, recoverable pipeline.

---

## 2. The three eras of pipeline orchestration

A short history because it explains why the tools have the shapes they do.

### Era 1: cron + bash (1990s–2010s)

A cron job runs `etl.sh` every morning. Logs go to a file. If it fails, an email pings someone (maybe). Dependencies are encoded by hoping the upstream cron finished by 4 AM. This still runs much of the world's data infrastructure. It also breaks all the time.

### Era 2: Airflow (2014+)

Airbnb open-sourced **Apache Airflow** in 2015. Tasks defined as Python objects, dependencies as `>>` operators, runs scheduled by Airflow's scheduler, history persisted in a metadata DB, status visible in a web UI. The first orchestrator that the data world adopted en masse. Still the most-used.

### Era 3: asset-aware / dynamic / Python-native (2019+)

**Dagster**, **Prefect 2**, **Mage**, **Flyte** approached orchestration with first-principles takes:
- Treat data products (assets) as the unit, not just tasks.
- Real Python — pass values between tasks, branch on results.
- Test-first — pipelines are software.
- Cloud-native deploy.

Airflow has been catching up (TaskFlow API, dynamic task mapping, Datasets, Astro Cloud). The competition is healthy.

---

## 3. The orchestrators that matter

### Apache Airflow

Defines DAGs in Python:

```python
from airflow import DAG
from airflow.decorators import task
from datetime import datetime, timedelta

with DAG("daily_orders", start_date=datetime(2026,1,1),
         schedule="@daily", catchup=False,
         default_args={"retries": 3,
                       "retry_delay": timedelta(minutes=5)}) as dag:

    @task
    def extract():
        # pull from source
        ...

    @task
    def transform(data):
        # massage
        ...

    @task
    def load(rows):
        # write to warehouse
        ...

    load(transform(extract()))
```

Strengths:
- Massive ecosystem — operators for hundreds of systems.
- Mature scheduling with backfills, SLAs, sensors.
- Huge install base, easy to hire for.
- Managed offerings: Astronomer (Astro), AWS MWAA, GCP Cloud Composer.

Weaknesses:
- Heavy: scheduler + worker + metadata DB + web UI. Not lightweight.
- Tasks were historically state-less; passing data between tasks meant XComs or external storage.
- Idempotency / partitioning model is bolted-on (data interval, run_id).
- Local dev clunky; iterating on a DAG is slow.

Airflow 2.x cleaned up much of this with the TaskFlow API and Datasets, but the gravity of the design persists.

### Dagster

Defines **assets** rather than tasks:

```python
from dagster import asset, AssetIn

@asset
def orders():
    """Raw orders from Stripe."""
    return fetch_stripe_orders()

@asset
def daily_revenue(orders):
    """Aggregated by day."""
    return orders.groupby("date")["amount"].sum()
```

`daily_revenue` depends on `orders` because it takes it as an argument. The graph is implicit; no `>>` operators.

Strengths:
- **Asset-centric** — the DAG is "what tables / files exist," not "what scripts run."
- **Strong typing** — assets can declare schemas, dtypes, partition strategies.
- **Excellent testability** — assets are functions, easy to unit-test.
- **First-class dbt integration** — Dagster knows about dbt models.
- **Software-defined assets** — concept lets you reason about partial materializations.
- **Polished UI**.

Weaknesses:
- Younger ecosystem than Airflow (still smaller, growing fast).
- Some Airflow patterns (sensors, complex backfills) take more learning.
- Concept overhead — assets, ops, jobs, sensors, schedules, partitions — to absorb.

Managed offerings: Dagster Cloud.

### Prefect

Defines **flows** as decorated Python functions:

```python
from prefect import flow, task

@task(retries=3, retry_delay_seconds=60)
def extract():
    ...

@task
def transform(data):
    ...

@flow
def daily_orders():
    raw = extract()
    transform(raw)

if __name__ == "__main__":
    daily_orders()
```

Strengths:
- Lowest-friction Python orchestration.
- Dynamic flows — generate tasks at runtime.
- Hybrid execution: workflow logic in Prefect Cloud, code runs in your infra.
- Strong observability and event-driven model.

Weaknesses:
- Smaller ecosystem than Airflow.
- Prefect 2 was a rewrite of Prefect 1; some long-time users were burned.
- Less "asset-aware" than Dagster (though improving).

Managed: Prefect Cloud.

### Others worth knowing

| Tool | Niche |
|---|---|
| **Mage** | Newer, opinionated, blocks-not-DAGs UI. Younger but elegant. |
| **Argo Workflows** | Kubernetes-native; container per task. Good for ML / batch jobs already on K8s. |
| **Flyte** | Lyft-built; very strong typing; ML-focused. |
| **Kubeflow Pipelines** | ML pipelines on K8s. |
| **AWS Step Functions** | Serverless state machine; AWS-native. |
| **Temporal** | Durable workflows for app code; less analytics-focused. Excellent for transactional workflows. |
| **dbt Cloud** | Orchestrates dbt + simple jobs around it. Often combined with Airflow/Dagster, not a replacement. |
| **Luigi** | Spotify's predecessor to Airflow. Mostly retired. |
| **Oozie** | Hadoop-era. Retired. |
| **Apache Beam (Dataflow / Flink)** | Pipeline framework, not an orchestrator per se. |

---

## 4. The core concepts every orchestrator shares

### Task / op / activity

The atomic unit of work. A Python function, a SQL statement, a container, a Spark submit. Has inputs, outputs, retries, timeout, owner.

### DAG / flow / job

A directed acyclic graph of tasks. Has a schedule, a name, tags, ownership.

### Schedule

When the DAG runs: cron expression, interval, event-driven trigger (a file arrived, a Kafka message, an asset materialized).

### Run

A single execution of the DAG, with a logical date / partition. Distinct from the wall-clock time when it ran.

### Logical date / data interval / partition

The window of *data* this run covers. A "daily" run on 2026-05-20 processes data for 2026-05-20 (or 2026-05-19, depending on convention). Critical for idempotency and backfills.

### Backfill

Re-running the DAG for past dates. Should produce identical results to the original runs (idempotency).

### Retry policy

Failures get retried with backoff. Exponential, jittered, capped. Idempotency assumed.

### Sensors / event triggers

Wait for an external condition before proceeding: a file landing, a partition being available, a webhook arriving. Modern orchestrators prefer event-driven over polling sensors.

### Connections / variables / secrets

External credentials, configs. Managed centrally, encrypted at rest.

### Lineage / observability

What ran when, what it produced, what it consumed, how long it took. Dashboards, metrics, alerts.

---

## 5. Idempotency — the property that makes everything work

A pipeline is **idempotent** if running it twice with the same inputs produces the same result as running it once. This is the single most important property of a well-designed pipeline.

Why:
- Retries don't double-count.
- Backfills are safe.
- A failed mid-run pipeline can be re-run from the start (or from checkpoint) without manual cleanup.
- Two parallel runs (a scheduled one and a manual one) don't corrupt the output.

Techniques to achieve idempotency:

- **Overwrite by partition.** Write to `s3://bucket/orders/dt=2026-05-20/` always overwriting that prefix.
- **Upsert by key.** `MERGE INTO target USING source ON keys WHEN MATCHED UPDATE WHEN NOT MATCHED INSERT`.
- **Delete-and-insert.** Delete the date's rows, then insert. Inside a transaction.
- **Append-only with deduplication.** Append events; deduplicate downstream by event ID.
- **Tagged outputs.** Include a run_id in output paths; clean stale tags later.

Pipelines that aren't idempotent are pipelines that humans babysit at 3 AM.

---

## 6. Time, partitions, and backfills

This trips up everyone new to orchestration.

### Logical date vs wall-clock

A daily DAG runs *at* wall-clock 04:00 to process *data for* the previous day. The **logical date** (`ds`, `data_interval_start`) is the previous day's date. Your SQL should use the logical date, not `CURRENT_DATE`:

```sql
SELECT *
FROM orders
WHERE order_date = '{{ ds }}'   -- logical date templated by Airflow
```

If you use `CURRENT_DATE`, backfills produce wrong results — they pull "today's" data instead of "the date being backfilled."

### Partitions

Each run "owns" a partition (typically a date or hour). The output for a given partition should be deterministic from the input partition. Dagster's partition concept is first-class; Airflow handles it via templated dates and execution_date.

### Catchup vs no-catchup

`catchup=False` in Airflow means "don't backfill all missed runs on first deploy." Almost always the right default unless you explicitly want history.

### Backfills

When a transformation changes, you backfill: re-run for past partitions with new logic. Cost: warehouse compute. Benefit: history conforms to current model.

Plan for backfill cost from day one. Tables that backfill in seconds at small data become hours at scale. Materialization choices (full vs incremental) matter.

---

## 7. Failure handling

Pipelines fail. The orchestrator's value is partly in how gracefully they fail.

### Retry policies

```python
@task(retries=3, retry_delay=timedelta(minutes=5), retry_exponential_backoff=True)
def fragile_api_call():
    ...
```

Defaults: 1–3 retries, exponential backoff, cap at some maximum. Transient errors retry; permanent errors fail fast.

### Alerting

- Slack / PagerDuty / email on task failure.
- SLA misses (task didn't finish by expected time).
- Volume anomalies (row counts off by 50%).
- Source freshness (raw data didn't arrive on time).

### Owners

Every DAG has an owner team. Failure pages route to the right channel. Without ownership, the "data team Slack" becomes a graveyard of failed runs nobody investigates.

### Quarantine for bad data

If a row is malformed, quarantine it for review. Don't crash the pipeline; don't silently drop. Most modern orchestrators / dbt have patterns for this.

### Sentinel / canary jobs

A small canary DAG that exercises the system end-to-end. Alerts you when the platform itself is broken before real DAGs do.

---

## 8. Modern best practices

A composite checklist drawn from the field:

```
PIPELINE DESIGN
[ ] Tasks small enough to retry cheaply
[ ] Idempotent by partition / key
[ ] Logical date used; never CURRENT_DATE in transforms
[ ] DAG file imports are cheap (heavy imports inside tasks)
[ ] Code lives in your repo, not in the orchestrator UI
[ ] Code review on DAG changes
[ ] Tests in CI (unit, integration on a small dataset)

CONFIG AND SECRETS
[ ] Secrets in a secrets manager, fetched at runtime
[ ] Connection configs versioned in code
[ ] Per-environment configs (dev / staging / prod)

OBSERVABILITY
[ ] DAG-level SLA alerting
[ ] Task-level error → on-call channel
[ ] Lineage published or auto-tracked (dbt + Dagster)
[ ] Cost dashboards per DAG / per warehouse
[ ] Source freshness checks before kicking off downstream

ITERATION
[ ] Backfill plan documented for major changes
[ ] Old DAGs / models deprecated, not abandoned
[ ] Schema-changing PRs go through expand-contract
```

These don't require any particular orchestrator. They require culture.

---

## 9. A representative pipeline

A daily revenue pipeline using Airflow + dbt + Spark (a common shape):

```python
from airflow import DAG
from airflow.decorators import task
from airflow.providers.amazon.aws.operators.emr import EmrServerlessStartJobOperator
from airflow.providers.dbt.cloud.operators.dbt import DbtCloudRunJobOperator
from datetime import datetime

with DAG("daily_revenue",
         start_date=datetime(2026, 1, 1),
         schedule="@daily", catchup=False,
         max_active_runs=1) as dag:

    # 1. Spark ETL: raw events → conformed events on S3
    spark_job = EmrServerlessStartJobOperator(
        task_id="spark_etl",
        application_id="{{ var.value.emr_app_id }}",
        execution_role_arn="{{ var.value.emr_role }}",
        job_driver={
            "sparkSubmit": {
                "entryPoint": "s3://code/etl/conformed_events.py",
                "entryPointArguments": ["--dt", "{{ ds }}"],
            }
        },
    )

    # 2. dbt transforms: conformed → marts
    dbt_run = DbtCloudRunJobOperator(
        task_id="dbt_run",
        job_id=42,
        check_interval=30,
        timeout=3600,
    )

    # 3. Reverse-ETL to Salesforce
    @task
    def reverse_etl():
        ...

    spark_job >> dbt_run >> reverse_etl()
```

Each step is idempotent, each is observable, each has its own retries.

---

## 10. Where orchestration overlaps and conflicts with other tools

A frequent source of debate:

- **Orchestrator vs dbt scheduler**: dbt Cloud has its own scheduler. For pure-warehouse work, that may be enough. For anything that touches Spark, Kafka, external APIs, you want an orchestrator and use dbt as one of many node types.
- **Orchestrator vs workflow framework (Temporal, Step Functions)**: Temporal/Step Functions excel at *transactional* workflows — long-running app logic with strong guarantees. Airflow/Dagster excel at *batch data* workflows. They overlap; pick by the dominant workload.
- **Orchestrator vs CI/CD**: CI/CD deploys code; the orchestrator runs it. Sometimes confused (and sometimes you do want CI to run a smoke pipeline).
- **Orchestrator vs streaming**: Don't try to orchestrate per-event work in Airflow. Streaming engines (Flink, Kafka Streams) own that. Orchestrators handle batch and trigger-based work.
- **Orchestrator vs message queue**: A queue is fire-and-forget event processing. An orchestrator owns the DAG and the schedule.

A practical rule: **anything you want to backfill belongs in the orchestrator**. Anything you want to react to instantly belongs in a streaming engine.

---

## 11. Common Mistakes / Anti-Patterns

- **`CURRENT_DATE` in transforms.** Backfills produce wrong results.
- **Heavy imports at DAG-parse time.** Airflow re-parses DAGs constantly; heavy imports slow the scheduler. Move imports inside tasks.
- **Tasks too big.** A 4-hour task that retries from scratch wastes hours. Split into smaller, restartable tasks.
- **Tasks too small.** Per-row tasks crush scheduler throughput. Batch where it makes sense.
- **DAGs that depend on `previous_run_completed` implicitly.** Two parallel runs corrupt state. Use `max_active_runs=1` or design for parallelism.
- **No idempotency.** Retries double-write. Backfills are scary.
- **Secrets in code or environment variables checked into git.**
- **No owner per DAG.** Failures fall on the floor.
- **Alerting on every failure including retries.** Page fatigue → alerts ignored.
- **No alerting at all.** Failures discovered by users when dashboards are wrong.
- **Pipelines that depend on "the box at IP 10.0.0.5 with the cron job."** Single point of failure with no successor.
- **Building everything as one mega-DAG.** Encode meaningful subgraphs, separate concerns.
- **DAGs scheduled at the same exact minute (`0 4 * * *`).** All hit shared resources simultaneously. Stagger.
- **`schedule_interval` confused with "delay between runs."** It's the data interval, not the gap.
- **Manual UI clicks to fix things, no code change.** Drift. The orchestrator UI is read-only in mature setups.
- **No way to dry-run / test a DAG locally.** Iteration cost balloons.
- **Storing pipeline state in the orchestrator's metadata DB instead of in business outputs.** When you migrate orchestrators, you lose history.
- **Treating sensors as free.** A polling sensor that runs every 30 seconds for 8 hours is 960 useless task executions.

---

## 12. Cheat Card

```
PURPOSE   Run data pipelines reliably: dependencies, schedules,
          retries, backfills, observability, lineage — all as code.

CHOICES
  Airflow    biggest ecosystem; easiest to hire for; mature
  Dagster    asset-centric; strong typing; modern UX
  Prefect    Pythonic, dynamic flows; clean local dev
  Mage       younger, opinionated, blocks-not-DAGs UI
  Argo / Flyte / Kubeflow  K8s + ML-focused
  Step Functions / Temporal  app workflows, not analytics

CORE CONCEPTS
  Task / op / activity    one unit of work
  DAG / flow / job         the graph + schedule
  Schedule                 cron / interval / event
  Run + logical date       which partition this run owns
  Backfill                 rerun history with current logic
  Sensors / triggers       wait for external event
  Lineage / observability  see what produced what

IDEMPOTENCY (NON-NEGOTIABLE)
  Overwrite by partition / Upsert by key / Delete+Insert
  Use logical date, never CURRENT_DATE
  Same inputs + same code → same output

FAILURE HANDLING
  Retries with backoff (3 typical, exponential, capped)
  Alert on real failures, not retry storms
  Owners per DAG; on-call routing
  Quarantine bad data, don't drop silently

BEST PRACTICES
  Tasks small enough to retry cheaply
  DAG parse fast (heavy imports inside tasks)
  Code in git; PRs reviewed; CI tests DAGs
  Per-env configs; secrets via secrets manager
  Cost dashboards per DAG
  Documented backfill plan for big schema changes
  Stagger schedules; don't pile all DAGs at 04:00

PITFALLS
  CURRENT_DATE in SQL
  Long single tasks that can't checkpoint
  DAGs depending on previous-run side effects
  No owner / no alerting / alerting on every retry
  Sensors that poll forever
  Manual fixes in the UI; drift from code
  Mega-DAGs covering unrelated concerns

RULE   Orchestration is plumbing. Make it boring: code-defined,
       idempotent, observable, owned. Save the cleverness for
       the transforms.
```

---

## 13. Resources

### Documentation
- **Apache Airflow** — <https://airflow.apache.org/docs/>
- **Dagster** — <https://docs.dagster.io>
- **Prefect** — <https://docs.prefect.io>
- **Mage** — <https://docs.mage.ai>
- **Argo Workflows** — <https://argo-workflows.readthedocs.io>
- **Flyte** — <https://docs.flyte.org>
- **dbt** — <https://docs.getdbt.com>
- **AWS Step Functions** — <https://docs.aws.amazon.com/step-functions/>
- **Temporal** — <https://docs.temporal.io>

### Books
- *Data Pipelines Pocket Reference* — James Densmore.
- *Fundamentals of Data Engineering* — Reis & Housley.
- *The Data Engineering Cookbook* — Andreas Kretz (free PDF).

### Articles
- "Functional data engineering" — Maxime Beauchemin (Airflow's creator): <https://medium.com/@maximebeauchemin/functional-data-engineering-a-modern-paradigm-for-batch-data-processing-2327ec32c42a>
- "Software-defined assets" — Dagster blog.
- "Why Prefect 2" — Prefect engineering.
- "Airflow at Airbnb" / "Dagster at Eventbrite" / "Prefect at SciPy" — user stories.

### Videos
- *Airflow Summit*, *Dagster Day*, *Prefect Halloween* — annual talks.
- ByteByteGo — "Workflow Orchestration Explained."
- Maxime Beauchemin talks on functional data engineering.

### Tools
- **Airflow**: Astronomer (Astro), AWS MWAA, GCP Cloud Composer.
- **Dagster Cloud**, **Prefect Cloud**.
- **Hightouch / Census** (reverse ETL).
- **Great Expectations / Soda / Monte Carlo** (data quality).
- **OpenLineage / Marquez** (cross-tool lineage).

### Adjacent reading
- [MapReduce →](./mapreduce.md)
- [Hadoop Ecosystem →](./hadoop.md)
- [Apache Spark →](./spark.md)
- [Apache Flink →](./flink.md)
- [ETL vs ELT →](./etl-vs-elt.md)
- [Data Modeling for Analytics →](./dimensional-modeling.md)
- [Change Data Capture →](../04-databases/cdc.md)
- [Lakehouse Architecture →](../04-databases/lakehouse.md)
- [CI/CD Pipelines →](../15-deployment/cicd.md)
- [Idempotency →](../03-apis/idempotency.md)
- [SLA, SLO, SLI →](../11-reliability/sla-slo-sli.md)
- [Retry, Timeout, and Exponential Backoff →](../11-reliability/retry-timeout-backoff.md)

---

*Previous:* [← ETL vs ELT](./etl-vs-elt.md)  |  *Next:* [Data Modeling for Analytics →](./dimensional-modeling.md)
