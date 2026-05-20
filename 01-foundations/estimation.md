# Back-of-the-Envelope Estimation

> **TL;DR** — Back-of-the-envelope (BOTE) math turns a vague "scale" into concrete numbers (QPS, storage, bandwidth, memory) in 60 seconds. Those numbers then *force* the design — whether you need one database or 100, whether a cache fits in RAM, whether the bill will be $500/month or $5M. Master 5 base numbers, 3 ratios, and 4 formulas, and you can size *any* system on demand.

---

## 1. Why BOTE Matters

You cannot design without numbers. "Scalable" is not a requirement; "10,000 writes/sec, 100 TB/year" is.

BOTE math:
- Pins down whether one machine is enough (often it is — many "design X" problems are over-scaled by candidates).
- Tells you *where* to optimize. If reads are 100× writes, design for reads first.
- Stops you from reaching for distributed systems when a single Postgres handles it.
- Makes trade-offs concrete: "this design costs $50k/month, this one costs $5k — is the latency worth it?"

The whole point: be *roughly right* in 5 minutes, not *exactly right* in 5 hours.

---

## 2. The Numbers You Must Memorize

### Powers of 10 (cheat scale)
```
10^3  = thousand  (K)
10^6  = million   (M)
10^9  = billion   (B)
10^12 = trillion  (T)
```

### Bytes (powers of 2, but approximate as 10)
```
1 KB ≈ 10^3 bytes
1 MB ≈ 10^6 bytes
1 GB ≈ 10^9 bytes
1 TB ≈ 10^12 bytes
1 PB ≈ 10^15 bytes
```

### Time (seconds in...)
```
1 day  ≈ 86,400 sec   → round to 10^5
1 month ≈ 2.6M sec    → round to ~2.5 × 10^6
1 year ≈ 31.5M sec    → round to ~3 × 10^7
```

### Jeff Dean's Latency Numbers (every engineer should know)
```
L1 cache reference                       0.5 ns
Branch mispredict                        5   ns
L2 cache reference                       7   ns
Mutex lock/unlock                       25   ns
Main memory reference                  100   ns
Compress 1 KB with Zippy             3,000   ns  =  3 µs
Send 1 KB over 1 Gbps network       10,000   ns  = 10 µs
SSD random read                     150,000   ns  = 150 µs
Read 1 MB sequentially from memory  250,000   ns  = 250 µs
Round trip within same datacenter   500,000   ns  = 0.5 ms
Read 1 MB sequentially from SSD   1,000,000   ns  = 1 ms
Disk seek (HDD)                  10,000,000   ns  = 10 ms
Read 1 MB sequentially from HDD  20,000,000   ns  = 20 ms
Round trip CA → Netherlands     150,000,000   ns  = 150 ms
```

Rules of thumb:
- **RAM is ~100× faster than SSD; SSD ~100× faster than HDD.**
- **Network within DC ≈ memory access. Network across continents ≈ 1 million times slower.**
- **Sequential disk is ~10× faster than random disk.**

[Full reference: latency numbers visualized](https://colin-scott.github.io/personal_website/research/interactive_latency.html)

---

## 3. Three Universal Ratios

These show up everywhere. Burn them in.

- **Peak ≈ 2–10× average.** Most consumer products peak at 2–3×; viral / event-driven peak at 5–10×.
- **Reads ≈ 10–100× writes** for most consumer apps (Twitter, Instagram, news, e-commerce browsing). For analytics / logging it's the opposite.
- **10% of users produce 90% of traffic** (Pareto-ish). Plan for hot keys/users.

---

## 4. The Four Formulas

### F1 — QPS from DAU
```
QPS_avg = (DAU × actions_per_user_per_day) / 86,400
QPS_peak = QPS_avg × 3   (conservative)  or  × 10  (aggressive)
```

### F2 — Storage
```
total_bytes = records_per_day × bytes_per_record × retention_days
```
Add ~20–50% for indexes, replication overhead.

### F3 — Bandwidth
```
bandwidth_in_bits  = QPS × avg_payload_bytes × 8
```
Divide by 10^9 to get Gbps.

### F4 — Memory for a cache
```
cache_RAM = working_set_size × replication_factor
```
The "working set" is usually the top 10–20% of keys (the hot ones).

---

## 5. A Full Worked Example: "Design Twitter"

The interviewer says: *"Design Twitter."* You ask: *"How many users?"* They say: *"~300M MAU, ~150M DAU."*

### Reads/Writes per user
- A user reads the home timeline ~10×/day → **10 reads/day/user**.
- A user writes ~0.2 tweets/day on average → **0.2 writes/day/user**.

### QPS — writes
```
writes/day = 150M × 0.2 = 30M
writes/sec  = 30M / 86,400 ≈ 350/sec    (average)
peak ≈ 1,500–3,500/sec                  (using 5–10× peak factor)
```

### QPS — reads
```
reads/day = 150M × 10 = 1.5B
reads/sec  = 1.5B / 86,400 ≈ 17,000/sec   (average)
peak ≈ 50,000–170,000/sec                 (using 3–10× peak factor)
```

### Storage
Avg tweet: 280 chars + metadata ≈ 1 KB. (Add media URLs separately.)
```
tweet text/day  = 30M × 1 KB = 30 GB/day
                ≈ 11 TB/year
```
Media (images/video) is 10–100× larger, so call it ~100–500 TB/year of media, stored in object storage (S3-style), with thumbnails cached in CDN.

### Bandwidth
Read path:
```
17,000 reads/sec × ~5 KB per timeline page = 85 MB/sec
                                            ≈ 680 Mbps egress, average
                                            ≈ a few Gbps peak
```

### Cache sizing
Hot timeline = last 200 tweets/user. Assume 10% of users active in any 5-min window:
```
hot users = 0.1 × 150M = 15M
each user's hot timeline ≈ 200 tweets × 1 KB = 200 KB
cache size = 15M × 200 KB = 3 TB
```
That's a lot for a single Redis box → shard across ~30 nodes of 100 GB each.

### What the numbers tell us
- **Read path dominates** (17k vs 350 QPS) → cache aggressively, denormalize, precompute timelines.
- **Storage is non-trivial but manageable** in cheap object storage.
- **Bandwidth is fine within a single AWS region.**
- **Fan-out on write is the interesting problem.** A celebrity with 100M followers writing a tweet → 100M timeline writes if you precompute. That's the *real* design problem.

Without the BOTE math, you wouldn't know which problem to solve. *That's the whole point.*

---

## 6. Estimation Cheat Sheet (memorize)

### Time
```
1 day   ≈ 86,400 s     → ~10^5
1 month ≈ 2.6M s
1 year  ≈ 31.5M s
```

### Data sizes
```
ASCII char        1 byte
UTF-8 char        1–4 bytes
Int32             4 bytes
Int64 / timestamp 8 bytes
UUID              16 bytes (binary) / 36 bytes (text)
Tweet (text)     ~300 bytes payload, ~1 KB w/ metadata
Web page (HTML)  ~100 KB
Photo            ~200 KB – 5 MB
HD video / min   ~50 MB
```

### Hardware (typical commodity cloud)
```
CPU             ~10–30k QPS per core for simple JSON APIs
                (much lower for heavy work, much higher for tiny ops)
RAM             64–512 GB typical server
SSD             ~100k IOPS, ~500 MB/s sequential
Network NIC     1–25 Gbps
```

### Throughput rough capacities
```
Single MySQL/Postgres node    ~5–10k writes/sec
                              ~30–50k reads/sec (w/ indexes & cache)
Single Redis node             ~100k ops/sec
                              ~1M ops/sec with pipelining
Single Kafka broker           ~100k–1M msgs/sec
Single Nginx box              ~50k–100k RPS
S3 bucket                     ~3,500 PUT/sec, ~5,500 GET/sec per prefix
```

These are *order-of-magnitude*. Real workloads vary 10× either direction.

---

## 7. How to Round in BOTE Math

- Round to one significant figure. `350` becomes `~400`.
- Use friendly numbers: 1, 2, 5, 10, 100.
- Convert seconds to days when storing: 86,400 ≈ 10^5.
- For exponents: just add them. `10^6 × 10^3 = 10^9`.
- Pessimism > optimism. If unsure, double the estimate.

The goal is **decision-grade math**, not engineering-grade math.

---

## 8. Common Traps

- **Confusing average and peak.** A system designed for average QPS will fall over at the morning rush.
- **Forgetting index overhead.** Indexes can equal or exceed data size.
- **Forgetting replication.** 3× replication factor → 3× storage.
- **Forgetting bandwidth caps.** A cloud VM has a NIC limit. 50 Gbps doesn't come free.
- **Hot keys.** Average load looks fine, but one shard is at 100% — see [Hot Partition](../10-scalability/hot-partitions.md).
- **Storage grows forever.** Always ask: what's the retention policy?

---

## 9. Worked Mini-Examples (60 seconds each)

### URL Shortener at 100M new links/month
```
writes/sec  = 100M / 2.5M sec ≈ 40/sec  (peak ~200/sec)
reads/sec   = ~10× writes      = ~400/sec  (peak ~2,000/sec)
storage     = 100M × 12 mo × 500 bytes = 600 GB/year
              → trivial, fits in one DB easily, but cache the hot 1%.
```

### Chat App at 50M DAU, 40 messages/user/day
```
msg/day     = 50M × 40 = 2B
msg/sec     = 2B / 86,400 ≈ 23k/sec  (peak ~100k/sec)
storage/msg = ~100 bytes payload
storage/day = 200 GB/day = ~73 TB/year
              → needs sharded write path; needs object store for attachments.
```

### Video Streaming Service at 200M DAU, 1 hour/user/day
```
hours/day   = 200M
bandwidth   = 200M hours × 5 Mbps × 3600 s / 86,400 s
            ≈ 42 million Mbps total streaming
            ≈ 42 Tbps continuous
              → cannot be served from one region.
              → CDN is mandatory; edge caching is the entire design.
```

### Logging Pipeline at 10k servers × 100 log lines/sec × 1 KB each
```
events/sec  = 10k × 100 = 1M/sec
bandwidth   = 1M × 1 KB × 8 bits = 8 Gbps
storage     = 1M × 1 KB × 86,400 = ~80 TB/day (raw)
              → needs Kafka-class ingestion; needs compression (≥5×); needs tiered storage.
```

You can do each of these in 60 seconds with the numbers from Section 2. That's the muscle to build.

---

## 10. Practice Drill

Set a timer for 5 minutes. For each:

1. Slack — 10M DAU, 100 messages/user/day. QPS, storage/year, bandwidth?
2. Instagram — 500M DAU, 5 photo views + 1 upload/user/day. Bandwidth?
3. Uber — 2M rides/day worldwide. Peak QPS for matching service?
4. Netflix — 250M subscribers, avg 2 hours/day at 5 Mbps. Bandwidth?
5. A doorbell-cam service — 5M cameras streaming 24/7 at 500 kbps each. Bandwidth + storage?

Don't peek at solutions. Compare your numbers afterward to public reports — usually within an order of magnitude.

---

## 11. Resources

### Foundational
- **Jeff Dean's "Numbers Everyone Should Know"** — the original Stanford talk. Slides: <https://www.cs.cornell.edu/projects/ladis2009/talks/dean-keynote-ladis2009.pdf>
- **Latency Numbers Visualized** — Colin Scott's interactive: <https://colin-scott.github.io/personal_website/research/interactive_latency.html>
- **Donne Martin's BOTE section** — <https://github.com/donnemartin/system-design-primer#back-of-the-envelope-calculations>

### Articles
- "Numbers Every Programmer Should Know" — Brendan Gregg: <https://www.brendangregg.com/blog/>
- "The Tail at Scale" — Dean & Barroso (CACM 2013): <https://research.google/pubs/the-tail-at-scale/>
- "Capacity Planning" — Google SRE Book chapter: <https://sre.google/sre-book/software-engineering-in-sre/>

### Videos
- ByteByteGo: "Back-of-the-envelope estimation" — <https://www.youtube.com/@ByteByteGo>
- Gaurav Sen: "Capacity Estimation" walkthroughs.

### Tools
- **Wolfram Alpha** — sanity-check unit conversions on the fly.
- **Excalidraw** — keep a small "scratch corner" for BOTE math during interviews.

---

## 12. One-Page Reference

```
TIME           1 day ≈ 10^5 s    1 month ≈ 2.5×10^6   1 year ≈ 3×10^7
PREFIX         K=10^3  M=10^6  B=10^9  T=10^12
SIZES          char 1B  int32 4B  int64 8B  uuid 16B  tweet ~1KB
                page ~100KB  photo ~500KB  HD/min ~50MB
LATENCY        L1 .5ns  RAM 100ns  SSD .15ms  DC RTT .5ms  X-cont 150ms
RATIOS         peak 3–10× avg    reads 10–100× writes    P80/P20 rule
CAPACITY       core ~20k QPS     RDB ~10k W/s   Redis 100k–1M ops/s
                CDN absorbs all photo/video traffic — design for that
FORMULAS       QPS = DAU × A/86,400
                Storage = N × bytes × retention
                BW = QPS × payload × 8
                Cache = hot_set × replicas
```

---

*Previous:* [← How to Approach a System Design Interview](./interview-approach.md)  |  *Next:* [Powers of Two & Latency Numbers →](./latency-numbers.md)
