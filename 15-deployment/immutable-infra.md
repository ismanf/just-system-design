# Immutable Infrastructure

> **TL;DR** — **Immutable infrastructure** means servers (or containers, or VM images, or Lambda packages) are **never modified after they are created**. To change behavior, you build a new image, deploy new instances, and discard the old ones. No SSH, no `apt upgrade`, no in-place patching. The wins are huge: predictable state, identical environments, fast and clean rollback, and an audit trail that's just "what image was deployed when." The cost is upfront — you need a build pipeline that can spit out a complete artifact quickly, a deploy system that can roll new instances at scale, and a separation of **code** from **state** so that "replace the server" doesn't mean "lose customer data." Pioneered by Netflix, formalized by Chad Fowler, and now the default model behind containers, AWS AMIs, Packer, and serverless. **Immutable is no longer a nice-to-have; it's the baseline for any team above five engineers.**

---

## 1. The big picture

Two ways to update a server:

**Mutable (the old way):**

```
SSH in → apt update → apt upgrade nginx → reload → done
```

The server's state changes over time. Different servers in the same fleet drift apart. After a year, no two are identical. Reproducing the environment from scratch is impossible.

**Immutable (the new way):**

```
Build new image (nginx 1.27)
  → deploy to new instances
  → switch traffic
  → terminate old instances
```

You never modify the running server. You replace it. The "configuration" is the image; the "deploy" is the swap. The old server is destroyed, not patched.

```
┌──────────────────────────────────────────────────┐
│  Image build pipeline                            │
│  (Packer / Dockerfile / nix / Bazel)             │
└──────────────┬───────────────────────────────────┘
               │ artifact (AMI, OCI image, .tar)
               ▼
┌──────────────────────────────────────────────────┐
│  Artifact registry                               │
│  (ECR / GCR / Artifactory)                       │
└──────────────┬───────────────────────────────────┘
               │ deploy
               ▼
┌──────────────────────────────────────────────────┐
│  Fleet rolls forward                             │
│  - new instances spun up from artifact           │
│  - traffic shifted                               │
│  - old instances terminated, never reused        │
└──────────────────────────────────────────────────┘
```

The image is the unit of change. State (databases, object storage, persistent volumes) lives elsewhere and survives the churn.

---

## 2. Why this is the right default

### Predictable state

Every instance of v1.5.0 is identical to every other instance. If one works, they all work. If one has a bug, they all have it. No "weird one in us-east-1c" mysteries.

### Reproducible rollback

To roll back, you redeploy the previous image. Same image, same code, same OS packages, same everything. No fear that a config drift or a manual fix has been lost.

### Cleaner audit

"What is running in production?" → a list of image digests. That's it. No "let me SSH in and check the package version." Compliance loves this; so do incident responders.

### Cattle, not pets

Each instance is interchangeable. You don't name them, don't get attached, don't tune them individually. You scale by adding more. Failures are routine; replacement is automated. See the famous "Pets vs Cattle" framing (Bill Baker, then everyone).

### Composable deploys

Immutable artifacts compose well with everything that came after: blue-green, canary, rolling, autoscaling, IaC, GitOps. None of these work cleanly on mutable infra.

---

## 3. Levels of immutability

| Level | What's immutable | Where you see it |
|---|---|---|
| **VM image** | The whole OS + app baked in | AMIs (Packer), GCE images, Azure images |
| **Container image** | OS userland + app | Docker / OCI on K8s, ECS, Cloud Run |
| **Function package** | Code + deps zipped | AWS Lambda, GCP Cloud Functions |
| **Filesystem snapshot** | App + minimal OS | NixOS, Talos Linux, Bottlerocket |
| **Binary** | Just the app | Single static Go/Rust binary in a tiny container |

The pattern is the same at each level: a tagged, content-addressed artifact that you deploy and replace, never modify.

Containers are by far the dominant choice today. But VM images still matter: anything that requires a full OS (legacy apps, GPU drivers, specific kernels) lives in an AMI built with Packer or a similar tool. Lambda is the extreme — you don't even own the OS; you just hand over a zip.

---

## 4. The four hard problems immutability creates

Immutability solves a lot but introduces real challenges. The work goes into solving these.

### 4.1 State must live somewhere

If the server is ephemeral, where do databases, uploaded files, session data, and caches live?

- **Databases** → managed services (RDS, Aurora, CloudSQL) or stateful Kubernetes workloads with PersistentVolumes.
- **Files / blobs** → object storage (S3, GCS, Azure Blob).
- **Sessions** → Redis, a stateless JWT, or sticky-session-free design.
- **Caches** → external cache (Redis, Memcached) or accept warmup cost.
- **Logs / metrics** → ship to a central system, never relied on locally.

The discipline: **no important state on the instance's local disk**. Local disk is scratch, log buffer, container layer cache. Anything you can't afford to lose goes to a service that outlives the instance.

### 4.2 Configuration must be externalized

If the image is immutable but the database URL differs between staging and production, where does it come from?

Three sources:

- **Environment variables** — injected at runtime, never baked.
- **Config service** — etcd, Consul, AWS AppConfig, Kubernetes ConfigMaps.
- **Secrets manager** — Vault, AWS Secrets Manager, GCP Secret Manager.

The image is identical across environments; the *behavior* differs because the environment around it differs. The Twelve-Factor App rules formalize this. See [12factor.net](https://12factor.net) — Section III ("Config") is essentially the immutable-infra config rule.

### 4.3 Fast builds matter

If your image takes 30 minutes to build, you'll find reasons to skip it ("I'll just SSH and tweak it"). That's the slippery slope back to mutable.

Make image builds:
- **Fast** — under 5 minutes for most app changes (multi-stage Dockerfiles + cached layers).
- **Reproducible** — same input → same digest. Pin everything.
- **Signed** — Cosign, Notary v2, SLSA provenance.
- **Reviewed** — image change goes through PR, not ad hoc.

### 4.4 Patching and emergency changes

A CVE drops. Old way: `apt upgrade openssl` across the fleet. New way: build a new base image with the patch, rebake every dependent image, redeploy. That's slower per step but cheaper in aggregate because *every server is patched simultaneously and verifiably*.

Tools that help:
- **Renovate / Dependabot** for base image updates.
- **Trivy / Grype** scanning at push.
- **Base image registries** (distroless, Chainguard, Wolfi) that publish patched versions promptly.
- **Image deduplication** in registries to keep storage manageable.

For absolute emergencies (hot-patching a kernel CVE in production), some teams still allow temporary mutations with strict tracking. The rule: any mutation must be rolled into the next image build and the mutated instance must die soon after.

---

## 5. Building immutable artifacts — the tools

### Containers (the common case)

- `docker build` / `docker buildx` / `kaniko` / `buildah` / `ko` (Go).
- Multi-stage builds keep images small.
- Tag by content (digest) for true immutability — `:latest` is mutable by definition.
- Sign with Cosign, store SBOM with Syft, scan with Trivy.

See [Containers (Docker) →](./docker.md).

### VM images

- **Packer** (HashiCorp) — the standard. Builds AMIs, GCE images, Azure VHDs, OVAs, etc. from a JSON/HCL definition.
- **EC2 Image Builder** — AWS-native, integrates with AWS pipelines.
- **Customizing official images** — Ubuntu, Amazon Linux 2/2023, RHEL — using cloud-init plus a pre-bake step.

```hcl
# Packer example
source "amazon-ebs" "app" {
  region        = "us-east-1"
  instance_type = "t3.medium"
  source_ami    = "ami-0abc..."  # base Amazon Linux 2023
  ssh_username  = "ec2-user"
  ami_name      = "app-${var.git_sha}-${formatdate("YYYYMMDDhhmm", timestamp())}"
}

build {
  sources = ["source.amazon-ebs.app"]
  provisioner "shell" {
    scripts = ["./install-app.sh", "./harden.sh"]
  }
}
```

### Function packages

- AWS Lambda — zipped deployment package or container image. Versioned by Lambda; aliased for traffic shifting.
- GCP Cloud Functions / Cloud Run — image or source zip.
- Azure Functions — image or zip.

### Nix / NixOS

Nix is the radical version of immutability — every package, every configuration, every dependency is content-addressed and reproducible bit-for-bit. NixOS lets you roll back to any previous system generation. The learning curve is steep, but for teams who go in, the determinism is unmatched.

### Talos Linux / Bottlerocket / Flatcar

Minimal, API-driven, immutable Linux distributions designed for containers. No SSH (Talos has none at all). System config is declarative. They are Kubernetes-only OSes that lean fully into the immutable model.

---

## 6. Pets vs cattle — the cultural shift

This phrase from Bill Baker (later popularized in the DevOps community) captures the operational mindset shift:

> **Pets** are servers you name (`db01`, `web02`), care for, patch by hand, and grieve when they fail. You know each one's quirks.
>
> **Cattle** are interchangeable instances with auto-generated IDs. When one is sick, you shoot it and bring up another. No grief.

Immutable infrastructure makes cattle possible at scale. You can lose any instance at any time and the system shrugs. Chaos engineering (deliberately killing instances to verify resilience) is feasible because you've already accepted that every instance is disposable.

For databases and other stateful systems, pets still exist — you cannot blindly destroy a Postgres primary. But around the stateful core, everything is cattle: stateless services, batch workers, web frontends, ML inference workers, all replaceable in minutes.

---

## 7. Immutable + IaC + GitOps — the whole loop

These three reinforce each other. The combination is what modern cloud-native infrastructure actually is:

```
   ┌─────────────────────────────────────────────────┐
   │  Git repo (source of truth)                     │
   │  - Dockerfile / Packer template (image source)  │
   │  - Terraform / Pulumi (infra)                   │
   │  - Helm / Kustomize (deploys)                   │
   └─────────────────────┬───────────────────────────┘
                         │
                         ▼
   ┌─────────────────────────────────────────────────┐
   │  CI builds artifacts                            │
   │  - Container image (signed)                     │
   │  - AMI (Packer build)                           │
   │  - Terraform plan / apply                       │
   └─────────────────────┬───────────────────────────┘
                         │
                         ▼
   ┌─────────────────────────────────────────────────┐
   │  CD deploys + reconciles                        │
   │  - new instances replace old                    │
   │  - traffic shifts                               │
   │  - old instances terminated                     │
   └─────────────────────────────────────────────────┘
```

- **IaC** describes the infrastructure declaratively (see [IaC →](./iac.md)).
- **Immutable artifacts** flow through the pipeline (see [CI/CD →](./cicd.md)).
- **GitOps** reconciles the cluster / cloud to the desired state.

If any of these is missing, the others bleed value. Mutable infra makes IaC drift constantly. No CI means images are built on laptops. No GitOps means deploys are still manual.

---

## 8. When immutability is overkill

Yes, there are cases:

- **One-off scripts, ad hoc data work.** Nobody is building a Dockerfile for a one-shot SQL migration. Just don't pretend it's production infra.
- **Legacy systems that resist containerization.** Some 90s enterprise apps with weird licensing, hardware keys, or huge state assume a long-lived host. Bake an AMI if you can; otherwise, document the mutation discipline meticulously.
- **HPC / scientific computing.** Sometimes mutable workstations with shared filesystems are the right tool. The cattle-vs-pets line moves.

But for nearly any service serving real traffic with more than a couple of instances, immutability is the right default. The argument is no longer "should we?" but "how thoroughly?"

---

## 9. Common Mistakes / Anti-Patterns

- **SSHing into containers/instances to "just fix one thing."** That fix dies with the next deploy. Worse, it papers over the real issue.
- **`:latest` tags treated as immutable.** They're not — they get re-pushed. Use digests.
- **Building images on developer laptops.** Inconsistent builds, no audit. Build in CI.
- **State on local disk that "doesn't really matter."** Until it does (one server gets re-imaged, half a day of in-memory work is gone).
- **Config baked into the image.** Now you need a different image per environment. Externalize config.
- **No image rebake discipline for security patches.** Patched OS in one image, vulnerable in another, no idea which is where.
- **Long-lived servers that "we'll containerize someday."** That day never comes by accident.
- **No retention policy on registries.** Hundreds of GB of dead images accumulate.
- **Hot-patch in prod, never roll into the image.** The image fleet has the bug, the hot-patched instances are correct, the next deploy *regresses the fix*.
- **Treating Lambda / serverless as mutable because "it's just code."** The package is your image; treat it that way.
- **Stuffing the image with debug tools "just in case."** Bigger image, larger attack surface. Use ephemeral debug sidecars (`kubectl debug`) instead.
- **Killing the old image before new is healthy.** Always health-check before terminate; never simultaneously.

---

## 10. Cheat Card

```
PURPOSE   Never modify a running server. Replace it. The image
          (or function package) is the unit of change. Old state
          dies; new state replaces it.

CORE PRINCIPLES
  Build once, deploy many times
  Identical artifact across all environments
  Externalize config + state outside the artifact
  Roll forward (or back) by replacing instances
  No SSH-and-fix in production

LEVELS
  Container image   (Docker / OCI) — default choice
  VM image          (AMI via Packer) — when you need a full OS
  Function package  (Lambda / Cloud Functions / Cloud Run)
  Filesystem image  (NixOS, Talos, Bottlerocket)

EXTERNALIZE
  Database          managed service or PV
  Files / blobs     object storage (S3, GCS)
  Sessions          Redis or stateless JWT
  Config            env vars / config service
  Secrets           Vault / Secrets Manager
  Logs / metrics    central service

WHEN TO USE
  Every production service, basically
  Anywhere blue-green / canary / rolling matters
  Anywhere reproducibility matters
  Anywhere "what's running?" must have a precise answer

WHEN NOT (RARELY)
  Tiny one-off scripts
  Legacy apps that can't be containerized at all
  HPC workstations with shared filesystems

PITFALLS
  SSH-and-fix → invisible drift
  :latest tags → not actually immutable
  State on local disk → lost on replace
  Config baked in → image-per-env explosion
  No image rebake discipline → vulnerable patches everywhere
  Killing old before new is healthy

RULE   The image is the unit of change. The instance is cattle.
       State lives outside. To change the system, replace it.
```

---

## 11. Resources

### Books
- *Infrastructure as Code* — Kief Morris. Foundational chapters on immutability.
- *The Practice of Cloud System Administration* — Limoncelli, Chalup, Hogan. The pets/cattle framing in detail.
- *Continuous Delivery* — Humble & Farley. Builds the case for immutable artifacts.

### Documentation
- **The Twelve-Factor App** — <https://12factor.net>
- **Packer** — <https://developer.hashicorp.com/packer>
- **NixOS** — <https://nixos.org>
- **Talos Linux** — <https://www.talos.dev>
- **Bottlerocket** — <https://aws.amazon.com/bottlerocket/>
- **Chainguard / Wolfi** — <https://www.chainguard.dev>

### Articles
- "Immutable Infrastructure" — Chad Fowler (the original 2013 essay).
- "Trash Your Servers and Burn Your Code: Immutable Infrastructure and Disposable Components" — Chad Fowler.
- "Pets vs Cattle" — Randy Bias / Bill Baker.
- "The History of Pets vs Cattle and How to Use the Analogy Properly" — Randy Bias.
- Netflix Tech Blog — Asgard, Aminator, immutable deploy history.

### Videos
- *Immutable Infrastructure* — Chad Fowler talks on YouTube.
- *NixOS in Production* — various conference talks.
- ByteByteGo — "Immutable Infrastructure Explained."

### Tools
- **Docker / BuildKit / Buildah / Kaniko** — container builders.
- **Packer** — VM image builder.
- **Nix / NixOS** — declarative, reproducible builds and OS.
- **Talos / Bottlerocket / Flatcar** — immutable container-host OSes.
- **Renovate / Dependabot** — base image and dependency PRs.
- **Trivy / Grype / Snyk** — image scanning.
- **Cosign / Notary v2 / Sigstore** — signing and provenance.
- **Distroless / Chainguard Images** — minimal, regularly patched base images.

### Adjacent reading
- [Containers (Docker) →](./docker.md)
- [Container Orchestration (Kubernetes) →](./kubernetes.md)
- [Infrastructure as Code →](./iac.md)
- [CI/CD Pipelines →](./cicd.md)
- [Blue-Green Deployment →](./blue-green.md)
- [Canary Deployment →](./canary.md)
- [Rolling Deployment →](./rolling.md)
- [Chaos Engineering →](../11-reliability/chaos-engineering.md)
- [Stateful vs Stateless Services →](../01-foundations/stateful-vs-stateless.md)

---

*Previous:* [← CI/CD Pipelines](./cicd.md)  |  *Next:* [Multi-Tenancy →](./multi-tenancy.md)
