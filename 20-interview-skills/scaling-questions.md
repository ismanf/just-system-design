# Handling "What if scale 10x?" Follow-Ups

> **TL;DR** — When the interviewer asks "what if we 10x?", they're testing whether you understand **where your current design breaks**. The answer isn't "I'd scale everything." The answer is **"first, this one component will fail — here's how and here's the fix."** Strong candidates name a specific bottleneck (DB writes, single Redis, fanout job, hot key), explain the symptom, then describe the surgical fix — sharding, caching, async, partitioning. Then they pause and let the interviewer ask the next "and then 10x more?" The whole game is identifying the *next* bottleneck before the interviewer does.

---

## 1. What the Question Is Really Asking

"What if scale 10x?" is shorthand for:
- Do you know where your design's weakest link is?
- Do you have a vocabulary for scaling specific components?
- Can you reason about how parts of the system grow at different rates?

It's not "redesign everything." It's "show me you know what would break."

---

## 2. The Universal Scaling Playbook

There are ~7 things you can do to scale a component. Memorize the menu:

| Bottleneck | Fix |
|---|---|
| Single DB primary write QPS | **Shard** by key |
| Single DB read QPS | **Read replicas** + cache |
| Hot row / hot key | **Sharded counters / replicated hot keys** |
| Synchronous slow path | **Move to async via queue** |
| Single Redis | **Redis Cluster** (consistent hashing) |
| Single service instance | **Horizontal scale** behind LB |
| Slow page render | **CDN / edge cache** |
| Compute-heavy operation | **Pre-compute / batch / cache results** |
| Network bandwidth at one node | **Spread across regions / partition** |
| Stateful service contention | **Make stateless; push state to DB/Redis** |

When asked "10x?", scan your diagram for which of these applies and call it out.

---

## 3. The Pattern of a Good Answer

Three steps:
1. **Name the bottleneck**.
2. **Explain the symptom** (what would fail and how).
3. **Apply the fix**.

Example:

> Interviewer: "What if traffic 10x'd to 100K req/sec?"
>
> You: "The DB primary would be the first thing to fall over. Right now we're at maybe 5K writes/sec on a single Postgres — at 50K writes/sec, we'd hit IO and lock contention. I'd shard by user_id; with eight shards we're at ~6K each, headroom intact. Reads I'd offload to replicas plus a Redis cache layer in front. After sharding, the next constraint is probably the load balancer or our connection pools."

You named the bottleneck (DB primary), explained why (writes/sec + IO), and fixed it (sharding + replicas + cache). Then you previewed the next bottleneck. That's the senior pattern.

---

## 4. Where Bottlenecks Usually Appear First

Order of likely failure as you scale (varies by problem, but a useful default):

1. **Single primary DB writes** — write QPS ceiling.
2. **Synchronous external calls** in the request path — payment, third-party API.
3. **Hot keys / hot partitions** — viral content, popular SKU.
4. **Single cache node** — Redis on one box.
5. **Fanout amplification** — celebrity tweet → million list inserts.
6. **Load balancer / app tier** — usually easiest to fix (horizontal scale).
7. **Network egress** — single-region capacity.

Walk through this list mentally when asked "scale 10x."

---

## 5. The "Different Components Grow Differently" Insight

A great answer acknowledges that not every component scales at the same rate.

> "At 10x reads but the same writes, my DB primary is fine — I just need more replicas and more cache. The write path doesn't change."

> "At 10x users but the same per-user activity, I need horizontal scale on the app tier but my data model holds."

> "At 10x messages per user, my fan-out worker becomes the bottleneck — not the storage."

This nuance separates "knows the playbook" from "thinks about systems."

---

## 6. Common Scaling Scenarios

### 6.1 "What if reads 10x?"
- Read replicas.
- Cache layer (Redis).
- CDN if cacheable.
- Denormalize for read efficiency.

### 6.2 "What if writes 10x?"
- Shard primary.
- Async writes (queue + worker).
- Switch to LSM storage if read patterns allow.
- Batch writes (debounce).

### 6.3 "What if a single user 10x's their activity?" (hot key)
- Sharded counter for that user.
- Per-host local cache.
- Reject or rate-limit if abusive.

### 6.4 "What if data 10x's?" (storage)
- Shard / partition.
- Tiered storage (hot / cold).
- Cold archive to object storage.

### 6.5 "What if latency requirement 10x's tighter?"
- Move computation closer (edge / CDN).
- Pre-compute (no work at request time).
- In-memory storage.
- Tail-latency mitigation (hedged requests).

---

## 7. The "And Then What?" Follow-up

Interviewers chain. After you fix one bottleneck, they ask about the next.

> Interviewer: "Great, sharded the DB. Now scale 10x more."
>
> You: "Now my shards are at 60K writes/sec each, which is the same problem at a new layer. I have two options: re-shard (more shards), or move write-heavy operations to async via Kafka. Given the workload — these are user posts, not financial — I'd queue the writes and have workers persist asynchronously. The user response returns immediately."

Each "10x" should peel back another layer.

---

## 8. Pre-empt the Question

Strong candidates announce scaling concerns proactively:

> "I'm using a single Redis here. At our current scale that's fine, but if we hit 100K ops/sec we'd need to move to Redis Cluster or a sharded setup."

You've called out the limitation before being asked. The interviewer mentally checks the box.

---

## 9. When Numbers Don't Add Up

If the interviewer asks "10x more" and the system genuinely doesn't need to change, say so:

> "Honestly at 10x we're still well within Postgres's comfort zone — we'd go from 5K to 50K writes/sec, and a beefy Postgres handles 100K. I wouldn't shard yet; that's premature complexity. If we hit 200K I'd revisit."

Knowing when **not** to scale is a senior signal. Premature sharding is a real failure mode at startups.

---

## 10. The "Vertical Before Horizontal" Move

Sometimes the right answer is "bigger box":

> "Before sharding, I'd vertically scale the primary. m5.24xlarge gives us substantially more headroom. Sharding is operationally expensive — I'd defer it until vertical scaling stops paying off."

Postgres on modern hardware is wildly capable. Many designs don't need to shard until you're at significant scale. Saying "let's go vertical first" is mature.

---

## 11. Scaling People and Operations, Not Just Machines

For staff-and-above interviews, "scale 10x" sometimes means org / process:

> "At 10x the team size, we'd need to break this monolith into bounded contexts. Otherwise PRs queue up and deploys conflict. Each domain gets its own service and team."

This is the [microservices →](../14-architecture/microservices.md), [bounded contexts →](../14-architecture/bounded-contexts.md), and Conway's-law conversation. See [Senior vs Staff vs Principal →](./level-bars.md).

---

## 12. Common Mistakes

- **"I'd add more servers."** Specific to what? Doing what? This is a non-answer.
- **"I'd use Kafka."** Kafka where? For what? Why?
- **Sharding everything immediately.** Premature.
- **Ignoring hot keys.** They're often the real bottleneck even at moderate scale.
- **Not previewing the next bottleneck.** The interviewer will, so beat them to it.
- **Believing all components scale together.** They don't.
- **Forgetting cost.** "I'd 10x the infra" can mean 10x the AWS bill. Senior engineers consider that.

---

## 13. Cheat Card

```
PATTERN     1. Name the bottleneck (be specific).
            2. Explain the symptom (what fails, how).
            3. Apply the fix (specific technique).
            4. Preview the next bottleneck.

PLAYBOOK    Reads → replicas + cache + CDN
            Writes → shard / async / batch
            Hot key → sharded counter / per-host cache
            Storage → partition + tier
            Latency → edge / pre-compute / in-memory

ORDER       DB primary writes → sync external calls → hot keys
            → cache nodes → fanout → app tier → network egress

DON'T       "add more servers", premature sharding,
            ignoring cost, treating all components as one.

RULE        Where does the first thing break?
            Fix surgically. Then preview the next break.
```

---

## Resources

### Books
- *Designing Data-Intensive Applications* — Kleppmann.
- *Database Internals* — Alex Petrov.
- *Site Reliability Engineering* — Google.
- *Release It!* — Michael Nygard (failure modes at scale).

### Articles
- "Architecting for Scale" — High Scalability blog archives.
- "Premature Optimization is the Root of All Evil" — Knuth (relevant: don't shard early).
- Cloud architecture frameworks: AWS Well-Architected, Google Cloud Architecture Center.

### Videos
- ByteByteGo: "Scaling" series.
- "Scaling Instagram Infrastructure" — Mike Krieger talks.
- "How Discord Scaled Elixir to 5M Concurrent Users".

### Adjacent reading
- [Communicating Trade-Offs →](./tradeoffs.md)
- [Driving the Conversation →](./driving-conversation.md)
- [Senior vs Staff vs Principal Bar →](./level-bars.md)
- [Back-of-the-Envelope Estimation →](../01-foundations/estimation.md)
- [Capacity Planning →](../10-scalability/capacity-planning.md)
- [Sharding & Partitioning →](../04-databases/sharding-partitioning.md)
- [Hot Partition Problem →](../10-scalability/hot-partitions.md)
- [Backpressure →](../10-scalability/backpressure.md)

---

*Previous:* [← Drawing System Diagrams](./diagrams.md)  |  *Next:* [Common Mistakes to Avoid →](./common-mistakes.md)
