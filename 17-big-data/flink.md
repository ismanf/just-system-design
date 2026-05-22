# Apache Flink

> **TL;DR** — **Apache Flink** is a distributed stream-processing engine designed around the philosophy that **streaming is the foundational primitive, and batch is just a bounded stream**. Its defining features: **true event-at-a-time processing** (no micro-batches), **strong exactly-once guarantees** via aligned checkpoints, **event-time semantics** with watermarks, and a **stateful operator model** that scales to terabytes of in-flight state. Born at TU Berlin / Stratosphere, productionized by data Artisans (now Ververica), Flink became the dominant choice for **sub-second, stateful streaming** at companies like Netflix, Uber, Stripe, Pinterest, and Alibaba. The honest take: **if your problem is "I need millisecond-latency processing of an unbounded stream with correctness under failure," Flink is the right answer**. If it's "I have a daily batch job," use Spark. If it's "I want to query streams with SQL and a low operational footprint," consider Kafka Streams or ksqlDB first. Flink is more capable than its alternatives — and operationally heavier.

---

## 1. The big picture

Flink turns the standard "batch first, stream as an afterthought" mental model upside down:

```
                    ┌────────────────────────────┐
                    │   Unbounded streams         │
                    │   (Kafka, Pulsar, Kinesis)  │
                    └─────────────┬───────────────┘
                                  │
                                  ▼
                ┌──────────────────────────────────┐
                │           Flink                  │
                │   stateful operators in a DAG    │
                │   event-time windows             │
                │   exactly-once via checkpoints   │
                └──────────────────┬───────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              ▼                    ▼                    ▼
        ┌─────────┐         ┌──────────┐         ┌───────────┐
        │  Sinks  │         │  Sinks   │         │  Sinks    │
        │ Kafka   │         │ JDBC     │         │ Iceberg / │
        │ Iceberg │         │ Cassandra│         │ Delta     │
        └─────────┘         └──────────┘         └───────────┘
```

A Flink **job** is a **dataflow graph** of operators (sources → transformations → sinks). Each operator runs in parallel across slots in the cluster. State (counters, aggregates, joins, windows) lives in **embedded state backends** (RocksDB on local disk, with periodic snapshots to durable storage).

The whole picture is: **events flow continuously through a graph of stateful operators, with consistent snapshots taken in the background to make failures recoverable.**

---

## 2. Why Flink exists

The streaming systems that came before — Storm, Spark Streaming (micro-batch), Samza, Kafka Streams — each made trade-offs that left a gap:

- **Storm**: low latency, at-least-once semantics. State management was bolted on. No event-time.
- **Spark Streaming (DStreams)**: micro-batches mean latency floor of seconds. Event-time support was retrofitted.
- **Samza / Kafka Streams**: simple, library-style, partition-aligned. Hard to scale state beyond what RocksDB on a single node can hold.

Flink filled the gap with:

- **Continuous (per-event) processing** — milliseconds end-to-end, not seconds.
- **First-class event-time** semantics with **watermarks**.
- **Exactly-once** across sources, operators, and sinks (when the sink supports it).
- **Large stateful operators** — gigabytes to terabytes per operator instance.
- **Same engine for batch** — bounded streams reuse the streaming runtime.

The trade-off: more concepts (event-time vs processing-time, watermarks, checkpoints, savepoints, state TTL, exactly-once sinks) and more operational complexity than Kafka Streams or simple Spark batch jobs.

---

## 3. Event time, processing time, watermarks

This trio is the single biggest mental shift Flink demands.

### Three notions of time

- **Event time** — when the event actually happened, embedded in the event payload.
- **Ingestion time** — when Flink first saw the event.
- **Processing time** — wall clock on the operator.

You almost always want **event time** for correctness — late, out-of-order, replayed events all produce the same result. Processing time is only for non-correctness-sensitive things (UI dashboards, "now" calculations).

### The out-of-order problem

Networks reorder events. Mobile clients buffer offline and send batches. Kafka partitions interleave. The same minute of "real" data arrives over minutes or hours of wall-clock.

```
real time:       :00 :01 :02 :03 :04 :05
events arrive:    e1 e3 e0 e5 e2 e4   ← jumbled
event time:       :00 :02 :00 :04 :01 :03
```

If a windowed aggregation closes the `:00–:01` window at processing-time `:01`, it misses events whose **event time** falls in that window but arrived later.

### Watermarks

A **watermark** is a promise from the source: "no future event will have event-time earlier than W." Operators use watermarks to decide when it's safe to close a window.

A typical watermark generator: "current max observed event-time minus 5 seconds." That allows up to 5 seconds of out-of-orderness; anything later is *late data*.

```
events arriving with watermark advancing:
  e1 (t=:00, wm=:00)
  e3 (t=:02, wm=:02)
  e0 (t=:00, wm=:02)   ← late by 2s, still within bound
  ...
```

Late data policies:
- **Drop** — anything past watermark is gone.
- **Side output** — send late events to a separate stream for inspection.
- **Allowed lateness** — extend window close by N more seconds; updates flow downstream.

Choosing watermark lateness is a tradeoff between latency (close windows fast) and completeness (wait for stragglers). 5–30 seconds is common; minutes for mobile or batch-delivered data.

---

## 4. Stateful operators

Stateful processing is Flink's superpower. Every operator can keep arbitrary keyed state — counters, aggregates, sessions, join tables, ML features.

```java
public class FraudDetector
        extends KeyedProcessFunction<String, Transaction, Alert> {

    private transient ValueState<Double> lastAmount;
    private transient ValueState<Long> lastTimestamp;

    @Override
    public void open(Configuration cfg) {
        lastAmount = getRuntimeContext().getState(
            new ValueStateDescriptor<>("lastAmount", Double.class));
        lastTimestamp = getRuntimeContext().getState(
            new ValueStateDescriptor<>("lastTimestamp", Long.class));
    }

    @Override
    public void processElement(Transaction tx, Context ctx,
                               Collector<Alert> out) throws Exception {
        Double prev = lastAmount.value();
        if (prev != null && tx.amount > prev * 10) {
            out.collect(new Alert(tx.cardId, "Large jump", tx));
        }
        lastAmount.update(tx.amount);
        lastTimestamp.update(tx.eventTime);
    }
}
```

Each card sees its own `lastAmount`. State is keyed by `cardId` (via `keyBy`). The state is partitioned across the cluster; events for the same key always go to the same operator instance.

State backends:
- **HashMap state backend** — JVM heap. Fast, limited to RAM size.
- **RocksDB state backend** — on-disk LSM-tree, snapshots to durable storage. Scales to terabytes per operator instance. The default for production.

State features:
- **TTL** — auto-expire stale state.
- **Broadcast state** — replicate small data (rules, configs) to every operator instance.
- **State migration** — savepoints let you redeploy with code changes while keeping state.

The state-first design is what makes Flink uniquely good for fraud detection, real-time recommendations, session windows, complex event processing, and feature stores.

---

## 5. Checkpoints, savepoints, and exactly-once

### Checkpoints

Flink's **distributed snapshot algorithm** (Chandy-Lamport) periodically takes a consistent snapshot of all operator state across the cluster.

How it works:
- The coordinator injects a **barrier** into every source stream at a chosen time.
- Each operator, when it sees the barrier, writes its state to durable storage (S3, HDFS).
- The barrier flows downstream; when every operator has finished its checkpoint, the snapshot is complete.

On failure, Flink rolls back to the last successful checkpoint, replays from sources (Kafka offsets included), and continues. Inputs and outputs end up exactly-once *for sinks that participate* in the protocol.

Checkpoints are incremental for RocksDB — only changed SST files ship to S3. Typical frequency: every 30s–5min depending on state size and latency target.

### Savepoints

A **savepoint** is a manually triggered, durable snapshot used for upgrades, code changes, and migrations. Versioned, tagged, never auto-cleaned.

Workflow: take savepoint → stop job → deploy new version of code → start from savepoint. State carries over even when operator graphs change (within compatibility rules).

This is the operational killer feature for streaming: you can upgrade a stateful streaming job without losing state or restarting from scratch.

### Exactly-once sinks

Exactly-once requires sink cooperation. Flink supports:

- **Kafka with transactional producers** (two-phase commit).
- **Iceberg / Delta Lake** (atomic table commits aligned to checkpoints).
- **JDBC with idempotent upserts** (best-effort exactly-once).
- **Files with `_SUCCESS` markers** (rename-only commit).
- **Cassandra with idempotent writes**.

Sinks without transaction support can only achieve at-least-once + idempotent application logic.

The general rule: **exactly-once is end-to-end, not just at Flink**. The sink, the source, and Flink's checkpoint coordinator all participate.

---

## 6. Windows

Windowing breaks unbounded streams into bounded chunks for aggregation.

| Window type | Definition | Example |
|---|---|---|
| **Tumbling** | Fixed-size, non-overlapping | "1-minute counts" |
| **Sliding** | Fixed-size, overlapping by slide interval | "5-minute count, sliding 1 minute" |
| **Session** | Period of activity separated by gaps of inactivity | "user sessions with 30-minute gap" |
| **Global** | All events in one infinite window (custom triggers) | "anytime a threshold is hit, emit" |

```java
stream
  .keyBy(Event::userId)
  .window(TumblingEventTimeWindows.of(Time.minutes(5)))
  .aggregate(new SumAmount())
```

Windows can be:
- **Event-time** (default and right for almost everything).
- **Processing-time** (fast, less correct).
- **Custom triggers** for early or late firings.

Session windows are particularly powerful — defining a "session" as "events with no more than 30-minute gap" lets you model user sessions, ride durations, gaming sessions, etc. with one operator.

---

## 7. Architecture and execution

A Flink cluster has two main components:

- **JobManager** — accepts jobs, plans execution, coordinates checkpoints, schedules tasks. HA via ZooKeeper or Kubernetes.
- **TaskManager** — runs tasks. Each TaskManager has N **slots** (parallelism units), each holding part of the operator pipeline.

```
Job = StreamGraph → JobGraph → ExecutionGraph
                                     │
            ┌────────────────────────┼────────────────────────┐
            ▼                        ▼                        ▼
    ┌──────────────┐         ┌──────────────┐         ┌──────────────┐
    │ TaskManager  │         │ TaskManager  │         │ TaskManager  │
    │ ┌─┬─┬─┬─┐    │         │ ┌─┬─┬─┬─┐    │         │ ┌─┬─┬─┬─┐    │
    │ │S│S│S│S│    │         │ │S│S│S│S│    │         │ │S│S│S│S│    │
    │ └─┴─┴─┴─┘    │         │ └─┴─┴─┴─┘    │         │ └─┴─┴─┴─┘    │
    └──────────────┘         └──────────────┘         └──────────────┘
       slots run operator subtasks; one keyed stream → one slot per partition
```

Deployment modes:
- **Standalone** — fine for dev.
- **YARN** — legacy Hadoop estates.
- **Kubernetes** — the modern default. Operators like the Flink Kubernetes Operator simplify lifecycle.
- **Managed**: AWS Kinesis Data Analytics (now Managed Service for Apache Flink), Ververica Platform, Aiven for Apache Flink, Confluent Flink, Decodable.

For new builds: Flink on Kubernetes or a managed service. The Flink Kubernetes Operator (Apache) handles deployments, savepoints, and upgrades declaratively.

---

## 8. Flink APIs

Flink layers its API like Spark:

| API | Level | Use |
|---|---|---|
| **DataStream API** | Low | Custom stateful operators, complex event processing |
| **Table API** | Medium | Declarative table-style transforms |
| **Flink SQL** | High | SQL on streams and tables |
| **Stateful Functions** (separate framework) | Function-as-a-service | Event-driven app model |

For 90% of users, **Flink SQL + Table API** is the right choice. It's:

- Declarative — engine picks the optimal execution.
- Same engine for batch (bounded) and streaming (unbounded).
- Catalog-aware — reads Iceberg, Hive, JDBC, Kafka with one consistent SQL surface.
- Cheaper to maintain than Java/Scala DataStream code.

Drop to DataStream when:
- You need custom keyed state with complex semantics.
- You're writing complex event processing.
- You need fine control over checkpointing, side outputs, broadcast state.

---

## 9. Worked example — real-time fraud scoring

A typical Flink job:

```sql
CREATE TABLE transactions (
  tx_id STRING,
  card_id STRING,
  amount DOUBLE,
  merchant STRING,
  country STRING,
  event_time TIMESTAMP(3),
  WATERMARK FOR event_time AS event_time - INTERVAL '10' SECOND
) WITH (
  'connector' = 'kafka',
  'topic' = 'transactions',
  'format' = 'avro-confluent',
  ...
);

CREATE TABLE alerts (
  card_id STRING,
  reason STRING,
  window_start TIMESTAMP(3),
  count_in_window BIGINT
) WITH (
  'connector' = 'kafka',
  'topic' = 'fraud-alerts',
  'format' = 'json',
  ...
);

INSERT INTO alerts
SELECT
  card_id,
  'velocity' AS reason,
  TUMBLE_START(event_time, INTERVAL '1' MINUTE) AS window_start,
  COUNT(*) AS count_in_window
FROM transactions
GROUP BY card_id, TUMBLE(event_time, INTERVAL '1' MINUTE)
HAVING COUNT(*) > 10;
```

Properties of this job:
- Reads from Kafka with exactly-once semantics.
- Event-time windowing with 10-second watermark tolerance.
- Writes to a Kafka alerts topic.
- Restartable from checkpoint; survives task failures with no data loss.

Three lines of declarative SQL replace what would be hundreds of lines of Storm or hand-rolled Kafka consumer code.

---

## 10. Flink vs Spark vs Kafka Streams

| | Flink | Spark Structured Streaming | Kafka Streams |
|---|---|---|---|
| Latency floor | Milliseconds | Seconds (micro-batch) | Milliseconds |
| Throughput | Very high | High | High |
| Stateful operators | TB-scale, RocksDB | GB-scale, micro-batch | Partition-bound RocksDB |
| Exactly-once | Excellent (broad sink support) | Good (some sinks) | Excellent (within Kafka) |
| Event-time + watermarks | First-class | Supported, second-class feel | First-class |
| Batch unification | Yes (bounded streams) | Yes (DataFrames) | No (streaming only) |
| Ops complexity | High | Medium | Low (library) |
| Best for | Sub-second stateful streaming, large state | Batch ETL + seconds-level streaming | App-embedded streaming inside a Kafka-only world |

If you're in a Kafka-only world and your latency / state needs are modest, **Kafka Streams** is the cheapest operationally. For batch + seconds-level streaming, **Spark**. For everything else streaming, **Flink**.

See [Stream Processing →](../07-messaging/stream-processing.md), [Batch vs Stream Processing →](../07-messaging/batch-vs-stream.md).

---

## 11. Who actually uses Flink

A representative sample:

- **Netflix** — real-time analytics, content recommendation features, fraud, video metadata pipelines.
- **Uber** — pricing, ETAs, fraud detection (AthenaX is their Flink SQL layer).
- **Stripe** — radar fraud features, real-time aggregations.
- **Pinterest** — homepage feed signals, ads.
- **Alibaba** — Singles' Day real-time dashboards; a massive Flink user, also a major contributor (Blink → merged into Flink).
- **ING / Comcast / Yelp** — banking, telecom, restaurant data.

Flink jobs in production frequently process millions of events per second per cluster with millisecond latencies and gigabytes-to-terabytes of state per job.

---

## 12. Common Mistakes / Anti-Patterns

- **Processing-time everywhere.** Convenient, incorrect under any reordering. Use event-time for correctness-sensitive jobs.
- **No watermark.** Windows never close; state grows; job stalls. Always define watermarks.
- **Watermark too tight.** Drops legitimate late data; results are wrong.
- **Watermark too loose.** Windows close late; results are stale.
- **Unbounded state without TTL.** State grows forever; checkpoints get huge.
- **Hash-keyed operators without `keyBy`.** No partitioning, no parallel state; everything funnels.
- **Heap state backend on a job with TB of state.** OOM. Use RocksDB.
- **No savepoint discipline.** Every restart loses state. Take savepoints before every redeploy.
- **Treating checkpoints as backups.** They're not — they're consistent snapshots for failure recovery. Take savepoints for retention.
- **Non-idempotent sinks without transactional support.** At-least-once arrives as "duplicated events."
- **Kafka source with `auto.offset.reset=latest` in production.** Lose data on restart. Use committed offsets via Flink checkpoint.
- **Flink job state living on TaskManager-local disk with no S3 snapshot.** Node dies → state gone. Configure remote checkpoint storage.
- **One job processing many keyed streams that should be separate jobs.** Resource contention; complicated savepoint surgery on changes.
- **Mismatched parallelism between operators causing skew.** Tune per-operator parallelism for hot stages.
- **Treating Flink SQL like batch SQL.** Streaming SQL has semantic differences — append vs upsert vs retract streams. Learn them.
- **No metrics or backpressure visibility.** Flink dashboard shows it; ignoring it means surprises in production.
- **Trying to do everything in one mega-job.** Smaller, focused jobs are easier to operate.

---

## 13. Cheat Card

```
PURPOSE   Sub-second stateful streaming over unbounded data,
          with event-time correctness and exactly-once semantics.

CORE CONCEPTS
  Event time   when the event happened
  Watermark    "no future event earlier than W"
  Stateful op  keyed RocksDB state per operator instance
  Checkpoint   consistent distributed snapshot (Chandy-Lamport)
  Savepoint    manual durable snapshot for upgrades

EXECUTION
  JobManager (HA) + TaskManagers with slots
  StreamGraph → JobGraph → ExecutionGraph
  Per-operator parallelism

STATE BACKENDS
  HashMap   on-heap, fast, RAM-bound
  RocksDB   on-disk LSM, snapshots to S3 — production default

WINDOWS
  Tumbling   non-overlapping fixed
  Sliding    overlapping fixed
  Session    activity-then-gap
  Global     custom triggers

API LAYERS
  DataStream    custom operators, full power
  Table API     declarative, batch-like
  Flink SQL     SQL on streams and tables
  Stateful Fns  function-as-a-service style

EXACTLY-ONCE END-TO-END
  Source with replay (Kafka offsets, files)
  Operators participate in checkpoint
  Sink with txn / idempotent commit (Kafka, Iceberg, Delta, JDBC upsert)

WHEN FLINK WINS
  Millisecond latency
  Large keyed state (GB–TB per operator)
  Strict exactly-once across heterogeneous sinks
  Complex event processing, fraud, sessionization, joins
  Unified batch + streaming via Flink SQL

WHEN IT LOSES
  Pure batch ETL → Spark
  Daily aggregations → batch SQL
  Tiny Kafka-only apps → Kafka Streams
  Interactive sub-second SQL → ClickHouse / Druid / Pinot

PITFALLS
  Processing-time when event-time was needed
  No watermark / wrong watermark
  Unbounded state without TTL
  Heap state for terabyte-scale jobs
  No savepoint discipline
  Non-transactional sinks
  Auto-offset-reset latest in production
  Local-only checkpoint storage

RULE   Streaming first, batch as a special case. Event-time
       always. State with TTL always. Savepoints before any
       redeploy.
```

---

## 14. Resources

### Documentation
- **Apache Flink** — <https://flink.apache.org/docs/stable/>
- **Flink SQL** — <https://nightlies.apache.org/flink/flink-docs-stable/docs/dev/table/sql/overview/>
- **Flink Kubernetes Operator** — <https://nightlies.apache.org/flink/flink-kubernetes-operator-docs-stable/>
- **Iceberg + Flink** — <https://iceberg.apache.org/docs/latest/flink/>
- **Confluent Flink** — <https://docs.confluent.io/cloud/current/flink/>

### Papers
- "Apache Flink: Stream and Batch Processing in a Single Engine" — Carbone et al. (2015).
- "Lightweight Asynchronous Snapshots for Distributed Dataflows" — Carbone et al. (the Flink checkpoint paper).
- "The Dataflow Model" — Akidau et al. (Google, 2015). The watermark-and-trigger model that Flink (and Beam) implements.

### Books
- *Stream Processing with Apache Flink* — Hueske & Kalavri. The canonical book.
- *Designing Data-Intensive Applications* — Martin Kleppmann. The chapter on stream processing is worth reading twice.

### Articles
- "Streaming 101 & 102" — Tyler Akidau on O'Reilly: <https://www.oreilly.com/radar/the-world-beyond-batch-streaming-101/>
- "Flink at Netflix" / "Flink at Uber" / "Flink at Stripe" — engineering blogs.
- "How Alibaba uses Flink" — Singles' Day-scale war stories.

### Videos
- *Flink Forward* — annual conference recordings.
- *Stream Processing with Apache Flink* — Kostas Kloudas talks.
- ByteByteGo — "Apache Flink Explained."

### Tools
- **Flink Kubernetes Operator** — declarative job management.
- **Ververica Platform**, **AWS Managed Service for Apache Flink**, **Confluent Flink**, **Aiven for Flink**, **Decodable** — managed offerings.
- **Apache Iceberg / Delta Lake** — exactly-once sinks for the lakehouse.
- **Apache Beam** — portable model that runs on Flink (among others).

### Adjacent reading
- [Apache Spark →](./spark.md)
- [Stream Processing →](../07-messaging/stream-processing.md)
- [Batch vs Stream Processing →](../07-messaging/batch-vs-stream.md)
- [Kafka Deep Dive →](../07-messaging/kafka.md)
- [Event-Driven Architecture →](../07-messaging/event-driven-architecture.md)
- [Event Sourcing →](../07-messaging/event-sourcing.md)
- [Delivery Guarantees →](../07-messaging/delivery-guarantees.md)
- [Data Pipelines & Orchestration →](./data-pipelines.md)
- [Lakehouse Architecture →](../04-databases/lakehouse.md)
- [Tail Latency & p99 →](../16-performance/tail-latency.md)

---

*Previous:* [← Apache Spark](./spark.md)  |  *Next:* [ETL vs ELT →](./etl-vs-elt.md)
