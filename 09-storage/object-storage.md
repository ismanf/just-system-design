# Object Storage (S3, GCS, Azure Blob)

> **TL;DR** — Object storage exposes a flat **bucket / key → blob** namespace over an HTTP REST API. It is the **default substrate of the cloud** — eleven nines of durability, effectively infinite capacity, and 5–200× cheaper than block storage per GB. Trade-offs: higher per-request latency (tens of ms), no in-place updates, and request-priced APIs. **Amazon S3** is the prototype; **GCS, Azure Blob, Cloudflare R2, Backblaze B2** are the major alternatives; **MinIO and Ceph RGW** are the dominant self-hosted equivalents. Modern data lakes, backups, ML pipelines, static asset hosting, and most cloud-native architectures put almost everything except hot transactional data into object storage.

---

## 1. The Mental Model

```
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│   bucket: my-company-prod-uploads                                │
│                                                                  │
│      key: users/42/avatar.jpg     →  [binary blob, 24 KB]        │
│      key: invoices/2026/05/INV-1.pdf  →  [binary blob, 380 KB]   │
│      key: backups/db/2026-05-18.sql.gz  →  [binary blob, 12 GB]  │
│                                                                  │
│   Each key:                                                      │
│     - opaque string up to ~1024 bytes                            │
│     - maps to one blob (object)                                  │
│     - object has bytes + metadata (content-type, ETag, custom    │
│       headers, ACL, storage class, version ID, encryption keys)  │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

A bucket is a flat container. Keys are arbitrary strings; the slashes inside `users/42/avatar.jpg` are convention only — there are no directories. You read and write whole objects (or specified byte ranges) over HTTPS. That's it. The whole API is essentially `PUT`, `GET`, `HEAD`, `DELETE`, `LIST`, plus a few specialized verbs (`POST` for browser uploads, `COPY` for server-side copies, multipart for large objects).

This simplicity is the entire point.

---

## 2. Why It Won

When AWS launched S3 in March 2006, it sold one API and four properties:

1. **Durability** — 11×9 (objects effectively never disappear).
2. **Availability** — 99.99% target.
3. **Capacity** — pretend it's infinite. Buckets routinely hold exabytes.
4. **Cost** — at the time, an order of magnitude cheaper than the alternatives.

Twenty years later the same four properties drive every architectural decision involving object storage. If you can move data out of a database or filesystem and into object storage without breaking the access pattern, you almost always do, because object storage is the cheapest, most durable, most scalable layer in the cloud.

The pattern that emerged:
- **Hot transactional data** → database (block-backed).
- **Hot cached data** → Redis / Memcached.
- **Everything else** → object storage.

Lakehouses (Delta, Iceberg, Hudi) extended this to analytics: store **all** structured data as Parquet on object storage, query with engines that pretend it's a table.

---

## 3. The API in Five Verbs

```
PUT    /bucket/key                  upload an object
GET    /bucket/key                  download an object (or byte range)
HEAD   /bucket/key                  metadata only
DELETE /bucket/key                  remove
LIST   /bucket?prefix=foo/          enumerate keys with a prefix
```

Plus, in practice:

```
COPY                copy server-side from one key to another (no client transfer)
POST                browser direct uploads via presigned forms
Multipart upload    INIT → PART × N → COMPLETE; required for >100 MB
Range GET           Range: bytes=1000-2047 — partial download
Conditional ops     If-Match / If-None-Match / If-Modified-Since
```

Every operation is **stateless and idempotent at the object level** (writes replace, deletes are sticky if not versioned). There is no transactional `RENAME` — rename is `COPY + DELETE`, two API calls, two charges, and it's not atomic.

---

## 4. The Major Implementations

| Provider | Service | Notable |
|---|---|---|
| **AWS** | S3 | The original. Defines the API. 11×9 durability target. Strong read-after-write since Dec 2020. |
| **Google** | GCS | Single global namespace; strong consistency since launch. Multi-region buckets across continents. |
| **Azure** | Blob Storage | Three blob types: Block, Append, Page. Hot / Cool / Cold / Archive tiers. |
| **Cloudflare** | R2 | S3-compatible API. **No egress fees** — the killer feature for read-heavy workloads. |
| **Backblaze** | B2 | Cheapest of the majors (~$6/TB-month). S3-compatible. |
| **Wasabi** | — | Flat-rate storage, no egress, no API fees. |
| **MinIO** | — | Self-hosted S3-compatible. Single binary. The de facto OSS choice. |
| **Ceph RGW** | — | Self-hosted; unified block/file/object on Ceph. |
| **DigitalOcean** | Spaces | Simple, S3-compatible, predictable pricing. |
| **OpenStack** | Swift | Older OSS implementation; declining in relevance. |
| **IBM, Oracle** | COS / OCI Object | S3-compatible offerings from non-hyperscalers. |

Most "S3-compatible" services implement enough of the S3 API for common SDKs to work. Edge cases (multipart, versioning, lifecycle, ACL semantics) differ — test before betting on full compatibility.

---

## 5. Storage Classes (Tiering)

Each provider tiers blobs by access frequency vs cost.

### AWS S3
| Class | $/GB-mo | First-byte latency | Min. duration | Use |
|---|---|---|---|---|
| Standard | ~$0.023 | ms | none | Hot, frequent reads |
| Intelligent-Tiering | $0.023 (auto-moves) | ms | none | Mixed / unknown patterns |
| Standard-IA | ~$0.0125 | ms | 30 days | Infrequent, immediate access |
| One Zone-IA | ~$0.01 | ms | 30 days | Re-creatable infrequent data |
| Glacier Instant Retrieval | ~$0.004 | ms | 90 days | Rare reads, instant when needed |
| Glacier Flexible Retrieval | ~$0.0036 | minutes–hours | 90 days | Archive with occasional restore |
| Glacier Deep Archive | ~$0.00099 | 12 hours | 180 days | Compliance archives |

GCS and Azure have equivalent ladders (Nearline / Coldline / Archive; Hot / Cool / Cold / Archive).

### Lifecycle policies

Buckets accept rules like:

```
After 30 days  → Standard-IA
After 90 days  → Glacier Instant Retrieval
After 365 days → Glacier Deep Archive
After 7 years  → Delete
```

Lifecycle rules are how you actually save money in production. A bucket without lifecycle rules pays Standard prices for data that hasn't been read in three years.

### Retrieval cost
Cold tiers charge per-GB on read **and** sometimes a per-object retrieval fee. Restoring 1 PB from Deep Archive is its own line item on the invoice. Budget carefully.

---

## 6. Consistency

Object storage spent 14 years with various flavors of eventual consistency. As of December 2020, **S3, GCS, and Azure Blob all provide strong read-after-write and overwrite consistency**: after a successful `PUT` or `DELETE`, every subsequent `GET` sees the new state.

This sounds obvious, but in pre-2020 S3:
- A `PUT new key` was strongly consistent, but...
- A `PUT overwriting an existing key` was eventually consistent, and...
- A `DELETE` was eventually consistent.

Anyone who built data pipelines pre-2020 has war stories about reading stale Parquet manifests because S3 hadn't propagated the new write. Today this class of bug is dead — assume strong consistency.

What's **still not** strongly consistent:
- **LIST after PUT** is strongly consistent for **the existence** of the key, but pagination through millions of keys takes time and the snapshot may shift across pages.
- **Cross-region replication** is asynchronous. Reads from the secondary region trail the primary.

---

## 7. Multipart Upload

Single PUT is capped at 5 GB (S3). Anything larger — and in practice, anything over ~100 MB — should use multipart upload.

```
1. POST /bucket/key?uploads          → upload ID
2. PUT  /bucket/key?partNumber=1&uploadId=...   (5 MB – 5 GB per part)
3. PUT  /bucket/key?partNumber=2&uploadId=...
   ...
4. PUT  /bucket/key?partNumber=N&uploadId=...
5. POST /bucket/key?uploadId=...     → complete with list of part ETags
```

Properties:
- **Parallel** — upload many parts concurrently. This is how you saturate gigabit links.
- **Resumable** — a failed part is just re-uploaded. The upload doesn't restart from zero.
- **Atomic completion** — the object becomes visible only after the final POST.
- **Server-side stitching** — parts are assembled by the storage, not the client.
- **Costs hidden until you complete** — incomplete multipart uploads accumulate and you pay for the parts until you abort them. Set a lifecycle rule to auto-abort after N days.

The 10,000-part limit and the 5 MB minimum (except last) define the geometry: max single object is ~5 TB on S3.

---

## 8. Presigned URLs

A presigned URL is a regular `GET` or `PUT` URL with an HMAC-SHA256 signature embedded as a query parameter. Anyone with the URL can perform the specified operation until it expires.

```
https://my-bucket.s3.amazonaws.com/uploads/foo.png?
   X-Amz-Algorithm=AWS4-HMAC-SHA256&
   X-Amz-Credential=.../...&
   X-Amz-Date=20260519T120000Z&
   X-Amz-Expires=900&
   X-Amz-SignedHeaders=host&
   X-Amz-Signature=abcd1234...
```

Used everywhere:
- **Direct browser uploads** — backend signs a URL, browser PUTs the file straight to S3, bypassing your server.
- **Time-limited downloads** — share a private file for an hour.
- **Mobile apps** — same, without exposing the storage credentials to the client.

Best practice: keep expirations short (minutes for uploads, hours for downloads), and tie them to specific keys / content-length / content-type to prevent abuse.

---

## 9. Pricing — What Actually Costs Money

S3 has at least seven distinct line items. The big ones:

| Item | Order of magnitude |
|---|---|
| Storage (Standard) | $0.023 / GB-month |
| PUT/COPY/POST/LIST | $0.005 / 1,000 |
| GET | $0.0004 / 1,000 |
| Egress to Internet | $0.05–0.09 / GB |
| Egress to other region | $0.02 / GB |
| Egress to same-region service | $0 |
| Lifecycle transition | $0.01 / 1,000 |
| Cross-region replication | egress + PUT on destination |

Observations:
- **Egress dominates** for read-heavy public workloads — sometimes more than storage. This is why CDNs in front of S3 are universal.
- **Tiny objects are expensive in API calls.** 1 billion 1 KB files = 1 KB storage cost trivial, but the PUT charges to create them are $5,000. Pack into Parquet / tar.
- **LIST is paginated** (1,000 keys per call) and charged at PUT prices. A `LIST` over 10 M keys is $50 and slow.
- **Cross-region replication** charges egress and PUT on the destination. PB-scale replication needs a real budget conversation.

Cloudflare R2 explicitly attacked this model — same storage cost, **zero egress** — which is why R2 has grown fast for read-heavy assets and AI training datasets.

---

## 10. Durability — Where 11×9 Comes From

S3 advertises 99.999999999% annual object durability. The math:

- Objects are split into chunks and **erasure-coded** across at least 3 Availability Zones inside a region. Reed-Solomon variants (S3 publicly cites a "redundantly stored on multiple devices across multiple facilities" model).
- AZs are isolated power, cooling, and network domains. A full AZ failure does not lose data.
- Chunks are continuously scrubbed against checksums; corrupted chunks are reconstructed from parity.
- Regular fixity checks compare object integrity hashes.

The 11×9 figure is statistical: if you store 10 million objects, you'd expect to lose, on average, one object every 10,000 years. (Compare to a single SSD with annual failure rate of ~0.5%.)

What 11×9 does **not** cover:
- Application bugs that delete data (use **versioning + MFA delete**).
- Account compromise (replicate to a second account / region).
- Ransomware-style mass deletion (use **Object Lock** / WORM).
- Misconfigured lifecycle rules (you set the rule that deleted everything).

Backups are still your responsibility; durability protects bytes, not policies.

---

## 11. Security

The default attack vector for cloud breaches over the last decade has been **misconfigured public buckets**. Treat object storage security as a first-class concern.

### Identity and access
- **IAM policies** — coarse access at the principal level.
- **Bucket policies** — JSON attached to the bucket itself, including `aws:SourceIp`, `aws:RequestedRegion`, `aws:PrincipalOrgID` conditions.
- **ACLs** — older, per-object; AWS now discourages these. Disable ACLs at the bucket level.
- **Block Public Access** — flip every flag on, at account and bucket level, unless you specifically need a public bucket.
- **Access Points** — named entry points with their own policies, useful for multi-tenant scenarios.

### Encryption
- **SSE-S3** — server-side, AWS-managed keys. Default and free.
- **SSE-KMS** — server-side, customer-managed KMS keys. Audit trail, key rotation, $0.03/10k operations.
- **SSE-C** — customer-provided keys in each request. Rarely used.
- **Client-side encryption** — application encrypts before upload. Maximum control, your responsibility to manage keys.

For sensitive data, KMS-backed encryption + bucket policy `aws:SecureTransport: true` (deny non-TLS) + IAM scoped to specific principals is the baseline.

### Object Lock (WORM)
S3 Object Lock prevents deletion or overwrite for a retention period or under a legal hold. Required for SEC, HIPAA, GDPR-style retention guarantees. Once locked, even the account root can't delete the object until the retention expires.

### Audit
- **CloudTrail data events** — log every API call against the bucket. Expensive at high scale; sample wisely.
- **S3 Access Logs** — server-side access logs delivered to another bucket.

---

## 12. Performance and Scaling

S3 internally partitions a bucket by **key prefix**. As traffic grows, the partition tree splits. This has two important consequences:

### Per-prefix request limits
S3 supports **3,500 PUT/POST/DELETE and 5,500 GET/HEAD per second per prefix.** If your write pattern hammers one prefix (`logs/2026-05-19/...`), you'll hit `503 SlowDown` errors.

The fix: **prefix randomization.** Instead of:
```
logs/2026-05-19T12:00:00.123Z-...
logs/2026-05-19T12:00:00.124Z-...
```

write:
```
logs/8f3a/2026-05-19T12:00:00.123Z-...
logs/2b07/2026-05-19T12:00:00.124Z-...
```
where the leading 4 hex chars are a hash of the key. This spreads writes across thousands of partitions.

Since 2018 S3 auto-splits partitions, so monotonic prefixes work better than they used to — but they still throttle initially. Hash-prefixing is still the safe default for high-write workloads.

### Throughput shape
- Per-request throughput peaks around **100 MB/s**.
- Aggregate throughput is effectively unlimited — scale by parallelism (many concurrent requests across many connections).
- Range GETs let you read a single large object in parallel; tools like `s5cmd` and `mountpoint-for-s3` do this transparently.

### S3 Express One Zone
Single-AZ, single-digit-millisecond latency tier launched in 2023, aimed at ML and analytics workloads that need lower latency than Standard. Costs more per GB, less per request. Useful when your compute and storage are co-located.

---

## 13. Lifecycle, Versioning, and Replication

### Versioning
Enable on a bucket → every PUT creates a new version, every DELETE creates a delete marker but keeps the prior version. Survives accidental deletion and overwrite. **Lifecycle policy is required** to clean up old versions or they accumulate forever (and pay forever).

### Lifecycle rules
JSON rules on prefixes / tags. Examples:
- "Move to Glacier after 90 days."
- "Delete incomplete multipart uploads after 7 days."
- "Expire current versions after 7 years."
- "Delete noncurrent versions after 30 days."

Always set incomplete-multipart cleanup. It's the most-forgotten rule and quietly costs money.

### Cross-region replication (CRR)
Asynchronous replication of new objects from a source bucket to a destination in another region. Used for:
- Disaster recovery
- Compliance (data residency in a second jurisdiction)
- Low-latency reads in a second region

CRR replicates **new objects only** by default; existing objects need a one-time batch copy. Replication time SLA can be configured (S3 Replication Time Control: 15-minute target).

### Same-region replication (SRR)
For account-level isolation (replicate to a backup account in the same region for ransomware protection) or compliance separation.

---

## 14. Object Storage as an Application Substrate

Beyond "store some files," object storage backs entire classes of architecture:

### Data lakes / lakehouses
Parquet/ORC files in S3, queried by Athena / BigQuery / Trino / Spark / Snowflake. Table formats like **Iceberg, Delta Lake, Apache Hudi** layer transactions and time travel on top of object storage. See [Lakehouse Architecture](../04-databases/lakehouse.md).

### Static site hosting
HTML/JS/CSS in a bucket + CDN. Cheap, infinitely scalable, no server to run.

### Backups
Postgres `pgbackrest` ships base backups + WAL archives to S3. Same for MySQL, MongoDB, etc. Snapshots of EBS volumes are themselves stored in S3 internally.

### ML training data
Datasets in object storage; training jobs stream batches via parallel range reads or via FUSE mounts. Mountpoint-for-S3 was built specifically for this.

### Log archives
Hot logs in Elasticsearch or Loki; older logs flushed to S3 (Glacier IR) for compliance retention.

### Event ingestion
Kinesis Firehose, GCP Dataflow, etc., land raw events in object storage as the durable record before downstream processing.

### Distributed databases use object storage
Modern systems (Snowflake, Databricks, Neon for Postgres, Tigris, Turbopuffer) push their primary storage to object storage and keep only a thin compute / cache layer. The trade — higher base latency, near-zero storage cost — has reshaped database architecture in the last five years.

---

## 15. Operational Reality

### Eventually-consistent listing
Even with strong read-after-write, `LIST` can briefly trail. A pattern that writes a manifest, then writes data files, then lists by prefix can race. Idiom: name files deterministically and avoid relying on `LIST` for correctness in tight loops.

### Throttling under bursty load
The first time you launch a job that does 100k PUT/s against a fresh bucket, you'll see `503 SlowDown` while S3 splits partitions. Production-readiness checklist: pre-warm by issuing a gradual ramp.

### Mountpoint, fuse, and the lies they tell
`s3fs`, `goofys`, `mountpoint-for-s3` make a bucket look like a filesystem. Read-mostly: fine. Random writes: undefined behavior — fuse layers map writes to PUT-overwrite-the-whole-object. Treat as read-only or write-once.

### The 5 GB single-PUT and 5 TB max object
Anything over 5 GB needs multipart. Max object size is 5 TB on S3, 5 TB on GCS, 200 GB on Azure block blobs (with single PUT) up to 4.77 TB via Put Block.

### Inventory reports
S3 Inventory delivers a daily CSV/Parquet listing of every object in a bucket. Use this instead of `LIST` for analytics over very large buckets — it's faster and cheaper.

### Cross-account access
Bucket policies + IAM roles. The pattern: source account grants read/write to a role in the destination account, destination account assumes the role. Avoid the historical mess of cross-account ACLs.

---

## 16. Self-Hosted Object Storage

When cloud isn't an option (compliance, latency, cost at extreme scale), the open-source choices:

### MinIO
- Single Go binary, drop-in S3 API.
- Erasure-coded across drives in a deployment.
- Operator-friendly; dominant in Kubernetes-native deployments.
- Federation across multiple deployments for global namespaces.

### Ceph RGW (RADOS Gateway)
- Object API in front of the Ceph RADOS storage layer.
- Same cluster can also serve block (RBD) and file (CephFS).
- Operationally complex but very flexible.

### Garage
- Distributed object storage from the Deuxfleurs collective.
- Geo-distributed by design, focused on small clusters with replication.

### SeaweedFS
- Object + file storage with a tiered architecture.
- Smaller community but interesting design.

For a small cluster (TBs, not PBs) MinIO is the obvious starting point. Ceph is where you go when you also need block and file from the same hardware.

---

## 17. Common Mistakes / Anti-Patterns

- **Public bucket misconfiguration.** Default to "Block all public access." Every major data breach involving S3 has been this. Audit regularly.
- **Treating S3 like a filesystem.** No directories, no atomic rename, no concurrent append. Apps that depend on these will misbehave.
- **Many tiny objects.** A million 1 KB objects is a hostility to your wallet and to performance. Pack into Parquet, tar, or larger objects.
- **Monotonic prefixes at high write rate.** Sharded prefixes solve this; you'll feel it at first contact otherwise.
- **No lifecycle policy.** Storage grows forever; old versions and aborted multipart uploads cost money silently.
- **Forgetting incomplete multipart cleanup.** Common cause of unexplained S3 cost growth.
- **Cross-region egress on hot reads.** Use CRR if you need multi-region reads; otherwise serve via CDN.
- **No versioning on critical buckets.** Ransomware and bugs delete things. Versioning + Object Lock = recoverable. Without them, you have one chance.
- **Logs into S3 with one tiny object per event.** Buffer locally; batch into MB-scale files. This is the single biggest cost mistake in observability pipelines.
- **Using S3 as a queue.** It's tempting — write a file, have a worker poll the bucket — but you'll race, you'll pay for LIST, and you'll be sad. Use SQS / Kinesis / Kafka.
- **Bucket per tenant in B2B SaaS.** AWS limits buckets to 1,000 per account (raisable but not unbounded). Use prefixes / Access Points instead.
- **Trusting "S3-compatible" without testing.** Multipart, versioning, presigned URL semantics, header capitalization — surprise differences abound.

---

## 18. Decision Rule

```
Need POSIX semantics?              → not object storage
Need <10 ms first-byte latency?    → block / cache, not standard object
Need atomic in-place updates?      → not object storage
Everything else?                   → object storage

Cold? Tier to Glacier / Archive.
Multi-region reads? Replicate or CDN.
Public assets? CDN in front of bucket, always.
```

Object storage is the default. The job is to know when to step away from it — usually for latency, sometimes for POSIX, occasionally for cost at extreme egress.

---

## 19. Cheat Card

```
PURPOSE     Flat bucket/key→blob storage over HTTP REST.
            Cheapest, most durable, infinitely scalable cloud
            storage substrate.

API         PUT · GET · HEAD · DELETE · LIST · COPY · multipart
            No directories, no rename, no append-in-place.

PROPERTIES  11×9 durability  ·  99.99% availability target
            Strong read-after-write (since Dec 2020 on S3)
            ~50 ms first-byte; aggregate throughput unbounded
            Per-prefix limits: 3.5k writes/s, 5.5k reads/s

ECONOMICS   $0.023/GB-month hot
            $0.001/GB-month archive (Deep Archive)
            Egress is the silent killer
            Lifecycle rules are how you actually save money

PATTERNS    Multipart for >100 MB
            Presigned URLs for direct browser/mobile uploads
            CDN in front for public reads
            Prefix-hash high-write workloads
            Versioning + Object Lock for compliance/ransomware

PITFALLS    Public-bucket misconfig · tiny files · no lifecycle ·
            no multipart cleanup · monotonic prefix throttling ·
            cross-region egress · S3 as a filesystem · S3 as a queue

USE FOR     Blobs · backups · archives · data lakes · ML datasets ·
            static assets · log archives · event ingestion

AVOID FOR   Hot OLTP · low-latency caches · in-place updates ·
            POSIX-locking workloads

RULE        Default to object storage. Step away only for latency,
            POSIX, or in-place updates.
```

---

## 20. Resources

### Books
- *Designing Data-Intensive Applications* — Martin Kleppmann. Chapter 10 covers batch processing on object-storage-like substrates.
- *AWS Well-Architected Framework* — the Storage pillar's S3 sections.

### Documentation
- **AWS S3** — <https://docs.aws.amazon.com/s3/>
- **GCS** — <https://cloud.google.com/storage/docs>
- **Azure Blob Storage** — <https://learn.microsoft.com/azure/storage/blobs/>
- **Cloudflare R2** — <https://developers.cloudflare.com/r2/>
- **MinIO** — <https://min.io/docs/minio/>

### Articles
- "Diving Deep on S3 Consistency" — AWS, Dec 2020: <https://aws.amazon.com/blogs/aws/amazon-s3-update-strong-read-after-write-consistency/>
- "Building and Operating a Pretty Big Storage System (S3)" — Andy Warfield, AWS re:Invent 2023 talk.
- "S3 Express One Zone" — AWS, 2023.
- Cloudflare R2 launch and "egress is over" posts.
- Tigris and Turbopuffer engineering blogs — modern S3-as-substrate database architecture.
- "Mountpoint for Amazon S3" — AWS blog explaining the FUSE driver.

### Videos
- ByteByteGo — "S3 Architecture" overview.
- AWS re:Invent — "STG303 — Deep Dive on S3" sessions every year; consistently the best architecture talks.
- AWS re:Invent — "Building and Operating a Pretty Big Storage System" (Andy Warfield, 2023).

### Tools
- **s5cmd** — fast parallel S3 CLI, often 20–50× faster than `aws s3 cp`.
- **rclone** — universal storage migration tool.
- **mountpoint-for-s3** — official AWS S3 FUSE driver.
- **MinIO Client (`mc`)** — works against any S3-compatible store.
- **AWS CLI v2** — `aws s3 sync` and `aws s3api` for the lower-level operations.

### Adjacent reading
- [Block, File, and Object Storage](./storage-types.md)
- [Distributed File Systems (HDFS, GFS)](./distributed-file-systems.md)
- [Erasure Coding vs Replication →](./erasure-coding.md)
- [CDN — Content Delivery Networks](../05-caching/cdn.md)
- [Data Warehouses & Data Lakes](../04-databases/warehouses-lakes.md)
- [Lakehouse Architecture](../04-databases/lakehouse.md)
- [Design Google Drive / Dropbox](../18-case-studies/dropbox.md)
- [Encryption at Rest & In Transit](../12-security/encryption.md)

---

*Previous:* [← Distributed File Systems (HDFS, GFS)](./distributed-file-systems.md)  |  *Next:* [Storage Engines (LSM-Trees vs B-Trees) →](./storage-engines.md)
