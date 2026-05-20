# Dead Letter Queues

> **TL;DR** — A **dead letter queue (DLQ)** is where messages go when they fail to be processed after exhausting retries. Without one, a single bad message can block consumption forever, grow the backlog, and topple your service. The DLQ is the **safety valve**: take the bad message out of the main flow, alert humans, keep processing the rest. The hard parts are: deciding **what counts as a permanent failure** (vs a transient one worth retrying), **what metadata to attach** (original topic, attempts, last error, trace), **how long to keep messages there** (until a human looks, with cap), and **how to replay them** safely once fixed. Every queue, every stream consumer group, every Lambda trigger needs a DLQ. Treat the DLQ depth as a P1 metric.

---

## 1. Why DLQs Exist

Without a DLQ:

```
   queue ──► consumer ──► error ──► retry ──► error ──► retry ──► ...
                                                                  forever
                                       ▼
                       backlog grows; consumer never progresses;
                       fresh messages wait behind the poison one
```

A single "poison message" — bad schema, bad data, a bug — can block the entire consumer group. The queue grows. Lag alarms fire. Other healthy messages wait. Eventually the system fails entirely.

With a DLQ:

```
   queue ──► consumer ──► error ──► retry × N ──► STILL FAILING
                                                       │
                                                       ▼
                                                    DLQ
                                                  (paged human)
   queue continues; bad message moved out of the path
```

The principle: **poison messages must not block fresh ones.**

---

## 2. The Anatomy of a Failure

Three failure modes determine the right response:

### 2.1 Transient
A network blip, a dependency timeout, a brief upstream 503. Will likely succeed on retry.
- **Action**: retry with backoff.

### 2.2 Permanent (poison)
Bad payload, schema mismatch, business invariant violated, missing required field. Will fail every time you try.
- **Action**: move to DLQ.

### 2.3 Slow / dependent
Downstream service down for hours. Retrying immediately wastes work; eventually it'll come back.
- **Action**: long backoff or pause consumer.

The DLQ catches case 2 (and sometimes 3 after enough retries).

---

## 3. Detecting Permanent Failure

The standard heuristic: **retry N times with exponential backoff**. If still failing, treat as permanent → DLQ.

```python
def process(msg):
    try:
        do_work(msg)
        commit(msg)
    except TransientError:
        nack(msg)               # broker re-delivers
    except PermanentError:
        send_to_dlq(msg)
        commit(msg)
    except Exception as e:
        if msg.attempts < MAX_RETRY:
            requeue_with_delay(msg, backoff(msg.attempts))
        else:
            send_to_dlq(msg, last_error=str(e))
            commit(msg)
```

Smarter setups classify errors:
- 4xx-equivalents (bad input) → DLQ immediately. No point retrying.
- 5xx-equivalents (server / transient) → retry N times.
- Schema parse errors → DLQ immediately.

Don't burn 30 retries on something that can't possibly succeed.

---

## 4. How Brokers Implement DLQs

### Kafka
No native DLQ; the pattern is a **dead-letter topic** (DLT). Consumer publishes failed messages to a `<topic>.DLT` topic.

Frameworks help:
- **Kafka Streams** has built-in `DeserializationExceptionHandler` and `ProductionExceptionHandler`.
- **Spring Kafka** has a `DeadLetterPublishingRecoverer`.
- **Kafka Connect** has `errors.deadletterqueue.topic.name`.

```yaml
# Kafka Connect sink config
errors.tolerance: all
errors.deadletterqueue.topic.name: orders-dlt
errors.deadletterqueue.context.headers.enable: true
```

### RabbitMQ
Native dead-letter exchanges (DLX). Configure on the queue:

```
x-dead-letter-exchange: dlx
x-dead-letter-routing-key: failed
x-message-ttl: 60000        # also dead-letter on TTL expiry
```

When a message is `nack`'d (with requeue=false), exceeds TTL, or is dropped due to queue length, it's routed to the DLX.

### AWS SQS
Native redrive policy:

```json
{
  "RedrivePolicy": {
    "deadLetterTargetArn": "arn:aws:sqs:...:my-dlq",
    "maxReceiveCount": 5
  }
}
```

After 5 receives without successful delete, the message is automatically moved to the DLQ. The DLQ is just another SQS queue.

SQS also has **redrive from DLQ** — a managed API to replay DLQ messages back to the main queue once you fix the bug.

### Google Pub/Sub
Native dead-letter topics:

```
subscription:
  dead_letter_policy:
    dead_letter_topic: projects/.../topics/orders-dl
    max_delivery_attempts: 5
```

### Azure Service Bus
Built-in DLQ as a subqueue of every queue/topic subscription. Access via `<queue>/$DeadLetterQueue`. Messages move there on max-delivery-count exceeded, TTL expiration, or explicit dead-lettering.

### Lambda
For asynchronous Lambda triggers, configure a DLQ (SQS or SNS) for failed invocations. Stream-based triggers (Kinesis, DynamoDB Streams) have `OnFailure` destinations for the same purpose.

---

## 5. What to Attach (Metadata)

A DLQ message without context is useless. Always include:

```json
{
  "original_payload": "...",
  "original_topic": "orders",
  "original_partition": 5,
  "original_offset": 123456,
  "first_attempt_at": "2026-05-19T14:00:00Z",
  "last_attempt_at": "2026-05-19T14:05:00Z",
  "attempts": 5,
  "last_error": "JSONDecodeError: unexpected character at line 3",
  "last_error_stacktrace": "...",
  "consumer_group": "inventory",
  "consumer_host": "pod-inventory-7c4d8",
  "trace_id": "abc-123",
  "message_id": "msg-456"
}
```

Some of this goes in message headers (Kafka, Pub/Sub support headers). Some in the body wrapping the original payload.

The goal: a human looking at one DLQ entry should be able to **reproduce the failure** without context-switching to logs.

---

## 6. Triage and Replay

A DLQ without operators is just a graveyard. The operational flow:

### 6.1 Alerting
Alert on DLQ depth > N or growth rate. Page someone.

### 6.2 Triage
For each (or each cluster of) messages:
- What was the error?
- Was the payload bad (producer bug) or the consumer bad (consumer bug)?
- How many affected?

### 6.3 Fix
- Producer bug → patch and prevent recurrence.
- Consumer bug → patch and prepare replay.
- Bad payload from external source → maybe drop, maybe correct.

### 6.4 Replay
- If the cause is fixed and messages are now processable, **replay** them back to the main queue.
- If the data is corrupted beyond fix, **drop** (with audit log).

```
# SQS
aws sqs start-message-move-task \
    --source-arn arn:aws:sqs:...:my-dlq \
    --destination-arn arn:aws:sqs:...:my-queue \
    --max-number-of-messages-per-second 10

# Kafka — script that reads DLT and produces back to main topic
```

### 6.5 Audit
Every DLQ resolution should leave a record. What was the cause? How many messages? Was anything lost?

---

## 7. Common DLQ Anti-Patterns

### 7.1 No DLQ
The most common. A bad message blocks the consumer; investigation takes hours; backlog grows.

### 7.2 DLQ with no alerts
Bad messages pile up silently; first you hear about it is a data inconsistency a week later.

### 7.3 Infinite retries with no DLQ exit
"We retry until it succeeds." The retry loop tortures the broker and pollutes logs.

### 7.4 DLQ as a permanent dump
Messages accumulate forever; nobody triages. The DLQ becomes its own backlog problem.

### 7.5 Replaying without fixing
"Just replay; let's see if it works." Replays the same failure. Five retries × 1000 messages × DLQ loop. Pollutes.

### 7.6 Sending success metrics to DLQ
A consumer error handler that catches *everything* including successful business logic that returned a non-empty result. The DLQ fills with "successfully processed but I wasn't sure" messages.

### 7.7 Loss in the move
If "move to DLQ" isn't transactional with "commit offset," you can either lose the message (move failed, offset committed) or duplicate (move succeeded, offset not committed). Use broker-native DLQ where available.

---

## 8. DLQ Capacity and Retention

### How big?
DLQ depth should be small in steady state — ideally zero. A growing DLQ is a signal.

Plan capacity for:
- A burst of bad messages during a producer bug (could be thousands per second).
- Multi-day retention while a fix ships.

Typical config: retention 7–14 days, alert on depth > 10.

### Compression / archive
For DLQ messages older than a few days, archive to S3 or equivalent. Keep the DLQ small for human-scale triage.

---

## 9. Per-Type Sub-DLQs

For high-volume systems, a single DLQ becomes hard to triage. Split by error class:

- `orders.dlq.parse_error`
- `orders.dlq.business_invariant_violation`
- `orders.dlq.downstream_unavailable`

Each gets its own retention and resolution playbook. Parse errors usually mean producer bug; business invariants mean data quality; downstream means waiting.

---

## 10. Worked Example: An Order Processing Pipeline

Topic: `orders`. Consumer group: `fulfillment`. Producer: `order-service`. Downstream: payment, inventory, fulfillment.

### Setup
- Main topic: `orders` (Kafka, retention 7 days).
- DLT: `orders.dlt` (Kafka, retention 30 days).
- Per-class sub-topics: `orders.dlt.parse_error`, `orders.dlt.payment_failed`, etc.

### Consumer logic
```python
def handle(record):
    try:
        order = parse(record)  # may raise ParseError
    except ParseError as e:
        produce("orders.dlt.parse_error", record.value, headers={
            "error": str(e),
            "attempts": "1",
            "first_seen": now(),
            "original_topic": record.topic,
            "original_partition": record.partition,
            "original_offset": record.offset,
        })
        return commit(record)

    try:
        process(order)
    except RetryableError:
        record.attempts += 1
        if record.attempts >= MAX_RETRY:
            produce("orders.dlt.processing_failed", record.value, headers=...)
            return commit(record)
        requeue_with_delay(record, backoff(record.attempts))
    except Exception as e:
        produce("orders.dlt.unknown", record.value, headers={"error": str(e), ...})
        return commit(record)
```

### Monitoring
- `orders.dlt.*` depth dashboards.
- Alert on depth > 10 in any DLT.
- Page on growth > 100/min.

### Triage runbook
- Parse errors → ping producer team; usually a schema breakage.
- Processing failed → check the consumer error; fix and replay.
- Unknown → page on-call engineer; investigate.

### Replay
- Once fixed, a tool reads from the DLT and produces back to `orders`.
- Replays are bounded (rate-limited) to not crush downstream.
- Audit log records every replay.

---

## 11. DLQ in Stream Processing

For Flink, Spark Streaming, Kafka Streams: errors during processing should:
- Not crash the job (a transient parse error in one record shouldn't kill a multi-day pipeline).
- Send the bad record to a side output or DLT.
- Continue processing the rest.

Flink: side outputs. Kafka Streams: `DeserializationExceptionHandler`, `ProductionExceptionHandler`, with options `CONTINUE` or `FAIL`. Use `CONTINUE` + side output to DLT for resilience.

See [Stream Processing →](./stream-processing.md).

---

## 12. DLQ vs Retry Topic

Some patterns separate **retry topics** from **dead-letter topics**:

```
   orders ──► consumer ──► retry-5s ──► retry-1m ──► retry-10m ──► DLT
                            (5s delay)   (1m delay)   (10m delay)
```

Failed messages move to a retry topic with delay. The retry topic re-feeds the main consumer. After N retries, drop to DLT.

This avoids tight retry loops in the main topic; gives transient failures time to clear; isolates retry traffic from fresh.

Used by: Uber's Cherami, Confluent's recommended Kafka pattern for retries.

---

## 13. Common Mistakes

- **No DLQ configured.** Poison messages eventually take down the consumer.
- **No alerting on DLQ depth.** Bad messages pile up silently.
- **No retry classification.** Burning 30 retries on a parse error.
- **DLQ doesn't include error context.** Useless for debugging.
- **Forgetting to replay.** "We fixed it but never replayed those 500 messages." Data loss.
- **Replaying without fixing.** Same failure, repeated.
- **DLQ with no retention bound.** Grows forever.
- **Single global DLQ for all topics.** Triage becomes impossible.
- **Move-and-commit not atomic.** Lose or duplicate during DLQ move.
- **DLQ → DLQ → DLQ loops.** Bad consumer code can recurse. Add max-attempts in DLT consumers too.
- **Treating all errors as transient.** Even bad payloads get retried; consumer wastes time.

---

## 14. Cheat Card

```
WHAT          a side queue/topic for messages that exhausted retries
              prevents poison messages from blocking consumers

WHY           backlog growth, head-of-line blocking, undetected bugs

DETECT        retry N with backoff; classify errors
              4xx-like → DLQ immediately
              5xx-like → retry then DLQ
              parse-error → DLQ immediately

ATTACH        original payload, topic, partition, offset, attempts,
              last error, trace id, message id

IMPLEMENT
  Kafka       DLT (dead-letter topic); per consumer group
  RabbitMQ    DLX (dead-letter exchange)
  SQS         redrive policy with maxReceiveCount
  Pub/Sub     dead_letter_policy
  Lambda      OnFailure destination

TRIAGE        alert on depth, route to humans
              fix root cause, replay or drop, audit

PER-CLASS     split DLQs by error type for easier triage

PITFALLS      no DLQ, no alerts, no metadata, no replay,
              infinite retries, DLQ → DLQ loops,
              treating all errors as transient

RULE          Every consumer needs a DLQ.
              DLQ depth > 0 is an incident.
```

---

## 15. Resources

### Documentation
- **AWS SQS DLQ**: <https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html>
- **RabbitMQ DLX**: <https://www.rabbitmq.com/dlx.html>
- **Kafka Connect DLQ**: <https://docs.confluent.io/platform/current/connect/concepts.html#dead-letter-queue>
- **Google Pub/Sub DLT**: <https://cloud.google.com/pubsub/docs/dead-letter-topics>
- **Azure Service Bus DLQ**: <https://learn.microsoft.com/en-us/azure/service-bus-messaging/service-bus-dead-letter-queues>

### Articles
- "Designing dead-letter queues" — AWS Architecture Blog.
- "Kafka retry topics and dead-letter topic" — Confluent.
- "How Uber built Cherami" — Uber Engineering.
- "Anti-patterns and patterns for DLQs" — various.

### Videos
- ByteByteGo — "Dead Letter Queues Explained".
- Hussein Nasser — "DLQ patterns".

### Tools
- All major brokers have DLQ support.
- **kafka-console-consumer / kcat** for inspection.
- **AWS SQS message-move-task** for redrive.
- Custom replay tools per pipeline.

### Adjacent reading
- [Delivery Guarantees →](./delivery-guarantees.md)
- [Message Brokers →](./message-brokers.md)
- [Kafka Deep Dive →](./kafka.md)
- [Retry, Timeout, Backoff →](../11-reliability/retry-timeout-backoff.md)
- [Circuit Breaker →](../11-reliability/circuit-breaker.md)
- [Idempotency →](../03-apis/idempotency.md)

---

*Previous:* [← Delivery Guarantees](./delivery-guarantees.md)  |  *Next:* [Stream Processing →](./stream-processing.md)
