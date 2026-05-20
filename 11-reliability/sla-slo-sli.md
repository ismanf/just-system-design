# SLA, SLO, SLI, Error Budgets

> **TL;DR** — Four acronyms, one idea: define what "working" means with numbers, then run the system to those numbers. An **SLI (Service Level Indicator)** is a measurement — e.g., "fraction of requests served in <300 ms." An **SLO (Service Level Objective)** is the internal target for that indicator — "99.9% of requests in <300 ms over 30 days." An **SLA (Service Level Agreement)** is the contractual promise to customers, usually weaker than the SLO with money on the line. An **error budget** is *(100% − SLO)* — the allowed unreliability you can spend on risky changes. The framework forces honest conversations: when the budget is empty, ship less; when it's flush, ship more. Adopted at scale by Google, then everyone else. The hard part isn't the math — it's picking SLOs that reflect what users actually experience, and having the discipline to act on them.

---

## 1. The Four Acronyms

```
SLI   what you measure         "% of requests served < 300 ms"
SLO   what you aim for         "99.9% of requests < 300 ms over 30 days"
SLA   what you contractually   "99.5% or we credit your bill" (weaker than SLO)
      promise customers
ERR.  what you can spend       100% − 99.9% = 0.1% = 43 min/month
BUDGET on risk
```

The relationship matters:

```
              SLI ⊆ user experience
              SLO ⊆ SLI target
              SLA ⊆ SLO (always weaker)

         error budget = 100% − SLO
```

The SLA is *always* looser than the SLO. You promise less to customers than you target internally, so you have margin to detect and fix before breaching the contract.

---

## 2. The Origin Story

The framework comes from Google SRE, codified in the [SRE Book](https://sre.google/sre-book/) and made famous by the chapter "Embracing Risk." The core argument:

> 100% is the wrong reliability target for basically everything.

Why?
- The cost of going from 99.9% to 99.99% is exponential — often 10× more engineering for the same product value.
- Users don't experience your reliability in isolation. ISPs, devices, and the public internet are typically ~99% reliable. A "100% reliable" service still sees 1% user-facing failures.
- 100% is also operationally suffocating: you can't deploy, can't experiment, can't refactor.

Pick a number short of 100% that reflects user needs. Defend the gap. Use it.

The error budget is the mechanism. **Below SLO → freeze risky changes, focus on reliability. Above SLO → ship features, take controlled risks.** It turns reliability from "more is always better" into "exactly enough is the goal."

---

## 3. Picking a Good SLI

An SLI is a ratio of `good events / total events`. The trick is defining "good."

Common SLI patterns:

| SLI Type | Formula | Used for |
|---|---|---|
| **Availability** | successful_requests / total_requests | API, web tier |
| **Latency** | requests_under_threshold / total_requests | Anything with a latency budget |
| **Quality** | full_responses / total_responses | Degradable systems |
| **Freshness** | items_processed_within_lag / total | Pipelines, ETL |
| **Correctness** | correct_results / total_results | Computations, ML |
| **Durability** | objects_retrievable / objects_stored | Storage |
| **Coverage** | jobs_succeeded / jobs_scheduled | Batch / cron |

### The good-SLI checklist
1. **User-perceived.** "CPU < 80%" is *not* an SLI — users don't care about your CPU.
2. **Measurable.** From real telemetry, not estimates.
3. **Aggregatable** over a time window (typically 30 days).
4. **Compositional.** You can compute it from request-level data.
5. **Stable.** The definition doesn't change every quarter.

The single most important rule: **measure from the user's vantage point**, not the server's. A server-side 200 OK that took 12 seconds is a failure to the user. The best SLI sources, in order:
1. Client-side telemetry (real-user monitoring).
2. Synthetic probes (Pingdom, Catchpoint, internal probers).
3. Load balancer logs (least biased server-side).
4. Application logs (most biased — the request reached you, so it's already "good" by some measures).

---

## 4. Picking a Good SLO

The SLO is the target. Set it where it matters, not at a marketing-friendly 99.99%.

### Question-driven SLO setting

```
1. What's the worst the SLI can be before users complain?
   - That's roughly your SLO floor.

2. What can the current system actually achieve?
   - That's your SLO ceiling for the next quarter.

3. What does the business need from this service?
   - Critical money-handling: 99.95%+
   - Customer-facing app: 99.9%
   - Internal tools: 99% or 99.5%
   - Batch / ETL: 99% with longer measurement window

4. What can the team afford to operate?
   - 99.999% requires 24×7 on-call, multi-region, deep observability.
   - 99.9% requires good on-call and observability.
   - 99% lets engineers go home.
```

### The "nines" cheat sheet

| SLO | Downtime / 30 days | Downtime / year |
|---|---|---|
| 99% | 7h 18m | 3d 15h |
| 99.5% | 3h 39m | 1d 19h |
| 99.9% | 43m | 8h 45m |
| 99.95% | 21m | 4h 22m |
| 99.99% | 4m 19s | 52m 35s |
| 99.999% | 26s | 5m 15s |

Realistic SLOs for most production systems are between 99% and 99.99%. Three nines is "professional"; four nines is "mature operations"; five nines is "cellular architecture, multi-region active-active, dedicated SRE team, real money on the line."

### Beware false precision
"99.9%" implies you know the rate to 0.1%. Most teams don't measure tightly enough to make that meaningful. Default to one or two-decimal-place targets and revisit annually.

---

## 5. Latency SLOs

Latency SLOs require a *threshold* and a *percentile*. Two common shapes:

```
Shape A: "99% of requests < 300 ms"     ← percentile + threshold
Shape B: "p99 < 300 ms"                 ← percentile-based directly
```

Shape A is preferable for compounding (you can multiply availability and latency SLOs together) and for error budget math. Both are acceptable.

What percentile?
- **p50 / median** — useful for capacity planning, not for SLOs. Most users don't have p50 experience.
- **p95** — common for internal tools.
- **p99** — common for customer-facing APIs.
- **p99.9** — for high-stakes flows.
- **p99.99** — only when one in 10,000 users matters individually (financial transactions, certain consumer experiences).

See [Tail Latency & p99 →](../16-performance/tail-latency.md) for why "average latency" deceives.

Common pattern: **two latency SLOs** — `p95 < 200 ms` AND `p99 < 1000 ms`. Catches both typical and tail experience.

---

## 6. Error Budgets

```
SLO = 99.9% over 30 days
Total events in 30 days = 1 billion requests
Error budget = 0.1% = 1 million failed requests over 30 days
            = ~43 minutes of full downtime
            = ~30,000 failed requests per typical day
```

The error budget is a **finite resource**. You spend it on:
- Planned outages (deploys, migrations).
- Experiments (canary rollouts, feature flags).
- Unplanned incidents (this is what's left after the planned spend).

### The policy
The most valuable artifact in the framework isn't the budget number — it's the **agreed-upon policy** for what happens when the budget is spent:

```
While budget > 50%:    Ship freely. Take risks. Run chaos experiments.
While budget 25–50%:   Slow risky deploys. More review on releases.
While budget < 25%:    Freeze non-critical changes. Reliability-only work.
While budget < 0%:     Freeze all features. Focus on regressions and ops.
```

The policy must be written down, signed by product + engineering leadership, and *actually enforced*. Otherwise the framework collapses into "we have SLOs but everyone ignores them."

### Burn rate
A useful derived metric: how fast you're consuming the budget.

```
Burn rate = (current error rate) / (allowed error rate)
```

- Burn rate of 1× means you'll exactly exhaust the budget in the SLO window.
- Burn rate of 10× means you'll exhaust the 30-day budget in 3 days.
- Burn rate of 100× means you'll burn the whole budget in 7 hours.

Alerting should fire on **high burn rate** rather than absolute error rate. A 10× burn rate over the last hour means "act now or breach the SLO." See [Alerting →](../13-observability/alerting.md).

The canonical SRE pattern is **multi-window, multi-burn-rate alerts**:

```
Page if:  (1h burn rate > 14.4 AND 5m burn rate > 14.4)
        OR (6h burn rate > 6 AND 30m burn rate > 6)
Ticket if:  (24h burn rate > 3 AND 2h burn rate > 3)
            (1w burn rate > 1 AND 1d burn rate > 1)
```

These combinations balance sensitivity (catch fast-burn fires) and specificity (don't page for slow trends that may resolve).

---

## 7. SLAs — Where Money Enters

An SLA is a contract. Key properties:
- **Customer-facing.** Sets expectations.
- **Legally binding** with consequences (service credits, refunds, churn).
- **Looser than the SLO.** You promise less than you target.
- **Defined in customer-friendly terms.** Often availability or "successful request rate" — rarely p99 latency.

Typical SLA structure:

| Achieved availability | Credit |
|---|---|
| ≥ 99.95% | 0% |
| 99.0–99.95% | 10% credit |
| 95.0–99.0% | 25% credit |
| < 95.0% | 50% credit |

The SLA exists primarily to:
1. Set legal exposure boundaries.
2. Signal seriousness to enterprise customers.
3. Force internal discipline on measurement.

The SLA is *not* primarily an engineering tool. Your engineering target is the SLO. Confusing SLA and SLO is the classic mistake — engineers should not be running to the SLA number; if they are, you'll breach it.

---

## 8. Composing SLOs

Microservices complicate things. If your API depends on auth, DB, cache, and search:

```
   user_request
        │
        ▼
   ┌──────────┐
   │   API    │
   └─┬─┬─┬─┬──┘
     │ │ │ │
     ▼ ▼ ▼ ▼
   auth db cache search
```

Even if each downstream is 99.9%, the dependent composition is roughly:

```
99.9% × 99.9% × 99.9% × 99.9% = 99.6%
```

If you want the API to be 99.9%, **each downstream must be ~99.975%** — or you must make the API resilient to downstream failures (retries, caching, graceful degradation).

This is why **graceful degradation** matters so much for reliability: it breaks the multiplication. See [Graceful Degradation →](./graceful-degradation.md).

The "compose by multiplication" rule is approximate; in practice independent failures correlate (one bad deploy affects many services) and the math is more like a Bayesian network than a product. But the directional intuition holds — **deep dependency chains require very high per-link reliability**.

---

## 9. Operational Reality

### What gets measured gets gamed
- Choose SLIs that can't be trivially gamed by retrying or hiding.
- Be explicit about what counts as a "request" (background polls? health checks? failed-fast 4xx?).
- Exclude *expected* errors (4xx client errors usually don't count against availability SLO).

### Measurement window choice
- 7 days: too short for typical incident frequencies; budget rebuilds too fast.
- 30 days: standard; aligns with billing cycles.
- 90 days: smoother but slow to react.
- Rolling vs calendar: rolling is preferable — calendar resets create artificial deadlines.

### Excluding events
You'll be tempted to exclude "we did maintenance" or "AWS had an outage" from SLO accounting. Resist:
- The user doesn't care whose fault it was.
- "Excluded" minutes are how SLAs get inflated and become meaningless.
- Maintenance is part of the service quality; if you can't do it without outages, that's information.

### Cross-team SLO contracts
Service A depends on Service B. A's SLO is built on B's SLO. Make this explicit:
- Write the dependency in A's SLO doc.
- Negotiate the consumer/provider relationship.
- B owes A enough reliability for A to meet its commitments — quantified.

### The "no SLO" service
Some services don't deserve SLOs:
- Pure dev / staging.
- Best-effort batch jobs.
- Experimental features.

Don't pretend everything needs an SLO. Pick the customer-facing critical paths.

### SLO vs feature flags
Feature flags are the canonical way to spend the error budget *deliberately*:
- Roll a feature out to 1% of users; observe SLO impact; expand.
- Cheap, low-risk, reversible.
- A team without feature flags often spends the budget catastrophically rather than gracefully.

See [Feature Flags →](../15-deployment/feature-flags.md).

---

## 10. SLI Example — A Reasonable Stack

For an HTTP API:

```yaml
# Availability SLI
sli:
  name: api_availability
  numerator: >
    sum by (job) (rate(http_requests_total{
      job="api",
      status!~"5..",
      code_excluded!="429"     # rate-limit responses don't count as failures
    }[5m]))
  denominator: >
    sum by (job) (rate(http_requests_total{
      job="api"
    }[5m]))

# Latency SLI
sli:
  name: api_latency_p99
  numerator: >
    sum by (job) (rate(http_request_duration_seconds_bucket{
      job="api",
      le="0.5"               # 500 ms threshold
    }[5m]))
  denominator: >
    sum by (job) (rate(http_request_duration_seconds_count{
      job="api"
    }[5m]))

slos:
  - sli: api_availability
    target: 99.9
    window: 30d
  - sli: api_latency_p99
    target: 99.0     # 99% of requests under 500 ms
    window: 30d
```

This style maps cleanly to Prometheus + Sloth, OpenSLO, or Google's own internal tooling.

---

## 11. Common Mistakes / Anti-Patterns

- **SLIs nobody actually checks.** A dashboard nobody opens is not an SLI.
- **SLOs no one is empowered to enforce.** "Ship-anyway" policies turn the framework into theater.
- **Server-side-only measurement.** Users see what their browser/app sees; your LB doesn't.
- **Excluding maintenance windows.** Users still notice. Excluded minutes erode trust in your numbers.
- **Same SLO for all endpoints.** A login flow doesn't deserve the same SLO as a "view analytics" page.
- **Mixing client and server errors.** 4xx and 5xx mean different things; many 4xx are not your failures.
- **SLO = SLA.** Now you have no margin between internal target and external promise.
- **Setting SLOs from aspirational marketing numbers.** "We'll be 99.99%!" — but the system measured at 99.5% last quarter. You'll burn the budget Monday.
- **No error budget policy.** SLOs without consequences are decoration.
- **Counting health-check requests in the SLI.** Inflates your numbers.
- **Composing without graceful degradation.** Deep dependency chains lose nines fast.
- **Measuring only at the top.** You can't fix what you can't decompose into component SLIs.
- **Quarter-long SLO windows.** Too slow to feedback into engineering decisions.
- **Single point-in-time alerting.** Burn-rate alerts are far more useful than threshold alerts.

---

## 12. Decision Rule

```
For every customer-facing critical path:
  1. Define an SLI that reflects user experience.
  2. Set an SLO from the SLI, slightly above current performance.
  3. Compute the error budget.
  4. Write the policy for what happens at budget tiers.
  5. Alert on burn rate, not threshold.
  6. Review monthly; revise the SLO if it's never tight or always blown.

For SLAs:
  - Promise less than the SLO. Always.
  - Tie credits to thresholds that are easy to measure.
  - Include exclusions (force majeure, customer error) carefully.

For everything else:
  - Don't write an SLO you can't enforce.
  - Don't measure what doesn't reflect user experience.
  - 100% is the wrong target. Pick a number, defend it, run to it.
```

---

## 13. Cheat Card

```
PURPOSE     Define "working" with numbers. Run the system to those
            numbers. Spend the gap between SLO and 100% on velocity.

SLI         what you measure (good_events / total_events)
SLO         what you target  (e.g., 99.9% over 30 days)
SLA         what you contract (always weaker than SLO; money attached)
ERR.BUDGET  100% − SLO; the allowed unreliability

NINES       99%       3d 15h / yr   internal tools
            99.9%     8h 45m / yr   typical SaaS
            99.95%    4h 22m / yr   critical SaaS
            99.99%    52m / yr      requires real investment
            99.999%   5m 15s / yr   cellular, multi-region, SRE team

POLICY      >50% budget:   ship freely
            25–50%:        slow risky deploys
            <25%:          freeze non-critical
            <0%:           reliability-only

BURN RATE   error_rate / allowed_rate. Alert on this, not absolute.
            Multi-window pattern: short window for fast burns,
            long window for slow trends.

LATENCY     Threshold + percentile.  "99% < 300 ms" or "p99 < 300 ms"

COMPOSITION 99.9% × 99.9% × 99.9% × 99.9% ≈ 99.6%
            Graceful degradation breaks the multiplication.

PITFALLS    Server-only measurement · SLO = SLA · no policy ·
            same SLO for all endpoints · excluding maintenance ·
            counting healthchecks · aspirational SLOs

RULE        100% is the wrong target. Pick a number that reflects
            user needs and team capacity. Have a policy. Enforce it.
            The error budget is not a stretch goal — it's a tool.
```

---

## 14. Resources

### Books
- *Site Reliability Engineering* — Google. The original book; chapters 2–4 cover SLOs end-to-end.
- *The Site Reliability Workbook* — Google. Practical implementation; great chapter on SLO engineering.
- *Implementing Service Level Objectives* — Alex Hidalgo. The practitioner's manual.
- *Seeking SRE* — David Blank-Edelman, ed. Multiple perspectives on SLO adoption.

### Articles
- "Implementing SLOs" — Google SRE Workbook chapter: <https://sre.google/workbook/implementing-slos/>
- "The Calculus of Service Availability" — ACM Queue, Treynor, Dahlin, Rau, Beyer.
- "Multi-window, Multi-burn-rate Alerts" — Google SRE Workbook.
- "How to Define SLOs that Actually Matter" — Charity Majors, Honeycomb.
- "Embracing Risk" — SRE Book chapter on error budgets.
- "Beyond the Three Nines" — Liz Fong-Jones (Honeycomb).

### Videos
- Liz Fong-Jones SRE talks — many on YouTube.
- "SLOs from First Principles" — Honeycomb Hound Conf.
- "SLOs Are Not Enough" — SREcon talks.
- ByteByteGo — "SLA, SLO, SLI Explained."

### Tools
- **Sloth** — Prometheus SLO generator: <https://github.com/slok/sloth>
- **OpenSLO** — vendor-neutral SLO spec: <https://openslo.com>
- **Nobl9** — commercial SLO platform.
- **Pyrra**, **Sloth**, **slo-generator** — open-source SLO tooling.
- **Datadog / Grafana / Honeycomb / New Relic** — built-in SLO products.
- **Grafana SLO** — open-source SLO views.

### Adjacent reading
- [Alerting & On-Call](../13-observability/alerting.md)
- [Tail Latency & p99](../16-performance/tail-latency.md)
- [Graceful Degradation →](./graceful-degradation.md)
- [Failover & Disaster Recovery →](./failover-dr.md)
- [Capacity Planning](../10-scalability/capacity-planning.md)
- [Three Pillars of Observability](../13-observability/three-pillars.md)
- [Feature Flags & Dark Launches](../15-deployment/feature-flags.md)
- [Chaos Engineering →](./chaos-engineering.md)

---

*Up:* [README ↑](../README.md)  |  *Next:* [Fault Tolerance Patterns →](./fault-tolerance.md)
