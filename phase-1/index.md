---
layout: default
title: "Phase 1: Foundations"
nav_order: 2
has_children: true
permalink: /phase-1/
---

# Phase 1: Foundations & Refresher
{: .no_toc }

**Duration:** Weeks 1–4
{: .label .label-yellow }

Even with 15 years of experience, refreshing core computer science fundamentals at scale is critical. This phase covers the building blocks that every system design question traces back to.
{: .fs-5 }

---

## Topics

| # | Topic | Key Concepts |
|:--|:------|:-------------|
| 1 | [Networking Fundamentals](networking/) | TCP/UDP, HTTP/2/3, DNS, TLS, WebSockets, L4/L7 LB |
| 2 | [OS Concepts](os-concepts/) | Processes, Threads, I/O models, epoll, File systems |
| 3 | [Data Structures for System Design](data-structures/) | Consistent Hashing, Bloom Filters, LSM Trees, B+ Trees |
| 4 | [Java Platform Deep Dive](java-deep-dive/) | JVM GC, JMM, Virtual Threads, Reactive, GraalVM |

---

## Why Phase 1 Matters

As a senior engineer, you know *what* Kafka, Redis, and ZooKeeper do. Phase 1 is about understanding **why** they work the way they do — the networking constraints that force you to batch, the OS I/O models that make epoll-based servers fast, the data structures that make Cassandra's writes O(1) while making reads expensive.

Every architecture decision in the later phases traces back to a fundamental covered here.

---

## Phase 1 Checklist

- [ ] Understand TCP 3-way handshake, connection teardown, TIME_WAIT
- [ ] Explain HTTP/2 multiplexing vs HTTP/1.1 head-of-line blocking
- [ ] Trace a DNS lookup from browser to authoritative NS and back
- [ ] Explain TLS 1.3 handshake — 1-RTT vs 0-RTT
- [ ] Know when to use WebSocket vs SSE vs Long Polling
- [ ] Explain L4 vs L7 load balancing trade-offs
- [ ] Understand process vs thread vs virtual thread in Java
- [ ] Explain blocking vs non-blocking vs async I/O with epoll
- [ ] Implement consistent hashing ring with virtual nodes in your head
- [ ] Explain false positive rate in Bloom Filters and how to tune it
- [ ] Understand why LSM Trees are write-optimized and read-expensive
- [ ] Know difference between G1GC, ZGC, and Shenandoah and when to use each
- [ ] Understand Java Memory Model (happens-before) and volatile semantics
- [ ] Explain Project Loom Virtual Threads and their server design implications
