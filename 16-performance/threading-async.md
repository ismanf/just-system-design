# Threading, Async I/O, Event Loops

> **TL;DR** — There are three execution models you'll keep running into: **OS threads** (real kernel threads, scheduled by the OS), **async / event loops** (single-threaded cooperative multitasking — Node.js, Python asyncio), and **green threads / fibers / coroutines** (M:N user-space scheduling — Go goroutines, Java virtual threads, Erlang processes). They solve the same fundamental problem — *how do I serve many concurrent operations without dedicating one OS thread per operation?* — with very different trade-offs. The honest take: **async I/O dominates network-heavy services**, **threads still rule for true CPU work**, and **M:N runtimes** (goroutines, Loom) give you the developer ergonomics of threads with the scale of async. The pitfalls are predictable: **blocking calls in an event loop**, **lock contention in threads**, **starvation in cooperative schedulers**, and **forgetting that the kernel still has to do the actual I/O**.

---

## 1. The big picture

```
OS Threads                Event loop / async         M:N green threads
─────────────             ───────────────────        ────────────────────
[Thread] [Thread] ...     [single thread w/ queue]    [N goroutines]
   ▼        ▼                ▼                            ▼  ▼  ▼
 kernel  kernel              kernel epoll / kqueue        M OS threads
                                                              ▼  ▼
                                                            kernel
```

All three end up at the kernel for I/O. The difference is *what's between your code and the kernel*:

- **Threads** — each unit of work owns an OS thread (a stack, scheduler entry, kernel resources). Blocking is fine; the OS reschedules.
- **Event loop** — one OS thread (per loop) services many in-flight operations by **never blocking**. When work waits, it yields back to the loop, which picks up something else ready to run.
- **M:N** — your code looks like blocking code, but the runtime intercepts blocking calls, parks the user-space task, and gives the OS thread to another task.

You can run all three on the same machine. Real systems often mix them.

---

## 2. Where each model wins and loses

| | Threads | Event loop / async | M:N green threads |
|---|---|---|---|
| Mental model | "Do work, the OS handles waiting" | "Yield on every I/O" | "Do work, runtime handles waiting" |
| Per-task overhead | 1–8 MB stack, kernel slot | A few KB closure / task | 2–8 KB goroutine stack |
| Scale (1 box) | Thousands | 100K+ connections | Millions |
| Blocking safe? | Yes | **No** — kills the loop | Yes — runtime parks the task |
| Parallel CPU? | Yes | One core per loop | Yes |
| Function coloring? | None | `async` viral | None |
| Stack traces | Native | Often async-broken | Native or close |
| GC pressure | Low | Medium-High | Medium |
| Best for | CPU-heavy, simple flows | I/O fan-out, websockets, APIs | Big API services, message brokers |
| Examples | Java threads, pthreads | Node, Python asyncio, nginx | Go, Erlang/Elixir, Java virtual threads |

There's no universal winner. There's the right tool for the workload.

---

## 3. The C10K problem — why event loops exist

In the late 90s, web servers used "one OS thread per connection." That works until you hit roughly **10,000 concurrent connections** (the famous *C10K* problem named by Dan Kegel). Beyond that:

- The kernel scheduler thrashes.
- Memory per thread dominates RAM.
- Context-switch overhead exceeds useful work.

Two solutions emerged:

1. **Asynchronous I/O syscalls + an event loop** — `epoll` (Linux), `kqueue` (BSD/macOS), `IOCP` (Windows). One thread can wait on tens of thousands of sockets with a single syscall, react when any are ready.
2. **Lightweight user-space tasks** — Erlang did this from day one; Go formalized it for the mainstream.

Modern Linux additions:
- **`io_uring`** (2019+) — submission and completion ring buffers shared with the kernel. Lower syscall overhead than `epoll` for high-throughput workloads. Used by ScyllaDB, modern Tokio, Node.js (experimentally), and others.

This is the basis of every high-performance network service today.

---

## 4. OS threads — the baseline

A thread is a kernel-scheduled execution context inside a process. They share memory, file descriptors, and signal handlers.

What threading is good at:

- **CPU-bound parallel work** that benefits from multiple cores.
- **Simple straight-line code** where blocking I/O is part of the natural flow (database calls, sync HTTP).
- **Legacy and library compatibility** — most languages have decades of thread-safe code.

What threading struggles with:

- **High connection counts.** 50,000 threads cost 50–400 GB of stack RAM. The OS scheduler buckles.
- **Lock contention.** Shared mutable state across many threads requires synchronization (see [Concurrency vs Parallelism →](./concurrency-parallelism.md)).
- **Per-thread overhead.** Default Java thread stack: 512 KB–1 MB. Default Linux pthread: 8 MB. Tune carefully.

### Thread pools

A bounded set of threads pulling tasks from a queue. The default shape for almost anything that uses threads in modern code.

```java
// Java
ExecutorService pool = Executors.newFixedThreadPool(32);
for (Task t : tasks) {
    pool.submit(() -> handle(t));
}
```

Tuning a pool:
- CPU-bound: `workers ≈ cores`.
- Blocking I/O: `workers ≈ cores × (1 + wait/compute)`. Often dozens to hundreds.
- Bounded queue, with a rejection policy — never unbounded.

### Virtual threads (Java 21+)

Project Loom changed the game for Java. **Virtual threads** are M:N lightweight tasks scheduled by the JVM. You write blocking code as you always have; the JVM parks the virtual thread when it blocks, lets the OS thread serve someone else, and resumes when ready. Goroutine-grade ergonomics with the entire JVM ecosystem.

```java
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    IntStream.range(0, 100_000).forEach(i ->
        executor.submit(() -> handle(i))
    );
} // 100k virtual threads, ~few MB total
```

For new Java services in 2026, virtual threads are the default choice for I/O-heavy workloads.

---

## 5. Event loops — how they actually work

Single thread, infinite loop:

```
loop forever:
    ready = epoll_wait(fd_list, timeout)   # blocks once, returns ready FDs
    for fd in ready:
        run_callback_for(fd)               # synchronous work
    run_due_timers()
    run_microtasks()
```

`epoll_wait` (or `kqueue`, `io_uring`) hands the kernel a list of file descriptors and gets back the ones ready for I/O. The loop wakes, services them, and goes back to sleep.

This is wildly efficient: one syscall per batch of ready events, no thread overhead per connection, no scheduler thrash. nginx famously handles tens of thousands of connections per worker process this way.

### Async / await

`async`/`await` is syntactic sugar for "yield until this Promise/Future completes." The compiler transforms an `async` function into a state machine; each `await` is a yield point.

```python
# Python asyncio
async def handle(req):
    user = await db.user(req.id)          # yield to loop while DB responds
    orders = await db.orders(user.id)     # yield again
    return {"user": user, "orders": orders}
```

The function looks linear. Under the hood, the loop runs other ready tasks while this one is parked on `await`.

### The blocking-call problem

The single thread is doing *all* the work. If anything on that thread takes a long time — synchronously — every other task waits.

Examples of accidental blocking:

- `fs.readFileSync` in Node.js.
- `requests.get(url)` (blocking HTTP) inside `asyncio` code.
- A CPU-heavy JSON parse / regex / crypto op on a hot loop.
- Calling a synchronous third-party library from async code.

Symptoms: p99 latency spikes; "the server feels frozen for a second." The fix is one of:

1. **Make the call async** — use the async variant (`aiohttp`, `fs.promises.readFile`).
2. **Offload to a thread pool** — Node.js worker threads, Python `loop.run_in_executor`, .NET `Task.Run`.
3. **Offload to another process / service** — heavy CPU stays out of the loop entirely.

In Node, the rule of thumb: **a single iteration of the loop should never exceed ~10 ms** for latency-sensitive services. Profile with `clinic.js` doctor or with the event-loop-lag metric.

### Event loop saturation metrics

- **Node**: `perf_hooks.monitorEventLoopDelay()` — measures lag distribution.
- **Python**: `asyncio` slow-callback debug mode, or instrumentation.
- **Browser**: long task API; >50 ms is a long task.

Treat event-loop lag like a top-level SLI. When it climbs, your async server is dying.

---

## 6. M:N runtimes — the best of both

Go, Erlang/Elixir, and now Java virtual threads (Loom) hide the threading model from you.

```go
// Go — 100k goroutines, blocking-style calls, no async coloring
for _, req := range requests {
    go func(r Req) {
        resp, _ := http.Get(r.URL)   // blocks the goroutine, not the OS thread
        process(resp)
    }(req)
}
```

Behind the scenes, the runtime:
- Multiplexes N goroutines onto M OS threads (M ≈ `GOMAXPROCS`).
- Intercepts blocking syscalls — the runtime knows when to park a goroutine and reuse the OS thread.
- Schedules cooperatively (preemption added in newer Go versions to prevent goroutine hogging).
- Grows goroutine stacks as needed (start tiny, ~2 KB).

The win: you write **straight-line code that scales to millions of concurrent tasks** without async coloring.

The catches:

- **Blocking C calls** still pin an OS thread for the duration. A goroutine calling into CGo doesn't yield.
- **GC matters**. High allocation rates create pause-time issues.
- **Goroutine leaks** are easy: spawn one per request, forget to bound it, blow up under load.
- **Channel deadlocks** — two goroutines waiting on each other's channels.

Erlang/Elixir go further: each "process" is fully isolated (no shared memory), and the language enforces it. Process crashes are local; supervisors restart them. The combination of isolation + supervision is why Erlang systems run for years at 99.999% uptime.

---

## 7. Linux's I/O primitives — the layer underneath

All async/event-loop systems on Linux ultimately use one of:

| Primitive | Era | Notes |
|---|---|---|
| `select` / `poll` | 1980s/1990s | O(n) per call; limited FD counts. Avoid. |
| `epoll` | 2002+ | O(1) per event, edge- or level-triggered. The workhorse since Node.js, nginx, libuv. |
| Async POSIX AIO | 2003 | Limited, never popular. |
| `io_uring` | 2019+ | Ring buffers shared with kernel; minimal syscalls; supports both networking and disk I/O. The future. |

For most application engineers, the runtime hides this. For low-level perf work or building runtimes, knowing the difference matters. Modern Tokio (Rust), recent libuv versions, and ScyllaDB lean on `io_uring` for top-tier throughput.

---

## 8. Choosing the model

A practical decision tree:

```
Is the workload mostly CPU?
├── Yes → threads with a fixed pool sized to cores
│         (or M:N runtime is fine — same outcome)
└── No (I/O-heavy)
    │
    Are we writing in Go / Erlang / Java 21+?
    ├── Yes → goroutines / processes / virtual threads
    │         (write blocking-style code, runtime handles it)
    └── No
        │
        Does the language/runtime have async (Node, Python, .NET, Rust)?
        ├── Yes → async + event loop, watch for accidental blocking
        └── No  → threads with a generously sized pool
```

A second consideration: **how does your team think?** If everyone is fluent in async/await, that's a real productivity gain. If your team writes blocking-style code naturally, M:N (Go, Loom) is a much smaller cognitive jump.

---

## 9. The cost of context switches

A context switch costs roughly:

| Switch type | Cost |
|---|---|
| Function call | ~1 ns |
| Goroutine / virtual thread switch | ~200 ns |
| `await` yield in async runtime | ~100 ns – 1 µs |
| OS thread context switch (same core) | ~1 µs |
| OS thread context switch (cross-core, cache cold) | ~5–10 µs |
| Process context switch | ~10 µs |

The numbers shift with workload, cache state, and kernel version. The hierarchy stays roughly stable: function call << M:N switch < async yield < OS thread switch.

If you have 100K concurrent connections each switching frequently, those costs add up. Async and M:N runtimes amortize switching cost dramatically — that's their whole point.

---

## 10. Common deadlocks and starvations

### Deadlock from lock ordering

```
Thread A: lock(L1); lock(L2);   # holds L1, waits for L2
Thread B: lock(L2); lock(L1);   # holds L2, waits for L1
```

Fix: define a global lock order and never violate it. Or use a higher-level abstraction (transactions, actors).

### Async-await deadlock (.NET classic)

Calling `.Result` or `.Wait()` on a Task from the UI thread while the Task tries to resume on that thread. Fix: `ConfigureAwait(false)`, or stay fully async.

### Goroutine leak via unbuffered channel

A sender writes to a channel; no receiver runs. Goroutine blocks forever. Run `pprof goroutine` and you'll see thousands of them stuck.

### Event-loop saturation

A CPU-heavy task blocks the loop, all callbacks queue. p99 explodes.

### Cooperative-scheduler starvation

In purely cooperative runtimes, a task that never yields can starve everyone. Modern Go and modern asyncio add preemption to mitigate.

### Thread pool exhaustion

All workers blocked on the same downstream (slow DB, hung HTTP). New work queues forever. Fix: timeouts on all external calls, separate pools for unrelated downstreams (bulkhead pattern — see [Bulkhead Pattern →](../11-reliability/bulkhead.md)).

---

## 11. Backpressure, timeouts, cancellation

Concurrency without backpressure is just queuing. If producers run faster than consumers, queues grow until you OOM.

The full triad:

- **Backpressure** — slow down producers when consumers fall behind. Push back through bounded channels/queues. See [Backpressure →](../10-scalability/backpressure.md).
- **Timeouts** — every external call. No exceptions. A request hanging forever blocks a worker forever.
- **Cancellation** — propagate "give up" through the call graph. Go's `context.Context`, .NET's `CancellationToken`, Java's `Thread.interrupt()`, asyncio's `cancel()`.

Cancellation is one of the most-skipped basics. Without it:
- Client gave up after 5s; your server still computes for 5 minutes.
- One slow user pins a goroutine forever; their friends get rate-limited.
- Shutdown blocks because pending work never knew to stop.

Every async / concurrent codebase should plumb a cancellation primitive through every call. It's not optional.

---

## 12. Common Mistakes / Anti-Patterns

- **Blocking call inside an event loop.** Disastrous. Profile event-loop lag.
- **`await` in a tight CPU loop expecting parallelism.** Async yields *I/O wait*, not CPU. You're still single-threaded.
- **Spawning unbounded goroutines/tasks/threads.** Spike in load → OOM.
- **Sync libraries called from async code.** "It works" until concurrency rises and the pool saturates.
- **No timeout on external calls.** First slow downstream takes the service down.
- **No cancellation plumbing.** Work continues after the caller left.
- **Thread pool too big.** More threads ≠ more throughput; usually the opposite.
- **Mixing async runtimes.** Two different event loops in one process, blocking on each other.
- **Calling `.Result` / `.Wait()` on async from sync (.NET) without `ConfigureAwait(false)`.** Deadlock.
- **`GOMAXPROCS` left at container's apparent CPU count (incorrect on cgroups < 1.21+).** Set explicitly in containers.
- **Sleep as a synchronization tool.** "Wait a bit so the goroutine finishes" — works in tests, fails in prod.
- **Goroutines / Tasks created but never awaited or cancelled.** Hidden leaks.
- **Locking on the event loop's only thread.** Single-thread loop means no need for a lock — adding one does nothing useful and slows things.
- **Logging synchronously inside a hot loop.** Disk I/O on the critical path; latency jumps.
- **Treating async/await as a perf optimization rather than a concurrency model.** Adding `async` doesn't make code faster; structuring it to overlap I/O does.

---

## 13. Cheat Card

```
PURPOSE   Serve many concurrent operations without one OS thread
          per operation. Pick the model that fits the workload.

THREE MODELS
  OS threads          CPU-bound, simple flows, library compat
  Event loop / async  10K+ I/O connections, single-thread per loop
  M:N green threads   Best of both: blocking-style code at scale

KERNEL PRIMITIVES (Linux)
  select / poll       O(n), legacy, avoid
  epoll               O(1), workhorse since 2002
  io_uring            Modern, lowest overhead, ring-based

WHEN TO USE WHICH
  CPU-bound parallel    threads pool (cores-sized)
  10K+ network conn     async/event loop or M:N
  Massive concurrency   M:N (Go, Erlang, virtual threads)
  Legacy ecosystem      threads + virtual threads if available

THE ASYNC RULES
  Never block the loop (no sync I/O, no big CPU work)
  Offload blocking work to a thread pool
  Watch event-loop lag as an SLI
  No async coloring shortcuts (don't mix sync/async carelessly)

THE THREAD RULES
  Bounded pools, never unbounded
  Pool size: cores for CPU, cores × (1 + wait/compute) for I/O
  Locks: short hold times, no I/O inside critical sections
  Different downstreams → different pools (bulkhead)

CONTEXT SWITCH COSTS
  function call    ~1 ns
  goroutine        ~200 ns
  async yield      ~1 µs
  OS thread        ~5–10 µs
  process          ~10 µs

ALWAYS
  Timeout every external call
  Plumb cancellation through the call graph
  Bound queues; backpressure on overflow

PITFALLS
  Block in event loop → p99 catastrophe
  Spawn unbounded tasks → OOM
  No timeouts → first slow downstream takes you down
  No cancellation → wasted work, slow shutdown
  Sync .Result/.Wait() in .NET → deadlock
  GOMAXPROCS wrong in containers

RULE   Match the model to the work. Never block what shouldn't
       block. Bound everything. Cancel everything.
```

---

## 14. Resources

### Books
- *Java Concurrency in Practice* — Brian Goetz et al.
- *The Linux Programming Interface* — Michael Kerrisk (deep on threads, signals, I/O).
- *Concurrency in Go* — Katherine Cox-Buday.
- *Programming Erlang* — Joe Armstrong.
- *Working with TCP/IP* — for context on what the kernel does on async I/O.

### Articles
- "The C10K problem" — Dan Kegel: <http://www.kegel.com/c10k.html>
- "What every systems programmer should know about concurrency" — Matt Kline.
- "Asynchronous I/O on Linux: io_uring" — Jens Axboe and follow-ups.
- "Project Loom" — Ron Pressler: <https://openjdk.org/projects/loom/>
- "Why Goroutines aren't lightweight threads" — multiple Go blog posts.
- "Don't block the event loop" — Node.js docs.

### Videos
- *Project Loom* — Ron Pressler talks.
- *Asyncio: We did it wrong* — David Beazley (Python).
- *Go runtime scheduler* — KubeCon / GopherCon talks.
- ByteByteGo — "Async vs Multithreading."

### Documentation
- **Node.js docs** — Don't block the event loop: <https://nodejs.org/en/docs/guides/dont-block-the-event-loop/>
- **Python asyncio** — <https://docs.python.org/3/library/asyncio.html>
- **Go runtime** — <https://go.dev/doc/articles/race_detector>
- **Tokio (Rust)** — <https://tokio.rs/tokio/tutorial>
- **Java virtual threads** — <https://openjdk.org/jeps/444>

### Tools
- **`perf`, `bpftrace`, `eBPF`** — system-level introspection.
- **`async-profiler`** — JVM threads / lock contention.
- **`go run -race`, `go tool pprof`** — Go race detection / profiling.
- **`clinic.js doctor`** — Node event-loop diagnostics.
- **`py-spy`** — sampling profiler for Python.
- **`Tokio Console`** — tasks and resources in Tokio apps.

### Adjacent reading
- [Concurrency vs Parallelism →](./concurrency-parallelism.md)
- [Connection Pooling & Keep-Alive →](./connection-pooling.md)
- [Profiling & Benchmarking →](./profiling.md)
- [Tail Latency & p99 →](./tail-latency.md)
- [Backpressure →](../10-scalability/backpressure.md)
- [Bulkhead Pattern →](../11-reliability/bulkhead.md)
- [Retry, Timeout, and Exponential Backoff →](../11-reliability/retry-timeout-backoff.md)
- [TCP vs UDP →](../02-networking/tcp-vs-udp.md)

---

*Previous:* [← Concurrency vs Parallelism](./concurrency-parallelism.md)  |  *Next:* [Connection Pooling & Keep-Alive →](./connection-pooling.md)
