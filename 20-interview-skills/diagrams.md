# Drawing System Diagrams

> **TL;DR** — A whiteboard system diagram is **not architecture art** — it's a communication tool you build incrementally while explaining the system. The most effective interview diagrams are **layered boxes and arrows in left-to-right data flow**, with components added as you talk. Keep it simple: client → load balancer → service → database. Cache, queue, and worker get added when you mention them. Resist the urge to draw the perfect diagram first and explain later — it leaves the interviewer watching you draw in silence. The diagram is a narrative aid; build it like one.

---

## 1. The Purpose of the Diagram

A diagram exists to:
1. Help the interviewer follow your thinking.
2. Anchor specific deep-dives ("let's expand this box").
3. Make scale and bottlenecks visually obvious.
4. Document trade-offs ("this arrow is async, that arrow is sync").

It does **not** exist to look pretty. It does not exist to be exhaustive. It does not exist as a self-contained artifact someone reads after the interview.

---

## 2. The Universal Skeleton

Almost every system design diagram starts with this skeleton:

```
[Client] → [Load Balancer] → [App Tier] → [Database]
```

Add as you go:
- **Cache** between app and database.
- **Queue** between app tier and async workers.
- **CDN** in front of static assets.
- **Search index** alongside the database.
- **Worker fleet** consuming from the queue.

Build it in this order. Don't pre-draw everything before explaining.

```
Step 1:   Client → API → DB
Step 2:   Client → LB → API → DB
Step 3:   Client → LB → API → Cache → DB
Step 4:   Client → LB → API → Cache → DB
                          ↓
                       Queue → Worker
```

---

## 3. Left-to-Right Flow

Read direction is left-to-right (in English). Make your data flow match.

- Client on the left.
- Storage / leaf services on the right.
- Async work below the main flow.
- Cross-cutting (auth, observability) often on top.

This isn't a rule; it's a convention that lets the interviewer parse the diagram without effort.

---

## 4. Boxes Are Services, Cylinders Are Stores

| Shape | Meaning |
|---|---|
| Rectangle | Service / component |
| Cylinder | Database, store, queue (anything stateful) |
| Cloud | External service / "the internet" |
| Stick figure or "User" label | The client |

Don't mix cylinders for services or rectangles for storage. Consistency matters.

---

## 5. Arrows Carry Information

Arrows aren't just "connection." Annotate them:

- Direction: who initiates the call?
- Protocol: HTTP / gRPC / WebSocket / Kafka topic?
- Sync vs async (dashed line for async is a common convention).
- Volume / QPS if you've quantified.

Example:
```
[API] ──HTTP POST──> [Order Service]
[Order Service] ─ ─ Kafka ─ ─> [Inventory Worker]
                   (async)
```

You don't need every annotation. One or two on the interesting arrows is plenty.

---

## 6. Build Incrementally, Narrate Constantly

The interviewer scores during the diagram-building, not after. Don't draw silently for five minutes.

Good rhythm:
> "I'm going to start with a client and an API gateway."
> *draws two boxes, arrow between them*
> "Behind the gateway, I have a service tier — let's say two services, the User service and the Order service."
> *draws two more boxes*
> "Both share a Postgres database, which I'll show as a cylinder."

Every drawing action has a sentence. The interviewer follows along.

---

## 7. Zoom In / Zoom Out

When the interviewer asks for depth, draw a "zoomed in" view next to the main diagram:

> "The fan-out worker is interesting. Let me zoom in here."

*Draws a separate cluster of boxes showing the worker's internal pipeline.*

Keep both visible if possible — the high-level overview and the deep-dive. Don't erase the overview unless you have to.

---

## 8. Distinguish Sync from Async

This is a frequent ambiguity. Spell it out:

```
[Order Service] ──> [Payment]    (sync, blocks order)
[Order Service] -- > [Email Q]    (async, fire-and-forget)
```

Dashed arrows, "(async)" labels, or explicit queues — pick one and use it consistently.

---

## 9. Label Boundaries

Use a dotted box to enclose components that share a boundary:
- Same machine.
- Same data center.
- Same trust zone (inside vs outside DMZ).
- Same blast radius (a cell in cell-based architecture).

This lets you talk about availability ("if this whole DC fails, traffic shifts to the other") without redrawing.

---

## 10. Avoid Spaghetti

Two-arrow rules:
- If every box connects to every other box, your diagram is wrong (or your architecture is). Simplify.
- Cross-cutting concerns (logging, metrics) usually don't need explicit arrows — mention them verbally.

If you find yourself drawing twelve arrows from one box, that box is probably a god-service in disguise.

---

## 11. Numbers on the Diagram

If you've quantified, drop the numbers into the diagram:

```
[Client] ──10K req/s──> [LB] ──95% cache hit──> [App] ──500 req/s──> [DB]
```

Numbers anchor scale conversations. They also catch the interviewer's eye.

---

## 12. Erase Less Than You Think

Whiteboards run out of space. Common mistake: erase the high-level diagram to draw a deep-dive, then forget what the original looked like.

Strategies:
- Compact early — use small boxes, leave room.
- Annotate to the side, not on top.
- When zooming in, write the zoom-in to one side of the main diagram.
- If you must erase, photograph mentally — sketch a small reminder somewhere.

---

## 13. Online Whiteboarding

Most interviews are now remote on shared tools (Excalidraw, Miro, CoderPad, native interview tools). Differences:
- Easier to move boxes around — use this.
- Harder to be expressive — practice using the tool.
- Some tools have shape libraries — learn them ahead of time.
- Audio cues matter more (since the interviewer may not be watching the screen every second).

Practice on the tool the company tells you they'll use. Stripe's not the same as Google's, which isn't Meta's.

---

## 14. Common Mistakes

- **Drawing the "perfect" diagram first, then narrating.** Build it as you talk.
- **Too small to read.** Aim for ~5-8 boxes per "section" of the board.
- **Inconsistent shapes.** Service boxes and DB cylinders should look different every time.
- **No arrows / undirected arrows.** Direction matters.
- **Spaghetti.** If everything connects to everything, simplify.
- **Erasing the overview to draw a detail.** Keep both.
- **Skipping the cache or queue and adding it as an afterthought.** Decide your skeleton upfront.
- **Drawing technologies, not roles.** "PostgreSQL" is fine; "MyOrderPostgresDB-prod-01" is not.

---

## 15. Cheat Card

```
SKELETON   Client → LB → App → Cache → DB
            with Queue → Worker hanging off

SHAPES     Rect = service. Cylinder = store. Cloud = external.
            Dashed arrow = async.

FLOW       Left-to-right. Storage on the right. Async below.

NARRATE    Every drawn element gets a sentence as you draw it.

ZOOM       Detail diagrams to the side of the overview. Don't erase.

NUMBERS    QPS, hit rate, replication factor — annotate the arrows.

DON'T      silent drawing, spaghetti, inconsistent shapes,
           pre-draw perfection, erase the overview.

RULE       The diagram is a narration aid.
           Build it the same way you'd tell the story.
```

---

## Resources

### Books
- *Software Architecture for Developers* — Simon Brown. The C4 model is a thoughtful approach to layered diagrams.
- *Software Systems Architecture* — Rozanski & Woods.
- *The DevOps Handbook* — Kim et al. (illustrations are clear architecture diagrams).

### Articles
- "The C4 Model for Software Architecture" — <https://c4model.com>
- "Architectural Decision Records" — Michael Nygard.

### Tools
- **Excalidraw** — <https://excalidraw.com>. The most common interview whiteboard.
- **Miro / Mural** — corporate whiteboards.
- **draw.io / diagrams.net** — for off-interview architecture work.
- **Mermaid** — diagrams as code; great for documentation.

### Videos
- Hello Interview / ByteByteGo whiteboard walkthroughs — watch how diagrams are built progressively.

### Adjacent reading
- [Communicating Trade-Offs →](./tradeoffs.md)
- [Driving the Conversation →](./driving-conversation.md)
- [Common Mistakes to Avoid →](./common-mistakes.md)
- [How to Approach a System Design Interview →](../01-foundations/interview-approach.md)

---

*Previous:* [← Driving the Conversation](./driving-conversation.md)  |  *Next:* [Handling "Scale 10x" Follow-Ups →](./scaling-questions.md)
