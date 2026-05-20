# Failover & Disaster Recovery

> **TL;DR** — **Failover** is the runtime mechanism that shifts traffic from a failed component to a healthy replacement — automatically, ideally in seconds. **Disaster recovery (DR)** is the broader practice of restoring service after catastrophic loss (region down, datacenter on fire, ransomware, accidental `DROP TABLE`). The two terms blur but mean different things: failover is "the standby takes over"; DR is "we rebuild from backups in another region." Both live and die by two numbers — **RPO** (Recovery Point Objective: max acceptable data loss) and **RTO** (Recovery Time Objective: max acceptable downtime). Picking RPO/RTO honestly is the entire game; the architectures, costs, and operational practices flow from those numbers. The harder truth: **untested failover doesn't work**. If you haven't run a real DR drill in the last 90 days, assume the failover scripts are broken, the standby is misconfigured, and the backups can't be restored.

---

## 1. The Two Numbers — RPO and RTO

```
RPO  Recovery Point Objective
     "How much data can we afford to lose?"
     Window between the last durable replication/backup and the
     moment of failure.

RTO  Recovery Time Objective
     "How long can we be down?"
     Time from failure to fully restored service.
```

```
                  ─────────────┬────────────────────
                  RPO          │            RTO
   ←─────────────►             │  ←──────────────────►
   last           failure      restore           service
   replicated     event        starts            restored
   state
                       ▼
                  some data lost = RPO window
```

Every DR conversation should start with explicit RPO and RTO targets. Without them you're discussing aesthetics.

### Typical targets

| Workload | RPO | RTO | Implication |
|---|---|---|---|
| Financial system of record | 0 (no loss) | seconds | Synchronous replication, hot standby, active-active |
| OLTP critical (payments, orders) | <1 s | <1 min | Sync replicas in same region, near-zero-lag cross-region async |
| OLTP general | <1 min | <5 min | Streaming replication + standby + automated failover |
| Internal SaaS | <15 min | <1 h | Async replica + manual failover |
| Analytics / OLAP | hours | hours | Daily snapshot + replay |
| Cold archives | days | days | Periodic snapshots; rebuild on demand |

RPO and RTO drive cost. RPO = 0 + RTO = 0 is expensive (active-active, multi-region, consensus protocols). RPO = hours + RTO = days is cheap (one nightly backup).

Pick what the business actually needs, not what marketing wants on the brochure.

---

## 2. The DR Tier Ladder

A useful framework: name your DR tier and what it means.

```
TIER 0   No DR. Restore from whatever's there.
         RPO/RTO: undefined. Hope.

TIER 1   Backup & Restore (pilot light dark)
         Backups in another region; nothing running.
         RPO: backup interval (hours-days)
         RTO: hours (build infra + restore)
         Cost: cheapest

TIER 2   Pilot Light
         Minimal infrastructure in DR region (DBs running, replica;
         compute scaled to zero).
         RPO: replication lag (seconds-minutes)
         RTO: ~30 min (scale compute, switch traffic)
         Cost: ~10–15% of primary

TIER 3   Warm Standby
         Scaled-down full stack in DR region.
         RPO: seconds-minutes
         RTO: 5–15 min
         Cost: ~50% of primary

TIER 4   Hot Standby / Active-Passive
         Full-capacity DR region serving no traffic but ready to.
         RPO: seconds
         RTO: 1–5 min (DNS flip)
         Cost: ~100% of primary

TIER 5   Active-Active
         Both regions serve real traffic continuously.
         RPO: 0 (synchronous) or seconds (async, with conflict
              resolution)
         RTO: 0 (no failover needed — traffic shifts to healthy region)
         Cost: ~150–200%+ of single region
```

Most real production systems sit at Tier 2 or 3 for the bulk of services and Tier 4 or 5 for the critical core.

---

## 3. Failover vs Disaster Recovery

The distinction matters:

| Failover | Disaster Recovery |
|---|---|
| Automatic (usually) | Often manual decision |
| Component-level (DB, AZ, region) | Region or platform-level |
| Seconds to minutes | Minutes to hours (or longer) |
| Runs many times per year | Runs rarely (we hope) |
| Tested via drills + chaos | Tested via game days |
| Standby is ready | DR site may need building |
| Same architecture both sides | Possibly degraded DR architecture |

In practice, the distinction blurs. AZ failover is often automatic; region failover is often a deliberate human decision because the stakes (data loss, split-brain, customer comms) demand caution.

---

## 4. The Mechanics of Failover

### Detection
The failover starts with knowing something is broken. Detection mechanisms:
- **Health checks** at LB / DNS / consensus layer.
- **Heartbeats** between active and standby.
- **Failure detectors** (φ-accumulating, SWIM-style gossip).
- **Manual operator decision** for ambiguous cases.

The detection must distinguish **dead** from **slow** — paging on a slow-but-alive primary is the classic false-positive that causes more outages than it prevents. See [Health Checks & Heartbeats →](../13-observability/health-checks.md).

### Decision
Who decides to fail over?
- **Automatic** with consensus (Raft / Patroni / RDS) — the cluster votes.
- **Automatic** with single arbiter — riskier, possible split-brain.
- **Manual** — human pushes the button.

Automatic is faster but riskier (split-brain, false positives). Manual is safer but slower. The right mix depends on the workload: stateless services can fail over automatically; databases often need a manual decision because a wrongly-promoted replica eats your data.

### Promotion
The new primary takes over:
- DB: standby promoted to writable; replication direction reversed.
- LB / DNS: traffic redirected to the new endpoint.
- Cache / session store: failover or rebuild.
- Background workers: pick up the new primary endpoint.

### Traffic shift
Several mechanisms:
- **DNS** — change record; respect TTLs (clients and resolvers may cache).
- **Anycast** — withdraw routes; traffic shifts immediately.
- **Load balancer endpoint update** — global LB pulls failed region.
- **Service mesh** — endpoint discovery reconfigures.
- **Client SDK** — knows about both endpoints; client-side failover.

DNS is the most common and the slowest. Anycast (BGP) and global LBs (Global Accelerator, Cloud Load Balancer) shift traffic in seconds.

### Verification
After failover, verify:
- New primary accepts writes.
- Replication is back (now from new primary to old / new replicas).
- All dependent services see the new primary.
- Data integrity is intact.

### Failback
The original primary comes back. Now what?
- Synchronize old primary from new primary as a replica.
- Once caught up, optionally switch back to original (or stay on the new one).

Failback is the under-tested half of the failover story. Document it; drill it.

---

## 5. Common Failover Scenarios

### Database failover

```
Primary fails:
   1. Detection (~10–30 s with conservative health checks)
   2. Standby promoted to primary (~5–60 s depending on DB)
   3. Application reconnects to new primary
   4. Old primary, when back, joins as standby

Tools: Postgres Patroni / repmgr, MySQL Group Replication / Orchestrator,
       MongoDB replica set elections, RDS / Aurora Multi-AZ
```

Critical: synchronous vs asynchronous replication.
- **Sync**: zero data loss on planned failover; primary blocks if replica down.
- **Async**: replica trails by some lag window; failover loses that window of writes.

Most production setups use **semi-sync** or **quorum-based sync** — at least one (or N of M) replicas must ack each commit. See [Replication →](../04-databases/replication.md) and [Quorum →](../08-distributed-systems/quorum.md).

### Application failover (stateless)
Stateless services fail over for free: LB health checks pull bad instances; new requests go elsewhere. Important: **in-flight requests must complete or fail cleanly** — set up graceful shutdown.

### AZ failover
Most cloud services (RDS Multi-AZ, ELB, ASG) handle this automatically. A single-AZ outage transitions to other AZs in minutes.

### Region failover
The big one. Components:
- DNS / Global LB redirect.
- DB cross-region failover (often manual).
- Cache rebuild in DR region.
- Verify application config / IAM / secrets work.
- Monitor for cascading issues during shift.

Region failover is rarely automatic and often takes 15–60 minutes even when well-rehearsed. Real reasons it takes that long:
- Caches must warm.
- Connection pools rebuild.
- Stragglers (background jobs, long-lived connections) take time to migrate.
- Verification of data integrity before re-enabling writes.

---

## 6. Backups — The Foundation

Failover assumes a working replica. DR assumes a working backup. The discipline:

### Backup types

- **Full backup**: complete copy of the data.
- **Incremental backup**: changes since the last full or incremental.
- **Differential backup**: changes since the last full.
- **Snapshot**: point-in-time block-level copy.
- **WAL archive / change log**: continuous shipping of every change.
- **Logical backup**: SQL dump or equivalent; portable but slower.
- **Physical backup**: file-level binary copy; fast but version-locked.

### Backup cadence
Typical:
- **Continuous WAL archive** to object storage.
- **Daily** full snapshot.
- **Hourly** incremental between snapshots.
- **Weekly / monthly** snapshots retained for longer windows (compliance, deletion recovery).

### Where backups live
The rule: **backups in a separate failure domain from the primary**.
- Different region.
- Different account / project (defense against ransomware, account compromise).
- Different storage class (S3 Standard primary, S3 IA / Glacier for older snapshots).
- Possibly different provider (rare; expensive; protects against provider-level events).

### Backup retention
Designed against multiple failure scenarios:
- **Daily**: 7–30 days. Operator mistakes, application bugs.
- **Weekly**: 3 months. Slower-discovered corruption.
- **Monthly**: 1 year. Compliance, long-tail recovery.
- **Yearly**: 7+ years. Regulatory.

### Point-in-time recovery (PITR)
Combining a full backup with continuous WAL archive lets you restore to any second in the retention window. Critical for recovering from "we ran a bad migration at 14:23, restore to 14:22."

Postgres + pgbackrest, MySQL + Percona XtraBackup, AWS RDS automated PITR — all variations on this theme.

### The most important rule

> **A backup you haven't restored is not a backup.**

Untested backups fail. Causes:
- Corruption discovered only at restore time.
- Missing dependencies (WAL gaps, schema mismatches).
- Permissions / credentials no one documented.
- Tooling that worked in test but not at scale.
- Compatibility breaks (DB version mismatch).

Test restores monthly. Yes, monthly. From a real snapshot. To an isolated environment. With timing. Document the runbook based on what actually happens.

---

## 7. Replication for Failover

### Synchronous
```
Primary commits → waits for replica ack → ACKs client
```
- RPO = 0 on graceful failover.
- Write latency = network RTT to replica.
- If replica is unreachable, primary blocks (or you've configured "fall back to async," which compromises RPO).

### Asynchronous
```
Primary commits → ACKs client → replicates lazily to replica
```
- RPO = replication lag (typically seconds; can spike to minutes under load).
- Write latency = local primary fsync.
- On unplanned failover, recent writes are lost.

### Semi-synchronous
```
Primary commits → waits for AT LEAST ONE replica to ack → ACKs client
```
- Compromise: RPO usually 0 with one healthy replica, async behavior otherwise.
- Used by MySQL semi-sync, Postgres `synchronous_standby_names = ANY 1 (s1, s2, s3)`.

### Quorum-based
```
Primary commits → waits for N of M replicas to ack
```
- Tunable durability vs latency.
- Used by Spanner, CockroachDB, Cassandra (with QUORUM consistency), Raft-based systems.

The choice is a [PACELC →](../08-distributed-systems/pacelc.md) decision: in normal operation, you're trading latency for consistency.

---

## 8. The Hard Parts

### Split-brain
After a partition + failover:
- Old primary thinks it's still primary on its side.
- New primary is taking writes on the other side.
- Two divergent histories.

Mitigations:
- **Fencing**: physical isolation of old primary (STONITH — Shoot The Other Node In The Head).
- **Lease / lock services**: ZooKeeper, etcd, Consul hand out leadership leases; no two nodes both hold a lease.
- **Quorum-based promotion**: only with majority agreement.
- **Read-only mode** for old primary if it can't confirm leadership.

See [Split-Brain Problem →](../08-distributed-systems/split-brain.md).

### Stale data after async failover
Async replicas trail. Failover promotes the most-current replica, but recent committed writes on the old primary may be lost. Production patterns:
- **Application reconciles**: idempotency keys + checkpoints help replay.
- **Outbox pattern**: durable event log in the database catches up downstream consumers.
- **Customer comms**: "Some recent activity may need to be re-entered."
- **Take the loss**: for many systems, losing a few seconds of writes is acceptable.

### Detection-to-action delay
Cassandra-style multi-second failure detectors trade false positives for slow detection. AZ failover may take 30–60 s. Region failover often takes 5–15 minutes including human-in-the-loop. Plan for this in latency budgets and customer expectations.

### Failover during a deploy
A bad deploy that crashes the primary right after the standby was also broken by the same deploy. Classic. Mitigation: progressive deploys (canary, region-by-region), keep standby on the previous version briefly, deploy DR region after primary stabilizes.

### Replication lag during failover
Standby is 30 s behind. Promoting it loses 30 s of writes. Worst case: write surge before failure made lag balloon to 5 minutes.

### Caches after failover
The new region's caches are cold. First requests slow → users notice. Mitigations: cache warming, pre-fetch popular keys, graceful degradation while caches warm.

### IAM, secrets, DNS, certs
The non-database state your system depends on. All must be available in the DR region. Replicate KMS keys, Vault secrets, DNS configs, TLS certs. These are the single most common reason "we couldn't failover" stories happen.

---

## 9. Operational Discipline — Game Days

The only failover that works is one that's been tested recently. The practice: **game days**.

### Quarterly game day
- Schedule with the team.
- Pick a scenario (region failover, DB failover, dependency outage).
- Execute the runbook.
- Time every step.
- Document what went wrong.
- Fix the gaps before the next quarter.

### What you'll find
- Runbooks reference dead URLs.
- Credentials expired.
- "Push this button" tools no one has access to.
- DNS TTLs longer than the runbook claims.
- Standby was running an old version.
- Backups have gaps.
- A dependency has its own failover dance no one knew about.

These findings are the whole point. The first game day finds the most. By the third, the team can do a region failover in an hour with mild stress instead of three hours of panic.

### Friction-reducing tools
- Documented runbooks in the same repo as the code.
- One-button (or one-command) failover scripts.
- Practice failover in staging weekly.
- Chaos engineering for routine validation. See [Chaos Engineering →](./chaos-engineering.md).

---

## 10. Worked Example — A Postgres Failover

A typical setup with Patroni + etcd:

```
Region us-east-1:
  AZ-a: primary (Patroni + Postgres)
  AZ-b: sync replica (Patroni + Postgres)
  AZ-c: async replica (Patroni + Postgres)
  etcd cluster across all 3 AZs

Region us-west-2:
  AZ-a: async replica (cross-region)
  daily snapshots to S3
  WAL archive to S3 with 15-min replay window
```

Failover scenarios:

**Primary instance dies**:
1. etcd notices missed heartbeats (~10 s).
2. Patroni promotes sync replica.
3. Application reconnects via DNS or virtual IP.
4. Total RPO: 0 (sync ack). Total RTO: ~30 s.

**AZ-a dies**:
- Same flow; sync replica in AZ-b takes over.

**Region us-east-1 dies**:
1. Manual decision (significant stakes).
2. Promote us-west-2 replica.
3. Update DNS / global LB.
4. Validate data integrity, replication lag at moment of failure.
5. Reroute traffic.
6. Total RPO: replication lag (seconds-minutes). Total RTO: 30 min – 1 h.

**Bad migration at 14:23**:
1. PITR restore from snapshot + WAL replay to 14:22.
2. Validate.
3. Replace primary with restored DB.
4. Replay missing writes from outbox / logs if possible.
5. Total RPO: 1 minute. Total RTO: 1–4 hours depending on data size.

This is roughly what production Postgres looks like at companies that take reliability seriously.

---

## 11. Cloud Service Failover

The good news: most managed services do automatic AZ failover for free.

- **AWS RDS Multi-AZ**: automatic failover in ~60 s. RPO 0 (sync). Single-region only.
- **Aurora**: faster (~30 s). Multi-AZ by design.
- **Aurora Global Database**: cross-region replicas with <1 s lag; manual promotion.
- **DynamoDB**: multi-AZ inside region by default. Global Tables for cross-region active-active.
- **ElastiCache Multi-AZ**: automatic failover. RPO depends on snapshot frequency.
- **S3**: multi-AZ by default; Cross-Region Replication for cross-region.
- **GCP Cloud SQL HA / Spanner**: similar shape; Spanner is cross-region by design.
- **Azure SQL DB / Cosmos DB**: built-in HA + georeplication.

The catch: **managed services hide complexity, not consequences**. RDS Multi-AZ doesn't help if your application code can't reconnect after a 60 s outage. Test the failover from the application's perspective.

---

## 12. Common Mistakes / Anti-Patterns

- **Untested failover.** It's broken. Statistical certainty.
- **Untested backups.** Same.
- **RPO/RTO targets you can't meet.** Marketing claims of "99.99% availability" with daily backups in another region. Math doesn't lie.
- **RPO/RTO targets you don't need.** Spending 10× budget for 99.999% on a service the business is fine being down for an hour.
- **Backups in the same account / region as primary.** Account compromise or region disaster takes both.
- **No PITR.** You can restore to "yesterday at midnight" but not "14:22 today."
- **Async replication treated as zero data loss.** It isn't. Plan for the lag window.
- **Split-brain unprotected.** No fencing → two primaries → divergent data.
- **No fencing.** Old primary keeps serving writes after promotion.
- **DNS TTLs in the failover plan.** Resolvers cache; clients cache; failover takes longer than the runbook claims.
- **Standby on stale config / version.** Failover succeeds; service fails because the standby doesn't know about new features.
- **No application-level reconciliation.** Some writes lost; no way to detect or replay.
- **No drills.** Untested means broken.
- **Game day in a friendly environment.** Run drills under production-like load.
- **One person knows the failover process.** They're on vacation when you need them.
- **Failback unplanned.** You failover once and now run on the DR region for months because you don't know how to fail back.
- **Trusting the dashboard during a failure.** Dashboards often share infrastructure with the failed service.
- **Backups encrypted with keys only in the failed region.** Backups exist but you can't decrypt them.
- **No business communication plan.** Engineering knows what's happening; customers see silence.

---

## 13. Decision Rule

```
For every system:
  1. Pick honest RPO and RTO.
  2. Pick a DR tier (1–5) that matches.
  3. Implement replication / standby / DR infrastructure for that tier.
  4. Document the failover runbook.
  5. Drill it quarterly.
  6. Drill failback too.

For backups:
  - Multiple cadences (daily, weekly, monthly).
  - Separate failure domain (account, region, possibly provider).
  - PITR if RPO < hourly.
  - Test restore monthly.
  - Document retention.

For replication:
  - Sync if RPO = 0 is required.
  - Async with quantified lag otherwise.
  - Fencing / quorum / leases against split-brain.

For region failover:
  - Manual decision for high-stakes systems.
  - Automatic for stateless.
  - Tested. Always tested.
```

---

## 14. Cheat Card

```
PURPOSE     Failover shifts traffic to a healthy replacement when
            something breaks. DR restores service after a catastrophic
            event. Both are measured by RPO and RTO.

RPO    max acceptable data loss     (0, seconds, minutes, hours)
RTO    max acceptable downtime      (seconds, minutes, hours, days)

TIERS  0  no DR                      hope
       1  backup & restore           cheapest; RPO hours, RTO hours
       2  pilot light                ~15% cost; DBs running, compute off
       3  warm standby               ~50%; scaled-down full stack
       4  hot standby (active-passive) ~100%; ready, no traffic
       5  active-active              ~150–200%; no failover needed

MECHANICS  Detection → decision → promotion → traffic shift →
           verification → failback

REPLICATION
  Sync           RPO=0, latency=RTT
  Async          RPO=lag, latency=local
  Semi-sync      RPO=0 with one healthy replica
  Quorum         tunable; consensus-based

HARD PARTS  Split-brain · stale data · detection-to-action delay ·
            cold caches after failover · IAM / secrets / DNS in DR ·
            untested failback

BACKUPS    Continuous WAL + daily full + weekly + monthly + yearly
           In separate failure domain (account, region)
           PITR for fine-grained recovery
           Restore tested monthly

GAME DAYS  Quarterly minimum. Without them, failover doesn't work.
           First drill finds the most issues; subsequent ones tune.

PITFALLS   Untested failover · untested backups · backups in same
           failure domain · no fencing · no PITR · async treated
           as zero-loss · DNS TTL surprises · stale standby config ·
           one-person dependency · no failback plan · no customer
           comms plan

RULE       Failover that hasn't been tested in 90 days is broken.
           Backups that haven't been restored aren't backups.
           Pick RPO/RTO from business needs, then design and price
           the system around them.
```

---

## 15. Resources

### Books
- *Site Reliability Engineering* — Google. Chapters on availability, error budgets, and incident response.
- *Database Reliability Engineering* — Laine Campbell, Charity Majors. Practical DR for databases.
- *Designing Data-Intensive Applications* — Martin Kleppmann. Replication, consensus, and consistency.
- *Patterns of Distributed Systems* — Unmesh Joshi. Leader election, replication patterns.

### Articles
- "Static Stability Using Availability Zones" — AWS Builders' Library: <https://aws.amazon.com/builders-library/static-stability-using-availability-zones/>
- "Failover and Disaster Recovery" — AWS Well-Architected Reliability Pillar.
- "Postgres High Availability" — Patroni / repmgr documentation.
- "MySQL Orchestrator" — GitHub engineering on MySQL failover automation.
- "MongoDB Replica Set Elections" — MongoDB docs.
- "Spanner: Truetime and Global Consistency" — Google paper.
- "Stripe Online Migrations" — engineering on zero-downtime changes.
- "Slack's Migration to Vitess" — failover and DR in a sharded system.
- "Disaster Recovery at Stripe" — engineering blog.

### Videos
- AWS re:Invent — "DR for the Cloud" and "Multi-Region Architectures" tracks.
- "How GitHub Failover Works" — GitHub Universe talks.
- SREcon — many talks on game days and DR drills.
- ByteByteGo — "Disaster Recovery Strategies."

### Tools
- **Patroni** — Postgres HA / failover orchestration.
- **repmgr** — alternative Postgres HA.
- **GitHub Orchestrator** — MySQL failover automation.
- **AWS RDS Multi-AZ / Aurora Global / Aurora Multi-Master** — managed failover.
- **GCP Cloud SQL HA**, **Azure SQL DB GeoReplication** — managed equivalents.
- **etcd / ZooKeeper / Consul** — leader election and fencing.
- **pgbackrest / wal-g / Percona XtraBackup** — backup tools with PITR.
- **Velero** — Kubernetes resource backup / DR.
- **AWS Backup, GCP Backup & DR Service** — managed backup platforms.
- **Gremlin / Chaos Mesh / Chaos Monkey** — chaos engineering for failover drills.

### Adjacent reading
- [Replication (Master-Slave, Master-Master, Multi-Region)](../04-databases/replication.md)
- [Multi-Region](../10-scalability/multi-region.md)
- [Quorum-Based Replication](../08-distributed-systems/quorum.md)
- [Split-Brain Problem](../08-distributed-systems/split-brain.md)
- [Leader Election](../08-distributed-systems/leader-election.md)
- [WAL — Write-Ahead Logging](../09-storage/wal.md)
- [Chaos Engineering →](./chaos-engineering.md)
- [Blast Radius & Cell-Based Architecture →](./cell-architecture.md)
- [Health Checks & Heartbeats](../13-observability/health-checks.md)

---

*Previous:* [← Graceful Degradation](./graceful-degradation.md)  |  *Next:* [Chaos Engineering →](./chaos-engineering.md)
