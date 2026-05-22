# Common Mistakes to Avoid

> **TL;DR** — Most system design interview failures are **process mistakes, not knowledge gaps**. The candidate who knows Cassandra cold but jumps straight to the diagram without scoping the problem will lose to the candidate who knows less but **clarifies, estimates, structures, and trades off out loud**. The recurring patterns: skipping requirements, drawing in silence, over-engineering, name-dropping technologies without justification, hedging instead of deciding, forgetting non-functional requirements, and running out of time. This page catalogs the most common pitfalls so you can spot them in yourself before the interviewer does.

---

## 1. Skipping Requirements Clarification

The single most common failure. The prompt is "Design Twitter," and the candidate starts drawing services thirty seconds later.

**Why it fails**: you don't know what scale you're designing for, what features are in scope, or what the success criteria are. You're solving a problem the interviewer didn't ask.

**Fix**:
- Spend the first 5 minutes on requirements.
- Distinguish functional (tweet, like, follow) from non-functional (latency, availability, scale).
- Restate scope back to the interviewer.

See [Functional vs Non-Functional Requirements →](../01-foundations/functional-vs-non-functional.md) and [How to Approach a System Design Interview →](../01-foundations/interview-approach.md).

---

## 2. Drawing in Silence

You're at the whiteboard for two minutes, drawing. The interviewer is taking notes. Neither of you is talking.

**Why it fails**: the interviewer scores during the work, not afterward. If they can't see your reasoning, they can't credit it.

**Fix**: narrate every element. "I'm adding a cache here because we have a 100:1 read-write ratio." Every box gets a sentence.

---

## 3. Over-Engineering for the Stated Problem

The prompt is "Design a URL shortener." The candidate proposes Kafka, ten microservices, multi-region replication, and a service mesh.

**Why it fails**: signals you can't size effort. Real engineers ship the simplest thing that works, then evolve. Sophistication is not a virtue; appropriate sophistication is.

**Fix**: start with the simplest design that meets the requirements. Add complexity in response to specific constraints — never preemptively.

> "I'd start with Postgres and a small Redis. If we hit 100K writes/sec I'd move to Cassandra. For our 1K writes/sec target, Postgres is enough."

---

## 4. Name-Dropping Technologies Without Justification

"I'd use Kafka, Cassandra, Redis, Elasticsearch, Kubernetes, Istio."

**Why it fails**: you're listing tools. The interviewer doesn't know if you understand what any of them do or why they fit.

**Fix**: for each technology, explain **what role it plays and why it beats the alternative**:

> "I'd use Kafka as the durable event bus between order intake and the inventory and notification consumers. I considered RabbitMQ but want partition-ordered replay for reconciliation; Kafka's log model fits that better."

---

## 5. Forgetting Non-Functional Requirements

Candidates love functional requirements (users can do X). They forget the SLAs that drive most of the actual engineering: latency, availability, durability, consistency, cost.

**Why it fails**: a perfectly correct system that's down 30% of the time isn't a working system. Non-functionals shape the architecture more than features.

**Fix**: explicitly state targets. p99 latency < 200 ms. 99.99% availability. Durability "never lose a paid order." These constraints drive every later decision.

---

## 6. Hedging Without Choosing

> "We could use Postgres, or Cassandra, or DynamoDB, or MongoDB, depending on the access patterns. It depends..."

**Why it fails**: the interviewer learns nothing about your judgment. The "depends" is the easy part; the *decision* is what matters.

**Fix**: present the options briefly, then pick. State the trade-off. Move on.

> "Three options here: Postgres for simplicity, Cassandra for write throughput, DynamoDB if we want managed. I'm picking Postgres because we're at 5K writes/sec and the relational model fits. If we hit 50K writes/sec on this path, I'd revisit."

---

## 7. Missing the Bottleneck

The interviewer asks "what if 10x?", and the candidate proposes scaling things that weren't the bottleneck.

**Why it fails**: shows you don't understand where load actually lands. You're rehearsing the playbook, not applying it.

**Fix**: identify the actual bottleneck first. Use your back-of-envelope numbers. See [Scaling Questions →](./scaling-questions.md).

---

## 8. Premature Database Sharding

Reaching for sharding when a single primary would handle the load.

**Why it fails**: sharding is operationally expensive. Cross-shard joins are painful. Re-sharding is a rite of passage. You don't shard until you must.

**Fix**: vertical scale first. Read replicas. Cache. Then shard. Postgres handles a stunning amount of traffic on big hardware.

---

## 9. Ignoring Failure Modes

The happy path is described in glorious detail. What happens when the cache is down? When the third-party API times out? When a region fails?

**Why it fails**: in real production, everything breaks. Senior interviewers grade heavily on failure thinking.

**Fix**: after the happy path, walk through:
- What if this service is down?
- What if the network partitions?
- What if the queue backs up?
- What if a worker crashes mid-job?

See [Fault Tolerance →](../11-reliability/fault-tolerance.md), [Circuit Breaker →](../11-reliability/circuit-breaker.md), [Retry, Timeout, Backoff →](../11-reliability/retry-timeout-backoff.md).

---

## 10. Forgetting Operational Reality

The design is beautiful but never mentions deployments, monitoring, on-call, cost, or schema migrations.

**Why it fails**: code is the easy part. Running it is the job. Senior signals come from talking about ops.

**Fix**: mention briefly:
- How do we deploy this safely? (canary, blue-green)
- How do we know it's broken? (metrics, alerts, traces)
- What does on-call look like?
- What does this cost to run?
- How do we evolve the schema without downtime?

See [Deployment & Infrastructure →](../15-deployment/cicd.md), [Observability →](../13-observability/three-pillars.md).

---

## 11. No Estimation

You're asked to design for "high scale" and propose architectures without quantifying anything.

**Why it fails**: every decision depends on the numbers. "I'd cache this" depends on hit rate, key cardinality, value size, latency budget. Without numbers, you're guessing.

**Fix**: take 60 seconds to estimate.
- Users.
- QPS = users × ops/user / seconds-per-day.
- Storage = items × bytes × retention.
- Bandwidth.

See [Back-of-the-Envelope Estimation →](../01-foundations/estimation.md) and [Latency Numbers →](../01-foundations/latency-numbers.md).

---

## 12. Treating "Eventual Consistency" as a Magic Wand

> "I'd use eventual consistency."

The interviewer asks what that means for the user, and you stall.

**Why it fails**: "eventual" without specifying how long, what the user sees in the meantime, and how the system reconciles is hand-waving.

**Fix**: be specific.

> "Eventual consistency here means after a tweet is posted, it shows in your own profile within ~1 second (read-your-writes via primary) but in other users' timelines within ~5–10 seconds as fan-out completes. We're choosing this because strong consistency would cap our fan-out at the slowest replica."

See [Consistency Models →](../08-distributed-systems/consistency-models.md).

---

## 13. Ignoring the Interviewer's Hints

The interviewer says "interesting, what about the case where two users edit at once?" and you ignore it and keep going.

**Why it fails**: they're telling you where they want to dig. Following hints lets you spend time on what they want to evaluate.

**Fix**: when they ask, dig. When they linger on a topic, that's the topic.

---

## 14. Running Out of Time

You spent 25 minutes on requirements and high-level, then have 5 minutes left and haven't done a single deep-dive.

**Why it fails**: the deep-dive is where senior signal lives. Surface-level coverage of everything beats nothing on anything, but a mix of coverage and depth beats both.

**Fix**: budget. Aim for ~20 minutes on deep-dives. Use a rough timetable (see [Driving the Conversation →](./driving-conversation.md)).

---

## 15. Capitulating Under Pushback

Interviewer: "Are you sure that's right?"
You: "Oh, no, you're right, let me change it."

You weren't wrong. They were testing.

**Why it fails**: signals lack of conviction. Senior engineers hold positions they can defend.

**Fix**: ask what's making them push back. If they offer a substantive reason, update. If they just push, hold.

> "I'm pretty confident in this — the reason I picked it is X. Is there a constraint I missed that would change the calculus?"

---

## 16. Forgetting Security and Privacy

No auth in your API. PII flows in plaintext. Nothing about rate limiting or DDOS.

**Why it fails**: real systems live in a hostile environment. Forgetting security is forgetting half the problem.

**Fix**: mention auth, encryption (in transit + at rest), rate limiting, secrets management. Briefly is fine.

See [Security →](../12-security/authn-vs-authz.md).

---

## 17. Going Quiet When Stuck

You don't know the answer. Instead of working through it, you freeze.

**Why it fails**: interviewers care more about how you think than whether you know. Going silent removes their only signal.

**Fix**: think out loud.

> "I'm not sure of the best algorithm here off the top of my head. Let me reason through it. This is a top-K problem with high write volume, so candidates are sorted sets, count-min sketch, or stream aggregation. Given exact counts are needed, I'll go with sharded sorted sets."

You arrived somewhere reasonable by visibly thinking.

---

## 18. Cheat Card

```
PROCESS    Clarify requirements first.
           Estimate before designing.
           Narrate as you draw.
           Trade off out loud.
           Manage the clock.

CONTENT    Start simple, evolve.
           Identify bottlenecks, not "scale everything."
           Mention failure modes.
           Mention ops (deploy, observe, cost).
           Specify what consistency / latency mean.

DON'T      silent drawing, name-drop without why,
           hedge without choosing, ignore non-functionals,
           capitulate under pushback, freeze when stuck.

DO         scope, quantify, narrate, decide, defend,
           preview your own gaps before being asked.

RULE       Most failures are process failures.
           Strong process beats strong memorization.
```

---

## Resources

### Books
- *System Design Interview Vol. 1 & 2* — Alex Xu.
- *Acing the System Design Interview* — Zhiyong Tan.
- *Cracking the Coding Interview* — Gayle Laakmann McDowell.
- *Designing Data-Intensive Applications* — Kleppmann (for the technical depth that lets you decide quickly).

### Articles
- "How to crush a system design interview" — various FAANG engineering blogs.
- "What we look for in a system design interview" — Amazon Leadership Principles in practice.
- "Mock interview retrospectives" — Hello Interview, Interviewing.io.

### Videos
- Hello Interview / ByteByteGo mock interview videos — watch the *bad* ones; mistakes are more instructive than wins.
- "How not to fail your system design interview" — Pramp talks.

### Adjacent reading
- [Communicating Trade-Offs →](./tradeoffs.md)
- [Driving the Conversation →](./driving-conversation.md)
- [Drawing System Diagrams →](./diagrams.md)
- [Handling "Scale 10x" Follow-Ups →](./scaling-questions.md)
- [Senior vs Staff vs Principal Bar →](./level-bars.md)
- [How to Approach a System Design Interview →](../01-foundations/interview-approach.md)

---

*Previous:* [← Handling "Scale 10x" Follow-Ups](./scaling-questions.md)  |  *Next:* [Senior vs Staff vs Principal Bar →](./level-bars.md)
