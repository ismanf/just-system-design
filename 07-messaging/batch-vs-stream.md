# Batch vs Stream Processing

> **TL;DR** — Two ways to process data. **Batch** processes a **bounded set** at scheduled intervals (every night, every hour): higher latency, simpler operationally, easy retries, optimized throughput, mature tooling. **Stream** processes an **unbounded flow** as it arrives: low latency, continuous, sophisticated semantics (event time, watermarks, exactly-once), more complex to operate. The classical view said batch for analytics, stream for real-time. The modern view (Kappa architecture, Beam, Flink's unified API) says **streams subsume batches** — a batch is just a finite stream. In practice, most companies run both: batch for backfills, training data, large complex joins, financial reports; streams for dashboards, alerts, derived data, real-time features. The trade-off you balance: **latency vs simplicity vs cost**.

---

## 1. The Two Models

```
BATCH
   data accumulates                     job runs on schedule
   ┌────────────────────┐               ┌────────────┐
   │  yesterday's data  │ ─────────►   │  job       │ ──► output
   └────────────────────┘               │  (Spark,   │
   (bounded; well-known size)           │   Airflow) │
                                        └────────────┘
                                        hours later

STREAM
   data arrives                         processor runs continuously
   event ──► stream operator ──► output (seconds later)
   event ──►                  ──► output
   event ──►                  ──► output
   (unbounded; never finishes)
```

The defining axes:

| Axis | Batch | Stream |
|---|---|---|
| Data bound | Bounded (a chunk) | Unbounded (continuous) |
| Latency | Hours to days | Seconds |
| Compute pattern | Periodic | Continuous |
| Failure recovery | Restart the job | Checkpoint + replay |
| State | Loaded fresh | Maintained across events |
| Complexity | Simpler | More moving pieces |
| Cost | Idle outside windows | Always-on |

---

## 2. When Batch Wins

Batch processing is great when:

### 2.1 Latency is not critical
Daily reports, weekly billing, monthly invoices, ML training data. Hours of latency is fine.

### 2.2 The job is complex
Multi-stage joins across many large tables. Easier as one Spark/SQL job over snapshots than as a streaming pipeline.

### 2.3 You want determinism and easy reruns
Bug in yesterday's report? Just rerun the batch. No state to worry about; the output is a pure function of the input.

### 2.4 Storage and compute are cheap, full-time isn't
Run a 1000-node Spark cluster for 2 hours nightly. Cheaper than running 100 nodes continuously.

### 2.5 The data is fundamentally batched
Daily exports from a third party. Weekly partner files. Quarterly reports. The input itself is batch.

### 2.6 Compliance / audit
You can hash the exact dataset processed. Reruns produce identical results. Auditors love this.

---

## 3. When Stream Wins

Stream processing is great when:

### 3.1 Latency matters
Fraud detection, alerts, dashboards, recommendations. Seconds matter.

### 3.2 Continuous derived data
Materialized views, search indexes, caches kept up-to-date.

### 3.3 The input is naturally a stream
Clicks, log lines, IoT sensors, transactions. They arrive continuously.

### 3.4 Stateful continuous computation
Sessionization, anomaly detection over rolling windows, real-time aggregations.

### 3.5 Capacity-smoothing
Batch jobs cause spikes (idle then load); streams smooth load over time.

### 3.6 Event-driven systems
The data IS events; the processing should be event-driven too.

---

## 4. The Lambda Architecture (Historical)

Pre-2015 pattern: maintain **both** batch and streaming pipelines.

```
   input ──► batch layer ──► batch view (yesterday's accurate truth)
         ──► speed layer ──► realtime view (today's approximation)
                                                  │
                                              query layer
                                              (merge views)
```

- **Batch layer**: Hadoop / Spark / Hive. Processes everything; produces the canonical answer.
- **Speed layer**: Storm / Spark Streaming. Provides a rough answer for the most recent data not yet in batch.
- **Query layer**: combines the two.

Pros: batch's accuracy + stream's freshness.

Cons: **two codebases for the same logic** — every change must be made twice; results diverge; debugging hell.

Most teams that built Lambda regretted it. Hence Kappa.

---

## 5. The Kappa Architecture (Modern)

Jay Kreps' 2014 alternative: **stream-only**. Use a stream processing engine for both real-time and historical (by replaying the event log).

```
   input → Kafka (durable log)
              │
              ▼
            stream processor (Flink / Kafka Streams)
              │
              ▼
            output (cache, DB, dashboards)

   want to recompute? rewind Kafka and replay.
```

- One codebase.
- Historical recompute = replay from offset 0.
- Live processing = tail current offset.

Modern Kappa works because:
- Engines (Flink, Spark Structured Streaming) handle both bounded and unbounded.
- Kafka retains long enough (days, weeks, sometimes forever).
- Replay is a streaming run with the same code.

This is what most companies are converging to.

---

## 6. The Beam / Unified Model

**Apache Beam** (and Google's Dataflow) take this further: write your pipeline once; run it on batch or stream engines.

```python
# Beam pipeline (Python)
events
  | beam.WindowInto(beam.window.FixedWindows(60))
  | beam.Map(parse)
  | beam.GroupByKey()
  | beam.CombinePerKey(sum)
  | beam.io.WriteToBigQuery(...)
```

The same code runs on:
- Google Dataflow (batch + streaming).
- Apache Flink runner.
- Apache Spark runner.
- Bounded source = batch behavior; unbounded source = streaming.

This makes the "batch vs stream" choice **a runtime decision, not a code decision**.

---

## 7. Engines That Do Both

| Engine | Strength |
|---|---|
| **Apache Flink** | Streaming-first; batch is "bounded streaming." Excellent for both. |
| **Apache Spark** | Batch-first; streaming via Structured Streaming (micro-batch). |
| **Apache Beam** | Engine-agnostic; runs on top of Flink, Spark, Dataflow. |
| **Google Dataflow** | Managed Beam; truly unified. |
| **Kafka Streams** | Streaming-only. |
| **Apache Hadoop / MapReduce** | Batch-only; legacy. |
| **dbt** | SQL-based batch transformations; warehouse-centric. |
| **Snowflake / BigQuery** | Batch SQL on the warehouse; streaming inserts increasingly common. |

In 2026, most green-field choices:
- **Pure batch**: Spark, dbt, BigQuery, Snowflake, EMR.
- **Pure stream**: Flink, Kafka Streams.
- **Both**: Flink (lean streaming first), Beam/Dataflow (cloud-agnostic unified).

---

## 8. Comparison Across Concrete Dimensions

| Dimension | Batch | Stream |
|---|---|---|
| Typical latency | 1 hour – 1 day | 100 ms – 5 sec |
| Throughput | Very high (TB/hour) | High (millions/sec) |
| Failure recovery | Rerun job | Checkpoint replay |
| Idempotency | Trivial (re-derive from input) | Needs care (exactly-once or idempotent sinks) |
| Schema evolution | Easy (process old data with new code) | Trickier (state migrations) |
| Operational complexity | Schedule, monitor jobs | Always-on infrastructure |
| Backfills / reprocessing | Native; "rerun" | Replay event log |
| Cost model | Burst (job runs, then idle) | Continuous |
| Determinism | Strong | Weaker (event ordering, lateness) |
| Tooling maturity | Decades | Mature recently (2015+) |
| Joins | Easy (full data available) | Harder (windowed) |
| Late events | Trivially handled (next run) | Watermarks + allowed lateness |

---

## 9. Patterns That Combine Both

Real systems often blend:

### 9.1 Stream for freshness, batch for accuracy
- Stream pipeline emits "good enough" real-time numbers.
- Nightly batch reconciliation produces the canonical version.
- Used for billing, financial reporting where audit demands precision.

### 9.2 Stream for derived stores, batch for ML
- Stream populates Redis / search / dashboards.
- Batch generates ML training data, runs model retraining.

### 9.3 Stream for ops, batch for analytics
- Real-time alerting and operational dashboards from streams.
- BI queries hit a warehouse populated by batch ETL.

### 9.4 Stream for ingestion, batch for transformation
- Stream lands raw events into S3 / data lake.
- Batch (Spark, dbt) does heavy transformation later.

### 9.5 Kappa for everything
- One stream pipeline.
- Replay for backfills.
- Modern preference where the stack and team support it.

---

## 10. Worked Example: A SaaS Analytics Pipeline

Customer events → analytics dashboards + ML features + reports.

### Approach A: Lambda
- **Stream**: Flink processes events → Redis dashboards (live).
- **Batch**: Spark nightly → BigQuery for canonical analytics.
- **ML**: Batch joins clean batch tables.
- Duplicated logic; eventually painful.

### Approach B: Kappa
- **Stream**: Kafka → Flink for everything.
  - Real-time output → Redis.
  - Historical output → BigQuery via Flink sink (batched commits).
  - ML features → Flink → feature store.
- One codebase.
- Reprocess by replaying Kafka.

### Approach C: Modern compromise
- Stream → Redis + ClickHouse (real-time).
- dbt + BigQuery for SQL-based batch transforms over the same events landed by stream.
- Both reasonable; team chooses based on SQL vs code preference.

### Approach D: Streaming-first warehouse
- Kafka → Snowflake / BigQuery streaming inserts.
- Materialized views / Tabular streaming SQL transforms.
- Combines stream ingestion with batch-style SQL.

For most teams in 2026, Approach C or D is the sweet spot.

---

## 11. Cost Considerations

### Batch cost shape
- Pay for compute during the job. Cluster scales to zero between.
- Good for unpredictable / spiky workloads.
- Snowflake, BigQuery: per-query pricing.

### Stream cost shape
- Pay for always-on workers.
- Need to size for peak.
- Network costs (cross-AZ) accumulate continuously.

In rough numbers: stream is often 1.5–3× more expensive than equivalent batch *for the same volume*. The trade is paid in latency.

If the use case doesn't need real-time, stream is wasteful. If it does, batch is unusable.

---

## 12. Migration Stories

### "We moved batch ETL to streams"
- **Reason**: dashboards needed minute-level freshness; nightly reports were too stale.
- **Pain**: training the team on Flink; handling late events; checkpoint storage.
- **Win**: dashboards 60× fresher.

### "We moved streams back to batch"
- **Reason**: real-time wasn't actually needed; ops cost was eating margin.
- **Pain**: rebuilding pipelines.
- **Win**: half the operational burden.

### "We adopted Kappa and stopped maintaining Lambda"
- **Reason**: code-once was the dream and now real (Beam / Flink unified).
- **Pain**: one team learns deep streaming engine.
- **Win**: half the codebase to maintain.

The honest pattern: **don't move to streams just because they're cool**. Move when the latency or cost of batch is unacceptable.

---

## 13. Common Mistakes

- **Streaming everything for the sake of it.** Real-time costs money; many use cases tolerate hours.
- **Lambda architecture by default.** Two pipelines for the same logic. Most teams regret.
- **Treating stream output as canonical for compliance.** Reconcile with batch if regulation demands.
- **Not designing batch jobs to be idempotent.** Rerun causes duplicates / inconsistencies.
- **Streaming with no replay path.** Recompute requires manual heroics.
- **Choosing stream engine without team skill.** Flink is powerful but unforgiving.
- **Picking batch when the requirement is real-time.** "We can run it every 5 minutes" — no, that's a stream.
- **Not handling late events in either.** Batch reruns next day; streams need watermarks. Same problem in different clothes.
- **Heavy batch SLAs on slow source data.** "Hourly ETL" but the source produces hourly partitions 90 minutes late.
- **No SLO per pipeline.** "It should be fast" is not a target.

---

## 14. Choosing in Practice

```
Need fresh dashboards / alerts in seconds?
  → Stream.

Daily/weekly reports, complex joins across snapshots?
  → Batch.

ML training data, feature snapshots?
  → Batch (with streaming for online features).

Derived stores (search index, cache, materialized views)?
  → Stream.

Financial reconciliation, audit trail?
  → Batch (with stream for live monitoring).

CDC pipeline?
  → Stream.

You want to choose once and forget?
  → Beam / Flink unified.

Latency requirement < 1 min?
  → Stream.
Latency requirement > 1 hour?
  → Batch.
Between?
  → Either; cost wins.
```

---

## 15. Cheat Card

```
BATCH         bounded input, scheduled, high throughput, easy reruns
              tools: Spark, Hadoop, dbt, BigQuery, Snowflake, Airflow

STREAM        unbounded input, continuous, low latency, stateful
              tools: Flink, Kafka Streams, Spark Streaming, Beam

LATENCY       batch: hours-days     stream: sub-second to seconds
COST          batch: bursty cheap   stream: always-on
COMPLEXITY    batch: low            stream: medium-high
DETERMINISM   batch: strong         stream: weaker (events, lateness)

LAMBDA        batch + stream for accuracy + freshness
              two codebases; mostly an anti-pattern now

KAPPA         stream-only; replay event log for reprocess
              modern preference where supported

UNIFIED       Beam, Flink unified, Dataflow:
              same code, batch or stream at runtime

CHOOSE
  freshness < 1 min        → stream
  freshness > 1 hour        → batch
  in between                → cheaper / team skill wins
  complex joins, simple ops → batch
  derived stores, alerts    → stream

PITFALLS      streaming everything; Lambda by default;
              no idempotent batch; no replay-able stream;
              choosing without latency SLO

RULE          Latency requirement decides.
              Then cost. Then team skill.
```

---

## 16. Resources

### Books
- *Streaming Systems* — Akidau, Chernyak, Lax (the unified theory).
- *Hadoop: The Definitive Guide* — Tom White (batch foundations).
- *Designing Data-Intensive Applications* — Kleppmann (batch/stream chapters).
- *Spark: The Definitive Guide* — Chambers, Zaharia.

### Articles
- "Questioning the Lambda Architecture" — Jay Kreps (the Kappa origin).
- "The world beyond batch: Streaming 101 / 102" — Tyler Akidau (foundational).
- "The unbundling of the database" — Martin Kleppmann.

### Documentation
- **Apache Beam programming guide**: <https://beam.apache.org/documentation/programming-guide/>
- **Flink: Batch and Streaming Unified**: <https://flink.apache.org/>
- **Google Dataflow**: <https://cloud.google.com/dataflow>

### Videos
- Tyler Akidau — Streaming 101 / 102.
- Jay Kreps — Kappa Architecture.
- ApacheCon Flink keynotes.

### Tools
- Batch: Spark, Hadoop, EMR, dbt, Airflow, Dagster, Prefect, BigQuery, Snowflake.
- Stream: Flink, Kafka Streams, Spark Streaming, ksqlDB, Beam, Dataflow.

### Adjacent reading
- [Stream Processing →](./stream-processing.md)
- [Event-Driven Architecture →](./event-driven-architecture.md)
- [Kafka Deep Dive →](./kafka.md)
- [Lambda vs Kappa Architecture →](../14-architecture/lambda-kappa.md)
- [Data Pipelines & Orchestration →](../17-big-data/data-pipelines.md)
- [ETL vs ELT →](../17-big-data/etl-vs-elt.md)
- [Apache Spark →](../17-big-data/spark.md)
- [Apache Flink →](../17-big-data/flink.md)

---

*Previous:* [← Stream Processing](./stream-processing.md)  |  *Next:* [Saga Pattern →](./saga-pattern.md)
