---
layout: default
title: "1.4 Java Platform Deep Dive"
parent: "Phase 1: Foundations"
nav_order: 4
---

# Java Platform Deep Dive
{: .no_toc }

<details open markdown="block">
  <summary>Table of Contents</summary>
  {: .text-delta }
1. TOC
{:toc}
</details>

---

## JVM Garbage Collectors

Choosing the right GC is an architectural decision with latency, throughput, and memory trade-offs. Understanding internals is essential for tuning high-traffic services.

### Generational Hypothesis

Most objects die young. GC exploits this: cheap minor GCs collect short-lived objects frequently; expensive major GCs collect the old generation rarely.

### G1GC (Garbage-First) — Default since Java 9

**Design:** Region-based heap (1–32 MB regions), no fixed young/old boundaries. Prioritizes collecting regions with most garbage first.

```
Heap divided into equal-sized regions:
[E][E][S][O][O][E][H][O][E][S][O][E] ...
 E = Eden   S = Survivor   O = Old   H = Humongous (large objects)
```

**Phases:**
1. **Minor GC (Young-only):** Collect Eden + Survivor regions → pause (stop-the-world)
2. **Concurrent Marking:** Background threads mark live objects in Old gen (concurrent, no pause)
3. **Mixed GC:** Collect young + some Old regions (stop-the-world)
4. **Full GC:** If mixed GCs can't keep up → stop-the-world full GC (avoid this!)

**Pause target:** `-XX:MaxGCPauseMillis=200` (default 200ms, best-effort)

**When to use:** General purpose. Good balance of latency and throughput. Default JVM choice for most services.

```bash
java -XX:+UseG1GC -Xms4g -Xmx4g -XX:MaxGCPauseMillis=100 -jar app.jar
```

### ZGC — Ultra-Low Latency (Java 15+ production-ready)

**Design:** Concurrent, mostly-concurrent GC. Almost all GC work done while application runs.

**Key properties:**
- **Sub-millisecond pauses** regardless of heap size (even 1 TB heaps)
- Uses **colored pointers** and **load barriers** to track object state concurrently
- Stop-the-world only for root scanning (few microseconds)
- Higher CPU overhead than G1 (~15–20%) due to concurrent work

**When to use:** Latency-sensitive services where GC pauses hurt SLOs (trading systems, real-time recommendations, interactive APIs).

```bash
java -XX:+UseZGC -Xms16g -Xmx16g -jar app.jar
```

### Shenandoah — Ultra-Low Latency (Red Hat, OpenJDK)

**Design:** Like ZGC but also performs concurrent compaction (moves objects while app runs).

**Key difference from ZGC:**
- ZGC: Concurrent mark + relocate (uses load barriers for both)
- Shenandoah: Concurrent mark + concurrent evacuation using **Brooks forwarding pointers**

**When to use:** Similar to ZGC. Good if you're on a distribution where ZGC isn't available or Shenandoah is more tuned.

### GC Selection Guide

| GC | Pause | Throughput | Memory Overhead | Use For |
|:---|:------|:----------|:----------------|:--------|
| G1GC | ~100–500ms | High | Low | Most services |
| ZGC | < 1ms | Slightly lower | Medium | Low-latency APIs |
| Shenandoah | < 1ms | Slightly lower | Medium | Low-latency (RHEL) |
| ParallelGC | Seconds | Highest | Lowest | Batch processing |

### GC Tuning Tips

```bash
# Always run with GC logging to understand what's happening
-Xlog:gc*:file=gc.log:time,uptime,level,tags

# G1: Avoid Full GC by sizing heap correctly
-Xms=Xmx (avoid heap resizing — causes Stop-the-World)
-XX:G1HeapRegionSize=16m  (for large heaps > 8GB)

# Check if GC is the problem
-XX:+PrintGCDateStamps -XX:+PrintGCDetails
```

**Latency rule of thumb:** If your P99 latency spike aligns with GC pauses, switch from G1 to ZGC.

---

## Java Memory Model (JMM)

### The Problem: CPU Caches and Reordering

Modern CPUs don't execute instructions in order. They have multiple levels of caches (L1, L2, L3) and reorder reads/writes for performance. Without explicit ordering guarantees, one thread might not see writes made by another thread.

### Happens-Before Relationship

The JMM defines when one action is **guaranteed to be visible** to another via **happens-before** (HB) rules:

| Action A | Action B | A HB B? |
|:---------|:---------|:--------|
| Write to `volatile` field | Read of same `volatile` field | ✅ |
| `Thread.start()` | All actions in started thread | ✅ |
| All actions in thread | `Thread.join()` returns | ✅ |
| `synchronized` block exit | `synchronized` block entry (same monitor) | ✅ |
| Static initializer | Any access to static field | ✅ |

### volatile

`volatile` guarantees:
1. **Visibility:** Every write to a volatile field is immediately visible to all threads reading it
2. **No reordering:** Prevents load/store reordering around the volatile access
3. **NOT atomicity for compound operations:** `volatile int x; x++` is NOT atomic (read-modify-write)

```java
class Singleton {
    // Double-checked locking — correct with volatile
    private static volatile Singleton instance;
    
    public static Singleton getInstance() {
        if (instance == null) {                    // First check (no lock)
            synchronized (Singleton.class) {
                if (instance == null) {            // Second check (with lock)
                    instance = new Singleton();    // volatile ensures partial construction not visible
                }
            }
        }
        return instance;
    }
}
```

Without `volatile`: `instance = new Singleton()` involves 3 steps:
1. Allocate memory
2. Initialize object
3. Assign reference

CPU/JIT can reorder 2 and 3. Another thread might see a non-null but uninitialized object.

### synchronized

`synchronized` guarantees: mutual exclusion + memory visibility (full barrier).

- Entry: read from main memory (clears stale cache)
- Exit: flush writes to main memory

**Intrinsic locks (monitors) vs `ReentrantLock`:**

| | `synchronized` | `ReentrantLock` |
|:-|:--------------|:----------------|
| Acquisition | Implicit | Explicit `lock()` |
| Release | Block exit | Must call `unlock()` in `finally` |
| Timeout | No | `tryLock(timeout)` |
| Interruptible | No | `lockInterruptibly()` |
| Fairness | No | `new ReentrantLock(true)` |
| Condition variables | `wait()/notify()` | `newCondition()` |
| Virtual thread pinning | YES (pins virtual thread) | No pinning |

{: .important }
**Virtual Threads and synchronized:** In Java 21, `synchronized` blocks **pin** a virtual thread to its carrier OS thread — defeating the purpose of virtual threads for blocking I/O. Replace with `ReentrantLock` in critical paths.

### Atomic Operations

```java
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();   // atomic read-modify-write (CAS loop)
counter.compareAndSet(5, 10); // only set to 10 if current value is 5
```

Atomics use CAS (Compare-And-Swap) CPU instructions — no locking, much faster for uncontended access.

---

## Project Loom: Virtual Threads (Java 21)

### The Motivation

Traditional thread-per-request model:
- Thread creation: ~1 ms, ~1 MB stack
- A single JVM: ~1000 threads practical limit
- Blocking DB call: thread is parked, wasting OS thread for the entire wait

```
Request arrives → allocate thread → [blocking DB call: 50ms] → release thread
                                         ↑
                                   Thread doing nothing for 50ms
                                   but OS thread is still consumed
```

### Virtual Thread Architecture

```
Virtual Thread 1 → requests DB
                → JVM detects blocking call
                → unmounts from carrier thread
Carrier Thread → picks up Virtual Thread 2
Virtual Thread 2 → runs until next blocking point
...
[DB responds to Virtual Thread 1]
Virtual Thread 1 → rescheduled onto a free carrier thread
```

### What Changes with Virtual Threads

**Before (reactive):**
```java
// Had to write async/reactive to handle 10K concurrent requests
return webClient.get()
    .uri("/api/data")
    .retrieve()
    .bodyToMono(String.class)
    .flatMap(data -> processAsync(data))
    .map(result -> Response.ok(result));
```

**After (Java 21 virtual threads):**
```java
// Blocking code — simple to read, scales like reactive
String data = httpClient.send(request, BodyHandlers.ofString()).body();
String result = process(data);  // blocking calls are fine
return Response.ok(result);
```

**Spring Boot 3.2+ virtual thread configuration:**
```yaml
# application.yml
spring:
  threads:
    virtual:
      enabled: true
```

```java
// Or explicitly
@Bean
public TomcatProtocolHandlerCustomizer<?> virtualThreadCustomizer() {
    return handler -> handler.setExecutor(
        Executors.newVirtualThreadPerTaskExecutor()
    );
}
```

### Virtual Thread Gotchas

1. **Synchronized pinning:** `synchronized` pins virtual thread to carrier. Replace with `ReentrantLock`.
2. **ThreadLocal overhead:** Virtual threads are cheap to create — don't cache them. But `ThreadLocal` storage is allocated per virtual thread, can blow up memory.
3. **CPU-bound work:** Still use a bounded thread pool for CPU-intensive work. Virtual threads don't help CPU-bound tasks.
4. **Native methods:** Calls to blocking native code (not Java NIO) still block the carrier thread.

---

## Reactive Programming (Project Reactor / WebFlux)

### When Reactive Still Makes Sense (Java 21+)

Even with virtual threads, reactive is useful for:
- **Back-pressure:** Naturally model producer/consumer with different speeds
- **Declarative pipelines:** Functional operators (`map`, `filter`, `flatMap`, `zip`) compose well
- **Streaming responses:** Streaming large data sets to clients
- **CPU-intensive async pipelines** with explicit scheduler control

### Project Reactor Core Concepts

```java
// Mono = 0 or 1 element
Mono<User> userMono = userRepository.findById(userId);

// Flux = 0 to N elements
Flux<Order> orderFlux = orderRepository.findByUser(userId);

// Composition
Mono<OrderSummary> summary = userMono
    .flatMap(user ->
        orderFlux
            .filter(o -> o.getStatus() == COMPLETED)
            .collectList()
            .map(orders -> buildSummary(user, orders))
    );

// Subscriptions are lazy — nothing happens until subscribe()
summary.subscribe(
    result -> sendResponse(result),
    error -> handleError(error)
);
```

### Schedulers

```java
// Control which thread pool executes each part of the pipeline
Flux.range(1, 100)
    .publishOn(Schedulers.boundedElastic())   // I/O-bound operations
    .flatMap(id -> fetchFromDB(id))
    .publishOn(Schedulers.parallel())          // CPU-bound operations
    .map(data -> transform(data))
    .subscribeOn(Schedulers.single());         // Where subscription starts
```

### Back-Pressure

```java
// Consumer signals to producer how fast it can consume
Flux.range(1, 1_000_000)
    .onBackpressureDrop(dropped -> log.warn("Dropped: {}", dropped))
    .subscribe(new BaseSubscriber<>() {
        @Override
        protected void hookOnSubscribe(Subscription subscription) {
            request(100); // request 100 at a time
        }
        
        @Override
        protected void hookOnNext(Integer value) {
            process(value);
            request(1); // request one more after processing each
        }
    });
```

---

## GraalVM Native Image

### What It Does

Compiles Java bytecode ahead-of-time (AOT) into a **native executable**. No JVM at runtime.

```bash
# Build native image
native-image -jar app.jar

# Result: standalone binary
./app  # starts in ~50ms, uses ~50MB RAM vs JVM's 500ms, 200MB
```

### Benefits

| Metric | JVM (JIT) | GraalVM Native |
|:-------|:----------|:---------------|
| Startup time | 500ms – 5s | 10–100ms |
| Memory (RSS) | 200–500 MB | 30–100 MB |
| Peak throughput | Higher (JIT warms up) | Lower (no JIT) |
| Cloud cost | Higher (longer startup) | Lower (serverless, faster scaling) |

### Use Cases

- **Serverless (AWS Lambda, GCP Cloud Functions):** Cold start cost is critical. Native image eliminates warmup.
- **CLI tools:** Fast startup for command-line utilities
- **Kubernetes scale-to-zero:** Pods that restart frequently benefit from fast startup

### Limitations

- **Closed-world assumption:** All code paths must be known at build time. Reflection, dynamic class loading, serialization need manual configuration.
- **Spring Boot native:** Spring 3.x supports native image but requires `@NativeHint` for reflection.
- **Build time:** Native image compilation is slow (~5–15 minutes) — CI pipeline consideration.

### Quarkus vs Spring Boot for Native

| | Spring Boot (GraalVM) | Quarkus |
|:-|:---------------------|:--------|
| Native support | Spring 3.x — production ready | Native-first since day 1 |
| Startup (native) | ~100ms | ~20ms |
| Dev experience | Superior ecosystem | Better dev mode for native |
| Team knowledge | Leverage existing Spring skills | Learning curve |

---

## JFR (Java Flight Recorder)

### What It Is

Low-overhead (< 2% CPU) continuous profiling framework built into the JVM. Records JVM events: GC, JIT compilation, thread activity, I/O, exceptions, allocations.

### Production Profiling

```bash
# Start recording (safe in production)
jcmd <pid> JFR.start duration=60s filename=recording.jfr

# Or via JVM flags at startup
java -XX:StartFlightRecording=duration=60s,filename=recording.jfr -jar app.jar
```

### Analyzing with JDK Mission Control (JMC)

Open `recording.jfr` in JDK Mission Control. Look for:
- **Hot Methods:** Which methods consume most CPU?
- **Allocation Profiling:** What objects are being created most? Where?
- **GC Events:** Pause durations, promotion failures
- **Thread Blocking:** Which threads are waiting and for how long?
- **Socket I/O:** Which connections are slow?

### async-profiler (Alternative)

Lower overhead than JFR for CPU profiling. Generates flame graphs.

```bash
# Attach to running JVM
./profiler.sh -e cpu -d 30 -f flamegraph.html <pid>
```

**Flame graph reading:** Wider = more CPU time spent in that method. Look for wide flat-topped bars near the top.

---

## Key Takeaways for Interviews

1. **GC choice is an architecture decision:** ZGC for latency-sensitive, G1GC for general, ParallelGC for batch. Always set `-Xms = -Xmx`.
2. **`volatile` ≠ atomic:** It prevents reordering and ensures visibility but doesn't make compound operations atomic.
3. **Double-checked locking needs `volatile`:** Classic Java interview question with real-world importance.
4. **Virtual Threads replace reactive for I/O-bound** in Java 21+. Avoid `synchronized` with virtual threads — use `ReentrantLock`.
5. **GraalVM Native** is the answer for serverless Java and fast-starting containers. Spring 3.x makes it practical.
6. **JFR in production** is safe and essential. If you're not profiling, you're guessing.

---

## References

- [Java Memory Model specification](https://docs.oracle.com/javase/specs/jls/se21/html/jls-17.html)
- [Project Loom JEP 444](https://openjdk.org/jeps/444) — Virtual Threads
- [ZGC documentation](https://wiki.openjdk.org/display/zgc)
- *Java Concurrency in Practice* — Brian Goetz (Chapters 3, 5, 16)
- [GraalVM Native Image docs](https://www.graalvm.org/latest/reference-manual/native-image/)
- [JFR event reference](https://sap.github.io/SapMachine/jfrevents/)
