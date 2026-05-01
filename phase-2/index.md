---
layout: default
title: "Phase 2: Core System Design"
nav_order: 3
has_children: true
permalink: /phase-2/
---

# Phase 2: Core System Design Concepts
{: .no_toc }

**Duration:** Weeks 5–10
{: .label .label-yellow }

This phase introduces the fundamental building blocks of large-scale systems — how to make them fast, resilient, and consistent. These concepts appear in virtually every system design interview and every real production incident.
{: .fs-5 }

---

## Topics

| # | Topic | Key Concepts |
|:--|:------|:-------------|
| 1 | [Scalability Patterns](scalability/) | Vertical/Horizontal scaling, Load Balancing algorithms, CDN |
| 2 | [Caching](caching/) | Cache strategies, Eviction policies, Redis internals, Caffeine |
| 3 | [Back-of-Envelope Estimation](estimation/) | Latency numbers, QPS, Storage, Bandwidth worked examples |
| 4 | [High Availability Patterns](high-availability/) | Replication, Failover, Circuit Breaker, Retry+Jitter, Bulkhead |
| 5 | [CAP & PACELC Theorem](cap-theorem/) | Consistency models, Real-world system placement |
| 6 | [Rate Limiting & Throttling](rate-limiting/) | Token Bucket, Sliding Window, Distributed Redis rate limiter |

---

## Why Phase 2 Matters

Phase 1 was about understanding the *substrate* — how networks, operating systems, and data structures behave. Phase 2 is about **patterns**: the recurring solutions engineers reach for when a single server stops being enough.

A strong command of this phase means walking into any system design interview and immediately identifying:
- Where the bottleneck will be as traffic grows
- Which failure mode matters most (latency vs availability vs consistency)
- How to trade between cost and reliability

---

## Phase 2 Checklist

- [ ] Explain horizontal vs vertical scaling and where each hits its limit
- [ ] Compare Round Robin, Least Connections, IP Hash, and Consistent Hashing load balancing — when to use each
- [ ] Explain the difference between L4 and L7 load balancing in the context of algorithm choice
- [ ] Design a caching strategy for a read-heavy service (99% reads)
- [ ] Explain cache-aside vs write-through vs write-back with their failure modes
- [ ] Choose the right eviction policy (LRU vs LFU vs ARC) for a given access pattern
- [ ] Explain the Thunder Herd problem and three ways to prevent it
- [ ] Estimate QPS, storage, and bandwidth for a URL shortener
- [ ] Explain CAP theorem with concrete CP and AP examples
- [ ] Know where Cassandra, DynamoDB, ZooKeeper, and MongoDB sit on the CAP diagram
- [ ] Explain PACELC and what it adds to CAP
- [ ] Implement a circuit breaker with Resilience4j in Spring Boot from memory
- [ ] Explain exponential backoff + jitter and why jitter is essential
- [ ] Explain token bucket vs sliding window counter and their burst behavior
- [ ] Implement distributed rate limiting with Redis + Lua script (concept)
