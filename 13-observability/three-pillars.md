# The Three Pillars of Observability

> **TL;DR** — The "three pillars" model says observability rests on **logs, metrics, and traces** — three complementary signals that together answer the questions production raises. Logs are discrete events with full context; metrics are aggregate numerical samples over time; traces are the path of a single request across services. They overlap but don't substitute for one another. Modern thinking adds **profiles** as a fourth pillar (continuous CPU/memory profiling) and reframes the three as *outputs* of a single underlying observation: **rich, wide events**. The practical takeaway: don't pick one. Instrument all three at the boundaries of every service, **correlate them via trace IDs**, and design dashboards and alerts that walk from one to the other in seconds.

---

## 1. The Three Pillars

```
┌──────────────────────────────────────────────────────────────────┐
│                                                                  │
│   LOGS                METRICS                TRACES              │
│   ─────                ───────                ──────              │
│   events               numbers                paths               │
│   per record           aggregate over time    per request         │
│                                                                  │
│   What happened?       How is it doing?       Where did time go? │
│                                                                  │
│   Storage: $$          Storage: $              Storage: $$        │
│   Query: slow          Query: fast             Query: medium      │
│   Cardinality: high    Cardinality: bounded    Cardinality: medium│
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

The model was popularized by Peter Bourgon (2017) and adopted everywhere. It is a useful map even if (as we'll see) modern systems blur the lines.

---

## 2. What Each Pillar Answers

### Logs
*"What happened to this particular request? What was the error message? What did the system see at this moment?"*

Logs are discrete events with rich context. They're the only signal that captures the **narrative**. They're irreplaceable for incident forensics, audit, and security.

See [Logging Best Practices →](./logging.md).

### Metrics
*"How is the system doing? Is the error rate up? Is p99 latency over budget? How many requests are we doing per second?"*

Metrics are pre-aggregated numerical samples. They're what dashboards and alerts run on. Cheap to store at scale (because cardinality is bounded), fast to query.

See [Metrics & Time-Series →](./metrics.md).

### Traces
*"This request was slow — which hop ate the time? Which dependency failed? What's the dependency graph between services?"*

Traces follow a single request across service boundaries. They're indispensable in microservice systems where a single user action touches a dozen services.

See [Distributed Tracing →](./tracing.md).

---

## 3. The Overlap

Logs, metrics, and traces overlap significantly:

```
            ┌─────────┐
            │  LOGS   │
            │         │
       ┌────┤  events ├────┐
       │    │         │    │
       │    └────┬────┘    │
       │         │         │
  ┌────┴────┐    │   ┌─────┴────┐
  │ METRICS │────┼───│  TRACES  │
  │ counts/ │    │   │ per-req  │
  │ rates   │    │   │ spans    │
  └─────────┘    │   └──────────┘
```

- A **log line** can be derived from a span event.
- A **metric** can be computed from a stream of log events.
- A **trace span** has attributes that overlap with what a log record carries.

Modern thinking (Charity Majors and others): **all three are projections of the same underlying "wide events."** A rich event carrying 50 attributes can be summarized into a metric (count over time), inspected per-record (log), or chained with related events (trace).

The point isn't which model is correct — it's that artificial walls between them defeat the value. Trace IDs in logs. Exemplars in metrics that link to traces. One observability surface.

---

## 4. Strengths and Limits

| | Logs | Metrics | Traces |
| --- | --- | --- | --- |
| **Best at** | Forensics, narrative, audit | Trends, alerts, dashboards | Per-request paths, latency attribution |
| **Worst at** | Aggregates, trends | Per-event details | High volume without sampling |
| **Cardinality** | Unbounded | Strictly bounded | Bounded with sampling |
| **Cost model** | Bytes ingested | Active series | Events × sampling rate |
| **Query speed** | Seconds to minutes | Milliseconds | Seconds |
| **Retention** | Days–months ($$) | Months–years (cheap) | Days–weeks |
| **Used by** | Engineers, SOC, auditors | SREs, on-call, leadership | Engineers debugging perf |

The wrong tool for a question is painful:
- "Why is p99 slow?" → don't grep logs; query traces.
- "How many 5xx?" → don't sample traces; count metrics.
- "Why did **this** user fail?" → don't query metrics; find the log line.

Each pillar saves the others from drowning.

---

## 5. The Fourth Pillar: Profiles

**Continuous profiling** has emerged as the fourth widely-used signal:

- **What it is:** sampled CPU, memory, lock, allocation profiles emitted continuously from production processes (always-on flame graphs).
- **What it answers:** *"Why is this service burning CPU? Where are the goroutines stuck? Why is memory growing?"*
- **Tools:** Pyroscope, Parca, Polar Signals, Grafana Cloud Profiles, Datadog Continuous Profiler, New Relic CodeStream.
- **Cost:** low, ~1–5% CPU overhead.

Profiling answers questions metrics+logs+traces can't. "p99 latency is high, traces show the time is in `compute_pricing`" — profiling shows you the exact function in the binary burning CPU.

For 2026 systems, plan for four signals, not three.

---

## 6. Other Adjacent Signals

The pillar model is convenient but incomplete. Production observability also includes:

| Signal | Purpose |
| --- | --- |
| **Events** | Discrete deploys, scale events, config reloads — annotated on dashboards |
| **Audit logs** | Tamper-evident records of sensitive actions |
| **Real User Monitoring (RUM)** | Browser/mobile-side performance |
| **Synthetic monitoring** | Probes simulating user journeys |
| **Network flows / packet captures** | Underlay-level diagnostics |
| **eBPF traces** | Kernel-level visibility (network, syscalls) |
| **Runtime stats** | GC pauses, thread states, FD counts |

These supplement the three pillars; they don't replace them.

---

## 7. The Modern Stack

A canonical observability stack in 2026:

```
your services
   │
   │ OpenTelemetry SDK
   │   (logs + metrics + traces, OTLP wire format)
   ▼
OpenTelemetry Collector  (sidecar or fleet)
   │
   ├──► logs ──────► Loki / Elasticsearch / Splunk
   ├──► metrics ───► Prometheus / Mimir / VictoriaMetrics / Datadog
   ├──► traces ────► Tempo / Jaeger / Honeycomb / Datadog APM
   └──► profiles ──► Pyroscope / Grafana Cloud Profiles
                         │
                         ▼
                  Grafana / Datadog / Honeycomb
                  (unified UI, click between signals)
```

The key idea: **one SDK, one wire format, multiple backends**. You can swap backends without re-instrumenting.

For SaaS choice: Datadog, Honeycomb, New Relic, Dynatrace, Grafana Cloud, Lightstep, SigNoz, Splunk Observability are all major options. They differ on price, query language, and which pillar they're strongest at — but all support OTLP ingest now.

---

## 8. Correlation Is the Whole Game

A trace ID in every log line and every metric exemplar is what makes the system feel cohesive:

```
alert fires
  → "p99 latency > 800ms for 5m"
  → click → metric chart with exemplars (links to slow traces)
  → click an exemplar → trace waterfall
  → identify the slow span (say, db query)
  → click span → log lines emitted within that span (with same trace_id)
  → see exact SQL, parameters, error
```

This walk takes **seconds** with proper correlation. Without it (separate dashboards, no shared IDs), it's the worst kind of incident: time-pressured archaeology.

Implementations:
- **OpenTelemetry** auto-injects `trace_id` and `span_id` into log records and metric exemplars.
- **Grafana** unifies Loki + Mimir + Tempo with click-through navigation.
- **Datadog APM ↔ Logs ↔ Metrics** integrated natively.
- **Honeycomb's wide events** treat all signals as one queryable layer.

If your observability stack doesn't support this walk, fix it before fixing anything else.

---

## 9. Pillars Map to Questions

Use this when designing instrumentation for a new service. For each pillar, ask:

```
LOGS
  - What request-boundary events do I want to capture?
  - What business events are worth logging? (signups, orders, payments)
  - What sensitive data must be redacted?
  - What's the retention need? (debugging 30 d? audit 1+ year?)

METRICS
  - What's the RED (rate, errors, duration) per route?
  - What's the USE (utilization, saturation, errors) per resource?
  - Which metrics tie to SLOs?
  - What's the label cardinality budget?

TRACES
  - What are the boundaries to instrument? (HTTP, gRPC, DB, queue, cache)
  - Which custom business spans add value?
  - What attributes (user.id, org.id, etc.) make traces searchable?
  - What's the sampling rate plan?

PROFILES
  - Is the language supported by my profiler?
  - What flame-graph granularity will I keep?
```

---

## 10. SLOs Tie It All Together

SLOs (Service Level Objectives) sit on top of metrics but pull from all signals:

```
SLO        "99.9% of /checkout completes in <500ms over 30d"
         │
         ├─► SLI from metrics: success rate, latency histogram
         ├─► Error budget from same → alerting on burn rate
         ├─► When budget burns: traces show why → fix
         └─► Logs preserve forensic trail
```

The three pillars aren't ends in themselves — they exist to answer questions about user-visible behavior. SLOs define the questions; the pillars provide the answers. See [SLA, SLO, SLI, Error Budgets →](../11-reliability/sla-slo-sli.md).

---

## 11. Failure Modes Per Pillar

Each pillar has its own failure pattern. Watch for:

**Logs:**
- Log volume explodes during incidents; aggregator chokes.
- Sensitive data sneaks in; compliance event.
- Local log files fill disk; pods die.
- Free-text logs unparseable; nobody can search.

**Metrics:**
- Label cardinality grows unchecked → TSDB OOM.
- Histograms with wrong buckets; quantiles meaningless.
- Counters reset on deploy and look like crashes.
- No alerts on SLI burn rate; outages happen unnoticed.

**Traces:**
- Propagation breaks at one service; downstream looks orphaned.
- Sampling rate too low; nothing to look at when needed.
- Attributes too noisy or too sparse to search.
- Time skew between hosts → spans look out of order.

Treat the observability stack as a production system itself — it needs SLOs, capacity planning, and runbooks.

---

## 12. Cost and Pragmatism

Observability is expensive. Common cost realities:

- A logging bill 10× the production cluster's compute bill is possible.
- A metrics system with high cardinality can demand 100s of GB RAM.
- Tracing at 100% costs more than the underlying services.

Cost controls:

- **Logs:** drop verbose categories; sample high-volume; redact and shorten.
- **Metrics:** prune labels; group rare buckets; lower retention on internal metrics.
- **Traces:** head-based sampling 1–5%, tail-based for errors/slow.
- **Profiles:** sample lower-frequency in dev.
- **Tiered storage:** hot store recent (Elasticsearch/Loki), cold in S3 (long-term).

The trap: cost-controlling so hard that observability doesn't fire when you need it. Have an explicit budget; instrument deliberately.

---

## 13. Common Mistakes / Anti-Patterns

- **One pillar only.** "We have great logs" — and nothing else. Each pillar answers different questions.
- **No correlation IDs.** Tracing data exists but logs don't carry trace_id; can't pivot.
- **Pillars siloed in different vendors with no integration.** Logs in Splunk, metrics in Prometheus, traces in Jaeger, no glue → context-switch hell.
- **Treating logs as metrics.** "Counting log lines" is expensive and lossy. Emit metrics directly.
- **Treating metrics as logs.** "I'll just put `user_id` in a tag" → TSDB explosion.
- **Treating traces as logs.** Logging every variable into span attributes → unsearchable, costly.
- **Building dashboards before understanding the system.** Pretty charts of meaningless numbers.
- **Alerting on every metric.** Alert fatigue; real incidents missed.
- **No SLOs; alerting on raw metrics.** Pages fire on momentary blips; nobody trusts the alerts.
- **Skipping profiling because it's "not one of the three pillars".** It absolutely is now.
- **Not budgeting for observability cost or scale.** Goes wrong at the worst time.
- **Observability stack with no SLO of its own.** When it's down, you're flying blind.

---

## 14. Cheat Card

```
THREE PILLARS
  LOGS      events with context        "what happened?"
  METRICS   aggregate numbers         "how is it doing?"
  TRACES    request paths             "where did time go?"

+ PROFILES  continuous flame graphs   "why this CPU/memory?"   (modern 4th pillar)

EACH IS A PROJECTION of the underlying rich event.
   Real value = correlation between them via trace_id.

PER PILLAR
  Logs       structured JSON, trace_id, sampled high-volume, redacted
  Metrics    RED/USE/Golden Signals, bounded cardinality, OTel semconv
  Traces    W3C propagation, head/tail sampling, attrs for search
  Profiles   continuous, low overhead, language-supported

STACK (OTEL-NATIVE)
  app → OpenTelemetry SDK → OTLP → OTel Collector → Loki/Mimir/Tempo/Pyroscope
                                                  ↓
                                                Grafana / Honeycomb / Datadog

CORRELATION   trace_id in every log, exemplars on metrics,
              span links to logs, profiles tagged with trace
              Goal: click from alert → metric → trace → log in seconds

DON'T   silo pillars in unconnected vendors · count log lines for metrics ·
        put user_id in metric labels · alert on raw numbers ·
        skip SLOs · skip profiling

RULE: instrument boundaries; correlate by trace_id; alert on SLOs.
```

---

## 15. Resources

### Books
- *Observability Engineering* — Charity Majors, Liz Fong-Jones, George Miranda. Best modern treatment.
- *The SRE Workbook* — Google. SLI/SLO derivation from signals.
- *Distributed Tracing in Practice* — Austin Parker et al.
- *Site Reliability Engineering* — Google.

### Documentation
- **OpenTelemetry** — <https://opentelemetry.io/docs/>
- **OpenTelemetry semantic conventions** — <https://opentelemetry.io/docs/specs/semconv/>
- **Google SRE Book — Monitoring** — <https://sre.google/sre-book/monitoring-distributed-systems/>
- **CNCF Observability Whitepaper** — <https://github.com/cncf/tag-observability>

### Articles
- "Logs and Metrics" — Cindy Sridharan (the genre-defining post).
- "Three Pillars with Zero Answers" — Charity Majors / Honeycomb.
- "Observability ≠ Three Pillars" — Charity Majors.
- "Continuous profiling: the new pillar" — Grafana Labs / Polar Signals blogs.

### Videos
- "Observability for Developers" — Charity Majors.
- "What is Observability?" — Liz Fong-Jones, SREcon talks.
- ByteByteGo — "Logs, Metrics, Traces".

### Tools
- **OpenTelemetry SDKs + Collector** — instrumentation + transport.
- **Grafana stack** — Loki, Mimir, Tempo, Pyroscope, Grafana.
- **Honeycomb, Datadog, New Relic, Lightstep, Splunk Observability, SigNoz** — SaaS / OSS.
- **Pyroscope, Parca, Polar Signals** — continuous profiling.

### Adjacent reading
- [Logging Best Practices →](./logging.md)
- [Metrics & Time-Series →](./metrics.md)
- [Distributed Tracing →](./tracing.md)
- [Alerting & On-Call →](./alerting.md)
- [Centralized Log Aggregation →](./log-aggregation.md)
- [Health Checks & Heartbeats →](./health-checks.md)
- [SLA, SLO, SLI, Error Budgets →](../11-reliability/sla-slo-sli.md)

---

*Previous:* [← Distributed Tracing](./tracing.md)  |  *Next:* [Alerting & On-Call →](./alerting.md)
