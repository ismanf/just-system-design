# Design a Distributed ID Generator (Snowflake)

> **TL;DR** — Distributed unique ID generation is one of the most-asked design questions because the solutions are deceptively elegant. The constraints: **unique across the cluster, sortable by time (often), 64 bits to fit in a BIGINT, no central coordination per ID**. Twitter's **Snowflake** is the canonical answer — a 64-bit ID packing **timestamp + machine ID + sequence**. Variants: Sonyflake, Instagram's ID scheme (Postgres-based), MongoDB ObjectId, UUIDv7. Avoid pure UUIDv4 if you need DB-friendly ordering — random 128-bit IDs ruin B-tree locality.

---

## 1. Requirements

### Functional
- Generate unique IDs across the cluster, forever.
- Sortable by creation time (in most use cases).
- High throughput (millions of IDs/sec across the cluster).
- No coordination per ID.

### Non-Functional
- Latency p99 < 1 ms.
- Survive single-node failure.
- 64-bit (BIGINT-friendly).

---

## 2. Why Not Just UUIDv4?

- 128 bits — wastes space and B-tree locality.
- Random → DB writes become random inserts (bad for LSM trees too).
- Not sortable by time.

UUIDs are fine for opaque external identifiers but bad as DB primary keys at high write throughput.

---

## 3. Snowflake — The Canonical 64-bit Layout

Twitter Snowflake (2010):
```
| 1 bit | 41 bits          | 10 bits     | 12 bits    |
|  0    | timestamp ms     | machine_id  | sequence   |
```

- **Sign bit**: 0 (positive).
- **Timestamp**: ms since custom epoch (e.g., 2010-01-01). 41 bits = ~69 years.
- **Machine ID**: identifies the worker (1024 workers max).
- **Sequence**: incrementing counter per ms; 4096 IDs per ms per worker.

Throughput per worker: 4096 × 1000 = ~4 M IDs/sec.
Cluster: 1024 workers × 4 M = 4 B IDs/sec.

---

## 4. Generation Logic

```python
def next_id():
    now = current_ms()
    if now == last_ms:
        seq = (seq + 1) & 0xFFF
        if seq == 0:
            wait_until(now + 1)  # sequence overflow; wait next ms
            now = current_ms()
    else:
        seq = 0
    last_ms = now
    return (now << 22) | (machine_id << 12) | seq
```

Generation is purely local — no network call. Latency: microseconds.

---

## 5. Machine ID Assignment

Each worker needs a unique machine_id (1024 max).

Options:
- **Config file** per host (simple but error-prone).
- **ZooKeeper / etcd** ephemeral nodes assign on startup.
- **DB sequence** allocating machine IDs.
- **Kubernetes** stable identity from StatefulSet ordinal.

If you accidentally start two workers with same machine_id, you'll generate duplicate IDs. Single most important check.

---

## 6. Clock Skew

Snowflake depends on monotonic clock progress.

If the clock goes backward:
- **Refuse to generate** until clock catches up (Twitter's choice).
- **Use the previous timestamp** + sequence to continue.
- **Re-allocate machine_id** (loses up to 1024 IDs in transit).

Modern systems: monotonic clock from OS + NTP synchronization. ChronyD recommended over ntpd.

Some variants use **wall clock + monotonic counter** to avoid sleep on small backslides.

---

## 7. Variants

### 7.1 Sonyflake (Sony)
```
| 1 | 39 bits (10ms units) | 8 bits machine | 16 bits seq |
```
- Lower resolution (10ms) for longer life (~174 years).
- Fewer machines (256) but more IDs per unit (65K).

### 7.2 Instagram (Postgres-based)
```
| 41 bits ms | 13 bits shard_id | 10 bits per-shard sequence |
```
Sequence drawn from a per-shard Postgres function. Avoids machine-id management.

### 7.3 MongoDB ObjectId (96 bits)
```
| 32 bits seconds | 24 bits machine | 16 bits pid | 24 bits counter |
```

### 7.4 UUIDv7 (2022)
Time-ordered UUID. 128 bits but first 48 bits are a timestamp. Best of both worlds if you can spend 128 bits.

### 7.5 ULID
26-char string, 128 bits, time-ordered. Common in newer systems.

---

## 8. Time-Sortability

Snowflake IDs are time-monotonic per worker (and approximately monotonic globally given synchronized clocks).

Benefits:
- DB inserts are mostly append (B-tree right side stays hot).
- Pagination by ID is chronological (Twitter timelines).
- Cache locality.

---

## 9. ID as Cursor

For pagination:
```
GET /tweets?before_id=12345678
```

Works because IDs encode time. No need for separate `created_at` column.

---

## 10. Coordination-Free vs Centralized

### 10.1 Snowflake (coordination-free)
Each worker generates independently. No network call per ID.

### 10.2 Centralized counter
A single service (Redis, ZooKeeper) hands out IDs.
- Simple.
- Bottleneck and SPOF.
- Latency includes RTT.

### 10.3 Batch allocation
Workers fetch ranges (e.g., 1000 IDs) from central; use locally.
- Mostly coordination-free.
- Coordinator only loaded on range exhaustion.

Used in many production systems for sequence-style IDs.

---

## 11. Common Mistakes

- **Two workers with same machine_id** — collisions. Use ZooKeeper or equivalent.
- **Clock goes backward; system keeps generating** — duplicate IDs.
- **Single worker for whole cluster** — bottleneck, SPOF.
- **Using sequence as the high bits** — defeats time-sortability.
- **UUIDv4 as primary key on a high-write table** — index fragmentation.
- **Allocating machine_id at runtime without persistence** — restart picks same id as another live worker.

---

## 12. Cheat Card

```
PURPOSE    Generate unique 64-bit IDs across cluster, time-sortable, fast.

CORE       Snowflake layout: 1 sign + 41 ms + 10 machine + 12 seq
           Local generation; no network per ID
           Machine ID assigned via ZooKeeper/etcd/StatefulSet ordinal
           Throughput: ~4M IDs/sec per worker

VARIANTS   Sonyflake: 10ms units, 256 machines
           Instagram: per-shard Postgres sequence
           UUIDv7 / ULID: 128 bits but time-ordered

PITFALLS   duplicate machine_id, clock backslide,
           central counter as SPOF, UUIDv4 PKs.

RULE       64 bits, time-ordered, coordination-free.
           Snowflake is the right answer 95% of the time.
```

---

## Resources

### Articles
- "Announcing Snowflake" — Twitter Engineering 2010
- "Sharding & IDs at Instagram" — Instagram Engineering
- "UUIDv7 RFC draft" — IETF

### Documentation
- **Twitter Snowflake** — <https://github.com/twitter-archive/snowflake>
- **Sonyflake** — <https://github.com/sony/sonyflake>

### Books
- *Designing Data-Intensive Applications* — Kleppmann

### Videos
- ByteByteGo: "How does Snowflake ID Generator work?"

### Adjacent reading
- [URL Shortener →](./url-shortener.md)
- [Twitter →](./twitter.md)
- [Clocks (Logical, Vector, HLC) →](../08-distributed-systems/clocks.md)
- [Sharding & Partitioning →](../04-databases/sharding-partitioning.md)

---

*Previous:* [← To-Do App with Offline Sync](./todo-offline-sync.md)  |  *Next:* [Leaderboard →](./leaderboard.md)
