# Design a Metrics & Monitoring System

> **TL;DR** — A monitoring system collects **numeric time-series data** at scale (CPU, memory, request rate, error rate, business metrics) and lets you query, graph, and alert on it. Two collection models: **push** (apps push to ingest endpoint — Datadog, StatsD) or **pull/scrape** (server pulls from `/metrics` endpoints — Prometheus). Storage is a **time-series database** with columnar layout, downsampling, and very high compression. The hard problems: **cardinality** (every tag combination is a separate series — explodes memory), **retention** (raw at 10 s, downsampled at 1 min/1 h/1 d), and **fast aggregation** over billions of points. Prometheus + Grafana is the FOSS reference; Datadog and Honeycomb are the SaaS leaders.

---

## 1. Requirements

### Functional
- Collect metrics: gauges, counters, histograms, summaries.
- Store time-series with labels (tags).
- Query with a flexible language (PromQL, InfluxQL, ...).
- Visualize on dashboards.
- Alert on threshold violations or anomalies.

### Non-Functional
- Write throughput: millions of samples/sec.
- Query latency p99 < 1 sec for typical dashboard queries.
- Retention: ~13 months typical, with downsampling.
- High cardinality handling.

---

## 2. Back-of-the-Envelope

- 10K services × 100 metrics × 10 tags × 1 sample/10 s = ~1 M samples/sec.
- 100 K hosts × 100 metrics each × 1 sample/10 s = ~1 M more.
- Raw: ~10 PB/year if uncompressed. Compressed: ~100×.

---

## 3. High-Level Architecture

```mermaid
flowchart TB
    App --> Exp[/metrics endpoint or push agent]
    Exp --> Scraper[Prometheus / push gateway]
    Scraper --> TSDB[(Time-Series DB)]
    TSDB --> Query[Query Engine]
    Query --> Dash[Grafana / Dashboards]
    Query --> Alert[Alert Manager]
    Alert --> Notify[Pager / Slack / Email]
```

Two-leg flow: collect + store, then query + alert.

---

## 4. Push vs Pull

### 4.1 Push (Datadog, StatsD, InfluxDB classic)
Apps push samples to a central collector.
- Pros: works for ephemeral jobs; no service discovery needed by central.
- Cons: failure isolation is harder (push storm = central goes down).

### 4.2 Pull (Prometheus)
Central scrapes `/metrics` endpoints on each app.
- Pros: central knows targets via service discovery; can detect dead targets (scrape failed).
- Cons: short-lived jobs need a push gateway helper.

Both have valid use cases. Prometheus's pull model became dominant in Kubernetes.

---

## 5. The Time-Series Model

A metric is identified by `(name, tags)`:
```
http_requests_total{method="GET", status="200", svc="checkout"}
http_requests_total{method="POST", status="500", svc="checkout"}
```

Each unique combination is a separate **time series**. A series is `(timestamp, value)` pairs over time.

Types:
- **Counter**: monotonically increasing (request count). Rate computed from differences.
- **Gauge**: arbitrary value (memory in use).
- **Histogram**: bucketed distribution (request latency).
- **Summary**: quantiles computed client-side.

---

## 6. Cardinality — The Pitfall

Each tag value creates a new series. `user_id` as a tag with 1 M users = 1 M series. Memory and storage explode.

Rule: **tags are for low-cardinality dimensions** (host, region, status_code). High-cardinality goes in logs or traces, not metric labels.

This single mistake destroys metric systems regularly.

---

## 7. Storage — Time-Series DB

Built for time-series workloads:
- **Append-only**, write-heavy.
- **Columnar** storage (each metric's values stored contiguously).
- **High compression** — delta-of-delta encoding for timestamps; XOR encoding (Gorilla, Facebook's paper) for values. 1.4 bytes per sample typical.
- **Per-series chunks** of recent data in memory; older flushed to disk.

Reference implementations: Prometheus TSDB, InfluxDB, M3DB (Uber), VictoriaMetrics, Thanos, Cortex/Mimir.

---

## 8. Downsampling

Recent data at full resolution; older data downsampled.

```
0–24 h:    10 s resolution
24 h–7 d:  1 min resolution
7 d–30 d:  5 min
30 d+:     1 h
```

Storage cost drops sharply per tier. Query latency improves.

Implemented via continuous rollups (e.g., Thanos compactor, Prometheus recording rules).

---

## 9. Query Language

PromQL is the standard model:
```
rate(http_requests_total{svc="checkout", status=~"5.."}[5m])
```
- `rate(...)` computes per-second rate over 5-minute windows.
- Labels filtered with matchers.
- Aggregations: sum, avg, max by labels.

Most systems converge on a similar model: select, filter, aggregate, window.

---

## 10. Alerting

Alerts are queries that produce results when an alert condition is met:
```
error_rate > 0.05 for 5 minutes
```

Alert manager:
- Deduplicates alerts (multiple Prometheus instances raising the same alert).
- Groups related alerts.
- Routes by service / severity to notification channels.
- Silences for maintenance.

Common pitfall: alert fatigue. Tune thresholds; alert on symptoms, not causes.

See [Alerting & On-Call →](../13-observability/alerting.md).

---

## 11. Service Discovery

Pull-based systems need to know what to scrape. Integrated with:
- Kubernetes API (pods + endpoints).
- Consul, etcd.
- AWS EC2 / ECS / EKS.

Targets discovered dynamically; configuration is mostly hands-off.

---

## 12. High Availability

Prometheus pairs run in parallel, both scraping the same targets. Alertmanager dedupes.

For very long retention or global query, use Thanos / Cortex / Mimir:
- Prometheus stores recent locally.
- Long-term data written to object storage (S3).
- Query gateway federates across instances.

---

## 13. Common Mistakes

- **High-cardinality labels** (`user_id`, `request_id`) — series count explodes.
- **No downsampling** — storage cost grows linearly without bound.
- **Alerts on every metric** — fatigue. Alert on user impact.
- **Polling intervals too short** — 1-second scraping is rarely worth it.
- **Single Prometheus instance** — no HA; rotating restarts kill alerting.
- **Treating logs and metrics interchangeably** — different shapes, different costs.

---

## 14. Cheat Card

```
PURPOSE    Numeric time-series collection, storage, query, alerting.

CORE       Pull or push collection; service discovery for pull
           Time-series DB: columnar + Gorilla compression
           PromQL-style query language
           Downsampling tiered by age
           Alertmanager for dedup, grouping, routing

CARDINALITY  Low-cardinality labels only (status, svc, region).
             High-cardinality dimensions belong in logs/traces.

PITFALLS   user_id as a label, no downsampling,
           alert on every metric, single-instance Prometheus.

RULE       Metrics are about aggregate behavior.
           Per-request details belong in traces.
```

---

## Resources

### Articles
- "Gorilla: A Fast, Scalable, In-Memory Time Series Database" — Facebook 2015
- "Prometheus: Up & Running" — Brian Brazil's blog
- "How we built Cortex" — Weaveworks
- "Honeycomb's storage engine" — Honeycomb blog

### Documentation
- **Prometheus** — <https://prometheus.io/docs/>
- **Thanos** — <https://thanos.io>
- **OpenMetrics** standard — <https://openmetrics.io>

### Books
- *Prometheus: Up & Running* — Brian Brazil
- *Observability Engineering* — Charity Majors et al.

### Videos
- ByteByteGo: "Design a Monitoring System"
- "How to Monitor the SRE Golden Signals" — talks

### Adjacent reading
- [Metrics & Time-Series →](../13-observability/metrics.md)
- [Three Pillars of Observability →](../13-observability/three-pillars.md)
- [Time-Series Databases →](../04-databases/time-series-databases.md)
- [Logging System →](./logging-system.md)
- [Alerting →](../13-observability/alerting.md)

---

*Previous:* [← Logging System](./logging-system.md)  |  *Next:* [Ad Click Aggregator →](./ad-click-aggregator.md)
