---
layout: default
title: "1.2 OS Concepts"
parent: "Phase 1: Foundations"
nav_order: 2
---

# Operating System Concepts
{: .no_toc }

<details open markdown="block">
  <summary>Table of Contents</summary>
  {: .text-delta }
1. TOC
{:toc}
</details>

---

## Process vs Thread vs Coroutine

### Process

**What:** An independent execution unit with its own memory space, file descriptors, and OS resources.

- **Isolation:** Crash in one process doesn't affect others (unless shared memory)
- **Communication:** IPC (pipes, sockets, shared memory, message queues) — expensive
- **Context switch cost:** ~1–10 µs (saves/restores full CPU state, TLB flush)
- **Memory:** Each process gets its own virtual address space

**Use in system design:** Separate microservices are processes. Nginx forks worker processes. Database servers (PostgreSQL) use process-per-connection model (vs thread-per-connection).

### Thread

**What:** A lightweight execution unit that shares memory and file descriptors with other threads in the same process.

- **Communication:** Shared memory — fast but requires synchronization
- **Context switch cost:** ~0.1–1 µs (no TLB flush if same process)
- **Memory:** Shares heap, code; each thread has its own stack (~1 MB by default)
- **Failure isolation:** Thread crash can bring down the entire process

**Java Thread — Traditional 1:1 mapping:**
- One Java thread = one OS thread (kernel thread)
- OS can schedule it on any CPU core
- Thread creation is expensive (~1 ms, ~1 MB stack)
- Typical JVM server: 200–500 threads before memory/scheduling overhead hurts

### Virtual Threads (Project Loom — Java 21+)

**The problem with traditional threads:** Blocking I/O (e.g., waiting for database response) parks the OS thread. Server can only handle as many concurrent requests as threads (Tomcat default: 200).

**Virtual Thread model:** Many-to-many. Millions of virtual threads multiplexed over a small pool of OS carrier threads.

```
Virtual Thread 1 ──┐
Virtual Thread 2 ──┼── [Carrier Thread Pool (CPU count)] ── OS
Virtual Thread 3 ──┘
...
Virtual Thread 1M ─/
```

**When a virtual thread blocks on I/O:**
1. JVM unmounts it from the carrier thread
2. Carrier thread picks up another runnable virtual thread
3. When I/O completes, virtual thread is rescheduled

```java
// Java 21: Each request on its own virtual thread
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    for (int i = 0; i < 100_000; i++) {
        executor.submit(() -> {
            // blocking I/O is fine here — virtual thread yields automatically
            String result = httpClient.send(request, BodyHandlers.ofString()).body();
            processResult(result);
        });
    }
}
```

**Impact on server design:**
- Thread-per-request model becomes viable at massive scale again
- Reactive programming (WebFlux) less necessary for I/O throughput
- BUT: CPU-bound work still benefits from thread pools (no benefit from virtual threads)
- Synchronized blocks still pin virtual threads to carrier threads (use `ReentrantLock` instead)

### Coroutines

**What:** User-space cooperative scheduling. Coroutines voluntarily yield control (at suspension points). No kernel involvement.

- **Examples:** Kotlin coroutines, Python asyncio, Go goroutines
- **Scheduling:** Application-level, cooperative (vs OS preemptive)
- **Cost:** Extremely cheap to create (few KB stack)
- **Downside:** Long-running CPU work starves other coroutines (no preemption)

**Comparison:**

| | Process | OS Thread | Virtual Thread | Coroutine |
|:-|:--------|:---------|:--------------|:---------|
| Isolation | Full | Shared memory | Shared memory | Shared memory |
| Creation cost | High (fork) | Medium (~1ms) | Low (~1µs) | Very low |
| Memory | Full address space | ~1MB stack | ~few KB stack | ~few KB |
| Scheduling | OS | OS | JVM + OS | Application |
| Blocking I/O | Blocks process | Blocks thread | Yields to JVM | Yields if async |
| Best for | Isolation | CPU-bound work | I/O-bound work | High concurrency |

---

## Memory Model: Stack, Heap, Off-Heap

### JVM Memory Layout

```
JVM Process
├── Heap (GC managed)
│   ├── Young Generation
│   │   ├── Eden Space
│   │   └── Survivor Spaces (S0, S1)
│   └── Old Generation (Tenured)
├── Non-Heap
│   ├── Metaspace (class metadata, JVM internal)
│   ├── Code Cache (JIT-compiled code)
│   └── Thread Stacks (per thread)
└── Off-Heap / Direct Memory
    └── DirectByteBuffer (java.nio)
```

### Heap vs Stack

| | Stack | Heap |
|:-|:------|:-----|
| What lives here | Method frames, local primitives, references | Objects, arrays |
| Allocation | `push`/`pop` (O(1), very fast) | `new` (GC must find space) |
| Size | Fixed per thread (~1MB default) | Configured via `-Xmx` |
| GC involvement | No GC — stack frame pops on method return | Yes — GC reclaims unreachable objects |
| Thread safety | Per-thread, no sharing | Shared — requires synchronization |
| Lifetime | Method lifetime | Until GC collects |

### Off-Heap Memory (DirectByteBuffer)

```java
ByteBuffer direct = ByteBuffer.allocateDirect(1024 * 1024); // 1MB off-heap
```

**Why off-heap matters for system design:**
- **Zero-copy I/O:** Direct buffers can be passed to OS for I/O without copying to heap. Critical for NIO servers, Netty, Kafka.
- **Avoids GC pressure:** Large data structures (caches like Chronicle Map, off-heap caches) placed off-heap to prevent GC pauses.
- **Controlled via:** `-XX:MaxDirectMemorySize`
- **Netty** uses direct buffers extensively for its I/O path.
- **Kafka producer/consumer** uses page cache (OS-level) — zero-copy via `sendfile()` syscall.

---

## I/O Models

Understanding I/O models is critical for understanding why Nginx, Node.js, and Netty outperform traditional thread-per-connection servers.

### 1. Blocking I/O (BIO)

```
Thread: ----[ READ ]----waiting----waiting----[ DATA RECEIVED ]----continue
           syscall                              kernel copies to userspace
```

- Thread blocks until data arrives
- Simple model — used by old Apache httpd (fork/thread per connection)
- **Problem at scale:** 10K concurrent connections = 10K blocked threads

### 2. Non-Blocking I/O (NIO)

```java
// Java NIO example
ServerSocketChannel ssc = ServerSocketChannel.open();
ssc.configureBlocking(false);

// Must poll repeatedly — wastes CPU if nothing ready
SocketChannel sc = ssc.accept(); // returns null immediately if no connection
```

- System call returns immediately (with `EAGAIN/EWOULDBLOCK` if not ready)
- Application must poll repeatedly — **busy waiting** wastes CPU
- Not used directly — used as building block for multiplexing

### 3. I/O Multiplexing (select/poll/epoll)

**The key insight:** A single thread can watch thousands of file descriptors and be notified when any is ready.

**`select` (POSIX, old):**
- Passes a bitmask of FDs to kernel; kernel scans all of them
- O(n) scan, max 1024 FDs — not scalable

**`epoll` (Linux, modern):**
- Register FDs with the kernel once; kernel maintains a ready-list
- `epoll_wait()` returns only the ready FDs — O(1) for the wait
- No FD limit (beyond system limits)

```
Application registers FDs with epoll
          ↓
epoll_wait() → blocks until some FDs are ready
          ↓
Application reads ONLY the ready FDs
          ↓
repeat
```

**This is how Nginx, Node.js (libuv), Netty, and Java NIO Selector work.**

### 4. Async I/O (AIO, io_uring)

- Application submits I/O request and gets a callback/completion notification
- OS handles I/O in the background
- Application never blocks — not even to check readiness
- `io_uring` (Linux 5.1+): modern async I/O with submission/completion queues
- Java currently uses epoll (NIO); `io_uring` support coming via Project Panama/Loom

### Comparison: I/O Models for a Web Server

| Model | Threads per 10K connections | Syscall cost | Used by |
|:------|:--------------------------|:------------|:--------|
| BIO (blocking) | 10,000 threads | 1 per connection | Old Apache, JDBC |
| NIO + epoll | 1–few threads | O(1) with epoll | Nginx, Node.js, Netty |
| Async I/O | 1–few threads | Zero wait | io_uring apps |

### Why Reactive Matters (Java)

Project Reactor / WebFlux builds on NIO/epoll:

```java
// Reactive: never blocks a thread waiting for I/O
WebClient.create("https://api.service.com")
    .get()
    .uri("/data")
    .retrieve()
    .bodyToMono(String.class)
    .flatMap(data -> processAsync(data))   // no thread parked while waiting
    .subscribe(result -> send(result));
```

With Virtual Threads, blocking I/O in a virtual thread is equivalent to reactive — the JVM handles the yielding automatically. This is why WebFlux becomes less necessary in Java 21+.

---

## CPU Scheduling

### Context Switching

When the OS switches from one thread to another:
1. Save current thread's CPU registers to memory (PCB/TCB)
2. Load next thread's registers
3. (If switching processes) flush TLB, switch page tables

**Cost:** ~1–10 µs for thread context switch, ~10–100 µs for process switch (TLB invalidation).

**System design impact:** At 10K threads, each context switch is ~10µs × 10K = 100ms lost per second just on scheduling. This is why event-driven (single-threaded event loop) servers can outperform thread-per-connection at high concurrency.

### CPU-bound vs I/O-bound Thread Pools

```java
// I/O-bound work: large thread pool (wait a lot, CPU idle)
ExecutorService ioPool = Executors.newFixedThreadPool(200);

// CPU-bound work: pool = CPU cores (no point having more)
ExecutorService cpuPool = Executors.newFixedThreadPool(
    Runtime.getRuntime().availableProcessors()
);

// Or better — use Virtual Threads for I/O-bound (Java 21+)
ExecutorService ioPool = Executors.newVirtualThreadPerTaskExecutor();
```

### NUMA (Non-Uniform Memory Access)

On multi-socket servers, each CPU socket has its own memory bank. Accessing remote memory (another socket's RAM) is 2–4× slower.

**For Java:** `-XX:+UseNUMAInterleaving` or pin JVM to a NUMA node for latency-sensitive workloads. Garbage collectors (G1, ZGC) are NUMA-aware.

---

## File System Internals

### inodes and File System Structure

```
Disk
├── Superblock (FS metadata: block size, inode count, free blocks)
├── inode table
│   ├── inode 1: root directory
│   ├── inode 42: /var/log/app.log (size, perms, timestamps, data block pointers)
│   └── ...
└── Data blocks
    ├── Block 100: actual file data
    └── ...
```

**inode stores:** file permissions, owner, size, timestamps, and **pointers to data blocks** — but NOT the filename. The filename lives in the directory entry.

**Hard links:** Two directory entries pointing to the same inode. `rm` decrements the link count; space freed only when count = 0.

### Page Cache

The kernel caches disk reads in RAM (page cache). On subsequent reads, data comes from RAM (~100ns) not disk (~10ms for HDD).

**Why Kafka is fast:**
1. Messages written to disk sequentially (OS buffers writes, flush is async)
2. Consumer reads are often served from page cache (no disk I/O)
3. `sendfile()` syscall: OS transfers data from page cache to network socket **without copying to userspace** (zero-copy)

```
Producer → [Kafka Broker] → page cache → sequential disk write
Consumer → [Kafka Broker] → page cache → sendfile() → network
```

### B-Trees and inodes

Traditional file systems (ext4, NTFS) use B-Tree variants for directory indexing. Finding a file in a directory with 1 million entries:
- Linear scan: O(n)
- B-Tree: O(log n)

`ext4` uses **HTree** (hash-tree variant of B-Tree) for directory indexing.

---

## Key Takeaways for Interviews

1. **Thread vs Virtual Thread:** Traditional threads → 1:1 OS threads; Virtual Threads → M:N. For I/O-bound servers, virtual threads eliminate the need for reactive programming.
2. **epoll is the foundation** of all high-performance servers: Nginx, Node.js, Netty, Redis. It's O(1) readiness notification.
3. **Context switch cost (1–10µs)** is why thread counts matter at scale. 10K threads = significant scheduling overhead.
4. **Page cache** explains why sequential I/O is fast and why Kafka's disk-based design works at high throughput.
5. **Zero-copy (sendfile)** is the optimization behind Kafka's consumer performance.

---

## References

- *Operating Systems: Three Easy Pieces* — Arpaci-Dusseau (free online: ostep.org)
- *The Linux Programming Interface* — Michael Kerrisk
- [Project Loom overview](https://openjdk.org/projects/loom/)
- Brendan Gregg's systems performance blog
