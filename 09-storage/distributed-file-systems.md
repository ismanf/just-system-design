# Distributed File Systems (HDFS, GFS)

> **TL;DR** — A **distributed file system (DFS)** spreads files across a cluster of cheap commodity machines, replicating chunks for durability and parallelizing reads/writes for throughput. **Google File System (GFS)** introduced the model in 2003; **Hadoop Distributed File System (HDFS)** is the open-source clone that powered the big-data era. Both assume **large files, append-mostly writes, sequential scans, and frequent hardware failures**. They are not POSIX filesystems — they optimize for throughput, not low-latency random IO. Modern systems (S3, Tectonic, Colossus) extend the same ideas to cloud scale. Understanding GFS/HDFS is essential because their design choices — chunking, single metadata server, replication, write pipelines — show up in nearly every large-scale storage system since.

---

## 1. The Idea

A distributed file system makes a fleet of disks look like one giant filesystem. You read and write files; under the hood the system splits them into chunks, places those chunks on different machines, replicates them, and serves clients in parallel.

```
                ┌──────────────┐
                │   Client     │
                └──────┬───────┘
                       │ 1. open("/logs/2026.txt")
                       ▼
                ┌──────────────┐
                │  Metadata    │   ← single (or HA pair) coordinator
                │  Server      │      knows: file → chunk IDs → locations
                └──────┬───────┘
                       │ 2. returns chunk handles + locations
                       ▼
   ┌────────────┐ ┌────────────┐ ┌────────────┐
   │ ChunkServer│ │ ChunkServer│ │ ChunkServer│   ← stores actual data
   │   (DN1)    │ │   (DN2)    │ │   (DN3)    │      typically 3× replicated
   └────────────┘ └────────────┘ └────────────┘
        ▲              ▲              ▲
        └──────────────┴──────────────┘
              3. client streams directly from chunkservers
```

Three roles:
1. **Metadata server** (GFS Master / HDFS NameNode) — owns the namespace, file → block mapping, ACLs.
2. **Chunkservers / DataNodes** — store fixed-size blocks of raw bytes.
3. **Client library** — does the dance: ask metadata, read/write to chunkservers, retry on failure.

This separation of **metadata** and **data** is the single most important architectural choice in DFS history. It enables the cluster to serve massive parallel data throughput while a single brain coordinates placement.

---

## 2. Why It Exists — The 2003 Context

Google had a problem: petabytes of web crawl, billions of pages, jobs that needed to scan it all. The traditional answer (Network Attached Storage, big NAS appliances) was too expensive, too small, and too unreliable at the scales they needed.

The GFS paper laid out the new assumptions:
- **Failures are normal**, not exceptional. With thousands of machines, something dies every day.
- **Files are huge** — gigabytes are the norm, not the exception.
- **Writes are mostly appends**, rarely random rewrites.
- **Reads are mostly large sequential scans.**
- **Bandwidth matters more than latency.**
- **Co-design the application and the filesystem** — POSIX is not a sacred boundary.

Those assumptions still hold for analytics, logs, ML datasets, backups, and most "big data" workloads. They are wrong for OLTP, for desktop filesystems, and for anything latency-sensitive.

---

## 3. GFS — The Original

### Architecture
- **Single Master** (with shadow replicas).
- **Chunkservers** — store 64 MB chunks (much later raised). Each chunk is replicated **3 ways** by default.
- Files are decomposed into chunks identified by a 64-bit handle.

### Key design decisions

**64 MB chunk size (huge for 2003).**
- Reduces metadata: a 1 PB filesystem with 64 MB chunks has 16M chunk entries — fits in master RAM.
- Amortizes TCP setup cost over big sequential transfers.
- Allows persistent TCP from client to chunkserver for many reads.

**Master holds metadata in memory.**
- The entire namespace, ACLs, and file→chunk mappings live in RAM.
- Operations log persisted to disk + replicated; periodic checkpoints.
- This is fast but caps cluster size to whatever the master can hold (originally ~50M files).

**Relaxed consistency.**
- GFS guarantees that successful concurrent appends become atomic, but it does **not** guarantee exact ordering, nor that all replicas have identical bytes — only that data has been written somewhere within the file region. Consumers must tolerate duplicates and pad bytes.
- This was a deliberate trade for throughput. Applications were co-designed: MapReduce knows how to dedupe; BigTable encodes record framing.

**Lease-based primary.**
- For each chunk, the master grants a 60-second lease to one replica designating it as **primary**. Writes go to the primary, which orders mutations and forwards to secondaries.

**Write pipeline.**
```
Client → push data to all replicas (via a chain optimized by network topology)
Client → ask primary to commit
Primary  → assigns serial number → applies → tells secondaries to apply
```
Data flow is decoupled from control flow: bytes flow through the most network-efficient chain (often one replica per rack hop), while commits are coordinated through the primary.

---

## 4. HDFS — The Open-Source Clone

Doug Cutting wrote HDFS as part of Hadoop, the open-source MapReduce stack. It's a faithful clone of GFS with renamed components.

| GFS Term | HDFS Term |
|---|---|
| Master | NameNode |
| Chunkserver | DataNode |
| Chunk (64 MB) | Block (default 128 MB; 256 MB common today) |
| Lease | Lease (same idea) |
| Operation Log | EditLog + FsImage |

### Key features
- **Block size 128 MB by default.** Some installs go to 256 MB or even 1 GB for very large files.
- **Replication factor of 3.** Configurable per file.
- **Rack-aware placement.** First replica on writer's rack; second on a different rack; third on the same different rack as the second. Survives a full rack failure.
- **Single NameNode** with persistent journal + standby (HA via QuorumJournalManager since Hadoop 2).
- **Append-only** initially; appends and limited truncate added later, still no random in-place writes.
- **Write semantics:** one writer per file at a time. No concurrent overwrites.

### Standard write flow
```
1. Client → NameNode: create("/foo.txt")
2. NameNode → returns lease + first block location list [DN1, DN2, DN3]
3. Client → builds pipeline DN1 → DN2 → DN3
4. Client → streams packets to DN1; DN1 forwards to DN2; DN2 to DN3
5. ACKs return up the chain
6. Block full → client requests next block locations
7. Client → close() → NameNode commits
```

### Standard read flow
```
1. Client → NameNode: open("/foo.txt")
2. NameNode → returns ordered block list with replica locations
3. Client → picks "closest" replica (same node > same rack > remote) for each block
4. Client streams blocks in parallel
```

### Strengths
- Massive sequential throughput. Linear scan rates of GB/s per node, aggregated to TB/s in a big cluster.
- Tolerates commodity-disk failure transparently.
- Cheap per TB on bare metal.

### Weaknesses
- **NameNode is a SPOF for scale.** Memory footprint grows linearly with files + blocks. Heap pressure becomes the real limit (1 GB heap ≈ 1M files at typical replication).
- **Small-file problem.** A million 1 KB files still consume 1 million entries in NameNode memory, plus 3M block reports from DataNodes. Big-data clusters dread small-file workloads.
- **No random write.** Append + read; no in-place updates. Layer like HBase needed for OLTP-ish access.
- **Not POSIX.** Standard tools (ls, cp) work via the `hdfs` CLI, not via the kernel.

---

## 5. GFS vs HDFS — Side by Side

| Property | GFS | HDFS |
|---|---|---|
| Year | 2003 | 2006 (open source) |
| Origin | Google internal | Apache / Yahoo |
| Block size | 64 MB (later larger) | 128 MB default |
| Replication | 3× default | 3× default |
| Master | Single + shadow | NameNode + HA standby |
| Metadata in RAM | Yes | Yes |
| Consistency on append | Atomic-ish (may dup, may pad) | Single-writer; clean atomic |
| Random writes | No | No |
| Languages / clients | C++ Google internal | Java + everything-on-Hadoop |
| Closed/Open | Closed | Open (Apache) |

The conceptual model is nearly identical. The implementation details differ in degree, not in kind.

---

## 6. Worked Example — Reading a 1 GB File from HDFS

Block size = 128 MB, replication = 3, file size = 1 GB → 8 blocks.

```
File: /datasets/sales/2025.parquet  (1 GB)

NameNode metadata:
  block_0001 → [DN12, DN3,  DN21]   offsets [0 – 128MB)
  block_0002 → [DN8,  DN17, DN5 ]   offsets [128MB – 256MB)
  block_0003 → [DN3,  DN19, DN12]   ...
  block_0004 → [DN21, DN2,  DN6 ]
  block_0005 → [DN15, DN9,  DN4 ]
  block_0006 → [DN1,  DN12, DN18]
  block_0007 → [DN22, DN7,  DN13]
  block_0008 → [DN10, DN16, DN3 ]

Reader (a Spark executor on host RackA-Node5):
  for each block, pick the closest replica:
    block_0001: DN3   (same-rack)
    block_0002: DN5   (same-rack)
    block_0003: DN3   (same-rack)
    ...
  Stream 8 blocks in parallel → 1 GB delivered at line rate.
```

If DN3 dies mid-read, the client transparently fails over to the next replica (DN12) and re-issues the read for the missing range. The job doesn't fail; it slows down for a few seconds.

---

## 7. Operational Reality

### NameNode memory pressure
The NameNode in HDFS keeps every file, directory, and block in heap. Rough rule of thumb: **150 bytes per file + 150 bytes per block × replication factor.** A 100M-file cluster needs tens of GB of heap, and full GC pauses become operational events. This is why "small files are evil" in HDFS folklore — they multiply heap pressure without buying you any data.

Mitigations:
- **HAR (Hadoop Archive)** files — pack many small files into a single archive.
- **HDFS Federation** — multiple NameNodes, each owning a subtree.
- **Ozone** — Apache project replacing HDFS with an object store, decoupling metadata size from file count.

### Rack awareness
HDFS pipelines depend on accurate rack mapping (via a script or topology config). Wrong mapping → all 3 replicas on the same rack → one rack failure kills the data. Watch this carefully on migrations.

### Balancer
DataNodes accumulate skew over time as some disks fill up. The HDFS Balancer redistributes blocks. Run it regularly, but throttle bandwidth — a runaway balancer will saturate the network.

### Decommissioning
Removing a node properly: mark it decommissioning → NameNode re-replicates its blocks elsewhere → wait for all blocks to be safe → physically remove. Skip the wait at your peril.

### Replication storms
When a DataNode dies, the NameNode schedules re-replication of all blocks that fall under the replication factor. If the cluster is fragile, replicating PB of data hammers the network and can trigger more failures. Tune `dfs.namenode.replication.work.multiplier.per.iteration` and rack-aware throttling.

### Encryption and Kerberos
HDFS supports **transparent encryption zones** (EZ) backed by a KMS, and **Kerberos** for auth. Both are non-optional in regulated environments and notoriously fiddly to deploy.

---

## 8. The Modern Successors

Cloud and post-Hadoop architectures evolved the GFS idea in several directions:

### Google Colossus
The successor to GFS inside Google (announced 2010, papers in 2021). Improvements:
- **Federated metadata** — no single master, scales to exabytes.
- **Erasure coding** as default instead of 3× replication.
- **Strict tenant isolation** via curators.
- **Hierarchical storage** — hot SSD, warm HDD, cold tape — all managed transparently.

### Facebook Tectonic
Unified blob + warehouse storage replacing Haystack + HDFS (paper, 2021):
- **Disaggregated metadata** in a sharded ZippyDB.
- **Single multi-EB filesystem** instead of many smaller clusters.
- **Reed-Solomon erasure coding** for cold data.

### Apache Ozone
Apache's HDFS replacement designed to handle billions of objects:
- **S3-compatible API** plus HDFS-compatible API.
- **Decoupled metadata service** so namespace isn't bound by NameNode RAM.

### S3 (Amazon)
Different design (flat namespace, REST API, no POSIX), but it solves the same problem — scalable, durable, parallelizable file storage on commodity hardware. Modern lakehouses (Delta Lake, Iceberg, Hudi) treat S3 as the storage layer the way Hive treated HDFS.

### Lustre, BeeGFS, WekaFS, IBM Spectrum Scale (GPFS)
Different family: **POSIX-compliant parallel filesystems** optimized for HPC and ML training. Higher random IO, fewer scalability constraints than HDFS, but more expensive and more complex.

---

## 9. When to Use a DFS Today

### Reasons to deploy HDFS (or Ozone)
- You already have a Hadoop / Spark / Hive footprint on-prem.
- You want to keep PBs of data inside your network without cloud egress costs.
- Compliance forbids cloud object storage.
- HPC-style throughput in your own datacenter.

### Reasons not to
- You're in the cloud and S3/GCS exists — you'll save money and operational pain. Run Spark/Trino directly against object storage.
- You have small files and lots of them.
- You need a real POSIX filesystem (use EFS, FSx, Lustre).
- You only have terabytes, not petabytes — HDFS is over-engineered for small clusters.

The honest take: **outside of large on-prem big-data shops, HDFS is in decline.** The cloud lakehouse pattern (object storage + Iceberg/Delta + a compute engine) has eaten its lunch since 2020.

---

## 10. Lessons That Outlived GFS/HDFS

Even where the systems themselves are fading, their design lessons are everywhere:

1. **Separate metadata from data.** Used by S3, Ceph, Tectonic, every modern object store.
2. **Append-mostly, big-chunk writes.** The blueprint for Kafka log segments, Parquet files, LSM-tree SSTables.
3. **Co-design app and storage.** MapReduce, Spark, BigQuery all bake assumptions about the storage layer into the engine.
4. **Replication first, erasure coding when you're sure.** Modern systems start with 3× replication for hot data, switch to EC when access patterns stabilize.
5. **Rack awareness is mandatory** for any multi-host storage that wants to survive correlated failures.
6. **Heap-bound metadata is a ceiling.** Every system that learns this rebuilds metadata to be sharded.

---

## 11. Common Mistakes / Anti-Patterns

- **Treating HDFS like a database.** Random writes, low-latency reads, lots of small files — all anti-patterns.
- **Tiny files.** A million 1 KB files crush the NameNode. Always batch into Parquet, SequenceFile, ORC, Avro, or HAR.
- **Wrong rack topology.** All three replicas land on one rack → one rack outage = data loss. Verify the topology script.
- **Replication factor 1** "to save space." You will lose data. Use erasure coding for cold data instead.
- **No NameNode HA.** A single-master HDFS without standby and quorum journal is a fragile beast. HA has been mandatory since 2014; some legacy clusters still don't have it.
- **Running HDFS in the cloud.** Almost never the right answer — pay for S3, run Spark against it directly.
- **Forgetting the balancer.** Skew grows silently until some DataNodes are 99% full and rejecting writes.
- **Not monitoring under-replicated blocks.** A small number is fine; a growing number is an emergency.
- **Ignoring snapshot semantics.** HDFS snapshots are not full copies; they're shallow point-in-time views, easy to misunderstand.

---

## 12. Cheat Card

```
PURPOSE     Spread huge files across a cluster of cheap machines,
            replicate for durability, scan in parallel for throughput.

DESIGN      Metadata server  +  many chunkservers  +  client library
            File = ordered list of fixed-size chunks (64–256 MB)
            Each chunk replicated 3× across racks
            One writer per file; append-mostly

GFS (2003)  Google's original. Single Master. Relaxed append atomicity.
            Blueprint for MapReduce.

HDFS        Open-source GFS clone. NameNode + DataNodes.
            Block size 128 MB. RF=3 default. Rack-aware placement.

ASSUMPTIONS
            ✓ Big files                ✗ Small files (heap killer)
            ✓ Sequential reads          ✗ Random low-latency IO
            ✓ Append-only writes        ✗ Random in-place writes
            ✓ Failures are normal       ✗ POSIX semantics

PITFALLS    Tiny files · wrong rack topology · no HA NameNode ·
            replication storms · forgetting the balancer ·
            running HDFS in the cloud

RULE        DFS = throughput, not latency. In the cloud, use object
            storage. On-prem with PBs of analytics, HDFS/Ozone still
            earn their keep.
```

---

## 13. Resources

### Papers
- "The Google File System" — Ghemawat, Gobioff, Leung. SOSP 2003. <https://research.google/pubs/pub51/>
- "The Hadoop Distributed File System" — Shvachko et al. MSST 2010. <https://storageconference.us/2010/Papers/MSST/Shvachko.pdf>
- "Facebook's Tectonic Filesystem" — Pan et al. FAST 2021.
- "Colossus under the Hood" — Google blog series, 2021.
- "Apache Ozone" design docs — <https://ozone.apache.org/docs/>

### Books
- *Hadoop: The Definitive Guide* — Tom White. Still the canonical HDFS reference even after Hadoop's decline.
- *Designing Data-Intensive Applications* — Martin Kleppmann. Chapter 10 on batch processing builds on the GFS/HDFS assumptions.

### Articles
- "HDFS Federation" — Apache docs.
- "HDFS Architecture" — Apache docs: <https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HdfsDesign.html>
- "The Small Files Problem" — Cloudera engineering blog.
- "Why HDFS is Dying" — various lakehouse-era essays from Databricks and Snowflake.

### Videos
- ByteByteGo — HDFS architecture walkthrough.
- CMU 15-721 — lectures on distributed storage and shared-nothing analytics.
- MIT 6.824 — covers GFS as a foundational system, with a great paper discussion.

### Tools
- **HDFS CLI** (`hdfs dfs -*`) — basic operations.
- **Apache Ozone** — modern Apache replacement for HDFS.
- **MinIO**, **Ceph**, **Tectonic-inspired** projects for self-hosted object storage.

### Adjacent reading
- [Block, File, and Object Storage](./storage-types.md)
- [Object Storage (S3, GCS, Azure Blob) →](./object-storage.md)
- [Erasure Coding vs Replication →](./erasure-coding.md)
- [MapReduce](../17-big-data/mapreduce.md)
- [Hadoop Ecosystem](../17-big-data/hadoop.md)
- [Data Warehouses & Data Lakes](../04-databases/warehouses-lakes.md)
- [Lakehouse Architecture](../04-databases/lakehouse.md)

---

*Previous:* [← Block, File, and Object Storage](./storage-types.md)  |  *Next:* [Object Storage (S3, GCS, Azure Blob) →](./object-storage.md)
