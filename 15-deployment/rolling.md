# Rolling Deployment

> **TL;DR** — A **rolling deployment** replaces instances of the old version with the new version a few at a time, keeping the service available throughout. It's the **default strategy** in Kubernetes, ECS, and most modern orchestrators because it gives you continuous availability at ~1× infrastructure cost. The trade-off: there is **always a mixed-version window** during the rollout, and **rollback is also a rolling operation** — not instant. Rolling works beautifully for stateless services with backward-compatible APIs and schemas; it gets dangerous when interactions between old and new versions matter. Two knobs control everything: **`maxSurge`** (how many extras can run during rollout) and **`maxUnavailable`** (how many can be missing). Tune those right and rolling is invisible to users. Get them wrong and you take an outage on every deploy.

---

## 1. The big picture

```
Step 0 (start)        v1 v1 v1 v1 v1 v1
Step 1 (maxSurge=1)   v1 v1 v1 v1 v1 v1 [v2]
Step 2                v1 v1 v1 v1 v1     [v2]  ← drained one v1
Step 3                v1 v1 v1 v1 v1 v2
Step 4                v1 v1 v1 v1 v1 v2 [v2]
Step 5                v1 v1 v1 v1     v2 [v2]
...
Final                                    v2 v2 v2 v2 v2 v2
```

The orchestrator brings up new pods, waits for them to pass readiness, then drains and terminates old pods. The loop continues until every instance is on the new version. Throughout, your aggregate capacity stays close to the target (controlled by `maxSurge` + `maxUnavailable`).

The defining property: **gradual, in-place replacement** with no second environment, no traffic switch — just slow churn.

---

## 2. Why rolling exists

Before rolling: you stopped the service, deployed, started the service. Outage on every deploy. Acceptable for nightly maintenance windows; unacceptable for the modern web.

Rolling lets you deploy in the middle of business hours with no scheduled downtime, at the same hardware cost as steady-state. That made it the dominant strategy for most apps once orchestrators (Kubernetes, ECS, Nomad, Mesos) made it the default behavior.

It's the **boring** strategy — which is precisely why it's right for most services. If you don't have a specific reason to do something else, you do a rolling deploy.

---

## 3. The two knobs: `maxSurge` and `maxUnavailable`

Kubernetes Deployment defaults:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 25%        # extra pods allowed above replica count
    maxUnavailable: 25%  # pods allowed to be missing
```

| Setting | Meaning | Effect |
|---|---|---|
| `maxSurge: 25%, maxUnavailable: 0` | Add new before removing old | Zero capacity dip, costs 1.25× briefly |
| `maxSurge: 0, maxUnavailable: 25%` | Remove old before adding new | No surge cost, 0.75× capacity briefly |
| `maxSurge: 100%, maxUnavailable: 0` | Spin up full new fleet, then drain old | Effectively blue-green via rolling |
| `maxSurge: 1, maxUnavailable: 0` | One-at-a-time, no capacity dip | Slow, safe, the boring-and-good default |

**Rules of thumb**:

- For latency-sensitive services where every replica counts: **`maxSurge=1` or `25%`, `maxUnavailable=0`**. You never drop below target capacity.
- For batch workers / queue consumers: **`maxSurge=0, maxUnavailable=25%`** is fine. Brief under-capacity is acceptable.
- For tiny replica counts (1–3): set both as absolute numbers, not percentages. `maxSurge=1, maxUnavailable=0` is usually right.
- Never set both to 0 — the rollout will deadlock.

---

## 4. The rollout loop, step by step

A simplified version of what the Kubernetes Deployment controller does:

```
loop:
  desired_new   = replicas - current_new
  if can_surge_more:
      create new pod
  for each new pod with passed readiness:
      add to service endpoints
  for each old pod with replacement ready:
      remove from service endpoints
      send SIGTERM
      wait terminationGracePeriodSeconds
      send SIGKILL if still alive
  if current_new == replicas and current_old == 0:
      done
```

The two critical kernel-level events on a rolling deploy:

1. **Readiness probe passes** on a new pod → traffic starts flowing.
2. **Pre-stop hook + SIGTERM** on an old pod → it must finish in-flight work and exit cleanly within `terminationGracePeriodSeconds` (default 30s).

If your app ignores SIGTERM, you'll get SIGKILL'd in the middle of requests. That's where most "rolling deploy causes 5xx spike" stories come from.

---

## 5. Graceful shutdown — the part everyone gets wrong

A rolling deploy without graceful shutdown is silently terminating connections in flight. Symptoms: error-rate blips, mysterious p99 spikes, customer reports of "I refreshed and it worked."

The handshake you need on every server:

```
1. preStop hook (Kubernetes) or shutdown signal
2. Remove self from LB/service endpoints (typically 1–5s wait)
3. Stop accepting new connections
4. Drain: finish in-flight requests
5. Close DB connections, flush logs
6. Exit cleanly within terminationGracePeriodSeconds
```

In Kubernetes:

```yaml
spec:
  terminationGracePeriodSeconds: 60
  containers:
    - name: api
      lifecycle:
        preStop:
          exec:
            command: ["sleep", "10"]  # delay so LB removes us before SIGTERM
      # Your app must handle SIGTERM → graceful shutdown
```

In Go: `signal.Notify(c, syscall.SIGTERM)` and call `server.Shutdown(ctx)`.
In Node: handle `process.on('SIGTERM', ...)` and call `server.close()`.
In Python (gunicorn/uvicorn): use `--graceful-timeout`.
In Java (Spring Boot): `server.shutdown=graceful`.

**Why the preStop sleep?** When a pod is terminating, two things happen in parallel: the Service endpoints controller removes the pod from rotation, *and* kubelet sends SIGTERM. The endpoint removal takes a few seconds to propagate through kube-proxy on every node. If SIGTERM races ahead, the LB still sends new traffic to a dying pod. Sleeping in preStop bridges that gap.

---

## 6. Probes are load-bearing

Rolling deploys depend on readiness probes to know when a new pod can take traffic. If readiness is wrong:

- **Too eager (returns 200 before app is ready)** → traffic hits a cold app → errors during every deploy.
- **Too strict (probes deeper than necessary)** → readiness flaps under load → pods cycle in and out of the LB.

Good readiness checks:

- HTTP GET to a `/ready` endpoint that returns 200 only when the app is actually ready to serve.
- Includes: bound port, completed DB pool warm-up, completed cache warm-up, schema migrations done (or known compatible).
- Does **not** include: liveness checks of every downstream (cascading failures).

See [Health Checks & Heartbeats →](../13-observability/health-checks.md).

---

## 7. The mixed-version window

The defining trade-off of rolling: **for the duration of the deploy, old and new versions are running side by side**. A request can hit either. So can a background job, a queue consumer, or a database write.

The compatibility burden:

- **API requests** — old clients must work against the new server; new clients must work against the old server. The mesh / LB doesn't pin a user to a version.
- **Database schema** — must be readable and writable by both versions. Use expand-contract. See [Database Migrations at Scale →](../04-databases/migrations.md).
- **Queue messages** — both versions consume from the same queue. The message format must be backward compatible.
- **Inter-service contracts** — if service A v2 calls service B with a new field, B must already accept it.
- **Caches** — same key, different value shape → poisoned cache. Version your keys.

A useful heuristic: **whatever the contract is at the start of the deploy, both versions must honor it until the deploy completes.** Breaking changes always require a multi-deploy plan: add the new path, deploy, switch callers, remove the old path.

---

## 8. Rollback — slower than you think

Rolling forward is gradual. Rolling back is also gradual — by replacing v2 instances with v1 instances at the same rate. If your rollout took 10 minutes, your rollback takes another 10. During that window, you're still partly on the broken version.

Kubernetes:

```bash
kubectl rollout undo deployment/api
# or:
kubectl rollout undo deployment/api --to-revision=42
```

This is the most common surprise for teams coming from blue-green: rolling isn't instant rollback. If you need instant rollback as a property of your strategy, choose **blue-green** or **canary with manual abort to 0%**. See [Blue-Green Deployment →](./blue-green.md), [Canary Deployment →](./canary.md).

Rolling rollback also re-pulls the old image and re-warms it — if your nodes have evicted the v1 image, you'll wait on registry pulls during the rollback. This is a real failure mode in disaster scenarios.

---

## 9. Worked example — a Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: {name: api}
spec:
  replicas: 12
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 2          # up to 14 pods total
      maxUnavailable: 0    # never below 12
  minReadySeconds: 20      # new pod must stay ready 20s before counting
  revisionHistoryLimit: 10
  selector: {matchLabels: {app: api}}
  template:
    metadata: {labels: {app: api}}
    spec:
      terminationGracePeriodSeconds: 60
      containers:
        - name: api
          image: ghcr.io/acme/api@sha256:abc...
          ports: [{containerPort: 8080}]
          readinessProbe:
            httpGet: {path: /ready, port: 8080}
            periodSeconds: 5
            failureThreshold: 3
          lifecycle:
            preStop:
              exec:
                command: ["sh", "-c", "sleep 10 && kill -TERM 1"]
          resources:
            requests: {cpu: 200m, memory: 256Mi}
            limits:   {cpu: 1000m, memory: 512Mi}
```

What this gives you:
- Aggregate capacity never dips below 12.
- Up to 2 new pods come up at a time.
- A new pod must pass readiness *and* stay ready for 20s before counting toward progress. Catches "ready, then crash 5s later" bugs.
- 60s grace period for in-flight requests to complete.
- preStop sleep gives endpoints time to propagate before SIGTERM.

For a busy production service, this is the boring-and-correct config.

---

## 10. Rolling beyond Kubernetes

The same model exists everywhere:

| Platform | Concept |
|---|---|
| **AWS ECS** | "Minimum healthy percent" + "maximum percent" — same idea as `maxUnavailable` / `maxSurge`. |
| **AWS Auto Scaling Group + CodeDeploy** | "Rolling update" deployment config with batch size. |
| **AWS Elastic Beanstalk** | "Rolling" or "Rolling with additional batch" — adds surge capacity. |
| **GCP Managed Instance Group** | `--max-surge` + `--max-unavailable`. |
| **HashiCorp Nomad** | `update.max_parallel` + `update.canary` + `update.min_healthy_time`. |
| **HashiCorp Consul / Terraform** | Resource lifecycle `create_before_destroy`. |

The terminology shifts, the mechanics don't.

---

## 11. When rolling wins / when it loses

### Wins

- **Default for most services.** Stateless, backward-compatible, mixed-version-safe — rolling is the lowest-overhead choice.
- **Latency-sensitive services where you can't afford a capacity dip.** `maxSurge>0, maxUnavailable=0` gives you continuous coverage.
- **Cost-sensitive deploys.** No 2× environment like blue-green.
- **Many small services with frequent deploys.** Rolling fits the cadence.

### Loses

- **Risky binary changes where mixed versions are unsafe.** Use blue-green for atomic cutover.
- **When you need real-traffic signal at a small slice before committing.** Use canary.
- **When your app can't be made backward compatible.** You need a different strategy or a maintenance window.
- **When startup is slow.** A 5-minute warm-up per pod across 100 pods is hours of rollout — consider blue-green or pre-warmed canary instead.

---

## 12. Operational checklist

```
PRE-DEPLOY
[ ] App handles SIGTERM and drains cleanly
[ ] preStop hook with delay (5–10s typical)
[ ] terminationGracePeriodSeconds longer than your slowest request
[ ] Readiness probe checks "ready," not "alive"
[ ] minReadySeconds set high enough to catch fast-failures
[ ] Schema migration is expand-only (no destructive DDL)
[ ] API/queue/cache changes are backward compatible
[ ] maxSurge / maxUnavailable tuned for the workload

DURING DEPLOY
[ ] Watch error rate, p99 latency in real time
[ ] Watch new pods passing readiness — flapping = bad sign
[ ] Watch downstream dependency health (DB, cache, queues)
[ ] No surge in connection resets / 502/504s

POST-DEPLOY
[ ] All replicas on new image
[ ] No leftover old pods (occasionally a Deployment leaks orphans)
[ ] revisionHistory preserved for rollback
[ ] Contract migrations (if any) only after stability window

ROLLBACK
[ ] kubectl rollout undo (or equivalent) — knows it's gradual
[ ] If urgent: scale down then redeploy v1 image to cut time
[ ] Don't run contract migrations until next attempt succeeds
```

---

## 13. Common Mistakes / Anti-Patterns

- **No graceful shutdown.** Connections get reset mid-flight on every deploy.
- **No preStop sleep.** SIGTERM races endpoint removal → traffic to a dying pod.
- **terminationGracePeriodSeconds too short.** Long requests get SIGKILL'd.
- **Readiness probe that returns 200 before the app is actually ready.** Cold pods take traffic.
- **`maxUnavailable: 25%` on a 3-replica service.** That's losing 1 of 3 → 33% capacity drop.
- **Treating rolling as instant rollback.** It's gradual. Plan for that.
- **Schema-breaking migrations during a rolling deploy.** Half the pods see broken schema for minutes.
- **API breaking changes deployed in one shot.** Add new, deploy, switch callers, deploy, remove old — never in one PR.
- **No `minReadySeconds`.** A pod passes readiness, then crashes 3 seconds later — rollout keeps going, capacity collapses.
- **Image not pulled to all nodes.** First-time deploy stalls on registry timeouts. Pre-pull or warm with a DaemonSet.
- **Both `maxSurge` and `maxUnavailable` set to 0.** Rollout deadlocks.
- **Treating background workers and queue consumers like web servers.** They have different drain semantics; in-flight messages need to finish or be requeued.

---

## 14. Cheat Card

```
PURPOSE   Replace old instances with new, a few at a time,
          keeping the service available throughout.

KNOBS
  maxSurge        = extra pods allowed during rollout
  maxUnavailable  = pods allowed to be missing
  Surge>0, Unavail=0   →  never drop capacity (latency apps)
  Surge=0, Unavail>0   →  no surge cost (batch / async)

REQUIRED HABITS
  SIGTERM-aware app + graceful drain
  preStop hook with short sleep
  terminationGracePeriodSeconds ≥ slowest request
  Readiness probe = "really ready"
  minReadySeconds ≥ 10s (catch fast-failure pods)
  Mixed-version compatibility on API/DB/cache/queue

WHEN TO USE
  Default for most stateless services
  Continuous availability at ~1× cost
  Frequent deploys, microservices

WHEN NOT TO USE
  Atomic cutover required (use blue-green)
  Real-traffic signal at 1% needed first (use canary)
  Mixed versions are unsafe by design

ROLLBACK
  kubectl rollout undo — gradual, not instant
  Plan for the rollback to take ~as long as the rollout
  Instant rollback requires blue-green or canary

PITFALLS
  No graceful shutdown → 5xx blips on every deploy
  No preStop sleep → traffic to dying pods
  Surge+Unavail both 0 → rollout deadlock
  Schema-breaking migration during rollout → half pods broken
  Probe returns 200 before app is ready → cold pods take traffic

RULE   Make every deploy survive mixed versions and a kill -TERM
       arriving mid-request. Then you can deploy any time.
```

---

## 15. Resources

### Documentation
- **Kubernetes** — Rolling updates: <https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-update-deployment>
- **AWS ECS** — Deployment configurations: <https://docs.aws.amazon.com/AmazonECS/latest/developerguide/deployment-type-ecs.html>
- **GCP MIG** — Rolling updates: <https://cloud.google.com/compute/docs/instance-groups/rolling-out-updates-to-managed-instance-groups>
- **HashiCorp Nomad** — Update stanza: <https://developer.hashicorp.com/nomad/docs/job-specification/update>

### Articles
- "Kubernetes deployments: rolling updates" — Kubernetes blog.
- "Graceful shutdown in Kubernetes" — Learnk8s: <https://learnk8s.io/graceful-shutdown>
- "How Stripe ships" — Stripe engineering.
- "Continuous Delivery" — Jez Humble & David Farley (book reference).

### Videos
- *Zero-downtime deploys in Kubernetes* — Learnk8s.
- ByteByteGo — "Rolling Deployment Explained."
- KubeCon — talks on graceful shutdown patterns.

### Tools
- **kubectl rollout status / undo** — built-in.
- **Argo Rollouts** — adds canary/blue-green strategies on top of rolling.
- **Helm** — packages rolling-friendly deploys.
- **GoReplay / k6** — load-test the deploy itself.

### Adjacent reading
- [Blue-Green Deployment →](./blue-green.md)
- [Canary Deployment →](./canary.md)
- [Feature Flags & Dark Launches →](./feature-flags.md)
- [Container Orchestration (Kubernetes) →](./kubernetes.md)
- [CI/CD Pipelines →](./cicd.md)
- [Health Checks & Heartbeats →](../13-observability/health-checks.md)
- [Database Migrations at Scale →](../04-databases/migrations.md)
- [Graceful Degradation →](../11-reliability/graceful-degradation.md)

---

*Previous:* [← Canary Deployment](./canary.md)  |  *Next:* [Feature Flags & Dark Launches →](./feature-flags.md)
