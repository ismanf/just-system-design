# Database Sharding Strategies (Range, Hash, Geo, Directory)

> **TL;DR** — Sharding splits writes across N independent shards so each shard owns only 1/N of the data and traffic. The choice of **shard key** and **shard strategy** is the most consequential design decision in any sharded system — you live with it for years. The four major families are **range** (sorted contiguous slices; great for scans, prone to hot tail), **hash** (hash the key, modulo or consistent-hash to a shard; even distribution, no range scans), **geo** (shard by region; latency-aware, regulation-friendly, but cross-region traffic is the trap), and **directory / lookup** (separate lookup table maps keys to shards; flexible at runtime, single point of complexity). Real systems usually combine strategies — hash on tenant ID, then geo-pin per region, with a directory for migrations. Get the shard key right and you scale; get it wrong and every fix costs a quarter.

---

## 1. The Setup

This page sits between two close cousins. For the broader treatment of why sharding exists and what it costs, see [Sharding & Partitioning →](../04-databases/sharding-partitioning.md). This page focuses on the **four shard-strategy families** and how to choose one for a real workload.

```
                  ┌─────────────────────────┐
                  │      Router / Proxy     │
                  └────────────┬────────────┘
                               │
        ┌──────────┬───────────┼───────────┬──────────┐
        ▼          ▼           ▼           ▼          ▼
     Shard 0    Shard 1     Shard 2     Shard 3    Shard 4
     ──────     ──────      ──────      ──────     ──────
     primary    primary     primary     primary    primary
       + replicas  + replicas  ...
```

The router knows: *given a request, which shard owns the data?* The answer to "knows" is the shard strategy.

---

## 2. The Four Strategies

```
RANGE       contiguous slices of a sorted key

HASH        hash(key) → shard

GEO         shard by region/locality

DIRECTORY   lookup table maps key → shard
```

Each has a characteristic mathematical shape, a characteristic failure mode, and a characteristic class of system that loves it.

---

## 3. Range Sharding

Each shard owns a **contiguous range** of the shard key.

```
shard 0: user_id [        0 –  10,000,000)
shard 1: user_id [10,000,000 –  20,000,000)
shard 2: user_id [20,000,000 –  30,000,000)
shard 3: user_id [30,000,000 –  40,000,000)
```

Or with strings:

```
shard 0: org_name [a* – f*)
shard 1: org_name [f* – m*)
shard 2: org_name [m* – t*)
shard 3: org_name [t* – z*)
```

### How it routes
Router stores the range boundaries. Lookup is a binary search over ranges (or a direct map on coarse buckets).

### Strengths
- **Range scans are local.** `WHERE id BETWEEN 5_000_000 AND 6_000_000` hits one shard.
- **Natural sort order** preserved. Pagination, "next 100 records," and ordered iteration are fast.
- **Easy to understand.** The mapping is intuitive — useful for ops and debugging.
- **Cheap operational tooling** — easy to back up, copy, and move a shard.

### Weaknesses
- **Hot tail problem.** If new IDs are monotonic (auto-increment, timestamps), every new write hits the last shard. The other N-1 shards sit idle for writes. See [Hot Partitions →](./hot-partitions.md).
- **Splits and rebalancing** are awkward — splitting a range requires moving half the data to a new shard.
- **Skewed key distributions** create permanently unbalanced shards. If `org_name` starts with "A" for 30% of customers, shard 0 is in trouble.

### Used by
- **HBase, Bigtable, Spanner** — range-partitioned by row key; tablet/region split when too big.
- **CockroachDB, YugabyteDB** — range-partitioned KV with auto-splitting.
- **TiKV / TiDB** — range-partitioned, Raft per range.
- **MongoDB** — supports range and hash sharding; range is the default unless you opt in.

### When to choose
- Time-series and log-style data — but combine with TTL/expiry to discard old shards (and choose a non-monotonic component too).
- Workloads with frequent **range scans** or **ordered pagination**.
- When you have a small number of large keys and the system supports auto-splitting.
- Never with naked monotonic keys. Use time-bucketed or hash-prefixed keys at minimum.

---

## 4. Hash Sharding

Hash the shard key; route to a shard via `shard = hash(key) % N` or via **consistent hashing**.

```
shard = hash(user_id) % 16

   key:    user_id = 4242
   hash:   0x7a9b1c0e8f...
   mod 16: 14  → shard 14
```

### How it routes
Cheap math at the application or router. Stateless if N is fixed.

### Strengths
- **Uniform distribution** by default. No hot tail.
- **Predictable shard load** if hash is good and keys are diverse.
- **No coordination** required on routing — anyone can compute the shard.
- **Simple addition of shards** with consistent hashing (only ~1/N of keys move).

### Weaknesses
- **Range scans cross all shards.** `WHERE id BETWEEN X AND Y` is now a scatter-gather.
- **Cross-shard joins** are expensive — most application keys distribute differently.
- **Modulo schemes are brittle**: changing N from 8 to 16 reshuffles ~50% of all keys. Use consistent hashing.
- **Hot keys** still happen — one celebrity user's ID can still saturate one shard regardless of how good the hash is.
- **Order is lost.** No "next 100 ordered" without secondary indexes.

### Consistent hashing
With naive `hash % N`, adding a shard is catastrophic — keys jump everywhere. **Consistent hashing** maps both keys and shards onto a ring; adding a shard only moves keys between adjacent positions, typically ~1/N of total keys. See [Consistent Hashing →](../04-databases/consistent-hashing.md).

Variants:
- **Virtual nodes** — each physical shard owns many ring positions to even out load.
- **Jump hash** (Google, 2014) — simpler than rings, perfect for known shard counts.
- **Rendezvous hashing** (HRW) — every node computes a score for the key; the highest wins.

### Used by
- **Cassandra, ScyllaDB** — hash partitioner is the default. Murmur3 over the partition key.
- **DynamoDB** — partition key hashed across many internal partitions.
- **Redis Cluster** — CRC16 mod 16384, slots assigned to nodes.
- **Vitess hash vindex** — sharding MySQL on hashed keys.
- **Memcached / sharded Redis clients** — consistent hashing on the client.

### When to choose
- Workloads dominated by **point lookups** by primary key (logins, profiles, sessions).
- High-cardinality keys with no hot celebrity (or with explicit handling for hot keys).
- Write-heavy workloads where you need uniform write distribution.
- Almost always combined with a secondary index or a separate read store for non-key access.

---

## 5. Geo Sharding

Shard by user location, data residency, or regional account.

```
US-EAST     → shard "us-east"
US-WEST     → shard "us-west"
EU-WEST     → shard "eu-west"
AP-SOUTH    → shard "ap-south"
```

### How it routes
The application knows the user's primary region (from sign-up, profile, or IP geo). Reads/writes for that user route to that region's shard.

### Strengths
- **Low-latency reads/writes** for users near their data.
- **Data residency compliance** (GDPR, China data law, etc.) — EU users' data physically in the EU.
- **Blast radius isolation** — a regional outage affects only that region's users.
- **Regulatory simplification** — auditors see clear geographic separation.

### Weaknesses
- **Cross-region operations** are painful — a US user collaborating with an EU user means one of them is far.
- **User mobility** breaks the model — what if a user moves? Most systems lock the region at signup or migrate manually.
- **Global queries** (admin dashboards, analytics) are scatter-gather across regions.
- **Replication for DR** within a region is mandatory; cross-region replication adds cost and latency.
- **Cross-region transactions** are essentially impossible without Spanner-class infrastructure.

### Used by
- **B2B SaaS** with EU + US tenants (Salesforce, Shopify, GitHub Enterprise Cloud).
- **Consumer products** with strong geographic affinity (food delivery, ride-sharing).
- **Regulated industries** — banking, healthcare with cross-border data restrictions.
- **CDN-backed platforms** that route users to nearest edge.

### When to choose
- Users are geographically clustered and rarely cross regions.
- Latency budgets demand <50 ms RTT for primary operations.
- Compliance/regulation drives a hard split.
- Combined with hash sharding inside each region for further scale.

### Hybrid pattern (most common)
```
Top-level: geo sharding   → user's region
Within region: hash sharding by tenant or user ID
                          → many shards inside one region
```

This is roughly the Shopify, Stripe, Slack model.

---

## 6. Directory (Lookup) Sharding

A separate **lookup service** maps each key (or each chunk of keys) to its current shard.

```
lookup:
  user_id=42   → shard 7
  user_id=88   → shard 3
  user_id=109  → shard 12
  ...

Or coarser:
  tenant_id=acme    → shard cluster A
  tenant_id=globex  → shard cluster B
  tenant_id=initech → shard cluster C
```

### How it routes
1. App receives a request with a key.
2. App asks the directory: "where does this key live?"
3. App routes the request to that shard.
4. Directory result heavily cached (per-app, per-LB, per-proxy).

### Strengths
- **Maximum flexibility** — any key can live on any shard.
- **Live migrations are easy** — update the directory; data follows.
- **Per-tenant pinning** — large customers get dedicated shards; small ones share.
- **Heterogeneous shards** — bigger machines for whales, smaller for minnows.
- **Granular rebalancing** — move specific tenants/users, not whole ranges.

### Weaknesses
- **New dependency** — the directory is on every request path.
- **SPOF risk** if directory isn't HA.
- **Cache invalidation** during moves — stale routing sends traffic to old shard.
- **Lookups cost** — extra hop, extra latency. Aggressive caching required.
- **Two-phase commit during moves** — directory and data must agree, or you get split-brain.

### Used by
- **Slack** — uses Vitess with per-team sharding driven by directory-like metadata.
- **GitHub** — Spokes / DGit; per-repository routing.
- **Many B2B SaaS** with per-tenant data isolation, especially when tenants vary in size by 1000×.
- **Most file/object systems** internally — buckets/files mapped to physical storage groups by metadata service.
- **Bigtable's METADATA tablet** — directory mapping ranges to tablet servers (so even range-sharded systems often have a directory layer).

### When to choose
- **Multi-tenant SaaS** with widely varying tenant sizes.
- Workloads where you'll need to move data around frequently.
- Hot-key isolation — give one celebrity user their own shard.
- When you've outgrown hash sharding but can't tolerate cross-shard joins for some access patterns.

---

## 7. Side-by-Side Comparison

| Property | Range | Hash | Geo | Directory |
|---|---|---|---|---|
| Distribution | by key order | uniform by hash | by region | arbitrary |
| Range scans | local to shard | scatter-gather | scatter | depends |
| Point lookups | cheap | cheap | cheap | cheap (with cache) |
| Cross-shard joins | sometimes possible | hard | hard | hard |
| Hot-spot risk | high (monotonic keys) | low (per-key collisions still possible) | moderate (popular regions) | mitigatable per-key |
| Rebalancing | split range, move data | consistent hashing helps | regional shifts rare | easy (update directory) |
| Operational simplicity | high | high | medium | medium-low |
| Latency for non-local users | n/a | n/a | high (cross-region) | depends on placement |
| Compliance / residency | n/a | n/a | strong | per-mapping |
| Live migration | hard | medium (CH) | hard | easy |
| Best for | time-series, ordered | OLTP, KV stores | multi-region SaaS | multi-tenant SaaS |
| Used by | HBase, Bigtable, Spanner, Cockroach, TiKV | Cassandra, Dynamo, Redis Cluster, Vitess | Stripe, Shopify, GitHub, banking, healthcare | Slack, GitHub, Vitess (per-keyspace), Citus |

---

## 8. Choosing the Shard Key

The choice of shard strategy matters less than the choice of **shard key**. Wrong shard key → hot partitions, cross-shard transactions, expensive migrations.

### Properties of a good shard key
1. **High cardinality** — billions of values, not 50.
2. **Even distribution** — no value dominates traffic.
3. **Co-locates related data** — common queries hit one shard.
4. **Immutable** — changing the key is changing the shard.
5. **Present on every query** that needs to be routed.

### Common shard-key choices

| Key | Pros | Cons |
|---|---|---|
| `user_id` | High cardinality, even distribution | Cross-user queries scatter |
| `tenant_id` / `org_id` | Co-locates one tenant's data; ACID per tenant | Tenant size skew (whales vs minnows) |
| `(region, tenant_id)` | Compliance + scale | Cross-region scattering for global users |
| `time` / `timestamp` | Excellent for time-series scans | Monotonic — hot tail |
| `(time_bucket, hash(key))` | Time scans + even writes | Compound, harder to reason about |
| `hash(user_id)` | Even by construction | No range scans |
| `device_id` | Per-device locality | If devices belong to users, cross-shard queries |

### Anti-patterns
- **Monotonic shard keys** (`created_at`, auto-increment, current timestamp). Hot tail.
- **Low-cardinality keys** (`status`, `country`, `category`). Whole tenant on one shard.
- **Keys with skewed popularity** (`celebrity_user_id`). Hot key.
- **Composite keys you can't construct at query time.** If the API doesn't know the shard key, you scatter.

### The compound key trick
For time-series with hot tail, use `(hash_prefix, time)`:

```
key = "ab/2026-05-19T12:00:00Z/user_42"
```

The leading hash prefix spreads writes across N partitions; time ordering is preserved within each prefix. Used by HBase / Bigtable for high-volume time-series.

---

## 9. Worked Example — Shopify Multi-Tenant Sharding

(Simplified; the actual setup is more complex.)

```
Top-level:    Geo sharding by region
              - US merchants → US "pod"
              - EU merchants → EU pod
              - APAC → APAC pod
              - reasons: latency, GDPR, blast radius

Inside each pod:  Directory (per-shop) sharding
              - lookup: shop_id → vitess keyspace
              - small shops share a keyspace
              - large shops (BFCM-volume) get isolated keyspaces

Inside each keyspace: Hash sharding by shop_id
              - Vitess scatters across MySQL shards
              - intra-shop transactions stay on one shard
```

Result: a small Shopify merchant lives on a shared shard with hundreds of similar shops; a giant merchant lives on a dedicated shard cluster; every shop is in the right region. Adding a region adds capacity; isolating a noisy neighbor is one directory update.

This is the canonical multi-tenant SaaS architecture.

---

## 10. Worked Example — Cassandra Time-Series

```
Schema:
  CREATE TABLE sensor_readings (
    sensor_id text,
    bucket text,        -- "2026-05-19/12"  (hour bucket)
    ts timestamp,
    value double,
    PRIMARY KEY ((sensor_id, bucket), ts)
  );
```

The partition key is `(sensor_id, bucket)`:
- `sensor_id` spreads writes across many sensors.
- `bucket` rolls over each hour so a single hot sensor doesn't accumulate a single ever-growing partition.
- `ts` within the partition keeps rows sorted for range queries inside the hour.

This is the canonical Cassandra anti-hot-tail recipe: **bucketed partition keys + hash distribution**.

---

## 11. Migrating Between Strategies

The hardest sharding decision isn't "which strategy" — it's "how to migrate when you got it wrong."

### Common migrations
- **Single primary → hash-sharded.** Pinterest, Notion, Instagram histories.
- **Hash → directory.** Slack moved to per-team metadata as the tenant skew grew.
- **Range → hash.** When the tail keeps lighting on fire.
- **Single-region → geo.** When you sell to Europe and the lawyers call.

### Migration mechanics
- **Dual-write** new data to both old and new shard, validate, then switch reads.
- **Backfill** historical data via batch jobs over weeks.
- **Read-through proxy** that splits traffic by key gradually.
- **Vitess online schema change** for cluster-internal moves.
- **Application-level routing** that flips per-key when ready.

This is multi-quarter work. Plan for it the first time. The classic [Stripe online migrations](https://stripe.com/blog/online-migrations) write-up is essential reading.

---

## 12. Operational Reality

### Resharding pain
Range-sharded systems with auto-split (Spanner, Cockroach, TiKV) handle resharding transparently. Most others require explicit migrations:
- Cassandra: vnodes help, but bootstrap of a new node still takes hours/days.
- DynamoDB: opaque internal partitioning; you control read/write capacity per partition key.
- MySQL/Postgres: Vitess / Citus / pg_shard handle it but require careful planning.

### Cross-shard transactions
Almost always avoid. If unavoidable:
- **Two-phase commit (2PC)** — distributed transactions with all the locking. See [2PC →](../08-distributed-systems/2pc-3pc.md).
- **Saga pattern** — compensating actions instead of distributed locks. See [Saga →](../07-messaging/saga-pattern.md).
- **Outbox pattern** — atomic publish from one shard, eventual consumption. See [Outbox →](../07-messaging/outbox-pattern.md).
- **External coordinator** — a separate "transaction" service that orchestrates.

### Cross-shard queries
- Pre-aggregate via CDC into a query store (Elasticsearch, Snowflake).
- Use a query proxy (Vitess `vtgate`, Citus coordinator) that fan-outs and gathers.
- Denormalize so the join no longer crosses shards.
- Accept that the global "SELECT COUNT(*) FROM users" is now a scatter-gather, and only run it via analytics.

### Hot key / hot partition
Even with hash sharding, a single key can saturate one shard. Mitigations:
- **Split the key** in application (`celebrity_user_id_${random(8)}`), then merge on read.
- **Tiered cache** in front for read-heavy hot keys.
- **Dedicated shard** for the celebrity via directory.

See [Hot Partition Problem →](./hot-partitions.md).

### Backups
Per-shard backups are independent — N times the operational surface. Tooling:
- Coordinated snapshot across shards at the same logical time.
- Or accept slight skew across shards (most teams do, for cost).

### Schema migrations
A migration that takes 5 minutes on one DB takes 5 minutes × N shards (often in parallel, sometimes not). Tools like **gh-ost**, **pt-online-schema-change**, **Postgres logical replication** make this survivable. See [Migrations at Scale →](../04-databases/migrations.md).

---

## 13. Common Mistakes / Anti-Patterns

- **Monotonic shard key** (timestamp, auto-increment). Permanent hot tail.
- **Low-cardinality shard key.** `country` as the key → 5 shards for 200 countries, 80% on US.
- **Sharding by mutable field.** User changes email → entire row must move. Don't.
- **Forgetting the shard key on every query.** Now you scatter-gather instead of routing.
- **Using `% N` for shard count.** Adding a shard reshuffles everything. Use consistent hashing.
- **One shard per tenant in a tiny-tenant SaaS.** 10k tenants → 10k databases → operational nightmare.
- **No directory caching.** Every request waits on directory service → directory becomes the bottleneck.
- **Designing without a migration plan.** The first re-shard is a quarter-long project; pretending you'll never need it is hopeful.
- **Sharding too early.** Vertical scale + functional partitioning solves most loads <10k writes/sec.
- **Sharding too late.** Now there's no obvious shard key and the migration is impossible without downtime.
- **Ignoring whale tenants in multi-tenant systems.** One customer is 100× bigger than the rest; they're going to break a shared shard.
- **Cross-shard transactions.** They exist; they shouldn't be your default for routine operations.
- **No plan for hot keys.** Celebrity users, viral events, BFCM merchants. They will exist.

---

## 14. Decision Tree

```
Do you need range scans / ordered iteration?
   YES → range sharding (but compose with hash/bucket to avoid hot tail)
   NO  → continue

Do you have multi-region users / compliance requirements?
   YES → geo (top-level) + hash or directory (within region)
   NO  → continue

Is the workload multi-tenant with huge size variation?
   YES → directory sharding (per-tenant pinning)
   NO  → continue

Are point-lookups by a high-cardinality key dominant?
   YES → hash sharding (consistent hashing for elasticity)

Watch for:
  Hot keys → directory split / app-level fanout
  Cross-shard queries → CDC to analytical store
  Cross-shard txns → sagas, outbox, or 2PC if you must
  Whale tenants → directory + dedicated shard
```

---

## 15. Cheat Card

```
PURPOSE     Split the write surface across N shards so each shard
            owns 1/N of the data and traffic.

STRATEGIES
  RANGE     contiguous slices of sorted key
            ↑ scans, ordering   ↓ hot tail with monotonic keys
            HBase, Bigtable, Spanner, Cockroach, TiKV, MongoDB

  HASH      hash(key) → shard
            ↑ uniform distribution   ↓ no range scans, hot keys
            Cassandra, DynamoDB, Redis Cluster, Vitess hash vindex

  GEO       shard by region
            ↑ latency, compliance   ↓ cross-region ops painful
            multi-region B2B SaaS, banking, healthcare

  DIRECTORY lookup table maps key → shard
            ↑ flexibility, hot-key isolation, live moves
            ↓ extra dependency, caching required
            Slack, GitHub, Vitess per-keyspace, Citus

HYBRID      Most real systems compose: geo at top, then hash or
            directory inside each region.

SHARD KEY
  Properties: high cardinality · even distribution · co-locates
              related data · immutable · present on every query
  Anti-patterns: monotonic · low cardinality · mutable · skewed

PITFALLS    Wrong shard key · hot tail · cross-shard txns · ignoring
            whale tenants · % N instead of consistent hashing ·
            sharding too early/late · no migration plan

RULE        The shard key is the architectural decision you live
            with for years. Design it like a foreign-key constraint
            on the future.
```

---

## 16. Resources

### Books
- *Designing Data-Intensive Applications* — Martin Kleppmann. Chapter 6 on partitioning is the canonical introduction.
- *Database Internals* — Alex Petrov.
- *Patterns of Distributed Systems* — Unmesh Joshi. Sharding chapters.

### Articles
- "Online Migrations at Stripe" — Brandur Leach: <https://stripe.com/blog/online-migrations>
- "Scaling Datastores at Slack with Vitess" — Slack engineering.
- "Sharding Pinterest" — early write-up, still a great reference.
- "DBA's Guide to Citus" — sharding Postgres at Citus.
- "How Discord Stores Trillions of Messages" — wide-column hash sharding.
- "Vitess: Scaling MySQL" — YouTube engineering history.
- "Spanner: TrueTime and External Consistency" — Google paper, range-sharded.
- "Cassandra Partition Key Patterns" — DataStax.

### Videos
- ByteByteGo — "Database Sharding Explained."
- Hussein Nasser — sharding series.
- CMU 15-721 — Andy Pavlo's distributed databases lectures.
- KubeCon / Velocity talks on Vitess and Citus.

### Tools
- **Vitess** — Production MySQL sharding (YouTube → Slack → Shopify, GitHub).
- **Citus** — Sharded Postgres.
- **CockroachDB / TiDB / YugabyteDB** — Auto-sharded distributed SQL.
- **MongoDB Sharded Cluster** — Built-in range and hash sharding.
- **Cassandra / ScyllaDB** — Hash-sharded wide-column.

### Adjacent reading
- [Sharding & Partitioning](../04-databases/sharding-partitioning.md)
- [Consistent Hashing](../04-databases/consistent-hashing.md)
- [Hot Partition Problem →](./hot-partitions.md)
- [Capacity Planning →](./capacity-planning.md)
- [Multi-Region →](./multi-region.md)
- [Saga Pattern](../07-messaging/saga-pattern.md)
- [Two-Phase Commit (2PC)](../08-distributed-systems/2pc-3pc.md)
- [Database Migrations at Scale](../04-databases/migrations.md)

---

*Previous:* [← Scaling Reads vs Scaling Writes](./reads-vs-writes.md)  |  *Next:* [Hot Partition Problem →](./hot-partitions.md)
