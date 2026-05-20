# Outbox Pattern

> **TL;DR** — The **outbox pattern** solves the **dual-write problem**: how do I atomically update my database AND publish an event to a broker? The naive `db.save() ; broker.publish()` is broken — if the broker call fails (or the process crashes between), the DB has the change but the event isn't published. The fix: **write the event into an `outbox` table in the same database transaction as the business change**. A separate process polls (or CDC-streams) the outbox and publishes events to the broker, marking each as sent. Because writes to the business table and the outbox happen in one ACID transaction, you never lose events. Because publishing is at-least-once, consumers must be idempotent. The outbox is the **canonical solution** for reliable event publishing alongside database writes, used by Stripe, Shopify, Wise, and roughly everyone serious about event-driven architecture.

---

## 1. The Problem: Dual Writes

You want to do two things when an order is placed:
1. Insert the order into the orders table.
2. Publish an `OrderPlaced` event for downstream services.

Naive code:

```python
def place_order(order):
    db.save(order)             # (1)
    broker.publish(OrderPlacedEvent(order))  # (2)
```

Failure modes:
- **DB succeeds, broker fails** — order placed; downstream never knows. Inventory not reserved, payment not charged, customer not emailed.
- **DB succeeds, process crashes before (2)** — same thing.
- **Broker succeeds, DB fails (less common)** — downstream told about an order that doesn't exist.
- **Both succeed but in wrong order** — broker fires; consumer reacts before DB row is visible to other readers.

This is the **dual-write problem**. You're trying to commit to two systems in one logical operation, but they don't share a transaction. **You cannot make it reliable with retry loops alone.**

---

## 2. The Outbox: One Transaction, Two Effects

Add an **outbox table** in the same database:

```sql
CREATE TABLE outbox (
  id          UUID PRIMARY KEY,
  aggregate   TEXT NOT NULL,
  type        TEXT NOT NULL,
  payload     JSONB NOT NULL,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  published_at TIMESTAMPTZ NULL
);
```

The write becomes one transaction:

```sql
BEGIN;
  INSERT INTO orders (id, ...) VALUES (...);
  INSERT INTO outbox (id, aggregate, type, payload) VALUES (
    gen_random_uuid(),
    'order',
    'OrderPlaced',
    '{"order_id": "...", "items": [...]}'::jsonb
  );
COMMIT;
```

A separate process publishes outbox rows:

```python
while True:
    rows = db.query("""
        SELECT id, type, payload FROM outbox
        WHERE published_at IS NULL
        ORDER BY created_at
        LIMIT 100
        FOR UPDATE SKIP LOCKED
    """)
    for row in rows:
        broker.publish(row.type, row.payload, headers={"outbox_id": row.id})
        db.execute("UPDATE outbox SET published_at = now() WHERE id = %s", row.id)
```

Why this works:
- The **DB transaction is atomic**: either both `orders` and `outbox` rows are saved, or neither.
- The **publisher process is decoupled**: it can crash and resume; outbox rows are durable.
- **At-least-once publish**: if the publisher dies between `publish` and `UPDATE`, the row is republished on restart. Consumers must dedupe.

```
   request ──► [ DB transaction ]
                ├─ INSERT order
                └─ INSERT outbox row
                       │ (atomic commit)
                       ▼
              [ outbox poller / CDC ]
                       │ publishes
                       ▼
                   broker
```

---

## 3. Why Not Just Try-Catch + Retry?

A common (wrong) attempt:

```python
def place_order(order):
    db.save(order)
    try:
        broker.publish(event)
    except:
        retry_queue.add(event)  # in-memory retry
```

Fails because:
- Process crashes → retry queue is in memory → lost.
- Retry queue is itself a broker → same dual-write problem one level up.
- "We'll write to a local file and retry" → a poor reinvention of the outbox.

The outbox is the right shape. Accept it.

---

## 4. Variants

### 4.1 Polling outbox
A worker polls the outbox table, publishes, marks. Simple, works everywhere.

```python
# Worker
SELECT ... FROM outbox WHERE published_at IS NULL FOR UPDATE SKIP LOCKED LIMIT 100;
for row: publish; UPDATE published_at = now();
```

Trade-offs:
- **Pros**: simple, requires no special DB features.
- **Cons**: polling latency (typically 100ms–1s); polling load on the DB.

### 4.2 CDC outbox (transactional log tailing)
Instead of polling, use **Change Data Capture** on the outbox table. The DB's WAL/binlog is streamed (Debezium → Kafka) and outbox inserts become events.

```
DB WAL ──► Debezium ──► Kafka topic "outbox.events"
              │
              parses INSERTs on outbox table
              produces events to broker
```

Trade-offs:
- **Pros**: low latency (~ms), no polling load on DB, scales naturally.
- **Cons**: requires CDC infrastructure (Debezium + Kafka + connectors).

The CDC variant is **the modern best practice** at scale. Stripe, Shopify, Wise, and many large engineering orgs use this exact pattern.

### 4.3 Delete after publish
Instead of marking `published_at`, delete the outbox row. Keeps the table small.

Trade-offs:
- Smaller table → faster.
- Loses the publish audit trail — can't tell what was published when.
- Compensating: emit metrics / logs at publish time.

Most teams **mark, then periodically delete old marked rows** (retention policy: 7 days).

---

## 5. Idempotent Consumers (Required)

The outbox guarantees at-least-once publishing. The same event WILL be published more than once on rare occasions. Consumers must handle duplicates.

Patterns:
- **Idempotency key**: the `outbox_id` is unique per event. Consumers track processed IDs and skip duplicates.
- **Natural idempotency**: the operation is safe to apply twice (`SET status = 'paid'`).
- **Upsert**: `INSERT ... ON CONFLICT DO NOTHING`.

```python
def handle_event(event):
    if processed.exists(event.outbox_id):
        return
    process(event)
    processed.add(event.outbox_id)
```

See [Idempotency →](../03-apis/idempotency.md).

---

## 6. The Inbox Pattern (Dual)

The consumer-side analog: when consuming an event and writing to your DB, you face the dual-read-and-write problem. Solution: an **inbox table**.

```sql
BEGIN;
  INSERT INTO inbox (event_id) VALUES ('outbox_id_42');
  -- if duplicate INSERT, unique violation; we already processed
  UPDATE business_state SET ...;
COMMIT;
```

The `inbox` table dedupes events. Combined with consumer-group offset commit, gives effectively exactly-once processing.

This is sometimes called the **inbox pattern** or **dedup table** pattern.

---

## 7. Ordering

Events for the same aggregate should reach consumers in order. Two pieces:

### 7.1 Outbox publish order
Publish in insertion order. Polling with `ORDER BY created_at` works for low concurrency. At higher concurrency, multiple poller workers may interleave.

The cleanest: partition the outbox by aggregate and have one publisher per partition. Or use CDC, which respects transaction order from the WAL.

### 7.2 Broker partition key
Send `outbox_id` events to a partition by `aggregate_id` (`order_id`, `user_id`). Same aggregate → same partition → ordered.

---

## 8. Outbox + CDC Architecture in Detail

The high-end implementation:

```
   service writes business state + outbox row in one tx
                          │
                          ▼
                   Postgres / MySQL (WAL changes)
                          │
                          ▼
                       Debezium
                          │ tails WAL, parses INSERTs to outbox
                          ▼
                        Kafka topic outbox.events
                          │
                          ▼
                    consumers (downstream services)
```

Debezium's outbox event router:
- Filters to only the outbox table.
- Extracts the event type / payload from columns.
- Routes to the right Kafka topic.

```yaml
# Debezium connector config
"transforms": "outbox",
"transforms.outbox.type": "io.debezium.transforms.outbox.EventRouter",
"transforms.outbox.table.field.event.id": "id",
"transforms.outbox.table.field.event.type": "type",
"transforms.outbox.table.field.event.payload": "payload",
"transforms.outbox.route.by.field": "aggregate"
```

The result: the moment a transaction commits, the event is in Kafka within milliseconds. No polling. Atomic. Reliable.

---

## 9. Worked Example: Order Service

### Schema
```sql
CREATE TABLE orders (
  id UUID PRIMARY KEY,
  user_id UUID NOT NULL,
  total NUMERIC NOT NULL,
  status TEXT NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE outbox (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  aggregate TEXT NOT NULL,
  aggregate_id TEXT NOT NULL,
  type TEXT NOT NULL,
  payload JSONB NOT NULL,
  created_at TIMESTAMPTZ DEFAULT now(),
  published_at TIMESTAMPTZ NULL
);

CREATE INDEX outbox_unpublished_idx ON outbox(created_at) WHERE published_at IS NULL;
```

### Place order
```python
def place_order(user_id, items):
    order = Order.create(user_id, items)
    payload = {
        "order_id": order.id,
        "user_id": user_id,
        "total": order.total,
        "items": items,
    }
    with db.transaction():
        db.insert("orders", order)
        db.insert("outbox", {
            "aggregate": "order",
            "aggregate_id": order.id,
            "type": "OrderPlaced",
            "payload": json.dumps(payload),
        })
    # Note: no broker.publish here. The outbox poller does that.
    return order
```

### Poller
```python
def outbox_poller():
    while True:
        with db.transaction():
            rows = db.query("""
                SELECT * FROM outbox
                WHERE published_at IS NULL
                ORDER BY created_at
                LIMIT 100
                FOR UPDATE SKIP LOCKED
            """)
            for row in rows:
                broker.publish(
                    topic=f"{row.aggregate}.events",
                    key=row.aggregate_id,
                    value=row.payload,
                    headers={"outbox_id": row.id, "type": row.type},
                )
                db.execute("UPDATE outbox SET published_at=now() WHERE id=%s", row.id)
        time.sleep(0.1)
```

`FOR UPDATE SKIP LOCKED` lets multiple poller instances run safely — each grabs a different batch.

### Cleanup
```sql
DELETE FROM outbox
WHERE published_at IS NOT NULL
  AND published_at < now() - interval '7 days';
```

A scheduled job, daily.

---

## 10. Operational Concerns

### Backlog growth
If the poller falls behind, the outbox grows. Monitor:
- `SELECT count(*) FROM outbox WHERE published_at IS NULL` — should stay low.
- Oldest unpublished row's age — alert if > N minutes.

### Poller scaling
Multiple poller instances using `FOR UPDATE SKIP LOCKED`. Each processes a different batch. Scales horizontally.

For CDC-based: Debezium handles parallelism via Kafka partitions.

### Schema evolution
The outbox `payload` is JSON. Events are stable contracts. Use schema registry; treat outbox events like any other published event.

### Failure of the broker
The poller retries. Events accumulate in the outbox until the broker is back. Outbox = durable buffer.

### Failure of the poller
Poller dies; nothing publishes; outbox grows. Monitor poller liveness. When restarted, it picks up where it left off (no work lost, thanks to `published_at IS NULL`).

### Large payload
JSON in DB is fine for small/medium events. For large blobs, store a reference (`s3_key`) in the outbox and put the blob in object storage.

---

## 11. When NOT to Use the Outbox

Sometimes the outbox is overkill or wrong.

- **Pure reads** — no state change, no event. No outbox needed.
- **Single-store flows** — your business state IS in the broker (event-sourced Kafka). No need for a separate DB outbox.
- **Acceptable loss** — telemetry, low-stakes notifications. Direct publish is fine.
- **Synchronous request/response** — you wait for a response anyway; failure surfaces immediately.

But for any multi-system flow where you must reliably publish an event after a DB change: outbox.

---

## 12. Alternatives (and Why Outbox Wins)

### 12.1 Listen/Notify (Postgres LISTEN/NOTIFY)
Inside a transaction, call `NOTIFY` after `INSERT`. A worker listens.

- **Cons**: NOTIFY payload is small (~8 KB); not durable (if no listener, lost); not horizontally scalable.

### 12.2 Two-phase commit (XA)
Use XA transactions across DB and broker. Most brokers don't support it; ops complexity is high. Don't.

### 12.3 Compensating transactions
Publish first; if DB fails, "unpublish" with a compensating event. Brittle, leaks abstractions, hard to test.

### 12.4 Best-effort broker publish
Just publish; accept rare losses. Fine for non-critical events. Wrong for anything important.

### 12.5 Trigger-based eventing
DB trigger fires on insert and writes to a queue table or calls a worker. Effectively the same as the outbox, just with implicit insertion. Less common but valid.

The outbox is **the boring right answer**.

---

## 13. Outbox + Saga

Sagas use the outbox at every step. The orchestrator (or each participating service):
1. Performs its local DB change.
2. Writes the next-step event to the outbox in the same transaction.
3. Poller publishes the event.
4. Next service consumes, performs its change + outbox write atomically.

This makes the entire saga reliable end-to-end. See [Saga Pattern →](./saga-pattern.md).

---

## 14. Common Mistakes

- **Publishing in the same code path as the DB write but outside the transaction.** The naive dual-write. The bug you don't see until production.
- **No idempotency on consumers.** Outbox is at-least-once; consumers will see duplicates.
- **Letting outbox grow forever.** No cleanup → DB bloat.
- **No monitoring of poller lag.** Silent breakage: outbox grows, downstream never gets events.
- **Single poller in production.** Bottleneck. Scale horizontally with `FOR UPDATE SKIP LOCKED`.
- **Large payloads** — outbox table bloats, queries slow.
- **No schema discipline** — events are forever; outbox payloads same.
- **Forgetting the outbox during local dev.** Tests pass; prod breaks. Make the poller part of the dev stack.
- **Coupling outbox-publish to business-write latency.** If the poller is slow, business writes still succeed; events arrive late. Often fine; monitor it.
- **Publishing every row of every table.** Outbox is for **business events**, not raw row-changes. Use CDC for raw changes; outbox for domain events.

---

## 15. Cheat Card

```
PROBLEM       atomically write DB + publish event
              naive dual-write loses events on failure

SOLUTION      INSERT into outbox table in same DB transaction
              separate poller/CDC publishes outbox rows

VARIANTS
  polling       SELECT WHERE published_at IS NULL FOR UPDATE SKIP LOCKED
  CDC           Debezium tails WAL, routes to broker (modern best)
  mark vs del   mark for audit; periodic cleanup

GUARANTEE     at-least-once publish; consumers must be idempotent

ORDERING      per-aggregate via partition key + ordered publish

REQUIRED
  outbox table in same DB
  poller or CDC pipeline
  monitoring of unpublished count + age
  idempotent consumers
  cleanup of old published rows

PAIRED WITH   sagas (per-step), event sourcing, CQRS

DON'T         publish in code outside the DB tx
DON'T         use it for raw row changes (use CDC for that)
DON'T         skip consumer idempotency

RULE          The outbox is the reliable bridge between
              your transactional DB and your event broker.
```

---

## 16. Resources

### Articles
- "The Outbox Pattern" — Chris Richardson, microservices.io.
- "Reliable Microservices Data Exchange With the Outbox Pattern" — Gunnar Morling, Debezium blog.
- "Implementing the outbox pattern" — Wise / Shopify / Stripe engineering blogs.
- "Pattern: Transactional Outbox" — Microsoft architecture docs.

### Documentation
- **Debezium outbox event router**: <https://debezium.io/documentation/reference/stable/transformations/outbox-event-router.html>
- **PostgreSQL FOR UPDATE SKIP LOCKED**: <https://www.postgresql.org/docs/current/sql-select.html#SQL-FOR-UPDATE-SHARE>

### Videos
- ByteByteGo — "Outbox Pattern Explained".
- Gunnar Morling — Outbox pattern with Debezium.
- Confluent — CDC + outbox patterns.

### Tools
- **Debezium** — CDC for Postgres, MySQL, Mongo, SQL Server.
- **Apache Kafka Connect** — connector framework.
- **Any RDBMS** with `FOR UPDATE SKIP LOCKED` (Postgres 9.5+, MySQL 8+).
- Built-in support in some frameworks: Eventuate Tram, Axon, Spring Modulith.

### Adjacent reading
- [Event-Driven Architecture →](./event-driven-architecture.md)
- [Saga Pattern →](./saga-pattern.md)
- [Delivery Guarantees →](./delivery-guarantees.md)
- [Event Sourcing →](./event-sourcing.md)
- [CDC →](../04-databases/cdc.md)
- [Idempotency →](../03-apis/idempotency.md)
- [Kafka Deep Dive →](./kafka.md)
- [Database Transactions & Isolation Levels →](../04-databases/transactions-isolation.md)

---

*Previous:* [← Saga Pattern](./saga-pattern.md)  |  *Up:* [README ↑](../README.md)
