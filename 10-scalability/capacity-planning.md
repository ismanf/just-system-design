# Capacity Planning

> **TL;DR** — **Capacity planning** is the practice of figuring out how much hardware (CPU, RAM, IO, network, storage) and headroom you need to serve a known or projected workload at acceptable latency and reliability. It's a mix of **back-of-the-envelope arithmetic**, **load testing**, **observability**, and **judgment about growth and surprises**. The two failure modes are both expensive: under-provisioning (latency, errors, outages) and over-provisioning (wasted money, complacency). The discipline lives between rough math and production reality — the goal isn't a perfect number, it's a defensible plan with explicit headroom for traffic spikes, retries, GC pauses, and the failure of any single component. Done right, capacity planning is what stops you from being surprised by Black Friday, regional failover, or a viral tweet.

---

## 1. What Capacity Planning Actually Asks

Three questions, in order:

1. **How much traffic / data / load do we expect?** (Demand)
2. **What does our system consume per unit of that load?** (Cost-per-unit)
3. **How much capacity do we provision, with what headroom?** (Supply + safety)

Answer those three and you've done capacity planning. Most of the difficulty is in the inputs being uncertain — projected traffic, mystery cost-per-request, and the headroom you have to *pick* rather than derive.

```
Required capacity = (expected load × cost per unit) × headroom factor
                  ───────────────────────────────────────────────
                                  utilization target
```

The utilization target captures "we don't run hot." Typical values: 60–70% sustained CPU is healthy; >80% is risk; >90% is on the edge.

---

## 2. Back-of-the-Envelope First

Before opening dashboards, get the order of magnitude right. See [Back-of-the-Envelope Estimation →](../01-foundations/estimation.md) and [Latency Numbers →](../01-foundations/latency-numbers.md).

A simple example: a chat service expecting **10M daily active users**, each sending **20 messages/day**, peak/avg = 3×.

```
Average message rate:
   10M × 20 / 86,400 s ≈ 2,300 messages/sec

Peak message rate:
   2,300 × 3 ≈ 7,000 messages/sec

Per-message cost (measured): 0.5 ms of one CPU core for write path
   7,000 msg/sec × 0.5 ms = 3.5 CPU-seconds/sec
   ≈ 4 cores at peak

With 70% utilization target:
   4 / 0.7 ≈ 6 cores

With +50% headroom for spikes/failovers:
   6 × 1.5 ≈ 9 cores

Storage:
   200 bytes/msg × 7,000 msg/sec × 86,400 = 120 GB/day
   Hot retention 30 days → 3.6 TB
   3× replication → 10.8 TB
```

That's enough to scope a fleet, pick a database, and shape a deployment. Refine with real measurements as soon as you have them.

---

## 3. The Inputs You Need

### Demand
- **Current load**: requests/sec, writes/sec, GB ingested/day, peak vs average.
- **Growth rate**: month-over-month, year-over-year, seasonal.
- **Peak ratio**: peak/avg traffic. For consumer products, 2–5×; for retail, can be 50×+ on Black Friday.
- **Distribution shape**: smooth diurnal vs sharp event-driven.

### Cost per unit
- CPU per request
- RAM per session / connection / cached object
- Disk IOPS per write, bytes per write
- Network bandwidth per response
- DB cost per query (read/write)

These come from measurement: load tests, production profiles, or knowledge of similar systems.

### Constraints
- Latency SLO (p50, p99, p999)
- Availability SLO (% uptime, error budget)
- Durability SLO (data loss tolerance)
- Cost budget
- Operational complexity tolerance

### Headroom / safety
- Failure of N% of capacity (typically 1 AZ or 1 region)
- Background work (compactions, backups, repairs)
- GC pauses, tail latency events
- Retries and amplification
- Mystery surprise factor

---

## 4. Utilization Targets

The single most important number in capacity planning is **how full you run things**. The temptation is to run at 90% to save money. The reality is that systems behave nonlinearly above ~70%.

```
        latency
          ▲
          │                            *
          │                          *
          │                        *
          │                      *
          │                    *
          │                  *
          │               *
          │           *
          │ * * * *
          └─────────────────────────────────► utilization
            0%   30%   50%   70%   85%   95%
```

The "knee" is real. Past it:
- Queues grow.
- Tail latency explodes (p99/p999 fall off a cliff while p50 still looks fine).
- Headroom for spikes evaporates.
- Failover capacity disappears.

Rule of thumb starting points:

| Component | Healthy max sustained util |
|---|---|
| CPU | 60–70% |
| Memory | 70–80% (leave room for spikes, caches) |
| Disk IOPS | 60% |
| Disk throughput | 70% |
| Network bandwidth | 50–60% |
| Database connections | 70% of pool |
| Queue depth | <1 (anything queueing is a smell) |
| Thread pools | 60–70% |

These are **starting points**. Specific systems vary — Redis can run at 90% CPU happily; a JVM doing GC will hate even 70% sustained.

---

## 5. The N+1 / N+2 Pattern

Plan for **failure of capacity**, not just peak demand.

```
N      = capacity needed at peak with everything healthy
N+1    = capacity needed when one unit (AZ, rack, region) is down
N+2    = capacity when two units are down (rare; high-availability tier only)
```

Concrete example: 3 AZs, peak load = 30k QPS, each AZ takes 10k QPS at steady state.
- **N+1** plan: each AZ provisioned for 15k QPS — if one AZ dies, the other two absorb. Cluster utilization in steady state = 67%.
- **N+2** plan: each AZ provisioned for 30k QPS — survive a 2-AZ outage. Cluster utilization in steady state = 33%.

N+1 is the standard for most production systems. N+2 is for the systems where outages cost millions per hour.

The cost: visible over-provisioning. **You pay for capacity you don't normally use, because the day you need it you will need it badly.**

This is why "cluster looks 33% utilized" can be the *correct* answer in a healthy three-AZ deployment — anything higher means you can't survive the loss of one AZ.

---

## 6. Headroom for Tail Behaviors

Even when steady-state load is below the utilization target, real systems have spike sources:

- **Diurnal peaks** — 2–5× average; size for peak, not average.
- **Weekly / monthly peaks** — payday, Mondays, ends of quarters.
- **Annual peaks** — Black Friday, Cyber Monday, Super Bowl, World Cup.
- **Event-driven spikes** — viral content, marketing launches, news events.
- **Retry storms** — a downstream slowdown amplifies inbound traffic; without backoff, retries double or triple effective load.
- **Cache stampedes** — invalidation event + cold cache = traffic burst against the origin.
- **Deployment effects** — rolling deployments halve fleet capacity transiently.
- **Background work** — compactions, backups, garbage collection, scheduled reports.

Plan accordingly:
- Provision for **peak + 50%** as a baseline.
- Provision for **3× average** for consumer products with diurnal cycles.
- Provision for **20× average** for retail at peak season.
- Bake in **rate limiting + backpressure** to cap retry amplification.

---

## 7. The Storage Side

Storage capacity planning is a separate exercise from compute. Different inputs:

- **Daily ingest rate** — events × bytes/event.
- **Retention** — how long data lives.
- **Compression ratio** — depends on encoding (Parquet, Snappy, Zstd) and data shape.
- **Replication / EC overhead** — see [Erasure Coding vs Replication →](../09-storage/erasure-coding.md).
- **Backup overhead** — point-in-time copies, archive retention.
- **Tier-by-temperature** — hot to NVMe, warm to HDD, cold to object/glacier. See [Compaction & Tiered Storage →](../09-storage/compaction.md).

Worked example: an analytics system ingesting **500 GB/day** raw, **100 GB/day** compressed Parquet, 90-day hot tier on S3, 7-year cold tier on Deep Archive.

```
Hot:      100 GB/day × 90 days = 9 TB     ($0.023/GB-month × 9000 GB = $207/month)
Cold:     100 GB/day × 7 years = 256 TB   ($0.001/GB-month × 256,000 GB = $256/month)
Backups:  +10% of hot                     ($21/month)
Total:                                    ~$485/month + request costs
```

The order-of-magnitude difference between hot and cold tier is why tiering policies are load-bearing. Without them, that 256 TB would be $5,700/month instead of $256.

---

## 8. Latency-Driven Capacity

Throughput planning is one axis. Latency planning is the other — and they don't always agree.

A system at 60% CPU may still violate latency SLOs because:
- A small number of slow queries dominate p99.
- GC pauses inject 100 ms spikes.
- Locking causes contention.
- Network jitter on cross-AZ calls.
- Cold caches on deploy.

Capacity for latency means:
- **Provision so each request has a clear queue-free path.** Heavier provisioning than throughput suggests.
- **Tail-latency-aware deployments.** See [Tail Latency & p99 →](../16-performance/tail-latency.md).
- **Hedged requests / parallel attempts** at the application level.
- **More smaller instances** instead of fewer large ones (reduces the impact of single-instance spikes).

Latency budgets compose: if you have 200 ms total, and the DB takes 80, the cache takes 5, the LB takes 5, network takes 10, you have 100 ms for your service. Each downstream sets its own SLO; capacity per service is sized against its slice.

---

## 9. Cost Modeling

Capacity planning that ignores cost is incomplete. The shape of the bill:

| Layer | Common costs |
|---|---|
| Compute | EC2 / GCE instances by hour |
| Storage | EBS, S3, Glacier per GB-month + request fees |
| Network | Egress, cross-AZ, cross-region, load balancer |
| Database | RDS / Aurora / DynamoDB units + storage + IO |
| Cache | Redis / Memcached node-hours |
| Edge | CDN bandwidth + requests |
| Observability | Logs, metrics, traces (often a top-3 line item) |

A useful exercise: **compute cost per request** and **cost per active user**. If a user costs $0.50/month to serve but your ARPU is $0.10, you have a business model problem disguised as a capacity problem.

Knobs to pull on cost:
- **Reserved / committed instances** vs on-demand (40–70% discount for 1-yr / 3-yr).
- **Spot / preemptible** for batch / fault-tolerant workloads (50–90% off).
- **Autoscaling** to match capacity to actual load.
- **Tiered storage** to push cold data off hot media.
- **CDN** to reduce origin traffic.
- **Compression** to reduce egress and storage.
- **Right-sizing** — moving from `m5.xlarge` to `m6i.large` etc.
- **Multi-tenant pooling** instead of per-tenant infrastructure.

---

## 10. Load Testing — From Math to Measurement

Math gives you order of magnitude. **Load tests give you confidence.**

### Patterns
- **Smoke test** — small, quick, "does it work?"
- **Load test** — sustained at expected peak; measures latency, errors, resource use.
- **Stress test** — push past peak until it breaks; learn the failure mode.
- **Soak test** — sustained for hours; finds leaks, slow degradation.
- **Spike test** — sudden load jump; finds autoscaling lag and cold-cache effects.
- **Chaos test** — kill components mid-load; validate failover. See [Chaos Engineering →](../11-reliability/chaos-engineering.md).

### Tools
- **k6** — modern, scriptable, popular.
- **Locust** — Python, good for complex user flows.
- **Vegeta** — Go, simple HTTP attack tool.
- **wrk / wrk2** — high-throughput micro-benchmarking.
- **Gatling** — Scala/Java, enterprise heritage.
- **AWS Distributed Load Testing**, **Azure Load Testing** — cloud-native.
- **k6 / Locust / Artillery** for synthetic API load.
- **JMeter** — old but still common.

### What to measure
- **Throughput** (req/s) sustained without errors.
- **Latency distribution** — p50, p95, p99, p999.
- **Error rate** at each load level.
- **Resource utilization** — CPU, memory, IO, network on each component.
- **Queue depths** at every queueing point.
- **Breakpoint** — at what load do errors / latency spike?

### Gotchas
- **Test in a realistic environment.** Production-like data, network, latencies. A 1k-row dev DB is not your production DB.
- **Coordinated omission** — naive RPS-based tools under-count tail latency by ignoring requests delayed by the test client itself. Use wrk2, k6 (constant-arrival-rate executor), or Gil Tene's HdrHistogram-based tools.
- **Synthetic data ≠ real data.** Hot keys, skew, and access patterns matter. Replay production traces if possible.
- **Warm-up matters.** First-hit latency is much worse than warm latency. Run for at least 5–10 minutes before measuring.
- **Don't extrapolate linearly.** A system that handles 10k RPS may not handle 100k by 10× the boxes; nonlinear effects (lock contention, GC, network) kick in.

---

## 11. Observability Is Capacity Planning Fuel

You can't plan capacity if you can't see actual cost-per-request. Required telemetry:

- **Per-endpoint latency & throughput** histograms.
- **Per-component utilization** (CPU/RAM/IO/net per service, per host).
- **Per-query cost** (DB time, IO, bytes).
- **Queue depths** at every queueing point (LB, pool, message broker).
- **Error rates** by class.
- **Cost dashboards** mapped to services / features / tenants.

See [Logging](../13-observability/logging.md), [Metrics](../13-observability/metrics.md), [Tracing](../13-observability/tracing.md), and the upcoming [Three Pillars](../13-observability/three-pillars.md).

A system without these is being capacity-planned by guesswork. The single biggest leverage in capacity planning is **better telemetry** — once you can see per-request cost, planning becomes arithmetic.

---

## 12. Growth Planning — Looking Forward

Daily capacity plans look at "today." Strategic capacity planning looks at "the next 12–18 months."

### Inputs
- **Trended growth** — last 12 months, extrapolated.
- **Business plans** — launches, expansions, marketing pushes.
- **Cohort growth** — new users today × retention curve = users in N months.
- **Storage compounding** — events keep accumulating; storage grows even when traffic doesn't.
- **Geographic expansion** — new regions, new compliance.

### Outputs
- Compute scale (cores, machines) at +6, +12, +18 months.
- Storage scale (TB/PB) at +6, +12, +18, +36 months.
- Network egress projections.
- Cost projections.
- Architectural inflection points (when to shard, when to add region, when to move to a new database).

### Honest practice
The point of long-horizon planning isn't precision — it's spotting inflection points early. "We'll hit our single-DB write ceiling in 9 months at current growth" is the kind of finding that prevents emergencies.

Re-plan quarterly. Reality drifts.

---

## 13. Capacity Planning by Workload Type

Different workloads have different planning shapes:

### Stateless web tier
- Easy. Autoscale on CPU / requests.
- Sized for peak with headroom; spikes absorbed by autoscaling.
- Cost roughly linear with traffic.

### Database (single primary)
- Hard. Can't autoscale writes. Vertical scaling is the lever.
- Plan headroom over multi-year horizon — migrating between instance classes is disruptive.
- Watch for write-saturation 12+ months out.

### Database (sharded)
- Plan per-shard headroom + cluster-wide.
- Resharding is multi-quarter work — leave space.

### Cache layer
- Memory is the constraint.
- Plan for working-set size + 25% growth + replication.
- Hit-rate degradation when caches fill is sudden, not gradual.

### Message brokers / streams
- Disk + retention + partition count.
- Number of partitions caps parallelism — pick wisely; resizing is painful.
- Plan for at least 2× retention (failure recovery, replay).

### Search / analytics
- Storage compounds aggressively.
- Index size, JVM heap, query parallelism all interrelated.
- Plan for **shrink ratio** — actual indexed bytes / raw bytes.

### Background workers / batch
- Queue depth is the planning unit.
- Worker count = (work rate × work duration) / utilization target.
- Plan for catch-up after outages — sustained 2× capacity.

### Object storage
- Storage is "infinite" but request rates throttle.
- Plan prefix sharding for high-write workloads.
- Lifecycle policies fundamentally change the storage cost curve.

---

## 14. Common Mistakes / Anti-Patterns

- **Sizing for average load.** A 2–5× peak ratio is normal; you'll be on fire daily.
- **Sizing for current load with no growth budget.** Surprises in 4 months.
- **Running at >80% utilization sustained.** Tail latency dies first, then everything else.
- **No N+1 plan.** Lose an AZ, lose the service.
- **Ignoring retry amplification.** Downstream slow → retries → upstream traffic 2–3× → cascading collapse.
- **Capacity planning without per-request cost data.** You're guessing.
- **Load testing only the happy path.** Errors, slow queries, hot keys all change the cost shape.
- **Coordinated omission in load tests.** p99 looks 10× better than reality.
- **Optimizing for cost without latency targets.** Cheapest provisioned config violates SLO.
- **Optimizing for latency without cost discipline.** Easy to spend 10× for the last 5 ms.
- **No headroom for deploys.** Rolling deploys halve fleet — but you didn't plan for it.
- **Forgetting storage compounds.** Compute steady-state; storage cumulative.
- **Treating cloud as infinite.** Subnet IP exhaustion, quota limits, region capacity limits all exist.
- **No quarterly re-planning.** Markets, products, and growth all move.

---

## 15. The Iterative Loop

Real capacity planning is a continuous loop:

```
1. PROJECT      demand for the planning horizon
2. MEASURE      current cost-per-unit
3. COMPUTE      required capacity with headroom
4. PROVISION    deploy, validate via load test
5. OBSERVE      production reality vs plan
6. ADJUST       refine inputs, repeat next cycle
```

The right cadence:
- **Daily/weekly** — auto-scaling, on-call dashboards.
- **Monthly** — review utilization, cost trends, anomalies.
- **Quarterly** — strategic re-plan, architecture review.
- **Yearly** — multi-year storage and inflection planning.

Treat capacity planning as an ongoing engineering discipline, not a one-off spreadsheet.

---

## 16. Cheat Card

```
PURPOSE     Provision enough hardware to serve the expected load
            with acceptable latency and reliability, with headroom
            for spikes, failures, and growth.

FORMULA     required = (load × cost/unit) × headroom_factor
                       ───────────────────────────────────
                            utilization target

UTILIZATION
  CPU         60–70% healthy   >80% risky   >90% on fire
  Memory      70–80%
  Disk IOPS   60%
  Network     50–60%
  Queue       ≈0; anything queueing is a smell

N+1 / N+2   Plan for capacity loss, not just demand growth.
            Three-AZ system at 33% utilization may be correct.

INPUTS      Demand (peak, growth) · cost/unit · constraints
            (SLOs, budget) · headroom (spikes, failures, retries)

LOAD TESTS  Smoke · load · stress · soak · spike · chaos
            Watch for coordinated omission; warm up before measuring

HORIZONS    Daily ops  ·  Monthly trends  ·  Quarterly strategy
            Yearly storage & inflection points

PITFALLS    sizing for average · no headroom · >80% sustained ·
            no N+1 · ignoring retry amplification · happy-path-only
            tests · coordinated omission · forgetting storage
            compounds · no per-request cost data

RULE        Capacity planning is per-request cost × demand ×
            safety. You can't fix what you can't measure. Run cold
            enough that you survive the bad day.
```

---

## 17. Resources

### Books
- *Designing Data-Intensive Applications* — Martin Kleppmann.
- *Site Reliability Engineering* — Google. Chapters on demand forecasting and capacity planning.
- *The Practice of Cloud System Administration* — Limoncelli, Chalup, Hogan. Practical capacity planning chapter.
- *Systems Performance* — Brendan Gregg. Methodologies (USE, RED) that feed capacity planning.
- *High Performance Browser Networking* — Ilya Grigorik. Where the end-to-end latency budget actually goes.

### Articles
- "USE Method" — Brendan Gregg: <https://www.brendangregg.com/usemethod.html>
- "RED Method" — Tom Wilkie. Rate / Errors / Duration as service-level metrics.
- "Coordinated Omission" — Gil Tene. The bug in 90% of load testing.
- "Practical Capacity Planning" — Charity Majors / Honeycomb.
- "How We Plan Capacity at Stripe" — Stripe engineering posts.
- "How Netflix Scales for Peak" — Netflix tech blog.
- "Latency Numbers Every Programmer Should Know" — Jeff Dean's classic table.

### Videos
- Gil Tene — "How NOT to Measure Latency" (the coordinated omission talk).
- ByteByteGo — "Capacity Planning" overview.
- AWS re:Invent — "Right Sizing" and "Cost Optimization" tracks.
- USENIX SREcon — capacity planning talks every year.

### Tools
- **k6**, **Locust**, **wrk2**, **Gatling**, **Vegeta** — load generators.
- **HdrHistogram** — high-resolution latency histograms.
- **Prometheus + Grafana** — utilization dashboards.
- **AWS Trusted Advisor / Compute Optimizer**, **GCP Recommender**, **Azure Advisor** — right-sizing hints.
- **Kubecost / OpenCost** — Kubernetes cost attribution.
- **Datadog, Honeycomb, Lightstep** — per-request cost analysis.

### Adjacent reading
- [Back-of-the-Envelope Estimation](../01-foundations/estimation.md)
- [Latency Numbers Every Engineer Should Know](../01-foundations/latency-numbers.md)
- [Throughput vs Latency vs Bandwidth](../01-foundations/throughput-latency-bandwidth.md)
- [Auto-Scaling →](./auto-scaling.md)
- [Backpressure →](./backpressure.md)
- [SLA, SLO, SLI, Error Budgets](../11-reliability/sla-slo-sli.md)
- [Tail Latency & p99](../16-performance/tail-latency.md)
- [Compaction & Tiered Storage](../09-storage/compaction.md)

---

*Previous:* [← Hot Partition Problem](./hot-partitions.md)  |  *Next:* [Auto-Scaling (Horizontal Pod Autoscaler, AWS ASG) →](./auto-scaling.md)
