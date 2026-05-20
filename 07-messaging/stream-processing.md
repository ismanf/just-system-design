# Stream Processing (Kafka Streams, Flink, Spark Streaming)

> **TL;DR** — **Stream processing** is computing over **unbounded, continuous data** — events arriving in real time. Unlike batch (process all the data, then output), stream processing emits results **as data arrives**, with handling for **windowing**, **state**, **late events**, **out-of-order data**, and **fault tolerance**. The dominant engines: **Kafka Streams** (lightweight Java library, embedded in your service, runs on top of Kafka), **Apache Flink** (gold-standard for low-latency, stateful streaming with exactly-once and event-time semantics), **Apache Spark Streaming / Structured Streaming** (micro-batch model, integrates with the broader Spark ecosystem), and **ksqlDB** (SQL on Kafka, declarative). The hard problems are **event time vs processing time**, **watermarks**, **stateful operations** (joins, aggregations), **fault recovery** (checkpoints), and **exactly-once** semantics. Choose by latency requirements, statefulness, ecosystem fit, and team skill.

---

## 1. The Mental Shift: Stream vs Batch

```
BATCH (yesterday's data, computed nightly)
   ┌────────────────────────────────┐
   │     data sits in storage       │
   └───────────────┬────────────────┘
                   │
                   ▼ run a job; wait; output
              [aggregations]
                   │
                   ▼
                result


STREAM (data flows; output flows)
   events ──► [stream operator] ──► output
            (transform, aggregate, join)
                       │
                       state (windows, joins)
```

Stream processing **runs continuously**, processes one event at a time (or micro-batches), and emits results as they're ready. Latency from event to result is **seconds, not hours**.

Concretely:
- Batch: "compute yesterday's daily active users."
- Stream: "show DAU as it accumulates, updated every minute."

---

## 2. The Hard Problems

### 2.1 Event time vs processing time
- **Event time** — when the event happened in the real world.
- **Processing time** — when your operator processes it.

These differ. A user click at 2:00:00 may arrive in your pipeline at 2:00:05 (or 5 min later from a mobile retry). Aggregations done by processing time mis-bucket events. **Event-time processing is the correct way** for almost all analytics.

### 2.2 Out-of-order events
Events arrive late. A 2:00:00 click arriving at 2:05:30 belongs in the 2:00–2:01 window, but your window already closed.

### 2.3 Watermarks
A watermark is the system's estimate of "we've seen all events up to time T." When the watermark advances past a window's end, the window closes; results emit.

```
events:   2:00:00, 2:00:30, 2:00:05, 2:00:42 (late), ...
watermark advances:    2:00:30 → 2:01:00 → 2:02:00
                                              ↑
                                  close window 2:00–2:01
```

Late events arriving after the watermark either:
- Are dropped.
- Update previous results (recompute).
- Are sent to a side stream.

Each engine handles this differently. Flink is the most rigorous.

### 2.4 Stateful operations
- **Joins** between two streams (order events × payment events).
- **Aggregations** (count per minute, sum per user).
- **Sessions** (group events by user-activity-burst).

All require **state**, often gigabytes. The engine must:
- Store the state durably.
- Survive crashes (checkpoints).
- Scale state across nodes.

### 2.5 Exactly-once
A consumer-and-write-back pipeline that doesn't duplicate output on failure. See [Delivery Guarantees →](./delivery-guarantees.md). Flink and Kafka Streams support this; Spark Streaming approximates via micro-batch idempotency.

---

## 3. The Major Engines

### 3.1 Kafka Streams

A Java/Scala library. You run it as part of your service. State stored in local RocksDB + replicated via Kafka topics. Scales via partition assignment (like a consumer group).

```java
StreamsBuilder b = new StreamsBuilder();
KStream<String, Order> orders = b.stream("orders");
orders
    .groupBy((k, o) -> o.userId())
    .windowedBy(TimeWindows.of(Duration.ofMinutes(5)))
    .count()
    .toStream()
    .to("user-orders-5m");

new KafkaStreams(b.build(), config).start();
```

**Pros**
- No separate cluster; library in your app.
- Tight Kafka integration; exactly-once built in.
- Simple operational model.
- Stateful via RocksDB; state replicated to Kafka topics.

**Cons**
- JVM only (officially); confluent-kafka-streams in other languages is rare.
- Limited to Kafka input.
- No SQL DSL (use ksqlDB for SQL).
- Not great for huge state.

**When to use**
- You're already on Kafka.
- You want simple operations (no cluster).
- Per-service stream processing inside a microservice.
- Modest state (< 100 GB per pod).

### 3.2 Apache Flink

A full distributed stream-processing engine. Runs as its own cluster. **State-of-the-art for low-latency stateful streaming.**

```java
DataStream<Order> orders = env.fromSource(kafkaSource, ...);
orders
    .keyBy(Order::userId)
    .window(TumblingEventTimeWindows.of(Time.minutes(5)))
    .aggregate(new OrderCount())
    .sinkTo(kafkaSink);
env.execute();
```

**Pros**
- True event-time + watermarks.
- Exactly-once.
- Stateful operations at terabyte scale.
- Both bounded (batch) and unbounded (stream) modes.
- SQL via Flink SQL.
- Rich connector ecosystem (Kafka, Pulsar, Kinesis, JDBC, etc.).
- Low latency (sub-second).

**Cons**
- Separate cluster to operate.
- JVM-based.
- Steeper learning curve.
- Heavier than Kafka Streams for small jobs.

**When to use**
- Low-latency, high-throughput stream processing.
- Complex stateful operations (joins, sessions, complex event processing).
- Multi-source pipelines.
- You can invest in operating the cluster (or use managed: Ververica, AWS KDA).

### 3.3 Apache Spark Streaming / Structured Streaming

Spark's streaming model. **Micro-batch** by default: process small batches (~100ms–1s) of data continuously. Recent versions have continuous processing mode (lower latency).

```python
df = spark.readStream.format("kafka").load()
result = df.groupBy(window(df.timestamp, "5 minutes"), df.user_id).count()
result.writeStream.format("kafka").option("topic", "out").start()
```

**Pros**
- Unified API with Spark batch.
- SQL natively.
- Mature ecosystem.
- Good for ML pipelines (MLlib).

**Cons**
- Micro-batch = higher latency than true streaming (~100ms+).
- More resource-heavy than Flink for the same throughput.
- Stateful operations less mature than Flink.

**When to use**
- Already on Spark for batch.
- Latency requirements are seconds, not milliseconds.
- ML / data-engineering shops.

### 3.4 ksqlDB

SQL on top of Kafka Streams. Declarative, easy to read.

```sql
CREATE STREAM orders (id VARCHAR, user_id VARCHAR, total DECIMAL)
  WITH (KAFKA_TOPIC='orders', VALUE_FORMAT='JSON');

CREATE TABLE user_5min_count AS
  SELECT user_id, COUNT(*) AS c
  FROM orders
  WINDOW TUMBLING (SIZE 5 MINUTES)
  GROUP BY user_id
  EMIT CHANGES;
```

**Pros**
- SQL — accessible.
- Built on Kafka Streams under the hood; same guarantees.
- Quick prototyping.

**Cons**
- Less flexible than code.
- Kafka-only.
- Operational maturity less than Flink SQL.

**When to use**
- SQL-first teams.
- Quick stream-transformation jobs without writing services.

---

## 4. The Core Operators

### 4.1 Map / Filter / FlatMap
Stateless transformations.

```python
events.filter(e => e.country == "US").map(e => e.with_tax())
```

### 4.2 GroupBy / KeyBy
Partition the stream by a key. Required for stateful ops.

### 4.3 Windowing
Bucket events into windows.

- **Tumbling** — fixed-size, non-overlapping (every 5 min).
- **Hopping / sliding** — fixed-size, overlapping (every 5 min, hop 1 min).
- **Session** — gap-based grouping (a user's burst of activity).
- **Global** — one window, never closes.

```
TUMBLING 5min:    |--5min--|--5min--|--5min--|--5min--|
HOPPING  5min/1m: |--5min--|
                    |--5min--|
                      |--5min--| ...
SESSION (30s gap):  ▓ ▓ ▓     ▓ ▓ ▓         ▓ ▓
                    └─session─┘ └─session─┘ └sess┘
```

### 4.4 Aggregations
Reduce, sum, count, average within a window. Stateful.

### 4.5 Joins
Combine two streams or a stream and a table. Stateful.
- **Stream-stream join** — both sides are infinite; needs a window to bound the state.
- **Stream-table join** — table is a small lookup or materialized state; stream events enrich.

### 4.6 State stores
RocksDB-backed local store (Kafka Streams, Flink). Replicated to a Kafka changelog topic for recovery.

---

## 5. Event Time and Watermarks in Detail

```
events:  e1@10:00:05  e2@10:00:03  e3@10:00:10  e4@10:00:01 (late)

clock at processing:  10:00:08    10:00:08    10:00:11    10:00:15

window 10:00:00 - 10:00:05 (5-second tumbling):
  e1 belongs in 10:00:05–10:00:10 window (event time 05 is excluded)
  e2 belongs in 10:00:00–10:00:05
  e4 belongs in 10:00:00–10:00:05 — but it arrives at 10:00:15!

  watermark advances to ~10:00:10 by the time e3 arrives
  watermark advances past 10:00:05 → window 10:00:00–05 closes
  e4 arrives AFTER the close → late event
```

Late event handling options:
- **Drop**.
- **Sent to a side output**.
- **Trigger a window update / re-emit**.

Flink lets you specify `allowedLateness(Duration)` — keep the window state open this long; recompute on late events. Trade-off: more state, more re-emits.

---

## 6. Fault Tolerance: Checkpoints

State + offset committed atomically via **checkpoints**.

```
   process events ──► state changes ──► [periodic snapshot to durable store]
                                                  │
                                          on crash, restore from latest snapshot
                                          + replay events from saved offset
```

- **Kafka Streams**: state is materialized in a Kafka topic; offsets committed transactionally.
- **Flink**: distributed snapshot algorithm (Chandy-Lamport variant). State snapshots to S3/HDFS/etc. plus Kafka offset commit.
- **Spark Structured Streaming**: checkpoint dir on HDFS/S3; offsets + state.

Recovery time depends on snapshot size and replay length. Tune checkpoint interval: too short = overhead; too long = slow recovery.

---

## 7. Exactly-Once in Streaming

Within Kafka Streams: configure `processing.guarantee=exactly_once_v2`. Done. Reads, state updates, writes, offset commit all atomic.

Flink: enable exactly-once in checkpoint config; sinks must implement two-phase commit (Kafka, JDBC with idempotent write).

Spark Structured Streaming: idempotent sinks (file-based) + offset checkpoints. "Exactly-once" semantics with cooperation.

For end-to-end exactly-once (Kafka → DB), the DB sink must support transactions or be naturally idempotent. See [Delivery Guarantees →](./delivery-guarantees.md).

---

## 8. Common Patterns

### 8.1 Tumbling-window aggregation
Count events per 5-minute bucket per user.
- Pattern: `groupBy(user).window(TumblingWindow(5min)).count()`.

### 8.2 Sessionization
Group events into sessions based on inactivity gaps.
- Pattern: `groupBy(user).window(SessionWindow(gap=30s)).aggregate(...)`.

### 8.3 Anomaly detection
Compare incoming events against historical state.
- Pattern: enrich stream with state; flag deviations.

### 8.4 ETL: clean and route
Validate, transform, route to multiple sinks.
- Pattern: read from Kafka → filter → branch by type → write to N topics or to DBs.

### 8.5 Materialized view
Build a denormalized table from event stream.
- Pattern: stream-table join + state store → expose as KTable / table sink.

### 8.6 CDC pipeline
Database changes (via Debezium) → stream → derived stores (Elasticsearch, cache, materialized views).

### 8.7 Streaming join
Enrich events with reference data.
- Pattern: stream-table join with a regularly-refreshed table.

---

## 9. Operating Stream Processors

### What you watch
- **Lag** — input topic lag, per partition, per consumer.
- **Throughput** — events / sec.
- **Latency** — event time → output time. p50, p99.
- **State size** — RocksDB size, checkpoint sizes.
- **Backpressure** — slow sinks cause upstream backpressure; Flink visualizes this.
- **Restarts** — how often jobs restart; why.

### What hurts
- **State growth without TTL** — runaway memory.
- **Skewed key distribution** — one partition does 90% of the work.
- **Late events without bounds** — windows stay open forever.
- **Checkpoint storms** — too-frequent snapshots eat I/O.
- **Garbage collection** in JVM under load.
- **Schema evolution** — adding fields → state incompatible → forced replay.
- **DNS, network, broker churn** — restart loops.

### Things you tune
- Parallelism (per-operator).
- Checkpoint interval.
- State TTL.
- Allowed lateness.
- Buffer / batch sizes.

---

## 10. Choosing an Engine

```
Already on Kafka, simple per-service streaming, modest state?
  → Kafka Streams (or ksqlDB for SQL).

Need lowest latency, complex state, event-time correctness, biggest scale?
  → Flink.

Already deep in Spark / batch ecosystem; latency in seconds is fine?
  → Spark Structured Streaming.

SQL-first team, Kafka-centric?
  → ksqlDB.

Need to glue many sources without writing code?
  → Flink SQL or Spark SQL streaming.
```

For most green-field stream processing in 2026 with high stakes (low latency, large state, exactly-once): **Flink**.

For embedded-in-service stream processing on Kafka: **Kafka Streams**.

---

## 11. Worked Example: Real-Time Click Counts

We want to show every page's view count, updated in near-real-time. Bonus: detect "trending" pages (sudden spike).

### Pipeline
```
clicks topic (millions/sec)
   │
   ▼ Flink job
   keyBy(pageId)
     .window(SlidingEventTimeWindows(5min, hop 30s))
     .aggregate(count)
   │
   ▼ output to:
   - Redis (current 5-min count per page)
   - Kafka topic "trending" (when count exceeds threshold)
```

### Considerations
- Event time on click timestamp.
- Watermark = max observed - 10 seconds. Late clicks > 10s old are dropped.
- State: 5-min count per page. With 10M pages × 8 bytes = 80 MB per parallelism unit. Fits in RocksDB easily.
- Checkpoints every 30 seconds to S3.
- Output sink: Redis with PUT-by-page-id (naturally idempotent).
- Exactly-once: Kafka source + Redis idempotent sink + Flink exactly-once mode.

### Scaling
- 32 parallel sub-tasks. KeyBy(page_id) ensures same page goes to same sub-task.
- Hot pages (homepage) on one sub-task → skew. Mitigate: salt the key for the very hottest pages (split count across N salts, sum at query time).

---

## 12. Stream Processing vs Other Patterns

### vs Batch
- Batch processes bounded; stream processes unbounded.
- Stream gives sub-second latency; batch gives hourly/daily.
- Same engine often does both (Flink, Spark).
- See [Batch vs Stream →](./batch-vs-stream.md).

### vs Lambda Architecture
- Lambda: maintain batch + streaming paths in parallel.
- Stream-only is increasingly preferred (Kappa architecture).

### vs Event sourcing
- Event sourcing stores events as truth.
- Stream processing computes over events.
- Often combined.

### vs CDC
- CDC produces a stream from a DB.
- Stream processing consumes it.

---

## 13. Common Mistakes

- **Using processing time when event time matters.** Wrong buckets, especially with mobile retries.
- **Unbounded state.** Forever-growing without TTL.
- **No checkpointing or bad checkpoint config.** Crash = lose state.
- **Skewed keys.** One partition does all the work.
- **Joining two unbounded streams without windows.** State explodes.
- **Picking Flink when Kafka Streams would do.** Cluster ops you didn't need.
- **Picking Spark when latency requires Flink.** Micro-batch isn't milliseconds.
- **No DLQ for processing failures.** Bad events crash the job. Use side outputs.
- **Trying to do exactly-once without sink cooperation.** Sink dedup or transaction needed.
- **Schema evolution without compatibility.** State can't be loaded after deploy.
- **Operating Flink without snapshots to S3.** Recovery impossible.

---

## 14. Cheat Card

```
WHAT          process unbounded data continuously
              emit results as data arrives

CONCEPTS
  event time    real-world time (use this)
  processing    machine clock (avoid)
  watermark     "seen all events ≤ T" estimate
  window        tumbling / hopping / sliding / session
  state         joins, aggregations, sessions
  checkpoint    durable snapshot of state + offset

ENGINES
  Kafka Streams  library in your service, Kafka-only, simple
  Flink          gold standard for low-latency stateful streaming
  Spark Stream   micro-batch, mature ecosystem, latency in seconds
  ksqlDB         SQL on Kafka Streams

GUARANTEES     exactly-once within engine + idempotent sinks

OPERATORS      map, filter, keyBy, window, aggregate, join

WHEN STREAMING - latency matters, results emitted live,
                derived data, real-time analytics

PITFALLS       processing-time-not-event-time,
               unbounded state, skewed keys,
               no DLQ, wrong engine choice

RULE           Choose by latency + state size:
               seconds ok → Spark; ms + heavy state → Flink;
               embedded in service → Kafka Streams.
```

---

## 15. Resources

### Books
- *Streaming Systems* — Tyler Akidau et al. (the bible for the why and how).
- *Stream Processing with Apache Flink* — Hueske & Kalavri.
- *Kafka Streams in Action* — Bill Bejeck.
- *Spark: The Definitive Guide* — Bill Chambers, Matei Zaharia.

### Papers
- "The Dataflow Model" — Google, VLDB 2015 (the foundation).
- "Chandy-Lamport snapshots" (the algorithm Flink uses).

### Documentation
- **Apache Flink**: <https://flink.apache.org/>
- **Kafka Streams**: <https://kafka.apache.org/documentation/streams/>
- **Spark Structured Streaming**: <https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html>
- **ksqlDB**: <https://ksqldb.io/>

### Articles
- "The world beyond batch: Streaming 101 / 102" — Tyler Akidau (foundational reading).
- "Kappa Architecture" — Jay Kreps.
- "Why we chose Flink" — Pinterest, Lyft, Uber engineering blogs.

### Videos
- Tyler Akidau — Streaming 101 / 102 talks.
- Confluent — Kafka Streams series.
- ApacheCon Flink keynotes.

### Tools
- **Flink**, **Kafka Streams**, **Spark Streaming**, **ksqlDB**.
- **Beam** — unified batch/stream model; runs on Flink/Spark/Dataflow.
- **Hazelcast Jet**, **Apache Storm** (older), **Apache Samza** (LinkedIn).
- Managed: **AWS Kinesis Data Analytics**, **Confluent Cloud ksqlDB**, **Google Dataflow**, **Azure Stream Analytics**.

### Adjacent reading
- [Kafka Deep Dive →](./kafka.md)
- [Batch vs Stream →](./batch-vs-stream.md)
- [Event-Driven Architecture →](./event-driven-architecture.md)
- [Event Sourcing →](./event-sourcing.md)
- [Delivery Guarantees →](./delivery-guarantees.md)
- [Apache Spark →](../17-big-data/spark.md)
- [Apache Flink →](../17-big-data/flink.md)

---

*Previous:* [← Dead Letter Queues](./dead-letter-queues.md)  |  *Next:* [Batch vs Stream Processing →](./batch-vs-stream.md)
