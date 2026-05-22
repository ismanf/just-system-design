# Concurrency vs Parallelism

> **TL;DR** — **Concurrency** is dealing with many things at once — structuring your program so that independent units of work can make progress without waiting for each other. **Parallelism** is doing many things at once — actually running them simultaneously on different CPU cores. Rob Pike's line is the sharpest: *"Concurrency is about dealing with many things at once. Parallelism is about doing many things at once."* You can have concurrency on a single core (the OS interleaves work) and you can have parallelism without much concurrent structure (a parallel `map` over a list). They overlap but they aren't the same. Get the distinction wrong and you write code that's "multithreaded" but mysteriously not faster, or code that scales beautifully on benchmarks and deadlocks in production. The practical hierarchy: **right model for the workload → right primitives → right synchronization → measure**.

---

## 1. The big picture

```
Concurrency               Parallelism
───────────              ─────────────

[A][B][C] interleaved     [A on core 0]
on one CPU core:          [B on core 1]
A...B...A...C...B...C     [C on core 2]

= structure              = simultaneous execution
```

You can have:

- **Concurrency without parallelism** — Node.js single-threaded event loop running 10,000 connections.
- **Parallelism without concurrency** — a SIMD vector op multiplying 8 floats simultaneously inside one instruction.
- **Both** — Go program with 100,000 goroutines on 16 cores.
- **Neither** — a sequential script that does one thing at a time.

The distinction matters because the *bottleneck* differs:

| You're bound by | What helps |
|---|---|
| **I/O wait** | More concurrency (async, more threads, more sockets) |
| **CPU on one task** | More parallelism (split work across cores, vectorize) |
| **Lock contention** | Less concurrency, better data structures, less sharing |
| **Memory bandwidth** | Cache layout, less data movement, different algorithm |

Throwing parallelism at an I/O-bound problem buys you nothing. Throwing concurrency at a CPU-bound problem in a single-threaded runtime buys you nothing. Diagnose first.

---

## 2. Why the language community keeps debating this

Different runtimes pick different points on the spectrum:

| Runtime | Default model |
|---|---|
| **Node.js** | Single-threaded event loop; concurrency via async/await; parallelism via worker threads or child processes |
| **Python (CPython)** | GIL serializes Python bytecode; concurrency via asyncio/threading (great for I/O); parallelism via `multiprocessing` or C extensions |
| **Ruby (MRI)** | GIL-like (GVL); same picture as Python |
| **Java / Kotlin / Scala** | Real threads, full parallelism; concurrency via threads, virtual threads (Project Loom), futures, reactive |
| **Go** | Goroutines (M:N scheduling); concurrency and parallelism are built into the language |
| **Rust** | Real threads with strong type-system guarantees (`Send`, `Sync`); async via `tokio`/`async-std` |
| **Erlang/Elixir** | Lightweight processes, message-passing, parallel across cores |
| **C / C++** | Pthreads, OpenMP, TBB, C++ std::thread, C++20 coroutines |

The language fixes some of the trade-offs for you. Picking the wrong language for the wrong workload is one of the most expensive performance mistakes — for example, a CPU-heavy numerical service written in pure Python is the wrong tool for the job because of the GIL.

---

## 3. The five concurrency models

Different problems want different models. Pick deliberately.

### 3.1 Threads + shared memory (with locks)

The OS schedules many threads onto cores. Threads share memory; coordination uses **mutexes, condition variables, semaphores, RW locks**.

```python
# Python — illustrative; CPython GIL limits parallelism for pure-Python code
import threading
lock = threading.Lock()
total = 0
def add(n):
    global total
    with lock:
        total += n
```

**Pros**: matches what the CPU actually does. Fine-grained control. Predictable for CPU-bound work.
**Cons**: locks are hard. Deadlock, livelock, priority inversion, false sharing, races — all in play. Debugging is brutal at scale.

### 3.2 Async / event loop

A single thread handles many tasks by switching whenever one blocks on I/O. The runtime owns the loop; you write `async`/`await` (or callbacks).

```javascript
// Node.js
async function handle(req) {
  const user = await db.user(req.id);
  const orders = await db.orders(user.id);
  return { user, orders };
}
```

**Pros**: low memory per task (no stack). Excellent fit for I/O-bound concurrency (10K+ open sockets). No locks for shared state if the loop is single-threaded.
**Cons**: a CPU-bound task starves everyone else. "Function coloring" (`async` viruses through the codebase). Stack traces are harder. One uncaught exception can take the loop down.

### 3.3 M:N (green threads, goroutines, fibers, virtual threads)

User-space scheduler maps N lightweight tasks onto M OS threads. The runtime preempts cooperatively or via timing.

```go
// Go — 100k goroutines, ~8KB each
for i := 0; i < 100_000; i++ {
    go func(id int) { handle(id) }(i)
}
```

**Pros**: write straight-line code, get concurrency for free. Scales to millions of tasks. The runtime handles I/O blocking by parking the goroutine and reusing the OS thread.
**Cons**: shared memory still needs synchronization. Garbage collection is part of your performance story. Stack growth and scheduler behavior matter at scale.

Java's **Project Loom** (virtual threads, GA in Java 21) brought this model to the JVM. Same idea: write blocking-style code, runtime maps it to N OS threads. Massive deal for the Java ecosystem.

### 3.4 Actor / message passing

Independent actors with private state communicate by sending messages. No shared memory, no locks.

```elixir
# Elixir — actor-style processes
defmodule Counter do
  def loop(n) do
    receive do
      :inc -> loop(n + 1)
      {:get, from} -> send(from, n); loop(n)
    end
  end
end
```

**Pros**: avoids shared-state bugs entirely. Maps beautifully to distributed systems. Erlang/Elixir use this at the language level.
**Cons**: every interaction is a message — more overhead than a method call. Designing the actor topology and supervision tree takes practice.

### 3.5 Data-parallel / fork-join

Split a problem into independent pieces; run them in parallel; combine. SIMD, GPUs, MapReduce, parallel `map`/`reduce`, OpenMP.

```python
# Python multiprocessing for true parallelism
from multiprocessing import Pool
with Pool(processes=8) as p:
    results = p.map(expensive_fn, work_items)
```

**Pros**: simple mental model. Maps to many CPU/GPU architectures.
**Cons**: only works when work *is* independent. Coordination is implicit in the boundaries (`map`/`reduce` step).

---

## 4. Amdahl's Law and Gustafson's Law

The fundamental limits of parallel speedup.

### Amdahl's Law

If a fraction `p` of the work is parallelizable and `1 − p` is serial:

```
Speedup(N) = 1 / ((1 − p) + p/N)
```

If 90% is parallel and 10% serial:
- 4 cores: speedup ≈ 3.08×
- 16 cores: speedup ≈ 6.4×
- ∞ cores: speedup → 10× (never beats 10× because of the 10% serial)

This is why "just throw more cores at it" plateaus. **The serial fraction caps you.** Identifying that fraction in your code is much of the actual perf work.

### Gustafson's Law

Amdahl assumes fixed problem size. Gustafson observes that as machines get faster, we tackle *bigger* problems. The serial fraction often stays constant in absolute time while the parallel work grows — so practical speedup keeps scaling. In short: build for bigger problems, not just faster execution of small ones.

Both are right. The lesson: **measure your serial fraction, and either reduce it or accept the ceiling.**

---

## 5. The cost of synchronization

Adding a lock is never free. Real costs:

- **Atomic operations** — hundreds of cycles, force memory barriers.
- **Cache line bouncing** — a contended lock makes the cache line ping-pong between cores.
- **False sharing** — two unrelated fields share a 64-byte cache line; writes to one invalidate the other across cores. Symptom: scaling stops at 2–3 cores for no obvious reason.
- **Lock convoy** — many threads pile up at the same lock; throughput collapses.
- **Priority inversion** — a low-priority thread holds the lock a high-priority thread needs.
- **Contention** — when N threads compete for one lock, throughput is bounded by lock hold time. More threads makes it worse, not better.

Mitigations (rough order of effort):

1. **Don't share if you can avoid it.** Per-thread state, immutable data, partitioning.
2. **Use lock-free / wait-free data structures** when justified (atomic counters, ring buffers, lock-free queues).
3. **Pick the right lock.** RW locks for read-heavy; striped locks for high-contention maps; spinlocks only for very short critical sections.
4. **Reduce hold time.** Do work outside the critical section.
5. **Sharded counters / striped maps** — split contention by key.
6. **Reader-writer separation** — many readers, one writer.
7. **Batch updates** — N small mutations under one lock is cheaper than N lock-acquire/release.

---

## 6. The classic concurrency hazards

Names every engineer should know on sight:

| Hazard | What it is |
|---|---|
| **Data race** | Two threads access the same memory, at least one writes, without synchronization. Undefined behavior. |
| **Race condition** | Wrong outcome because operations interleave in the wrong order. Race conditions can happen without data races. |
| **Deadlock** | Two or more threads each holding a lock the other needs. Stuck forever. |
| **Livelock** | Threads keep yielding to each other, no one makes progress. |
| **Starvation** | One thread never gets the resource because others always do. |
| **Priority inversion** | Low-priority holds resource needed by high-priority. |
| **False sharing** | Adjacent fields on the same cache line, written by different cores, cause slowdown without sharing in semantics. |
| **TOCTOU** | Time-of-check vs time-of-use. The value changes between check and action. |
| **Memory reordering** | Compiler / CPU reorders reads/writes; without barriers/atomics, you see impossible states. |

Languages with strong concurrency safety (Rust's `Send`/`Sync`, Erlang's no-shared-memory) prevent whole classes of these at compile time or by design.

---

## 7. Memory models — what reads and writes actually mean

In a single-threaded program, reads and writes happen in source order. In a multithreaded program, they don't — unless you use atomics, locks, or memory barriers.

Without synchronization, a write on core 0 can be invisible to core 1 for an unbounded time. Worse, the order of two writes on core 0 can be observed in the opposite order on core 1. This is not a bug in your hardware — it's the **memory model**.

Each language has its own:

- **Java Memory Model** (JMM) — happens-before via `volatile`, `synchronized`, `final`, `j.u.c.atomic`.
- **C++ memory model** — `std::atomic` with `memory_order_*` parameters.
- **C# memory model** — `volatile`, `Interlocked`, `lock`.
- **Go memory model** — channels and synchronization primitives establish happens-before.
- **Rust** — type system enforces; `std::sync::atomic::Ordering` for atomics.

You almost never need to think in raw memory orderings if you use high-level primitives (channels, mutexes, futures). You absolutely need to when writing lock-free code. **Most engineers should stay in the high-level primitives** and call lock-free experts when there's a real bottleneck.

---

## 8. Patterns that scale

Choose patterns that avoid sharing rather than patterns that synchronize sharing.

### Worker pool

A fixed number of workers pull tasks from a queue. Bounded resource use, predictable behavior. The default shape for almost everything.

```
   ┌──────────────┐
   │   Queue      │  (channel, deque, MQ)
   └──────┬───────┘
          │
   ┌──────┼──────┬──────┬──────┐
   ▼      ▼      ▼      ▼      ▼
  W1     W2     W3     W4     W5
```

Tune pool size to the bottleneck:
- CPU-bound: workers ≈ cores
- I/O-bound (with blocking): workers >> cores; sized by external resource limits (DB connections, downstream RPS budget)

### Pipeline / fan-out fan-in

Stages connected by queues. Each stage's degree of parallelism tuned independently.

```
[Read] → [Parse] → [Transform] → [Write]
   1        N          M             1
```

Common in stream processing, ETL, log pipelines.

### Single-writer principle (LMAX disruptor, Kafka partitions)

For high-throughput single-machine systems: assign each piece of state to exactly one writer thread. Other threads send messages. Eliminates locks on the hot path.

### Sharded state

Partition shared state by key. Each shard owns its data with its own lock (or no lock at all if assigned to one thread). Scales close to linearly with shards.

### Immutable + functional

If data never changes, threads can read freely. Updates produce new versions. The cost is allocation; the win is no synchronization on the read path. Erlang, Clojure, Haskell, Scala lean here.

### CSP / channels (Go)

Pass messages, don't share memory. Go's mantra: "Do not communicate by sharing memory; share memory by communicating." Often clearer than locks for orchestration logic.

---

## 9. Picking the right number of workers / threads

This is the single most-asked tuning question.

### CPU-bound workloads

Workers ≈ **physical cores**. Hyperthreaded "logical cores" sometimes help (NUMA, hyperthreading), sometimes hurt (contention). Test both.

### I/O-bound workloads (blocking I/O)

Workers ≈ **cores × (1 + wait_time/compute_time)**.
If a task waits 90% of the time and computes 10%, you want ~10× cores worth of workers.

### Async / event-loop I/O

One event loop per core. The async runtime handles the rest.

### Database connection pools

Pool size ≈ **min(server-side max_connections, your concurrency budget)**.
Default of "200 connections" is almost always too high for Postgres — PgBouncer at 25–50 connections per shard often outperforms direct pools at 200. Measure under load. See [Connection Pooling →](./connection-pooling.md).

The general rule: **start small, watch saturation, increase until throughput stops improving or tail latency starts climbing.**

---

## 10. Concurrency on a single core — yes, it's a thing

Concurrency is about structure, not cores. A single-core machine running Node.js with 10K WebSocket connections is highly concurrent. Each connection is a logical task that makes progress; the loop interleaves them.

Conversely, a 32-core machine running a fully sequential program has zero concurrency and uses 1/32nd of the available parallelism.

The point: **concurrency is a software design decision. Parallelism is a hardware capability.** They're related but separate.

---

## 11. Distributed concurrency

Once your concurrent units live on different machines, you've inherited an entirely new problem set: partial failure, network partitions, clock skew, message reordering. See [Consistency Models →](../08-distributed-systems/consistency-models.md), [Consensus →](../08-distributed-systems/consensus.md), [CAP Theorem →](../08-distributed-systems/cap-theorem.md).

Same principles apply, though:
- Prefer message-passing over distributed shared memory.
- Partition state.
- Make operations idempotent.
- Watch for distributed deadlock (two services waiting on each other's locks).
- Choose between coordinated correctness (Paxos/Raft) and eventually-consistent throughput (CRDTs) deliberately.

---

## 12. Common Mistakes / Anti-Patterns

- **Adding threads to a GIL-bound workload.** Python threads don't parallelize CPU work in CPython. Use `multiprocessing` or a native extension.
- **Adding parallelism without measuring the bottleneck.** I/O-bound? More cores does nothing.
- **One giant lock around everything.** "It works" → throughput stops scaling at 2 threads.
- **Locking too long.** Hold-while-doing-I/O turns a lock into a serialization point.
- **Using `volatile` for synchronization** in Java/C++ when you needed full happens-before.
- **Sharing mutable state across goroutines/threads without atomics or channels.** Subtle, intermittent corruption.
- **Unbounded goroutine / task spawning.** 10M tasks → OOM. Use worker pools or semaphores.
- **Async function inside a sync caller.** Calling `.then(...)`/`asyncio.run(...)` from a sync context blocks the wrong thing. Concurrency ruined.
- **Blocking call inside an event loop.** Single-thread event loops are exquisitely sensitive to a hidden `fs.readFileSync` or `requests.get()`.
- **Connection pool too big.** Adds queuing at the DB, kills throughput. Smaller, smarter pools are usually faster.
- **False sharing in performance-critical structs.** Pad to cache-line boundaries when you have hot per-thread counters.
- **Goroutines/threads spawned but never joined.** Hidden leaks; cleanup never runs.
- **Treating thread-safety annotations as documentation.** They're not; the compiler doesn't check most of them.
- **Optimizing concurrency before correctness.** A fast wrong answer is still wrong.
- **`Thread.sleep` / `time.sleep` as a "fix" for races.** It "passes the tests"; it explodes in production.
- **Hot retries on the same lock or resource.** Adds load to a contended thing; backoff with jitter.

---

## 13. Cheat Card

```
PURPOSE   Concurrency = structure for many things at once.
          Parallelism = hardware running many things at once.
          Different problems, different solutions.

DIAGNOSE THE BOTTLENECK FIRST
  I/O bound          → more concurrency (async, more sockets)
  CPU bound on 1     → more parallelism (more cores, vectorize)
  Lock contention    → less sharing, sharded/striped state
  Memory bandwidth   → cache layout, fewer allocations

FIVE MODELS
  Threads+locks      OS threads + mutexes
  Async              event loop, single thread, awaits
  M:N green threads  goroutines, Loom, fibers
  Actors             message-passing, no shared memory
  Data-parallel      fork/join, SIMD, map/reduce

AMDAHL
  Speedup(N) = 1 / ((1-p) + p/N)
  90% parallel → cap at 10× no matter the core count

PATTERNS THAT SCALE
  Worker pool, bounded
  Pipeline / fan-out fan-in
  Single-writer principle
  Sharded state
  Immutable + functional
  CSP / channels

TUNING WORKERS
  CPU bound        workers ≈ cores
  Blocking I/O     workers ≈ cores × (1 + wait/compute)
  Async loop       1 loop per core
  DB pool          start small, measure under load

PITFALLS
  Threads on a GIL language for CPU work
  Lock around the whole thing
  Long hold + I/O inside critical section
  Unbounded spawning
  Blocking inside an event loop
  False sharing on hot counters
  Sleep as a race-condition "fix"
  Optimizing concurrency before correctness

RULE   Choose the model that matches the workload. Avoid sharing
       before you synchronize sharing. Measure or you're guessing.
```

---

## 14. Resources

### Books
- *Java Concurrency in Practice* — Brian Goetz et al. The classic; concepts apply far beyond Java.
- *The Art of Multiprocessor Programming* — Herlihy & Shavit. Theory + practice.
- *Concurrency in Go* — Katherine Cox-Buday.
- *Programming Rust* — Blandy/Orendorff (good chapters on ownership-based concurrency).
- *Designing Data-Intensive Applications* — Martin Kleppmann. Distributed angle.

### Articles
- "Concurrency is not Parallelism" — Rob Pike: <https://go.dev/blog/waza-talk>
- "The C10K problem" — Dan Kegel (historical but essential).
- "What every programmer should know about memory" — Ulrich Drepper (long, worth it).
- "Mechanical Sympathy" — Martin Thompson essays on LMAX disruptor.
- "Project Loom" — Ron Pressler / OpenJDK.

### Videos
- *Concurrency is not Parallelism* — Rob Pike (Heroku Waza 2012).
- *Project Loom* — Ron Pressler talks.
- *Mechanical Sympathy* — Martin Thompson.
- ByteByteGo — "Concurrency vs Parallelism Explained."

### Documentation
- **Go memory model** — <https://go.dev/ref/mem>
- **Java JSR-133 / JMM** — primer in any modern JCIP reprint.
- **C++ memory model** — cppreference.com `std::memory_order`.
- **Rust Async Book** — <https://rust-lang.github.io/async-book/>

### Tools
- **ThreadSanitizer (TSan)** — Clang/GCC race detector. Use it.
- **Helgrind / DRD (Valgrind)** — race detection.
- **Java Flight Recorder** — thread states, lock contention.
- **`go run -race`** — Go race detector.
- **async-profiler** — lock contention on the JVM.
- **`perf c2c`** — cache line contention on Linux.

### Adjacent reading
- [Threading, Async I/O, Event Loops →](./threading-async.md)
- [Connection Pooling & Keep-Alive →](./connection-pooling.md)
- [Profiling & Benchmarking →](./profiling.md)
- [Tail Latency & p99 →](./tail-latency.md)
- [Backpressure →](../10-scalability/backpressure.md)
- [Consensus Algorithms →](../08-distributed-systems/consensus.md)
- [Stateful vs Stateless Services →](../01-foundations/stateful-vs-stateless.md)
- [CRDTs →](../08-distributed-systems/crdts.md)

---

*Previous:* [← Profiling & Benchmarking](./profiling.md)  |  *Next:* [Threading, Async I/O, Event Loops →](./threading-async.md)
