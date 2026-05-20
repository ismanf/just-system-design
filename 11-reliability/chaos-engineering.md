# Chaos Engineering

> **TL;DR** — **Chaos engineering** is the discipline of deliberately injecting failures into a production (or production-like) system to discover weaknesses **before they cause outages**. The premise: complex systems fail in surprising ways; the only way to find the surprises is to look for them. Coined and operationalized at Netflix with **Chaos Monkey** (kill a random instance), the practice now spans **Chaos Kong** (kill a region), **Latency Monkey** (slow services down), **fault injection libraries** (configurable failure at the code level), and product platforms (**Gremlin**, **Chaos Mesh**, **LitmusChaos**, **AWS Fault Injection Service**). Chaos engineering is *not* breaking things for fun — it's the **scientific method applied to reliability**: form a hypothesis ("the system handles a node failure gracefully"), test it under controlled conditions, learn, fix. Done right, it surfaces the latent fragility every distributed system accumulates. Done wrong, it's an outage you caused on purpose. The discipline lives in the gap between cowardice ("we'll fail over in real outages") and recklessness ("YOLO `kill -9 1`").

---

## 1. Why It Exists

Two facts collide:

1. **Distributed systems fail in surprising ways.** The failure mode you didn't think of is the one that takes you down. Race conditions, retry storms, partial failures, cascading dependencies, cold caches, DNS quirks, certificate expirations, leader election bugs — all real, all hard to anticipate from code review.

2. **Real failures happen at the worst time.** 3 AM on a Sunday, during a holiday, in the middle of a deploy.

The Netflix observation around 2010: instead of *hoping* nothing breaks, **break things on purpose during business hours**. If a single instance death takes the service down, find out at 2 PM on a Tuesday when engineers are at their desks — not at 3 AM on Sunday during a real outage.

This is the inversion that makes chaos engineering work: **convert unknown failures into known, scheduled, observable failures**.

---

## 2. The Scientific Method

Chaos engineering is empirical science applied to reliability. The four steps:

```
1. HYPOTHESIS
   "We believe the system handles X without user-visible impact."

2. EXPERIMENT
   Inject X in a controlled way.

3. OBSERVE
   What happens? Compare to hypothesis.
   - Hypothesis confirmed: build confidence.
   - Hypothesis falsified: you found a real weakness.

4. FIX (if needed) + REPEAT
   Mitigate the weakness; run the experiment again.
```

The discipline is the hypothesis. "Let's kill a node and see what happens" is not chaos engineering — it's reckless. "We expect killing a random pod to cause no user-visible errors because of pod redundancy + load balancer health checks" is the start of a real experiment.

When the experiment confirms the hypothesis, you build justified confidence. When it falsifies the hypothesis, you've found the bug you were going to ship to production anyway — just earlier.

---

## 3. The Five Principles (Netflix)

Netflix formalized the discipline in the [Principles of Chaos Engineering](https://principlesofchaos.org/):

### 1. Build a hypothesis around steady-state behavior
Measure something user-visible (orders per second, latency, error rate). The hypothesis is about this metric, not about internal state.

### 2. Vary real-world events
Failures that happen in the wild — instance death, network latency, disk full, DNS resolution failure, dependency timeout. Not "what if our code is buggy" — actual production failures.

### 3. Run experiments in production
This is the controversial one. The point is that staging environments diverge from production in subtle ways. Chaos that works in staging may not in production; chaos that breaks production wasn't being tested in staging. **Caveat: only run in production after building confidence in lower environments.**

### 4. Automate experiments to run continuously
A one-off chaos experiment is a curiosity. Continuous chaos — Chaos Monkey running every workday — turns failure-handling into a routine property of the system rather than an aspiration.

### 5. Minimize blast radius
Start small. One instance. One AZ. One percent of traffic. Build confidence, expand scope. **The point of chaos engineering is to find weaknesses safely, not to cause outages.**

---

## 4. The Failure Catalog — What to Inject

A non-exhaustive menu of failures worth testing:

### Infrastructure failures
- **Instance / pod kill** — sudden process death.
- **Instance reboot** — slower restart, IP changes.
- **Disk full** — `/var/log` fills, can't write WAL.
- **Memory pressure** — OOM killer.
- **CPU exhaustion** — pin a CPU at 100%.
- **AZ outage** — drop all traffic to one AZ.
- **Region outage** — full regional failure.

### Network failures
- **Latency injection** — add 200 ms / 2 s / 30 s delay.
- **Packet loss** — randomly drop 1%, 10%, 50% of packets.
- **Network partition** — isolate nodes from each other.
- **Connection reset** — close TCP mid-flow.
- **DNS resolution failure** — what happens when DNS hangs?
- **Bandwidth throttling** — degrade to 100 KB/s.

### Dependency failures
- **Downstream returns 500** — all calls fail.
- **Downstream slow** — p99 → 30 s.
- **Downstream returns garbage** — malformed response.
- **Downstream times out** — never responds.
- **Auth provider down** — login fails.
- **Cache cluster down** — every read misses.

### Data plane failures
- **DB primary fails over** — application reconnects.
- **DB replication lag spikes** — read replicas stale.
- **Hot key in cache** — one key gets 100× traffic.
- **Disk corruption** — flip random bytes.

### Time and clock failures
- **Clock skew** — node thinks it's an hour ahead.
- **NTP fails** — clocks drift.
- **Certificate expiry** — TLS handshake fails today.

### Configuration / control plane failures
- **Bad config deploy** — feature flag flipped wrong.
- **Stale service discovery** — endpoints point to dead instances.
- **IAM rotation** — credentials expire mid-flight.

### "Human" failures
- **Operator runs `DROP TABLE`** — can we restore?
- **Deploy reverts** — does rollback actually work?
- **Backup restore drill** — does PITR work end-to-end?

Each category has its own family of experiments. A mature chaos program touches every row above at least quarterly.

---

## 5. The Chaos Maturity Ladder

Where teams typically progress:

```
LEVEL 0   No chaos. Real outages are the only test.

LEVEL 1   Manual game days quarterly.
          One scheduled exercise, with humans in the room.
          Finds a lot the first few times.

LEVEL 2   Automated chaos in staging.
          Daily / weekly experiments in a non-prod environment.
          Catches code-level fragility before deploy.

LEVEL 3   Automated chaos in production, business hours only.
          Single instance kills, single-AZ blackholing.
          Strict guardrails (abort criteria, blast radius limits).

LEVEL 4   Continuous chaos in production.
          Multiple experiment types running constantly.
          Failures injected without engineer involvement.

LEVEL 5   Chaos as default; experiments by hypothesis.
          New features ship with their own chaos tests.
          Failure resilience is a verified property of every change.
```

Most teams sit at Level 1 — quarterly game days. Level 3+ requires both technology (chaos tooling, observability) and culture (psychological safety to break things in production).

---

## 6. The Netflix Simian Army

The original chaos tooling, now mostly retired in name but lives on conceptually:

### Chaos Monkey
The grandfather. Randomly terminates EC2 instances during business hours. Forces every service to handle instance death.

### Chaos Kong
Simulates entire AWS region outage. Validates that traffic shifts and failover work at the regional level.

### Latency Monkey
Injects latency into RPC calls between services. Catches services that don't have timeouts or circuit breakers.

### Conformity Monkey
Audits instances for compliance (right AMIs, right tags, right config). Not chaos in the strict sense — more "configuration hygiene as scheduled enforcement."

### Janitor Monkey
Cleans up unused resources. Operational discipline as automation.

### Doctor Monkey
Detects unhealthy instances and removes them. Active health management.

### 10-18 Monkey (l10n + i18n testing)
Tests internationalization variations.

### Security Monkey
Audits for security vulnerabilities.

The lesson: **chaos can be specialized**. Different failure modes need different tooling. A "Chaos Monkey" that only kills instances tests one thing; a fleet of monkeys covers many.

---

## 7. Modern Chaos Tooling

### Open source

**Chaos Mesh** — Kubernetes-native, rich failure types (pod kill, network delay, IO chaos, time chaos, DNS chaos). The most popular K8s chaos platform.

**LitmusChaos** — CNCF project; Kubernetes-native chaos workflows.

**Chaos Toolkit** — language-agnostic CLI; YAML/JSON experiments, pluggable providers.

**ChaosBlade** — Alibaba's open chaos tool; supports K8s, host-level, and Docker.

**Toxiproxy** — TCP proxy injecting latency, bandwidth limits, slow close, etc. Great for testing connection-level resilience.

**tc / netem** — Linux traffic control; raw, powerful, hand-rolled.

**iptables / nftables** — direct packet drops and rerouting; the actual mechanism behind many chaos tools.

**Pumba** — Docker / Kubernetes container chaos.

### Commercial / SaaS

**Gremlin** — production-grade chaos engineering platform; large failure catalog; safety controls.

**AWS Fault Injection Service (FIS)** — native AWS chaos. Inject failures into EC2, RDS, ECS, EKS via templates.

**Azure Chaos Studio**, **GCP Disaster Recovery testing** — cloud-native equivalents.

### Application-level

**resilience4j fault injection**, **Hystrix fault injection**, **Polly fault injection** — code-level failure injection in resilient libraries.

**Spring Cloud Circuit Breaker / Spring Cloud Sleuth** — instrumented failure modes.

### Service mesh

**Istio fault injection** — declarative latency + abort rules. Inject failures via `VirtualService` config; no app code changes.

```yaml
spec:
  http:
    - fault:
        delay:
          percentage:
            value: 10
          fixedDelay: 5s
        abort:
          percentage:
            value: 5
          httpStatus: 500
      route:
        - destination:
            host: my-service
```

Modern service meshes make chaos a routing-config exercise. Powerful for testing without modifying applications.

---

## 8. Worked Example — A First Chaos Experiment

A team running a Kubernetes microservice wants to confirm: "Killing one pod has no user-visible impact."

```
1. Steady state
   Measure: requests/sec, p99 latency, error rate.
   Tools: Grafana dashboard, Honeycomb / Datadog traces.
   Baseline: 1000 rps, p99 200 ms, 0.1% errors.

2. Hypothesis
   "When we kill one of the N=10 replicas, error rate stays under
    0.5% for less than 30 s; p99 stays under 300 ms; requests/sec
    stays steady."

3. Blast radius
   - One pod.
   - Mid-day Tuesday.
   - Abort if error rate exceeds 5%.

4. Inject
   kubectl delete pod my-service-abc123
   (or via Chaos Mesh PodChaos manifest)

5. Observe
   - Did pod restart cleanly?
   - Did LB health checks notice and remove the dead pod?
   - Did remaining pods absorb the load without timing out?
   - Did inflight requests complete or fail cleanly?

6. Outcome
   error rate spiked to 1.2% for 8 s; p99 stayed under 250 ms.
   Hypothesis holds — but error rate higher than expected.

7. Investigate
   Inflight requests on the killed pod returned errors instead of
   being rejected pre-shutdown. Missing preStop hook and connection
   draining.

8. Fix
   Add preStop sleep, deregister from service mesh, drain
   connections, then exit. Re-run experiment. Error rate now <0.3%.
```

This is what chaos engineering looks like in practice: small experiment, observed steady state, hypothesis, controlled injection, real finding, fix, re-validation.

---

## 9. Where to Run Chaos

A practical progression:

### Local dev
Unit tests with mocked failures. Integration tests with chaos libraries. Fast, safe, limited fidelity.

### Staging / pre-prod
Chaos against a production-shaped environment. Catches integration-level fragility. Limited — traffic patterns and scale differ.

### Production canary
Inject chaos in a small canary (1% of pods, one cell, one AZ). Real production conditions; bounded blast radius.

### Production fleet
Continuous chaos against the full fleet. Highest fidelity; requires strongest tooling, observability, and safety controls.

The Netflix doctrine: **prod is the only environment that reproduces prod**. Many teams disagree; both are correct in different contexts. For a brand-new chaos program, **start in staging and graduate to prod when you have the muscle**.

---

## 10. Safety — How Not to Cause Outages

Chaos engineering must be safer than the failures it simulates. The discipline:

### Blast radius limits
- One instance at first, not the whole fleet.
- One AZ at a time, not all of them.
- One percent of traffic, not 100%.
- One service, not all services.

### Abort criteria
Predefined conditions under which the experiment auto-aborts:
- Error rate exceeds X%.
- Latency p99 exceeds Y ms.
- A dependent service's SLO is breaching.

### Steady-state validation pre-experiment
Don't start a chaos experiment when the system is already degraded. If the steady-state metrics look bad, postpone.

### Business-hour scheduling
Run experiments when engineers are awake and on call. Critical for novel experiments; less so for proven continuous chaos.

### Communication
Announce chaos experiments. Slack notifications, dashboards. So when something looks weird, ops doesn't think it's a real incident.

### Rollback / undo
Every experiment has a "stop the experiment" button. Make it one click.

### Permission to abort
Anyone on the team (including SRE on-call) can abort a chaos experiment without needing approval.

### Postmortem culture
If chaos found something, write it up. Not as a failure, but as a finding.

The whole point: chaos engineering must improve reliability, not reduce it. A program that causes more outages than it prevents is failing.

---

## 11. Game Days vs Continuous Chaos

Two complementary modes:

### Game days
- Scheduled, human-led exercises.
- Larger scope (region failover, full DB failure, complex multi-service scenarios).
- Quarterly cadence.
- High learning per session.
- Cost: significant prep + execution time.

### Continuous chaos
- Automated, frequent (hourly, daily).
- Smaller scope (one pod, one network call).
- Low cost per run.
- Catches regressions when new code is deployed.

Both are needed. Game days catch the big architectural surprises. Continuous chaos catches the day-to-day regressions ("oh, we forgot to add a timeout in the latest service").

---

## 12. Operational Reality

### Observability is a prerequisite
Without good metrics, traces, and logs, you can't see what chaos broke. **Observability comes first; chaos comes second.** A team without dashboards trying to inject failures is flying blind.

### Cultural buy-in
"We're going to break production on purpose" requires leadership support. The first time something does break, the team needs to be able to say "that's a finding, not a failure" without political consequences.

### Postmortem outcomes
Findings from chaos experiments go into the same postmortem / action-item pipeline as real incidents. Without that, chaos becomes theater.

### Day-of-prod responsibilities
Run chaos when the team can respond. Avoid the week of Black Friday. Avoid the day of a major release.

### Production environment hygiene
Resources / accounts must be tagged for chaos. Some workloads (security-critical, certified) may be excluded. Define the rules.

### Tooling investment
Chaos tooling is real engineering. Budget for it.

### Don't reinvent
Use Gremlin, Chaos Mesh, or AWS FIS unless you have a specific need they don't meet. Hand-rolled `kill -9` scripts have caused real outages.

### Coordination with other teams
Don't break the API team's service while they're shipping a release. Calendar awareness, opt-in / opt-out lists.

### "Chaos engineering, but for…" — extensions
The discipline now extends beyond infrastructure:
- **Security chaos**: inject misconfigurations, leaked credentials, audit detection.
- **Data chaos**: corrupted records, schema drift.
- **Application chaos**: feature flags flipped wrong.
- **People chaos**: simulate key engineers being unreachable.

---

## 13. Famous Real-World Examples

### Netflix Chaos Monkey (2011)
The origin story. Engineers tired of being paged for instance failures decided to terminate instances during business hours. Within months, services that didn't handle instance death were re-architected — by necessity. The practice spread across Netflix and then the industry.

### Amazon GameDays (long-running)
Internal exercises that test major failure scenarios — region outages, service-level dependency failures, customer-facing incident simulation.

### Google DiRT (Disaster Recovery Testing)
Multi-day, company-wide exercises. Entire services unplugged. Findings drive months of follow-up work.

### LinkedIn (Waterbear)
LinkedIn's internal chaos platform; tested distributed systems by killing components.

### Slack
Famous for game days that uncovered race conditions during complex multi-service interactions. Many internal blog posts describe the practice.

### Stripe
Maintains a comprehensive chaos program; engineers can request specific failure injections in production.

### GitHub
Uses chaos to validate database failover, including planned exercises with their MySQL setup.

### AWS Fault Injection Service
AWS productized the idea as a managed service after years of internal use.

---

## 14. When Chaos Engineering Helps (and When It Doesn't)

### Helps
- Distributed systems with many components.
- Workloads that need high reliability.
- Teams that have observability but suspect blind spots.
- Architectures that depend on automation (autoscaling, failover, self-healing).
- Stateful systems where failure modes are subtle.

### Doesn't help (much)
- Toy services with one component.
- Pre-launch products without users.
- Codebases that crash on every deploy already — fix the basics first.
- Teams without observability — you can't measure what chaos finds.
- Cultures that punish failure — engineers won't run the experiments.

### Anti-application
- Chaos as performance review material.
- Chaos to "prove" the system is broken before a launch (use it to find issues earlier instead).
- Chaos as a substitute for capacity planning, observability, or proper architecture.

---

## 15. Common Mistakes / Anti-Patterns

- **Chaos without observability.** You break things; can't see what happened.
- **No hypothesis.** "Let's just kill things" is reckless, not chaos engineering.
- **No abort criteria.** Experiment causes a real outage because no one defined when to stop.
- **Too-large blast radius for the first experiment.** Don't take down a whole region as the first test.
- **Running in production before building confidence in staging.** Yes, prod is special, but ladder up to it.
- **No rollback mechanism.** Experiment leaves the system in a worse state.
- **Findings ignored.** Chaos uncovers a bug; no action item; same bug paged a year later.
- **Chaos as a one-time event.** Annual game day; nothing in between. Software changes; old confidence stales.
- **Treating chaos as the SRE team's job.** Every team should run chaos against its own services.
- **Punishment culture.** Engineer broke prod with chaos; gets blamed; nobody runs chaos again.
- **Chaos to skip proper engineering.** "We'll catch it with chaos" is not a substitute for thinking through failure modes.
- **Chaos in environments unlike production.** Staging chaos that doesn't reproduce prod fragility = false confidence.
- **No comms.** Chaos experiment running; on-call thinks it's a real incident; pages everyone.
- **Excluding the most critical services.** "Too risky for chaos" — they're the ones that most need it.
- **Chaos on day-of-launch.** Bad timing makes the team look reckless.
- **Borrowing other teams' chaos plans.** Different services have different failure modes.

---

## 16. Decision Rule

```
Prerequisites:
  ✓ Observability (metrics, traces, logs)
  ✓ Known steady-state behavior
  ✓ SLOs / error budgets defined
  ✓ Runbooks for common failures
  ✓ Leadership buy-in

If yes to all → start chaos engineering.
If no to any → fix the prerequisite first.

Start with:
  - Manual game days quarterly.
  - One scenario per quarter.
  - Smallest meaningful blast radius.
  - Hypothesis → inject → observe → fix.
  - Document findings as action items.

Graduate to:
  - Staging chaos (continuous).
  - Production chaos (bounded blast radius).
  - Multi-service / multi-failure scenarios.
  - Full continuous chaos as cultural norm.

Always:
  - Hypothesis first.
  - Blast radius limits.
  - Abort criteria.
  - Postmortems on findings.
  - Improve, then re-test.
```

---

## 17. Cheat Card

```
PURPOSE     Deliberately inject failures into a (production-like)
            system to discover weaknesses before real outages do.
            Scientific method for reliability.

STEPS       Hypothesis → Experiment → Observe → Learn → Fix → Repeat

PRINCIPLES  Build hypothesis around steady-state behavior
            Vary real-world events (instance death, latency, partition)
            Run experiments in production (eventually, carefully)
            Automate continuously
            Minimize blast radius

FAILURE CATALOG
  Infrastructure   instance / disk / memory / CPU / AZ / region
  Network          latency, packet loss, partition, DNS
  Dependencies     500s, slow, garbage, timeouts
  Data plane       failover, replica lag, hot keys, corruption
  Time             clock skew, NTP fail, cert expiry
  Config           bad deploys, stale discovery, IAM rotation
  Human            DROP TABLE, bad rollback, restore drill

MATURITY    L0: no chaos
            L1: quarterly game days
            L2: continuous in staging
            L3: production chaos, business hours
            L4: continuous production
            L5: chaos as default; experiments by hypothesis

SAFETY      Blast radius limits · abort criteria · steady-state
            pre-check · business-hours · comms · one-click rollback ·
            postmortem culture

PREREQS     Observability · steady-state metrics · SLOs ·
            runbooks · leadership buy-in

TOOLS       Chaos Monkey (origin) · Chaos Mesh · LitmusChaos ·
            Chaos Toolkit · Gremlin · AWS FIS · Toxiproxy ·
            Istio fault injection · resilience4j

PITFALLS    No hypothesis · no observability · no abort criteria ·
            too-large first blast · findings ignored · one-time event ·
            punishment culture · chaos as substitute for engineering ·
            excluding critical services

RULE        Convert unknown failures into known, scheduled, observable
            ones. Test what you hope is true. The bugs you find on
            Tuesday afternoon are the outages you don't have on
            Sunday at 3 AM.
```

---

## 18. Resources

### Books
- *Chaos Engineering* — Casey Rosenthal, Nora Jones (O'Reilly). The book.
- *Learning Chaos Engineering* — Russ Miles.
- *Site Reliability Engineering* — Google. Reliability and game-day chapters.
- *The DevOps Handbook* — Gene Kim et al. Resilience and recovery practices.

### Articles
- "Principles of Chaos Engineering" — <https://principlesofchaos.org/>
- "Chaos Monkey" — Netflix tech blog, the original.
- "The Discipline of Chaos Engineering" — Casey Rosenthal.
- "Chaos Engineering at LinkedIn" — LinkedIn engineering.
- "Disaster Recovery Testing at Google (DiRT)" — Google SRE Workbook.
- "How Slack Tests in Production" — Slack engineering.
- "How Complex Systems Fail" — Richard Cook. The companion essay.
- "Gameday: Creating Resiliency Through Destruction" — Jesse Robbins (former AWS).

### Videos
- "Chaos Engineering at Netflix" — Casey Rosenthal, various conferences.
- "How to Practice Chaos Engineering" — Kolton Andrus (Gremlin).
- "DiRT: Disaster Recovery Testing at Google" — SREcon talks.
- "Game Days at Stripe" — engineering conference talks.
- ByteByteGo — "Chaos Engineering Explained."

### Tools
- **Chaos Monkey / Simian Army** — Netflix originals (Chaos Monkey still maintained as Spinnaker plugin).
- **Chaos Mesh** — CNCF Kubernetes chaos.
- **LitmusChaos** — CNCF Kubernetes chaos workflows.
- **Chaos Toolkit** — language-agnostic CLI.
- **Gremlin** — commercial chaos engineering platform.
- **AWS Fault Injection Service** — native AWS chaos.
- **Azure Chaos Studio** — native Azure chaos.
- **Toxiproxy** — TCP-level chaos for application testing.
- **Pumba** — Docker / Kubernetes container chaos.
- **Istio / Linkerd / Envoy fault injection** — service-mesh-level chaos.

### Adjacent reading
- [Fault Tolerance Patterns](./fault-tolerance.md)
- [Failover & Disaster Recovery](./failover-dr.md)
- [Graceful Degradation](./graceful-degradation.md)
- [Circuit Breaker Pattern](./circuit-breaker.md)
- [Bulkhead Pattern](./bulkhead.md)
- [SLA, SLO, SLI, Error Budgets](./sla-slo-sli.md)
- [Three Pillars of Observability](../13-observability/three-pillars.md)
- [Blast Radius & Cell-Based Architecture →](./cell-architecture.md)

---

*Previous:* [← Failover & Disaster Recovery](./failover-dr.md)  |  *Next:* [Blast Radius & Cell-Based Architecture →](./cell-architecture.md)
