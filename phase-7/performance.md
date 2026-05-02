---
layout: default
title: "7.3 Performance Engineering"
parent: "Phase 7: Observability & Reliability"
nav_order: 3
---

# Performance Engineering
{: .no_toc }

<details open markdown="block">
  <summary>Table of Contents</summary>
  {: .text-delta }
1. TOC
{:toc}
</details>

Performance engineering is the discipline of measuring, diagnosing, and improving a system's speed, throughput, and resource efficiency under realistic conditions. The sequence matters: **measure first** (with profiling), **load test to validate** (with Gatling or k6), **tune the bottleneck** (GC, threading, I/O), **measure again**. Never tune without profiling — optimization without measurement is guesswork.

---

## Profiling Java Applications

### Why async-profiler

Most JVM profilers use JVMTI's `SuspendThread` or `GetAllStackTraces` — they only sample at **safepoints** (points where the JVM can safely pause all threads). Code between safepoints is invisible. Tight loops are among the most common CPU hotspots, and they often don't contain safepoints — so traditional profilers undercount them.

**async-profiler** avoids this by using:
- `AsyncGetCallTrace` — a non-safepoint-safe JVM API that can walk the stack at any point
- Linux `perf_events` — hardware performance counters for native code and kernel time

Result: accurate CPU, allocation, and lock profiling without safepoint bias.

### CPU Profiling with async-profiler

```bash
# Attach to a running JVM — get PID first
pgrep -f order-service.jar

# CPU flamegraph: 30-second snapshot
./profiler.sh -e cpu -d 30 -f /tmp/cpu-flamegraph.html 12345

# Inside a Docker container
docker exec -u root -it order-service-container bash -c \
    "/profiler/profiler.sh -e cpu -d 30 -f /tmp/cpu.html \$(pgrep java)"
docker cp order-service-container:/tmp/cpu.html ./cpu-flamegraph.html

# Allocation profiling (find what's filling the Eden space)
./profiler.sh -e alloc -d 30 -f /tmp/alloc.html 12345

# Lock contention (find what's blocking threads)
./profiler.sh -e lock -d 30 -f /tmp/lock.html 12345

# Wall-clock profiling (includes threads blocked on I/O — useful for finding slow DB queries)
./profiler.sh -e wall -t -d 30 -f /tmp/wall.html 12345
```

```java
// Programmatic profiling API (embed in load test or benchmark)
AsyncProfiler profiler = AsyncProfiler.getInstance();
profiler.start(Events.CPU, 1_000_000);    // sample every 1ms
// ... run the workload under test ...
profiler.stop();
profiler.dumpFlat(System.out, 20);        // top 20 methods by sample count
```

### Java Flight Recorder (JFR)

JFR is built into the JDK (11+). It continuously records JVM events (GC pauses, JIT compilation, thread states, socket I/O, method timing) with ~1–2% overhead. Unlike snapshots, JFR can record a circular buffer and dump on a trigger — ideal for capturing the state leading up to a production incident.

```bash
# Start a 60-second JFR recording from command line
java \
  -XX:StartFlightRecording=duration=60s,filename=/tmp/app.jfr,settings=profile \
  -jar order-service.jar

# Trigger a dump from a running JVM via jcmd
jcmd 12345 JFR.start duration=60s filename=/tmp/incident.jfr

# Dump continuous buffer on OOM
java \
  -XX:+FlightRecorder \
  -XX:StartFlightRecording=maxage=5m,dumponexit=true,filename=/tmp/oom-dump.jfr \
  -XX:+HeapDumpOnOutOfMemoryError \
  -jar order-service.jar
```

```java
// Spring Boot: expose a JFR recording endpoint for on-demand profiling
@RestController
@RequestMapping("/internal/profiling")
public class JfrController {

    @PostMapping("/start")
    public ResponseEntity<String> startRecording(@RequestParam int durationSeconds) throws Exception {
        Recording recording = new Recording();
        recording.enable("jdk.GarbageCollection");
        recording.enable("jdk.JavaMonitorWait").withThreshold(Duration.ofMillis(10));
        recording.enable("jdk.SocketRead").withThreshold(Duration.ofMillis(20));
        recording.enable("jdk.ObjectAllocationInNewTLAB");
        recording.setMaxAge(Duration.ofSeconds(durationSeconds + 10));
        recording.setDestination(Path.of("/tmp/recording-" + System.currentTimeMillis() + ".jfr"));
        recording.start();

        Executors.newSingleThreadScheduledExecutor()
            .schedule(recording::stop, durationSeconds, TimeUnit.SECONDS);

        return ResponseEntity.ok("JFR recording started for " + durationSeconds + "s");
    }
}
```

Open `.jfr` files in **JDK Mission Control (JMC)** for visual analysis: flame graphs, GC event timeline, thread state analysis, hot methods by allocation rate.

### How to Read a Flame Graph

```
   ┌──────────────────────────────────────────────────────────────────┐
   │  OrderController.createOrder (wide = ~35% of CPU samples)        │  ← entry point
   ├─────────────────────────────────┬────────────────────────────────┤
   │ OrderService.processItems (25%) │ ValidationService.validate(9%) │
   ├─────────────────┬───────────────┤                                │
   │ JacksonMapper   │ DB query(18%) │                                │
   │ .serialize(7%)  ├────────────┐  │                                │
   │                 │PostgreSQL  │  │                                │
   │                 │Driver(18%) │  │                                │
   └─────────────────┴────────────┴──┴────────────────────────────────┘
          Y-axis: call stack depth (bottom = entry, top = leaf)
          X-axis: sample frequency (wider = more CPU time)
```

**Reading rules:**
- **Wide plateau at the top** of a stack = CPU hotspot in that method. Optimize it.
- **Wide plateau at the bottom** = framework overhead (serialization, reflection). Consider caching or batching.
- **Thin spikes** = short-lived hot paths. Less impactful.
- **Unexpected library frames** (JDBC, GC, serialization) at the top = profile allocation or I/O instead of CPU.

In the example above: 18% of CPU spent in the PostgreSQL driver on a path that runs per-item. This suggests an N+1 query — batching the DB calls would be the fix.

---

## Load Testing

### Choosing a Tool

| Tool | Language | Strengths | Use When |
|:-----|:---------|:----------|:---------|
| **Gatling** | Scala DSL (JVM) | Detailed HTML reports, JVM-native, CI/CD-friendly | Java teams, load tests as code in the same repo |
| **k6** | JavaScript | Simple scripting, cloud execution (k6 Cloud), good CLI output | Polyglot teams, developer-friendly smoke tests |
| **JMeter** | GUI + XML | Wide plugin ecosystem, long enterprise history | Legacy environments where JMeter is already standardized |
| **Locust** | Python | Easy to write dynamic behaviour | Python teams, complex user journeys |

### Gatling (JVM-Native DSL)

```scala
class OrderServiceLoadTest extends Simulation {

  val httpProtocol = http
    .baseUrl("https://api.example.com")
    .acceptHeader("application/json")
    .contentTypeHeader("application/json")
    .header("Authorization", "Bearer ${accessToken}")

  val createAndQueryOrder = scenario("Order lifecycle")
    .exec(
      http("POST /orders - create order")
        .post("/orders")
        .body(StringBody(
          """{"customerId":"cust-1","items":[{"productId":"prod-123","quantity":2}]}"""
        ))
        .check(status.is(201))
        .check(jsonPath("$.orderId").saveAs("newOrderId"))
    )
    .pause(500.milliseconds)
    .exec(
      http("GET /orders/:id - fetch created order")
        .get("/orders/#{newOrderId}")
        .check(status.is(200))
        .check(jsonPath("$.status").is("CONFIRMED"))
    )

  setUp(
    createAndQueryOrder.inject(
      nothingFor(5.seconds),                          // warm up
      rampUsersPerSec(1) to 50 during 60.seconds,    // ramp up
      constantUsersPerSec(50) during 300.seconds,    // sustained load
      rampUsersPerSec(50) to 0 during 30.seconds     // ramp down
    )
  ).protocols(httpProtocol)
   .assertions(
     global.responseTime.percentile(95).lt(200),     // p95 < 200ms
     global.responseTime.percentile(99).lt(500),     // p99 < 500ms
     global.failedRequests.percent.lt(1),            // < 1% errors
     forAll.responseTime.max.lt(5000)                // no single request > 5s
   )
}
```

```bash
# Run in CI
mvn gatling:test -Dgatling.simulationClass=OrderServiceLoadTest

# Results in target/gatling/ordersimulation-*/index.html
```

### k6 (JavaScript, Great for Developer Workflows)

```javascript
import http from 'k6/http';
import { check, sleep } from 'k6';
import { Counter, Trend } from 'k6/metrics';

const ordersCreated = new Counter('orders_created');
const orderLatency  = new Trend('order_create_latency_ms');

export const options = {
  stages: [
    { duration: '30s', target: 10  },   // ramp up
    { duration: '5m',  target: 100 },   // sustained load
    { duration: '30s', target: 0   },   // ramp down
  ],
  thresholds: {
    http_req_duration:    ['p(95)<200', 'p(99)<500'],
    http_req_failed:      ['rate<0.01'],
    orders_created:       ['count>1000'],    // must create at least 1000 orders
  },
};

export default function () {
  const payload = JSON.stringify({
    customerId: `cust-${Math.floor(Math.random() * 1000)}`,
    items: [{ productId: 'prod-123', quantity: 1 }],
  });

  const res = http.post('https://api.example.com/orders', payload, {
    headers: { 'Content-Type': 'application/json' },
  });

  const ok = check(res, {
    'status is 201':         (r) => r.status === 201,
    'has orderId in body':   (r) => r.json('orderId') !== undefined,
    'response time < 200ms': (r) => r.timings.duration < 200,
  });

  if (ok) {
    ordersCreated.add(1);
    orderLatency.add(res.timings.duration);
  }

  sleep(1);
}
```

### Interpreting Load Test Results

What to look for:
- **p95 and p99 latency** — not average. Averages hide outliers that matter to real users.
- **Latency cliff** — at what concurrency level does p99 sharply increase? That's your capacity limit.
- **Error rate vs load** — does the error rate rise gradually or cliff suddenly? A cliff suggests a resource saturation point (connection pool, thread pool, DB connections).
- **GC pause correlation** — overlay JVM GC logs with load test timeline. Latency spikes during Full GC are a tuning signal.

---

## GC Tuning

### Choosing the Right Collector

| GC | Primary Goal | Pause Characteristics | Recommended Heap | Java Version |
|:---|:------------|:---------------------|:-----------------|:-------------|
| **G1GC** | Balance throughput and latency | Configurable pause target (default 200ms) | 4 GB – 32 GB | Java 9+ (default) |
| **ZGC** | Sub-millisecond pauses | < 1ms even at multi-TB heaps | Any (designed for large heaps) | Java 15+ (production-ready) |
| **Shenandoah** | Low pause concurrent | < 10ms typical | Any | Java 12+ |
| **ParallelGC** | Maximum throughput | Hundreds of ms STW acceptable | Any | All versions |
| **SerialGC** | Minimal footprint | Acceptable pauses | < 1 GB | Serverless/CLI tools |

**Decision guide:**
- Latency-critical APIs (SLO p99 < 100ms): **ZGC**
- General-purpose services (SLO p99 < 500ms): **G1GC** with tuned pause target
- Batch jobs where throughput > latency: **ParallelGC**
- Tiny containers (< 512MB): **SerialGC**

### G1GC Tuning

```bash
java \
  -XX:+UseG1GC \
  -Xms4g -Xmx4g \                       # set equal to avoid heap resizing
  -XX:MaxGCPauseMillis=100 \             # G1 targets this pause time (soft goal)
  -XX:G1HeapRegionSize=16m \             # region size 1–32MB; larger = fewer regions
  -XX:G1NewSizePercent=20 \              # min young gen % of heap
  -XX:G1MaxNewSizePercent=40 \           # max young gen %
  -XX:InitiatingHeapOccupancyPercent=45 \# start concurrent marking when heap 45% full
  -XX:ParallelGCThreads=8 \             # STW GC threads (≈ vCPUs for short pauses)
  -XX:ConcGCThreads=2 \                 # concurrent marking threads (don't steal too much)
  -jar app.jar
```

### ZGC Tuning (Much Simpler)

ZGC is largely self-tuning. Key flags:

```bash
java \
  -XX:+UseZGC \
  -Xms8g -Xmx8g \
  -XX:SoftMaxHeapSize=6g \              # ZGC tries to keep live data under this
  -XX:ConcGCThreads=4 \               # concurrent GC threads
  -jar app.jar
```

ZGC automatically adjusts heap sizing to meet the throughput goal (default: 99% of CPU time for application, 1% for GC). Tune `ConcGCThreads` if GC CPU usage is too high.

### GC Logging and Analysis

```bash
# Enable detailed GC logging (JDK 9+ unified logging)
java \
  -Xlog:gc*:file=/var/log/gc/gc.log:time,uptime,level,tags:filecount=5,filesize=20m \
  -jar app.jar

# Quick analysis: find pause outliers
grep "Pause" /var/log/gc/gc.log | awk '{print $NF}' | sort -n | tail -20

# Full GC events indicate tuning is needed
grep "Full GC" /var/log/gc/gc.log | wc -l
```

Open GC logs in **GCEasy.io** or **GCViewer** for visual analysis: heap occupancy over time, pause duration histogram, allocation rate.

### Common GC Problems

| Symptom | Root Cause | Diagnosis | Fix |
|:--------|:-----------|:----------|:----|
| Frequent minor GC, high CPU | Short-lived object storm, Eden exhausted | async-profiler `-e alloc` | Reduce allocations; object pooling |
| Long Full GC (> 1s) | Heap too small or G1 humongous allocations | GC log: `Full GC [Ergonomics]` | Increase heap; reduce allocations > G1RegionSize/2 |
| GC overhead limit (98% GC time) | Near-OOM, likely memory leak | Heap dump via `-XX:+HeapDumpOnOutOfMemoryError` + MAT | Fix leak or increase heap |
| High promotion failure | Survivor space overflow, premature tenuring | GC log: `to-space exhausted` | Increase heap; tune `SurvivorRatio` |
| Latency spikes every N seconds | Scheduled Full GC (explicit or System.gc()) | GC log timestamp alignment | Disable `System.gc()` calls; check `-XX:+DisableExplicitGC` |

### JVM Heap Sizing in Containers

Containers introduce a sizing trap: the JVM defaults to using a fraction of the *physical host* memory, not the container's cgroup limit. `UseContainerSupport` (on by default since JDK 10) reads the cgroup limit correctly.

```bash
# Container with 4GB memory limit
java \
  -XX:+UseContainerSupport \
  -XX:MaxRAMPercentage=75.0 \     # use 75% of container limit = 3GB heap
  -XX:InitialRAMPercentage=50.0 \ # start at 50% = 2GB
  -jar app.jar

# Why not 100%? The remaining 25% (~1GB) is needed for:
# - JIT compiled code cache (default 240MB)
# - Thread stacks (1MB per thread × N threads)
# - DirectByteBuffer (NIO, Netty, Kafka client)
# - Metaspace (class metadata)
# Setting Xmx = container limit → OOMKilled
```

```bash
# Inspect what the JVM thinks it has
java -XX:+PrintFlagsFinal -version | grep -E "MaxHeapSize|InitialHeapSize|UseContainerSupport"

# Check native memory breakdown (requires NMT)
java -XX:NativeMemoryTracking=summary -jar app.jar &
jcmd $(pgrep java) VM.native_memory summary
```

### Heap Dump Analysis

```bash
# Capture heap dump from running JVM
jmap -dump:format=b,file=/tmp/heapdump.hprof $(pgrep java)

# Or auto-capture on OOM (add to JVM flags)
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/tmp/heapdump.hprof

# Analyze with Eclipse Memory Analyzer (MAT)
# 1. Open heapdump.hprof in MAT
# 2. Run "Leak Suspects Report"
# 3. Look for: one object holding references to millions of others
#              unexpected cache growth (Guava, Caffeine, Ehcache)
#              thread-local maps not cleared (MDC, ThreadLocal stores)
```

---

## Key Takeaways for Interviews

1. **Profile before you optimize.** The bottleneck is almost never where you expect. async-profiler's allocation mode finds object storms; CPU mode finds hot loops; lock mode finds contention. Run all three before touching code.
2. **async-profiler avoids safepoint bias.** JVMTI-based profilers only sample at safepoints — they systematically miss tight loops, which are the most common CPU hotspots. This is why async-profiler results often differ dramatically from JVisualVM.
3. **Flame graph reading rule: wide == slow.** A wide plateau at the top of a call stack is a CPU hotspot. Wide near the bottom is framework overhead — usually less actionable.
4. **p95/p99, not average.** Load test assertions on average latency are meaningless. Averages hide the 1% of requests that take 10× longer and that users most remember.
5. **G1GC for general use; ZGC for latency-critical.** G1GC is a good default for 4–32GB heaps with SLO p99 < 500ms. For p99 < 100ms, switch to ZGC — it achieves sub-millisecond pauses at the cost of slightly higher CPU.
6. **Never set `-Xmx` equal to the container memory limit.** Off-heap memory (JIT cache, thread stacks, NIO buffers, Metaspace) consumes memory outside the heap. Use `-XX:MaxRAMPercentage=75` and let the JVM calculate the right heap size from the cgroup limit.

---

## References

- [async-profiler GitHub](https://github.com/async-profiler/async-profiler)
- [Java Flight Recorder (JDK Mission Control)](https://openjdk.org/projects/jmc/)
- [Gatling Documentation](https://gatling.io/docs/gatling/)
- [k6 Documentation](https://grafana.com/docs/k6/latest/)
- [GCEasy — Online GC Log Analyzer](https://gceasy.io/)
- Alexey Shipilev — *JVM Anatomy Quarks* (shipilev.net) — deep JVM internals
- [OpenJDK: ZGC](https://openjdk.org/projects/zgc/)
- *Java Performance* — Scott Oaks (O'Reilly, 2nd Ed)
