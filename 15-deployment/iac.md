# Infrastructure as Code (Terraform, Pulumi, CloudFormation)

> **TL;DR** — **Infrastructure as Code (IaC)** means defining cloud resources — networks, servers, databases, IAM roles, DNS records, Kubernetes clusters — in declarative source files that live in version control. The IaC tool reads the desired state from your code, compares it to the actual state in the cloud, and applies the diff. You get **reviewable, repeatable, auditable** infrastructure instead of "what did Bob click in the console six months ago." The dominant choices: **Terraform** (HCL, multi-cloud, the de facto standard), **Pulumi** (real programming languages — Python, TypeScript, Go), and **CloudFormation** (AWS-native, less popular but tightly integrated). The honest take: **Terraform won the war** for general-purpose IaC; use it unless you have a specific reason not to. The two enduring hard problems aren't the tools — they're **state management** and **drift**, and every IaC story eventually wrestles with both.

---

## 1. The big picture

Before IaC, infrastructure changed through web consoles and CLIs. The result was unreproducible environments, undocumented dependencies, and outages that began with "I have no idea how this was set up."

IaC inverts the model:

```
   ┌──────────┐    plan      ┌──────────┐   apply    ┌────────────┐
   │  Code    │ ───────────► │  State   │ ─────────► │  Cloud     │
   │ (HCL/TS) │              │  diff    │            │ resources  │
   └──────────┘              └──────────┘            └────────────┘
        ▲                                                   │
        │                  refresh / drift detect           │
        └───────────────────────────────────────────────────┘
```

You describe the **desired state** in code. The tool calculates the **diff** between desired and actual. You review the diff. You apply it. The state file becomes the source of truth that lets the tool reason about what already exists.

The lasting wins of IaC:

- **Reviewable in pull requests.** Infrastructure changes show up as diffs that humans can read and reason about.
- **Reproducible.** Spin up dev, staging, prod from the same code with different variables.
- **Versioned.** Git history shows when a security group rule was added and why.
- **Recoverable.** "The region went down" becomes a `terraform apply` to a new region, not a panicked weekend.
- **Composable.** Modules and providers let you reuse patterns.

---

## 2. The landscape

| Tool | Approach | Language | Best for | Notes |
|---|---|---|---|---|
| **Terraform / OpenTofu** | Declarative | HCL | General-purpose, multi-cloud | De facto standard. OpenTofu is the open-source fork. |
| **Pulumi** | Declarative via imperative | Python, TS, Go, C#, Java | Teams that want real code | Same desired-state model under the hood. |
| **AWS CloudFormation** | Declarative | YAML/JSON | AWS-only shops | Tight AWS integration, slow, verbose. |
| **AWS CDK** | Imperative compiles to CFN | Python, TS, etc. | AWS teams who want code | Generates CloudFormation. |
| **Azure Bicep** | Declarative | Bicep DSL | Azure-only shops | Successor to ARM templates. |
| **Google Cloud Deployment Manager** | Declarative | YAML/Python | GCP-only (waning) | Largely superseded by Terraform on GCP. |
| **Ansible / Chef / Puppet / Salt** | Procedural config mgmt | YAML/DSL | OS-level config | Different layer: config of *machines*, not provisioning of *cloud*. |
| **Crossplane** | Kubernetes CRDs as IaC | YAML | K8s-native shops | Manages cloud resources as K8s objects. |

**OpenTofu** is the Linux Foundation fork of Terraform created after HashiCorp switched Terraform to the Business Source License (BSL) in 2023. It's API-compatible. Most teams that aren't already paying for Terraform Cloud are moving to OpenTofu.

For this page we'll mostly use Terraform/OpenTofu syntax since it dominates the market — but the patterns translate.

---

## 3. Terraform in 90 seconds

A minimal example: an AWS S3 bucket and an IAM policy:

```hcl
terraform {
  required_version = ">= 1.6"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
  }
  backend "s3" {
    bucket         = "acme-tf-state"
    key            = "prod/web.tfstate"
    region         = "us-east-1"
    dynamodb_table = "acme-tf-lock"
    encrypt        = true
  }
}

provider "aws" {
  region = var.region
}

variable "region" {
  type    = string
  default = "us-east-1"
}

resource "aws_s3_bucket" "uploads" {
  bucket = "acme-uploads-${terraform.workspace}"
  tags = {
    Environment = terraform.workspace
    Team        = "platform"
  }
}

resource "aws_s3_bucket_public_access_block" "uploads" {
  bucket                  = aws_s3_bucket.uploads.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

output "bucket_name" {
  value = aws_s3_bucket.uploads.id
}
```

Workflow:

```bash
terraform init     # download providers, configure backend
terraform plan     # show what will change (the diff!)
terraform apply    # execute the plan
terraform destroy  # tear down everything
```

The **plan** is the most important command. Always read the diff. Always.

---

## 4. State — the part that breaks teams

Terraform's state file is the canonical record of *what Terraform knows about the world*. It maps resource addresses in your code (`aws_s3_bucket.uploads`) to actual cloud resource IDs (`acme-uploads-prod-abc123`). Without it, Terraform doesn't know what it created.

State is also:

- **Sensitive.** It contains passwords, private keys, and full resource configuration. Never commit `terraform.tfstate` to git.
- **Concurrent-unsafe by default.** Two engineers running `apply` at once corrupt it.
- **The single point of truth.** Lose the state, and Terraform thinks everything is new and tries to recreate it.

### Remote state + locking

For any non-trivial team:

| Backend | Lock mechanism |
|---|---|
| **AWS S3 + DynamoDB** | DynamoDB conditional write |
| **Terraform Cloud / HCP Terraform** | Built-in |
| **GCS** | Built-in (GCS native) |
| **Azure Blob** | Lease |
| **Consul / etcd** | Built-in |
| **Spacelift / Env0 / Atlantis** | Platform-managed |

Setup once, ignored forever. Without it, you'll eventually corrupt state and have a bad day.

### State splitting

One giant state file is a footgun:

- `terraform plan` takes minutes (must refresh every resource).
- One typo can destroy everything.
- Multiple teams can't work in parallel.

Split state by **blast radius**:

```
infra/
  network/           # VPC, subnets, peering — rarely changes
  data/              # RDS, ElastiCache, S3 — slow-changing
  compute/           # EC2 / EKS / ECS — moderate
  apps/              # per-service infra — fast-changing
```

Each directory is its own state. Cross-state references use **remote state data sources** or **outputs**.

### Workspaces and per-env state

Terraform supports `workspaces` for shared state files. For real environments (dev/staging/prod), use **separate state files** (different backend keys), not workspaces. Workspaces are convenient for personal/temporary environments; they're brittle for permanent ones.

```
backend "s3" {
  key = "${var.env}/network.tfstate"  # Pseudo — real syntax uses var passthrough
}
```

In practice, a `terragrunt.hcl` or directory-per-env pattern works better than Terraform workspaces.

---

## 5. Drift — the other hard problem

Drift = actual cloud state ≠ Terraform's recorded state. Sources:

- Console clicks. Someone "just fixed it" in the UI.
- Auto-managed resources (autoscaling group changes, manually scaled services).
- Out-of-band tools (other IaC stacks, cloud-managed defaults).
- Provider bugs.

Drift detection: `terraform plan` against a clean working directory will show diffs even if you didn't touch the code. CI runs `plan` on a schedule and alerts if anything diverges.

Reactions to drift:

- **Reconcile**: bring code in line with reality (or vice versa).
- **Ignore intentional drift**: `lifecycle { ignore_changes = [tags["LastTouched"]] }` for fields managed elsewhere.
- **Lock the door**: SCPs / IAM policies that prevent direct console writes for IaC-managed resources.

The discipline that prevents drift: **if it's managed by Terraform, you never edit it outside Terraform**. No console exceptions. No "just this once." The moment you allow exceptions, the value of IaC erodes.

---

## 6. Modules — reuse without templating hell

A module is a directory of `.tf` files. You can call it like a function:

```hcl
module "service" {
  source = "git::https://github.com/acme/tf-modules.git//service?ref=v3.2.0"

  name        = "checkout-api"
  image       = "ghcr.io/acme/checkout@sha256:..."
  replicas    = 6
  cpu         = "500m"
  memory      = "1Gi"
  cluster_id  = module.cluster.id
  vpc_id      = data.terraform_remote_state.network.outputs.vpc_id
}
```

Good module hygiene:

- **Version pinned.** `?ref=v3.2.0` not `?ref=main`.
- **Single purpose.** A module that creates "a service" is fine. A module that creates "the entire platform" is a maintenance nightmare.
- **Inputs validated.** Use `variable` blocks with types, defaults, and `validation` rules.
- **Outputs documented.** Other modules consume them; treat them as a public API.
- **Tested.** `terratest`, `kitchen-terraform`, or simple plan-diff tests.

Common pattern: a **`tf-modules`** repo with reusable modules (`vpc`, `eks-cluster`, `service`, `rds-postgres`), each tagged for versions. Application repos consume them. Module changes flow through the same review and release process as any other code.

---

## 7. Pulumi vs Terraform — real languages

Pulumi looks like Terraform's underbelly but with real code on top:

```typescript
import * as aws from "@pulumi/aws";

const bucket = new aws.s3.Bucket("uploads", {
  bucket: `acme-uploads-${pulumi.getStack()}`,
  tags: { Environment: pulumi.getStack(), Team: "platform" },
});

new aws.s3.BucketPublicAccessBlock("uploads-block", {
  bucket: bucket.id,
  blockPublicAcls: true,
  blockPublicPolicy: true,
  ignorePublicAcls: true,
  restrictPublicBuckets: true,
});

export const bucketName = bucket.id;
```

Pulumi advantages:

- **Real conditionals, loops, types.** No `count`, no `for_each` workarounds.
- **IDE support.** Autocomplete, refactoring, type-checking.
- **Reusable patterns are just functions.**
- **Tests in your normal test framework.**

Terraform advantages:

- **Larger ecosystem.** Far more modules, providers, examples, hires.
- **Simpler mental model.** HCL is easier to read for non-engineers.
- **Drift is easier to reason about.** Imperative code that constructs declarative state can hide.

For teams of strong engineers building platform tooling, Pulumi often wins. For broad organizational use, Terraform's lower learning curve and bigger community usually win. AWS CDK is a third option (TypeScript/Python → CloudFormation) with the AWS-only ecosystem behind it.

---

## 8. Provider model

Terraform providers translate HCL to API calls. Major providers:

- **aws**, **azurerm**, **google** — the big three clouds.
- **kubernetes**, **helm** — manage K8s objects.
- **github**, **gitlab** — manage repos, teams, branch protections.
- **datadog**, **pagerduty** — observability and on-call config.
- **cloudflare**, **fastly** — DNS, CDN.
- **vault** — secrets.
- **mongodbatlas**, **snowflake**, **databricks** — SaaS-managed data systems.

The provider ecosystem is the real moat. Almost any product with a public API has a Terraform provider; that's why Terraform won.

### Pin providers

```hcl
required_providers {
  aws = { source = "hashicorp/aws", version = "~> 5.30" }
}
```

`~> 5.30` means "5.30.x but not 5.31 or 6.0." Pin tight enough to avoid surprise breaking changes, loose enough to get bug fixes.

---

## 9. Patterns that scale

### One repo, many states (a.k.a. mono-repo platform)

```
infra/
  modules/                  # shared modules
  environments/
    prod/
      network/
      data/
      compute/
      apps/
        checkout/
        billing/
    staging/
      ...
```

Each leaf directory is a state. CI runs plans on the changed directories per PR. Tools like **Atlantis**, **Spacelift**, **Env0**, or **Terraform Cloud** automate the PR → plan → apply flow.

### Many repos (per-team / per-service IaC)

Each team owns the Terraform for their service in their service repo. Platform team owns shared infra (network, clusters, IAM baselines) in a platform repo. Boundaries follow ownership.

### GitOps for IaC

PR → CI runs `plan`, posts the diff as a PR comment → reviewer approves → merge → CI runs `apply`. The git log is the audit log. No human runs `apply` from their laptop.

### Policy as code

**Sentinel** (HashiCorp), **OPA / Conftest**, **Checkov**, **tfsec**, **Terrascan** — scan plans for policy violations before apply:

- "No S3 bucket without encryption."
- "No security group with `0.0.0.0/0` on port 22."
- "RDS instances must have backups enabled."
- "All resources must have an `Owner` tag."

Run these in CI. Fail the PR if violated. Far more reliable than humans remembering rules.

---

## 10. Secrets in IaC

Secrets in Terraform state are an open problem. The state file holds every resource attribute, including DB passwords and access keys. Options:

- **Don't put secrets in IaC.** Generate them at apply time via a provider (random_password, aws_secretsmanager_secret_version), pass to the resource that needs them, retrieve through a runtime mechanism.
- **Use a secrets manager.** Provision the secret resource in Terraform, but never the value. The app reads from Vault / AWS Secrets Manager / GCP Secret Manager at runtime.
- **Encrypt state at rest.** S3 backend with KMS encryption. Mandatory.
- **Restrict state access.** IAM permissions on the state bucket — only CI and platform engineers.
- **Use SOPS or git-crypt.** For files that must be in git (Helm values, K8s manifests).

See [Secrets Management →](../12-security/secrets-management.md).

---

## 11. Cost and quotas

IaC makes it shockingly easy to provision expensive things. A typo can stand up 100 c5.24xlarge instances and bill you $200/hour.

Practices:

- **Cost estimation in PRs.** Tools like **Infracost** annotate the PR with the diff cost.
- **Budgets and alerts** in the cloud account. Hard caps for non-prod.
- **Resource limits.** Terraform variable validation. Required tags for cost allocation.
- **Sandbox accounts.** Engineers play in their own account; production is a different blast radius.

Quotas (cloud-side limits) are the other surprise. Provision an EKS cluster with 30 NAT gateways and discover the regional quota is 5. Always check quotas; raise them ahead of large rollouts.

---

## 12. CI/CD for IaC

A working pipeline:

```
PR opened
  → terraform fmt -check
  → terraform validate
  → terraform plan (no apply)
  → policy checks (Checkov, OPA)
  → cost diff (Infracost)
  → comment plan as PR comment

PR approved & merged
  → terraform apply (idempotent)
  → record state lock release
  → post completion notification

Nightly
  → terraform plan against all stacks
  → alert on unexpected drift
```

Run apply in a dedicated CI environment with the right credentials, never on engineer laptops. Use OIDC federation (GitHub Actions → AWS, GitLab → AWS) so no long-lived credentials sit anywhere.

See [CI/CD Pipelines →](./cicd.md).

---

## 13. Common Mistakes / Anti-Patterns

- **Committing `.tfstate` to git.** Secrets in git history. Use a remote backend, always.
- **No state locking.** Two concurrent applies corrupt the state file.
- **One giant state file.** Slow plans, big blast radius, no team isolation.
- **Console edits to IaC-managed resources.** Drift, then surprise destruction on next apply.
- **Unpinned provider versions.** A provider minor release breaks `apply` in the middle of an incident.
- **Unpinned module versions (`?ref=main`).** Same problem, internal.
- **Treating Terraform as configuration management.** Terraform provisions cloud resources; Ansible/Chef configure the OS inside them. Different tools, different layers.
- **Ignoring the plan.** "Just press apply." This is how 50 services get deleted.
- **Hardcoded values.** Region, account ID, names — everything gets parameterized eventually. Start that way.
- **Massive PRs.** "Refactor all our Terraform" → 4,000-line plan → no one can review it.
- **No drift detection.** First sign of drift is the outage.
- **No backup of state.** State backend has versioning enabled? It should.
- **Secrets in `.tfvars` checked into git.** Use environment variables, vault, or encrypted files.
- **`terraform destroy` on production by accident.** Use lifecycle rules (`prevent_destroy = true`), separate credentials, and CI gates.
- **Resources outside Terraform's knowledge.** "I'll just clickops this one thing." Now you have drift forever.
- **Mixing imperative orchestration (loops, scripts) with declarative resources outside the IaC tool.** Either commit to declarative or use a real programming language (Pulumi).

---

## 14. Cheat Card

```
PURPOSE   Define cloud infrastructure in code so it's reviewable,
          repeatable, auditable, and recoverable from git.

TOOL CHOICE
  Terraform / OpenTofu  →  default; multi-cloud; biggest ecosystem
  Pulumi                →  real languages; great for engineers
  AWS CDK               →  AWS-only; compiles to CloudFormation
  CloudFormation        →  AWS-native, verbose, slow
  Crossplane            →  K8s-native; cloud resources as CRDs

LOOP
  code (HCL/TS) → plan (diff) → review → apply → state updated

CORE OBJECTS
  Resource    — a thing in the cloud
  Provider    — the API translator (aws, gcp, k8s, github, ...)
  Module      — reusable composition unit
  State       — what Terraform knows about the world
  Backend     — where state lives (S3, GCS, TF Cloud, ...)

NON-NEGOTIABLES
  Remote state with locking
  State encrypted at rest
  Pinned provider + module versions
  Plans posted as PR comments
  Policy checks (Checkov, OPA) in CI
  Apply runs in CI, not laptops
  Drift detection nightly

WHEN TO USE
  Any cloud workload above "one server"
  Any environment that needs to be reproducible
  Any team with more than one person making changes

WHEN NOT TO USE
  Pure OS-level config (use Ansible/Chef/Puppet)
  Things outside any IaC tool's coverage (manual + documented)
  One-shot experiments in a sandbox account

PITFALLS
  State in git, no locking, one giant state file
  Console edits → drift → surprise destruction
  Unpinned providers / modules / image refs
  Massive PRs nobody can review
  Secrets in state and tfvars
  prevent_destroy not set on critical resources

RULE   If it's in the cloud, it's in code. If it changed, it
       changed in a PR. If you don't trust the plan, don't apply.
```

---

## 15. Resources

### Books
- *Terraform: Up & Running* — Yevgeniy Brikman. The canonical intro.
- *Infrastructure as Code* — Kief Morris (3rd edition). Tool-agnostic patterns.
- *The Terraform Book* — James Turnbull.

### Documentation
- **Terraform** — <https://developer.hashicorp.com/terraform/docs>
- **OpenTofu** — <https://opentofu.org/docs/>
- **Pulumi** — <https://www.pulumi.com/docs/>
- **AWS CDK** — <https://docs.aws.amazon.com/cdk/>
- **CloudFormation** — <https://docs.aws.amazon.com/cloudformation/>
- **Crossplane** — <https://docs.crossplane.io>

### Articles
- "Patterns for Managing Source Code Branches" — Martin Fowler (applies to IaC repos too).
- "Terraform best practices" — Google Cloud: <https://cloud.google.com/docs/terraform/best-practices-for-terraform>
- "Production-grade Terraform" — Gruntwork blog.
- "An opinionated guide to Terraform" — multiple takes worth reading.

### Videos
- *HashiConf* keynotes / sessions.
- *Pulumi Up* / KubeCon Crossplane talks.
- ByteByteGo — "Infrastructure as Code Explained."

### Tools
- **Terragrunt** — wraps Terraform for DRY config and multi-env.
- **Atlantis** — PR-based Terraform automation, self-hosted.
- **Spacelift / Env0 / Terraform Cloud** — managed IaC platforms.
- **Infracost** — cost estimates in PR comments.
- **Checkov / tfsec / Terrascan / KICS** — static analysis.
- **OPA / Conftest / Sentinel** — policy as code.
- **terraform-docs** — auto-generated module docs.
- **tflint** — linter for Terraform.
- **driftctl / Cloud Custodian** — drift detection.

### Adjacent reading
- [CI/CD Pipelines →](./cicd.md)
- [Immutable Infrastructure →](./immutable-infra.md)
- [Container Orchestration (Kubernetes) →](./kubernetes.md)
- [Secrets Management →](../12-security/secrets-management.md)
- [Multi-Region →](../10-scalability/multi-region.md)
- [Failover & Disaster Recovery →](../11-reliability/failover-dr.md)

---

*Previous:* [← Feature Flags & Dark Launches](./feature-flags.md)  |  *Next:* [CI/CD Pipelines →](./cicd.md)
