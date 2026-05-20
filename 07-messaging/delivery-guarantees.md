# Delivery Guarantees (At-Most-Once, At-Least-Once, Exactly-Once)

> **TL;DR** — Three delivery semantics for messaging systems. **At-most-once**: messages may be lost; never duplicated. Fast and cheap, fits non-critical metrics. **At-least-once**: messages never lost; may be duplicated. The de-facto default for most brokers. Consumers MUST be idempotent. **Exactly-once**: never lost, never duplicated. Famously hard; "exactly-once messaging" as marketed often means "exactly-once **processing** with consumer cooperation" — not a transport-level guarantee. Kafka offers exactly-once **within Kafka** via the idempotent producer + transactions; end-to-end (Kafka → external DB) requires the consumer to participate. The pragmatic truth: **at-least-once + idempotent consumers** is what real systems use. Don't fight for true exactly-once; design for duplicates.

---

## 1. The Three Promises

```
AT-MOST-ONCE     send → maybe arrives → never duplicated
                  Losing data is acceptable.

AT-LEAST-ONCE    send → arrives at least once → may duplicate
                  Losing data isn't acceptable; duplicates are.

EXACTLY-ONCE     send → arrives exactly once
                  Neither losing nor duplicating acceptable.
                  Real implementations require effort on both sides.
```

The difference comes from what happens at three points:
1. Producer → broker.
2. Broker storage.
3. Broker → consumer + consumer ack.

Each step can fail and retry. Each retry creates a duplicate-vs-lost dilemma.

---

## 2. The Two Generals Problem (Why It's Hard)

Imagine producer P sends a message to broker B over an unreliable channel.

- **P sends, gets no ack** — did B receive it? Don't know.
- If P retries → may duplicate (if B did receive).
- If P doesn't retry → may lose (if B didn't receive).

You cannot resolve this with messages alone. **No finite protocol gives perfect agreement over a lossy channel.** This is the **Two Generals Problem**, formalized in 1975.

Real systems sidestep it with:
- **Idempotency** — duplicate sends produce the same effect.
- **Dedup IDs** — broker / consumer recognizes a retry as the same message.
- **Transactions** — atomic commits across multiple writes.

These mechanisms turn "we can't guarantee one-delivery" into "duplicates don't matter."

---

## 3. At-Most-Once in Detail

The producer sends, doesn't wait for confirmation (or doesn't retry on failure).

```
producer ──fire-and-forget──► broker
   no ack required, no retry
```

If the message is lost in the network, broker crash, or producer crash mid-send — the message is gone. The consumer never sees it.

### Examples
- UDP-based metrics (StatsD).
- High-volume telemetry where loss < 1% is acceptable.
- `acks=0` in Kafka.

### Pros
- Fastest (no waiting for acks).
- Cheapest (no retry storage).
- Simplest implementation.

### Cons
- Data loss.
- Hard to know what was lost.

### When to use
- Metrics and observability data where some loss is tolerated.
- High-volume "fire-and-forget" scenarios.
- Cases where the value of any single message is low.

---

## 4. At-Least-Once in Detail

The producer sends, waits for an ack, retries on failure or timeout. The consumer processes, then acks; if it crashes before ack, the broker re-delivers.

```
producer ──► broker (waits for ack)
   if no ack → retry → may duplicate
   ↓ ack received
broker ──► consumer ──► process ──► commit/ack
            if crash before ack → re-deliver → duplicate
```

### Where duplicates come from
- **Producer-side retry**: producer sent, broker received, ack lost; producer retries; broker stores twice.
- **Consumer-side**: consumer processes, then dies before commit; broker re-delivers; consumer processes again.
- **Rebalance** in consumer groups: partition reassigned mid-process; new consumer re-reads.

### Examples
- Default Kafka behavior (with `acks=all` + retries).
- Default RabbitMQ ack-mode.
- SQS Standard (with at-least-once delivery semantics).
- Google Pub/Sub default.

### Pros
- Never lose data.
- Standard for most brokers.
- Robust to almost any failure.

### Cons
- Consumers will see duplicates.
- Must be idempotent (or have dedupe logic).
- Slightly higher latency than at-most-once.

### When to use
- Default for most use cases. Combined with idempotent consumers, gives "effectively exactly-once" processing.

This is what 90% of production systems use.

---

## 5. Exactly-Once in Detail

Each message arrives at the consumer **exactly once** — never lost, never duplicated.

Famously difficult because of the Two Generals Problem. The strict version is impossible over an unreliable channel. **Practical "exactly-once"** is achieved via cooperation:
- **Idempotent producer** — broker dedupes producer retries via sequence numbers.
- **Transactional producer** — write to multiple partitions / topics atomically.
- **Transactional consumer** — commit offset and write output in one transaction.
- **Idempotent sink** — external system rejects duplicate writes (dedupe ID, upsert by key).

### Kafka's Exactly-Once Semantics (EOS)

Within Kafka, EOS works like this:

```
producer:
  enable.idempotence = true
  transactional.id = "my-app"

  beginTxn()
  send(record_to_topic_A)
  send(record_to_topic_B)
  commitTxn()    # atomic: both records visible together or neither

consumer:
  isolation.level = "read_committed"   # skip uncommitted transactions
```

The producer assigns a unique ID to each message; the broker dedupes. Transactions atomically commit a batch across partitions. Consumers in `read_committed` mode skip aborted transactions.

For consume-process-produce loops (Kafka Streams), the framework wraps the consume-offset-commit + produce in one transaction. Result: even on consumer crash, no duplicate or lost output.

This is **exactly-once within Kafka.** It does NOT extend to external sinks unless the sink participates.

### End-to-end exactly-once

For Kafka → external DB to be exactly-once, the DB write must be either:
- **Idempotent** (upsert by message ID; ON CONFLICT DO NOTHING).
- **Transactional** (commit DB write + Kafka offset together via a connector or 2PC).

Kafka Connect's sinks with idempotent writes (e.g., the JDBC sink with PK) approximate this.

### Pros (where it really works)
- Strong correctness guarantee within the system.
- Removes the "must be idempotent" burden in some consumer code.

### Cons
- ~20–50% throughput hit on Kafka (transactional overhead).
- More complex consumer logic.
- Doesn't extend to all external systems.
- Easy to misconfigure and silently get at-least-once anyway.

### When to use
- Kafka Streams pipelines where the consume-and-produce cycle must be exact.
- High-correctness money / billing / accounting flows where building idempotent consumers is harder than enabling EOS.

The pragmatic alternative: **at-least-once + idempotent writes** is simpler, faster, and equally correct from the end-user perspective.

---

## 6. The Reality: At-Least-Once + Idempotency

Most production systems standardize on:

```
Producer: at-least-once delivery (retries on failure).
Broker:   durable, replicated, at-least-once retention.
Consumer: idempotent processing (handles duplicates as no-ops).
```

The consumer's job: **applying the same message twice produces the same final state**.

### Strategies for idempotent consumers

#### 1. Natural idempotency
Some operations are naturally idempotent.
```
SET status = 'paid'        # idempotent
DELETE FROM x WHERE id=42  # idempotent
INCR counter               # NOT idempotent
```

Design your operations to be naturally idempotent when possible.

#### 2. Dedup key / processed-ID table
```sql
INSERT INTO processed_events (event_id) VALUES ('evt_123');
-- if duplicate INSERT → unique violation → skip
```

Keep a table (or Redis set) of `processed_event_ids`. Check before processing.

```python
def handle(event):
    if cache.set(f"processed:{event.id}", 1, nx=True, ex=86400):
        process(event)
    # else: already processed; skip
```

Bound the table by time (events older than N days can't appear).

#### 3. Versioned writes (optimistic concurrency)
```sql
UPDATE orders SET status='paid', version=5
WHERE id=42 AND version=4
```

If the version doesn't match (because we already updated to 5), no rows are affected. Idempotent.

#### 4. Upsert with conflict resolution
```sql
INSERT INTO accounts (id, balance) VALUES (42, 100)
ON CONFLICT (id) DO UPDATE SET balance = excluded.balance
```

#### 5. Functional / pure operations
Compute the output deterministically from the input. Two identical inputs produce identical outputs. Reapplying is a no-op.

---

## 7. Comparison Table

| Guarantee | Loss | Dup | Throughput | Complexity | When to use |
|---|---|---|---|---|---|
| At-most-once | possible | none | highest | lowest | telemetry, tolerable loss |
| At-least-once | none | possible | high | medium (idempotent consumers) | default for most systems |
| Exactly-once (within broker) | none | none | reduced | high | Kafka Streams, billing |
| Exactly-once (end-to-end) | "none" | "none" | reduced | very high | rare; requires sink cooperation |

---

## 8. Failure Modes by Scenario

### Producer side
- **Send fails, no retry** → loss (at-most-once).
- **Send fails, retry without dedup** → duplicate (at-least-once).
- **Send fails, retry with idempotent producer** → no duplicate (Kafka EOS step 1).

### Broker side
- **Single broker fails before persisting** → loss if no replication.
- **`acks=all` with `min.insync.replicas=2`** → durable (no loss on single broker failure).
- **Cluster goes down entirely** → loss only on data not yet replicated.

### Consumer side
- **Commits before processing** → loss on crash (at-most-once-ish).
- **Commits after processing** → duplicate on crash (at-least-once).
- **Commits in same transaction as output** → no duplicate in output (exactly-once with EOS).

### Network
- **ACK lost mid-flight** → producer retry → duplicate.
- **Partition during consume** → re-deliver → duplicate.

---

## 9. Per-Broker Defaults

| Broker | Default delivery | How to get at-least-once | EOS available? |
|---|---|---|---|
| **Kafka** | at-least-once | `acks=all`, retries, manual commit | yes (idempotent + transactions) |
| **RabbitMQ** | at-least-once (with `ack`) | publisher confirms + ack consumers | no native EOS |
| **SQS Standard** | at-least-once | always | no |
| **SQS FIFO** | exactly-once processing (5-min dedup window) | with dedup ID | yes-ish (bounded window) |
| **Google Pub/Sub** | at-least-once | always | yes (recent feature, opt-in) |
| **Azure Service Bus** | at-least-once / at-most-once | configurable | sessions for ordering |
| **NATS JetStream** | at-least-once (with ack) | always | exactly-once available |

For most brokers: at-least-once is the default; exactly-once is opt-in.

---

## 10. Worked Example: Payment Processing

A payment system charges a user. The job: ensure the user is **never double-charged**, even if any component fails.

### Naive approach
```
producer publishes ChargePayment event
consumer processes → calls Stripe → returns
```

What can go wrong?
- Consumer processes, charges Stripe, dies before committing offset. Broker re-delivers. Stripe charged twice.
- Consumer commits offset, then dies before charging. Money not charged. Customer not billed.

### Correct approach (at-least-once + idempotent)
```
ChargePayment event includes idempotency_key (unique per payment)

consumer:
  if cache.set("processed:{key}", 1, nx=True, ex=86400):
    response = stripe.charges.create(
        amount, source, idempotency_key=key)
    record in DB
  commit offset
```

Two layers of idempotency:
1. Our consumer's local check (skips re-processing).
2. Stripe's idempotency key (Stripe itself dedupes within 24h).

Result: at-least-once delivery + idempotency = effectively exactly-once outcome.

### Why not EOS?
You could use Kafka's exactly-once if both producer and consumer are Kafka-only. But the sink is Stripe (external HTTP API). Stripe's idempotency-key is the actual safeguard. EOS gives you nothing here.

The pattern: **idempotency at every external boundary, at-least-once over the wire**.

---

## 11. Idempotency Key Design

A good idempotency key is:
- **Unique** per logical operation.
- **Stable** across retries (the retry must use the same key).
- **Caller-generated** (so the caller can retry safely).
- **Bounded** (eventually expires — typically 24h–7d).

Examples:
- `payment_{order_id}_{charge_attempt}` for payments.
- `notification_{user_id}_{event_id}` for emails.
- `outbox_{outbox_row_id}` for outbox publishing.

Avoid using broker-internal IDs (offset, message ID) as idempotency keys — they're not stable across reprocessing or different brokers.

See [Idempotency →](../03-apis/idempotency.md).

---

## 12. Ordering and Guarantees

Delivery semantics interact with ordering.

- **At-most-once** with retries: ordering preserved.
- **At-least-once** with retries: duplicates can arrive out of order with the original. Per-partition order still holds at the broker level.
- **Exactly-once** in Kafka: ordering preserved within a transaction.

If you need both ordering and at-least-once, partition by key (same key → same partition → ordered) and accept duplicates within that order.

---

## 13. Common Mistakes

- **"Exactly-once" is a marketing term.** Verify what "exactly-once" actually means in your stack. Usually it's "exactly-once processing" with broker + consumer cooperation, not transport-level.
- **At-most-once for important data.** Lost messages, untraceable.
- **At-least-once without idempotent consumers.** Duplicates cause real harm: double-charges, double-emails, corrupted counters.
- **Committing offset before processing.** Loss on crash.
- **Committing offset after processing without a transaction.** Duplicate if crash between process and commit. Use EOS or idempotent sink.
- **No idempotency key on external API calls.** Retries duplicate side effects.
- **Misconfigured Kafka EOS.** Producer not idempotent, transactional.id reused, isolation.level not `read_committed`. Result: silent at-least-once.
- **Replaying without dedup.** Reprocessing the event log "from scratch" runs all side effects again. Either consumers are idempotent or you're in trouble.
- **Trusting "exactly-once" cross-system.** It works inside Kafka. Across Kafka → DB requires sink cooperation.

---

## 14. Cheat Card

```
AT-MOST-ONCE   send, no retry, no ack
                accept loss; fastest
                use: telemetry, tolerable-loss feeds

AT-LEAST-ONCE  send with retry + ack; consumer commits
                duplicates possible
                use: default for production; pair with idempotency

EXACTLY-ONCE   within Kafka: idempotent producer + transactions +
                read_committed consumer
                end-to-end: requires sink cooperation (dedup/upsert)
                use: Kafka Streams; rare beyond that

PRACTICAL      at-least-once + idempotent consumers
                covers ~all real systems

IDEMPOTENCY    natural / dedup-key / version / upsert / pure functions

EXTERNAL APIS  always pass an idempotency key (Stripe, etc.)

PITFALLS       relying on at-most-once for critical data,
                no idempotent consumers, commit-before-process,
                false confidence in vendor "exactly-once" claims

RULE           Design for duplicates. They will come.
```

---

## 15. Resources

### Papers
- "The Two Generals Problem" (1975).
- "Exactly-Once Semantics in Apache Kafka" — Confluent / KIP-98.

### Articles
- "You Cannot Have Exactly-Once Delivery" — Tyler Treat.
- "Exactly-once delivery in Apache Kafka" — Jay Kreps, Confluent blog.
- "Idempotency tokens and you" — various engineering blogs (Stripe).
- "Building reliable distributed systems with at-least-once delivery" — AWS Builders Library.

### Books
- *Designing Data-Intensive Applications* — Kleppmann (the consistency / replication chapters).
- *Kafka: The Definitive Guide*.

### Documentation
- **Kafka EOS**: <https://docs.confluent.io/platform/current/streams/concepts.html#processing-guarantees>
- **SQS FIFO dedup**: <https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/FIFO-queues-exactly-once-processing.html>
- **Google Pub/Sub exactly-once**: <https://cloud.google.com/pubsub/docs/exactly-once-delivery>
- **Stripe idempotency**: <https://docs.stripe.com/api/idempotent_requests>

### Videos
- Confluent — "Exactly-once Semantics in Kafka".
- ByteByteGo — "Delivery Guarantees".
- Martin Kleppmann — distributed systems lectures.

### Adjacent reading
- [Kafka Deep Dive →](./kafka.md)
- [Message Brokers →](./message-brokers.md)
- [Dead Letter Queues →](./dead-letter-queues.md)
- [Outbox Pattern →](./outbox-pattern.md)
- [Idempotency →](../03-apis/idempotency.md)
- [Idempotent Operations & Retries →](../11-reliability/idempotency-retries.md)
- [Retry, Timeout, Backoff →](../11-reliability/retry-timeout-backoff.md)

---

*Previous:* [← CQRS](./cqrs.md)  |  *Next:* [Dead Letter Queues →](./dead-letter-queues.md)
