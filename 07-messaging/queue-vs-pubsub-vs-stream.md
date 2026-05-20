# Message Queues vs Pub/Sub vs Streams

> **TL;DR** — Three families of async messaging, often confused. A **message queue** delivers each message to **exactly one consumer** (work distribution). **Pub/Sub** delivers each message to **every subscriber** (fan-out broadcast). A **stream** is a **durable, replayable, ordered log** that multiple consumer groups read independently at their own pace — a pub/sub with persistence and rewind. Real brokers blur the lines: **RabbitMQ** is a queue + pub/sub broker; **Kafka** is a stream that mimics queues via consumer groups; **SQS** is pure queue; **Google Pub/Sub** and **AWS SNS** are pure pub/sub. The decision driver is: **do consumers compete for messages (queue) or each get a copy (pub/sub), and do you need replay / history (stream)?**

---

## 1. The Three Patterns

### 1.1 Queue (work distribution)

```
   producers ──► [ queue ] ──► consumer 1
                          ──► consumer 2
                          ──► consumer 3

Each message goes to ONE consumer (competing consumers).
Once acknowledged, it's gone.
```

Mental model: a job board. One job, one worker takes it.

### 1.2 Pub/Sub (broadcast)

```
                       ┌──► subscriber A
   publisher ──► topic ┼──► subscriber B
                       └──► subscriber C

Every subscriber gets every message.
Usually no replay; subscribers must be connected at publish time
(or have a per-subscriber queue behind them).
```

Mental model: an announcement. Every interested party hears it.

### 1.3 Stream (durable log)

```
   producers ──► [ partitioned, durable log ]
                        │
                ┌───────┼───────┐
                ▼       ▼       ▼
         consumer-group-A   consumer-group-B   consumer-group-C
         (each maintains its own offset; can replay from any position)
```

Mental model: a transaction log. Append-only. Many readers, each at their own position. Replay = rewind.

---

## 2. Side by Side

| Aspect | Queue | Pub/Sub | Stream |
|---|---|---|---|
| Delivery | one consumer | all subscribers | per-consumer-group; rewindable |
| Retention | until ack | until delivered (or short) | by time / size (days, weeks, forever) |
| Ordering | per-queue, often partial | per-topic | per-partition, strict |
| Replay | no | usually no | yes (defining feature) |
| Backpressure | natural (consumers ack) | tricky (slow sub holds others) | natural (offsets) |
| Use case | jobs, tasks, work distribution | notifications, fan-out, event broadcasting | event log, analytics, CDC, derived data |
| Examples | SQS, RabbitMQ (queue), Beanstalkd | SNS, Google Pub/Sub, Redis Pub/Sub | Kafka, Kinesis, Pulsar, Redpanda |

---

## 3. Why the Three Patterns Exist

Each solves a different coordination problem.

- **Queue** — distribute work across N workers without duplicates. The hard problem: only one consumer should handle a given message; both at-least-once and at-most-once delivery are interesting; failure handling needs DLQs.
- **Pub/Sub** — let many independent consumers react to the same event without the publisher knowing them. The hard problem: late or absent subscribers may miss messages; slow ones become bottlenecks.
- **Stream** — give every consumer a full, durable history, with the ability to replay. The hard problem: storage and ordering at scale; consumer-group coordination; offset management.

The patterns are composable. Kafka uses partitions to give pub/sub fan-out and stream-like replay; SQS + SNS combine to fan out to multiple queues; RabbitMQ exchanges route to queues for both fan-out and competing-consumer patterns.

---

## 4. Queues in Depth

### 4.1 The core promises
- **One consumer per message.** Even with N consumers, each message processed once (modulo retries).
- **At-least-once delivery** typically: the broker delivers, the consumer acks. If the consumer dies before ack, the message is redelivered.
- **FIFO or best-effort ordering** depending on broker.
- **Visibility timeout / ack timeout** — the broker hides the message during processing; on timeout, redelivers.
- **DLQ** — after N retries, route to a dead-letter queue. See [Dead Letter Queues →](./dead-letter-queues.md).

### 4.2 Architectures
- **AMQP queues** (RabbitMQ, ActiveMQ) — full-featured, exchanges, bindings.
- **JMS queues** (legacy Java).
- **AWS SQS** — managed, simple, FIFO and Standard variants.
- **Beanstalkd**, **Sidekiq + Redis**, **Resque**, **RQ** — lightweight.

### 4.3 Use cases
- Background jobs (email send, image resize, billing).
- Task fan-out within one service.
- Decoupling producers from slow consumers (buffer).
- Rate-limiting backend processing.

### 4.4 What hurts
- **Ordering across multiple consumers is hard** — typical queues only guarantee per-message order, not global.
- **Duplicate processing** — at-least-once means consumers must be idempotent.
- **Backed-up queues** — under sustained overload, you grow forever or shed. Plan retention.
- **Poison messages** — bad payloads that fail forever; need DLQ + max-retry.

---

## 5. Pub/Sub in Depth

### 5.1 The core promises
- **All subscribers get all messages** they're subscribed to.
- **Often fire-and-forget** — no persistence beyond delivery.
- **Filtering** — many brokers let subscribers filter by topic, headers, or attributes.

### 5.2 Architectures
- **Redis Pub/Sub** — in-memory, at-most-once. No persistence.
- **AWS SNS** — managed, fan-out to SQS / Lambda / HTTP / email.
- **Google Cloud Pub/Sub** — managed, at-least-once, durable, scales massively.
- **MQTT brokers** (Mosquitto, EMQX) — IoT-style pub/sub.
- **RabbitMQ topic exchanges** — pub/sub with routing patterns.

### 5.3 Use cases
- Notifications (system-wide events).
- Cache invalidation broadcasts.
- Real-time updates to many clients.
- Decoupling services in a microservice architecture (event-driven).

### 5.4 What hurts
- **Slow subscribers** — block or lose messages depending on broker.
- **Subscriber lifecycle** — a subscriber that disconnects and reconnects may miss messages (unless durable subscription).
- **Discovery / coupling** — pub/sub loosely couples but you still need someone to define the schema and topic.
- **No replay** — usually. If you need history, you need a stream.

### 5.5 Pub/Sub vs Stream
The line gets blurry. **Google Cloud Pub/Sub** is durable (~7 days retention by default) — closer to a stream. **Kafka** lets every consumer group read independently — pub/sub-like. The line is mostly about **ordering, replay, and partitioning model**, not durability.

---

## 6. Streams in Depth

### 6.1 The core promises
- **Durable, append-only log** — messages persist for the retention period (hours to forever).
- **Partitioned** — each topic split into partitions; per-partition order; parallel scale.
- **Consumer groups** — each group has its own offset; multiple groups read the same data independently.
- **Replay** — read from any offset, any time within retention.
- **High throughput** — millions of messages/sec per cluster (Kafka, Pulsar, Redpanda).

### 6.2 Architectures
- **Kafka** — the canonical stream broker. See [Kafka Deep Dive →](./kafka.md).
- **Apache Pulsar** — segmented architecture (storage and serving decoupled via BookKeeper).
- **Redpanda** — Kafka-compatible, single-binary, no ZooKeeper, written in C++.
- **AWS Kinesis** — managed stream, Kafka-like API at smaller scale.
- **Azure Event Hubs** — managed stream.
- **Redis Streams** — small-scale, in-process stream.

### 6.3 Use cases
- Event sourcing and CDC.
- Analytics pipelines (ingest events, compute later).
- Microservice event bus.
- Replication / data sync (logical replication via CDC + Kafka).
- Time-series data (logs, metrics, telemetry).
- Real-time stream processing (Kafka Streams, Flink).

### 6.4 What hurts
- **Operational complexity** — Kafka has a lot of knobs.
- **Storage cost** — durability isn't free; multi-week retention multiplies disk needs.
- **Per-partition ordering only** — no global order without single partition (which kills scale).
- **Schemas matter** — events live forever; consumers a year from now must parse them.

---

## 7. Choosing: A Decision Tree

```
Do consumers compete for each message (only one should handle)?
  └─ YES → QUEUE.

Do many independent consumers each need every message?
  ├─ ephemeral notifications, no replay → PUB/SUB.
  └─ durable, need to replay, build derived data → STREAM.

Do you need both? (most modern systems do)
  ├─ small scale → RabbitMQ (queue + topic exchange)
  ├─ medium → AWS SQS + SNS fan-out, or Pulsar
  └─ large → Kafka / Pulsar / Redpanda as the spine
```

In practice, large-scale systems pick **one stream broker** (Kafka) as the substrate and build queue / pub/sub semantics on top using consumer groups, retention, and topic naming conventions.

---

## 8. Mapping Patterns onto Brokers

### Kafka can do all three:
- **Queue**: one consumer group with multiple consumers — each message goes to one consumer in the group.
- **Pub/Sub**: many consumer groups on the same topic.
- **Stream**: the natural use case; retain for days, replay anytime.

### RabbitMQ:
- **Queue**: direct exchange → queue → competing consumers.
- **Pub/Sub**: fanout / topic exchange → multiple queues → one subscriber each.
- **Stream**: RabbitMQ Streams (since 3.9) — Kafka-like log.

### SQS + SNS:
- **Queue**: SQS by itself.
- **Pub/Sub**: SNS by itself (fan-out to many SQS / Lambda / HTTP).
- **Stream**: not really. Use Kinesis or MSK.

### Google Cloud Pub/Sub:
- **Pub/Sub**: native.
- **Queue**: ordering keys + single subscriber per ordering key.
- **Stream**: replay within retention period.

---

## 9. Ordering Models

Critical to understand because it drives partition / sharding decisions.

| Model | What it means | Examples |
|---|---|---|
| **No ordering** | Messages may be reordered arbitrarily | Redis Pub/Sub, SNS without ordering |
| **Per-queue / per-topic** | Order within a queue maintained on insertion | SQS Standard (best-effort), RabbitMQ |
| **Per-partition** | Order within a partition strict; cross-partition not guaranteed | Kafka, Kinesis, Pulsar |
| **Per-key** | Messages with the same routing key in order | Kafka (via partition key), SQS FIFO with message group ID |
| **Global** | All messages in one order | Single-partition Kafka (rare); some legacy systems |

The pattern: **partition by key** (`user_id`, `tenant_id`, `order_id`) so messages for the same logical entity are ordered, but the broker scales out across keys.

---

## 10. Delivery Semantics

A separate axis from queue-vs-pub/sub-vs-stream. Every broker chooses among:

- **At-most-once** — message may be lost; never duplicated. Use for non-critical metrics.
- **At-least-once** — message may be duplicated; never lost. Most common; consumers must be idempotent.
- **Exactly-once** — never duplicated, never lost. Hard. Kafka offers it within Kafka (transactional + idempotent producer); end-to-end is a fiction unless your consumers participate.

Full discussion in [Delivery Guarantees →](./delivery-guarantees.md).

---

## 11. Worked Examples

### Background email sending
- **Pattern**: queue.
- **Why**: one worker per email; failure → retry; eventually DLQ if email service rejects.
- **Tool**: SQS, RabbitMQ, Sidekiq.

### Order placed → notify many services (inventory, fulfillment, analytics, notifications)
- **Pattern**: pub/sub (or stream).
- **Why**: each service has its own logic; failure of one shouldn't block others.
- **Tool**: SNS → multiple SQS; or Kafka with multiple consumer groups.

### Analytics pipeline (ingest events, compute aggregates hourly + reprocess on schema change)
- **Pattern**: stream.
- **Why**: durable, replayable; many consumers (real-time + batch).
- **Tool**: Kafka + Flink / Spark / dbt downstream.

### Cache invalidation broadcast
- **Pattern**: pub/sub.
- **Why**: all pods need the invalidation; doesn't matter if a disconnected pod misses (TTL covers it).
- **Tool**: Redis Pub/Sub.

### Distributed transaction across services
- **Pattern**: stream (events) + saga orchestration.
- **Why**: each step emits an event; failures emit compensating events.
- **Tool**: Kafka + saga orchestrator (Camunda, Temporal). See [Saga Pattern →](./saga-pattern.md).

### Real-time stock ticker
- **Pattern**: pub/sub.
- **Why**: every connected client sees same ticks; missing one is fine (next tick comes).
- **Tool**: WebSocket fan-out; backed by a stream for durability.

---

## 12. Architectural Patterns

### 12.1 Event bus
A single Kafka cluster as the central "event bus" for an org. Every domain publishes events; every interested service subscribes. The spine of microservice architectures. See [Event-Driven Architecture →](./event-driven-architecture.md).

### 12.2 Queue per service
Each service has its own SQS / RabbitMQ queue for incoming work. Producers send to the right queue. Strong service boundaries; less inter-service coupling.

### 12.3 Fan-out (SNS-to-SQS)
Publisher sends to SNS; subscribers each have their own SQS. Combines pub/sub broadcast with per-subscriber durable queue.

### 12.4 Stream-as-database
Kafka is the source of truth (event sourcing). Read sides (databases, search indexes) are projections. See [Event Sourcing →](./event-sourcing.md).

### 12.5 CDC pipeline
Database commits → CDC stream → Kafka → consumers (search index, cache invalidation, derived stores). See [CDC →](../04-databases/cdc.md).

---

## 13. Operational Reality

### What you'll actually deal with:
- **Backlog growth** — consumers fall behind. Alarms on `consumer_lag` and queue depth.
- **Poison messages** — a bug or bad message stuck in retries. DLQ + investigation.
- **Schema drift** — producers change format; consumers break. Use schema registry (Avro / Protobuf / JSON Schema).
- **Cluster failures** — broker outages. Use replicated clusters; tolerate `acks=all` writes.
- **Duplicate processing** — at-least-once + retries. Make consumers idempotent.
- **Ordering surprises** — partition rebalances; cross-partition cases.
- **Cost** — Kafka storage and inter-AZ networking at scale.

Brokers are durable, useful, and dangerous when mis-configured. They become load-bearing walls in your architecture; treat them with the same care as your primary database.

---

## 14. Common Mistakes

- **Using pub/sub as a queue** by having one subscriber. You get pub/sub semantics (no redelivery on failure) when you wanted queue semantics. Use a queue.
- **Using a queue as a stream** by setting infinite retention and replaying. Get a stream broker.
- **Assuming exactly-once exists.** End-to-end exactly-once requires consumer participation. Build idempotent consumers.
- **No DLQ.** Bad messages loop forever; backlog grows; consumers blocked.
- **Single global partition for ordering.** Throughput caps at one partition's max. Partition by key instead.
- **No schema discipline.** Events live forever in a stream. Schema breakage = production incident.
- **Treating the queue as a database.** Read-once semantics + non-queryable storage. Wrong tool.
- **Coupling producer to consumer via direct knowledge.** Defeats the broker. Producer should publish; broker routes.

---

## 15. Cheat Card

```
QUEUE       one message → one consumer; work distribution
            SQS, RabbitMQ, Sidekiq, Beanstalkd

PUB/SUB     one message → all subscribers; broadcast
            SNS, Google Pub/Sub, Redis Pub/Sub, MQTT

STREAM      durable log; per-group offsets; replayable
            Kafka, Pulsar, Kinesis, Redpanda, Redis Streams

CHOOSE BY
  competing consumers?           → QUEUE
  broadcast, no replay?          → PUB/SUB
  durable, replayable, scale?    → STREAM

ORDERING    per-partition / per-key; never global at scale

DELIVERY    at-most / at-least / exactly-once (mostly fiction)

COMPOSE
  Kafka does all three patterns via consumer groups
  SNS+SQS fan-out is pub/sub + queues

PITFALLS    no DLQ, no schemas, ordering surprises,
            duplicate processing not idempotent

RULE        Pick by who-consumes-what, not by what's popular.
```

---

## 16. Resources

### Books
- *Designing Data-Intensive Applications* — Martin Kleppmann (Ch 11 covers all three patterns).
- *Kafka: The Definitive Guide* — Narkhede, Shapira, Palino.
- *Enterprise Integration Patterns* — Hohpe & Woolf (the canonical reference, 2003 but still relevant).

### Documentation
- **Kafka concepts**: <https://kafka.apache.org/documentation/#design>
- **RabbitMQ tutorials**: <https://www.rabbitmq.com/tutorials>
- **AWS SQS**: <https://docs.aws.amazon.com/AWSSimpleQueueService/>
- **Google Cloud Pub/Sub**: <https://cloud.google.com/pubsub/docs/overview>

### Articles
- "The Log: What every software engineer should know about real-time data's unifying abstraction" — Jay Kreps.
- "Streams vs Queues vs Pub/Sub" — Confluent blog.
- "Why I'm not over Kafka" — various engineering blogs.

### Videos
- ByteByteGo — "Top 5 Message Brokers".
- Tim Berglund — Kafka fundamentals.
- Martin Kleppmann — "Turning the database inside-out".

### Tools
- **Stream**: Kafka, Pulsar, Redpanda, Kinesis.
- **Queue**: SQS, RabbitMQ, Sidekiq, Beanstalkd.
- **Pub/Sub**: SNS, Google Cloud Pub/Sub, Redis Pub/Sub, MQTT brokers.

### Adjacent reading
- [Kafka Deep Dive →](./kafka.md)
- [Message Brokers →](./message-brokers.md)
- [Event-Driven Architecture →](./event-driven-architecture.md)
- [Delivery Guarantees →](./delivery-guarantees.md)
- [Dead Letter Queues →](./dead-letter-queues.md)
- [Stream Processing →](./stream-processing.md)
- [Synchronous vs Asynchronous Communication →](../03-apis/sync-vs-async.md)

---

*Up:* [README ↑](../README.md)  |  *Next:* [Kafka Deep Dive →](./kafka.md)
