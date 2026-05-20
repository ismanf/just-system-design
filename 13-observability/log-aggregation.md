# Centralized Log Aggregation (ELK, Loki, Splunk)

> **TL;DR** — **Centralized log aggregation** ships logs from every service to a single searchable system. The three dominant architectures are **ELK / OpenSearch** (full-text index everything — heavy, powerful), **Loki** (index labels only, store raw logs in object storage — cheap), and **Splunk** (commercial, comprehensive, expensive). Each makes a different trade-off between **search flexibility, cost, and operational complexity**. The collector tier (Fluent Bit, Vector, Fluentd, Promtail, OTel Collector) is increasingly more important than the backend — it normalizes, redacts, samples, and routes. Engineering the **storage tiers** (hot/warm/cold) and **retention** is where teams spend real money. Cardinality and free-form fields are the silent killers.

---

## 1. Why Centralize

Local logs in containers die with the container. Logs on individual VMs require SSH archaeology. Multi-service incidents require correlating events across hosts. **Centralization is non-negotiable** for any system bigger than a single VM.

A centralized pipeline gets you:

- **Unified search** across all services, all hosts.
- **Retention** independent of host lifecycle.
- **Correlation** by trace ID across services.
- **Compliance** — audit trails preserved.
- **Alerting** on log patterns.
- **Dashboards** — top errors, anomalies, business events.

```
┌────────────┐    ┌────────────┐    ┌────────────┐
│  service A │    │  service B │    │  service C │
└─────┬──────┘    └─────┬──────┘    └─────┬──────┘
      │ stdout          │ stdout          │ stdout
      ▼                 ▼                 ▼
┌─────────────────────────────────────────────────┐
│         Collector tier (per node)               │
│  Fluent Bit / Vector / Promtail / OTel          │
│     parse · enrich · redact · sample · batch    │
└──────────────────────┬──────────────────────────┘
                       │ OTLP / HTTP / TCP
                       ▼
              ┌─────────────────────┐
              │   Aggregator / Store│
              │   Loki / ELK /      │
              │   Splunk / Datadog  │
              └─────────┬───────────┘
                        │
                        ▼
                ┌──────────────┐
                │ Search / UI  │
                │ Grafana /    │
                │ Kibana /     │
                │ Splunk Web   │
                └──────────────┘
```

---

## 2. The Pipeline Stages

### Stage 1 — Emit
Apps log to **stdout/stderr** in structured JSON. The 12-factor pattern. No file rotation, no log directories per service.

See [Logging Best Practices →](./logging.md).

### Stage 2 — Collect
A **collector** runs per node (DaemonSet in Kubernetes) and:

- Reads container/file logs from the node.
- Parses fields (already JSON if you did stage 1 right).
- Enriches with metadata (`pod_name`, `namespace`, `host`, `region`, `cluster`).
- Redacts known sensitive patterns.
- Samples high-volume events.
- Batches.
- Ships to the aggregator (TLS, mTLS, signed).

### Stage 3 — Ingest & store
The aggregator receives batches and stores them — typically in a tiered architecture:

- **Hot tier:** recent logs (hours-days), indexed for fast search. Fast disks, expensive.
- **Warm tier:** older logs (weeks-months), still searchable but cheaper.
- **Cold tier:** archive (months-years), S3/GCS/Azure Blob, slower to query.

### Stage 4 — Query & alert
Engineers and tools query: ad-hoc search, dashboards, scheduled alerts on log patterns.

---

## 3. The Big Three Architectures

### ELK / OpenSearch — Index Everything

```
Elasticsearch  → distributed inverted index over JSON documents
Logstash       → original ingest/transform tier (now often replaced by Fluent Bit)
Kibana         → search UI and dashboards
Beats          → lightweight shippers (Filebeat, Metricbeat, etc.)
```

- **Strength:** any field is searchable, full-text on the message body, rich aggregations.
- **Cost:** disk + RAM for the index. ~3–10× the raw log size in storage; RAM-hungry.
- **Op load:** cluster management, shard rebalancing, JVM tuning.
- **OpenSearch:** AWS's fork of Elastic after Elastic's license change. Functionally the same; cheaper to operate in AWS.

ELK was the dominant stack 2014–2022; still very common. It's the right choice when **arbitrary search across all fields** matters more than cost.

### Loki — Label-Only Index

Grafana Labs' approach: don't index the log body. Index only **labels** (small set of key-value pairs identifying the stream), store the raw lines compressed in object storage (S3/GCS/Azure Blob).

```
Logs by stream:
  {service="api", env="prod", region="us-east"} → chunked, gzip'd in S3
```

Queries use LogQL — first filter by labels (cheap, indexed), then grep/regex the body (read from S3).

- **Strength:** very cheap storage (~10–100× cheaper than ELK), labels-only index keeps the system simple.
- **Cost:** queries that don't filter by labels can be slow; cardinality of labels still matters.
- **Op load:** modest.

Loki is the right choice for **high volume, moderate-frequency search**. Excellent for "metrics-like" log usage (count of errors, alerts on patterns) and acceptable for incident search if you have the labels right.

### Splunk — Commercial Heavy Hitter

- **Strength:** powerful query language (SPL), great UI, mature, full-featured.
- **Cost:** $$$$. Licensing was historically per GB ingested. Even after pricing changes, Splunk is famously expensive at scale.
- **Op load:** zero if SaaS; significant if self-hosted.

Splunk is dominant in enterprises with regulatory needs (SIEM, compliance) and budget. For greenfield, most teams now choose Loki/ELK/Datadog/etc. unless Splunk is mandated.

### The Hosted Alternatives

- **Datadog Logs** — easy ingest, integrated with metrics/APM, billed per GB.
- **AWS CloudWatch Logs** — default in AWS, slow search, cheap-ish ingest.
- **GCP Cloud Logging** — same idea, GCP-side.
- **Sumo Logic, Better Stack, New Relic Logs, Honeycomb (for events), Logz.io** — others.

Hosted is operationally cheap and dollar-expensive. For most teams under ~1 TB/day, hosted is the right answer. Above that, the savings of self-hosted Loki / OpenSearch can be massive.

---

## 4. The Collector Tier — Underrated

The collector is where most of your value is.

| Collector | Notes |
| --- | --- |
| **Fluent Bit** | C, tiny, fast, ubiquitous. Default in K8s. |
| **Vector** | Rust, very fast, modern, programmable transforms. |
| **Fluentd** | Ruby, older sibling of Fluent Bit. Plugin ecosystem. |
| **Promtail** | Loki's official agent. Discovers via Kubernetes labels. |
| **OpenTelemetry Collector** | Logs + metrics + traces unified. Future direction. |
| **Datadog Agent** | If you're already on Datadog. |
| **Filebeat / Logstash** | The ELK-native shippers. |

A modern collector does:

- **Source discovery** — find container logs by labels.
- **Parsing** — JSON, multi-line stack traces, custom regex.
- **Enrichment** — add `pod`, `namespace`, `host`, `cluster`, `region`, `env`, `version`.
- **Redaction** — regex out credit cards, SSNs, tokens.
- **Sampling** — drop verbose categories.
- **Routing** — different destinations per log type (security to SIEM, app to Loki, errors to Sentry).
- **Buffering** — disk-backed for backend outages.
- **Backpressure** — slow down sources when downstream is slow.

A well-configured collector can save 50%+ on log bills by dropping noise before ship.

---

## 5. Storage Tiering and Retention

```
Tier        Latency    Storage cost    Typical retention
─────       ───────    ─────────────   ─────────────────
Hot         < 1 s      $$$$            7–14 days
Warm        seconds    $$              30–90 days
Cold/Archive minutes   $               1–7 years
```

Most aggregators support automatic tier transitions (Elastic ILM, Loki S3 backends, Splunk SmartStore). Configure based on:

- **Compliance retention** — SOC 2, HIPAA, PCI all have minimums.
- **Investigation horizon** — most incidents resolved within 7 days; old logs rarely queried.
- **Audit log retention** — usually longer (1+ year).
- **Cost** — ingestion cost vs storage cost vs query cost.

Old logs in S3 cost ~$0.023/GB/month. The same in indexed ELK is 30–100× more.

---

## 6. Cardinality and Volume — The Twin Costs

Your log bill is a function of two things:

1. **Volume** (bytes ingested per day).
2. **Cardinality** (number of unique label combinations).

### Volume control

- Drop DEBUG/TRACE in production.
- Sample high-frequency low-value events (Redis GET, per-row iterations).
- Drop health-check requests (they're noise).
- Truncate large payloads.
- Aggregate repeated lines (`error X occurred 100 times in 1m` → one event with count).

### Cardinality control

- Loki labels: ~10 max per stream, low-cardinality values (service, env, pod).
- Don't put `user_id`, `trace_id`, `request_id` as Loki labels. Put them in the body (Loki) or as ES fields (no problem in ES, big problem in Loki).
- For ELK: control field mapping — `dynamic: false` to prevent random new fields blowing up the schema.

A common gotcha: a new log line adds a fresh field per request → after a week, the ES cluster has 100k fields and OOMs.

---

## 7. Search Patterns

Query strategies that work:

- **Filter by stream labels first.** `{service="api", env="prod"}` then grep.
- **Use trace_id when you have one.** Most precise way to reconstruct a single request.
- **Use error patterns sparingly.** Full-text search across days = slow and expensive in any backend.
- **Save common queries as panels** in your observability tool — dashboards beat re-typing.
- **Combine with metrics** — start at the metric anomaly, narrow to the time window, then query logs.

LogQL example (Loki):
```logql
{service="checkout", env="prod"} |= "error"
  | json
  | duration_ms > 1000
```

ES query DSL example (Kibana):
```json
{ "bool": {
  "filter": [
    { "term": { "service": "checkout" } },
    { "term": { "env": "prod" } },
    { "range": { "@timestamp": { "gte": "now-1h" } } },
    { "wildcard": { "message": "*timeout*" } }
  ]
}}
```

---

## 8. Alerting from Logs

Two modes:

### Convert logs to metrics, alert on metrics
Best practice. Use the collector to count or measure (Fluent Bit's filter plugins, Vector's transforms, ELK with `metricbeat`), emit a metric, alert via Prometheus/Alertmanager.

### Direct log alerts
Most aggregators support saved queries that trigger alerts ("send page if error count > N in 5 min"). Useful for low-volume, signal-rich events (security alerts, audit anomalies).

Don't over-rely on log-based alerts for high-volume metrics — logs are slower to ingest and more expensive to scan than metrics.

---

## 9. Security and Compliance Considerations

Centralized logs are a security treasure for attackers and a tool for defenders. Get these right:

- **Encryption in transit** (TLS between collectors and aggregators).
- **Encryption at rest** in the aggregator and S3 archive.
- **Access controls** — engineers can search; only specific roles can export.
- **Audit logs of who queried what** — yes, log your log queries.
- **PII redaction** at the collector — don't ship it.
- **Separate audit/security pipeline** — different stream, different storage, longer retention, immutability.
- **Tamper-evident archive** — write-once storage for compliance.

For SOC 2 / PCI DSS: assume auditors will read your logging architecture diagram. Make it defensible.

---

## 10. Operations of the Aggregator Itself

The aggregator is a production system. It needs:

- **Capacity planning** — ingest rate, queryable retention, query QPS.
- **Monitoring** — disk usage, indexing latency, query latency, dropped logs.
- **HA** — multiple replicas, multi-AZ, ideally multi-region for DR.
- **Backups** — for state that isn't in S3 (Elasticsearch snapshots).
- **Upgrades** — with maintenance windows; Elasticsearch major upgrades are real work.
- **Runbooks** — what to do when ingest backs up, when nodes fail, when retention runs over budget.

Forgetting that "the log system is also a system": classic recipe for "we can't see anything because our log system is down."

---

## 11. Worked Example — A Kubernetes Logging Stack

A modern open-source setup:

```
1. App pods log JSON to stdout.
2. Fluent Bit DaemonSet on every node:
   - Tail /var/log/containers/*.log.
   - Parse JSON.
   - Add labels: namespace, pod, container, node, region.
   - Redact known patterns.
   - Drop liveness/readiness logs and noisy debug.
   - Ship to Loki via push API.
3. Loki cluster:
   - Distributors → ingesters → store-gateway → S3.
   - Compactor merges chunks.
   - Retention: 30 days hot in S3, 1 year cold archive.
4. Grafana:
   - Datasource = Loki.
   - Dashboards: errors per service, request rate, top slow endpoints.
   - Alerts: error count > N in 5m → PagerDuty.
5. Audit logs:
   - Separate pipeline: app → Kafka → S3 (write-once) + Athena for query.
```

Cost at ~500 GB/day: ~$2–5k/month self-hosted; ~$30–80k/month equivalent on Splunk or Datadog. The savings fund a dedicated infra engineer.

---

## 12. Common Mistakes / Anti-Patterns

- **No central aggregation.** Logs in pods die with pods.
- **Unstructured / free-text logs.** Aggregator can't index meaningfully.
- **Ingesting everything DEBUG.** Crushes budget; signal lost in noise.
- **No PII redaction at the collector.** Compliance event waiting to happen.
- **High-cardinality fields indexed (esp. in Loki labels).** Cluster OOMs.
- **No retention policy.** Storage grows without bound.
- **Same retention for all logs.** Audit needs years; debug doesn't.
- **No tiering.** Paying hot prices for 1-year-old logs.
- **Manual log shipping (custom scripts).** Brittle, lossy, hard to debug.
- **Aggregator with no monitoring of itself.** Silent ingest gaps.
- **Skipping the collector tier** — apps push directly to the aggregator. Hard to upgrade, hard to redact centrally.
- **Confusing audit logs with operational logs.** Different retention, security, query patterns.
- **No alerting on log volume changes.** A misbehaving app spikes 100× and the bill explodes.
- **Saved-search alerts on slow queries.** Cron-killing the cluster.
- **Storing whole HTTP request/response bodies.** Cost + compliance disaster.
- **Trusting "Logs in S3 are immutable" without write-once mode.** S3 objects are deletable by default.
- **Ignoring time skew across hosts.** Timeline ordering subtly wrong. Use NTP; let the aggregator deduplicate by ingest time + event time.

---

## 13. Cheat Card

```
PIPELINE
  app stdout (JSON) → collector (per node) → aggregator → search UI

COLLECTORS    Fluent Bit · Vector · Fluentd · Promtail · OTel Collector
              parse + enrich + redact + sample + route + buffer

AGGREGATORS
  ELK / OpenSearch   index everything; flexible; $$ + ops heavy
  Loki               labels-only index + S3 bodies; cheap; modest ops
  Splunk             rich, commercial, $$$$
  Datadog / CloudWatch / GCP Logging   hosted, easy, $$

STORAGE TIERS
  hot 7–14d (indexed)   warm 30–90d   cold 1–7y in S3/GCS

COST DRIVERS
  volume bytes/day · cardinality of labels/fields · index multiplier
  cut: drop DEBUG · sample · redact · short hot retention · cold archive

QUERY PATTERN
  filter by labels first · use trace_id · save dashboards
  for high-volume signals: convert log → metric, alert on metric

SECURITY/COMPLIANCE
  TLS + at-rest encryption · access control · query audit logs
  PII redacted at collector · audit logs separate + write-once + long retention

OPERATE the aggregator like a service: capacity, HA, runbooks, alerts.

RULE: ship structured JSON to a centralized store; tune cardinality;
      tier storage; alert on metrics derived from logs, not logs themselves.
```

---

## 14. Resources

### Documentation
- **Elastic / OpenSearch** — <https://www.elastic.co/guide/index.html> / <https://opensearch.org/docs/>
- **Grafana Loki** — <https://grafana.com/docs/loki/>
- **Splunk Docs** — <https://docs.splunk.com/Documentation>
- **Fluent Bit / Fluentd** — <https://docs.fluentbit.io/manual/> / <https://docs.fluentd.org/>
- **Vector** — <https://vector.dev/docs/>
- **OpenTelemetry Logs** — <https://opentelemetry.io/docs/specs/otel/logs/>

### Articles
- "How we cut our log bill 90%" — every team's blog post, sometimes accurate.
- "Loki: Like Prometheus, but for logs" — Grafana Labs.
- "Why we left Splunk" / "Why we left Elastic" — Honeycomb / various engineering blogs.
- "Logging at Stripe scale" — Stripe engineering.
- "Designing log pipelines" — Charity Majors, Cindy Sridharan.

### Books
- *Observability Engineering* — Charity Majors et al.
- *The Logstash Book* — James Turnbull.

### Videos
- "Logs at scale" — Grafana ObservabilityCON talks.
- ByteByteGo — "Building a log pipeline".
- KubeCon — Fluent Bit / OTel talks.

### Tools
- **Collectors:** Fluent Bit, Vector, Fluentd, Promtail, OTel Collector.
- **Aggregators:** OpenSearch, Loki, Splunk, Datadog Logs, Sumo Logic.
- **Query/UI:** Grafana, Kibana, Splunk Web, OpenSearch Dashboards.
- **Pipelines:** Cribl Stream — vendor-neutral log routing.
- **Auditing:** AWS CloudTrail, GCP Audit Logs, write-once S3 buckets.

### Adjacent reading
- [Logging Best Practices →](./logging.md)
- [Metrics & Time-Series →](./metrics.md)
- [Distributed Tracing →](./tracing.md)
- [The Three Pillars of Observability →](./three-pillars.md)
- [Alerting & On-Call →](./alerting.md)
- [Object Storage →](../09-storage/object-storage.md)
- [Search Engines →](../04-databases/search-engines.md)

---

*Previous:* [← Alerting & On-Call](./alerting.md)  |  *Next:* [Health Checks & Heartbeats →](./health-checks.md)
