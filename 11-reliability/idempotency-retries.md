# Idempotent Operations & Retries

> **TL;DR** — An operation is **idempotent** if applying it more than once has the same effect as applying it once. Idempotency is the property that *makes retries safe*: without it, every retry is a potential double-charge, double-send, or duplicate insert; with it, you can retry freely and the system stays correct. The pattern that makes this practical is the **idempotency key** — the client picks a unique key per logical operation; the server deduplicates by key and stores the response. Stripe popularized the modern pattern in 2017; today, every serious payment, order, and money-handling API does the same. Naturally idempotent verbs (PUT, DELETE) get you partway; for POSTs and side-effectful RPCs you need explicit keys. Combine with [retries + timeouts + backoff →](./retry-timeout-backoff.md), [circuit breakers →](./circuit-breaker.md), and [graceful degradation →](./graceful-degradation.md), and remote calls finally become reliable. This page is the implementation playbook.

---

## 1. The Definition

An operation is **idempotent** if executing it N times has the same effect as executing it once:

```
   f(x); f(x); f(x)        ≡       f(x)
   for all valid inputs x and any N ≥ 1
```

Examples:

| Operation | Idempotent? |
|---|---|
| `SET user.email = 'a@b.com'` | Yes — same final state |
| `email = 'a@b.com'` (overwrite) | Yes |
| `email_counter += 1` | **No** — increments each call |
| `DELETE user/42` | Yes — final state is "user 42 gone" |
| `INSERT INTO orders (...)` | **No** — N rows on N calls |
| `INSERT IF NOT EXISTS` (with unique constraint) | Yes |
| `pay(user, $100)` (no key) | **No** — N charges |
| `pay(user, $100, idempotency_key=K)` | Yes if server dedupes by K |
| `send_email(to, subject, body)` | **No** by default |
| `send_email(..., idempotency_key=K)` | Yes with dedup |

Pattern: **read-only operations are idempotent by nature; write operations require care**.

---

## 2. Why It Matters

Without idempotency, retries are dangerous:

```
client: POST /charge { amount: 100 }
   │
   ▼
server: charges card, replies 200 OK
   │
   ▼
network error: response lost
   │
   ▼
client: retries
   │
   ▼
server: charges card AGAIN — user double-charged
```

There is no way for the client to know whether the original request succeeded. It could have:
- Failed before reaching the server.
- Succeeded, but the response was lost.
- Succeeded, and the response arrived after the timeout.

Each case demands a different action. Without idempotency the client has to choose between **never retrying** (a transient failure becomes a real error) and **retrying** (acceptable failure becomes a double-effect bug).

Idempotency removes the dilemma: **the client can retry; the server guarantees the operation runs at most once**.

This is the foundation of every reliable money-handling system, every reliable message-delivery system, and every reliable workflow engine in production.

---

## 3. HTTP and Idempotency

HTTP gives you a head start. Per RFC 9110:

| Method | Safe (no side effects) | Idempotent |
|---|---|---|
| GET | Yes | Yes |
| HEAD | Yes | Yes |
| OPTIONS | Yes | Yes |
| TRACE | Yes | Yes |
| PUT | No | **Yes** (by spec) |
| DELETE | No | **Yes** (by spec) |
| POST | No | **No** |
| PATCH | No | Not necessarily |

Standard implications:
- **GET/HEAD** are safe to retry indefinitely (assuming the server actually behaves).
- **PUT** is idempotent because it sets state to a value — repeated PUTs produce the same end state.
- **DELETE** is idempotent because deleting an already-deleted thing has the same end state (gone).
- **POST** is the one you need to deal with explicitly. POST is the verb most APIs use for "create" — and creating something is not naturally idempotent.

A spec-compliant server makes PUT and DELETE idempotent. A real-world server may break this (returning different status codes, side effects, etc.). Verify your endpoints actually behave idempotently.

For POST: use **idempotency keys**.

---

## 4. The Idempotency Key Pattern

The pattern, popularized by Stripe:

```
Client                                       Server
──────                                       ──────
Generate unique key K (e.g., UUIDv4)
Send: POST /charge { amount: 100 }
      Idempotency-Key: K

                                             ┌────────────────────┐
                                             │ check dedup store  │
                                             │ for K              │
                                             └──┬─────────────────┘
                                                │
                                          K not seen → execute,
                                          store (K, response)
                                                │
                                          K seen    → return stored
                                                     response
                                                ▼
Receive response (200, with K linkage)
```

Properties:
- **Client generates the key** — must be globally unique per logical operation.
- **Server stores `(key, response)`** for some TTL (typically 24 h).
- **Repeated requests with the same key** return the original response, do not re-execute.

That's the whole pattern. Implementations differ in detail; the core is "dedup by key, store response, replay on retry."

---

## 5. Implementation

### Storage

```
table idempotency_records:
   key             primary key
   created_at      timestamp
   request_hash    sha256 of request body (optional, for safety)
   status          'in_progress' | 'completed' | 'failed'
   response_code   integer
   response_body   text or blob
   resource_id     foreign key to whatever was created (e.g., order ID)
   ttl_expires_at  timestamp (often now + 24 h)
```

Typical store: same database as the operation (so dedup + operation are in one transaction) or Redis with persistence. Stripe uses their primary database for this.

### The flow

```python
def charge(amount, card, idempotency_key=None):
    if not idempotency_key:
        # tolerated for backward compat, but not safe for retries
        return raw_charge(amount, card)

    # check for existing record
    record = db.execute("""
        SELECT * FROM idempotency_records
        WHERE key = %s AND ttl_expires_at > now()
    """, [idempotency_key])

    if record:
        if record.status == 'completed':
            return record.response
        if record.status == 'in_progress':
            # another request in flight; we wait or 409
            raise IdempotencyInProgress()
        if record.status == 'failed':
            # we tried before and failed; return failure or retry
            return record.response

    # claim the key
    db.execute("""
        INSERT INTO idempotency_records (key, status, created_at, ttl_expires_at)
        VALUES (%s, 'in_progress', now(), now() + '24 hours')
    """, [idempotency_key])

    try:
        result = raw_charge(amount, card)
        db.execute("""
            UPDATE idempotency_records
            SET status='completed', response_code=%s, response_body=%s,
                resource_id=%s
            WHERE key=%s
        """, [200, result, result.charge_id, idempotency_key])
        return result
    except Exception as e:
        db.execute("""
            UPDATE idempotency_records
            SET status='failed', response_code=%s, response_body=%s
            WHERE key=%s
        """, [500, str(e), idempotency_key])
        raise
```

### Atomicity matters
The most subtle bug: if the operation and the dedup record are not in the same transaction, you can:
1. Successfully execute the operation.
2. Fail to record the idempotency key.
3. Retry executes the operation again.

Fix: in a single DB transaction, both insert the idempotency record AND execute the operation. Or use a write-ahead pattern with explicit acks.

For operations across multiple systems (e.g., charging Stripe + writing to your DB), you can't have one transaction. Patterns:
- **Outbox**: write to your DB + outbox in one txn; worker reads outbox and dispatches with idempotency key. See [Outbox Pattern →](../07-messaging/outbox-pattern.md).
- **Two-phase commit (rare)**: see [2PC →](../08-distributed-systems/2pc-3pc.md).

### Request hash safeguard
Some implementations also store a hash of the request body. If a client sends the same key with a *different* body, the server returns 422 to flag the mismatch. Protects against clients accidentally reusing keys for different operations.

---

## 6. Key Generation

Client-generated, globally unique:

- **UUIDv4** — most common. Random 128-bit value, collision-impossibly rare.
- **UUIDv7** — newer, time-ordered, also unique. Recommended for DB index friendliness.
- **Snowflake / Sonyflake** — for systems already using these.
- **Application-specific**: order ID + retry-attempt nonce. Sometimes useful but more error-prone.

The key must be unique **per logical operation**. Common mistake:

```
# WRONG — same key reused for every request from this client
idempotency_key = client_id

# WRONG — same key reused for retries of any request
idempotency_key = f"{client_id}:{date}"

# RIGHT — unique per intent
idempotency_key = uuid.uuid4()

# RIGHT — derived but unique per logical action
idempotency_key = f"order:{order_id}:create"
```

The semantics: a key represents a *logical operation*. Two retries of the same operation share a key. Two different operations get different keys.

---

## 7. Key Lifetime

How long does the server remember a key?

- **Too short** (e.g., 5 minutes): retries after a network blip miss the dedup; double-execution risk.
- **Too long** (e.g., forever): storage cost grows; key collisions across years become possible.
- **Typical**: **24 hours**. Stripe's default. Enough for any practical retry window; bounded storage cost.

For sensitive financial operations, some teams keep keys for 7 days or longer.

After TTL expires, the key is forgotten. A retry after TTL is treated as a new operation. Document this.

---

## 8. Idempotency Beyond HTTP

The pattern applies wherever retries can happen.

### Message queues (at-least-once delivery)
Most queues (Kafka, SQS, RabbitMQ default) deliver **at least once**. Duplicates happen. Make consumers idempotent:

```
Consumer:
  read message
  check: have we processed message ID M before?
  if yes: ack and skip
  if no:  process; record "processed M"; ack

  the "process + record" should be in one transaction.
```

This is the [outbox pattern →](../07-messaging/outbox-pattern.md) and "exactly-once semantics" in general. See [Delivery Guarantees →](../07-messaging/delivery-guarantees.md).

### Distributed jobs / workflows
A workflow engine (Temporal, AWS Step Functions, Airflow) retries failed steps. Each step must be idempotent or use a step-specific idempotency key. Most engines manage this internally — but the *step implementation* still has to be designed for idempotency.

### Database operations
- `INSERT IF NOT EXISTS` (PG: `ON CONFLICT DO NOTHING`).
- `UPSERT` (PG: `INSERT ... ON CONFLICT ... DO UPDATE`).
- Explicit primary key derived from input rather than auto-increment.

### Webhooks
A webhook delivery system retrying a failed POST should send the same payload with an explicit ID — the receiver dedupes by ID. Stripe, GitHub, and most webhook systems do this.

### Email / notifications
Emails are usually "fire-and-forget" — duplicate sends are an annoyance, not a correctness issue. For transactional emails (order confirmations, password resets), implement idempotency on send.

---

## 9. Idempotency vs At-Most-Once vs At-Least-Once vs Exactly-Once

| Guarantee | Description | Achievable? |
|---|---|---|
| **At-most-once** | Operation runs 0 or 1 times. Loss possible. | Trivially. |
| **At-least-once** | Operation runs 1+ times. Duplicates possible. | Yes; default for retries. |
| **Exactly-once** | Operation runs exactly 1 time. | **Not natively achievable in general distributed systems.** |
| **Effectively-once** | At-least-once delivery + idempotent processing = duplicates have no effect. | Yes. The pragmatic answer. |

The deep insight: **true exactly-once delivery requires consensus, which is slow and limited**. The practical solution everyone uses is **at-least-once delivery + idempotent processing = effectively-once**.

That's why idempotency is non-optional. It's how you get exactly-once semantics out of a distributed system that doesn't natively provide them.

See [Delivery Guarantees →](../07-messaging/delivery-guarantees.md).

---

## 10. Designing Operations to Be Idempotent

When designing an operation that will be retried, make it idempotent by construction where possible.

### Pattern: "intent declarations"
The client declares its intent — `pay invoice I-42 by $100`. The server checks: has invoice I-42 been paid? If yes, return the existing payment. If no, charge.

Here the **intent** (the invoice + amount) is the natural deduplication key. No explicit idempotency key needed.

### Pattern: deterministic IDs
The resource ID is derived from input, not auto-generated:

```
POST /events { name: "signup", user_id: 42, occurred_at: "2026-05-20T..." }
   → resource ID = sha256(name + user_id + occurred_at)
```

Duplicates produce the same ID and `INSERT IF NOT EXISTS` succeeds quietly.

### Pattern: idempotent state machines
Each operation transitions a resource through known states. Repeated operations on a resource already in the target state are no-ops.

```
order.status: pending → paid → shipped → delivered
"mark as shipped" on a delivered order: no-op (return current state).
"mark as paid" on a paid order: no-op.
```

State machines naturally absorb duplicates.

### Pattern: incremental upserts
For counters or accumulators, use idempotency keys per increment:

```
INSERT INTO increments (key, delta) VALUES (K, +1)
   ON CONFLICT (key) DO NOTHING.
total = SELECT SUM(delta) FROM increments WHERE counter_id = X.
```

Each increment is keyed; duplicates are skipped. The total is computed by aggregation.

---

## 11. Operational Reality

### Idempotency keys are precious
A bug that reuses keys can silently mask new operations. Treat the key generation as critical code; test it.

### Storage cost
Idempotency records add up. A high-throughput API processing 1 M ops/day × 24-h TTL × ~1 KB/record ≈ 1 GB of dedup state. Manageable, but design for cleanup.

### TTL cleanup
Background job removes expired records. Avoid scanning huge tables — use indexed `ttl_expires_at`, partition by day, or rely on TTL features in the storage (Redis TTL, DynamoDB TTL).

### Concurrent retries on the same key
Two retries arrive simultaneously, both see "no record yet," both try to claim. Handle:
- Use `INSERT ... ON CONFLICT DO NOTHING` to claim atomically.
- If the second request fails to claim, it waits + polls or returns 409 with a `Retry-After`.

### Errors and idempotency
Should a failed operation be retryable? Two policies:
- **Retry failures**: failed operations are not stored as "completed"; retry behaves as a fresh request.
- **Cache failures**: failed operations are stored; retries get the cached failure response (don't re-execute).

Stripe's approach: cache failure for the TTL, but allow retries to skip the cache via a specific header. Most APIs cache failures by default; clients use new keys for "retry after a known failure."

### Cross-system idempotency
Often, your operation calls another idempotent system (Stripe, an internal microservice). Propagate or derive idempotency keys consistently:

```
Your endpoint receives Idempotency-Key=K
Your call to downstream → use derived key like sha256("downstream-call:" + K)
```

This way, your retry retries downstream too, and downstream dedupes.

### Auditing and observability
- Log idempotency key on every operation for tracing.
- Metric: "% of requests served from idempotency cache." Useful to see retry rates.
- Alert if a single key sees 100s of attempts — possible client bug.

---

## 12. Worked Example — Stripe-Style Payment

A payment API that's safe to retry:

```python
@app.route('/charges', methods=['POST'])
def create_charge():
    idempotency_key = request.headers.get('Idempotency-Key')
    if not idempotency_key:
        return 400, "Idempotency-Key required"

    body = request.json
    body_hash = sha256(canonicalize(body))

    with db.transaction():
        existing = db.fetch_one("""
            SELECT * FROM idempotency_records
            WHERE key = %s
        """, [idempotency_key])

        if existing:
            if existing.request_hash != body_hash:
                return 422, "Idempotency-Key reused for different request"
            if existing.status == 'completed':
                return existing.response_code, existing.response_body
            if existing.status == 'in_progress':
                return 409, "Request in progress; retry later"
            # else status == 'failed' → fall through to retry below

        # claim or replace
        db.execute("""
            INSERT INTO idempotency_records (key, request_hash, status, created_at, ttl_expires_at)
            VALUES (%s, %s, 'in_progress', now(), now() + '24 hours')
            ON CONFLICT (key) DO UPDATE
                SET status='in_progress', request_hash=%s
        """, [idempotency_key, body_hash, body_hash])

    # actually perform the charge — outside the dedup txn
    try:
        result = process_payment(body)
        with db.transaction():
            db.execute("""
                UPDATE idempotency_records
                SET status='completed', response_code=200, response_body=%s,
                    resource_id=%s
                WHERE key=%s
            """, [json.dumps(result), result['charge_id'], idempotency_key])
        return 200, result
    except Exception as e:
        with db.transaction():
            db.execute("""
                UPDATE idempotency_records
                SET status='failed', response_code=500, response_body=%s
                WHERE key=%s
            """, [str(e), idempotency_key])
        return 500, str(e)
```

This is roughly the structure Stripe documents in their [Idempotent Requests guide](https://docs.stripe.com/api/idempotent_requests).

Client side:

```python
import uuid
from retry import retry_with_backoff

@retry_with_backoff(max_attempts=5, jitter=True)
def charge(amount, card):
    idempotency_key = str(uuid.uuid4())  # generated once per logical charge
    return http.post("/charges",
                     json={"amount": amount, "card": card},
                     headers={"Idempotency-Key": idempotency_key})
```

The `uuid.uuid4()` is called *once*, outside the retry — same key on all attempts. Common mistake: regenerating the key on each attempt, which defeats dedup.

---

## 13. Common Mistakes / Anti-Patterns

- **Regenerating the idempotency key on each retry.** Now every retry is "new"; dedup defeated.
- **No idempotency key on POST.** Retries double-charge.
- **Key without a uniqueness guarantee.** `client_id` or `timestamp` collides across operations.
- **Dedup + operation not atomic.** Operation runs, dedup record fails to commit, retry re-executes.
- **Caching failures forever.** A transient failure permanently blocks the operation.
- **No TTL.** Storage grows without bound.
- **TTL too short.** Network blip beyond TTL retries are not deduplicated.
- **Idempotency only on the outer endpoint.** Inner side effects (emails, internal RPCs) still duplicate on retry. Propagate keys.
- **Key reused across different operations.** "I'll use the user ID as the key" — every operation by that user dedupes against the first one.
- **Same key, different request body.** Server should reject (422) — most don't, leading to silent bugs.
- **At-least-once delivery without idempotent consumers.** Duplicates leak through.
- **Idempotency for read operations.** Already idempotent by HTTP; adding a key adds nothing.
- **Mutable resources updated by PUT, but with append semantics.** PUT supposed to be idempotent; you've made it not.
- **Counter increments without keys.** Each retry increments again.
- **`DELETE` returning 404 on second call counted as failure.** The semantic intent (gone) is achieved; treat as 200 or 204.
- **No metrics on retry deduplication.** Can't see how often retries are happening.
- **No idempotency on workflow steps.** Workflow engine retries a step; partial state diverges.

---

## 14. Decision Rule

```
For every operation that may be retried:
  ✓ Is it naturally idempotent (read, set, delete)?
       Yes → safe; just retry.
       No  → add an idempotency key.

For every API that takes POST requests:
  ✓ Accept Idempotency-Key header.
  ✓ Store (key, response) in same DB as the side-effect work.
  ✓ TTL ≈ 24 hours.
  ✓ Reject reused keys with mismatched bodies (422).
  ✓ Document the contract.

For every message consumer:
  ✓ Treat delivery as at-least-once.
  ✓ Dedupe by message ID + business idempotency.
  ✓ Combine processing + dedup record in one transaction.

For every workflow step:
  ✓ Either step is naturally idempotent, or
  ✓ Step uses an explicit idempotency key.

For every cross-system call:
  ✓ Propagate / derive idempotency keys so downstream dedupes too.

Client side:
  ✓ Generate the key ONCE per logical operation.
  ✓ Reuse the same key across all retries.
  ✓ New operation = new key.

Always:
  ✓ Metrics on dedup hit rate.
  ✓ Alerts on suspicious patterns (one key seen 1000×).
```

---

## 15. Cheat Card

```
PURPOSE     Make operations safe to retry: applying N times has the
            same effect as applying once. Foundation for retries,
            at-least-once delivery, and "effectively-once" semantics.

HTTP        GET / HEAD / PUT / DELETE  idempotent by spec
            POST / PATCH               not — use idempotency keys

IDEMPOTENCY KEY PATTERN
  Client: generate unique key K per logical operation (UUID).
  Server: store (K, response) for TTL (typically 24 h).
  Retries with same K return stored response; do not re-execute.

ATOMICITY   Dedup record + side effect MUST be one transaction.
            For cross-system: outbox pattern.

KEY LIFETIME  Typical 24 h. Long enough for any practical retry
              window; bounded storage.

NATURALLY IDEMPOTENT
  SET, DELETE, PUT, INSERT IF NOT EXISTS, UPSERT,
  state-machine transitions, deterministic IDs

EFFECTIVELY-ONCE  at-least-once delivery + idempotent processing.
                  This is how you get exactly-once in practice.

EXTENDS TO  Message queues · workflows · webhooks · DBs · APIs.
            Anywhere retries can happen.

CLIENT      Generate key ONCE per operation. Reuse across retries.
            New operation = new key.

PITFALLS    Regenerate key on retry · no key on POST · low-cardinality
            key · dedup not atomic with op · cache failures forever ·
            no TTL · same key for different bodies · keys not
            propagated to downstream · counter without key · no
            metrics on dedup hits

RULE        Every retry-able write operation has an idempotency key.
            The client generates it once; reuses on every retry.
            The server stores it transactionally with the side effect.
            This single discipline turns brittle networks into safe
            ones.
```

---

## 16. Resources

### Books
- *Designing Data-Intensive Applications* — Martin Kleppmann. Chapter 9 covers exactly-once semantics and idempotency.
- *Release It!* — Michael Nygard. Idempotency in stability patterns.
- *Microservices Patterns* — Chris Richardson. Idempotent consumers, sagas.

### Articles
- "Idempotent Requests" — Stripe API docs: <https://docs.stripe.com/api/idempotent_requests>
- "Designing Robust and Predictable APIs with Idempotency" — Brandur Leach (Stripe): <https://stripe.com/blog/idempotency>
- "Designing Idempotent Receivers" — Hohpe & Woolf, Enterprise Integration Patterns.
- "Implementing Idempotent REST Endpoints" — REST API design guides.
- "Outbox Pattern" — Microservices.io, Chris Richardson.
- "Exactly-Once Delivery and Idempotency in Streaming Systems" — Confluent / Kafka blogs.

### Videos
- "Designing Robust APIs" — Brandur Leach, various conferences.
- "Idempotency Patterns in Distributed Systems" — InfoQ talks.
- ByteByteGo — "Idempotency Explained."
- "Exactly-Once Semantics in Kafka" — Confluent talks.

### Tools
- **Stripe SDK** — implements idempotency keys client-side automatically.
- **Temporal / AWS Step Functions / Airflow** — workflow engines with built-in idempotency primitives.
- **Kafka Idempotent Producer** — exactly-once semantics for Kafka writes.
- **Debezium / outbox patterns** — durable event publishing.
- **Redis with SETNX / SET NX EX** — quick idempotency primitive.
- **DynamoDB conditional writes** — atomic idempotent ops.

### Adjacent reading
- [Idempotency](../03-apis/idempotency.md) — the API-design treatment of the same idea.
- [Retry, Timeout, and Exponential Backoff](./retry-timeout-backoff.md)
- [Circuit Breaker Pattern](./circuit-breaker.md)
- [Graceful Degradation](./graceful-degradation.md)
- [Delivery Guarantees](../07-messaging/delivery-guarantees.md)
- [Saga Pattern](../07-messaging/saga-pattern.md)
- [Outbox Pattern](../07-messaging/outbox-pattern.md)
- [Two-Phase Commit (2PC) and Three-Phase Commit (3PC)](../08-distributed-systems/2pc-3pc.md)

---

*Previous:* [← Blast Radius & Cell-Based Architecture](./cell-architecture.md)  |  *Up:* [README ↑](../README.md)
