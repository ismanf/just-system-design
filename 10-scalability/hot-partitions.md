# Hot Partition Problem

> **TL;DR** — A **hot partition** is a single shard, partition, key, or replica taking dramatically more traffic than its peers — usually 10× to 1000× — because the workload concentrates on one slice of the keyspace. The total cluster has plenty of capacity, but one node is on fire while the others idle. Causes range from **monotonic shard keys** (everything writes to the tail) to **celebrity keys** (viral video, big-tenant ID, default category) to **broken hash distributions** to **scheduled jobs** that hammer one partition. Fixes are workload-specific: bucket the key, salt the prefix, split the celebrity into multiple sub-keys, add a read cache, isolate the hot tenant onto a dedicated shard, or change the access pattern entirely. Every large sharded system has stories about hot partitions taking it down. Knowing the pattern is half the battle.

---

## 1. The Symptom

```
Cluster of 16 shards. Aggregate capacity 10× current load.
   Yet:

   shard 0   ▓░░░░░░░░░░░░░░░░░░░░░░░░░░░   2% CPU
   shard 1   ▓░░░░░░░░░░░░░░░░░░░░░░░░░░░   2%
   shard 2   ▓▓░░░░░░░░░░░░░░░░░░░░░░░░░░   3%
   shard 3   ▓░░░░░░░░░░░░░░░░░░░░░░░░░░░   2%
   shard 4   ▓░░░░░░░░░░░░░░░░░░░░░░░░░░░   2%
       ...
   shard 14  ▓░░░░░░░░░░░░░░░░░░░░░░░░░░░   2%
   shard 15  ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓░  98%   ← p99 latency: 4 s
                                                  ← writes timing out
                                                  ← replication lag growing
```

Everything is "fine" — except for that one shard, which is the only one serving any actual load. Users notice; dashboards lie about cluster-average utilization.

The hot partition problem is **not a capacity problem**. It is a **distribution problem**. Adding shards doesn't help if the new shards stay cold.

---

## 2. Why It Happens — The Five Common Causes

### 2.1 Monotonic shard keys
The most frequent cause. The shard key is some time-correlated value (`created_at`, auto-increment ID, timestamp-based ULID/Snowflake when the high bits dominate), so **every new write hits the same shard** — the newest range or the last hash bucket.

```
shard 0: user_id [0–10M)        ← old, cold
shard 1: user_id [10M–20M)      ← old, cold
shard 2: user_id [20M–30M)      ← old, cold
shard 3: user_id [30M–40M)      ← ALL THE WRITES live here
```

This is the **hot tail** problem in range sharding. It bites users who range-shard by ID without thinking.

### 2.2 Celebrity keys
A single key receives orders-of-magnitude more traffic than the average.
- The Beyoncé Twitter account.
- The viral TikTok video's view counter.
- The Cyber Monday merchant in a multi-tenant database.
- The "default" or "system" user ID.
- The bank-of-last-resort account in a clearing system.

Hash sharding distributes *keys* evenly but not *traffic per key*. A single key always lives on one shard, and if that key gets 50% of the requests, that shard gets 50% of the traffic.

### 2.3 Skewed key distribution
The shard key has natural skew. Country = "US" might be 60% of users. Tenant = "BigCustomer" might be 90% of rows. Status = "active" might dominate.

This is the **low-cardinality shard key** failure mode. Even with hash sharding, only N unique values means at most N populated shards.

### 2.4 Hot range queries
Even if writes are evenly distributed, read patterns can concentrate. The most common case: "show me the most recent N items" hits the time-end of a range-sharded table. Pagination ("page 1") hits the start of every sorted index repeatedly.

### 2.5 Scheduled / cron-driven load
Every job runs at midnight UTC. Every report generates at 9 AM. Every backup happens at 02:00. The cluster average is fine; the 02:00 window is a riot.

If those workloads hit one shard (the "audit log" shard, the "metrics" shard), that shard burns for an hour.

---

## 3. Why It's So Damaging

Hot partitions have second-order failure modes that are nastier than the primary symptom:

- **Tail latency explodes.** p99 goes from 20 ms to 2 s while p50 looks fine.
- **Replication lag grows on the hot shard.** Replicas can't keep up with the primary's write rate; eventual consistency becomes very eventual.
- **Backups stall.** The backup tool tries to take a consistent snapshot; the hot shard refuses to quiesce.
- **Schema migrations stall.** Online migration tools (gh-ost, pt-osc, pg_repack) can't keep up on the hot shard.
- **Other workloads collateral-damage.** A noisy neighbor on a shared shard slows everyone else on that shard.
- **Failover risk.** Hot shard is closer to disk full, closer to OOM, closer to thread saturation.
- **Cost spirals.** You provision the whole cluster for the hot shard's worst case → 16× over-provisioning for a 16-shard cluster.

The cost asymmetry is what makes hot partitions an emergency. Other shards have all the headroom in the world; the workload is the bottleneck.

---

## 4. Famous Production Incidents

A few examples to internalize the pattern:

- **Twitter timeline (~2010)** — celebrity users' write fanout broke the timeline service repeatedly. Solution: hybrid fanout — push for normal users, pull-on-read for celebrities.
- **Instagram likes** — the "like" counter on viral posts was a hot key. Solution: counter sharding (one logical counter as N sub-counters that are aggregated on read).
- **DynamoDB pre-2018** — partition throughput was statically allocated per partition. A hot key throttled the partition. Adaptive capacity (2018) and on-demand mode (2018) mitigate, but careful key design is still recommended.
- **Cassandra default-user pattern** — apps that hashed `user_id` worked fine, then added "system" user with NULL user ID for system events → one shard ate the whole event firehose.
- **Kafka partition skew** — `KeyedProducer` with a low-cardinality key sends all messages to one partition; the consumer group hits a single-consumer bottleneck.
- **Redis Cluster slot 5474** — hot slot for a frequently-accessed key; the node hosting that slot saturates while others idle.
- **Pinterest sharding** — early sharding by user ID was fine until certain very-active users dominated; required directory-level isolation for whales.

These all share a shape: **everything looked fine until one key got popular, and then everything broke**.

---

## 5. Detection — Knowing You Have One

```
1. Per-shard CPU / disk / network / IOPS dashboards
   - Aggregate looks fine; one shard is 10× the others.

2. Per-key request rate sampled at the proxy / app layer
   - Sort by count(*) over the last minute by key.
   - The top 10 keys should be roughly balanced.
   - If the #1 key is 100× #10, you have a celebrity.

3. Per-partition metrics in the DB
   - Cassandra nodetool tablestats → partition_size, partition_count
   - DynamoDB CloudWatch ConsumedReadCapacityUnits per partition (proxied via metrics)
   - PostgreSQL pg_stat_user_tables → ratio of seq scans
   - Kafka kafka-topics --describe → partition assignment + lag

4. p99 / p999 latency per shard
   - p99 on one shard 50× the others = hot.

5. Throttling / error rate per partition
   - DynamoDB ProvisionedThroughputExceededException.
   - Cassandra timeouts on a single host.
   - Redis MOVED redirections concentrated on one slot.
```

Tools to make this visible:
- Datadog / Honeycomb traces grouped by partition key.
- Prometheus histograms per shard label.
- Cloud-native dashboards (DynamoDB Contributor Insights, Aurora Performance Insights).
- Custom sampling: log the partition key for 1% of requests; aggregate offline.

---

## 6. Fix Patterns

The right fix depends on what's causing the heat.

### 6.1 Bucket the key (the time-bucket trick)
For time-series data, append a time bucket to the shard key:

```
Before:  partition key = sensor_id              ← single growing partition per sensor
After:   partition key = (sensor_id, hour)      ← rotates hourly, bounded size
```

This is the standard Cassandra time-series recipe. Each (sensor, hour) is bounded; old buckets eventually drop via TTL. No partition grows forever, no partition becomes hotter than the others.

### 6.2 Salt the prefix
For monotonic keys, prepend a random or hashed prefix:

```
Before:  key = "logs/2026-05-19T12:00:00Z/..."
After:   key = "logs/8f3a/2026-05-19T12:00:00Z/..."
                    ^^^^ hash(key) mod 16

Before:  shard = range(user_id)
After:   shard = range((hash(user_id) mod 16, user_id))
```

Writes spread across N partitions. Reads must either know the prefix or query all prefixes (cheap if N is small).

This is the standard S3 high-write-prefix mitigation. Also widely used in HBase and Bigtable schemas.

### 6.3 Split the celebrity key
For a single hot key, split it into N sub-keys and aggregate on read:

```
Before:  INCR counter:viral_post_id
After:   INCR counter:viral_post_id:shard_0..15        (random pick on write)
         SUM(counter:viral_post_id:shard_*)            (on read)
```

Instagram, Twitter, and any system with hot like/view counters does this. Trade write simplicity for read complexity; absolute counts may lag a few seconds.

For databases:
- **Cassandra** — counter sharding with materialized aggregates.
- **DynamoDB** — *write sharding* on a single high-throughput partition key (AWS official guidance).
- **Redis** — `INCRBY` against a fan-out of keys, `SUMRANGE` on read.

### 6.4 Promote the hot tenant to its own shard
If the heat is a single tenant (whale customer, viral merchant), give them their own shard via **directory sharding**:

```
Before:  shop_id 12345 → shared shard B (with 9,999 small shops)
After:   shop_id 12345 → dedicated shard X
         All other small shops stay on shared shard B
```

Shopify, Slack, and most multi-tenant SaaS have an explicit "VIP shard" pattern. Sometimes the VIP gets a dedicated cluster for absolute isolation.

### 6.5 Read cache in front
If the hot partition is read-heavy (popular but small data), a cache solves it cheaply:

```
Read path:  cache (Redis/Memcached/CDN) → DB (only on miss)
```

A 99% cache hit rate turns 1M reads/sec into 10k DB reads/sec. The celebrity's profile, the viral post, the daily homepage — all good candidates.

See [Cache Pitfalls →](../05-caching/cache-pitfalls.md) for the failure modes (stampede, hot key in cache itself).

### 6.6 Change the read pattern
Sometimes the hot partition is your code's fault. Examples:
- "Get latest 100 messages" hammering one Cassandra partition → restructure to materialized views per-time-window.
- "Get all orders for tenant X" → denormalize per-tenant index into a different store.
- "Fanout on read" exploding for a celebrity → switch that user to fanout-on-write.

Twitter's [hybrid timeline architecture](https://www.infoq.com/presentations/twitter-timelines-2020/) is the canonical example of "change the access pattern" rather than fight the heat.

### 6.7 Two-tier counter / probabilistic structure
For pure counting (likes, views, ad impressions), exact precision often isn't required:
- **HyperLogLog** for unique-count approximation in O(1) memory.
- **Count-Min Sketch** for frequency approximation.
- Eventually-consistent counters with periodic reconciliation.

See [Probabilistic Data Structures →](../08-distributed-systems/probabilistic-data-structures.md).

### 6.8 Rate-limit the hot key at the edge
For abusive hot keys (one user spamming the API, one IP scraping aggressively), the answer is often **don't serve the traffic**:
- Per-key rate limiting at the API gateway.
- Token bucket scoped to the partition key.
- Backoff / degradation for known hot identifiers.

See [Rate Limiting →](../03-apis/rate-limiting.md).

### 6.9 Schedule jitter
For cron-driven heat, add randomized jitter to job start times so workload spreads:

```
Before: every customer's backup at 02:00
After:  backup at 02:00 + uniform(0, 3600)  → spread over an hour
```

This is the cheapest fix in the entire chapter and the most under-applied.

---

## 7. Worked Example — DynamoDB Hot Partition

**Problem**: A retail company stores audit events in DynamoDB. Partition key is `merchant_id`. One merchant — a Black Friday giant — has 100× the events of every other.

**Symptom**: That merchant's writes hit `ProvisionedThroughputExceededException` even when the table is mostly idle. The hot partition runs out of write capacity even though the table has tons of unused capacity.

**Fix path**:
1. **Diagnose**: enable DynamoDB Contributor Insights; confirm one partition key is dominant.
2. **Tactical** (same day): switch the table to on-demand mode → adaptive capacity helps; still throttles at very high single-key rates.
3. **Architectural** (next sprint): change partition key to `(merchant_id, shard_id)` where `shard_id` is `random(0, 9)`. Writes spread across 10 sub-partitions per merchant.
4. **Read path** (necessary follow-up): queries that previously fetched all events for a merchant now must fan-out across 10 shards and merge. Wrap in a helper.
5. **Backfill**: existing data either left as-is (queries handle the duality) or migrated via a one-time scan.

This is the textbook DynamoDB hot partition recipe.

---

## 8. Worked Example — Kafka Partition Skew

**Problem**: A Kafka topic uses `user_id` as the partition key. The application has 95% of traffic from one synthetic "system" user used for backfills, and the consumer group has 16 partitions but one consumer is constantly behind.

**Symptom**: One partition's lag grows continuously; CPU on one broker hot; aggregate broker load looks fine.

**Fix**:
1. **Tactical**: change the producer for the system user to use a round-robin partitioner (no key → uniform distribution).
2. **Strategic**: split the producer pipeline — system-user events go to a separate topic with no key; user events keep the keyed topic.
3. **Result**: keyed topic has balanced partitions; firehose topic round-robins.

Kafka partition skew is unusually easy to fix because producers control the key. Use it.

---

## 9. The General Method

When a hot partition shows up:

```
1. CONFIRM IT'S A HOT PARTITION (not a load spike)
   - Per-shard / per-partition metrics
   - Does one shard show 10×+ the others?

2. IDENTIFY THE KEY
   - Sample 1% of requests for a minute; group by partition key.
   - The top-1 should be obvious.

3. CLASSIFY THE CAUSE
   - Monotonic key?    → bucket/salt
   - Celebrity key?    → split / cache / promote / change pattern
   - Skewed dist?      → re-pick the shard key
   - Cron heat?        → jitter
   - Scheduled scan?   → throttle or push to off-peak

4. PICK A TACTICAL FIX (hours-to-days)
   - Cache, write-shard, isolate the tenant

5. PLAN THE STRATEGIC FIX (weeks-to-quarters)
   - Schema change, key reshape, access-pattern overhaul

6. ADD MONITORING
   - Top-N keys dashboard, per-shard load distribution.
   - Alert when distribution KL-divergence crosses threshold.
```

The most important step is **#6**. Hot partitions are visible only with per-shard / per-key telemetry. Without that, you debug by guessing.

---

## 10. Anti-Patterns

- **Solving by adding shards.** Adding cold shards doesn't help when one shard is hot. Must redistribute the existing heat.
- **Random sharding without reads in mind.** Spreading writes evenly via a salt prefix means reads must fan-out — sometimes worse than the original heat.
- **Caching without invalidation.** Stale cached celebrity data leads to user-visible inconsistency.
- **"It'll smooth out at scale."** Hot keys don't smooth — they sharpen. The fraction of traffic to the top key often *grows* as you scale.
- **Ignoring the bottom of the histogram.** A celebrity gets attention. The 1000 keys with 0.01% traffic each that *together* dominate often go unnoticed.
- **Hot-key allowlists.** "We hard-code an exception for user_id=42." Then user 43 goes viral.
- **Sharding by status / type / category.** Low-cardinality keys produce permanent skew; one value dominates.
- **No top-N dashboard.** You can't fix what you can't see.
- **Sharding the celebrity into too many sub-keys.** N=1000 sub-keys means 1000 reads to aggregate. Pick N to match traffic, not maximize spread.

---

## 11. Decision Quick-Reference

```
If the heat is:

  Monotonic key (timestamp, autoincrement)
    → bucket the key with a time window, OR
    → salt the prefix with hash bits

  One celebrity key (viral, big tenant)
    → split into N sub-keys + aggregate, OR
    → read cache, OR
    → promote to dedicated shard via directory

  Skewed key distribution (low cardinality)
    → choose a different shard key entirely
    → consider composite key with higher-card component

  Hot range read
    → materialized view per time-window
    → secondary index for the access pattern

  Cron / scheduled load
    → add jitter
    → throttle / queue / off-peak

  Hot partition in a queue (Kafka, SQS)
    → round-robin producer for unkeyed work
    → split topic by traffic class

  Hot key in cache itself
    → per-key replication / micro-shard the cache
    → request coalescing / single-flight
```

---

## 12. Cheat Card

```
PURPOSE     One shard / partition / key dominating traffic while
            the rest of the cluster idles. Total capacity is fine;
            distribution is broken.

CAUSES
  Monotonic shard key (timestamp, auto-id)     → hot tail
  Celebrity key (viral / VIP tenant)           → single-key heat
  Low-cardinality shard key                     → permanent skew
  Hot range read (latest N, page 1)            → read concentration
  Scheduled cron storm                          → time concentration

SYMPTOMS    One shard at 90%+, others <10%
            Tail latency spike, p99 cliff
            Replication lag on one shard only
            Throttling / timeouts on specific keys
            "Cluster average is fine" while users are broken

FIXES
  bucket the key          monotonic / time-series
  salt the prefix         random hex prefix on hot writes
  split celebrity         N sub-counters / sub-keys + sum on read
  promote to its own shard   directory sharding for whales
  read cache               popular small data
  change access pattern    fan-out vs fan-in, hybrid timelines
  schedule jitter          spread cron load over an hour
  rate-limit the hot key   abusive identifiers

PITFALLS    Adding shards (doesn't help) · over-spreading and
            killing reads · sharding by status / category · no
            top-N visibility · hard-coded exceptions · pretending
            it'll smooth at scale

RULE        Hot partitions are distribution bugs, not capacity bugs.
            Top-N key dashboards are the cheapest insurance you'll
            ever buy.
```

---

## 13. Resources

### Books
- *Designing Data-Intensive Applications* — Martin Kleppmann. Chapter 6 partitioning + section on skew.
- *Database Internals* — Alex Petrov.

### Articles
- "Best Practices for DynamoDB Partition Keys" — AWS docs.
- "DynamoDB Adaptive Capacity Explained" — AWS engineering blog.
- "How Discord Solved Hot Partitions in Cassandra" — Discord engineering blog.
- "Sharding Pinterest" — early example, hot-tenant pattern.
- "Twitter Timelines at Scale" — InfoQ talk, celebrity user fanout.
- "Slack Vitess Migration" — per-tenant directory sharding for hot teams.
- "Instagram Counter Sharding" — engineering blog history.
- "Avoiding Hot Spots in Cassandra Time Series Data Modeling" — DataStax academy.

### Videos
- ByteByteGo — "Hot Partitions and How to Fix Them."
- Hussein Nasser — multiple videos on sharding and hot partitions.
- AWS re:Invent — DynamoDB Deep Dive sessions (recurring).
- Strange Loop — talks on Twitter / Instagram fanout architectures.

### Tools
- **DynamoDB Contributor Insights** — top-N partition keys.
- **Cassandra nodetool tablestats / tablehistograms** — per-table heat.
- **Kafka kafka-consumer-groups + kafka-topics --describe** — partition lag and assignment.
- **Datadog / Honeycomb / Lightstep** — per-key tracing and aggregation.
- **pg_stat_statements + pg_partitions** — Postgres partition heat.

### Adjacent reading
- [Database Sharding Strategies](./sharding-strategies.md)
- [Sharding & Partitioning](../04-databases/sharding-partitioning.md)
- [Consistent Hashing](../04-databases/consistent-hashing.md)
- [Cache Pitfalls](../05-caching/cache-pitfalls.md)
- [Rate Limiting](../03-apis/rate-limiting.md)
- [Backpressure →](./backpressure.md)
- [Probabilistic Data Structures](../08-distributed-systems/probabilistic-data-structures.md)

---

*Previous:* [← Database Sharding Strategies (Range, Hash, Geo, Directory)](./sharding-strategies.md)  |  *Next:* [Capacity Planning →](./capacity-planning.md)
