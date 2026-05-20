# Erasure Coding vs Replication

> **TL;DR** — Two ways to keep data alive when disks die. **Replication** stores N full copies (typically 3) — simple, fast on reads and writes, and expensive (200%+ storage overhead). **Erasure coding (EC)** splits data into k chunks and adds m parity chunks, so any k of the k+m chunks can rebuild the data — much cheaper storage (often 30–60% overhead) but higher CPU/IO cost on writes, repairs, and small reads. Hot data uses replication; cold data uses erasure coding; everyone does both. **Reed-Solomon** is the dominant EC scheme; **3× replication** is the dominant replication scheme. The decision is durability target × cost × read latency, and the right answer is almost always *replication for hot, EC for cold*, which is exactly what S3, Colossus, Tectonic, Ceph, and HDFS all do under the hood.

---

## 1. The Problem

Disks fail. Datacenters fail. Racks lose power. Bit rot happens. A storage system that says "11 nines of durability" is making a statistical claim about how rare data loss is across all these failure modes.

You have two main tools to make data survive:

1. **Replication** — store N independent copies on N different failure domains.
2. **Erasure coding** — encode the data so that any k of n shards can reconstruct it.

Both are forms of redundancy, but the math, cost, and operational profile differ dramatically.

```
REPLICATION                  ERASURE CODING (e.g., RS(6,3))
─────────────                ──────────────────────────────
[D][D][D]                    [d1][d2][d3][d4][d5][d6][p1][p2][p3]
3 copies of D                6 data shards, 3 parity shards
Lose 2 → still recoverable   Lose any 3 → still recoverable
Storage overhead: 200%       Storage overhead: 50%
Reads: any copy works        Reads: typically just 1 shard works
                             if you read the right one; full
                             reconstruction needs k shards
Repair: copy one full copy   Repair: read k shards, decode,
                             write back the missing one
```

---

## 2. Replication — The Default

The simplest answer to "what if a disk dies?" is "have another disk with the same data."

### How it works
- Pick a replication factor (RF). Common values: 2, 3, 5.
- Place each replica on a different failure domain — different node, ideally different rack, sometimes different AZ.
- On write, send to all replicas (sync or async); on read, any replica suffices.

### Failure tolerance
With RF = N, you can lose **N - 1** replicas before the data is gone. RF = 3 survives any 2 simultaneous failures.

### Storage overhead
**(RF - 1) × 100%**. RF = 3 means you store **3× the logical data**, i.e., **200% overhead**.

For a 1 PB dataset:
- RF = 2 → 2 PB stored.
- RF = 3 → 3 PB stored.
- RF = 5 → 5 PB stored.

### Performance
- **Reads**: any replica works → naturally load-balanced.
- **Writes**: need to durably write to enough replicas to satisfy your consistency policy. Common patterns:
  - **All-replicas sync**: every replica acks before commit. Highest durability, highest latency.
  - **Quorum** (W of N): wait for majority. Most distributed systems use this. See [Quorum-Based Replication](../08-distributed-systems/quorum.md).
  - **Primary + async**: primary acks immediately, replicas catch up. Lowest latency, weakest durability.

### Repair
Detect a missing replica → pick a healthy node → copy the data over. Repair traffic is proportional to one full copy.

### Where it's used
- Cassandra (RF=3 default, often RF=5 for critical keyspaces)
- DynamoDB (3 replicas across 3 AZs)
- HDFS (3× default replication for hot data)
- Postgres / MySQL with streaming replicas
- Kafka (default replication factor 3 for production)
- Redis Sentinel / Cluster

### Strengths
- Simple, well-understood, easy to debug.
- Reads are cheap and parallelizable.
- Writes have low CPU cost.
- Recovery is straightforward: copy one full chunk.

### Weaknesses
- Storage cost. 200% overhead is brutal at PB scale.
- All copies eventually drift slightly if writes are async — consistency is its own problem.

---

## 3. Erasure Coding — The Cheap Tier

Erasure coding takes a different approach: instead of storing N copies, store the data plus enough mathematical redundancy that you can reconstruct it from a subset.

### The basic idea — Reed-Solomon

```
Data D is split into k equal-size shards: d1 d2 d3 d4 d5 d6
Compute m parity shards using Galois-field arithmetic: p1 p2 p3

Store all 9 shards on 9 different failure domains.

Lose any (m) of the (k+m) shards → reconstruct from any k.
```

In **RS(k, m)** notation:
- k = data shards
- m = parity shards
- n = k + m = total shards
- Survives up to m losses.
- Storage overhead = m/k.

Common configurations:

| Code | k | m | Total | Overhead | Failures tolerated |
|---|---|---|---|---|---|
| RS(2, 1) | 2 | 1 | 3 | 50% | 1 |
| RS(4, 2) | 4 | 2 | 6 | 50% | 2 |
| RS(6, 3) | 6 | 3 | 9 | 50% | 3 |
| RS(10, 4) | 10 | 4 | 14 | 40% | 4 |
| RS(12, 4) | 12 | 4 | 16 | 33% | 4 |
| RS(17, 3) | 17 | 3 | 20 | 17.6% | 3 |

S3 has publicly described using 17+3 schemes for some classes; Google Colossus uses RS(8,4), RS(6,3), and others. Facebook Tectonic uses RS(9,6) and RS(10,4) variants for different temperatures.

### How encoding works (in 30 seconds)

Treat each shard as a polynomial coefficient over a finite field (GF(2^8) for byte-level, or larger for word-level). The parity shards are evaluations of that polynomial at fixed points. Reconstruction is polynomial interpolation — given any k points, you can recover all the coefficients.

The math is beautifully self-correcting: as long as you have k surviving shards, you can recover the original k data shards regardless of *which* m were lost.

### Encoding/decoding cost
- **Encode** k data → k+m output: O(k·m) bytes processed plus matrix multiplication in the Galois field. Modern CPUs with AVX2 / AVX-512 SIMD reach 5–20 GB/s. Hardware accelerators (ISA-L from Intel, Jerasure library) help further.
- **Decode** (when shards missing): more expensive — matrix inversion, then re-multiplication. Costly enough to motivate "locally repairable codes" (see below).

### Repair cost — the dirty secret
Repairing one missing shard requires reading **k other shards** and decoding. For RS(10, 4):
- Lose 1 disk → read 10 disks worth of data to reconstruct → write 1 disk back.

That's a 10× read amplification during repair. In a healthy cluster this is fine; in a partially-degraded cluster (which is when you most need repair), it can saturate networks and trigger cascading failures.

**Locally repairable codes (LRC)** mitigate this:
- Microsoft Azure Storage's LRC(12, 2, 2) reads only 6 shards to repair a single failure.
- Facebook HDFS-LRC: similar idea.
- Hierarchical codes: a small set of "local" parities for single-disk repairs + a larger "global" parity for multi-disk failures.

Read amplification on repair is the #1 production cost of EC. LRC trades a slightly worse storage overhead for dramatically better repair behavior.

### Read performance
- **Sequential read of a whole object**: read any k shards in parallel. Total bytes read = original data size; throughput is excellent.
- **Small / single-block read**: in pure RS, you read one specific shard if available (just data, no decode); if that shard is offline, you have to read k others and reconstruct. Hot reads on degraded systems are painful.
- **Latency-sensitive reads**: replication wins. EC adds tail latency from waiting on the slowest of k shards.

### Where it's used
- S3 Standard, S3-IA, Glacier — Reed-Solomon variants under the hood.
- Google Colossus — RS-based; multiple codes per file class.
- Facebook Tectonic, HDFS-EC, Apache Ozone — RS for cold tier.
- Ceph (EC pools).
- MinIO (Reed-Solomon striping by default).
- Backblaze B2 (RS(17, 3)).
- Backup / archive systems (Restic, Borg, etc. — different math but similar idea).

### Strengths
- Dramatic storage savings — 30–60% overhead vs 200% for 3× replication.
- Same or better failure tolerance per byte.
- Cheaper at scale: for PBs of cold data, EC saves real money.

### Weaknesses
- Expensive repair (read k shards to fix 1).
- Higher CPU cost on encode (and especially decode).
- Worse small-read latency on degraded systems.
- Implementation complexity — bugs in EC have been found in production systems for years after launch.
- Worse write latency: must wait for the slowest of n shards.

---

## 4. Replication vs Erasure Coding — Side by Side

| Property | 3× Replication | RS(6, 3) | RS(10, 4) | RS(17, 3) |
|---|---|---|---|---|
| Total shards | 3 | 9 | 14 | 20 |
| Survives losses | 2 | 3 | 4 | 3 |
| Storage overhead | 200% | 50% | 40% | 17.6% |
| Bytes read for whole object | 1× | 1× (read 6/9) | 1× (read 10/14) | 1× (read 17/20) |
| Bytes read for repair (1 lost) | 1× (one full copy) | 6× | 10× | 17× |
| Write fan-out | 3 | 9 | 14 | 20 |
| Encode CPU | none | low–medium | medium | medium |
| Decode CPU (degraded read) | none | medium | medium–high | high |
| Best for | Hot, latency-sensitive | Warm/cold | Cold | Very cold / archive |

The pattern is unavoidable: **higher k/m ratios save more storage but make repair more expensive and reads on degraded systems slower**. Production codes pick a balance for the access pattern.

---

## 5. Durability — How Much Better?

Replication and EC both deliver many nines of durability — what changes is the *cost*. Rough calculations (assuming uncorrelated failures, AFR = 1%/year per disk):

| Scheme | Annual durability target |
|---|---|
| RF=1 (no redundancy) | ~99% |
| RF=2 | ~99.99% |
| RF=3 | ~99.999% (5 nines) |
| RS(6, 3) | ~99.9999999% (9 nines) |
| RS(10, 4) | ~99.999999999% (11 nines) |

Real production systems achieve 11+ nines because they also:
- Spread shards across **independent failure domains** (rack, AZ, region).
- **Scrub** data continuously, repairing detected corruption.
- **Geo-replicate** for region-level resilience.
- **Snapshot** for application-level errors (durability doesn't help if a bug deletes everything).

The math says EC can be more durable than 3× replication. The economics say it should be cheaper, too. That's why every hyperscaler uses EC for everything below the hot tier.

---

## 6. Hybrid Strategies in Practice

The right answer is almost never pure one or pure the other. Production systems combine them:

### S3
- Hot writes land replicated across AZs for low-latency durability.
- Older / colder objects re-encoded with stronger EC schemes.
- The exact scheme is internal, but talks describe at least 17+3 for cold tiers.

### HDFS
- Default: 3× replication for everything.
- HDFS-EC (3.0+): switch cold directories to RS(6, 3) or RS(10, 4).
- Storage policies (`hot`, `warm`, `cold`, `ec`) on a per-directory basis.

### Ceph
- Replicated pools for hot RBD / CephFS workloads.
- EC pools for cold buckets in RGW (object).
- Cache tier in front of EC pool to absorb hot working set.

### Google Colossus
- Multiple codes coexist. New writes start in one scheme; data ages into colder codes.
- Codes optimized for repair bandwidth in their datacenter networks.

### Facebook Tectonic
- Different codes per workload class (warehouse, blob, ML).
- Re-encoding moves data between codes as access patterns change.

### MinIO
- EC-only by default — uses Reed-Solomon striping (default n=8, with parity shards based on cluster size).
- Allows configurable parity per bucket.

### Backblaze
- Famously open about their architecture. RS(17, 3) on a single "Vault" of 20 storage pods, each on a different rack.

---

## 7. The Operational Reality

### Failure correlation
The math of EC and replication assumes **independent** failures. Real failures correlate:
- A bad batch of drives fails together.
- Power events take out a rack.
- Network partitions isolate AZs.
- Bugs in the storage software hit all nodes simultaneously.

This is why **placement matters as much as the code**. RS(6, 3) across 9 disks in one rack is much weaker than RS(6, 3) across 9 racks. AWS, Google, and Azure all carefully place shards across independent power, cooling, and network paths.

### Bit rot and scrubbing
Even healthy disks corrupt data silently (~1 unrecoverable bit per 10^14–10^16 bits read on SATA). Storage systems run **scrubbers** — background processes that read every shard, verify checksums, and repair detected corruption.

EC systems are especially sensitive: a silent corruption is just a "loss" — discovered only when you need to decode. Constant scrubbing catches corruption while reconstruction is still possible.

### Repair storms
When a disk or node dies:
- Replication: copy one stream to one new home. Predictable, bandwidth-bounded.
- EC: read k shards from many nodes, decode, write. Many nodes contribute simultaneously.

A clumsy EC implementation triggers **repair storms** that saturate the network, slow user reads, and risk cascading failures. Production EC systems throttle aggressively and prioritize repairs by risk (data with fewer remaining shards is repaired first).

### Locally repairable codes (LRC)
Microsoft Azure introduced production LRC codes to reduce repair bandwidth:
- LRC(k, l, r): k data + l local parities + r global parities.
- A single failure is repaired by reading only k/l + 1 shards (the local group), not k.
- Trade-off: small storage overhead increase, large repair cost decrease.

Facebook published their HDFS-LRC variant; Azure's Pelican and Mezzanine systems use LRC heavily.

### Read paths on degraded data
The first time you lose a shard in EC, two things happen:
1. Background repair starts.
2. Until repair completes, reads against that shard must reconstruct on the fly.

A well-tuned system completes repair before the next failure. A poorly-tuned one — slow network, busy CPUs, overlapping failures — falls behind and you get cascading data loss.

### Write latency
EC writes wait for all n shards to land (or quorum on shards if the system supports it). This means **write latency = slowest of n shards**, which is worse than **slowest of 3 replicas**. For latency-critical writes, replication wins.

### Reed-Solomon SIMD
Libraries (Intel ISA-L, Jerasure, Klauspost's reedsolomon for Go, leopard-codec, Backblaze blockstor) hand-tune AVX-512 and ARM Neon paths for encode/decode throughput. CPU is rarely the EC bottleneck on modern hardware — the network is. Confirm via profiling.

---

## 8. When to Use Each

```
USE REPLICATION WHEN
  - Latency matters more than $/GB.
  - Reads dominate the workload.
  - Cluster size is small (replication ratios make sense).
  - You want simple operations.
  - Data is hot.

USE ERASURE CODING WHEN
  - Storage cost is the dominant concern.
  - Data is large, cold, or rarely read.
  - You can absorb higher write/repair CPU cost.
  - You have many failure domains (enough to spread n shards).
  - Sequential or batch reads dominate (you read whole objects).

USE BOTH (in tiers) WHEN
  - You have a real range of access temperatures.
  - You're at PB+ scale where the savings move the budget.
  - You can build/buy a system that automates the transition.
```

This last case is almost everyone at scale. S3 does it; Colossus does it; HDFS-EC does it; you should too if you have the data volume.

---

## 9. Worked Numbers — When EC Pays Off

Suppose you have 1 PB of logical data, you store it in the cloud, and your effective cost is dominated by storage. Take typical 2026 cloud object-storage pricing of $0.023/GB-month (hot tier).

```
RF=3 (200% overhead) → 3 PB stored → $69k/month
RS(10, 4) (40%)      → 1.4 PB stored → $32k/month
RS(17, 3) (17.6%)    → 1.18 PB stored → $27k/month
```

That's **$37k–42k/month saved** by EC at 1 PB. At 100 PB, the savings cross the $4M/month line. This is the entire reason hyperscalers built EC pipelines.

For 100 GB of data, the savings are $7/month. Don't bother.

EC is a tool for scale.

---

## 10. Geo-Replication and EC

A single-region EC code (RS(10, 4) within one region) does not survive region-level disasters. To protect against that, you typically:

1. **Replicate the data across regions**, with each region running its own EC code internally.
2. Or use a **geo-EC code** spanning regions — academic and increasingly real (Facebook XOR codes, Cloudflare-style designs).

The second option is cheaper in storage but expensive in cross-region bandwidth. Most production systems stick with "replicated across regions, EC inside each region." This is what S3 Cross-Region Replication amounts to.

See [Multi-Region](../10-scalability/multi-region.md).

---

## 11. Common Mistakes / Anti-Patterns

- **EC for hot data with small reads.** Tail latency dominates; the savings aren't worth it.
- **EC across a single rack or AZ.** Failure correlation breaks the math; you can lose multiple shards in one event.
- **RF = 2 in production.** A single failure leaves you with no redundancy. Three is the production floor.
- **No scrubbing.** Silent corruption accumulates; the math of recovery breaks.
- **Forgetting that repair is its own load.** Big EC clusters need repair bandwidth budgeted alongside user traffic.
- **Using EC for write-amplified workloads.** Each small update has to rewrite all parity shards; you'll burn IOPS.
- **High k/m ratios without LRC.** Repair reads dominate; networks burn out.
- **Same code for all data temperatures.** Hot needs replication, cold needs EC; mixing forces compromises.
- **Trusting EC for application-level mistakes.** EC protects bytes, not "oops I deleted the bucket." Use versioning, snapshots, immutability.
- **Geo-replication via EC across continents** without budget for latency.
- **Treating replication factor as fixed.** Cassandra allows per-keyspace RF; not all data deserves the same.

---

## 12. Decision Tree

```
Is data hot (sub-second reads, small objects, frequent updates)?
   YES → REPLICATION (RF = 3 typical)
Is data warm (occasional reads, mostly read-only)?
   → REPLICATION or moderate EC (RS(6, 3))
Is data cold (rare reads, archival)?
   → AGGRESSIVE EC (RS(10, 4), RS(17, 3))
   → consider LRC for repair efficiency
Is data archival (compliance, never accessed)?
   → EC + tape/Deep Archive
   → cross-region replication of the encoded data

Do you have <10 failure domains?
   → REPLICATION is safer; EC needs enough domains to spread shards.
Do you have <100 GB of data?
   → REPLICATION; EC complexity outweighs savings.

Is correlated failure a real risk (one rack, one PDU)?
   → REPLICATION across distinct domains, no EC inside a single domain.
```

---

## 13. Cheat Card

```
PURPOSE     Keep data alive when disks/nodes/racks die.
            REPLICATION = N full copies.
            ERASURE CODING = k data + m parity shards.

REPLICATION
  Storage overhead   (RF-1) × 100%  (e.g., 200% for RF=3)
  Survives losses    RF - 1
  Repair cost        1× (copy one full replica)
  Strengths          fast reads, simple, low CPU
  Best for           hot data, low latency, small clusters

ERASURE CODING (RS(k, m))
  Storage overhead   m/k  (e.g., 50% for RS(6,3), 40% for RS(10,4))
  Survives losses    m
  Repair cost        k× (read k shards to reconstruct 1)
  Strengths          cheap at scale, more durability per byte
  Best for           cold data, large datasets, sequential reads
  Watch out for      repair storms, small-read latency, correlated
                     failures, high CPU on encode/decode

LRC                  Locally repairable codes; lower repair read
                     amplification at slightly higher storage cost.

HYBRID PATTERN       Hot tier replicated; warm/cold tier EC.
                     S3, Colossus, Tectonic, Ceph, HDFS-EC all do
                     this.

PITFALLS    EC for hot small reads · EC within a single rack ·
            RF=2 in production · no scrubbing · ignoring repair
            bandwidth · same code for all temperatures · trusting
            EC for app-level mistakes

RULE        Hot → replicate. Cold → EC. Place across failure
            domains. Scrub continuously. The savings are real
            but only at scale.
```

---

## 14. Resources

### Papers
- "Reed-Solomon codes" — Reed & Solomon, 1960. Foundational.
- "XORing Elephants: Novel Erasure Codes for Big Data" — Sathiamoorthy et al., VLDB 2013. Facebook's LRC work.
- "Erasure Coding in Windows Azure Storage" — Huang et al., USENIX ATC 2012. Microsoft LRC in production.
- "A Solution to the Network Challenges of Data Recovery in Erasure-coded Distributed Storage Systems" — Rashmi et al., HotStorage 2014.
- "Facebook's Tectonic Filesystem" — FAST 2021.
- "A Brief Primer on Reed-Solomon Codes for Storage" — James Plank tutorials.

### Books
- *Designing Data-Intensive Applications* — Martin Kleppmann. Chapter 5 covers replication; storage durability themes throughout.
- *The Theory of Error-Correcting Codes* — MacWilliams & Sloane. The math, in depth.

### Documentation
- **Backblaze** — public posts on RS(17, 3) Vaults: <https://www.backblaze.com/blog/reed-solomon/>
- **HDFS Erasure Coding** — Apache: <https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HDFSErasureCoding.html>
- **Ceph EC pools** — Ceph docs.
- **MinIO erasure code** — <https://min.io/docs/minio/linux/operations/concepts/erasure-coding.html>

### Articles
- "Erasure Coding for Storage Applications" — James S. Plank tutorial series.
- "Tackling the Repair Problem in Erasure Coding" — UC Berkeley, multiple papers from Kannan Ramchandran's group.
- "How Backblaze Uses Reed-Solomon Codes" — Backblaze blog.
- "Erasure Codes for Storage Systems: A Brief Primer" — Plank, USENIX login 2013.
- Cloudflare R2 / Tigris engineering blogs on object-storage internals.

### Videos
- ByteByteGo — "Replication vs Erasure Coding" overview.
- "Reed-Solomon Codes Explained" — visual explanations on YouTube (Computerphile, others).
- USENIX FAST conference videos — many EC papers and production talks.
- AWS re:Invent — "Deep Dive on Amazon S3" sessions touch on EC at scale.

### Tools
- **Jerasure** — long-standing C library for EC.
- **Intel ISA-L** — fast SIMD-accelerated RS encode/decode.
- **klauspost/reedsolomon** (Go) — high-perf RS library, used in MinIO.
- **leopard-codec** — fast O(n log n) Reed-Solomon for huge n.
- **par2** — Reed-Solomon-based file recovery utility (the EC algorithm most users have actually run).

### Adjacent reading
- [Object Storage (S3, GCS, Azure Blob)](./object-storage.md)
- [Distributed File Systems (HDFS, GFS)](./distributed-file-systems.md)
- [Compaction & Tiered Storage](./compaction.md)
- [Replication (Master-Slave, Master-Master, Multi-Region)](../04-databases/replication.md)
- [Quorum-Based Replication](../08-distributed-systems/quorum.md)
- [Failover & Disaster Recovery](../11-reliability/failover-dr.md)
- [Multi-Region](../10-scalability/multi-region.md)

---

*Previous:* [← Compaction & Tiered Storage](./compaction.md)  |  *Up:* [README ↑](../README.md)
