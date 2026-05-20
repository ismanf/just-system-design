# Auto-Scaling (Horizontal Pod Autoscaler, AWS ASG)

> **TL;DR** — **Auto-scaling** is the practice of adding and removing capacity automatically in response to load signals. **Horizontal scaling** adds/removes whole instances (or pods, containers, VMs); **vertical scaling** resizes one instance; **autoscaling** usually refers to the horizontal kind. Done right, it absorbs traffic spikes, follows diurnal patterns, and saves real money during quiet hours. Done wrong, it amplifies failures, oscillates, or fails to scale up in time for the very spike you needed it for. The two production families are **Kubernetes HPA / VPA / Cluster Autoscaler** (and its cousins KEDA, Karpenter) and **cloud-native groups** (AWS ASG, GCP MIG, Azure VMSS). Both rely on the same fundamentals: signal → policy → action → feedback, with carefully tuned thresholds, cooldowns, and warmup behavior. Most autoscaling failures aren't about the algorithm — they're about scaling **fast enough**, **stably enough**, and on the **right signal**.

---

## 1. What Autoscaling Is and Isn't

```
                      ┌────────────────────┐
   Load signal ──────►│   Autoscaler       │
   (CPU, RPS,         │  (controller loop) │
    queue depth,      │                    │
    custom metric)    └──────────┬─────────┘
                                 │ adds / removes
                                 ▼
                          ┌─────────────┐
                          │  Fleet      │
                          │  (pods,     │
                          │   VMs)      │
                          └──────┬──────┘
                                 │ effect on load → feedback
                                 ▼
                          Next sample
```

Autoscaling **is**:
- Adjusting the number (or size) of workers to match load.
- A feedback control loop with the load signal as input and the fleet size as output.
- A tool to optimize cost-vs-capacity and handle spikes.

Autoscaling **isn't**:
- A substitute for capacity planning. You still need to know your worst-case and your scaling limits.
- A way to scale stateful systems trivially. Databases, caches, brokers all need careful design before they autoscale.
- A way to fix a too-slow scale-up — many spikes happen faster than the autoscaler reacts.

The questions to ask before adopting autoscaling:
1. **What is the right signal** to scale on?
2. **How fast can we add capacity** in the spike window we care about?
3. **What happens during scale-down** — do we drop work?
4. **What's our maximum scale**, and can our downstream survive it?

---

## 2. Horizontal vs Vertical vs Reactive vs Predictive

```
                Horizontal          Vertical
              ─────────────       ────────────
                                                    
Reactive   ▶ HPA, ASG             VPA
            (most common)         (less common)
                                                    
Predictive ▶ AWS Predictive       (rare)
              Scaling, GCP
              forecasting
```

- **Horizontal**: more (or fewer) identical instances. Stateless workloads love this.
- **Vertical**: resize one instance up or down. Useful for pods with awkward right-sizing.
- **Reactive**: scale based on observed metrics over the last few minutes.
- **Predictive**: scale based on a forecast of future load (often using historical patterns).

For most apps, **horizontal + reactive** is the default. Predictive helps when reactive scale-up is too slow for known patterns (morning traffic ramp, ad campaign launch).

---

## 3. Kubernetes Autoscaling — The Three (or Four) Layers

Kubernetes splits the problem into layers:

```
   ┌─────────────────────────────────────────────────┐
   │ Cluster Autoscaler / Karpenter                  │  nodes
   │   adds / removes VMs in the underlying cluster  │
   ├─────────────────────────────────────────────────┤
   │ Vertical Pod Autoscaler (VPA)                   │  pod size
   │   recommends / applies pod CPU/memory requests  │
   ├─────────────────────────────────────────────────┤
   │ Horizontal Pod Autoscaler (HPA)                 │  pod count
   │   adds / removes pod replicas based on metrics  │
   ├─────────────────────────────────────────────────┤
   │ KEDA (event-driven autoscaler)                  │  pod count
   │   adds / removes pods based on external events  │
   │   (queue depth, Kafka lag, custom metrics)      │
   └─────────────────────────────────────────────────┘
```

### Horizontal Pod Autoscaler (HPA)
The standard mechanism. Watches a metric (CPU, memory, custom) and adjusts deployment replicas.

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-server
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  minReplicas: 3
  maxReplicas: 50
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
  behavior:
    scaleUp:
      policies:
        - type: Percent
          value: 100
          periodSeconds: 60
        - type: Pods
          value: 5
          periodSeconds: 60
      selectPolicy: Max
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
```

Notable knobs:
- `minReplicas` / `maxReplicas` — floor and ceiling.
- `averageUtilization` — the setpoint. 60% is a typical CPU target.
- `behavior.scaleUp/scaleDown` — rate limits, stabilization windows.
- `stabilizationWindowSeconds` — the "wait this long before reacting" buffer that prevents flapping.

The HPA defaults are aggressive on scale-down. Production deployments usually slow scale-down to avoid removing capacity right before a spike.

### Vertical Pod Autoscaler (VPA)
Adjusts CPU/memory **requests** of pods rather than count. Useful for batch workloads where over-requesting wastes a lot of cluster capacity. Trade-off: VPA usually requires pod restarts to apply.

### Cluster Autoscaler
When HPA wants more pods but nodes are full, the Cluster Autoscaler adds nodes. When pods drain off and nodes are under-utilized, it removes them. Two flavors:
- **Cluster Autoscaler** — works against node groups (ASGs, MIGs, VMSSes). Mature, stable.
- **Karpenter** — newer (AWS-born, now open source). Provisions individual instances of the right size/type for pending pods, without rigid node groups. Faster, more flexible, more efficient. Becoming the AWS default.

### KEDA
The Kubernetes Event-Driven Autoscaling project. Scales based on external events:
- Kafka consumer lag
- SQS queue depth
- Redis stream length
- Prometheus query
- Postgres / MySQL queries
- 60+ built-in scalers

KEDA can scale a deployment to zero — useful for event-driven workloads that idle most of the time. (Plain HPA can't go below 1.)

---

## 4. AWS Auto Scaling Groups (and Cousins)

The pre-Kubernetes-era machinery, still widely used and integrated everywhere:

### AWS Auto Scaling Group (ASG)
Manages a fleet of EC2 instances:
- **Desired / min / max** capacity.
- **Launch template / configuration** — defines the AMI and instance shape.
- **Scaling policies** — target tracking, step scaling, simple scaling, scheduled.
- **Lifecycle hooks** — pre-warming, draining, custom in/out behavior.
- **Mixed instance policies** — combine on-demand + spot, multiple instance types.
- **Health checks** — EC2 + ELB; unhealthy instances replaced automatically.

```
   ┌─────────────────┐
   │   CloudWatch    │ metrics
   └────────┬────────┘
            │
            ▼
   ┌─────────────────┐
   │  ASG Scaling    │
   │    Policy       │  target: avg CPU = 60%
   └────────┬────────┘
            │
            ▼
   ┌─────────────────┐
   │   ASG           │
   │  desired = 12   │  adjusts in/out
   └────────┬────────┘
            │
            ▼
   ┌──────┬──────┬──────┬─────┐
   │ EC2  │ EC2  │ EC2  │ ... │
   └──────┴──────┴──────┴─────┘
```

### GCP Managed Instance Group (MIG)
Same idea on GCP. Autohealing, regional or zonal, integrated with load balancers.

### Azure Virtual Machine Scale Sets (VMSS)
Azure's equivalent. Autoscale rules can be based on metrics or schedules.

### AWS Application Auto Scaling
A meta-service that scales other AWS resources: ECS services, DynamoDB, Aurora replicas, SageMaker endpoints, etc. Single API for "scale anything against a CloudWatch metric."

---

## 5. Scaling Signals — What to Scale On

The hardest decision is **what signal drives scaling**. Choices have radically different behaviors.

### CPU utilization
- **Pros**: universal, easy, free.
- **Cons**: lags real load; doesn't reflect IO-bound or network-bound work; idle services don't scale down.
- **Use when**: workload is CPU-bound and predictable.

### Memory utilization
- **Pros**: catches memory-bound workloads.
- **Cons**: memory rarely a good scaling signal — it accumulates; doesn't drop with load.
- **Use when**: memory growth predicts overload (caches, ML serving).

### Requests per second (RPS) per pod
- **Pros**: directly reflects user-facing load.
- **Cons**: requires custom metrics + Prometheus Adapter or similar.
- **Use when**: latency goal correlates with per-pod traffic.

### Queue depth / consumer lag
- **Pros**: directly measures work-to-do; perfect for async/batch workers.
- **Cons**: requires the metric to be reliably exported.
- **Use when**: workers consume from SQS, Kafka, RabbitMQ, etc. **The gold standard for event-driven workloads.**

### Concurrent connections / in-flight requests
- **Pros**: matches latency budgets directly.
- **Cons**: requires per-instance instrumentation.
- **Use when**: long-lived connections (WebSocket, SSE) or slow downstream.

### Custom business metrics
- **Pros**: scales on what actually matters (orders/sec, signups/sec).
- **Cons**: requires custom-metrics pipeline.
- **Use when**: you have a clear business signal that predicts capacity need.

### Predictive (forecast-based)
- **Pros**: scales **ahead of** the spike.
- **Cons**: only as good as the forecast; struggles with novel patterns.
- **Use when**: load patterns are highly periodic and reactive scaling is too slow.

The general rule: **scale on the metric closest to the bottleneck**. If your bottleneck is DB connection pool exhaustion, scale on that. If it's CPU, scale on CPU. If it's queue depth, scale on queue depth.

---

## 6. The Scale-Up Speed Problem

The most common autoscaling failure isn't oscillation — it's **scaling up too slowly**.

```
  RPS
   ▲                       ⎯ actual load (spike)
   │                  ┌──┐
   │                  │  │
   │                  │  │
   │                  │  │
   │                  │  └─────⎯ capacity (scale lags)
   │                  │
   │           ┌──────┘
   │      ┌────┘
   │      │
   │  ────┘
   └──────────────────────────────► time
                   spike

   capacity catches up after the spike ends — too late.
```

Time to add capacity is at least:

```
  detection delay (1–5 min)
+ decision delay (cooldown, evaluation window)
+ instance provision time (cloud: 30 s – 5 min)
+ container pull + start (30 s – 5 min)
+ application warmup (cache fill, JIT, connections — seconds to minutes)
= 2 – 15 minutes typically
```

Spikes shorter than this window won't be served by autoscaling. Mitigations:

- **Pre-warmed warm pools** — keep N instances ready (AWS Warm Pools, GKE node pool pre-provisioning).
- **Karpenter** — much faster than the cluster autoscaler for unscheduled pods.
- **Predictive scaling** for known traffic shapes.
- **Buffering with queues** — absorb the spike, drain at sustainable rate.
- **Pre-scale before known events** — manual or scheduled scaling.
- **Headroom over autoscaling minimum** — don't run at the wire; leave 20–30%.
- **Surge capacity from spot/preemptible** — cheap when burning, free when idle.

For sub-minute spikes, autoscaling will never be fast enough. **You need over-provisioning or queueing as the primary line of defense.**

---

## 7. Scale-Down Hazards

Scale-down isn't free either:

- **Dropping in-flight requests** — instance terminated while serving a request. Use graceful shutdown (SIGTERM → drain → finish → exit).
- **Cache cold-start on remaining nodes** — surviving nodes get extra load on a cold cache.
- **Connection-rebalance churn** — clients reconnect, downstream sees connection spike.
- **DB connection pool sizing** — total pool size scales with replica count; downsizing instances may free DB capacity, or may starve survivors.
- **Removing the wrong instance** — defaults pick by oldest or random, which may evict the warmest cache.
- **Premature scale-down before a spike** — common around predictable daily patterns.

Good scale-down hygiene:
- **Stabilization window** — wait long after load drops before removing pods.
- **Graceful shutdown** — `terminationGracePeriodSeconds` in K8s; ELB drain on AWS.
- **Pre-stop hooks** — final cleanup, deregistration.
- **Slow scale-down rate** — remove at most X% per minute.
- **Pod Disruption Budgets** in K8s — protect a minimum number of healthy pods.

The asymmetry: **scale up fast, scale down slow.** Mistakes scaling up cost money; mistakes scaling down cost SLOs.

---

## 8. Predictive Scaling

Reactive scaling reacts. Predictive scaling forecasts.

### How it works
- Take historical metrics (often 14+ days).
- Fit a model — seasonal ARIMA, Prophet, neural nets, or simple periodic averages.
- Forecast future load.
- Pre-scale capacity to match the forecast, ahead of the actual spike.

### When it helps
- **Sharp morning ramp** — load 5× in 10 minutes at 8 AM every weekday. Reactive lags; predictive nails it.
- **Marketing-driven spikes** — known launch times.
- **Diurnal cycles with predictable peaks.**

### When it doesn't
- **Novel spikes** — viral content, news events, ad campaigns the forecaster didn't see.
- **Highly variable workloads** without strong periodicity.
- **Very short horizons** — predictive needs minutes to scale; sub-minute spikes still buffer/queue.

Combine: predictive sets the **baseline** for known patterns; reactive handles deviation.

AWS Predictive Scaling, GCP Recommender, and various open-source projects (Goldilocks, custom forecasters) implement this.

---

## 9. Stateful Workloads

Autoscaling shines for stateless work. Stateful is harder.

### Databases
- **Read replicas** can autoscale somewhat (Aurora supports Reader endpoint autoscaling).
- **Write capacity** doesn't autoscale horizontally — sharding required.
- **DynamoDB** has on-demand and provisioned-with-autoscaling modes. Provisioned + auto handles sustained growth; on-demand absorbs spikes.
- **Aurora Serverless v2** scales DB capacity (ACUs) in real time. Cool, but not a panacea — large changes still take time, and certain workloads fight it.

### Caches
- Adding a Redis node requires resharding; not trivial.
- ElastiCache cluster mode supports scaling, with reshard latency.
- Modern serverless cache services (DynamoDB Accelerator, MemoryDB) handle some of this.

### Message brokers
- Kafka partition count is the cap on consumer parallelism. Add partitions to add scale, but rebalancing has cost.
- Consumers can autoscale within partition cap. KEDA + Kafka lag is the canonical pattern.
- Brokers themselves rarely autoscale — capacity-plan them.

### ML serving / inference
- Stateless if the model loads at startup; warmup time is the big cost.
- KEDA + custom metrics (request queue depth, GPU utilization) work well.

The rule: **stateful workloads need explicit, careful autoscaling** or none at all.

---

## 10. Cost Optimization with Autoscaling

The whole financial case for autoscaling:

| Pattern | Cost shape |
|---|---|
| Static, peak-sized | flat, expensive |
| Manual scaling | step-wise, error-prone |
| Reactive autoscaling | follows load, ~30–60% savings |
| Predictive + reactive | optimal for periodic workloads |
| Mixed on-demand + spot | 50–70% savings on the burst tier |
| Scale-to-zero (KEDA) | near-zero idle cost |

Levers to combine with autoscaling:
- **Spot / preemptible** for burst capacity (50–90% off, accept termination risk).
- **Reserved / Savings Plans** for the baseline floor (40–70% off, committed).
- **Right-sizing** — smaller instances for the autoscaling tier.
- **Scheduled scaling** — drop to a smaller floor overnight.
- **Per-namespace / per-tenant scaling** for multi-tenant K8s.

A well-tuned autoscaling stack with mixed on-demand + spot + savings plans routinely cuts cloud spend 40–60% versus static peak provisioning.

---

## 11. Operational Pitfalls

### Oscillation / flapping
The autoscaler adds pods, load drops below threshold, removes them, load spikes, adds again. Causes:
- Too-tight scaling thresholds.
- Too-short stabilization windows.
- Choosing a noisy metric.

Fixes:
- Hysteresis: different thresholds for scale-up vs scale-down.
- Longer evaluation windows.
- Smoother metrics (rolling average, p50).

### Retry amplification → false load
A slow downstream → retries → CPU goes up → autoscaler adds pods → more retries hit the same downstream → cascading. The scaling signal is fake; the right answer is rate limiting and circuit breakers, not more capacity.

### Autoscaling against your own tail
Scale-up on p99 latency that's caused by GC pauses or hot keys → more pods → more memory → more GC pressure → more pods. Identify the root cause; autoscaling can't fix it.

### Resource quota exhaustion
Autoscaling adds pods until the cloud account hits a subnet IP, IAM role, instance quota, or EBS quota limit. Plan quotas + monitor headroom.

### Downstream can't keep up
You scale to 200 pods; the downstream database has 100 connections. Your fleet is fine; the DB is on fire. Connection pooling + DB-side autoscaling + circuit breakers required.

### Scaling on the wrong metric
"Scaling on CPU" when the actual bottleneck is IO or downstream latency. The signal must reflect the bottleneck.

### Slow start / long warmup
Pods take 60 s to be ready (JIT, cache warmup, model load). HPA scales but they don't serve traffic in time. Mitigations: warm pools, init containers, faster startup, traffic ramping.

### Forgetting min/max sanity
`minReplicas: 1` with no over-provisioning means a single failure leaves zero. `maxReplicas: ∞` means a runaway scaling event empties your credit card.

### Ignoring graceful shutdown
Pods killed mid-request → user errors. Always implement graceful shutdown with appropriate grace periods.

### Tested only in steady state
Load tests at constant RPS don't catch scale-up lag. Add spike tests.

---

## 12. The Autoscaling Recipe

A production-ready autoscaling config typically has:

```
✓ Right signal (queue depth or RPS for most, CPU as fallback)
✓ Target utilization 60–70% (not 80+)
✓ minReplicas with N+1 / N+2 headroom
✓ maxReplicas safely bounded
✓ scaleUp fast (no stabilization, large step)
✓ scaleDown slow (5–10 min stabilization, small step)
✓ Pod readiness probes + warmup time accounted for
✓ Graceful shutdown with adequate terminationGracePeriod
✓ PDB to prevent disruption-driven outage
✓ Cluster autoscaler / Karpenter for the node side
✓ Quotas monitored
✓ Downstream capacity scaling in lockstep
✓ Load tested with spike + soak scenarios
```

Skip any of those and you'll hit one of the pitfalls above.

---

## 13. Common Mistakes / Anti-Patterns

- **Scaling on CPU when the bottleneck is elsewhere.** IO-bound or DB-bound apps need different signals.
- **Aggressive scale-down.** Removes capacity right before a spike. Slow it down.
- **No graceful shutdown.** Killed pods drop in-flight work.
- **`minReplicas: 1`.** Zero capacity for a moment is too much risk.
- **Unbounded `maxReplicas`.** Runaway scaling.
- **Autoscaling against retry amplification.** Treat the underlying slowdown, not the symptom.
- **Forgetting downstream capacity.** Fleet scales; DB doesn't.
- **No spike testing.** Steady-state tests miss the scale-up lag.
- **Scaling stateful workloads naively.** Resharding caches and brokers under load.
- **No quota monitoring.** Hit the cloud limit at the worst time.
- **Same scaling policy for very different workloads.** Stateless web ≠ batch worker ≠ ML inference.
- **Ignoring cold-start latency.** First request to a new pod is 10× slower; users notice.
- **Scaling on memory in JVM-heavy apps.** GC-driven oscillation.
- **Using only reactive scaling for known patterns.** Predictive (or scheduled) catches the morning ramp.

---

## 14. Cheat Card

```
PURPOSE     Automatically size the fleet to match load. Optimize
            cost vs capacity. Absorb spikes. Follow daily curves.

DIMENSIONS  Horizontal (count) vs Vertical (size)
            Reactive vs Predictive

K8S LAYERS  HPA          pod count by metric
            VPA          pod size by usage
            Cluster      nodes by pending pods
              Karpenter  faster, instance-aware
            KEDA         event-driven, scale to zero

CLOUD       AWS ASG / GCP MIG / Azure VMSS
            App Auto Scaling for managed services

SIGNALS     CPU · memory · RPS · queue depth · custom · forecast
            Pick the one closest to the bottleneck.

SCALE UP    Fast. No stabilization. Large step. Pre-warm if possible.
SCALE DOWN  Slow. Long stabilization. Small step. Graceful shutdown.

LAG         Detect (1–5m) + decide + provision + start + warm
            = 2–15 min typically. Plan over-provision or queue
            for spikes faster than this.

PITFALLS    wrong signal · oscillation · retry amplification ·
            downstream bottleneck · slow warmup · runaway max ·
            graceful-shutdown missing · no spike testing ·
            scaling stateful workloads naively

RULE        Autoscaling is a feedback control loop. Right signal,
            right rate, right limits, right downstream. It's
            cost optimization layered on capacity planning,
            never a replacement for it.
```

---

## 15. Resources

### Documentation
- **Kubernetes HPA** — <https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/>
- **KEDA** — <https://keda.sh/>
- **Karpenter** — <https://karpenter.sh/>
- **AWS Auto Scaling** — <https://docs.aws.amazon.com/autoscaling/>
- **GCP MIG Autoscaling** — <https://cloud.google.com/compute/docs/autoscaler>
- **Azure VMSS** — <https://learn.microsoft.com/azure/virtual-machine-scale-sets/>

### Books
- *Site Reliability Engineering* — Google. Chapters on autoscaling and capacity.
- *Production Kubernetes* — Josh Rosso et al. Autoscaling chapter.
- *Designing Data-Intensive Applications* — Martin Kleppmann. Backpressure and scaling intersect.

### Articles
- "AWS Predictive Scaling" — AWS engineering blog.
- "Karpenter: Just-in-time Nodes for any Kubernetes Cluster" — AWS.
- "Scaling Kubernetes with KEDA" — KEDA project blog.
- "Auto-scaling Pinterest Ads Infrastructure" — Pinterest engineering.
- "How Discord Auto-scales Voice Servers" — Discord engineering.
- "Scaling Hudl's Microservices" — Hudl engineering on KEDA-style scaling.

### Videos
- ByteByteGo — "Auto-scaling Explained."
- KubeCon talks on HPA / KEDA / Karpenter.
- AWS re:Invent — Auto Scaling and Spot deep-dives.
- Google Cloud Next — MIG and autoscaling talks.

### Tools
- **k6 / Locust** — for spike test scenarios.
- **kube-burner** — load test Kubernetes itself.
- **Goldilocks** — VPA recommendations UI.
- **KEDA** — event-driven scaling.
- **Karpenter** — fast node provisioner for K8s on AWS.
- **AWS Compute Optimizer / GCP Recommender / Azure Advisor** — right-sizing hints.
- **Kubecost / OpenCost** — attribute autoscaling cost to workloads.

### Adjacent reading
- [Capacity Planning](./capacity-planning.md)
- [Backpressure →](./backpressure.md)
- [Scaling Reads vs Scaling Writes](./reads-vs-writes.md)
- [Hot Partition Problem](./hot-partitions.md)
- [Circuit Breaker Pattern](../11-reliability/circuit-breaker.md)
- [Retry, Timeout, and Exponential Backoff](../11-reliability/retry-timeout-backoff.md)
- [Container Orchestration (Kubernetes)](../15-deployment/kubernetes.md)
- [Tail Latency & p99](../16-performance/tail-latency.md)

---

*Previous:* [← Capacity Planning](./capacity-planning.md)  |  *Next:* [Backpressure →](./backpressure.md)
