# Powers of Two & Latency Numbers Every Engineer Should Know

> **TL;DR** — Two tables, memorized, will get you through every system design conversation:
> 1. **Powers of two / data sizes** so you can size storage, RAM, and bandwidth in your head.
> 2. **Latency numbers** so you know whether a design is "fast enough" before you write code.
> Knowing the relative gaps (RAM vs SSD vs network) is more important than the exact digits.

---

## 1. Powers of Two — Why Engineers Think In Binary

Computers are binary, but the powers of 2 line up *almost* perfectly with powers of 10:

```
2^10 = 1,024            ≈ 10^3   → "Kilo"
2^20 = 1,048,576        ≈ 10^6   → "Mega"
2^30 = 1,073,741,824    ≈ 10^9   → "Giga"
2^40 = 1.1 × 10^12      ≈ 10^12  → "Tera"
2^50 = 1.13 × 10^15     ≈ 10^15  → "Peta"
2^60 = 1.15 × 10^18     ≈ 10^18  → "Exa"
```

The 2% gap between "kilobyte = 1,024" and "kilobyte = 1,000" doesn't matter for design-grade math. For interview math, **treat them as equal**.

### Quick byte-size table
| Unit | Bytes (binary, IEC) | Bytes (decimal, SI) | Mnemonic |
| --- | --- | --- | --- |
| 1 KB | 1,024 (KiB) | 1,000 | "thousand" |
| 1 MB | 1,048,576 (MiB) | 1,000,000 | "million" |
| 1 GB | 1.07e9 (GiB) | 1e9 | "billion" |
| 1 TB | 1.1e12 (TiB) | 1e12 | "trillion" |
| 1 PB | 1.13e15 (PiB) | 1e15 | "quadrillion" |

### Common data-size mental anchors
```
ASCII char            1 byte
Unicode char (UTF-8)  1–4 bytes
Boolean (in memory)   1 byte
int32 / float32       4 bytes
int64 / float64       8 bytes
UUID (binary / text)  16 / 36 bytes
IPv4 / IPv6 address   4 / 16 bytes
Timestamp (epoch ms)  8 bytes
SHA-256 digest        32 bytes

Tweet (text only)     ~300 B; ~1 KB with metadata
URL                   ~50–100 B
Compressed log line   ~200 B–1 KB
JPEG photo (web)      ~200 KB – 1 MB
Original-quality JPG  ~3–10 MB
4K video frame        ~3 MB raw, ~50 KB compressed
HD video (5 Mbps)     ~37 MB/minute
4K video (25 Mbps)    ~190 MB/minute
```

### Bandwidth conversions
```
1 byte  = 8 bits
1 Gbps  = 125 MB/s (since you divide by 8)
1 GB/s  = 8 Gbps
```

Network is measured in **bits**; storage in **bytes**. Mix them up and you'll under- or over-provision by 8×.

---

## 2. Jeff Dean's Latency Numbers (with 2020s adjustments)

The famous table from a Google talk in 2009, refreshed for modern hardware. Memorize the **order of magnitude**, not the exact digits.

```
─────────────────────────────────────────────────────────────────
 Operation                                  Time         Mnemonic
─────────────────────────────────────────────────────────────────
 L1 cache reference                          0.5 ns
 Branch mispredict                            5  ns
 L2 cache reference                           7  ns        ~14×  L1
 Mutex lock/unlock                           25  ns
 Main memory reference (DRAM)               100  ns        ~200× L1
 Compress 1 KB (Snappy/Zstd)              3,000  ns =  3 µs
 Send 1 KB over 1 Gbps network           10,000  ns = 10 µs
 SSD random read (NVMe, 2020s)           16,000  ns = 16 µs  (was 150 µs)
 Read 1 MB sequentially from memory     250,000  ns = 250 µs
 Round trip within same datacenter      500,000  ns = 0.5 ms
 Read 1 MB sequentially from SSD       1,000,000 ns =  1 ms
 Disk seek (spinning HDD)             10,000,000 ns = 10 ms
 Read 1 MB sequentially from HDD      20,000,000 ns = 20 ms
 RTT same continent (US east–west)    40,000,000 ns = 40 ms
 RTT cross continent (CA → NL)       150,000,000 ns = 150 ms
─────────────────────────────────────────────────────────────────
```

Interactive version: <https://colin-scott.github.io/personal_website/research/interactive_latency.html>

### Visualizing the gaps

If L1 cache = **1 second** (a heartbeat), then on the same scale:

```
L1 cache              1 second
RAM                   3 minutes
SSD random read       9 hours        (NVMe-modern)
Datacenter RTT        ~12 days
SSD sequential MB     ~23 days
HDD seek              ~7.5 months
Cross-continent RTT   ~9.5 years
```

That ratio — *milliseconds across an ocean, nanoseconds inside a CPU* — is why systems design is dominated by *where* data lives, not by clock speed.

---

## 3. The Memory Hierarchy

```
   FASTEST
   │
   │  CPU registers          ~ 1 ns
   │  L1 cache               ~ 0.5 ns
   │  L2 cache               ~ 7 ns
   │  L3 cache               ~ 30 ns
   │  Main memory (DRAM)     ~ 100 ns      ── persistence boundary ──
   │  Optane / PMEM          ~ 300 ns       (mostly retired)
   │  NVMe SSD               ~ 16 µs
   │  SATA SSD               ~ 100 µs
   │  Spinning disk          ~ 10 ms
   │  Tape / cold storage    ~ seconds
   │  Cross-region replica   ~ 50–250 ms
   │
   SLOWEST
```

Each layer is **~10–100×** slower than the one above. Cache hits at one layer cost ~1 layer below; misses fall through. *That's the entire game.*

---

## 4. Network Numbers

| Operation | Typical time |
| --- | --- |
| Localhost loopback RTT | 0.05 ms |
| Same rack (TOR switch) | 0.1 ms |
| Same AZ | 0.5 ms |
| Same region, different AZ | 1–2 ms |
| Same continent | 20–80 ms |
| Cross continent | 80–200 ms |
| Geosynchronous satellite RTT | ~500 ms |
| Starlink (LEO) RTT | 30–50 ms |
| DNS lookup (cold) | 20–120 ms |
| TLS 1.3 handshake (1-RTT) | ~1 RTT extra |
| TCP handshake (3-way) | 1 RTT |
| QUIC 0-RTT resumed | ~0 ms extra |

### Bandwidth
```
1 Gbps   → 125 MB/s   → "1 GB takes 8 sec"
10 Gbps  → 1.25 GB/s
25 Gbps  → 3.1 GB/s   (typical modern cloud NIC)
100 Gbps → 12.5 GB/s  (top-tier cloud / DC backbone)
```

---

## 5. Throughput Numbers for Real Systems

These are *order-of-magnitude*; tuning shifts them ~10× either way.

| System | Typical throughput per node |
| --- | --- |
| Single Postgres (writes) | 5k–10k/s |
| Single Postgres (reads w/ index+cache) | 30k–50k/s |
| Single MySQL (writes) | 5k–10k/s |
| Single Redis (GET/SET) | 100k/s, ~1M with pipelining |
| Single Memcached | ~1M ops/s |
| Single Kafka broker | 100k–1M msgs/s |
| Single Nginx | 50k–100k RPS |
| Single Cassandra node | 10k–50k writes/s |
| Single DynamoDB partition | 1k WCU, 3k RCU (~strongly consistent) |
| Single S3 prefix | 3.5k PUT/s, 5.5k GET/s |
| Single ElasticSearch shard | ~1k indexes/s, ~10k searches/s |

If your QPS exceeds these, you need to **shard, cache, or queue** — that's the whole "scale" toolbox.

---

## 6. Storage Cost Rules of Thumb (2025 cloud prices, rounded)

| Tier | $/GB/month | Use case |
| --- | --- | --- |
| RAM (in-memory cache like ElastiCache) | $5–20 | Hot data |
| NVMe SSD (instance store) | ~$0.10 | Fast working set |
| Block storage (EBS gp3) | ~$0.08 | DB volumes |
| Object storage standard (S3) | ~$0.023 | App data |
| Object storage IA | ~$0.012 | Infrequently accessed |
| Glacier / Archive | ~$0.004 | Compliance, cold |
| Tape | ~$0.001 | Deep cold |

Egress (data leaving cloud) is the silent killer: ~$0.09/GB to internet on AWS — orders of magnitude more than storage itself. That single fact shapes a lot of architectures.

---

## 7. Putting It Together: A Latency Budget

Your user opens an app. They expect a response in ~100 ms. Here's where it goes:

```
TLS handshake (cold)        50 ms
DNS lookup (cold)           20 ms
Cross-Atlantic round trip   80 ms
Load balancer hop           1 ms
App processing              5 ms
DB query                    2 ms
Cache lookup                0.5 ms
JSON serialization          1 ms
TCP/TLS back to client     ~ same as above
```

If the user is in Europe and your DB is in US-east, the **speed of light alone** eats 80 ms — there's literally nothing you can do in software. The fix is *physical*: replicate closer.

This is why latency numbers shape architecture: they tell you where the speed of light forces your hand vs. where careful engineering can help.

---

## 8. The "Rule of 5" Mnemonic

Each of these is **roughly 5× larger** than the previous:

```
L1 (~0.5 ns) → L2 (~7 ns)          ≈ 14×
RAM (~100 ns) → NVMe SSD (~16 µs)  ≈ 160×
SSD (~16 µs) → DC RTT (~500 µs)    ≈ 30×
DC RTT (~0.5 ms) → cross-cont RTT  ≈ 300×
```

So the safest mental model is: **each "layer" is 10–100× slower than the one before it.** That's enough to drive design choices.

---

## 9. Common Mistakes

- Mixing bits and bytes. Always double-check.
- Treating "RAM" and "SSD" as interchangeable when one is 100× faster.
- Forgetting RTT dominates everything between regions. No clever cache wins back the speed of light.
- Forgetting *fan-out* multiplies latencies — a slow downstream slows every parent service.
- Using the "happy path" latency for SLO planning instead of the *tail* (p99/p99.9).
- Designing for steady state and ignoring cold-cache, cold-start, and warm-up.

---

## 10. The Cheat Sheet (print this)

```
╭─────────────────────────────────────────────────────────────╮
│  POWERS OF TEN                                              │
│    10^3 K   10^6 M   10^9 B   10^12 T                       │
│                                                             │
│  SIZES                                                      │
│    char 1B   int 4B   uuid 16B   tweet 1KB                  │
│    page 100KB   photo 500KB   HD min 37MB                   │
│                                                             │
│  LATENCY (memorize the orders of magnitude)                 │
│    L1   0.5 ns          RAM   100 ns                        │
│    SSD  10–150 µs       DC RTT 0.5 ms                       │
│    Same-cont RTT 40 ms  X-cont RTT 150 ms                   │
│                                                             │
│  THROUGHPUT PER NODE                                        │
│    Postgres ~10k W/s     Redis ~100k–1M/s                   │
│    Kafka ~1M msgs/s      Nginx ~100k RPS                    │
│                                                             │
│  NET                                                        │
│    1 Gbps = 125 MB/s    bytes ≠ bits (×8)                   │
│                                                             │
│  RULES                                                      │
│    Reads ~ 10–100× writes     Peak ~ 3–10× avg              │
│    Each layer ~10–100× slower than the one above            │
╰─────────────────────────────────────────────────────────────╯
```

---

## 11. Resources

### Definitive
- **Interactive Latency Numbers** — Colin Scott: <https://colin-scott.github.io/personal_website/research/interactive_latency.html>
- **Jeff Dean LADIS 2009 slides** (original): <https://www.cs.cornell.edu/projects/ladis2009/talks/dean-keynote-ladis2009.pdf>
- **Hennessy & Patterson, *Computer Architecture: A Quantitative Approach*** — the textbook on memory hierarchy.

### Articles
- "Latency Numbers Every Programmer Should Know" — many adaptations; this one is well-maintained: <https://gist.github.com/jboner/2841832>
- Brendan Gregg, "Systems Performance" — <https://www.brendangregg.com/>
- AWS, "Cost of cross-region data transfer" — read the egress pricing page: <https://aws.amazon.com/ec2/pricing/on-demand/#Data_Transfer>

### Videos
- ByteByteGo: "Latency numbers you should know" — <https://www.youtube.com/@ByteByteGo>
- "Latency: the new web performance bottleneck" — Ilya Grigorik

### Calculators
- Wolfram Alpha — sanity-check unit conversions: <https://www.wolframalpha.com/>
- AWS pricing calculator — <https://calculator.aws/>

---

*Previous:* [← Back-of-the-Envelope Estimation](./estimation.md)  |  *Next:* [Throughput vs Latency vs Bandwidth →](./throughput-latency-bandwidth.md)
