# RabbitMQ, ActiveMQ, AWS SQS, Google Pub/Sub

> **TL;DR** — The non-Kafka brokers, each with a different shape. **RabbitMQ** is the most-deployed open-source broker — AMQP-based, exchanges + queues + bindings, flexible routing, ~100k messages/sec per node, great for traditional work queues and pub/sub at moderate scale. **ActiveMQ** is the older JMS-era broker; **Artemis** is its modern replacement; still common in JVM enterprise. **AWS SQS** is the managed standard — pure queue, FIFO and Standard variants, infinitely scalable, simplest API, the default queue on AWS. **Google Cloud Pub/Sub** is durable pub/sub at global scale, at-least-once, designed to absorb arbitrary throughput. Each has a sweet spot — pick by **scale**, **routing needs**, **managed-vs-self-hosted**, and **language/ecosystem fit**. For everything bigger or log-shaped, see [Kafka →](./kafka.md).

---

## 1. The Landscape

```
                          self-hosted                 managed
                       ┌─────────────────┬─────────────────────────┐
   queue + pub/sub     │  RabbitMQ       │  AWS SQS+SNS, GCP       │
                       │  ActiveMQ       │  Pub/Sub, Azure SB      │
                       ├─────────────────┼─────────────────────────┤
   stream / log        │  Kafka, Pulsar  │  AWS MSK, Confluent     │
                       │  Redpanda       │  Cloud, GCP Managed     │
                       │                 │  Kafka, Kinesis         │
                       └─────────────────┴─────────────────────────┘
```

This page covers the queue + pub/sub column. For streams, see [Kafka →](./kafka.md).

---

## 2. RabbitMQ

### What it is
The most popular open-source message broker, originally implementing **AMQP 0.9.1**. Erlang-based, runs as a cluster, supports many protocols (AMQP 1.0, MQTT, STOMP). Used by everyone from startups to banks.

### Core model: exchanges and queues

```
   publisher ──► EXCHANGE ──► (binding) ──► QUEUE ──► consumer
                    │
                    routing logic
```

**Exchanges** are routing nodes; **queues** are where messages wait; **bindings** connect them by routing rules.

Exchange types:
- **Direct** — route by exact routing key match. Pure queue semantics.
- **Fanout** — broadcast to all bound queues. Pub/sub.
- **Topic** — route by pattern (e.g., `orders.*.created`). Pub/sub with filtering.
- **Headers** — route by message headers. Rare.

This is RabbitMQ's superpower: **the routing topology is data, not code**. You can switch a single-queue setup to fan-out without changing producers.

### Strengths
- **Flexible routing** — direct, fanout, topic, headers; bindings are runtime config.
- **Multi-protocol** — AMQP, MQTT, STOMP, HTTP via plugins.
- **Mature** — battle-tested for 15+ years.
- **Featureful** — TTL, dead-lettering, priority queues, delayed messages (plugin), per-message expiration, quorum queues (Raft-based), streams (since 3.9).
- **Strong client libraries** in every language.
- **Management UI** — solid web console.

### Weaknesses
- **Throughput** is moderate: ~50–100k msgs/sec per node for classic queues. Streams are higher but newer.
- **Erlang ops** — not most teams' comfort zone.
- **Clustering quirks** — split-brain risk with old "mirrored queues" (deprecated). **Quorum queues** (Raft) are the modern answer.
- **No native replay** unless using RabbitMQ Streams (Kafka-like, added 3.9).

### When to use
- Traditional task queues at moderate scale (most use cases).
- Complex routing rules (topic exchange shines).
- Mixed protocols (MQTT for IoT + AMQP for backend).
- Per-message TTL, priority, delay needs.

### When not to use
- Massive log-shaped workloads (use Kafka).
- Simplest possible queue on AWS (use SQS).
- Need durable replay history beyond 24h (use Kafka or RabbitMQ Streams).

### Real users
Slack uses RabbitMQ extensively. Reddit used it. Instagram. Iterable. Discord (historically, before migrating some to Kafka).

### Sample config
```python
import pika

# producer
conn = pika.BlockingConnection(pika.ConnectionParameters('localhost'))
ch = conn.channel()
ch.exchange_declare(exchange='orders', exchange_type='topic')
ch.basic_publish(exchange='orders', routing_key='orders.us.created', body=msg)

# consumer
ch.queue_declare(queue='inventory', durable=True)
ch.queue_bind(queue='inventory', exchange='orders', routing_key='orders.*.created')
ch.basic_consume(queue='inventory', on_message_callback=callback, auto_ack=False)
```

---

## 3. ActiveMQ (and Artemis)

### What they are
- **ActiveMQ Classic** — original (2004), JMS-first, very stable but dated.
- **Apache Artemis** (a.k.a. ActiveMQ Artemis) — modern rewrite, more performant, intended to replace Classic. AMQP + JMS + STOMP + MQTT.

### Where they fit
- **Java/JEE enterprise** — JMS API is the natural fit. Banks, telco, insurance.
- **Legacy systems** that already use them.
- **Hybrid messaging** — Artemis supports many protocols cleanly.

### Strengths
- **JMS native** — fits Java enterprise stacks (Spring JMS, EJB).
- **Multi-protocol**.
- **Persistent and reliable**.

### Weaknesses
- **Smaller community** today than RabbitMQ / Kafka.
- **Operating overhead** vs cloud-managed alternatives.
- Often pushed by enterprise architects rather than chosen on merit.

For new projects in 2026, prefer RabbitMQ or Kafka unless you have specific JMS / IBM-shop reasons.

---

## 4. AWS SQS

### What it is
A fully managed message queue service. The oldest AWS service (launched 2006). Two variants:

- **SQS Standard** — at-least-once, best-effort ordering, virtually unlimited throughput.
- **SQS FIFO** — exactly-once processing (with dedupe IDs), strict ordering per message group, ~3000 msgs/sec per FIFO queue (or per group with high throughput mode).

### Core API
- `SendMessage` — push.
- `ReceiveMessage` — pull (long-poll up to 20s).
- `DeleteMessage` — ack.
- `ChangeMessageVisibility` — extend processing time.
- `SendMessageBatch` / `DeleteMessageBatch` — up to 10 at a time.

### Mental model
```
   producer ──► SendMessage ──► [ SQS queue (managed) ]
                                       │
                                  long-poll
                                       │
                                       ▼
                                   consumer
                                  process + delete
```

No exchanges, no bindings, no fanout. Just a queue. For pub/sub, combine with **SNS**:

```
   publisher ──► SNS topic ──► SQS queue A ──► consumer A
                           ──► SQS queue B ──► consumer B
                           ──► Lambda
                           ──► HTTP endpoint
```

### Strengths
- **Zero operations** — fully managed, infinitely scalable.
- **Cheap** at small scale ($0.40 per million standard).
- **DLQ built in** — configure max-receive-count and DLQ ARN.
- **Visibility timeout** is the at-least-once primitive.
- **Long polling** reduces wasted RPC.

### Weaknesses
- **No native fanout** — needs SNS.
- **No replay** — once deleted, gone.
- **FIFO throughput limited** (3k/sec; high-throughput mode raises to 30k+).
- **API quirks** — base64, XML/JSON, message size limit 256 KB.
- **Latency** — slightly higher than self-hosted (10–100 ms).
- **Per-message cost** at extreme scale gets expensive.

### When to use
- Default for **any new AWS-hosted background job system**.
- Simplest possible queue without ops burden.
- When you don't need replay.
- Lambda triggers.

### When not to use
- Need replay or history.
- Need complex routing (use SNS + SQS, or RabbitMQ).
- Throughput > what FIFO can provide if you need ordering.

### FIFO specifics
- **MessageGroupId** — messages with the same group ID are strictly ordered. Different groups process in parallel.
- **MessageDeduplicationId** — used for content-based dedup, 5-minute window.
- Standard SQS is way faster; only use FIFO when you actually need ordering.

---

## 5. Google Cloud Pub/Sub

### What it is
Google's managed pub/sub service. Designed for **at-least-once delivery at any throughput**. Pull or push subscribers. Powers internal Google services (Gmail, Ads) — battle-tested at extreme scale.

### Core model

```
   publisher ──► Topic ──► Subscription A ──► subscriber A
                       ──► Subscription B ──► subscriber B
                       (each subscription is independent)
```

A **subscription** is the consumer's persistent view of the topic. Messages stay in the subscription until acked. Multiple subscribers on one subscription compete (queue semantics); different subscriptions get separate copies (pub/sub).

This combines the best of pub/sub (per-subscription delivery) with queue (competing consumers in one subscription).

### Strengths
- **Global throughput** — no shard / partition limits. Just throw traffic at it.
- **Durable** — default 7 days retention.
- **Ordering** — supported per ordering key (similar to Kafka partitioning).
- **Push or pull** — subscribers can receive HTTP POSTs (push) or pull.
- **Dead-letter** — built-in DLT after N redeliveries.
- **Exactly-once delivery** (recent feature, per subscription).
- **Filter expressions** — subscribers can subscribe with filters.

### Weaknesses
- **Cost** — pay per message ingest + delivery + storage. At high volume, this adds up.
- **GCP-specific** — vendor lock-in.
- **Per-region** at base level; cross-region is a separate config.
- **Replay** is by snapshot/seek, not arbitrary offset like Kafka.

### When to use
- On GCP, default for inter-service messaging.
- Want pub/sub durability without operating Kafka.
- Variable / spiky throughput.

### Sample code
```python
from google.cloud import pubsub_v1
pub = pubsub_v1.PublisherClient()
topic = pub.topic_path('my-proj', 'orders')
pub.publish(topic, b'order placed', order_id='42')

sub = pubsub_v1.SubscriberClient()
sub.subscribe(sub.subscription_path('my-proj', 'inventory-sub'), callback)
```

---

## 6. Azure Service Bus, NATS, MQTT (Honorable Mentions)

### Azure Service Bus
- Managed, AMQP-based.
- Two tiers: **Queues** and **Topics + Subscriptions**.
- Supports sessions (FIFO), scheduled messages, duplicate detection.
- The Azure-native answer; similar shape to RabbitMQ for users.

### NATS
- Cloud-native, very fast, very lightweight.
- Originally pub/sub; **JetStream** adds persistence and queue groups.
- Sub-millisecond latency.
- Used by: Synadia, lots of IoT/edge platforms, internal systems where Kafka is overkill.

### MQTT (Mosquitto, EMQX, HiveMQ)
- IoT-focused pub/sub.
- Lightweight, QoS levels 0/1/2.
- Designed for low-bandwidth, low-power devices.
- Used heavily in connected cars, smart home, industrial sensors.

### Redis Streams
- Kafka-lite inside Redis.
- Best for small / medium scale or when you already have Redis.
- See [Redis Deep Dive →](../05-caching/redis-deep-dive.md).

---

## 7. Big Comparison Table

| Aspect | RabbitMQ | ActiveMQ Artemis | AWS SQS | Google Pub/Sub | Kafka (for context) |
|---|---|---|---|---|---|
| Type | queue + pub/sub | queue + pub/sub | queue | pub/sub (queue-via-subscription) | stream |
| Hosted | self / managed (CloudAMQP, AWS MQ) | self / AWS MQ | managed only | managed only | self / MSK / Confluent |
| Throughput per node/instance | 50–100k msg/s | ~50k msg/s | unlimited (managed) | unlimited (managed) | 500k+ msg/s per cluster |
| Latency | 1–10 ms | 1–10 ms | 10–100 ms | 50–200 ms | 5–50 ms |
| Routing | direct/fanout/topic/header | similar | none (use SNS) | filter / ordering key | partition key |
| Replay | no (Streams plugin: yes) | no | no | yes (snapshot/seek, ~7d) | yes (retention-bounded) |
| Delivery | at-least-once (typical) | at-least-once | at-least / exactly (FIFO) | at-least-once (exactly-once optional) | at-least / exactly |
| Persistence | optional / durable queues | yes | yes | yes (7d default) | yes (configurable) |
| Multi-protocol | yes (AMQP/MQTT/STOMP) | yes | no (AWS API) | no (GCP API) | Kafka wire protocol |
| Operational burden | medium | medium | none | none | high |
| Best fit | flexible routing | JMS / enterprise | AWS default queue | GCP default messaging | event streaming, log |

---

## 8. Choosing in Practice

Quick decision flow:

```
On AWS, just need a queue?            → SQS Standard
On AWS, need ordered queue?           → SQS FIFO
On AWS, need pub/sub fanout?          → SNS → SQS
On GCP?                               → Cloud Pub/Sub
On Azure?                             → Service Bus
Self-host, simple queue?              → RabbitMQ
Self-host, complex routing?           → RabbitMQ topic exchange
JMS / Java enterprise legacy?         → Artemis
Need streaming / replay / log?        → Kafka (or Pulsar / Redpanda)
IoT / millions of devices?            → MQTT broker (EMQX, Mosquitto)
Sub-ms latency, small footprint?      → NATS
```

For most teams in 2026:
- On cloud → use the cloud-managed broker for queues. Don't self-host unless you have reason.
- For event streaming → Kafka regardless.
- RabbitMQ is the sensible self-hosted choice when SQS/Pub/Sub isn't an option.

---

## 9. Operating Concerns by Broker

### RabbitMQ
- **Quorum queues** (Raft) over mirrored queues (deprecated).
- Set **`max-length`** to bound queue depth.
- Use **lazy queues** for very large backlogs (paged to disk).
- Watch memory limits — RabbitMQ will throttle producers when memory fills.
- Cluster topology: ≥3 nodes, network partition policy (`pause_minority` typical).

### ActiveMQ Artemis
- Use **persistent journals**.
- Configure address routing carefully — different from RabbitMQ.
- HA via replication; clustering for load distribution.

### SQS
- Almost zero ops — just configure DLQ, visibility timeout, max receive count.
- Visibility timeout > max processing time (otherwise duplicates).
- Long-poll wait time = 20s for efficiency.
- Batch where possible to cut request cost.

### Google Pub/Sub
- Per-subscription ack deadline; extend via `ModifyAckDeadline` for long processing.
- Push vs pull — pull is cheaper at scale; push is simpler for serverless.
- Ordering keys disable parallelism for that key — be deliberate.

---

## 10. Patterns

### Work queue
- Single queue, multiple consumers, at-least-once.
- Tools: SQS, RabbitMQ direct queue, Pub/Sub subscription.

### Fan-out pub/sub
- One publish, many consumers.
- Tools: SNS → multiple SQS; RabbitMQ fanout exchange; Pub/Sub multiple subscriptions.

### Topic-based routing
- Publish with a key; subscribers filter.
- Tools: RabbitMQ topic exchange; Pub/Sub filters; SNS filter policies.

### Delayed / scheduled messages
- Publish now, deliver at time T.
- Tools: SQS delayed messages (up to 15 min); RabbitMQ delayed-message plugin; Pub/Sub with scheduler.

### Priority queue
- Higher-priority messages processed first.
- Tools: RabbitMQ priority queues; or multiple queues with consumer routing.

### Request-reply (RPC over messaging)
- Producer publishes, waits for response on a reply queue.
- Tools: RabbitMQ reply-to; any broker with correlation IDs.
- Generally discouraged — use HTTP/gRPC for synchronous; messaging for async.

---

## 11. Common Mistakes

- **Using SQS Standard when you need ordering.** Standard is best-effort; use FIFO.
- **Treating SQS as a stream.** No replay; no history. If you need that, choose Kafka / Pub/Sub.
- **RabbitMQ memory blowing up** from unconsumed messages. Set `max-length`; use lazy queues for big backlogs.
- **Mirrored queues** in production. Migrate to quorum queues.
- **No DLQ.** Bad messages loop forever.
- **Visibility timeout too short** in SQS. Consumer takes longer than timeout → duplicate delivery.
- **No idempotent consumer.** All these brokers are at-least-once; duplicates happen.
- **Pub/Sub ordering key on every message.** Disables parallelism; latency spikes. Use only when ordering matters.
- **Self-hosting RabbitMQ at scales it doesn't enjoy.** > 500k msg/s → look at Kafka or Pulsar.
- **No monitoring of queue depth / consumer lag.** Backlog grows silently until something dies.

---

## 12. Cheat Card

```
RABBITMQ      classic broker; exchanges + queues; flexible routing
              direct / fanout / topic / headers
              ~100k/s per node; quorum queues for HA
              best for: flexible routing, multi-protocol, moderate scale

ACTIVEMQ/ARTEMIS  JMS-first; Java enterprise; multi-protocol
                  for: legacy JMS shops

SQS           managed AWS queue; standard or FIFO
              zero ops; combine with SNS for fan-out
              best for: AWS background jobs, any queue you'd self-host

GCP PUB/SUB   managed durable pub/sub; topics + subscriptions
              per-subscription view; unlimited throughput
              best for: GCP services, durable broadcast

AZURE SB      managed AMQP queue + topics; sessions for FIFO
              best for: Azure native

CHOOSING      cloud-managed > self-hosted unless you have reason
              SQS for AWS queues; Pub/Sub for GCP; RabbitMQ if self-host

PITFALLS      no DLQ; visibility timeout shorter than work;
              mirrored queues; no idempotency; SQS-as-stream

RULE          Pick the broker that matches the shape:
              compete = queue, broadcast = pub/sub, log = stream.
```

---

## 13. Resources

### Books
- *RabbitMQ in Depth* — Gavin M. Roy.
- *Enterprise Integration Patterns* — Hohpe & Woolf.
- *Designing Data-Intensive Applications* — Kleppmann.

### Documentation
- **RabbitMQ**: <https://www.rabbitmq.com/documentation.html>
- **Apache Artemis**: <https://activemq.apache.org/components/artemis/>
- **AWS SQS**: <https://docs.aws.amazon.com/AWSSimpleQueueService/>
- **AWS SNS**: <https://docs.aws.amazon.com/sns/>
- **Google Cloud Pub/Sub**: <https://cloud.google.com/pubsub/docs/overview>
- **Azure Service Bus**: <https://learn.microsoft.com/en-us/azure/service-bus-messaging/>

### Articles
- "Why we moved from RabbitMQ to Kafka" — common engineering blog post; usually about scale.
- "SQS at scale" — AWS blog.
- "How Google built Pub/Sub" — Google Cloud blog.

### Videos
- ByteByteGo — "Top 5 Message Brokers".
- Tim Berglund — broker comparisons.
- Hussein Nasser — RabbitMQ deep dive.

### Tools
- **RabbitMQ**, **Artemis**, **SQS**, **SNS**, **Cloud Pub/Sub**, **Azure SB**.
- **NATS**, **MQTT brokers** (EMQX, Mosquitto, HiveMQ).
- **CloudAMQP**, **AWS MQ** for managed RabbitMQ / Artemis.

### Adjacent reading
- [Queue vs Pub/Sub vs Stream →](./queue-vs-pubsub-vs-stream.md)
- [Kafka Deep Dive →](./kafka.md)
- [Delivery Guarantees →](./delivery-guarantees.md)
- [Dead Letter Queues →](./dead-letter-queues.md)
- [Event-Driven Architecture →](./event-driven-architecture.md)
- [Outbox Pattern →](./outbox-pattern.md)

---

*Previous:* [← Kafka Deep Dive](./kafka.md)  |  *Next:* [Event-Driven Architecture →](./event-driven-architecture.md)
