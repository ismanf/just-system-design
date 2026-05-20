# Metrics & Time-Series (Prometheus, Datadog)

> **TL;DR** — **Metrics** are numerical measurements emitted over time: request rate, error rate, latency percentiles, CPU usage, queue depth. Stored in a **time-series database** (Prometheus, Mimir, VictoriaMetrics, InfluxDB, Datadog) they power dashboards, alerts, capacity planning, and SLOs. Four metric types matter: **counters** (monotonic — total requests), **gauges** (point-in-time — active connections), **histograms** (distributions — latency buckets), **summaries** (similar to histograms but client-side quantiles). The big design trade-off is **cardinality** — every unique label combination is a new time series, and high cardinality is what kills metric systems. Pull-based (Prometheus) vs push-based (StatsD, OTLP) is a stylistic choice; the **RED** (rate, errors, duration) and **USE** (utilization, saturation, errors) methods are the standard ways to think about what to measure.

---

## 1. The Idea

A metric is a single numerical value tagged with labels and a timestamp:

```
http_requests_total{service="api-gateway", method="POST", path="/v1/charges", status="200"} 1247291  @ 2026-05-20T09:14:22Z
```

Every few seconds, the metric is scraped or pushed; over time you get a series of values. With a TSDB, you can ask:

- "What's the request rate for `/v1/charges` over the last 5 minutes?"
- "What's the 99th percentile latency, broken down by region?"
- "Are 5xx errors increasing?"
- "How does today compare to last Tuesday?"

The unit of recall is **aggregations over time** — you don't query individual events. That's logs' job.

---

## 2. Logs vs Metrics, Sharply

| Logs | Metrics |
| --- | --- |
| Discrete events with full context | Numerical samples over time |
| Per-event search ("what happened to request X?") | Aggregate analysis ("what's our error rate?") |
| Storage: bytes-heavy, $$ | Storage: cheap per dimension, scales with cardinality |
| Latency to query: seconds | Latency to query: milliseconds |
| Variable structure | Fixed schema, fixed cost |

You can derive metrics from logs (count of lines matching a pattern), but emitting metrics directly is far cheaper and faster.

---

## 3. The Four Metric Types

Stick to these. Prometheus and OpenMetrics standardize them; most TSDBs use the same model.

### Counter
Monotonically increasing. Never decreases (resets to 0 on process restart).

```
http_requests_total{status="200"} 14729
http_requests_total{status="500"} 12
```

You query the **rate of change**:

```promql
rate(http_requests_total[5m])
```

Use counters for: total requests, total errors, bytes sent, jobs processed.

### Gauge
Point-in-time value. Goes up and down.

```
queue_depth{queue="payments"} 142
memory_bytes{instance="api-1"} 2147483648
goroutines{instance="api-1"} 38
```

Use gauges for: current memory, current connections, queue length, temperature, anything that can decrease.

### Histogram
Buckets of observations. Each bucket counts observations ≤ its upper bound.

```
http_request_duration_seconds_bucket{le="0.005"}  1024
http_request_duration_seconds_bucket{le="0.01"}   1502
http_request_duration_seconds_bucket{le="0.025"}  2014
http_request_duration_seconds_bucket{le="0.1"}    2502
http_request_duration_seconds_bucket{le="0.5"}    2599
http_request_duration_seconds_bucket{le="+Inf"}   2614
http_request_duration_seconds_sum                 187.3
http_request_duration_seconds_count               2614
```

From buckets, you compute percentiles **on the server side**:

```promql
histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
```

Use histograms for: latencies, request sizes, response sizes — anything you want percentiles on. **Almost always the right choice over summaries** because they're aggregatable across instances.

### Summary
Like a histogram, but pre-computes quantiles client-side:

```
http_request_duration_seconds{quantile="0.5"} 0.025
http_request_duration_seconds{quantile="0.99"} 0.190
http_request_duration_seconds_sum 187.3
http_request_duration_seconds_count 2614
```

Cheaper to query but **not aggregatable** across instances (you can't average 99th percentiles). Use only when you need quantiles for a single instance and storage is constrained.

**Default rule:** use histograms.

---

## 4. Labels and Cardinality

Labels (Prometheus) or tags (Datadog) are key-value pairs attached to metrics. Each unique combination is a separate **time series**.

```
http_requests_total{service="api", method="GET", status="200"}   ← one series
http_requests_total{service="api", method="GET", status="500"}   ← another series
http_requests_total{service="api", method="POST", status="200"}  ← another
```

Cardinality = number of unique series. This is the **single most important operational concept** in metrics.

| Label | Cardinality |
| --- | --- |
| `service` | ~50 services |
| `method` | ~6 methods |
| `status` | ~50 codes |
| `path` (full URL) | thousands+ ← danger |
| `user_id` | millions ← never |
| `trace_id` | unbounded ← absolutely never |

Multiply your label cardinalities: `50 × 6 × 50 × 1000 = 15M series` just for HTTP requests. Most TSDBs start choking past a few million active series per instance. Prometheus uses about 3 KB of RAM per active series.

### Cardinality rules of thumb

- Bound label cardinality. Group `path` by route template, not concrete URL: `/users/:id` not `/users/42`.
- Never put high-cardinality fields (user_id, trace_id, request_id) in metric labels. They belong in **logs** or **traces**.
- Drop `status` codes you don't care about (group into `2xx`/`4xx`/`5xx` if needed).
- Audit cardinality regularly (`prometheus_tsdb_head_series`, `count by (__name__) ({__name__=~".+"})`).

Cardinality is what kills monitoring systems. Treat it like memory.

---

## 5. Pull vs Push

| Model | Examples | Pros | Cons |
| --- | --- | --- | --- |
| **Pull** (server scrapes targets) | Prometheus, OpenMetrics | Service-discovery integrated, target health visible | Hard with ephemeral jobs, NATs/firewalls |
| **Push** (clients send to backend) | StatsD, Datadog, OTLP, InfluxDB | Easy for short-lived jobs, mobile, batch | Backend must scale to incoming volume; lost signal if backend down |

Hybrid options:
- **Prometheus Pushgateway** for batch jobs that pull doesn't suit.
- **OpenTelemetry Collector** can scrape Prometheus targets and forward to any TSDB.

Prometheus's pull model has won the cloud-native space because Kubernetes-style service discovery makes target enumeration easy. But the difference matters less than people think — both work; pick what fits.

---

## 6. RED and USE — Standard Frameworks for What to Measure

### RED (per service)
Tom Wilkie's framework. For every service:

- **R**ate — requests per second.
- **E**rrors — failed requests per second (or error %).
- **D**uration — latency distribution.

```promql
# RATE
sum(rate(http_requests_total{service="api"}[5m])) by (service)

# ERRORS
sum(rate(http_requests_total{service="api", status=~"5.."}[5m])) by (service)

# DURATION
histogram_quantile(0.99,
  sum(rate(http_request_duration_seconds_bucket{service="api"}[5m])) by (le))
```

Every external-facing service should publish RED. It's the minimum.

### USE (per resource)
Brendan Gregg's framework. For every resource:

- **U**tilization — % time the resource was busy.
- **S**aturation — extra work waiting (queue depth).
- **E**rrors — error events.

Apply to CPU, memory, disk, network, DB connections, thread pools.

Together: RED tells you **how your service is doing**; USE tells you **why**.

### The Four Golden Signals (Google SRE)
Similar to RED with one addition:
- **Latency**
- **Traffic** (= rate)
- **Errors**
- **Saturation** (≈ USE's saturation)

For most teams, **RED at the service layer + USE at the resource layer** is the right starting point.

---

## 7. The Prometheus Query Language (PromQL)

PromQL is the lingua franca; OpenSearch / Datadog / Grafana have their own dialects but the concepts are the same.

Core operations:

```promql
# instant vector — current value
http_requests_total

# range vector — values over time
http_requests_total[5m]

# rate — per-second average of a counter
rate(http_requests_total[5m])

# aggregation
sum by (status) (rate(http_requests_total[5m]))
avg by (instance) (process_cpu_seconds_total)

# binary ops
rate(http_requests_total{status=~"5.."}[5m])
/
rate(http_requests_total[5m])
# = error rate as a fraction

# histogram percentile
histogram_quantile(0.99,
  sum by (le) (rate(http_request_duration_seconds_bucket[5m])))

# offset for comparisons
rate(http_requests_total[5m]) /
rate(http_requests_total[5m] offset 1d)
```

Performance hint: `rate()` over windows much smaller than the scrape interval is unreliable. Use `rate[5m]` for 15s scrapes; `rate[1m]` is jittery.

---

## 8. Histograms Done Right

Two things to get right:

### Bucket boundaries
Buckets too coarse → low resolution. Too fine → high cardinality. For latency in seconds, a reasonable default:

```
0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10
```

For request sizes in bytes:
```
100, 1k, 10k, 100k, 1M, 10M
```

You can't change bucket boundaries retroactively (the buckets aren't stored elsewhere) — pick well.

### Native histograms / sparse histograms
Prometheus's **native histograms** (2024+) and OpenTelemetry's **exponential histograms** auto-distribute buckets, giving high resolution without manual tuning. The future. Adopt where supported.

---

## 9. Service-Side Instrumentation

Most language ecosystems have idiomatic instrumentation libraries:

| Language | Library |
| --- | --- |
| Go | `prometheus/client_golang`, `otel-go` |
| Python | `prometheus_client`, OpenTelemetry SDK |
| Java | Micrometer, Prometheus Simpleclient, OTel |
| Node | `prom-client`, OpenTelemetry |
| Ruby | `prometheus-client-mmap`, OTel |

A minimal Go example:

```go
var (
    requestsTotal = promauto.NewCounterVec(
        prometheus.CounterOpts{Name: "http_requests_total"},
        []string{"method", "route", "status"},
    )
    requestDuration = promauto.NewHistogramVec(
        prometheus.HistogramOpts{
            Name: "http_request_duration_seconds",
            Buckets: prometheus.DefBuckets,
        },
        []string{"method", "route"},
    )
)

http.HandleFunc("/metrics", promhttp.Handler().ServeHTTP)
```

Instrument **boundaries** by default — HTTP middleware, gRPC interceptors, DB driver wrappers, cache wrappers. Internal function-by-function metrics are usually overkill.

---

## 10. OpenTelemetry — The Convergent Standard

The vendor-neutral standard for metrics (and traces, and logs). One SDK, one wire format (OTLP), any backend.

```
your app  → OTel SDK → OTLP →  OTel Collector  → Prometheus / Datadog / NewRelic / ...
                                ^
                            pull/push/route
```

Why use it:
- **Vendor portability** — switch backends without re-instrumenting.
- **Unified instrumentation** with traces and logs.
- **Standard semantic conventions** (`http.server.duration`, `db.system`, `messaging.system`) so dashboards are reusable.

It's been the right answer since ~2022. New instrumentation should be OTel-first.

---

## 11. Storage and Scaling

| TSDB | Notes |
| --- | --- |
| **Prometheus** | Single-node default; HA via pairs scraping the same targets |
| **Thanos / Cortex / Mimir** | Scale Prometheus horizontally + long-term S3 storage |
| **VictoriaMetrics** | Faster, lower-memory Prometheus-compatible alternative |
| **InfluxDB** | Push-based; popular for IoT |
| **TimescaleDB** | Postgres + time-series extension |
| **Datadog / NewRelic / Dynatrace** | Hosted SaaS |
| **Amazon Managed Service for Prometheus, GCP Managed Prometheus** | Cloud-native managed |

Default storage retention:
- Prometheus on local disk: 15 days.
- Long-term: ship to Thanos/Mimir/Cortex with S3 backend → years.
- SaaS: configurable per plan.

Scaling pattern: **federate** — local Prometheus per cluster scrapes services, remote-writes to a long-term store, dashboards query the long-term store.

---

## 12. Worked Example — Building a Service Dashboard

For an API service, the minimum dashboard:

1. **Request rate** — total, plus broken down by route.
2. **Error rate** — % 5xx; per route.
3. **Latency** — p50, p95, p99 over time; per route.
4. **Saturation** — CPU, memory, DB connections in use vs pool size, queue depth.
5. **Dependencies** — outbound calls' RED for each downstream.
6. **Business KPIs** — orders/min, signups/hour, revenue events (rate-of).

The **Four Golden Signals** dashboard takes about an hour to build for a well-instrumented service. Build it on the first deploy, not after the first incident.

---

## 13. Metrics for SLOs and Error Budgets

**SLO** (Service Level Objective): a target, e.g. "99.9% of requests succeed and complete under 200 ms over a rolling 30 days."

**SLI** (Service Level Indicator): the measurement, derived from metrics:

```promql
sum(rate(http_requests_total{status!~"5..", duration_le="0.2"}[30d]))
/
sum(rate(http_requests_total[30d]))
```

**Error budget**: 1 − SLO = 0.1% of requests can fail in 30 days. Burn through it too fast → freeze risky deploys.

Metrics power this. See [SLA, SLO, SLI, Error Budgets →](../11-reliability/sla-slo-sli.md).

---

## 14. Common Mistakes / Anti-Patterns

- **High-cardinality labels** — `user_id`, `trace_id`, full URL paths. Will kill your TSDB.
- **Using summaries when you need to aggregate across instances** — can't. Use histograms.
- **Free-text metric names** — inconsistent across services. Adopt OpenTelemetry semantic conventions.
- **No `_total` suffix on counters** — convention; helps queries.
- **Storing one metric per status code combination** — explosion. Group when you can.
- **Sampling away rare events at the metric layer** — counters and rates need the full count.
- **Confusing rate() over a counter with the counter itself** — `rate(http_requests_total[5m])` is requests/sec; `http_requests_total` is monotonic count.
- **Histogram buckets misaligned to your latency profile** — p99 sits between buckets, undefined.
- **Forgetting to expose `/metrics` on internal services** — black boxes during incidents.
- **No metrics on outbound calls** — you see your service is slow but not which dependency.
- **Building dashboards before defining what they monitor** — flashy, useless.
- **Alerting on raw metrics without time windows** — single bad scrape pages on-call. Always `rate(...)[5m]` or `avg_over_time(...)`.
- **Self-reported counters that never reset on deploy** — incompatible across processes; only `rate()` is reliable.
- **Per-request push of metrics** — saturates StatsD daemon. Use batched / pre-aggregated emission.
- **No SLO mapping** — metrics that don't tie to user-visible objectives.

---

## 15. Cheat Card

```
METRIC TYPES   counter (monotonic) · gauge (current) · histogram (buckets)
                rule: prefer histograms for distributions

CARDINALITY    each unique label combo = a series.   labels: low-cardinality only.
                NEVER user_id, trace_id, full URL in labels.

FRAMEWORKS     RED   per service: Rate · Errors · Duration
                USE   per resource: Utilization · Saturation · Errors
                Golden Signals (Google): Latency · Traffic · Errors · Saturation

PROMQL         rate(x[5m]) · sum by (label) (...) ·
                histogram_quantile(0.99, sum by (le) (rate(...)))

INSTRUMENT     boundaries first (HTTP/gRPC middleware, DB wrappers)
                OpenTelemetry SDK is the modern choice
                expose /metrics; one /metrics per process

STORAGE        Prometheus (15d) → Thanos/Mimir/Cortex (S3, years)
                or VictoriaMetrics, Datadog, etc.

ALERTS         on SLIs over windows (multi-window, multi-burn-rate)
                NOT on raw single-scrape values

PULL vs PUSH   pull = service discovery friendly · push = ephemeral jobs
                Prometheus default; Pushgateway for batch

RULE: name and label like OpenTelemetry; keep cardinality bounded;
      dashboard the Golden Signals; alert on SLOs.
```

---

## 16. Resources

### Books
- *Site Reliability Engineering* — Google. Chapter 6: Monitoring distributed systems.
- *The SRE Workbook* — Google. Practical SLO/SLI implementation.
- *Observability Engineering* — Charity Majors et al.

### Documentation
- **Prometheus** — <https://prometheus.io/docs/>
- **OpenMetrics spec** — <https://openmetrics.io/>
- **OpenTelemetry Metrics** — <https://opentelemetry.io/docs/specs/otel/metrics/>
- **Grafana docs** — <https://grafana.com/docs/>
- **Datadog metrics** — <https://docs.datadoghq.com/metrics/>
- **PromQL guide** — <https://prometheus.io/docs/prometheus/latest/querying/basics/>

### Articles
- "Monitoring distributed systems" — Google SRE: <https://sre.google/sre-book/monitoring-distributed-systems/>
- "The RED Method" — Tom Wilkie / Grafana Labs.
- "The USE Method" — Brendan Gregg: <https://www.brendangregg.com/usemethod.html>
- "Cardinality is the enemy" — Honeycomb / Robust Perception blog series.
- "Histograms with Prometheus: A tale of woe" — sample data on bucket-tuning.

### Videos
- "Prometheus 101" — CNCF / Brian Brazil.
- ByteByteGo — "Observability metrics in 10 minutes".
- "PromCon" talks — annual Prometheus conference.

### Tools
- **Prometheus, Mimir, Thanos, Cortex, VictoriaMetrics** — open source TSDBs.
- **Grafana, Datadog, NewRelic, Dynatrace, Honeycomb** — visualization / SaaS.
- **OpenTelemetry Collector** — ingest + route.
- **PromLens, Promtools** — query development helpers.

### Adjacent reading
- [Logging Best Practices →](./logging.md)
- [Distributed Tracing →](./tracing.md)
- [The Three Pillars of Observability →](./three-pillars.md)
- [Alerting & On-Call →](./alerting.md)
- [SLA, SLO, SLI, Error Budgets →](../11-reliability/sla-slo-sli.md)
- [Time-Series Databases →](../04-databases/time-series-databases.md)

---

*Previous:* [← Logging Best Practices](./logging.md)  |  *Next:* [Distributed Tracing →](./tracing.md)
