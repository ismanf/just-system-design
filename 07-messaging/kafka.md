# Kafka Deep Dive (Topics, Partitions, Consumer Groups)

> **TL;DR** — **Apache Kafka** is a distributed, partitioned, replicated, append-only log. The core primitives: **topics** (named streams of messages), **partitions** (parallel shards of a topic, each an ordered immutable log), **brokers** (the servers; a cluster has many), **producers** (write to partitions), **consumers** (read), and **consumer groups** (a set of consumers that share the read load — each partition is consumed by exactly one consumer in the group). Kafka's power comes from **decoupling storage from processing**: messages are durable for days/weeks; multiple consumer groups read independently at their own offset; replay is just "set the offset back." It scales horizontally by adding partitions and brokers. The hard parts: **partition keys** (the single most consequential decision), **retention** (storage cost vs replayability), **rebalancing** (consumer-group changes cause pauses), and **exactly-once** (achievable within Kafka via transactional producer + idempotent consumers).

---

## 1. The Mental Model

Kafka is **not** a queue, **not** a database, **not** a service bus. It's a **distributed commit log** that other systems can read from in any way they choose.

```
   producers  ──────►  ┌─────────────────────────────┐
                        │  topic: orders               │
                        │  ┌──────┐ ┌──────┐ ┌──────┐  │
                        │  │ P0   │ │ P1   │ │ P2   │  │
                        │  │ 0 1.. │ │ 0 1..│ │ 0 1..│  │
                        │  └──────┘ └──────┘ └──────┘  │
                        └──────┬─────┬─────┬───────────┘
                               │     │     │
                       ┌───────┴─────┴─────┴────────┐
                       │ consumer group: inventory  │
                       │  C1 reads P0, P1           │
                       │  C2 reads P2               │
                       └────────────────────────────┘
                       (each partition consumed by one
                        consumer in the group)

   another consumer group "analytics" reads the same
   partitions independently, at its own offset.
```

The defining property: once written, messages stay for the configured retention. Multiple consumer groups read them at their own pace. Rewind to any offset is just `seek(partition, offset)`.

---

## 2. The Building Blocks

### 2.1 Brokers
A Kafka cluster has multiple **brokers** (servers). Brokers hold partitions, accept produces, serve consumes. A typical production cluster: 3–9 brokers. Very large: dozens to hundreds.

### 2.2 Topics
A logical name for a stream of messages. `orders`, `payments.events`, `cdc.users`. Topics are not Kafka's unit of parallelism — partitions are.

### 2.3 Partitions
Each topic is split into N **partitions**. Each partition is:
- An **ordered, immutable, append-only log** of messages.
- Stored on **one broker** (the leader) and **replicated** to others (followers).
- The **unit of parallelism** — one partition = one consumer slot in a consumer group.

```
partition 0 of topic "orders":
  offset:  0     1     2     3     4     5    ...
           ┌─────┬─────┬─────┬─────┬─────┬─────┐
  message: │ M0  │ M1  │ M2  │ M3  │ M4  │ M5  │
           └─────┴─────┴─────┴─────┴─────┴─────┘
                              ▲ consumer reads here (offset=3)
```

Adding partitions adds parallelism. Removing them is impossible (you can only add).

### 2.4 Offsets
Each partition assigns each message a monotonically-increasing 64-bit offset. Consumers track their offset per partition. **Committed offsets** are stored in a special internal topic (`__consumer_offsets`).

### 2.5 Producers
Write messages to topics. Choose the partition either:
- Explicitly (`producer.send(topic, partition, key, value)`).
- By **key**: `partition = hash(key) % num_partitions`. Same key → same partition → same order.
- Round-robin if no key.

Acks:
- `acks=0` — fire and forget. May lose.
- `acks=1` — leader confirms. May lose if leader dies before replicas catch up.
- `acks=all` (or `-1`) — all in-sync replicas confirm. Durable.

The producer also batches (`linger.ms`, `batch.size`) for throughput.

### 2.6 Consumers and Consumer Groups
A **consumer group** is a set of consumers cooperating to read a topic. Kafka assigns each partition to exactly one consumer in the group. If you have 12 partitions and 4 consumers, each consumer reads 3.

Multiple consumer groups read the same topic independently:
- Group `analytics` is at offset 1M.
- Group `cache-invalidator` is at offset 1.2M.
- Group `fraud-detector` is at offset 999k (replaying).

### 2.7 Replication
Each partition has a configurable **replication factor** (typically 3). One broker is the **leader**; others are **followers**. Writes go to leader; followers tail. Reads typically come from leader (configurable in newer Kafka).

**In-Sync Replicas (ISR)** — followers that are caught up. `acks=all` writes require all ISRs to ack.

If the leader dies, a follower becomes leader. **`min.insync.replicas=2`** with `acks=all` and `replication.factor=3` is the standard durability config.

---

## 3. Partitioning (The Decision)

The partition key determines **ordering**, **throughput**, and **consumer parallelism**. Choose well.

### 3.1 Good keys
- `user_id`, `order_id`, `tenant_id` — events for the same entity stay ordered.
- Stable values that don't change over a message's lifetime.
- High cardinality (many distinct values → even distribution).

### 3.2 Bad keys
- `country` — low cardinality, skew.
- `timestamp` — all writes hit the partition of the current time bucket.
- `null` — round-robin, no ordering guarantee.

### 3.3 How many partitions?
Rough rule: aim for **target throughput / per-consumer throughput** = consumer parallelism. So if you want 100k events/sec and each consumer handles 10k, you need 10 partitions minimum.

But also consider:
- **Memory** — each partition has overhead (file descriptors, buffers).
- **Failover time** — more partitions = longer leader election after broker failure.
- **You can only add, never remove.** Don't over-provision wildly.

Typical numbers: small topics 3–6 partitions; large 100–1000.

Confluent guidance: **a single broker can handle a few thousand partitions cleanly; tens of thousands gets ugly.**

### 3.4 The hot partition problem
If your partition key is skewed (e.g., one tenant is 50% of traffic), one partition gets overwhelmed. Mitigations:
- Combine keys (`tenant_id + user_id`).
- Hash a salt into hot tenants' messages so they spread.
- Move the hot tenant to its own dedicated topic.

---

## 4. Storage Layout

Each partition is a directory on the broker disk. Inside:
- **Segment files** (`.log`) — bounded-size chunks of the log. Default 1 GB.
- **Index files** (`.index`, `.timeindex`) — sparse indexes by offset and timestamp.
- **Snapshot files** for the producer state (transactions).

Kafka writes are **sequential appends** — disk-friendly. Modern Kafka on NVMe can sustain 1+ GB/sec write per broker.

Reads are also sequential when consumers tail. The OS page cache absorbs most of it; production Kafka uses ~80% of broker RAM as page cache.

### Retention
- **Time-based** (`retention.ms=604800000` for 7 days).
- **Size-based** (`retention.bytes=...` per partition).
- **Compacted topics** — keep only the latest value per key. Used for tables-as-topics, K/V projections.

When retention hits, old segments are deleted (or compacted).

---

## 5. Consumer Groups in Depth

### 5.1 Assignment
The group coordinator (a broker) assigns partitions to consumers using a strategy:
- **Range** (default) — assign contiguous ranges. Can be unbalanced.
- **Round-robin** — alternate. More even.
- **Sticky** — minimize reshuffle on rebalance. Recommended.
- **CooperativeSticky** (newer) — incremental rebalance, no stop-the-world. Best for production.

### 5.2 Rebalancing
When a consumer joins or leaves the group, partitions are reassigned. Classic rebalance:
1. All consumers pause.
2. Coordinator computes new assignment.
3. Consumers resume.

The pause can be seconds. With **cooperative sticky** strategy, only the affected partitions move; others keep processing.

### 5.3 Commit semantics
- **Auto-commit** — periodically commit current offset. Easy but risky: you can commit before processing finishes, losing work on crash.
- **Manual commit** — commit after processing. Safer. The default for any production system.

```python
for msg in consumer.poll():
    process(msg)
    consumer.commit({msg.partition: msg.offset + 1})
```

Two ways:
- **commit before process** → at-most-once (lose on crash).
- **commit after process** → at-least-once (duplicate on crash).

Exactly-once requires more (see §8).

### 5.4 Offsets and lag
**Consumer lag** = `latest_offset - committed_offset`. The KPI of any Kafka deployment. Monitor it; alert when it grows.

```bash
kafka-consumer-groups.sh --describe --group inventory --bootstrap-server localhost:9092
```

---

## 6. Producers in Depth

### 6.1 Batching for throughput
Producer collects messages, batches them, sends. Knobs:
- `linger.ms` — wait this long before sending a batch (default 0; bump to 5–20 for higher throughput).
- `batch.size` — max bytes per batch (default 16 KB; bump to 64–256 KB).
- `compression.type` — `gzip`, `snappy`, `lz4`, `zstd`. `zstd` typically wins on ratio and speed.

A producer with `linger.ms=20, batch.size=256000, compression=zstd` is often **10× the throughput** of defaults.

### 6.2 Idempotent producer
`enable.idempotence=true` makes the producer assign sequence numbers per partition. The broker dedupes retries. **Required for exactly-once.** Recommend always on.

### 6.3 Transactions
Producer can write to multiple partitions / topics atomically. `enable.idempotence=true` + `transactional.id=...`. Used for Kafka Streams' exactly-once and for outbox-style atomic publishes.

---

## 7. Throughput, Latency, Scale

Realistic numbers from production:

| Setup | Throughput | Latency |
|---|---|---|
| Single broker, defaults | 50k msgs/sec | 1–5 ms |
| 3-broker cluster, tuned producer | 500k+ msgs/sec | 2–10 ms |
| Large cluster (LinkedIn-style) | millions msgs/sec | 5–50 ms |
| Kafka over network at low latency | < 2 ms p99 | depends on `acks` |

LinkedIn famously runs Kafka at 7 trillion messages per day. Netflix uses Kafka as the spine of their data infrastructure. Uber, Airbnb, Stripe — all Kafka shops.

Bottlenecks in order:
1. **Disk I/O** on producer-heavy workloads.
2. **Network bandwidth** across AZs (especially with `acks=all`).
3. **GC** on consumers in JVM.
4. **Single-partition hot spots**.

---

## 8. Exactly-Once Semantics (EOS)

Kafka's "exactly-once" is **within Kafka**: producer writes, consumer reads, results back to Kafka, atomically. End-to-end (Kafka → external DB) requires the consumer to participate (idempotent writes, transactional sink connectors).

Requirements:
- `enable.idempotence=true` on producer.
- `transactional.id` on producer.
- `isolation.level=read_committed` on consumer.
- Use Kafka transactions to commit consumer offset and produce result in one atomic step.

In Kafka Streams: just set `processing.guarantee=exactly_once_v2`. Done.

Caveat: it's slower (~30–50% throughput hit). For most workloads, **idempotent at-least-once** is the better trade. See [Delivery Guarantees →](./delivery-guarantees.md).

---

## 9. Replication and Durability

```
   producer ──► leader (broker A) ──► follower (B)
                                 └─► follower (C)
   acks=all waits for B and C in ISR
```

Settings that matter:
- `replication.factor=3` — three copies.
- `min.insync.replicas=2` — write requires at least 2 ISRs. Survives 1 broker failure.
- `acks=all` (producer) — writes wait for all ISRs.
- `unclean.leader.election.enable=false` — never elect an out-of-sync replica as leader. Loses availability under partition; preserves data.

The trade: `min.insync.replicas=2, replication.factor=3, acks=all` is the safe default. Lose any one broker, no data loss.

---

## 10. KRaft vs ZooKeeper

Old Kafka needed **ZooKeeper** for cluster metadata. ZooKeeper was a separate cluster, hard to operate.

**KRaft** (Kafka Raft Metadata) — built-in Raft-based consensus, replacing ZooKeeper. GA in Kafka 3.3 (2022). Default in 3.5+. ZooKeeper removed in 4.0.

If you're building a new Kafka deployment in 2026, **use KRaft mode**. Operationally far simpler.

---

## 11. Connect, Streams, Schema Registry

The broader ecosystem you'll touch:

### Kafka Connect
Framework for moving data between Kafka and other systems via connectors. Source connectors (Postgres → Kafka via Debezium) and sink connectors (Kafka → Elasticsearch, S3, BigQuery). Distributed, restartable.

### Kafka Streams
A Java library for stream processing on top of Kafka. Topology DSL, stateful operations, exactly-once. Lighter than Flink; lives inside your service.

### Schema Registry
Confluent's schema registry stores Avro/Protobuf/JSON schemas. Producers and consumers reference schemas by ID. Enforces compatibility (backward, forward, full). **Essential for any non-trivial Kafka deployment** — events live forever; schemas must evolve safely.

### MirrorMaker 2
Replication across Kafka clusters (e.g., cross-region). Tracks offsets and metadata.

---

## 12. Operational Concerns

### Things you watch
- **Consumer lag** per group per partition.
- **Under-replicated partitions** (`UnderReplicatedPartitions` > 0 = trouble).
- **Offline partitions** (`OfflinePartitionsCount`).
- **Request latency** (produce, fetch).
- **Disk usage** per broker.
- **Network throughput** per broker.
- **GC pauses** (still relevant for Kafka brokers).

### Things you tune
- **Producer**: `linger.ms`, `batch.size`, `compression.type`, `acks`, `enable.idempotence`.
- **Consumer**: `fetch.min.bytes`, `max.poll.records`, `enable.auto.commit`, `partition.assignment.strategy`.
- **Broker**: `num.io.threads`, `num.network.threads`, `log.segment.bytes`, `replica.fetch.max.bytes`, `min.insync.replicas`.
- **Topic**: `partitions`, `replication.factor`, `retention.ms`, `cleanup.policy`.

### Things that bite you
- **Topic auto-creation** with bad defaults (e.g., 1 partition). Disable in prod.
- **Long GC pauses** trigger consumer rebalances.
- **Rolling broker restart with no leader-rebalance pause** → spike of leadership transitions.
- **One huge consumer** that bottlenecks → lag grows; add more.
- **Cross-AZ Kafka traffic** dominates cost. Use rack-aware replica placement and `client.rack` for region-aware fetch.

---

## 13. Kafka vs Alternatives

| Aspect | Kafka | Pulsar | Redpanda | Kinesis |
|---|---|---|---|---|
| Storage | Per-broker disk | BookKeeper (separate) | Per-broker disk | Managed |
| Tiered storage | yes (3.6+) | native | yes | implicit |
| Language | Java (JVM) | Java | C++ | managed |
| Consensus | KRaft (Raft) | ZooKeeper / Oxia | KRaft / Raft | AWS internal |
| Latency p99 | 5–50 ms | similar | 1–10 ms | 10–100 ms |
| Compatibility | Kafka protocol | own + Kafka via proxy | Kafka wire-compat | own SDK |
| Ops | classic | complex | simple (single binary) | zero (managed) |
| Cost | self-host | self-host | self-host or BYOC | per-shard |

For most teams in 2026:
- **Kafka** if you have JVM expertise and need ecosystem (Connect, Streams).
- **Redpanda** if you want simpler ops, no JVM.
- **Pulsar** if you need geo-replication or message-level features.
- **Kinesis / MSK** if you're deep in AWS and want managed.

---

## 14. Worked Example: Order Events Pipeline

Service emits order events; downstream consumers: inventory, fulfillment, analytics, fraud detection.

### Topic design
- `orders.events` topic.
- Schema: Avro, registered, with backward-compatibility.
- Partition key: `order_id`. Same order's events in order.
- 24 partitions (plan for ~24 consumer parallelism per group).
- Replication factor: 3.
- Retention: 7 days.

### Producer
- `acks=all`, `enable.idempotence=true`, `compression=zstd`.
- Linger 20 ms, batch 128 KB.
- Throughput: ~50k events/sec.

### Consumer groups
- `inventory` group, 12 consumers (2 partitions each).
- `analytics` group, 24 consumers (1 partition each).
- `fraud-detector` group, 4 consumers (6 partitions each — heavier per-event work).

### Schema evolution
- Avro schema registered in Schema Registry.
- Compatibility: `BACKWARD` — new schemas can read old data.
- Producer always references the latest schema by id.
- Consumers handle multiple versions.

### Monitoring
- Lag per consumer group per partition.
- Producer error rate.
- Under-replicated partitions = 0 alarm.
- Disk usage per broker < 70% alarm.

---

## 15. Common Mistakes

- **Bad partition key.** Hot partition, lost throughput. Fix early; you can only add partitions.
- **`acks=1` or `acks=0` for important data.** Lose writes on broker failure.
- **Auto-commit** while processing → lose data on crash. Manual commit after processing.
- **No schema registry.** Producers change format; consumers break next day.
- **Forever retention with massive volume.** Disk fills, brokers die.
- **One consumer per service consuming massive volume.** Scale consumer groups; add partitions.
- **Trying to use Kafka as a queue.** Works (consumer group), but if you want simple work-queues, RabbitMQ / SQS may be friendlier.
- **Cross-AZ traffic ignored.** Bill shock. Tune for rack awareness.
- **No DLT (dead-letter topic).** Bad messages block consumption. Send to a DLT and continue.
- **Skipping idempotent producer.** Retries duplicate. Always idempotent.
- **Treating Kafka as a database.** It's a log. Use it as the log; project to a DB for queries.

---

## 16. Cheat Card

```
PRIMITIVES   broker · topic · partition · offset
             producer · consumer · consumer group

CORE PROMISE durable append-only log
             per-partition order, replayable, multi-consumer

KEY DECISION partition key — determines ordering, parallelism, skew
             prefer user_id / tenant_id / entity_id
             never country, timestamp, null

DURABILITY   replication.factor=3, min.insync.replicas=2,
             acks=all, enable.idempotence=true

CONSUMERS    consumer group = parallelism unit
             partitions per group ÷ consumers = per-consumer parallelism
             always manual commit AFTER processing
             watch consumer lag

EOS          idempotent producer + transactions + read_committed
             only within Kafka end-to-end (or transactional sink)

OPS          monitor lag, ISR, disk, request latency
             KRaft instead of ZooKeeper for new clusters

PITFALLS     bad partition key, no schema registry, auto-commit,
             cross-AZ cost, no DLT, treating Kafka as a DB

RULE         Kafka is a log. The log is the source of truth.
             Other stores are projections.
```

---

## 17. Resources

### Books
- *Kafka: The Definitive Guide* — Narkhede, Shapira, Palino (Confluent founders, essential).
- *Kafka Streams in Action* — Bill Bejeck.
- *Designing Data-Intensive Applications* — Kleppmann (Ch 11).

### Documentation
- **Apache Kafka docs**: <https://kafka.apache.org/documentation/>
- **Confluent Platform docs**: <https://docs.confluent.io/>
- **Confluent Schema Registry**: <https://docs.confluent.io/platform/current/schema-registry/index.html>
- **Kafka Connect**: <https://docs.confluent.io/platform/current/connect/index.html>

### Articles
- "The Log: What every software engineer should know" — Jay Kreps.
- "Building a Streaming Platform at LinkedIn" — Jay Kreps and team.
- "Effective Kafka" — Confluent blog series.
- "Kafka at Netflix" — Netflix Tech Blog.
- "Why we don't run Kafka" — various counter-examples.

### Videos
- Tim Berglund's Apache Kafka series.
- Confluent's "Streams 101 / 102" courses.
- ByteByteGo — "Kafka Explained".

### Tools
- **Kafka**, **Confluent Platform**, **Redpanda**, **MSK** (managed AWS).
- **kcat** (formerly kafkacat) — CLI Swiss Army knife.
- **Burrow** — consumer lag monitoring.
- **CMAK / Kafka UI** — web management.
- **Kafka Connect**, **Debezium** for CDC.
- **Kafka Streams**, **ksqlDB** for stream processing.

### Adjacent reading
- [Queue vs Pub/Sub vs Stream →](./queue-vs-pubsub-vs-stream.md)
- [Message Brokers →](./message-brokers.md)
- [Delivery Guarantees →](./delivery-guarantees.md)
- [Dead Letter Queues →](./dead-letter-queues.md)
- [Stream Processing →](./stream-processing.md)
- [Event Sourcing →](./event-sourcing.md)
- [CDC →](../04-databases/cdc.md)
- [Outbox Pattern →](./outbox-pattern.md)

---

*Previous:* [← Queue vs Pub/Sub vs Stream](./queue-vs-pubsub-vs-stream.md)  |  *Next:* [Message Brokers →](./message-brokers.md)
