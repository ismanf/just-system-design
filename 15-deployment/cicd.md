# CI/CD Pipelines

> **TL;DR** — **CI (Continuous Integration)** is the practice of integrating every developer's work to a shared mainline many times a day, with automated builds and tests guarding each merge. **CD** is either **Continuous Delivery** (every green build is *deployable*) or **Continuous Deployment** (every green build is *automatically deployed*). The two together turn the release process from a quarterly ritual into a five-minute background loop. A working pipeline is **fast** (under 15 minutes from push to production for app changes), **deterministic** (same inputs → same artifact), **reversible** (one-click rollback), and **observable** (you know exactly what's deploying where). Teams that ship multiple times a day rely on this loop more than on any other single piece of engineering infrastructure. Get it right early — every other practice (canary, feature flags, IaC) compounds on top of it.

---

## 1. The big picture

```
   Developer
      │ push
      ▼
┌──────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│   CI: build  │──►│  CI: test    │──►│  CI: package │──►│ CD: deploy   │
│  + lint      │   │  unit + int  │   │  image + sig │   │  rollout     │
└──────────────┘   └──────────────┘   └──────────────┘   └──────────────┘
                                              │
                                              ▼
                                       ┌──────────────┐
                                       │  Registry +  │
                                       │   artifacts  │
                                       └──────────────┘
                                              │
                                              ▼
                              ┌────────────────────────────────┐
                              │  Dev → Staging → Canary → Prod │
                              └────────────────────────────────┘
```

Code goes in one side. A vetted, signed artifact comes out the other. Then it deploys, gradually, with automated checks at each stage. The whole loop is the **delivery pipeline** — the system most engineering orgs invest in second only to the product itself.

---

## 2. CI vs CD vs CD — the words people confuse

| Term | Meaning |
|---|---|
| **Continuous Integration** | Every change merges to mainline often (multiple times/day) with automated tests. |
| **Continuous Delivery** | Every green build *could* be deployed at any time, manually or with a click. |
| **Continuous Deployment** | Every green build *is* deployed automatically — no human in the loop. |

You can have CI without any CD. You usually have Delivery without Deployment. Full Deployment to production is a strong claim that requires excellent tests, fast rollback, feature flags, and a culture of trust. Many mature teams sit at "delivery for production, deployment for staging."

The other definition you'll hear — "CI/CD pipeline" — usually means the whole orchestrated workflow regardless of the strict definition. We'll use it that way here.

---

## 3. Trunk-based development — the cultural prerequisite

CI/CD assumes a single mainline (`main` / `master` / `trunk`) that everyone merges to constantly. Long-lived feature branches are CI's enemy: merge conflicts pile up, integration is delayed, "main is broken" becomes routine.

The discipline:
- Small PRs, merged within hours to days, not weeks.
- Behind feature flags when work is incomplete (see [Feature Flags →](./feature-flags.md)).
- Main is always shippable.
- "Builds the world from main" is a fact, not an aspiration.

This isn't optional. CI/CD without trunk-based development is a build server, not a pipeline.

---

## 4. Pipeline stages in detail

### 4.1 Build

- Resolve dependencies (cached layer, lockfile-bound).
- Compile / bundle.
- Run static analysis: linters, type checkers, formatters.

```yaml
# Example: GitHub Actions
- uses: actions/checkout@v4
- uses: actions/setup-node@v4
  with: {node-version: 20, cache: 'npm'}
- run: npm ci
- run: npm run lint
- run: npm run typecheck
- run: npm run build
```

### 4.2 Test

The test pyramid:

```
       ┌────────────┐
       │  E2E       │   slow, fragile, few
       ├────────────┤
       │ Integration│   medium speed, focused
       ├────────────┤
       │   Unit     │   fast, many, drive most coverage
       └────────────┘
```

Run cheap tests first. Fail fast. Parallelize what you can. Cache dependencies and test results.

- **Unit tests** — pure functions, mocked I/O. Should take seconds.
- **Integration tests** — real database, real cache, real queue (testcontainers, Docker Compose). Minutes.
- **Contract tests** — verify service A's expectation of service B (Pact, gRPC schema diff).
- **End-to-end** — browser tests, full-stack smoke tests. Slow. Use sparingly.
- **Security scans** — SAST (Semgrep, CodeQL), dependency CVEs (Snyk, Trivy, Dependabot).
- **Performance baseline** — k6, Locust on critical paths. Fail if p99 regresses.

### 4.3 Package

Produce the **immutable, signed artifact** that flows through every later environment:

- Container image, tagged with git SHA and pushed to registry.
- Optional: jar, wheel, native binary, OCI image, Helm chart.
- **Sign** the artifact (Cosign, Notary v2, SLSA provenance).
- **Generate SBOM** (Software Bill of Materials) — Syft, CycloneDX.
- **Scan** for CVEs at push time — Trivy, Grype, Snyk Container.

The artifact you ship to prod must be **the exact one that passed tests**. Build once, deploy many times.

### 4.4 Deploy

- Dev → Staging → Canary → Prod, with gates between.
- Manual approval for production (Continuous Delivery) or automatic (Continuous Deployment).
- Use [Rolling →](./rolling.md) or [Canary →](./canary.md) or [Blue-Green →](./blue-green.md) strategies.
- Run **post-deploy verification**: smoke tests, synthetic probes, key metric checks.

### 4.5 Observe

- Track deploy events as a metric/annotation in dashboards.
- Wait through a **bake window** (5–30 min) watching error rate, latency, business KPIs.
- Auto-rollback on breach. Many platforms support this directly (Argo Rollouts, Spinnaker Kayenta).

---

## 5. Tooling — the landscape

| Tool | Strength |
|---|---|
| **GitHub Actions** | Simple, integrated with GitHub. Default for most teams. |
| **GitLab CI/CD** | Powerful, integrated with GitLab. Strong YAML DSL. |
| **CircleCI** | Fast, good caching. SaaS or self-hosted. |
| **Buildkite** | Hybrid agent model: SaaS control plane, your runners. Great for big monorepos. |
| **Jenkins** | Old guard. Most flexible, hardest to keep clean. Still huge install base. |
| **Argo CD / Flux** | Kubernetes GitOps for the CD half. |
| **Spinnaker** | Heavy-weight multi-cloud CD. Netflix-grade pipelines. |
| **Tekton** | Kubernetes-native CI/CD primitives. |
| **Drone** | Container-native, lightweight. |
| **Bazel + GitHub Actions** | Common monorepo combo for large codebases. |

Pick by:
- Where your code lives (GitHub → Actions is the lowest friction).
- Scale of monorepo (Bazel/Buck/Pants for huge ones).
- Team experience.
- Constraints on the runner (self-hosted, GPU, locked-down networks).

There is no single right answer. There are many wrong ones, mostly the ones where pipelines take hours and nobody owns them.

---

## 6. A minimal but real GitHub Actions pipeline

```yaml
name: deploy
on:
  push:
    branches: [main]
  pull_request:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: {node-version: 20, cache: 'npm'}
      - run: npm ci
      - run: npm run lint && npm run typecheck && npm run test
      - run: npm run build

  image:
    needs: build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write    # for OIDC
      packages: write    # for GHCR
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ghcr.io/acme/api:${{ github.sha }}
            ghcr.io/acme/api:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max
      - uses: aquasecurity/trivy-action@master
        with:
          image-ref: ghcr.io/acme/api:${{ github.sha }}
          severity: HIGH,CRITICAL
          exit-code: '1'
      - uses: sigstore/cosign-installer@v3
      - run: cosign sign --yes ghcr.io/acme/api:${{ github.sha }}

  deploy-staging:
    needs: image
    runs-on: ubuntu-latest
    environment: staging
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::xxx:role/github-deploy-staging
          aws-region: us-east-1
      - run: |
          aws eks update-kubeconfig --name staging
          kubectl set image deploy/api api=ghcr.io/acme/api:${{ github.sha }}
          kubectl rollout status deploy/api --timeout=5m

  deploy-prod:
    needs: deploy-staging
    runs-on: ubuntu-latest
    environment: production    # requires manual approval
    steps:
      # ...same as staging, against the prod cluster, possibly via Argo Rollouts
```

Notes worth absorbing:
- **OIDC**, not long-lived AWS keys. The `id-token: write` permission lets the workflow assume a role with a short-lived token.
- **Cached Docker layers** speed builds dramatically.
- **Trivy** fails the build on high/critical CVEs.
- **Cosign** signs the image — provable provenance.
- **environment: production** is a GitHub gate that requires an approver.

---

## 7. Push vs pull deploys (GitOps)

Two ways to apply a change to a cluster or environment:

### Push
CI runs `kubectl apply` or `terraform apply` to the cluster. Direct, simple, requires CI to hold credentials to every target.

### Pull (GitOps)
A controller in the cluster (Argo CD, Flux) watches a git repo and reconciles the cluster to match. CI updates the git repo with a new image tag; the controller picks it up and applies.

GitOps advantages:
- Cluster reconciles itself toward git — drift heals automatically.
- CI doesn't need cluster credentials.
- The git repo is the audit log.
- Easy multi-cluster — same repo, different agents.

GitOps disadvantages:
- Two-step deploy feedback loop (CI updates repo, controller picks up → slightly slower visible deploys).
- More moving parts, more learning curve.

Most Kubernetes-heavy organizations have moved to GitOps. For non-K8s workloads, push is still common. Both are valid.

---

## 8. Pipeline speed — the metric that matters most

Slow pipelines kill velocity. The numbers that matter:

- **Lead time for changes** (commit → production): minutes to hours = elite, days to weeks = low.
- **Deployment frequency**: multiple per day = elite, monthly = low.
- **Time to restore** (MTTR): minutes = elite, days = low.
- **Change failure rate** (deploys that cause incidents): under 15% = elite, over 30% = low.

These come from the DORA research (Dr. Nicole Forsgren, Jez Humble). Pipelines are the lever for the first two; testing + rollback are the lever for the last two.

### Speed tactics

- **Cache aggressively.** Build cache, dependency cache, Docker layer cache, test-results cache.
- **Parallelize.** Independent jobs run in parallel. Linters, tests, builds — fan out.
- **Test selection.** Run only tests affected by the diff (Bazel, Pants, Nx, Turborepo).
- **Trim dependencies.** A 4-minute `npm install` is 4 minutes you'll never get back.
- **Right-size runners.** A bigger runner is often cheaper than a slow build.
- **Fail fast.** Cheap checks (lint, typecheck) before expensive ones (e2e). One failure stops the rest.
- **Reuse artifacts.** Build the image once; tag it for every environment.

Aim for **under 15 minutes** push → staging for app changes. Under 1 hour for monorepos with heavy tests is acceptable. Anything over an hour for routine work is a productivity tax.

---

## 9. Secrets in CI/CD

CI systems need credentials to deploy. They also leak secrets faster than anything else.

Practices:

- **OIDC federation** for cloud credentials (GitHub Actions ↔ AWS/GCP/Azure). No static keys.
- **Secrets in a secrets manager** (Vault, AWS Secrets Manager, GitHub encrypted secrets, Doppler), pulled at job time.
- **Scoped access**. Each pipeline gets only what it needs. The staging pipeline can't deploy to prod.
- **Audit logs** on secret reads and pipeline runs.
- **Never echo secrets in logs.** CI providers usually mask configured secrets, but composite commands can leak.
- **Rotate often.** Static keys, when unavoidable, rotate quarterly minimum.
- **No secrets in PR previews.** Forks shouldn't have access to your prod secrets — limit which workflows run on `pull_request` from forks.

---

## 10. Environments — and how they relate

A typical promotion path:

```
local (dev) → CI ephemeral → integration → staging → canary → production
```

- **Local** — developer's machine; closest to production via containers/docker-compose.
- **CI ephemeral** — spun up per-PR, torn down after merge. Preview environments are gold for design review and stakeholder testing.
- **Integration / dev** — shared, less stable, integration with other services' mainline.
- **Staging** — production-shaped, "last gate" environment. Same artifacts, scaled down.
- **Canary** — small slice of production traffic on the new version. See [Canary →](./canary.md).
- **Production** — the real one.

**Avoid environment proliferation**. Every environment is overhead — different data, different secrets, different access. The fewer the better, with ephemeral PR environments handling the long tail.

---

## 11. Quality gates and policy

Gates are CI checks that **block** merges or deploys when violated:

| Gate | What it catches |
|---|---|
| Unit tests must pass | Logic regressions |
| Coverage above threshold (carefully) | Untested code paths |
| Linter clean | Style + simple bugs |
| Type checker clean | Type errors |
| Security scan clean (no HIGH/CRITICAL CVEs) | Vulnerable dependencies |
| Image scan clean | Vulnerable base images |
| Policy as code (OPA, Sentinel, Checkov) | Insecure IaC |
| License check | GPL in proprietary codebase |
| SBOM generated and signed | Supply-chain provenance |
| Required reviewers approved | Code review enforcement |
| Required CI checks green | All gates passed |

Make these checks **blocking** for production deploys. Optional gates get ignored. The deploy must not happen if the artifact didn't pass.

A subtle anti-pattern: code coverage thresholds gamed by useless tests. Coverage is a poor terminal goal; use it as a leading indicator, not a contractual requirement.

---

## 12. Monorepos and CI

Monorepos (Google-style, or modest with Nx/Turborepo/Pants/Bazel) have specific CI challenges:

- **Selective testing** — don't run the entire test suite on every PR. Affected-graph computation is non-trivial.
- **Selective deploys** — only deploy services whose code (or transitive deps) changed.
- **Distributed caching** — remote build cache (Bazel Remote, Turborepo Remote Cache) saves hours.
- **Merge queues** — `merge-queue` (GitHub), Bors, Aviator — serialize merges through a single CI run to avoid "main is broken" flapping.
- **Code ownership** — `CODEOWNERS` to require domain reviewers per directory.

Polyrepos sidestep these problems and create others (cross-repo refactors, version skew). There's no universal right answer; Google, Meta, Twitter run monorepos; Netflix, Spotify run polyrepos. Pick by team shape and tooling appetite.

---

## 13. Rollback — the most important capability

A pipeline that doesn't enable fast rollback is incomplete. Properties to test:

- **One-click rollback.** A button or command, not a multi-step runbook.
- **Image-level rollback.** Re-deploy the previous image, no rebuild.
- **Database safety.** Schema migrations don't break rollback. Use expand-contract (see [Migrations →](../04-databases/migrations.md)).
- **Feature flag kill switch.** Instant rollback for code-path issues without redeploy.
- **Auto-rollback on metric breach.** Argo Rollouts / Spinnaker auto-rollback on SLO violation during canary.
- **Rollback verified weekly.** A rollback that hasn't been tested is a rollback that doesn't work.

Mean Time To Restore is the metric. If your "rollback" requires a Slack thread, a release manager, and a 45-minute change-management process, your reliability ceiling is set by that, not by your tests.

---

## 14. Common Mistakes / Anti-Patterns

- **CI green, prod broken.** Tests passed but didn't cover the failure mode. Invest in integration / contract tests, not coverage theater.
- **No rebuild discipline — different artifact in staging vs prod.** "We tested the staging build but built a fresh one for prod." Build once, deploy many times.
- **Long-lived branches.** Integration debt accumulates. Trunk-based or bust.
- **Manual deploys hiding behind "automation."** `make deploy` from a laptop is not a pipeline.
- **No pinning.** Floating tags (`:latest`), unpinned Action versions, unpinned dependencies — every build is different.
- **Secrets in code.** Even rotated, even encrypted, code is the wrong place.
- **No environment parity.** Staging uses a different DB engine, different cache, different LB — surprises in prod become routine.
- **Slow pipelines tolerated for years.** Productivity dies a death of a thousand cuts.
- **No deploy event in metrics.** When p99 spikes, you can't tell if it's a deploy or a real event.
- **One pipeline per repo, never reviewed.** Pipelines are code. Treat them like code.
- **Test data in version control with secrets.** Anonymize, generate, or seed; don't dump.
- **`kubectl apply -f` against prod from a laptop.** GitOps or push from CI; never humans.
- **CD without good observability.** You shipped, then learned nothing for hours.
- **Coupling deploys across services.** If service A and service B must deploy together, you have one service.
- **Skipping staging for "small" changes.** Murphy's law. Always run the full path.
- **No merge queue for hot monorepos.** "Main is broken" becomes constant.

---

## 15. Cheat Card

```
PURPOSE   Turn every commit into a tested, signed artifact and
          (eventually) a production deploy — automatically, fast,
          reversible.

CI            build → lint → test → package → scan → sign
CD            dev → staging → canary → prod  (with gates)

PIPELINE PROPS
  Fast        target <15 min commit → staging
  Determin.   same input → same artifact
  Reversible  one-click rollback
  Observable  deploy events + bake metrics

NON-NEGOTIABLES
  Trunk-based dev, short-lived PRs
  Build once, deploy many times (same artifact)
  OIDC creds, no static cloud keys in CI
  Sign + SBOM + scan every image
  Manual approval (or strong auto-gates) for prod
  Auto-rollback on SLO breach during canary

TOOLS (pick by ecosystem)
  GitHub Actions  · GitLab CI  · CircleCI  · Buildkite
  Argo CD / Flux  (GitOps)
  Argo Rollouts / Spinnaker  (progressive delivery)
  Bazel / Turborepo / Nx  (monorepo speed)

METRICS THAT MATTER (DORA)
  Lead time for changes        elite: minutes–hours
  Deployment frequency         elite: multi-daily
  Time to restore (MTTR)       elite: <1 hour
  Change failure rate          elite: <15%

PITFALLS
  Long branches, big PRs, manual deploys
  No rebuild discipline — fresh build per env
  No deploy event in metrics
  Slow pipelines accepted as fact
  Secrets in code or logs
  Floating tags / unpinned anything
  No tested rollback path

RULE   The pipeline is the product. Invest in it like one.
       Fast, deterministic, signed, reversible — in that order.
```

---

## 16. Resources

### Books
- *Continuous Delivery* — Jez Humble & David Farley. The canonical text.
- *Accelerate* — Forsgren, Humble, Kim. The DORA research.
- *Continuous Deployment* — Valentina Servile. Hands-on patterns.
- *The DevOps Handbook* — Kim, Humble, Debois, Willis.

### Documentation
- **GitHub Actions** — <https://docs.github.com/en/actions>
- **GitLab CI/CD** — <https://docs.gitlab.com/ee/ci/>
- **Argo CD** — <https://argo-cd.readthedocs.io>
- **Spinnaker** — <https://spinnaker.io>
- **SLSA** — Supply-chain levels: <https://slsa.dev>
- **Cosign / Sigstore** — <https://docs.sigstore.dev>

### Articles
- "Continuous Integration" — Martin Fowler: <https://martinfowler.com/articles/continuousIntegration.html>
- "Trunk-Based Development" — <https://trunkbaseddevelopment.com>
- "How we deploy" — GitHub engineering: <https://github.blog/engineering/>
- "How Stripe ships" — Stripe engineering blog.
- DORA reports — <https://dora.dev>

### Videos
- *Continuous Delivery* channel (Dave Farley): <https://www.youtube.com/@ContinuousDelivery>
- *DevOps Days* talks.
- ByteByteGo — "CI/CD Pipeline Explained."

### Tools
- **GitHub Actions, GitLab CI, CircleCI, Buildkite, Jenkins** — CI orchestrators.
- **Argo CD, Flux** — GitOps controllers.
- **Argo Rollouts, Flagger** — progressive delivery.
- **Spinnaker, Harness** — heavyweight CD.
- **Buildkit, Kaniko, ko** — container builders.
- **Trivy, Grype, Snyk** — image scanning.
- **Cosign, Notary v2** — signing.
- **Syft, CycloneDX** — SBOMs.
- **Renovate, Dependabot** — dependency PRs.
- **Atlantis, Spacelift, Env0** — IaC pipelines.
- **k6, Locust** — load tests in CI.
- **Bazel, Pants, Buck, Nx, Turborepo** — monorepo build tools.

### Adjacent reading
- [Containers (Docker) →](./docker.md)
- [Container Orchestration (Kubernetes) →](./kubernetes.md)
- [Blue-Green Deployment →](./blue-green.md)
- [Canary Deployment →](./canary.md)
- [Rolling Deployment →](./rolling.md)
- [Feature Flags & Dark Launches →](./feature-flags.md)
- [Infrastructure as Code →](./iac.md)
- [Immutable Infrastructure →](./immutable-infra.md)
- [Database Migrations at Scale →](../04-databases/migrations.md)
- [Logging Best Practices →](../13-observability/logging.md)
- [SLA, SLO, SLI, Error Budgets →](../11-reliability/sla-slo-sli.md)

---

*Previous:* [← Infrastructure as Code](./iac.md)  |  *Next:* [Immutable Infrastructure →](./immutable-infra.md)
