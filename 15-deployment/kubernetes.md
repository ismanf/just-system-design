# Container Orchestration (Kubernetes)

> **TL;DR** — **Kubernetes** is a control loop on top of a cluster of machines that runs your containers, restarts them when they crash, schedules them onto nodes with capacity, gives them DNS and IPs, exposes them through services, mounts their storage, manages their secrets, and rolls them out without downtime. Its core idea is **declarative state**: you describe the desired state in YAML, **controllers** continuously reconcile actual state toward it. That model is what makes K8s self-healing — and what makes it operationally enormous. The honest take: **Kubernetes is the right answer for most teams above ~20 services or ~50 engineers, and overkill below that**. For smaller teams, ECS, Nomad, Fly, or Cloud Run will deliver value faster with one tenth of the operational surface. Once committed, treat K8s like a platform team responsibility, not "DevOps for the app team."

---

## 1. The big picture

Kubernetes is a distributed system whose job is to make sure your containers are running where they should be, in the configuration you said they should be in.

```
┌────────────────────────── CONTROL PLANE ──────────────────────────┐
│                                                                    │
│  ┌──────────┐  ┌────────────┐  ┌──────────────┐  ┌──────────────┐ │
│  │ API      │  │ Scheduler  │  │ Controller   │  │ etcd         │ │
│  │ Server   │  │            │  │ Manager      │  │ (state)      │ │
│  └──────────┘  └────────────┘  └──────────────┘  └──────────────┘ │
│                                                                    │
└────────────────────────────────────────────────────────────────────┘
        │              │              │              │
        ▼              ▼              ▼              ▼
┌─────────────────────── DATA PLANE (NODES) ───────────────────────┐
│                                                                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐         │
│  │ kubelet  │  │ kubelet  │  │ kubelet  │  │ kubelet  │  ...    │
│  │ + CRI    │  │ + CRI    │  │ + CRI    │  │ + CRI    │         │
│  │ + kube-  │  │ + kube-  │  │ + kube-  │  │ + kube-  │         │
│  │  proxy   │  │  proxy   │  │  proxy   │  │  proxy   │         │
│  │ + Pods   │  │ + Pods   │  │ + Pods   │  │ + Pods   │         │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘         │
│                                                                   │
└───────────────────────────────────────────────────────────────────┘
```

You submit a desired state to the **API server**. The state is persisted in **etcd**. The **scheduler** decides which node each new pod runs on. **Controllers** watch the API and drive the world toward the spec — restarting pods, scaling Deployments, balancing replicas. On each node, the **kubelet** receives pod assignments and talks to a container runtime (containerd, CRI-O) to actually run them. The **kube-proxy** programs iptables/IPVS so service IPs route to the right pods.

That's the whole loop. Everything else — Ingress, NetworkPolicy, HPA, CRDs, operators, service mesh — is built on top of this control-loop model.

---

## 2. Core objects

Kubernetes resources are nouns. You don't tell it "go restart this." You tell it "this is what should exist." The controller decides the steps.

| Object | What it is |
|---|---|
| **Pod** | The smallest deployable unit — one or more tightly coupled containers sharing network and storage |
| **Deployment** | Declarative manager for stateless replicated pods (handles rollouts) |
| **StatefulSet** | Like Deployment, but for pods that need stable identity and persistent storage |
| **DaemonSet** | One pod per node (logging agents, CNI components) |
| **Job / CronJob** | Run-to-completion or scheduled batch work |
| **Service** | Stable virtual IP that load-balances across a set of pods |
| **Ingress** | HTTP/S routing into the cluster (host, path → service) |
| **ConfigMap** | Non-secret key/value config injected into pods |
| **Secret** | Like ConfigMap, but base64-encoded and treated as sensitive |
| **PersistentVolume / PersistentVolumeClaim** | Abstraction over network storage (EBS, NFS, etc.) |
| **Namespace** | Logical partition of cluster resources |
| **ServiceAccount + Role + RoleBinding** | RBAC primitives |
| **NetworkPolicy** | Pod-level firewall rules |
| **HorizontalPodAutoscaler** | Scales replicas by CPU/mem/custom metrics |
| **CustomResourceDefinition (CRD)** | Extension mechanism — define your own object types |

The Pod is the atom. Almost everything else is a controller that creates or manages Pods.

---

## 3. A Deployment, minimal

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: prod
spec:
  replicas: 3
  selector:
    matchLabels: {app: api}
  template:
    metadata:
      labels: {app: api}
    spec:
      containers:
        - name: api
          image: ghcr.io/acme/api@sha256:abc123...
          ports: [{containerPort: 8080}]
          resources:
            requests: {cpu: 100m, memory: 128Mi}
            limits:   {cpu: 500m, memory: 512Mi}
          readinessProbe:
            httpGet: {path: /healthz, port: 8080}
            periodSeconds: 5
          livenessProbe:
            httpGet: {path: /livez, port: 8080}
            periodSeconds: 30
          env:
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef: {name: db-creds, key: url}
---
apiVersion: v1
kind: Service
metadata: {name: api, namespace: prod}
spec:
  selector: {app: api}
  ports:
    - port: 80
      targetPort: 8080
```

This is a complete production deployment minus ingress and PDB. It:
- pins the image by digest (reproducible)
- declares CPU/memory requests (for scheduling) and limits (for cgroups)
- has separate readiness and liveness probes (more on these below)
- pulls secrets from a Secret object, not env literals

---

## 4. The scheduler — how pods land on nodes

When a Pod is created, the scheduler walks every node and:

1. **Filters** — drop nodes that don't satisfy hard constraints (resources, node selector, taints, affinity).
2. **Scores** — rank surviving nodes by soft preferences (balanced load, image locality, topology spread).
3. **Binds** — write the chosen node onto the Pod spec. Kubelet on that node picks it up.

What you give the scheduler shapes its decisions:

- **`resources.requests`** — what the scheduler considers "reserved." If you request 500m CPU, it deducts 500m from node capacity. **This is the single most important field for cluster economics.** Under-request and you'll pack too tight (noisy neighbors); over-request and you'll waste capacity (and money).
- **`nodeSelector` / `affinity`** — "must run on a GPU node," "spread replicas across zones."
- **`taints` / `tolerations`** — nodes repel pods unless the pod tolerates the taint. Used for dedicated node pools (e.g., GPU, spot, system).
- **`topologySpreadConstraints`** — ensure pods spread across zones/racks/hosts. Critical for HA.
- **`podAntiAffinity`** — "don't run two of these on the same node."

### Limits vs requests — the subtle pair

```yaml
resources:
  requests: {cpu: 100m, memory: 128Mi}   # scheduling, guaranteed
  limits:   {cpu: 500m, memory: 512Mi}   # cap, OOM-kill if exceeded
```

- CPU is **compressible** — over the limit you get throttled.
- Memory is **incompressible** — over the limit you get killed.
- **No memory limit** is dangerous: a leak kills the node.
- **CPU limits are controversial.** Many production teams omit them, since CFS throttling can cause tail-latency spikes on bursty workloads. Memory limits are non-negotiable; CPU limits are situational.

---

## 5. Probes — readiness vs liveness vs startup

This trips up everyone at first.

| Probe | What it controls | Failure consequence |
|---|---|---|
| **readiness** | "Should I receive traffic?" | Pod removed from Service endpoints (still running) |
| **liveness** | "Is this process still alive?" | Pod **killed and restarted** |
| **startup** | "Is the app done initializing?" | Liveness/readiness paused until startup passes |

Common mistakes:

- **One endpoint for both probes.** A slow DB makes `/health` fail → liveness restarts the pod → no time to recover. Split them: liveness checks "is the process alive," readiness checks "are dependencies ready."
- **No startup probe on slow-booting apps.** JVM cold start triggers liveness → constant crashloops.
- **Probing too often.** kubelet hammers your app every second.

Good defaults: readiness every 5s with 1s timeout, liveness every 30s with 3 failures before kill, startup probe with `failureThreshold * periodSeconds > 5 minutes` for slow apps.

---

## 6. Services and networking

Every pod gets its own routable IP. Pods come and go; IPs change. **Services** are the stable abstraction:

| Service type | Purpose |
|---|---|
| **ClusterIP** (default) | Cluster-internal virtual IP, load-balanced to backing pods |
| **NodePort** | Exposes the service on a static port on every node |
| **LoadBalancer** | Cloud provider provisions an external LB (ELB, NLB, GCLB) |
| **ExternalName** | DNS CNAME to an external host |
| **Headless** (`clusterIP: None`) | Returns pod IPs directly via DNS; used by StatefulSets |

How a ClusterIP actually works: kube-proxy programs iptables (or IPVS) rules so that traffic to the cluster IP is DNAT'd to a randomly chosen healthy pod IP. There's no central proxy — the redirect happens in the kernel on each node.

DNS comes from **CoreDNS**, the in-cluster resolver. A service `api` in namespace `prod` is reachable at `api.prod.svc.cluster.local` (or just `api` from within the `prod` namespace).

### Ingress

External HTTP/HTTPS routing. You install an **Ingress controller** (nginx-ingress, Traefik, HAProxy, Contour, or a cloud one like AWS ALB Ingress Controller / GCE Ingress) and write Ingress resources:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: {name: api, port: {number: 80}}
  tls:
    - hosts: [api.example.com]
      secretName: api-tls
```

For more dynamic L7 routing, mTLS, and observability, layer in a **service mesh** like Istio or Linkerd. See [Service Mesh →](../03-apis/service-mesh.md).

The **Gateway API** is replacing Ingress for new clusters — richer routing, multi-team-friendly, vendor-neutral.

---

## 7. Storage and stateful workloads

Stateless apps are easy. Stateful apps need durable, identity-bound storage.

**StatefulSet** gives each pod:
- A **stable name** (`mysql-0`, `mysql-1`, `mysql-2`)
- A **stable network identity** (DNS that survives restarts)
- A **stable PersistentVolumeClaim** that follows the pod across reschedules

PVCs are claims for storage. The cluster's **StorageClass** decides what provisions them — EBS on AWS, GCE PD on GCP, NetApp on-prem, Longhorn / Rook for distributed. The **CSI** (Container Storage Interface) is the plugin API.

The honest take on databases in Kubernetes: **operators have matured to where running Postgres, MySQL, Cassandra, Kafka on K8s is reasonable** (CrunchyData/Zalando Postgres operators, Strimzi for Kafka, K8ssandra for Cassandra). But managed services (RDS, Aurora, CloudSQL, MSK, Confluent Cloud) still win on operational simplicity for most teams. Use Kubernetes for stateful workloads when you have a real reason — multi-cloud portability, regulatory isolation, cost — not because "it's all in K8s now."

---

## 8. Secrets, ConfigMaps, and configuration

**ConfigMap** = key/value, mounted as files or env vars.

**Secret** = same shape, base64-encoded, treated as sensitive (encrypted at rest in etcd if you configured it).

Secrets are **not strongly secret by default** — `kubectl get secret -o yaml` returns the base64. For real secret handling:

- Encrypt etcd at rest with a KMS provider (AWS KMS, GCP KMS).
- Use **External Secrets Operator** to sync from Vault, AWS Secrets Manager, GCP Secret Manager.
- Use **SOPS** + **Sealed Secrets** for GitOps workflows where secrets must live in git.

ConfigMaps and Secrets mounted as files are auto-updated when the source changes — but only the file on disk updates. Your app must re-read it. That's why you see SIGHUP handlers and config watchers in production code.

---

## 9. Autoscaling — three axes

| Scaler | Scales | Trigger |
|---|---|---|
| **HPA** (Horizontal Pod Autoscaler) | Replica count | CPU, memory, custom/external metrics |
| **VPA** (Vertical Pod Autoscaler) | Pod resource requests | Historical usage |
| **Cluster Autoscaler** / **Karpenter** | Number of nodes | Unschedulable pods |

HPA reacts on metrics from the metrics server or Prometheus (via adapters). KEDA extends HPA to scale on queue depth, Kafka lag, Redis lists, cron expressions — anything you can express as a metric.

**Karpenter** (originally AWS, now broader) is the modern alternative to Cluster Autoscaler. Instead of pre-defined node groups, it provisions exactly the instance shape your pending pods need, picking spot when possible. Cost savings of 30–60% for elastic workloads are common.

See [Auto-Scaling →](../10-scalability/auto-scaling.md).

---

## 10. Rollouts and deployment strategies

Deployments roll updates via a `rollingUpdate` strategy by default:

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 25%
    maxSurge: 25%
```

The Deployment controller creates a new **ReplicaSet** with the new image and scales it up while scaling the old one down, respecting `maxUnavailable` and `maxSurge`. A failed rollout can be paused or rolled back with `kubectl rollout undo`.

For more sophisticated strategies — canary, blue-green, automated analysis — use:

- **Argo Rollouts** — adds canary/blue-green to Deployments with metric-based promotion.
- **Flagger** — same idea, integrates with service meshes.

See [Blue-Green Deployment →](./blue-green.md), [Canary Deployment →](./canary.md), [Rolling Deployment →](./rolling.md).

### PodDisruptionBudget (PDB)

PDBs limit voluntary disruptions:

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata: {name: api-pdb}
spec:
  minAvailable: 2
  selector:
    matchLabels: {app: api}
```

Without a PDB, a node drain (during cluster upgrade, autoscaler scale-down) can take all your replicas down at once. PDBs are mandatory for any workload that must stay available.

---

## 11. RBAC and security

K8s authn is delegated (OIDC, certificates, cloud IAM). K8s authz is **RBAC**: ServiceAccounts get Roles or ClusterRoles bound by RoleBindings.

Principle: **every workload gets its own ServiceAccount with least privilege**. Default ServiceAccounts get more than they should — `automountServiceAccountToken: false` unless the pod needs API access.

Hardening checklist:

```
[ ] Encrypted etcd at rest
[ ] OIDC-backed kubectl access (no shared admin creds)
[ ] NetworkPolicies default-deny, explicit allows
[ ] Pod Security Admission (or Kyverno/OPA Gatekeeper) enforced
[ ] No :latest tags; pinned digests
[ ] Image scanning at push; admission controller blocks failed scans
[ ] Distroless or minimal base images
[ ] runAsNonRoot, readOnlyRootFilesystem, drop ALL capabilities
[ ] Workload identity (no static cloud creds on pods)
[ ] Audit logs to a SIEM
[ ] Regular CVE patching of the cluster itself
```

**Pod Security Admission** (replaced PodSecurityPolicy) labels namespaces `privileged | baseline | restricted`. For application namespaces, `restricted` is the right default.

For pod-to-cloud-API auth, use **Workload Identity** (GKE) / **IRSA** (EKS) / **Azure Workload Identity** — never static credentials in pods.

See [Zero Trust Architecture →](../12-security/zero-trust.md).

---

## 12. Helm, Kustomize, and GitOps

You don't write raw YAML by hand at scale. Two dominant approaches:

| Tool | Approach |
|---|---|
| **Helm** | Templated charts. Variables → rendered manifests. Strong ecosystem (charts on Artifact Hub). |
| **Kustomize** | Overlay-based. Base + patches per environment. No templating. Built into kubectl. |

Most teams use **both**: Helm to install third-party operators and apps, Kustomize for in-house apps.

**GitOps** (Argo CD, Flux) makes the cluster state a continuous reconciliation of a git repo. Pull-based instead of push-based. The repo becomes the source of truth; the cluster pulls and applies. Audit, rollback, and blast-radius control all improve. See [CI/CD Pipelines →](./cicd.md).

---

## 13. Operators and CRDs — extending Kubernetes

The control-loop model is open. You can define your own resource type (a CRD) and write a controller that reconciles it. This is an **operator**.

Real operators in production:
- **CrunchyData Postgres Operator**, **Zalando Postgres Operator** — managed Postgres.
- **Strimzi** — managed Kafka.
- **cert-manager** — manages TLS certificates via ACME.
- **External Secrets Operator** — pulls secrets from Vault/AWS/GCP.
- **Argo Workflows** — workflow engine as CRDs.

If your platform has a domain concept (a "Tenant," a "DataPipeline"), encoding it as a CRD with an operator is often the right move — your engineers operate on a familiar abstraction, and the controller handles the boring reconciliation.

---

## 14. Multi-tenancy

Sharing a cluster across teams or customers is hard. Options:

| Model | Isolation | Cost |
|---|---|---|
| **Cluster per tenant** | Strong | High |
| **Namespace per tenant** + NetworkPolicy + RBAC + ResourceQuota | Medium | Low |
| **Virtual cluster** (vcluster) | Strong-ish | Medium |
| **Pod-level isolation** with Kata/gVisor | Strong runtime | High |

Naïve namespace isolation leaks: shared CRDs, shared cluster-scoped resources, noisy neighbors via the kernel scheduler, image pull secrets. For SaaS multi-tenancy you almost always want at least namespace + NetworkPolicy + ResourceQuota + LimitRange + PodSecurityAdmission `restricted`. For regulated workloads, separate clusters are still common.

See [Multi-Tenancy →](./multi-tenancy.md).

---

## 15. Operational reality — what they don't tell you

Kubernetes works well. Running Kubernetes is *also* work. Things you'll wrestle with:

- **Cluster upgrades** — minor versions every ~4 months, EOL in ~14 months. Plan to upgrade at least twice a year. Tighten your API deprecation discipline now.
- **etcd** — single source of truth, gets slow at scale. Tune disk, limit object size, prune events.
- **API server load** — too many watchers, too many CRDs, too many leases. Lots of small controllers are healthier than one giant one.
- **DNS** — CoreDNS misconfiguration is the most common "everything is broken" symptom.
- **Networking** — CNI plugins behave differently under load. Cilium and Calico are the most production-proven.
- **Cost** — clusters drift expensive. Use Karpenter, spot, requests-right-sizing (Goldilocks, KRR), and cost-visibility tools (OpenCost, Kubecost).
- **Observability cost** — Prometheus + logs explodes. Cardinality discipline is required. See [Metrics →](../13-observability/metrics.md).
- **Day-2 ops** — backups (Velero), disaster recovery, secret rotation, certificate rotation, node OS patching.

**Use a managed control plane.** EKS, GKE, AKS — even if you self-manage workers. Running your own control plane is rarely worth it unless you have a strong reason (air-gapped, edge, regulatory).

---

## 16. When NOT to use Kubernetes

- **You have one service.** Use Cloud Run, App Runner, Fly, Render. Days vs months.
- **You have a small team (<10 engineers) and no platform expertise.** K8s ops is a real cost; pay it knowingly.
- **Your workload is fundamentally a function on demand.** Lambda, Cloud Functions, Workers.
- **You need GPU clusters with batch scheduling and gang scheduling.** Plain Kubernetes works but Slurm or Ray is sometimes a better fit.
- **You're on Windows-only stacks.** Kubernetes supports Windows nodes but the ecosystem is thinner.

The reverse: if you have many services, multi-region requirements, polyglot stacks, dynamic capacity needs, and the team to support it — Kubernetes is the consensus answer for a reason. It standardizes the boring middle layer so teams can ship features.

---

## 17. Common Mistakes / Anti-Patterns

- **Treating K8s as a fancy `docker run`.** Without resource requests, probes, PDBs, and NetworkPolicy, you have less reliability than a single VM.
- **Memory limits unset, OOM-killing the node.** Always set memory limits.
- **CPU limits set blindly on latency-sensitive workloads.** CFS throttling spikes p99.
- **One probe for liveness and readiness.** Cascading restarts when downstream slows.
- **`:latest` image tags.** Non-reproducible deploys, broken rollbacks.
- **Manual `kubectl edit` in production.** State drifts from git, GitOps reconciles back, surprise outage.
- **Hosting your own etcd to "save money."** You'll spend more in outages.
- **No PDB on critical services.** Cluster maintenance takes them all down at once.
- **One giant cluster for all environments.** Blast radius is the whole company.
- **Secrets in plain ConfigMaps "because they're not very sensitive."** They're sensitive in aggregate.
- **CRDs everywhere with no governance.** Each CRD is API surface and controller load.
- **Ignoring the cluster upgrade path.** v1.25 → v1.30 with no plan = pain.
- **Service mesh installed because the conference talk was cool.** Plain Services + Ingress + good libraries handle 80% of needs.

---

## 18. Cheat Card

```
PURPOSE   Run containers across many machines with self-healing,
          declarative state, and a uniform API.

CONTROL PLANE   API server  · scheduler  · controller manager  · etcd
DATA PLANE      kubelet  · CRI runtime  · kube-proxy

ATOMS
  Pod           = unit of execution (one+ containers, shared net/storage)
  Deployment    = replicated stateless pods, rolling updates
  StatefulSet   = identity-bound stateful pods + per-pod PVCs
  Service       = stable virtual IP load-balancing to pods
  Ingress       = HTTP/S routing into the cluster

MUST-HAVES (per workload)
  resources.requests/limits   probes (readiness + liveness)
  ServiceAccount + RBAC       NetworkPolicy
  PodDisruptionBudget         topologySpreadConstraints
  Image pinned by digest      runAsNonRoot, readOnlyRootFilesystem

SCALING
  HPA  – replicas by metric        KEDA – queue/lag-based
  VPA  – pod size by usage         Karpenter – nodes JIT

WHEN TO USE
  10+ services / 50+ engineers
  Multi-region, polyglot stacks
  Need declarative ops + uniform API
  Already burning ops time on per-service infra

WHEN NOT TO USE
  Tiny teams, single service     Cloud Run / Lambda are faster
  Pure batch / HPC              Slurm / Ray sometimes win
  Windows-only stacks           Ecosystem is thinner

PITFALLS
  no memory limit  →  OOMs nuke nodes
  bad probes      →  crashloops or zero traffic
  no PDB          →  cluster ops take you down
  :latest tags    →  reproducibility / rollback dead
  one cluster all envs  →  blast radius = company

RULE   Kubernetes is a platform. Treat it like one — give it a team,
       a budget, and a roadmap. Don't bolt it onto an app team.
```

---

## 19. Resources

### Books
- *Kubernetes Up and Running* — Hightower, Burns, Beda.
- *Production Kubernetes* — Dotson, Vyas, Saladi.
- *Programming Kubernetes* — Hausenblas, Schimanski (for operator work).
- *The Kubernetes Book* — Nigel Poulton (best intro).

### Documentation
- **Kubernetes** — <https://kubernetes.io/docs/>
- **CNCF Landscape** — <https://landscape.cncf.io>
- **Helm** — <https://helm.sh>
- **Argo CD** — <https://argo-cd.readthedocs.io>
- **Karpenter** — <https://karpenter.sh>

### Articles
- "Borg, Omega, and Kubernetes" — Google paper: <https://research.google/pubs/pub44843/>
- "11 ways (not) to get hacked" — Andrew Martin: <https://kubernetes.io/blog/2018/07/18/11-ways-not-to-get-hacked/>
- "Kubernetes patterns" — Bilgin Ibryam: <https://www.redhat.com/en/resources/oreilly-kubernetes-patterns-cloud-native-apps>
- Stripe — "Online migrations at scale" (Stripe's K8s adoption notes).

### Videos
- *Kubernetes the Hard Way* — Kelsey Hightower (the canonical walkthrough).
- *KubeCon talks* — search topic + year on YouTube.
- TGIK (Joe Beda) archive — deep dives into individual primitives.

### Tools
- **kubectl**, **k9s**, **stern** — operator UX essentials.
- **Argo CD / Flux** — GitOps.
- **Karpenter / Cluster Autoscaler** — node scaling.
- **Goldilocks / KRR** — right-size requests/limits.
- **Velero** — backup and restore.
- **OPA Gatekeeper / Kyverno** — policy enforcement.
- **cert-manager** — TLS automation.
- **External Secrets Operator** — secret sync.

### Adjacent reading
- [Containers (Docker) →](./docker.md)
- [Blue-Green Deployment →](./blue-green.md)
- [Canary Deployment →](./canary.md)
- [Rolling Deployment →](./rolling.md)
- [CI/CD Pipelines →](./cicd.md)
- [Infrastructure as Code →](./iac.md)
- [Immutable Infrastructure →](./immutable-infra.md)
- [Auto-Scaling →](../10-scalability/auto-scaling.md)
- [Service Mesh →](../03-apis/service-mesh.md)
- [Microservices Architecture →](../14-architecture/microservices.md)
- [Sidecar Pattern →](../14-architecture/sidecar.md)

---

*Previous:* [← Containers (Docker)](./docker.md)  |  *Next:* [Blue-Green Deployment →](./blue-green.md)
