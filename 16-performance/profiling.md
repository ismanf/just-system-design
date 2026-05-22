# Profiling & Benchmarking

> **TL;DR** — **Profiling** tells you where your program actually spends its time, memory, and I/O. **Benchmarking** measures how fast it is under defined conditions. Both are how you replace folklore ("the ORM is slow") with evidence ("48% of CPU is in JSON serialization, here's the flame graph"). Most performance work fails not because the tools are bad but because people **optimize without measuring first**. The professional discipline is short: profile in production-realistic conditions, identify the dominant cost, change one thing, measure again. Tools have converged on a small, sharp set — **flame graphs**, **`perf`**, **eBPF**, **language-native pprof / async-profiler / py-spy / pprof.js**, **`hyperfine`** and **JMH** for microbenchmarks. **Treat performance like a science, not a vibe.** If you can't show the before/after number, you didn't fix anything.

---

## 1. The big picture

Performance work is a measurement loop:

```
   ┌──────────────────────────────────────────────┐
   │ 1. Define the metric that matters            │
   │    (p99 latency, throughput, $/req, etc.)    │
   └──────────────────────────────────────────────┘
                       │
                       ▼
   ┌──────────────────────────────────────────────┐
   │ 2. Measure baseline under realistic load     │
   └──────────────────────────────────────────────┘
                       │
                       ▼
   ┌──────────────────────────────────────────────┐
   │ 3. Profile — find the dominant cost          │
   └──────────────────────────────────────────────┘
                       │
                       ▼
   ┌──────────────────────────────────────────────┐
   │ 4. Form a hypothesis. Change ONE thing.      │
   └──────────────────────────────────────────────┘
                       │
                       ▼
   ┌──────────────────────────────────────────────┐
   │ 5. Re-measure. Did the metric move?          │
   └──────────────────────────────────────────────┘
                       │
                       └──► repeat until done
```

Skipping any step is how you get folklore. The most common skip is step 3 — people change code based on intuition, and intuition is wrong about performance roughly half the time.

---

## 2. Donald Knuth's most misquoted line

> "Premature optimization is the root of all evil."

The full quote is:

> "Programmers waste enormous amounts of time thinking about, or worrying about, the speed of noncritical parts of their programs, and these attempts at efficiency actually have a strong negative impact when debugging and maintenance are considered. We should forget about small efficiencies, say about 97% of the time: **premature optimization is the root of all evil. Yet we should not pass up our opportunities in that critical 3%.**"

The cheap interpretation ("don't optimize") is wrong. The real lesson: **find the 3% that matters and optimize that ruthlessly; leave the 97% alone**. Profiling is how you tell which 3% you're in.

---

## 3. Latency vs throughput vs efficiency

These are different goals and they trade off:

| Goal | What you optimize | Typical metric |
|---|---|---|
| **Latency** | Time per individual operation | p50, p95, p99, p99.9 |
| **Throughput** | Operations per second under saturation | RPS, MB/s, msg/s |
| **Efficiency** | Resources per operation | CPU%/req, $/req, joules/op |

You can't maximize all three. Batching trades latency for throughput. Compression trades CPU for bandwidth. Caching trades memory for latency. The first question of any perf work: **which axis are we on?**

Foundational reading: [Throughput vs Latency vs Bandwidth →](../01-foundations/throughput-latency-bandwidth.md), [Tail Latency & p99 →](./tail-latency.md).

---

## 4. The USE method and the four golden signals

When you don't know where to start, scan the system top-down with one of these checklists.

### USE — for resources (Brendan Gregg)

For every resource (CPU, memory, disks, network, GC, file descriptors, threads, sockets), measure:

- **U**tilization — % of time busy
- **S**aturation — queue depth or wait time
- **E**rrors — failed events

### Four Golden Signals — for services (Google SRE)

For every service:

- **Latency** — distribution of request times (split successes from failures)
- **Traffic** — RPS / messages / bytes
- **Errors** — error rate, by class
- **Saturation** — how full the system is

These two lenses catch the vast majority of performance pathologies before you even open a profiler.

---

## 5. The profiling toolbox

### CPU profiling

What it does: samples the program counter at a fixed rate (usually 99 Hz or 999 Hz) and records call stacks. The result tells you which code paths consumed CPU.

- **`perf`** (Linux) — kernel-level, accurate, the canonical tool. `perf record -F 99 -g`, then `perf script` → flame graph.
- **`eBPF` + `bpftrace`** — modern, low-overhead, kernel-and-userland.
- **Go**: `pprof` (built-in). Trivial: import `_ "net/http/pprof"`, `go tool pprof http://host:6060/debug/pprof/profile`.
- **Java**: **async-profiler** (no safepoint bias — better than VisualVM's CPU sampler), **JFR** (Java Flight Recorder).
- **Python**: **py-spy** (no code changes needed, attaches to a running PID), **Pyinstrument**, **cProfile**.
- **Node**: `node --prof`, **0x** for flame graphs, **clinic.js**.
- **Ruby**: **rbspy**, **stackprof**.
- **Rust / C / C++**: `perf`, **callgrind**, **Coz** for causal profiling.
- **.NET**: **PerfView**, **dotnet-trace**.

### Memory profiling

- **Heap profiles**: Go's `pprof` heap, Java's heap dumps + Eclipse MAT, Python's **memray**, Node's `--inspect` + Chrome DevTools, Ruby's **memory_profiler**.
- **Allocation profiling** tells you *where* memory is allocated; **retention** tells you *what's keeping it alive*. Both matter.
- **`/proc/$pid/status`**, **`smem`**, **`pmap`** — quick OS-level looks.

### I/O and syscalls

- **`strace`** / **`dtrace`** / **`bpftrace`** — what syscalls is the process making, how often, how slow?
- **`iotop`**, **`iostat`**, **`pidstat -d`** — disk I/O.
- **`tcpdump`**, **`ss`**, **`tshark`** — network.
- **`perf trace`** — modern strace replacement.

### GC pressure

- **Java**: GC logs (`-Xlog:gc*`), GCViewer, JFR.
- **Go**: GODEBUG=gctrace=1, pprof allocs.
- **.NET**: `dotnet-counters`, ETW events.
- **Node**: `--trace-gc`.

Excessive GC almost always shows up as wasted CPU and erratic p99 latency. Always check GC before microoptimizing application code.

### Distributed tracing

For multi-service systems, single-process profiling tells you only part of the story. **OpenTelemetry** spans across services so you can see "this request took 800ms — 600ms was spent in service B." See [Distributed Tracing →](../13-observability/tracing.md).

### Continuous profiling

The new frontier. Tools like **Pyroscope**, **Polar Signals (Parca)**, **Datadog Continuous Profiler**, **Google Cloud Profiler**, **AWS CodeGuru Profiler** sample every running process all the time, in production, with under 1% overhead. You no longer need to reproduce a performance issue — you just look at last Tuesday's flame graph.

---

## 6. Flame graphs — read them like this

A flame graph (Brendan Gregg, 2011) visualizes a CPU profile:

```
       ┌────────────────────────────────────────────┐
       │            main                            │  ← width = total CPU
       └────────────────────────────────────────────┘
       │     handle_request    │   compact_gc    │
       │ parse │ db_query │... │                 │
       │       │ ssl │tcp│     │                 │
```

- **X axis**: NOT time — it's alphabetical or call-merged. Wider = more CPU.
- **Y axis**: call stack depth. Deeper = called by what's below.
- **Color**: usually meaningless (random). Some tools color by language or by type (yellow = JIT, green = native, red = JVM frames, etc.).

How to read one: **scan the top of each tower**. Those are the leaf functions where CPU was actually spent. A wide leaf is a hotspot.

Variants worth knowing:
- **Differential flame graphs** — overlay two profiles to see what changed (red = got worse, blue = got better). Brilliant for "did my change help?"
- **Off-CPU flame graphs** — where the process was *waiting* (locks, I/O), not running.
- **Icicle graphs** — same data, upside-down. Same information; some people prefer it.

---

## 7. Benchmarking — measuring with rigor

A "benchmark" can mean anything from a one-shot timing to a controlled experiment. Get specific.

### Microbenchmarks

Time a small piece of code (a function, a loop). They're treacherous — JIT warmup, branch predictor, CPU caches, dead-code elimination, the compiler optimizing your benchmark away.

Use a framework that handles the traps:

- **JMH** (Java) — the gold standard. Handles warmup, deoptimization, dead-code elimination via `Blackhole`.
- **`testing.B`** (Go) — `go test -bench=. -benchmem -count=10`.
- **`criterion.rs`** (Rust) — statistical comparisons, regression detection.
- **`pytest-benchmark`** / **timeit** (Python) — for what's possible in Python.
- **`hyperfine`** — for command-line tools.
- **Benchmark.js** (JavaScript) — handles statistical rigor.

```go
// Go example
func BenchmarkParseJSON(b *testing.B) {
    data := loadFixture()
    b.ResetTimer()
    b.ReportAllocs()
    for i := 0; i < b.N; i++ {
        var out Payload
        if err := json.Unmarshal(data, &out); err != nil {
            b.Fatal(err)
        }
    }
}
```

Run with multiple `-count` and use `benchstat` to compare distributions, not point estimates.

### Macrobenchmarks (load tests)

Simulate realistic traffic against the whole system:

- **k6** — JavaScript-scripted, modern, scales well.
- **Locust** — Python-scripted, good UX.
- **wrk** / **wrk2** — fast HTTP load generators. `wrk2` corrects for **coordinated omission** (see below).
- **JMeter** — old, comprehensive, GUI-heavy.
- **vegeta** — Go, constant-rate, great for steady traffic.
- **Gatling** — Scala DSL, strong reporting.
- **fortio** — Istio team's load tool, latency histograms baked in.

Aim for **realistic load shapes**: think-time, request mix, payload size distribution, header sizes. Synthetic traffic that's too clean misses the real failure modes.

### Coordinated omission — the trap

A naive load tool sending 100 RPS does this: send a request, wait for response, send the next. If a response takes 5 seconds, the tool **sent zero requests for 5 seconds** and didn't measure that gap. The reported p99 is wildly optimistic.

`wrk2`, `fortio`, and `vegeta -rate=...` send requests on schedule regardless of response, then record how long each took to be served — which is what you actually want. Without this, p99 is a lie. Read Gil Tene's "How NOT to Measure Latency" if you do any latency work at all.

### Benchmark hygiene

- **Warm up.** First few runs include JIT compilation, page faults, cold caches.
- **Repeat.** A single run is noise. 5–20 iterations; compare distributions.
- **Pin the environment.** Same machine, same kernel, same load, same NUMA, same governor (`cpupower frequency-set -g performance`).
- **Disable noisy neighbors.** No background builds, no IDE indexing.
- **Compare statistically.** `benchstat`, `criterion`, t-tests. "5% faster" with 8% variance is noise.
- **Match real conditions.** Microbenchmarks lie about cache effects, data sizes, branch mispredictions you'll see in production.
- **Track regressions.** Run benchmarks in CI, compare to the previous main.

---

## 8. The order of operations — what to look at first

A pragmatic order of investigation, not a religion:

1. **Define what's slow.** "The checkout page" → "p99 of POST /checkout is 4.2s, target 800ms."
2. **Look at the dashboards.** Golden signals, USE checklist. Often the answer is "DB CPU is at 95%."
3. **Trace a slow request.** Distributed tracing shows which span dominated.
4. **Profile the dominant component.** Flame graph the hot service.
5. **Read the code at the hot frames.** Often you'll spot it (a sync log in a hot path, a JSON parse of a 10MB blob, a missing index).
6. **Form a hypothesis.** "If I batch these N+1 queries, latency drops to X."
7. **Make one change.** Don't bundle ten "improvements" — you can't tell which one worked.
8. **Re-measure.** If you can't show the number moved, you didn't fix anything.

The vast majority of real performance problems are:
- **Missing or wrong index** ([Indexing →](../04-databases/indexing.md))
- **N+1 queries** ([N+1 Query Problem →](./n-plus-one.md))
- **Synchronous I/O in a hot path** ([Threading, Async I/O →](./threading-async.md))
- **No batching / no connection pooling** ([Batching →](./batching-debouncing.md), [Connection Pooling →](./connection-pooling.md))
- **Bad serialization** (huge JSON, repeated parsing) ([Serialization →](./serialization.md))
- **GC pressure** (allocation in a hot loop)
- **Cache miss** (where you assumed a hit)
- **Lock contention** (one mutex everyone fights for)
- **Misconfigured pool sizes** (DB connections, threads)
- **Sync logs to disk in the hot path**

You will rarely "discover" a CPU-bound hot function that needs a clever rewrite. The wins are almost always architectural.

---

## 9. Worked example — the right way

Suppose your `GET /orders/:id` is 1.2s at p99. Target 200ms.

1. **Dashboards**: HTTP latency dashboard. p50=180ms (fine), p99=1.2s (bad). DB p99 = 80ms (looks fine).
2. **Tracing**: pull 20 slow traces. Pattern: each takes ~30 DB calls. Aha — N+1.
3. **Code read**: `order.items` lazily loads each item separately.
4. **Hypothesis**: prefetch items in one query → 1 DB call instead of 30.
5. **Microbenchmark** the change in isolation: 12ms vs 280ms per request. Promising.
6. **Stage rollout**: deploy to canary, watch p99.
7. **Re-measure**: p99 = 210ms. Close to target. Diminishing returns from here would require bigger changes (caching).

What you didn't do: rewrite JSON parsing, switch to Rust, "use threads," replace the ORM. All would have been wrong answers without measurement.

---

## 10. Performance regressions in CI

Catching regressions before they ship:

- **Microbenchmark suite** runs on every PR. Compare against `main` using `benchstat`.
- **k6 / Locust** smoke test against staging. Reject if p99 regresses >X%.
- **Load test before major releases.** Don't find out under real traffic.
- **Continuous profiling** in production. Diff this week's flame graph against last week's.
- **Cost dashboards** — efficiency regressions show up as $$/req creeping up before users notice.

This is where the discipline pays off long-term: the team that catches a 5% regression every week is the team that doesn't have a "rewrite the slow service" project next year.

---

## 11. Common Mistakes / Anti-Patterns

- **Optimizing without profiling.** "I bet it's the JSON parsing" — flame graph shows 92% in DB driver. Hours wasted.
- **Trusting the wrong percentile.** Optimizing p50 while p99 burns. Different users feel different percentiles.
- **Single-run benchmarks.** Noise level often exceeds the signal. Always multiple runs + statistics.
- **Coordinated omission.** Load tool waits for responses; "p99=200ms" is fantasy. Use `wrk2`, `fortio`, or `vegeta -rate`.
- **Microbenchmarking in dev mode.** JIT off, debug symbols, `-O0`. Numbers are useless.
- **Benchmarks against `localhost` then deploying to multi-region.** Network round trip dominates everything else in distributed systems.
- **Profiler running but observer effect crushes the workload.** Some profilers add 30% overhead. Pick low-overhead ones (eBPF, async-profiler) for prod.
- **Memory profiling that confuses allocation rate with retained heap.** Different problems, different fixes.
- **Ignoring GC.** "It's the language" — no, it's the allocation in your hot loop.
- **One huge PR with ten "improvements."** Which one helped? Which one regressed? You'll never know.
- **No baseline.** "After my change, p99 = 250ms." Was that better or worse?
- **Synthetic load that doesn't match real traffic shape.** Real traffic has long-tail keys, bursty arrivals, cache misses. Clean uniform load gives clean misleading numbers.
- **Optimizing the wrong layer.** Hotspot in the SQL → spending a week on Go map allocations.
- **Premature parallelization.** Adding goroutines/threads to a problem that's bottlenecked on the DB. Now you have 10× more queued requests waiting on the same DB.
- **Forgetting to remove debug flags before perf-testing.** `pprof` enabled, `verbose: true`, debug logging — all skew results.

---

## 12. Cheat Card

```
PURPOSE   Replace performance folklore with evidence: where the
          time actually goes, and what changes actually moved it.

LOOP
  define metric  →  baseline  →  profile  →  one change  →  re-measure

THREE AXES
  Latency      p50 / p95 / p99 / p99.9
  Throughput   ops/sec, MB/sec under saturation
  Efficiency   $/req, CPU%/req, joules/op
  (You can't max all three.)

TOP-DOWN SCANS
  USE   util / saturation / errors  per resource
  Golden 4   latency / traffic / errors / saturation per service

PROFILERS
  CPU   perf · pprof · async-profiler · py-spy · 0x · rbspy
  Heap  pprof · memray · heap dumps · MAT · clinic.js
  I/O   strace · bpftrace · iostat · tcpdump · perf trace
  Cont. Pyroscope · Parca · Datadog · CodeGuru

VIEWS
  Flame graph        wide leaf = hotspot
  Differential       red = worse, blue = better
  Off-CPU            where you waited (locks, IO)

BENCHMARKING TRAPS
  Coordinated omission → use wrk2 / fortio / vegeta -rate
  No warmup / JIT cold runs
  Single-run noise
  Synthetic load too clean
  Compare distributions (benchstat) not point estimates

THE 80%
  Missing index           N+1 queries
  Sync I/O in hot path    No batching / pooling
  Huge JSON / bad codec   GC pressure
  Lock contention         Cache misses
  Wrong pool sizes        Logging on the hot path

RULE   No before/after number = no fix. Profile production
       conditions, change one thing, prove the diff.
```

---

## 13. Resources

### Books
- *Systems Performance* (2nd ed.) — Brendan Gregg. The definitive reference.
- *BPF Performance Tools* — Brendan Gregg.
- *Java Performance: The Definitive Guide* — Scott Oaks.
- *High Performance Browser Networking* — Ilya Grigorik.
- *Database Internals* — Alex Petrov. Where to look in storage layers.

### Documentation
- **Linux `perf`** — <https://perf.wiki.kernel.org>
- **Brendan Gregg's site** — <https://www.brendangregg.com> (flame graphs, USE, off-CPU, every relevant topic)
- **`pprof`** — <https://github.com/google/pprof>
- **OpenTelemetry** — <https://opentelemetry.io>
- **Pyroscope / Parca** — continuous profiling.

### Articles
- "How NOT to Measure Latency" — Gil Tene (must-read on coordinated omission): <https://www.youtube.com/watch?v=lJ8ydIuPFeU>
- "Flame Graphs" — Brendan Gregg: <https://www.brendangregg.com/flamegraphs.html>
- "USE Method" — Brendan Gregg: <https://www.brendangregg.com/usemethod.html>
- "Profiling Go Programs" — Russ Cox / Go Blog.
- "Always be measuring" — countless engineering blogs; pick one from Stripe, Cloudflare, or Discord.

### Videos
- *Systems Performance* — Brendan Gregg talks (USENIX, SREcon).
- *Continuous Profiling in Production* — Polar Signals talks.
- *Measuring Latency* — Gil Tene.
- ByteByteGo — "Performance Profiling Explained."

### Tools
- **perf, bpftrace, eBPF** — kernel-level Linux observability.
- **pprof, async-profiler, py-spy, rbspy, 0x, clinic.js** — language-native CPU/heap profilers.
- **memray, heaptrack** — memory profilers.
- **k6, wrk2, fortio, vegeta, Locust, Gatling** — load generators.
- **hyperfine, JMH, benchstat, criterion.rs, pytest-benchmark** — microbenchmark frameworks.
- **Pyroscope, Parca, Datadog, CodeGuru, Google Cloud Profiler** — continuous profilers.
- **Honeycomb, Lightstep, Datadog APM, Jaeger** — distributed tracing.

### Adjacent reading
- [Concurrency vs Parallelism →](./concurrency-parallelism.md)
- [Threading, Async I/O, Event Loops →](./threading-async.md)
- [Connection Pooling & Keep-Alive →](./connection-pooling.md)
- [Compression →](./compression.md)
- [Serialization Formats →](./serialization.md)
- [N+1 Query Problem →](./n-plus-one.md)
- [Batching & Debouncing →](./batching-debouncing.md)
- [Tail Latency & p99 →](./tail-latency.md)
- [Distributed Tracing →](../13-observability/tracing.md)
- [Database Indexing →](../04-databases/indexing.md)
- [Metrics →](../13-observability/metrics.md)

---

*Previous:* [← Multi-Tenancy](../15-deployment/multi-tenancy.md)  |  *Next:* [Concurrency vs Parallelism →](./concurrency-parallelism.md)
