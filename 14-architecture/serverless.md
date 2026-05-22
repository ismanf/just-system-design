# Serverless / FaaS

> **TL;DR** — **Serverless** is a deployment and pricing model where the cloud provider runs your code on demand, scales it automatically, and charges per invocation (or per millisecond). The most common form is **Function-as-a-Service (FaaS)**: AWS Lambda, Google Cloud Functions, Azure Functions, Cloudflare Workers, Vercel/Netlify Functions. Broader serverless includes managed services like DynamoDB, S3, SQS, EventBridge, BigQuery, Aurora Serverless — anything where you don't manage capacity. The pitch is real: zero ops, true pay-per-use, instant scale-from-zero. The price is also real: **cold starts**, **vendor lock-in**, **distributed-systems debugging**, **15-minute execution limits**, and **non-trivial cost** at high QPS. Serverless wins decisively for event-driven, bursty, glue-code, and edge workloads. It's the wrong tool for long-lived processes, large stateful applications, and ultra-low-latency hot paths.

---

## 1. The Idea

Traditional deployment: you provision servers (VMs, containers), keep them running, pay for them whether they're working or not.

Serverless deployment: you upload code, define triggers, and the provider runs your code in response to events. You pay only for the compute time consumed.

```
Event (HTTP, queue message, scheduled time, DB change)
      │
      ▼
┌────────────────────────────────────────┐
│  Provider runtime: cold-start a worker │
│  → run your function                   │
│  → tear down (or keep warm)            │
└────────────────────────────────────────┘
      │
      ▼
You pay: invocation × duration × memory
```

You write the function; the provider does **everything else** — provisioning, scaling, patching, networking, monitoring scaffolding. Whether that "everything else" is a net win depends on your workload.

---

## 2. The Serverless Spectrum

"Serverless" is a marketing umbrella covering several patterns:

| Layer | Example | "Serverlessness" |
| --- | --- | --- |
| **FaaS** | AWS Lambda, GCP Cloud Functions, Azure Functions, Cloudflare Workers | Highest — your code, ephemeral |
| **Container-on-demand** | AWS Fargate, Cloud Run, Azure Container Apps | High — your container, scaled by provider |
| **Backend-as-a-Service (BaaS)** | Firebase, Supabase, Amplify | High for backend; you write little |
| **Managed services** | DynamoDB, S3, SQS, EventBridge, Kinesis, Step Functions | Pay-per-use; no servers to manage |
| **Serverless databases** | Aurora Serverless, Neon, PlanetScale, Cosmos DB serverless | Auto-scale DB capacity |
| **Edge serverless** | Cloudflare Workers, Vercel Edge, Fastly Compute@Edge | FaaS at global PoPs |

A typical "serverless" application stitches several of these together: API Gateway in front of Lambda functions reading/writing DynamoDB, with EventBridge wiring async flows. No EC2, no Kubernetes.

---

## 3. FaaS Mechanics

A Lambda-style function:

```python
def handler(event, context):
    user_id = event["queryStringParameters"]["user_id"]
    user = ddb.get_item(Key={"id": user_id})["Item"]
    return {
        "statusCode": 200,
        "body": json.dumps(user),
    }
```

The provider:

1. Receives the event (HTTP request, queue message, timer tick, etc.).
2. **Cold start** — if no warm worker is available, allocates a container, downloads your code, loads runtime, calls your handler.
3. **Warm invocation** — if a worker is available from a recent invocation, reuses it (typically up to ~15 min idle).
4. After the handler returns, the worker is held warm for a while; eventually torn down.

Concurrency model:
- Each worker handles **one event at a time** (Lambda; some platforms allow N-per-worker).
- Provider scales horizontally — 1000 simultaneous events = 1000 workers spun up.
- Account-level concurrency caps apply (default 1000 on AWS, raisable).

---

## 4. Cold Starts — The Defining Challenge

The first invocation on a new worker pays an initialization cost:

| Runtime | Typical cold start |
| --- | --- |
| **Cloudflare Workers (V8 isolates)** | < 10 ms |
| **Node.js (Lambda)** | 100–500 ms |
| **Python (Lambda)** | 200–800 ms |
| **Go (Lambda)** | 100–300 ms |
| **Java (Lambda, JVM cold)** | 500 ms – several seconds |
| **.NET (Lambda)** | 500 ms – several seconds |
| **GraalVM native image / SnapStart** | 50–200 ms |

A cold start affects:
- **First user** of a function after idle.
- **Spikes** that exceed warm-worker pool.
- **Versioned deploys** — each deploy starts cold.

Mitigations:
- **Smaller deployment packages** (tree-shake, native compile).
- **Provisioned concurrency** (AWS Lambda) — keep N workers warm at a price. Defeats some of the on-demand premise but is pragmatic for user-facing endpoints.
- **Worker reuse** in the handler — cache DB clients, JSON parsers at module scope.
- **SnapStart** (AWS, JVM) — pre-initialize and snapshot the VM.
- **Edge/V8-isolate runtimes** — Cloudflare Workers, Deno Deploy, Vercel Edge — designed to start in milliseconds.

For ultra-low-latency hot paths, FaaS is often the wrong fit. For everything else, the cold-start budget is manageable.

---

## 5. Execution Model — Constraints

Each FaaS platform has limits:

| Limit | AWS Lambda |
| --- | --- |
| Max execution time | 15 minutes |
| Max memory | 10 GB |
| Max ephemeral disk | 10 GB (`/tmp`) |
| Max deployment package | 250 MB (or 10 GB container image) |
| Concurrency per account | 1000 (raisable) |
| Payload size | 6 MB sync / 256 KB async |

Equivalent platforms have similar caps. Implications:

- **No long-running processes.** Batch jobs over 15 min must be broken up.
- **No background threads after return.** You can't fire-and-forget on a worker that's about to be torn down.
- **No local state across invocations.** Each invocation may run on a different worker.
- **Limited disk.** Don't write big files to `/tmp` and expect them next time.

These constraints are real and shape what fits serverless: short, stateless, event-driven work.

---

## 6. When Serverless Shines

Strong fits:

- **Event-driven workloads.** Process S3 uploads, queue messages, DB stream changes.
- **Glue code.** Tie cloud services together. Step Functions / EventBridge orchestrate them.
- **Bursty, unpredictable traffic.** Auth callbacks, webhooks, occasional jobs.
- **Cron / scheduled tasks.** No server idle 99% of the day.
- **APIs with low-to-medium QPS.** Up to ~hundreds RPS, Lambda + API Gateway is cheap and operationally simple.
- **Edge logic.** Cloudflare Workers / Vercel Edge for personalization, A/B, geo routing.
- **CI/CD jobs and ChatOps bots.**
- **Startups in MVP/early-stage.** Defer infra investment until you have product fit.

Companies running real systems on serverless: Coca-Cola (vending), Lego.com (e-commerce on AWS Lambda), iRobot (telemetry), Capital One (significant portions), countless SaaS startups.

---

## 7. When Serverless Is the Wrong Tool

Weak fits:

- **Sustained high-throughput services.** At constant ≥ ~1000 RPS, traditional containers are usually cheaper.
- **Ultra-low-latency hot paths.** Cold starts hurt p99.
- **Long-running tasks.** > 15 min, you'll need a different compute primitive (Fargate, EC2, Step Functions).
- **Heavy stateful processing.** WebSocket servers with millions of connections, in-memory caches, big ML model loading per request.
- **Predictable, steady workloads.** Pre-paid reservations on EC2 / GKE are cheaper.
- **Vendor-agnostic architectures.** Lambda code is portable; bindings to API Gateway, EventBridge, Step Functions, DynamoDB triggers are not.

The economic crossover: serverless gets expensive past a few hundred-million requests per month at typical durations. At that point, container-based compute often wins on $/request.

---

## 8. Architectural Patterns

### API behind a function
```
API Gateway → Lambda → DynamoDB
                    ↘ → SQS
```
Default shape for small services and SaaS.

### Async event processing
```
S3 upload → Lambda (process file) → DynamoDB / S3 output
SQS message → Lambda → downstream
DynamoDB stream → Lambda (CDC) → events
```
Most "glue" workloads.

### Step Functions / Workflow
```
[start] → Lambda A → choice
                  → Lambda B
                  → Lambda C → [end]
```
For multi-step flows with retries, error handling, parallelism, human approvals. Better than nested Lambdas calling Lambdas.

### Fan-out
```
EventBridge / SNS → Lambda A
                  → Lambda B
                  → Lambda C
```
Decoupled fan-out to many consumers.

### Edge function
```
User request → Cloudflare Worker → cache or origin
```
Personalization, A/B, header rewrites, auth checks, all in <10 ms at the edge.

### Hybrid serverless + container
Keep the persistent / hot path on EC2/ECS/Fargate. Use Lambda for bursty events around it.

---

## 9. State and Storage

Functions are stateless. State lives elsewhere:

- **DynamoDB** — the canonical serverless DB; pay-per-request mode aligns.
- **Aurora Serverless v2 / Neon / PlanetScale** — Postgres/MySQL that auto-scales.
- **S3** — durable blob storage.
- **ElastiCache / Upstash Redis (serverless)** — for cache.
- **EventBridge / SQS / Kinesis / SNS** — eventing / queuing.
- **DynamoDB Streams / Kinesis** — CDC.

A serverless app is often **mostly orchestration of managed services** — your Lambda just glues them together. That's the model.

Connection pooling for SQL DBs is a notorious pain (Lambda's per-instance connections explode at scale). Solutions: RDS Proxy, Aurora Data API, Neon's pooler, PlanetScale's HTTP API.

---

## 10. Observability and Debugging

Serverless ops looks different:

- **No SSH; no `top`.** Everything via metrics, logs, traces.
- **CloudWatch / Stackdriver / Application Insights** default; OpenTelemetry preferred for portability.
- **Distributed tracing essential** — Lambda → Lambda → DynamoDB chains need [Distributed Tracing →](../13-observability/tracing.md).
- **Local development** via SAM, Serverless Framework, LocalStack, sst, Wing — imperfect but improving.
- **CI/CD** typically with the Serverless Framework, AWS SAM, CDK, or Terraform.
- **Failure modes** unique to serverless: throttling at concurrency limits, timeout vs out-of-memory, async retry semantics differ per service.

Investing in observability is non-optional. A `console.log` debugging strategy stops scaling at ~10 functions.

---

## 11. Cost Model

Lambda pricing example (rough numbers):
- **Free tier:** 1M invocations + 400k GB-seconds per month.
- **After:** $0.20 per 1M invocations + $0.0000166667 per GB-second.

A function with 1 GB memory and 100 ms duration costs ~$0.0000017 per call. 1M calls/day at this profile ≈ $50/month. 100M calls/day ≈ $5,000/month.

Pitfalls:
- **API Gateway / EventBridge / Step Functions** add per-call costs.
- **DynamoDB on-demand** is convenient but pricey vs provisioned at steady load.
- **Egress** out of AWS still costs $0.09/GB.

At scale, do the spreadsheet vs containers. Many teams hit "serverless was supposed to be cheap" frustrations because they didn't model the full bill.

---

## 12. Security Considerations

Serverless changes the threat surface:

- **IAM per function.** Each Lambda has its own role; principle of least privilege is easier to enforce.
- **No long-running process to compromise** — short lifetimes limit some attacks.
- **But more attack surface** — many functions, many event sources, many integrations.
- **Secrets management** via AWS Secrets Manager / Parameter Store / Vault — not env vars in plaintext.
- **Cold-start side channels** — shared infrastructure risk in extreme threat models; mostly irrelevant for typical apps.
- **OWASP Serverless Top 10** documents the common pitfalls — event-data injection, broken authentication, insecure deployment, over-privileged functions.

See [Secrets Management →](../12-security/secrets-management.md) and [OWASP Top 10 →](../12-security/owasp-top-10.md).

---

## 13. The "Functions of Functions" Anti-Pattern

The temptation: split every responsibility into its own Lambda. Result:

- Hundreds of functions for a single application.
- Function-to-function calls via HTTP or async events.
- Deployments require coordinating dozens of artifacts.
- Latency adds up across hops.
- Debugging requires reconstructing the path across many functions.

This is essentially **nano-microservices** with worse tooling. Better:
- **One function per use case** (or per HTTP route).
- **Coarse functions** sharing common code via layers / packages.
- **Step Functions / workflows** for orchestration, not Lambda→Lambda chains.

---

## 14. Common Mistakes / Anti-Patterns

- **Cold-start denial.** "Just use Java without provisioned concurrency." First user has a 3-second wait.
- **Functions calling functions synchronously.** Latency stacks; failures cascade.
- **Massive deployment packages.** 200 MB zip = long cold start.
- **DB connection pool exhaustion.** Each Lambda instance opens its own DB connection; 1000 concurrent functions = 1000 connections. Use RDS Proxy or HTTP-based DBs.
- **Hardcoded environment behavior.** Functions assuming local disk persistence between calls.
- **Long-running batch via Lambda.** Hits 15-min limit. Use Step Functions, Fargate, or Batch.
- **Polling SQS in a Lambda.** Use the native SQS trigger (event source mapping).
- **No idempotency.** Async retries and at-least-once delivery duplicate work.
- **Single Lambda for the entire app** ("monolambda"). Cold-start cost amortized, but every change redeploys everything.
- **Per-route Lambda explosion.** Hundreds of tiny functions for tiny endpoints. Hard to maintain.
- **Ignoring observability.** Production becomes black box.
- **Vendor lock-in surprise.** Three years in, "let's move to GCP" → rewriting all the EventBridge/Step Functions bindings.
- **No load testing.** Serverless scales beautifully... until throttled at concurrency limits or backed by a DB that can't keep up.
- **Cost surprise.** Burst traffic hits, bill goes up 10× because no cap was set. Always set billing alarms.
- **Forgetting deploy lifecycle.** Lambda versions and aliases let you ship safely; not using them means each deploy is YOLO.

---

## 15. A Worked Example

A simple webhook receiver on AWS:

```
Webhook POST → API Gateway → Lambda receiver → SQS queue
                                                  │
                                                  ▼
                                          Lambda processor
                                                  │
                                                  ▼
                                             DynamoDB write
                                                  │
                                                  ▼
                                       EventBridge "OrderProcessed"
                                                  │
                                            ┌─────┴─────┐
                                            ▼           ▼
                                       Lambda email   Lambda analytics
```

Properties:
- Webhook receiver Lambda is small (< 100 lines), validates and queues.
- Processor Lambda handles business logic, writes DB, publishes event.
- Email and analytics Lambdas react to events independently.
- Total infra to provision: zero. Just code + IaC.
- Bursty traffic: scales transparently. 10 webhooks/min or 10,000 webhooks/min — same architecture.
- Cost at 1M webhooks/day: ~$50–$200 depending on durations.

This shape is extremely common for SaaS webhook receivers, image/video processing pipelines, IoT ingestion, audit log fan-out.

---

## 16. Cheat Card

```
SERVERLESS = provider runs your code on demand, scales auto, pay per use.

FAAS         AWS Lambda · GCP Functions · Azure Functions · Cloudflare Workers
SPECTRUM     FaaS · container-on-demand (Fargate, Cloud Run) ·
             BaaS (Firebase, Supabase) · managed services (DDB, S3, SQS) ·
             serverless DBs (Aurora SL, Neon, PlanetScale) · edge (Workers)

EXEC MODEL   one event per worker; provider scales out; cold start on first
             AWS Lambda limits: 15 min · 10 GB RAM · 250 MB pkg · 1000 conc.

COLD STARTS  worst on JVM/.NET, best on V8 isolates / Go.
             mitigate with provisioned concurrency, SnapStart, smaller pkgs,
             module-scope caching.

GOOD FITS
  event-driven · glue · bursty · scheduled · low/medium QPS APIs ·
  edge personalization · CI/CD bots · MVPs

BAD FITS
  sustained high QPS (cost crossover) · ultra-low-latency hot path ·
  long-running (> 15 min) · stateful long-lived connections ·
  vendor-portability requirements

PATTERNS
  API+Lambda+DDB · S3 trigger → Lambda → output · SQS/EventBridge fan-out ·
  Step Functions for orchestration (not Lambda→Lambda chains) ·
  edge function for personalization

STATE        live in DDB / S3 / Aurora SL / Redis / SQS — not in the function.

OBSERVABILITY  CloudWatch + OpenTelemetry + distributed tracing — mandatory.
SECURITY       IAM per function · secrets in Vault/SM · OWASP Serverless Top 10.
COST           model the bill end-to-end · set billing alarms.

ANTI-PATTERNS  monolambda · nano-functions · sync Lambda→Lambda chains ·
               cold-start denial · no idempotency · DB connection storms ·
               vendor lock-in surprise

RULE: serverless wins for event-driven and bursty.  Build hybrid systems.
       Always model cold-start latency and steady-state cost before adopting.
```

---

## 17. Resources

### Books
- *Serverless Architectures on AWS* — Peter Sbarski.
- *Programming AWS Lambda* — John Chapin & Mike Roberts.
- *Production-Ready Serverless* — Yan Cui.
- *Cloud Native Patterns* — Cornelia Davis.

### Documentation
- **AWS Lambda** — <https://docs.aws.amazon.com/lambda/>
- **Cloudflare Workers** — <https://developers.cloudflare.com/workers/>
- **GCP Cloud Functions** — <https://cloud.google.com/functions/docs>
- **Azure Functions** — <https://learn.microsoft.com/azure/azure-functions/>
- **OWASP Serverless Top 10** — <https://owasp.org/www-project-serverless-top-10/>

### Articles
- "Serverless Architectures" — Mike Roberts: <https://martinfowler.com/articles/serverless.html>
- "When Not to Use Lambda" — Yan Cui (theburningmonk).
- "Serverless economics" — Bernardi / Cloudflare.
- "Cold starts in detail" — Mikhail Shilkov blog series.

### Videos
- "AWS Lambda Internals" — re:Invent talks.
- "Why we love Cloudflare Workers" — various startups.
- ByteByteGo — "Serverless explained".

### Tools
- **Frameworks:** AWS SAM, Serverless Framework, AWS CDK, sst, Wing, Architect, Pulumi.
- **Local dev:** LocalStack, SAM CLI, Wrangler (Workers), Functions Framework.
- **Observability:** AWS X-Ray, Lumigo, Datadog Serverless, Honeycomb, New Relic.
- **DB pooling:** AWS RDS Proxy, Neon pooler, PlanetScale HTTP, Supabase.

### Adjacent reading
- [Monolith vs Microservices vs Serverless →](../01-foundations/monolith-microservices-serverless.md)
- [Microservices Architecture →](./microservices.md)
- [Event-Driven Microservices →](./event-driven-microservices.md)
- [Edge Computing →](../19-advanced/edge-computing.md)
- [API Gateway →](../03-apis/api-gateway.md)
- [Auto-Scaling →](../10-scalability/auto-scaling.md)

---

*Previous:* [← Service-Oriented Architecture (SOA)](./soa.md)  |  *Next:* [Event-Driven Microservices →](./event-driven-microservices.md)
