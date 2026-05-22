# Feature Flags & Dark Launches

> **TL;DR** — A **feature flag** is a conditional in your code controlled by configuration rather than by deployment. Flip the flag, change behavior — no rebuild, no redeploy. **Dark launches** are the technique of shipping new code paths behind flags that are off (or visible only to internal users) and turning them on gradually. Together, flags decouple **deploy** from **release** — you ship code dozens of times a day but turn features on for users on your own schedule, sometimes weeks or months later. The win: smaller blast radius, instant kill switches, true A/B experiments, gradual rollouts independent of infrastructure. The cost: flag debt, untested code paths, and the operational burden of running a flag platform. **Treat flags as production state, not config**, and prune them aggressively — every flag is technical debt with a half-life.

---

## 1. The big picture

Two timelines that used to be coupled:

```
[ Deploy ]  ──────────── code goes to prod
                │
                ▼
[ Release ]  ─────────── users see the feature
```

A feature flag splits them:

```
[ Deploy ]    Mon  ──── code goes to prod (flag = off)
                          │
                          │     (days, weeks)
                          ▼
[ Release ]   Thu  ──── flag flipped on for 5%
              Fri  ──── flag flipped on for 100%
                          │
              months ───  flag removed from code
```

You can ship constantly, release deliberately. The change in mental model is fundamental: **deploys become low-stakes, releases become a product decision**. Most modern engineering orgs above ~30 engineers run on this model.

---

## 2. Why flags exist

The original motivation: **deploys are risky, but you have to deploy to ship**. If every deploy potentially exposes users to a bug, you deploy as little as possible. That breeds large, risky deploys — the opposite of what you want.

Flags break the loop. Code goes to production constantly, but only opens to users when a separate, reversible flip says so. Specifically, flags enable:

- **Kill switches** — a critical bug is found in production. Flip the flag off in seconds, no deploy needed.
- **Gradual rollouts** — 1% → 5% → 25% → 100%, like canary but at the code-path level rather than the instance level.
- **Internal-first / dogfood releases** — flip on for `@company.com` users for a week before public release.
- **A/B experiments** — split users into cohorts, measure outcomes, decide which variant wins.
- **Targeting** — different behavior per region, per plan, per tenant, per device.
- **Long-running migrations** — both old and new code paths in the codebase for months while you migrate carefully.
- **Trunk-based development** — work merges to main behind flags, no long-running branches.

That last one is the silent superpower. Without flags, large teams need feature branches, and feature branches kill velocity. With flags, every PR merges to main; the feature simply isn't live yet.

---

## 3. The taxonomy of flags

Pete Hodgson's classification (Martin Fowler's site) is canonical. Different flag categories have different lifespans and management strategies:

| Flag type | Purpose | Typical lifespan |
|---|---|---|
| **Release flag** | Hide a feature in progress; turn on at launch | Days to weeks (then delete) |
| **Experiment flag** | A/B test variants, measure outcomes | Weeks (then delete the loser) |
| **Ops flag** | Operational kill switch, throttle, circuit breaker | Months to years (keep around) |
| **Permission flag** | Entitlement gate (paid plan, beta cohort) | Permanent (it's a business rule) |

The killer mistake is treating all four the same. Release flags must die. Permission flags live forever. Mixing them in one codebase, one platform, one process is how you end up with 4,000 flags and no way to tell which ones still matter.

---

## 4. Anatomy of a flag check

A simple evaluation:

```python
if flags.is_enabled("new_checkout_flow", user=current_user):
    render_new_checkout()
else:
    render_legacy_checkout()
```

What `is_enabled` does, under the hood:

1. Look up the flag's current rule set (cached locally — typically polled every 30–60s from the flag service).
2. Evaluate the rules against the **context** — user ID, email domain, country, plan, device, custom attributes.
3. Apply percentage rollout (deterministic hash of user ID + flag key → 0–100, compare to threshold).
4. Return boolean (or a variant: `"control"`, `"variant_a"`, `"variant_b"`).

The deterministic hashing matters: **the same user must get the same answer every time** for a given flag state. Otherwise users flicker between variants and break their own experience.

```python
# Rough sketch of percentage rollout
def evaluate_percentage(flag_key, user_id, percentage):
    h = hash(flag_key + ":" + user_id) % 100
    return h < percentage
```

---

## 5. A working flag platform

You can roll your own with a Postgres table and a Redis cache. Most teams above ~20 engineers buy or adopt a platform:

| Platform | Notes |
|---|---|
| **LaunchDarkly** | Market leader. Targeting, experiments, sophisticated audit. Expensive. |
| **Statsig** | Strong analytics integration; founded by ex-Facebook flag-platform team. |
| **Optimizely** | Heavy on experimentation/A-B. |
| **Split** | Mid-market, decent feature parity. |
| **Unleash** | Open source (also commercial cloud). |
| **Flagsmith** | Open source, self-hostable. |
| **GrowthBook** | Open source, experiment-focused. |
| **ConfigCat** | Simple and inexpensive. |

Internal builds happen at scale — Facebook's GateKeeper, Google's Internal Tools, Uber, Netflix, etc. — but they're heavy investments. Use a vendor or open-source platform unless you have unusual constraints (regulatory isolation, ultra-low-latency edge eval).

### Server-side vs client-side eval

- **Server-side** (the safe default): the server evaluates flags, the client just calls APIs. Rules and user attributes never leave the server.
- **Client-side**: SDK in the browser/mobile fetches flag rules and evaluates locally. Faster UX, but the rules ship to the user. Don't put confidential targeting rules (e.g., "beta users we want to keep secret") in client-side flags.

### Edge eval

Recent trend: evaluate flags at CDN/edge (Cloudflare Workers, Fastly, Vercel Edge) for sub-millisecond latency. Useful for personalization at the edge — A/B tests on the landing page, A/B tests in HTML responses.

---

## 6. Worked example — a gradual rollout

```python
# Day 1: flag created, default off, code deployed
def checkout(user):
    if flags.is_enabled("new_checkout", user=user):
        return new_checkout(user)
    return legacy_checkout(user)

# Day 1 evening: flip on for internal users
# rule: email ends with @acme.com → on
# 24 hours of dogfooding

# Day 3: flip on for 1% of users
# percentage rollout: 1
# watch error rate, conversion, p99 — for 24 hours

# Day 5: flip on for 10%

# Day 8: flip on for 50%

# Day 10: flip on for 100%

# Day 30: code review — remove the flag entirely
def checkout(user):
    return new_checkout(user)
```

This is just canary, but at the **code-path** level rather than the **infrastructure** level. The difference is profound: you can canary inside a single binary, without orchestrator changes, with rule logic far more flexible than weighted load balancing.

---

## 7. Dark launches

A **dark launch** is shipping new code so that it runs in production but produces no user-visible effect. Used to:

- **Test performance at scale** — run the new code path on real traffic, discard the output. Measure CPU, memory, latency, downstream load.
- **Validate correctness** — run new and old in parallel, compare outputs (a "diff test"), log differences.
- **Warm up caches** — populate caches with new-shape data before flipping users over.

```python
result_legacy = legacy_path(req)
if flags.is_enabled("dark_new_path"):
    try:
        result_new = new_path(req)
        if result_new != result_legacy:
            metrics.increment("dark_launch.divergence")
            log.warn("divergence", legacy=result_legacy, new=result_new)
    except Exception as e:
        metrics.increment("dark_launch.error")
return result_legacy  # always return the legacy result
```

Famous use: Twitter's migration to a new timeline service, Facebook's relentless dark-launches before flipping any major feature, Stripe's API migrations.

The discipline: never let the dark launch affect user-visible behavior. If the new code path writes data, write to a shadow table, not the real one.

---

## 8. Targeting — beyond simple on/off

A mature flag platform supports rule-based targeting:

```
flag: new_checkout
rules:
  - if user.email ends_with "@acme.com" → ON
  - if user.country in ["US", "CA"] AND user.plan == "enterprise" → ON
  - else: percentage_rollout(15%)
default: OFF
```

Common dimensions:

- User ID, email, role
- Country, region, city
- Device type, OS, app version
- Plan, tenant, organization
- Custom attributes you pass at evaluation time

This is also where flags shade into segmentation and personalization. The line blurs intentionally.

---

## 9. A/B experiments

Flags are the substrate for experimentation:

```python
variant = experiments.assign("checkout_layout_2026", user=current_user)
# variant ∈ {"control", "variant_a", "variant_b"}

if variant == "variant_a":
    render_layout_a()
elif variant == "variant_b":
    render_layout_b()
else:
    render_control()

# Log assignment for analysis
analytics.track("variant_assigned", flag="checkout_layout_2026",
                variant=variant, user=current_user.id)
```

The assignment must be:

- **Deterministic** per user (same user → same variant).
- **Recorded** at assignment time, so the analysis can attribute outcomes.
- **Independent** of other experiments unless you explicitly want interaction.

Then you measure: did variant A's conversion beat control? Did variant B's latency regress? Statistical machinery — minimum sample size, p-values, sequential testing — sits on top. Statsig and Optimizely specialize in this layer; GrowthBook is the strong open-source alternative.

---

## 10. The flag debt problem

Every flag is a branch in production. Every branch is something that could go wrong. Over time, a flag platform accumulates thousands of flags, many for features long since launched and forgotten. This is **flag debt**, and it's the most predictable failure mode of flag adoption.

### What flag debt looks like

- The code has `if flags.is_enabled("v2_billing")` checks in 30 files, and the flag has been at 100% for two years.
- A developer flips a flag off because it "looks unused," and breaks production.
- An expired flag with no default behavior throws an exception on lookup.
- Tests don't cover the off path because nobody remembers the off path matters.
- New engineers spend hours figuring out what a flag does.

### How to manage it

```
[ ] Every flag has an owner (team, not person)
[ ] Every flag has a category (release / experiment / ops / permission)
[ ] Every release flag has a removal date set at creation
[ ] Stale flags surface in a weekly report
[ ] Removing a flag is a normal PR, not an exotic event
[ ] Tests cover both on and off until removal
[ ] CI lints for "is_enabled with no fallback"
[ ] An expired flag returns its default; doesn't error
```

The platform should make removal easy. If you have to file three tickets to delete a flag, nobody will.

---

## 11. Operational concerns

- **Latency.** A flag check on the request path must be fast. Cache rules locally in the SDK (poll every 30–60s, push via SSE for instant updates). Network round-trip on every request is a non-starter.
- **Availability.** If the flag platform is down, your app shouldn't go down. SDKs should serve stale rules and fall back to last-known values. Most platforms support this.
- **Consistency.** Two pods of the same service should evaluate the same user the same way. Deterministic hashing solves this — but make sure the SDK and the platform use the same hash.
- **Audit logs.** Every flag change is a production change. Log who changed what when. Some incidents are traced back to "someone flipped a flag in the UI without telling anyone."
- **Secret-ness of flag rules.** Targeting rules can leak business strategy ("we're launching X in country Y next month"). Server-side eval keeps them out of client code.
- **Performance of large flag sets.** Tens of thousands of flags strain SDKs (slow polls, big rule sets, memory). Sharding flags by service / namespace helps.
- **Coordination with caches and CDNs.** Cached responses include the variant assignment. Vary the cache key on the variant, or you'll serve cross-variant content.

---

## 12. Flags vs canary vs configuration

Three overlapping techniques, often confused:

| | Configuration | Feature flag | Canary deployment |
|---|---|---|---|
| Lives where | env var, config file | Flag service rules | Orchestrator / LB |
| Granularity | per service / instance | per code path, per user | per instance |
| Targeting | rare | rich (user, plan, region) | percentage of traffic |
| Lifespan | permanent | bounded (most types) | minutes |
| Reversibility | requires redeploy | instant flip | gradual or instant |
| Best for | knobs / endpoints / sizes | feature gating + experiments | infrastructure rollout |

Use the right tool. Flags aren't config; config isn't flags. A constant like "log level" is config. "Should this user see the new checkout?" is a flag.

---

## 13. Common Mistakes / Anti-Patterns

- **Flag debt.** No process for removing flags. Codebase fills with dead conditionals.
- **Using flags as config.** "max_retries" should be config, not a flag. You'll lose track.
- **No default behavior.** Flag service is down → flag lookup throws → entire request fails. Always have a safe default.
- **No audit trail.** Someone flipped a flag at 2 AM and the incident is unrelated, but you have no way to know.
- **Flag rules in client-side code.** Targeting strategy leaks to users; competitors read it.
- **Skipping the cleanup PR.** Flag is at 100%, code still has `if is_enabled(...)`. Compiler doesn't help you; only discipline.
- **Tests only cover the new path.** When the flag is off (e.g., during incident response), the off path silently broke 3 weeks ago.
- **Coupling many features to one flag.** A "rewrite_v2" mega-flag that controls 40 unrelated changes — flipping it off is now a catastrophe, not a kill switch.
- **Forgetting to invalidate caches on flip.** Users see the old behavior because their HTTP cache, CDN, or Service Worker still has the pre-flip response.
- **No staged rollout for risky flips.** Going 0% → 100% in one click negates the whole purpose. Go 1% → 10% → 100%.
- **Treating flags as deployment automation.** Flag changes are production changes. Treat them with the same review rigor as code (yes, even for ops flags).
- **Mixing experiment flags into critical path logic.** An experiment expires and starts returning default; suddenly your "control" path runs for everyone, no one notices for a month.

---

## 14. Cheat Card

```
PURPOSE   Decouple deploy from release. Ship code today, turn it
          on for users later — gradually, reversibly, per cohort.

FOUR FLAG TYPES   (manage differently)
  Release      — short-lived, delete after launch
  Experiment   — A/B test; delete after analysis
  Ops          — kill switch / throttle; long-lived
  Permission   — entitlement; effectively permanent

EVALUATION
  is_enabled(flag, context) → bool / variant
  deterministic per user (same hash every time)
  rules: user, country, plan, percentage, custom attrs
  cache rules locally; never round-trip per request

DARK LAUNCH
  Run new code path in prod, discard or compare output
  Catches perf / divergence before user-visible flip

WHEN TO USE
  Risky launches, gradual rollouts
  Kill switches for ops
  A/B experiments with statistical rigor
  Internal-first / dogfood releases
  Long-running migrations (months of mixed code)

WHEN NOT TO USE
  As ordinary config (use config)
  For deploy-time toggles (use deploys)
  Without a removal plan (use nothing)

PITFALLS
  Flag debt — no process to delete
  No safe default → outage when platform blips
  Client-side targeting rules leak strategy
  Skipping the cleanup PR
  Test coverage only on the new path
  One mega-flag controlling 40 features
  Forgetting to invalidate caches on flip

RULE   Every flag is debt. Create with an owner, a category, and
       a removal date. Treat flag flips as production changes.
```

---

## 15. Resources

### Articles
- "Feature Toggles (aka Feature Flags)" — Pete Hodgson on Martin Fowler's site: <https://martinfowler.com/articles/feature-toggles.html> (the canonical taxonomy)
- "Dark launching" — Facebook engineering (historic).
- "Trunk-based development" — <https://trunkbaseddevelopment.com>
- "Online migrations at scale" — Brandur Leach (Stripe): <https://stripe.com/blog/online-migrations>

### Documentation
- **LaunchDarkly** — <https://docs.launchdarkly.com>
- **Statsig** — <https://docs.statsig.com>
- **Unleash** — <https://docs.getunleash.io>
- **OpenFeature** (vendor-neutral spec) — <https://openfeature.dev>

### Videos
- *Decoupling deployment from release* — Jez Humble.
- *Feature flags at scale* — Statsig / LaunchDarkly engineering talks.
- ByteByteGo — "Feature Flags Explained."

### Tools
- **LaunchDarkly, Statsig, Optimizely, Split, ConfigCat** — commercial.
- **Unleash, Flagsmith, GrowthBook, FeatBit** — open source.
- **OpenFeature** — vendor-neutral SDK abstraction.
- **eppo** — open-source experimentation platform.

### Adjacent reading
- [Canary Deployment →](./canary.md)
- [Blue-Green Deployment →](./blue-green.md)
- [Rolling Deployment →](./rolling.md)
- [CI/CD Pipelines →](./cicd.md)
- [Circuit Breaker Pattern →](../11-reliability/circuit-breaker.md)
- [Graceful Degradation →](../11-reliability/graceful-degradation.md)
- [Database Migrations at Scale →](../04-databases/migrations.md)
- [Strangler Fig Pattern →](../14-architecture/strangler-fig.md)

---

*Previous:* [← Rolling Deployment](./rolling.md)  |  *Next:* [Infrastructure as Code →](./iac.md)
