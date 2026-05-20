# Logging Best Practices (Structured Logs)

> **TL;DR** — Logs are the time-ordered record of *what happened*. Done well, they're how you understand production at 3 a.m. Done badly, they're a wall of free-text noise nobody can search. The single most important shift in modern logging is **structured logging** — every log line is a JSON (or similar) document with consistent fields (`level`, `time`, `service`, `trace_id`, `user_id`, `event`, ...) instead of an unparseable `printf` string. Add **request-scoped context** (trace IDs, request IDs) propagated everywhere, **levels** that have rules, **sampling** for high-volume events, and **strict redaction** of secrets and PII. Ship to a centralized aggregator (Loki, ELK, Splunk, Datadog) — local logs that die with the pod are no logs at all.

---

## 1. The Idea

A log line is a fact, timestamped, about something that happened. You want:

```
2026-05-20T09:14:22.413Z  api-gateway  level=info
  trace_id=4f2c8a1b9d3e6f0a
  request_id=req_19hxc3
  user_id=user_42
  org_id=org_7
  method=POST path=/v1/charges status=201 duration_ms=87
  event=request_completed
```

Not this:

```
[Wed May 20 09:14:22 2026] INFO: Got a request from user 42 (POST /v1/charges)
that finished with status 201 in 87ms (request_id=req_19hxc3)
```

The first is parseable, indexable, searchable, and groupable in any modern log aggregator. The second is a tax on every engineer reading it. **Structured logging is the foundation — everything else builds on it.**

---

## 2. The Three Audiences for Logs

Logs serve three different uses, and ignoring one of them produces dysfunction:

| Audience | What they want |
| --- | --- |
| **Humans debugging incidents** | Coherent narrative, enough context per line, low noise, fast search |
| **Aggregation / SIEM** | Consistent schema, queryable fields, security events, audit trails |
| **Future-you** | Enough to reconstruct what happened weeks later when the bug repeats |

Design choices flow from "what's the audience for this log line?" An `INFO` line about every cron tick serves nobody. A `WARN` with no context serves nobody. Every log line should justify its existence.

---

## 3. Structured Logging — The Format

Pick **one** structured format and stick with it across services:

| Format | Notes |
| --- | --- |
| **JSON Lines (`.jsonl`)** | The dominant choice. Parseable, ubiquitous, OK on disk. |
| **logfmt** | `key=value key=value` — Heroku origin, very compact, easy to read raw |
| **OpenTelemetry log records** | Standardized, vendor-neutral; rising rapidly |
| **Protobuf / binary** | Internal high-volume systems only |

The format almost matters less than **consistency** — same field names, same shape, every service.

### A minimum schema

```json
{
  "ts": "2026-05-20T09:14:22.413Z",
  "level": "info",
  "service": "api-gateway",
  "version": "2026.05.18-3a9c1",
  "trace_id": "4f2c8a1b9d3e6f0a8c5b2d7e9f1a3c6e",
  "span_id": "9d3e6f0a8c5b2d7e",
  "request_id": "req_19hxc3",
  "user_id": "user_42",
  "org_id": "org_7",
  "event": "request_completed",
  "msg": "POST /v1/charges completed",
  "http.method": "POST",
  "http.path": "/v1/charges",
  "http.status": 201,
  "duration_ms": 87
}
```

Reserved fields (`ts`, `level`, `service`, `trace_id`) are mandatory; arbitrary fields are allowed as long as keys are consistent. Use **dotted names** (`http.path`, `db.statement`) to namespace.

### One message text per event, then structured details

```js
log.info({event: "user.signup", user_id, plan: "pro"}, "user signed up");
```

The free-text `msg` is for humans skimming. The structured fields are for grep/query.

---

## 4. Levels — Use Them With Rules

Six levels are typical; in practice you need three or four:

| Level | Meaning | Volume | Pages someone? |
| --- | --- | --- | --- |
| **FATAL** | The process is about to die | Tiny | Yes |
| **ERROR** | A request failed in an unexpected way; investigate | Low | Sometimes |
| **WARN** | Something off but request succeeded; recoverable | Low–med | No |
| **INFO** | Normal operations (request done, job ran) | Medium | No |
| **DEBUG** | Per-step internals; off in production by default | High | No |
| **TRACE** | Method entry/exit, finest-grained | Very high | No |

Have rules:

- Every **ERROR** should mean "a human should eventually look at this." If 95% are benign, you have an INFO masquerading as ERROR.
- **WARN** should be rare. If it's per-request, demote it to INFO.
- **DEBUG/TRACE** are not enabled in prod by default. Toggle on for a deploy or a user-specific debug flag.
- Set production default to **INFO**. Going higher (WARN) hides too much; going lower (DEBUG) costs too much.

The most common dysfunction: **everything logged at INFO** with no field structure. The signal-to-noise ratio collapses.

---

## 5. Request-Scoped Context

The single most useful pattern: a **correlation ID** propagated through every log line for a request.

```
incoming request
  → middleware mints trace_id = "4f2c8a..." or extracts from W3C traceparent header
  → all logs in this request include trace_id
  → outgoing HTTP calls add `traceparent` header → next service uses same trace_id
  → cross-service log search by trace_id reconstructs the full path
```

Tools:
- **OpenTelemetry** — the standard. Auto-injects `trace_id` and `span_id` into logs.
- **Go**: `context.Context` carrying the trace.
- **Node**: `AsyncLocalStorage`.
- **Java**: MDC (Mapped Diagnostic Context).
- **Python**: `contextvars`.

Other context worth attaching for every request: `user_id`, `org_id`, `route`, `method`, `tenant`, `region`, `pod`. These should be set once at the boundary and inherited by every line until the request ends.

### W3C Trace Context

The standard header carrying trace IDs cross-service:

```
traceparent: 00-4f2c8a1b9d3e6f0a8c5b2d7e9f1a3c6e-9d3e6f0a8c5b2d7e-01
```

Every modern HTTP library / framework knows this. See [Distributed Tracing →](./tracing.md).

---

## 6. Logging in the Lifecycle of a Request

A typical request produces ~3–10 log lines. Roughly:

1. **request_received** — method, path, key headers, source IP.
2. (optional) **auth_succeeded / auth_failed** — who authenticated.
3. **business_event** — what was done (`order.placed`, `subscription.upgraded`).
4. (only on failure) **error** — exception with stack trace, error class, fields.
5. **request_completed** — status, duration, bytes_out.

Avoid logging **inside** tight inner loops (per-row, per-iteration). Don't log "Entering function X" / "Leaving function X" — that's what tracing is for.

---

## 7. What to Log — and What Not To

Log:
- Request boundaries (in / out).
- Business events at decision points.
- Errors and unexpected paths with full context.
- State changes (deploys, config reloads, scale events).
- Security-relevant events (login success/failure, MFA, permission denials, role changes, data exports).
- Background job lifecycle (started, succeeded, failed, retried).
- External service calls — endpoint, status, duration.

Don't log:
- **Passwords, tokens, secrets, API keys.** Ever. Redact before logging.
- **PII you don't need.** Email, name, address may be okay if consented; SSN, credit card numbers, health data — never.
- **Large payloads.** Truncate to first/last few KB; full body goes to a separate diagnostic store if needed.
- **Per-iteration noise.** Use sampling or aggregation.
- **Inside hot loops or in render paths.** Logging is expensive enough to matter at high rates.
- **Multiple times per failed request.** One error with context beats five.

### Redaction patterns

```python
# DON'T
log.info({event: "login_attempt", email: user.email, password: req.password})

# DO
log.info({event: "login_attempt", email: redact_email(user.email)})

# Even safer — log a stable hash you can group on without recovering the value
log.info({event: "login_attempt", email_hash: sha256(email)[:16]})
```

Build a **redaction layer** at the logging boundary: known sensitive fields are stripped or hashed regardless of who tries to log them. Belt-and-braces.

---

## 8. Sampling — Logging Doesn't Have to Be 100%

Two cases call for sampling:

### High-volume, low-individual-value events
Logging every Redis GET is useless and expensive. Sample 1%:

```python
if random.random() < 0.01:
    log.info({event: "redis.get", key: ...})
```

Or log aggregates (P50/P99 latency, count) instead of each event.

### Errors that come in bursts
A misbehaving client sending 100k bad requests/sec doesn't need 100k ERROR lines. Use **rate limiting** (Slack's `glog` patterns) or **head-of-bucket sampling** — keep the first N per minute, drop the rest, but count them so you know N more occurred.

OpenTelemetry's log SDK supports **tail-based sampling** — buffer logs for a request, decide to keep/drop after seeing the outcome (always keep errors; sample successes).

---

## 9. Performance Considerations

Logging looks free. It isn't.

- **Synchronous logging** to stdout/file blocks the request thread. Heavy log volumes destroy throughput.
- **Async / buffered loggers** (Zap, Zerolog, Logback async, Python `QueueHandler`) push work off the request path.
- **Allocation-free libraries** matter at scale (Zap, Zerolog — Go's logging speed champions).
- **JSON marshaling cost** is real. Pre-allocate, reuse buffers.
- **Log level checks should be cheap.** `if (debugEnabled) log(...)` to avoid building debug strings that are dropped.

At Stripe / Cloudflare / Netflix scale, logging is a multi-percent of CPU. Profile.

---

## 10. Output Strategy

Modern services log to **stdout/stderr**. A collector ships logs centrally:

```
┌────────────┐
│   app pod  │  stdout (JSON lines)
└─────┬──────┘
      │ container runtime captures
      ▼
┌────────────┐
│  log agent │  Vector / Fluent Bit / Fluentd / Promtail / DD agent
└─────┬──────┘
      │ batch + ship
      ▼
┌────────────┐
│ Aggregator │  Loki / Elasticsearch / Splunk / Datadog Logs
└────────────┘
```

This is the **12-factor** pattern (treat logs as event streams). Benefits:
- Apps stay simple — write to stdout, done.
- The collector handles retries, batching, multi-destination routing.
- Easy to swap aggregators without app changes.

Anti-pattern: app writes to files on local disk. Files fill, pods restart, logs vanish. Don't.

See [Centralized Log Aggregation →](./log-aggregation.md) for the receiving side.

---

## 11. Indexing, Cardinality, and Cost

Aggregators charge by ingest volume or indexed cardinality. Two pitfalls:

- **High cardinality in indexed fields.** Putting `user_id` (millions of distinct values) in an indexed field can blow up Elasticsearch cluster memory. Loki and Datadog handle this differently — know your tool.
- **JSON field explosion.** Logging dynamic objects creates hundreds of distinct fields. Many aggregators map these to indexed schemas with limits.

Mitigations:
- Pin **labels** (low-cardinality, indexed: service, env, region) vs **fields** (high-cardinality, in body: user_id, request_id).
- Drop or sample heavy fields before shipping.
- Keep raw payloads in cheap object storage (S3) if needed; ship only the structured summary to the aggregator.

---

## 12. Worked Example — A Solid Logger Setup

Go with Zap (JSON):
```go
logger, _ := zap.NewProductionConfig().Build()
logger = logger.With(zap.String("service", "api-gateway"), zap.String("version", buildVersion))

// per-request
reqLogger := logger.With(
    zap.String("trace_id", traceID),
    zap.String("request_id", reqID),
    zap.String("user_id", userID),
)

reqLogger.Info("request received",
    zap.String("method", r.Method),
    zap.String("path", r.URL.Path),
)
```

Node with pino:
```js
const logger = pino({
  base: { service: 'api-gateway', version: BUILD_VERSION },
  redact: ['req.headers.authorization', '*.password', '*.token'],
});
// per-request child logger
const reqLog = logger.child({ trace_id, request_id, user_id });
reqLog.info({ method, path }, 'request received');
```

Python with structlog:
```python
import structlog
log = structlog.get_logger().bind(service="api-gateway", version=BUILD)

req_log = log.bind(trace_id=trace_id, request_id=req_id, user_id=user_id)
req_log.info("request_received", method=req.method, path=req.path)
```

Three different languages, same shape. That's the goal.

---

## 13. Logs vs Metrics vs Traces

Logs are the **discrete event** signal. They overlap with metrics and traces but aren't substitutes:

| Use logs for | Use metrics for | Use traces for |
| --- | --- | --- |
| Per-event narrative | Aggregate counts, rates, latencies | Path of a request across services |
| Debugging "what happened to **this** request" | Dashboards, alerts | "Where did the time go?" |
| Security/audit trail | Resource utilization | Service dependency graphs |

You need all three. See [The Three Pillars of Observability →](./three-pillars.md).

---

## 14. Audit Logging

Some logs are **legally required** to exist and be tamper-evident:

- Login success/failure.
- Permission/role changes.
- Data exports.
- Admin actions.
- Sensitive data access.

Audit logs should be:
- **Separated** from operational logs (different stream, different retention).
- **Append-only** (or tamper-evident — signed, write-once storage).
- **Long-retained** (1+ year typical for SOC 2; 7+ for some financial).
- **Reviewed periodically.**

AWS CloudTrail, GCP Cloud Audit Logs are the cloud-side examples; application audit logs need their own pipeline.

---

## 15. Common Mistakes / Anti-Patterns

- **Free-text logs** — unparseable; everyone writes their own grep. Use JSON.
- **`printf("entering func X")`** — that's tracing's job.
- **No request ID / trace ID** — can't correlate events for one request.
- **Logging secrets** — passwords, tokens, API keys, JWTs. Even hashes if the input is low-entropy.
- **Logging PII at INFO** — GDPR / CCPA exposure.
- **Logging full request/response bodies** — leaks data, costs $$.
- **One log per loop iteration** — collapses signal.
- **Everything at INFO** — no way to tune verbosity.
- **No timestamps with timezone** — RFC 3339 with UTC ("Z"), always.
- **String-formatted timestamps inconsistently** — different services, different formats.
- **Local-only logs in containers** — die with the pod.
- **Logging in hot paths synchronously** — kills throughput.
- **No structured fields for trace_id / user_id** — grep becomes the only tool.
- **No retention policy** — costs balloon and old logs nobody needs sit forever.
- **No central aggregation** — every incident becomes a tour of ten dashboards.
- **Treating logs as the only observability** — metrics and traces would have answered this question faster.

---

## 16. Cheat Card

```
LOGS = time-ordered events.   Use STRUCTURED (JSON / logfmt / OTel).

REQUIRED FIELDS    ts (RFC 3339 UTC) · level · service · version
                   trace_id · request_id · event · msg

CONTEXT TO PROPAGATE   trace_id (W3C traceparent) · user_id · org_id
                       set once at the boundary, inherited everywhere

LEVELS    INFO default in prod.   ERROR = a human looks.   DEBUG/TRACE off.

LOG       request in/out · business events · errors · state changes ·
          security events · job lifecycle · external calls

DON'T LOG passwords · tokens · API keys · sensitive PII · full bodies ·
          per-iteration noise · "entering function X"

OUTPUT    stdout (12-factor) → log agent → central aggregator
          Loki / ELK / Splunk / Datadog / OpenSearch

PERF      async + buffered + allocation-free libs (Zap, Zerolog, pino)
          sample high-volume events; rate-limit error bursts

SECURITY  redaction layer at logging boundary
          audit logs separate, append-only, long-retained

RULE: every line should answer "who, when, what, in which request, where to look next."
```

---

## 17. Resources

### Books
- *The Art of Monitoring* — James Turnbull.
- *Observability Engineering* — Charity Majors, Liz Fong-Jones, George Miranda. Modern bible.
- *Site Reliability Engineering* — Google. Chapters on logs and post-mortems.

### Documentation
- **OpenTelemetry Logs** — <https://opentelemetry.io/docs/specs/otel/logs/>
- **12-Factor App — Logs** — <https://12factor.net/logs>
- **W3C Trace Context** — <https://www.w3.org/TR/trace-context/>
- **Google SRE Book — Monitoring** — <https://sre.google/sre-book/monitoring-distributed-systems/>

### Articles
- "Logging at scale" — Charity Majors / Honeycomb blog.
- "Why we love structured logging" — Stripe engineering.
- "Anatomy of a Good Production Log" — Brandur Leach: <https://brandur.org/logfmt>
- "The death of the printf log" — various engineering blogs.

### Videos
- "Observability for engineers" — Charity Majors, KubeCon talks.
- ByteByteGo — "Logs vs Metrics vs Traces".

### Tools
- **Loggers:** Zap, Zerolog (Go); pino (Node); structlog (Python); Logback / Log4j 2 (Java); slog (Go stdlib).
- **Agents:** Vector, Fluent Bit, Fluentd, Promtail, Datadog Agent.
- **Aggregators:** Loki, Elasticsearch / OpenSearch, Splunk, Datadog Logs.
- **Search/analysis:** Grafana, Kibana, Splunk SPL.

### Adjacent reading
- [Metrics & Time-Series →](./metrics.md)
- [Distributed Tracing →](./tracing.md)
- [The Three Pillars of Observability →](./three-pillars.md)
- [Centralized Log Aggregation →](./log-aggregation.md)
- [Alerting & On-Call →](./alerting.md)
- [Health Checks & Heartbeats →](./health-checks.md)

---

*Up:* [README ↑](../README.md)  |  *Next:* [Metrics & Time-Series →](./metrics.md)
