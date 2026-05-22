# Batching & Debouncing

> **TL;DR** — **Batching** groups many small operations into one larger operation to amortize per-call overhead — the right tool when *throughput* dominates and *latency* can tolerate a short delay. **Debouncing** waits for a quiet period before acting on a stream of events — the right tool for *eventual* actions where only the final state matters (a search box, an autosave, a window-resize handler). **Throttling** is a sibling: it lets through *at most one* action per interval, regardless of how many trigger. Used in concert with **coalescing**, **buffering**, and **bulkhead-sized queues**, these are the everyday tools for trading a small amount of latency for huge wins in efficiency, downstream load, and tail behavior. Used carelessly they cause hard-to-debug stalls, lost events, and visual glitches. The discipline: pick the shape from the *user-perceived requirement*, then size the timing window to match.

---

## 1. The big picture

```
Without batching        With batching (50 ms window)
────────────────         ──────────────────────────

ev1 → call               ev1 ┐
ev2 → call               ev2 ├──► one call ([ev1,ev2,ev3])
ev3 → call               ev3 ┘
ev4 → call               ev4 → ... 50ms wait → call ([ev4,ev5,...])

N round trips            ~N/B round trips
```

```
Without debouncing       With debouncing (300 ms quiet)
──────────────────       ─────────────────────────────

keystroke → search       keystrokes... keep resetting timer
keystroke → search       (no calls during typing)
keystroke → search       300 ms quiet → fire one search
keystroke → search
```

Both techniques delay an action to do less work. They differ in *what they save*:

- **Batching**: amortizes setup cost (RTT, serialization, lock acquisition, transaction overhead) across many items.
- **Debouncing**: throws away intermediate states — only the *last* matters.

You'll often combine them: a user types in a search box (debounced), and the resulting search returns 50 highlighted results (batched response fetch).

---

## 2. Batching — what it actually buys

Per-call overhead has many components:

| Cost | Per call | Amortized across N |
|---|---|---|
| Network RTT | 0.5–60 ms | ~0 per item |
| TCP/TLS framing | bytes | ~0 per item |
| Serialization/parse setup | µs | ~0 per item |
| Transaction begin/commit | ms in DBs | ~0 per item |
| Lock acquisition | µs–ms | ~0 per item |
| Authentication / authorization | µs–ms | ~0 per item |
| Per-call CPU branch overhead | µs | ~0 per item |

Batching pays back wherever those fixed costs dominate per-item work. Some canonical examples:

- **Kafka producer** — `linger.ms` waits up to a few ms to fill a batch; throughput goes from ~10K msg/s/producer to 500K+ msg/s/producer.
- **Database bulk insert** — `INSERT ... VALUES (...), (...), (...)` with 1000 rows is 20–100× faster than 1000 single `INSERT`s.
- **Redis pipelining** — Send N commands without waiting for replies; one RTT for N ops.
- **gRPC streaming** — One HTTP/2 stream carrying many messages.
- **HTTP request batching** — `/users?ids=1,2,3,4` vs four separate calls.
- **DataLoader** in GraphQL — see [N+1 Query Problem →](./n-plus-one.md).
- **Log shippers** — buffer in memory, flush every N seconds or M bytes.
- **Vector DB / search ingestion** — index in batches of 1K–100K.

The shape is the same: collect into a buffer, flush by time, size, or pressure. The interesting choices are how you flush and how you handle failures.

---

## 3. The three flush triggers

A batcher decides "when do I send what I have?" Three triggers, usually combined:

| Trigger | "Send when..." | Best for |
|---|---|---|
| **Time** (linger / window) | ...this much time has passed | Bursty traffic, latency-bounded |
| **Size** (count or bytes) | ...the buffer has N items or B bytes | High-throughput, predictable cost |
| **Pressure / explicit flush** | ...the caller demands it | Shutdown, manual flushes, urgent items |

Real systems mix all three. Kafka producer: `linger.ms` *and* `batch.size`. Datadog tracer: `flush_interval` *and* `max_batch_size` *and* on `shutdown`.

The fundamental trade-off: **longer batches = better throughput, worse latency**. Set the time bound by the **latency budget you can spare**. For a Kafka producer behind a user-facing API, `linger.ms=5–20`. For analytics ingestion, `linger.ms=100–1000` is fine.

---

## 4. Coalescing — the special case of batching

**Coalescing** means: collapsing multiple updates to *the same key* into one. If 10 events come in for `user_id=42`, we send one update containing the merged result, not 10.

Coalescing is critical for cache invalidation, write-back caches, and counter increments:

```python
# Without coalescing
incr("page:views:home") × 100  # 100 round trips

# With coalescing in app memory
local["page:views:home"] += 100
... flush every 1s → 1 round trip with +100
```

When you can do this safely (the operations are commutative and you can tolerate eventual consistency), coalescing reduces downstream load by orders of magnitude. Redis pipelines + local coalescing turn millions of small increments per second into a manageable workload.

The CRDT-like rule: **coalescing is safe when the operation is associative and commutative**. Counter increments, set unions, max/min, last-write-wins — all good. Non-idempotent operations (charging a card, sending an email) — emphatically not.

---

## 5. Debouncing vs throttling — the pair you'll actually use

Two patterns that get mixed up constantly:

| | Debounce | Throttle |
|---|---|---|
| **Trigger** | Wait for quiet, then fire | Fire at most once per interval |
| **Frequency under storm** | Fires once, after storm ends | Fires once per interval during storm |
| **Use for** | "Do the work after the user is done" | "Do the work at most this often" |
| **Search box, autosave** | ✓ | |
| **Scroll handler, mousemove** | | ✓ |
| **Window resize → re-layout** | ✓ (after resize) | ✓ (during resize) — both valid |
| **Rate limiting outbound calls** | | ✓ |

```javascript
// Debounce — fires once, 300ms after last call
function debounce(fn, ms) {
  let timer;
  return (...args) => {
    clearTimeout(timer);
    timer = setTimeout(() => fn(...args), ms);
  };
}

// Throttle — fires at most once per interval
function throttle(fn, ms) {
  let last = 0;
  return (...args) => {
    const now = Date.now();
    if (now - last >= ms) {
      last = now;
      fn(...args);
    }
  };
}
```

A useful frame: **debounce when only the latest state matters; throttle when steady progress matters**.

Common UI examples:

- **Search-as-you-type**: debounce (300–500 ms). Wait until the user stops; then search.
- **Form autosave**: debounce (1–2 s). Save after they pause.
- **Window resize → re-layout**: throttle to ~60 fps (16ms). Smooth visual updates.
- **Scroll handlers**: throttle to one frame (`requestAnimationFrame`).
- **Drag-and-drop reorder**: throttle. Smoothness matters.

Wrong choice symptoms: debounced scroll handler → "the screen doesn't update while I scroll." Throttled search-as-you-type → "it queries every keystroke."

---

## 6. Patterns in practice

### Database write batching

```python
buffer = []
def enqueue(row):
    buffer.append(row)
    if len(buffer) >= 1000:
        flush()

# Periodic flush
async def flusher():
    while True:
        await asyncio.sleep(0.05)   # 50 ms
        if buffer:
            flush()

def flush():
    rows, buffer[:] = buffer[:], []
    cursor.executemany(
        "INSERT INTO events (...) VALUES (%s, %s, %s)", rows
    )
```

Three knobs: batch size (1000), time window (50ms), graceful shutdown flush. A real implementation adds: bounded buffer with backpressure, retry on transient errors, dead-letter on poison messages.

### Kafka producer

```properties
linger.ms = 10            # wait up to 10ms to fill a batch
batch.size = 65536        # 64 KB per partition
compression.type = zstd   # compress the batch
acks = all                # durability
buffer.memory = 67108864  # total buffer 64 MB
```

These four lines turn a "send one at a time" Kafka client into a high-throughput one. The cost: up to ~10ms latency added to the *first* message in a batch; subsequent messages in the same batch are essentially free.

### Redis pipelining vs MULTI

- **Pipelining**: send N commands, then read N replies. No transactional semantics. Pure RTT savings.
- **MULTI/EXEC**: same plus atomic execution. Slightly more overhead.

A 100-command pipeline executes in 1 RTT instead of 100. Most Redis client libraries support pipelines explicitly; embracing them is the difference between "Redis is slow" and "Redis is the fastest thing in the stack."

### gRPC client-side batching / streaming

```proto
service Logger {
  rpc StreamLogs(stream LogEntry) returns (Ack);
}
```

Open one stream, send many messages, receive a single ack at the end. Excellent for telemetry.

### Browser request coalescing (SWR, React Query)

Libraries like SWR and React Query *deduplicate* concurrent identical requests within a window: if three components ask for `/api/me` in the same render, one HTTP request runs and all three components share the result. It's coalescing without explicit batching — same family.

---

## 7. Failure handling — the hard part

A batch fails. Now what?

### All-or-nothing

The whole batch succeeds or fails together. Easy semantics; harsh consequences — one bad item poisons the rest.

**When to use**: transactional contexts (DB inserts in one transaction, Kafka transactional writes).

### Partial success

The batch endpoint returns per-item statuses. The caller retries failed items individually.

```json
POST /batch
[{"id": 1}, {"id": 2}, {"id": 3}]

200 OK
[
  {"id": 1, "status": "ok"},
  {"id": 2, "status": "error", "reason": "validation"},
  {"id": 3, "status": "ok"}
]
```

**When to use**: bulk APIs that ingest user-provided data; ETL pipelines.

### Retry the whole batch

Acceptable for idempotent operations. Less acceptable when one bad row will fail again.

### Quarantine the poison

Detect items that keep failing; move them to a **dead-letter queue** for human inspection. See [Dead Letter Queues →](../07-messaging/dead-letter-queues.md).

The decision tree:
- Idempotent items? → retry the batch a few times with backoff.
- Heterogeneous items where one bad apple is common? → partial-success API.
- Strict transactional semantics required? → all-or-nothing.

---

## 8. Backpressure — when the buffer fills

A batcher's buffer is finite. What happens when producers outpace the flusher?

Three policies, none of them painless:

- **Block the producer.** The caller waits until space is available. Pushes backpressure upstream — usually the right answer for ingestion paths where data must be preserved.
- **Drop on the floor.** Discard newest or oldest. Acceptable for metrics, traces, logs that can tolerate sampling.
- **Spill to disk.** Write to a local file when memory fills. Persistent buffer; survives restart. Used by Fluentd, Vector, and most log shippers.

Telemetry pipelines (OTLP collectors, Datadog/New Relic agents) routinely combine all three: bounded memory queue → drop-newest with a counter → spill-to-disk in degraded mode.

See [Backpressure →](../10-scalability/backpressure.md).

---

## 9. Latency budget — sizing the window

The single most useful question: **how much extra latency can this path afford?**

| Path | Typical budget | Linger / debounce |
|---|---|---|
| Synchronous API on user click | <100 ms | 1–10 ms linger |
| Form autosave | 500–2000 ms | 1–2 s debounce |
| Live search | 200–500 ms | 200–400 ms debounce |
| Analytics ingestion | seconds | 100–1000 ms linger |
| Backup / archival | minutes | size-bound batches |
| Dashboard refresh | 30 s+ | bigger time windows |

If you don't know your budget, you'll either batch too aggressively (latency suffers) or not enough (no efficiency win).

A pragmatic default that works most of the time: **start with a 10 ms window for hot paths, 100 ms for warm, 1000 ms for cold**. Then measure.

---

## 10. Common Mistakes / Anti-Patterns

- **Batching without an upper bound on wait time.** Buffer fills slowly, latency drifts up. Always have a `linger.ms`.
- **No flush on shutdown.** App exits, batch is lost. Always plumb graceful shutdown that flushes.
- **Per-item retry that doesn't reduce concurrency.** A failed batch retries with 1000 items immediately; just hammers the downstream. Backoff + jitter.
- **Unbounded buffer = OOM.** Bound buffer; choose a backpressure policy.
- **Batching non-idempotent operations.** Retry of a partial-fail batch double-charges customers.
- **Debouncing a navigation event.** User clicks away, your save fires *after* they leave — surprise data loss or wrong-context save.
- **Throttling so aggressively the UI looks broken.** Throttled at 1s for scroll updates feels stuck.
- **Mixing debounce with intermediate state.** Debounce + "show typing indicator" → typing indicator appears and disappears strangely.
- **Setting `linger.ms=0` to "avoid latency"** — and then wondering why Kafka throughput is 10× lower than the docs claim.
- **Batching across tenants.** One tenant's bad payload poisons the batch for everyone else. Batch per tenant or per partition key.
- **Sync `flush` from within an async handler.** Locks up the event loop while the flush runs.
- **Trailing-edge debounce on autosave with no save-on-blur.** User closes the tab during the wait; save never happens.
- **Coalescing non-commutative operations.** "Set status to A" then "Set status to B" coalesced to "B" is fine; "increment counter by 1" then "set counter to 5" coalesced is broken.
- **Different batch sizes for different tenants without per-tenant accounting.** Big tenants dominate the batch budget; small tenants wait.
- **Treating batches as transactions when the underlying isn't.** Bulk INSERT in MySQL with autocommit is N independent inserts under the hood.

---

## 11. Cheat Card

```
PURPOSE   Trade a little latency for huge wins in throughput,
          downstream load, and tail behavior — without losing
          correctness.

THE TRIO
  Batching     group many ops into one; amortize fixed overhead
  Debouncing   wait for quiet; act on the final state
  Throttling   fire at most once per interval

FLUSH TRIGGERS (combine all three)
  Time         linger.ms / window
  Size         items or bytes
  Pressure     explicit flush, shutdown, urgency

COALESCING
  Collapse repeated updates on same key
  Safe if the op is associative + commutative
  Massive savings for counters, cache invalidation,
    set unions, last-write-wins state

LATENCY BUDGET → WINDOW
  Hot user path        1–10 ms linger
  Warm async paths     50–200 ms
  Cold analytics       500–2000 ms
  Search-as-you-type   300–500 ms debounce
  Autosave             1–2 s debounce
  Scroll / resize      throttle to one frame

FAILURE HANDLING
  All-or-nothing    transactional context
  Partial-success   per-item statuses
  Retry whole       idempotent items, backoff + jitter
  Dead-letter       poison messages out of the loop

BACKPRESSURE
  Bound the buffer
  Block / drop / spill to disk — choose deliberately
  Producer slowdown is a feature, not a bug

PITFALLS
  No upper-bound wait time
  No flush on shutdown
  Unbounded buffer → OOM
  Batching non-idempotent ops
  Mixing tenants in one batch (poison spreads)
  Debounce so long the user moves on
  Throttle so coarse the UI looks broken
  Coalescing non-commutative ops

RULE   Pick the shape (batch / debounce / throttle / coalesce)
       from the user requirement. Size the window from the
       latency budget. Plumb graceful flush and backpressure.
```

---

## 12. Resources

### Documentation
- **Kafka Producer Config** — `linger.ms`, `batch.size`, `acks`: <https://kafka.apache.org/documentation/#producerconfigs>
- **Redis Pipelining** — <https://redis.io/docs/manual/pipelining/>
- **OpenTelemetry batching processors** — <https://opentelemetry.io/docs/specs/otel/trace/sdk/#batching-processor>
- **Lodash `debounce` / `throttle`** — <https://lodash.com/docs/#debounce> / <https://lodash.com/docs/#throttle>

### Articles
- "Debounce vs Throttle: Definitive Visual Guide" — Redd Developer / CSS-Tricks.
- "The 0/1 Trick: micro-batching with kafkajs" — Confluent / community engineering.
- "Pipelining vs MULTI/EXEC" — Redis docs.
- "DataLoader: Batch and Cache" — original Facebook engineering write-up.
- Pinterest / Netflix engineering: telemetry batch shipping at scale.

### Videos
- *High-performance Kafka clients* — Confluent talks.
- *Reactive streams + backpressure* — Erlang/Elixir, Akka, RxJS talks.
- ByteByteGo — "Debounce vs Throttle Explained."

### Tools
- **Lodash / Underscore** — `debounce`, `throttle`.
- **rxjs** — `debounceTime`, `throttleTime`, `bufferTime`, `windowTime`.
- **Kafka clients** — built-in producer batching (Java, librdkafka, kafkajs, sarama).
- **OpenTelemetry SDKs** — `BatchSpanProcessor`, `BatchLogRecordProcessor`.
- **DataLoader** — language ports for GraphQL batching.
- **Fluentd / Vector / Fluent Bit** — log batching + backpressure.

### Adjacent reading
- [Profiling & Benchmarking →](./profiling.md)
- [Tail Latency & p99 →](./tail-latency.md)
- [Connection Pooling & Keep-Alive →](./connection-pooling.md)
- [N+1 Query Problem →](./n-plus-one.md)
- [Backpressure →](../10-scalability/backpressure.md)
- [Rate Limiting →](../03-apis/rate-limiting.md)
- [Kafka Deep Dive →](../07-messaging/kafka.md)
- [Stream Processing →](../07-messaging/stream-processing.md)
- [Dead Letter Queues →](../07-messaging/dead-letter-queues.md)
- [CRDTs →](../08-distributed-systems/crdts.md)

---

*Previous:* [← N+1 Query Problem](./n-plus-one.md)  |  *Next:* [Tail Latency & p99 →](./tail-latency.md)
