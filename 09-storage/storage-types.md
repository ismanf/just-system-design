# Block, File, and Object Storage

> **TL;DR** — Three fundamental ways to expose persistent storage. **Block storage** hands you a raw disk (think `/dev/sda`) — fast, flexible, fully under your control; this is where databases and VMs live. **File storage** hands you a hierarchical filesystem accessed over a network (NFS/SMB) — shared mutable files with POSIX semantics. **Object storage** hands you an HTTP API over a flat namespace of immutable blobs with rich metadata (S3, GCS) — infinite capacity, cheap, but high per-request latency. Pick block for IOPS-heavy structured data, file for shared scratch and lift-and-shift legacy apps, object for everything else. The industry has spent 20 years moving as much as possible to object storage.

---

## 1. The Big Picture

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  Application                                                        │
│      │                                                              │
│      │ writes "user_avatar.jpg"                                     │
│      ▼                                                              │
│  ┌─────────────┐    ┌─────────────┐    ┌──────────────────────┐    │
│  │   BLOCK     │    │    FILE     │    │      OBJECT          │    │
│  │             │    │             │    │                      │    │
│  │  raw disk   │    │  /share/    │    │  bucket/key          │    │
│  │  iSCSI/FC   │    │  NFS/SMB    │    │  HTTPS REST API      │    │
│  │             │    │             │    │                      │    │
│  │  EBS, GCE   │    │  EFS, FSx,  │    │  S3, GCS, Azure Blob │    │
│  │  PD, SAN    │    │  Filestore  │    │  R2, B2, MinIO       │    │
│  └─────────────┘    └─────────────┘    └──────────────────────┘    │
│       low-level         middle             high-level               │
│       fastest           shared             cheapest at scale        │
└─────────────────────────────────────────────────────────────────────┘
```

All three exist because they optimize for different goals — latency, sharing semantics, and economics. A real system usually uses all three.

---

## 2. Block Storage

A **block device** is a fixed-size array of equal-size blocks (typically 512 B or 4 KiB). The storage system has no idea what a "file" is — it gives you a numbered grid of slots, and you read or write them by offset.

```
LBA 0   LBA 1   LBA 2   LBA 3   ...   LBA N
[block][block][block][block]  ...  [block]
```

A filesystem (`ext4`, `xfs`, `NTFS`) or a database (Postgres, MySQL) is what turns those blocks into files, tables, indexes, etc.

### How you get it
- **Local disk** — NVMe, SATA SSD, spinning rust attached to the machine.
- **Network-attached block** — iSCSI, Fibre Channel, NVMe-oF. The disk lives in a SAN, but the OS sees `/dev/sdX` and treats it like local.
- **Cloud block** — AWS EBS, GCP Persistent Disk, Azure Managed Disk. Networked block storage that pretends to be a local disk.

### Properties
- **Lowest-level abstraction** — you control filesystem, block size, alignment, caching.
- **Single-attach** by default. Most cloud block volumes can be attached to one VM at a time (multi-attach exists but is restricted).
- **Predictable IOPS and latency** — sub-millisecond for SSD, can be provisioned (io2, hyperdisk).
- **Mutable in place** — overwrite block 12345 and it changes.
- **Resizing is painful** — usually grow-only, requires filesystem-level extension.

### When to use
- Databases (Postgres, MySQL, MongoDB) — they need random IO and a filesystem.
- VM root disks and boot volumes.
- High-IOPS workloads (Kafka log segments, Elasticsearch shards).
- Anything that wants POSIX semantics with single-host access.

### When not to use
- Anything that needs to be shared across many machines (use file or object).
- Truly unbounded data (block scales by attach quotas, not data volume).
- Archive / cold data (it's 5–20× more expensive than object).

---

## 3. File Storage

A **network filesystem** exposes the familiar hierarchical tree — directories, files, permissions, locks — over a wire protocol. Multiple clients mount the same share and see the same files.

```
/share/
├── projects/
│   ├── alpha/
│   │   └── report.pdf
│   └── beta/
└── home/
    └── ismayil/
        └── notes.md
```

### How you get it
- **NFS** (v3 / v4) — Unix-flavored; the dominant protocol in cloud (EFS, Filestore, FSx for NetApp).
- **SMB / CIFS** — Windows-flavored; FSx for Windows, Azure Files.
- **POSIX-ish managed services** — AWS EFS, GCP Filestore, Azure Files, FSx for Lustre, Isilon.

### Properties
- **Shared mutable access** — concurrent reads and writes from many hosts.
- **POSIX semantics** — rename, lock, append, change permissions. Apps written for local files mostly "just work".
- **Hierarchical namespace** — directories, walks, globbing.
- **Latency hides the network** — single-file ops are millisecond-class; small-file metadata storms can crawl.
- **Not infinite** — EFS is logically unlimited, but throughput scales with stored data; Filestore tiers have hard ceilings.

### When to use
- **Lift and shift** — legacy apps that expect a local mount point.
- **Shared content for a cluster** — render farms, ML training datasets, CI artifacts, web app uploads served across pods.
- **Build, scratch, and home directories** in HPC and dev environments.
- **Containers that want shared volumes** across nodes (RWX in Kubernetes).

### When not to use
- Web-scale read-after-write of immutable blobs (use object).
- Database storage (use block — file's metadata operations and locking semantics will betray you).
- Multi-region active-active with strong consistency. NFS does not span regions well.

### Sharp edges
- **fsync over NFS** is honored differently by clients — some lie about durability.
- **File locking** (flock, lockd) over NFS is famously fragile.
- **Small-file workloads** kill performance because metadata ops dominate. One million 1 KB files is much worse than one 1 GB file.
- **Eviction and tiering** are limited compared to object storage.

---

## 4. Object Storage

An **object** is a self-contained blob — bytes + metadata — addressed by a key inside a flat **bucket** namespace. You read and write whole objects (or specified byte ranges) over HTTPS.

```
PUT  https://bucket.s3.amazonaws.com/users/42/avatar.jpg
GET  https://bucket.s3.amazonaws.com/users/42/avatar.jpg
HEAD https://bucket.s3.amazonaws.com/users/42/avatar.jpg
```

There are no directories — `/users/42/avatar.jpg` is one opaque key. The "/" in the key is purely a UI convention.

### Properties
- **Flat namespace, REST API** — no mounts, no filesystem driver.
- **Effectively unlimited** — objects up to 5 TB (S3), buckets sized in petabytes routinely.
- **Eleven nines of durability** (11×9 = 99.999999999%) achieved with replication + erasure coding under the hood.
- **Immutable-by-convention** — you can overwrite a key, but it replaces the object atomically; there's no "edit offset 4096" operation.
- **Rich metadata** — user-defined headers, content-type, versioning, lifecycle policies, server-side encryption, ACLs.
- **High base latency** — first byte typically 30–100 ms even within a region. Throughput per request is high but RTT-bound.
- **Strong read-after-write consistency** on the major clouds since 2020 (S3 made the switch in December 2020).
- **Cheap.** S3 Standard is about $0.023/GB-month; Glacier Deep Archive is ~$0.00099/GB-month — roughly 25× cheaper than EBS gp3 and 1000× cheaper than provisioned IOPS at rest.

### When to use
- **Static assets** — images, video, documents, JS bundles.
- **Backups and archives.**
- **Data lake** — Parquet/ORC files for analytics (S3 + Athena, BigQuery + GCS).
- **ML training data and model checkpoints.**
- **Log archives** after rotation out of hot stores.
- **Anywhere "write once, read many" describes the access pattern.**

### When not to use
- Hot transactional storage with millisecond latency budgets.
- Append-in-place workloads (object storage does append via "multipart upload" or "compose", which is awkward).
- Filesystem-rename semantics (rename in S3 is copy + delete, expensive at directory scale).
- Anything that needs hierarchical locking or POSIX permissions.

---

## 5. Side-by-Side Comparison

| Property | Block | File | Object |
|---|---|---|---|
| **Abstraction** | Raw blocks (LBA) | Hierarchical FS | Flat bucket + key |
| **Protocol** | iSCSI, FC, NVMe, local | NFS, SMB | HTTP/S REST |
| **Unit** | Block (512B–4KiB) | File | Object (≤5 TB on S3) |
| **Mutability** | Random in-place writes | Random writes, locks | Replace whole object |
| **Sharing** | Usually single host | Many hosts | Many hosts |
| **Latency** | <1 ms (local SSD) | 1–10 ms | 30–100 ms first byte |
| **Throughput per stream** | 100s MB/s – GB/s | 100s MB/s | Parallel only; 10s MB/s per request, GB/s aggregated |
| **Capacity ceiling** | TB scale per volume | TB–PB per share | Effectively unlimited |
| **Cost per GB-month (cloud)** | $0.08–$0.20 | $0.30 | $0.005–$0.023 (hot), <$0.005 (cold) |
| **Durability target** | 99.999% (volume) | 99.999999999% (zonal+) | 99.999999999% (11×9) |
| **Consistency** | Linearizable per block | Close-to-POSIX | Strong read-after-write |
| **Best for** | DBs, VMs, IOPS | Shared mutable trees | Blobs, archives, lakes |
| **Worst for** | Sharing | Web-scale assets | Random in-place writes |
| **Canonical AWS** | EBS | EFS, FSx | S3 |
| **Canonical GCP** | Persistent Disk | Filestore | GCS |
| **Canonical Azure** | Managed Disks | Azure Files | Blob Storage |

---

## 6. The Cost Ladder

Order-of-magnitude prices (US, AWS, mid-2025, hot tier):

```
Provisioned IOPS block (io2)  ~$0.20/GB-month + IOPS charges
General-purpose block (gp3)   ~$0.08/GB-month
NFS file (EFS Standard)       ~$0.30/GB-month
Object — hot (S3 Standard)    ~$0.023/GB-month
Object — IA  (S3 Standard-IA) ~$0.0125/GB-month
Object — archive (Glacier IR) ~$0.004/GB-month
Object — deep archive         ~$0.001/GB-month
```

The cost gradient drives architecture. The closer to bare blocks you go, the more you pay, but you also get more performance and finer control. As soon as data goes "cold," the right move is almost always to push it to object storage.

---

## 7. How They Combine in Real Systems

A typical SaaS application uses all three:

```
┌──────────────────────────────────────────────────────────────┐
│                                                              │
│  Web app pods (K8s)                                          │
│      │                                                       │
│      ├─ writes user uploads ─────────────► OBJECT (S3)       │
│      │   (avatars, attachments, backups)                     │
│      │                                                       │
│      ├─ reads/writes config from ────────► FILE (EFS)        │
│      │   shared mount (rare in modern apps)                  │
│      │                                                       │
│      ▼                                                       │
│  Postgres primary  ─── stores data on ───► BLOCK (EBS io2)   │
│      │                                                       │
│      └─ ships WAL archives ──────────────► OBJECT (S3)       │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

- **Block** under the database for IOPS.
- **Object** for user uploads, backups, archives, analytics data, ML datasets.
- **File** only when something legacy demands it.

This pattern is so dominant that "cloud-native" usually means "stateless compute + block-backed DB + everything else on object."

---

## 8. Under the Hood — What's Really There

All three are usually built on top of object-like storage at the lowest level:

- **Cloud block storage** (EBS) is implemented as a replicated chunked store backed by erasure-coded blob pools. The "disk" you see is an illusion built from many shards across many physical disks.
- **Cloud file storage** (EFS) is a metadata service plus a chunked data plane — much like a giant distributed filesystem.
- **Object storage** (S3) is itself a distributed key-value store layered on erasure-coded fragments.

Hardware doesn't actually have three storage types. The abstractions exist to fit different access patterns onto the same underlying replicated, erasure-coded substrate.

---

## 9. Throughput vs IOPS vs Latency — Which Matters

| Workload | Dominates | Pick |
|---|---|---|
| OLTP database (random reads/writes) | IOPS + latency | Block (provisioned IOPS) |
| Log streaming (sequential writes) | Throughput | Block (gp3 throughput-tuned) or Object multipart |
| Stream analytics scan | Throughput | Object (parallel range reads) |
| Static assets, low fan-out | Cost + cache hit rate | Object + CDN |
| ML training (read-heavy, parallel) | Aggregate throughput | Object with prefetch, or FSx for Lustre |
| Real-time leaderboard | Latency | Block-backed Redis, not file/object |

---

## 10. Operational Reality

### Block
- **Snapshot semantics matter.** EBS snapshots are incremental but the first snapshot is full; budget for it.
- **Resize is one-way.** Most cloud block volumes can only grow.
- **Throughput is metered.** gp3 caps at 1 GB/s; io2 Block Express goes higher but at price.
- **IOPS bursting** (gp2, gp3 baseline) deceives benchmarks — sustained throughput is often half of peak.

### File
- **Throughput scales with stored data** (EFS bursting model) — small shares are slow. Provisioned throughput exists but is pricey.
- **Backups are awkward.** Restoring a 50 TB share takes time; rsync at scale is brutal.
- **NFS clients differ** — Linux `mount.nfs` options (`hard`, `soft`, `nconnect=16`) have major correctness/perf implications.

### Object
- **Request cost matters.** PUT, COPY, POST, LIST: ~$0.005 per 1,000. A pathological logging system that writes one tiny object per request can spend more on API calls than on storage.
- **Prefix sharding** — S3 partitions internally by key prefix. Heavy traffic on `logs/2026-05-19/...` can hit per-prefix request limits. Spread writes across prefixes.
- **List operations are paginated** and slow at scale; treat object listing as a batch job, not a query.
- **Eventual delete propagation** on versioned buckets — old versions linger until lifecycle rules clean them up.
- **Multipart upload** above ~100 MB is mandatory for resilience and parallel throughput.

---

## 11. Hybrid and Specialty Variants

- **NVMe-over-Fabrics** — networked block at near-local latencies. Used inside hyperscaler fleets.
- **FSx for Lustre** — block-backed parallel filesystem for HPC and ML, with S3 integration for spillover.
- **FUSE-mounted object storage** (`s3fs`, `goofys`, `mountpoint-for-s3`) — exposes a bucket as a filesystem. Convenient but not POSIX-correct. Useful for read-mostly workloads.
- **WekaFS, VAST Data, Pure Storage FlashBlade** — high-performance hybrid file/object appliances for AI training.
- **Apache Ozone, MinIO, Ceph, OpenIO** — self-hosted S3-compatible object stores. Ceph also provides block (RBD) and file (CephFS) from one cluster.

---

## 12. Common Mistakes / Anti-Patterns

- **Putting a database on NFS** — locking, fsync, and rename semantics will eventually corrupt your data. Use block.
- **Treating S3 like a filesystem** — `aws s3 mv` on a "directory" copies and deletes each key one by one. List-then-copy on 10 million keys is a day-long job.
- **Many tiny objects** — a million 1 KB objects costs more in PUT requests than the storage itself, and listing them is painful. Pack into Parquet or tar.
- **Ignoring S3 prefix hot spots** — monotonic-timestamp keys (`logs/2026-05-19T12:00:01Z/...`) concentrate load on one partition. Hash the prefix.
- **Using EFS as a database backing store** — fsync over NFS is a minefield, and lock semantics differ from local.
- **Forgetting lifecycle policies** — buckets without expiration grow forever, including the bills.
- **Cross-region egress on hot reads** — object storage is regional; serve via CDN or replicate, don't pay egress.
- **Mounting many EFS clients without `nconnect`** — single TCP connection caps your throughput.
- **Treating block snapshots as backups** — they live in the same account/region; an account compromise wipes them. Replicate snapshots out.

---

## 13. Decision Rule

```
Need POSIX, single host, IOPS?       → BLOCK
Need POSIX, many hosts, shared?      → FILE
Anything else, especially blobs?     → OBJECT

If torn between FILE and OBJECT:     OBJECT (almost always)
If torn between BLOCK and FILE:      BLOCK (almost always)
```

The default in modern cloud architecture is object. You use block only where you must (databases, hot caches), file only where legacy forces you (mounts, NFS-based products), and object for everything else.

---

## 14. Cheat Card

```
PURPOSE     Three abstractions of persistent storage; differ in API,
            sharing model, latency, cost.

BLOCK       Raw disk (LBA grid). Single-host. Sub-ms latency.
            Use for DBs, VMs. AWS EBS, GCP PD, Azure Disks.
            $0.08–0.20/GB-month.

FILE        Hierarchical FS over NFS/SMB. Multi-host shared.
            Use for legacy, shared scratch. EFS, Filestore, FSx.
            $0.30/GB-month.

OBJECT      Flat key→blob over HTTP REST. Effectively infinite.
            11×9 durability. Use for assets, backups, lakes.
            S3, GCS, Azure Blob. $0.001–0.023/GB-month.

LATENCY     Block ≪ File < Object  (sub-ms vs ms vs ~50 ms)
COST        Block ≫ File ≫ Object (object is 5–200× cheaper)

PITFALLS    DB on NFS · S3 as filesystem · many tiny objects ·
            monotonic prefixes · no lifecycle policy · snapshot ≠ backup

RULE        Default to object. Reach for block only when you need
            IOPS or a real filesystem. Reach for file only when
            legacy demands a mount.
```

---

## 15. Resources

### Books
- *Designing Data-Intensive Applications* — Martin Kleppmann. Chapter 3 on storage and retrieval grounds the block/file/object distinction in engine internals.
- *Site Reliability Engineering* — Google. The chapter on data integrity touches on the durability targets that drive each tier.

### Documentation
- **AWS S3** — <https://docs.aws.amazon.com/s3/>
- **AWS EBS** — <https://docs.aws.amazon.com/ebs/>
- **AWS EFS** — <https://docs.aws.amazon.com/efs/>
- **GCS** — <https://cloud.google.com/storage/docs>
- **Azure Blob** — <https://learn.microsoft.com/azure/storage/blobs/>

### Articles
- "Diving Deep on S3 Consistency" — AWS, 2020: <https://aws.amazon.com/blogs/aws/amazon-s3-update-strong-read-after-write-consistency/>
- "Building and Operating a Pretty Big Storage System (S3)" — Andy Warfield (re:Invent), 2023.
- "Mountpoint for Amazon S3" — AWS engineering blog on the FUSE-style client.
- Cloudflare R2 launch posts — economics arguments for egress-free object storage.

### Videos
- ByteByteGo — "Block vs File vs Object Storage" overview.
- AWS re:Invent — "Deep Dive on Amazon EBS" and "Deep Dive on Amazon S3" sessions; the architecture talks are excellent.
- CMU 15-721 — Andy Pavlo's lectures on cloud storage primitives.

### Tools
- **MinIO** — self-hosted S3-compatible object storage.
- **Ceph** — unified block + file + object on commodity hardware.
- **rclone** — copies data between every block/file/object backend imaginable.
- **mountpoint-for-s3** — official AWS S3 FUSE driver.
- **s5cmd** — fast S3 CLI, often 20–50× faster than `aws s3 cp` for parallel work.

### Adjacent reading
- [Distributed File Systems (HDFS, GFS) →](./distributed-file-systems.md)
- [Object Storage (S3, GCS, Azure Blob) →](./object-storage.md)
- [Storage Engines (LSM-Trees vs B-Trees) →](./storage-engines.md)
- [Erasure Coding vs Replication →](./erasure-coding.md)
- [CDN — Content Delivery Networks](../05-caching/cdn.md)
- [Data Warehouses & Data Lakes](../04-databases/warehouses-lakes.md)

---

*Up:* [README ↑](../README.md)  |  *Next:* [Distributed File Systems (HDFS, GFS) →](./distributed-file-systems.md)
