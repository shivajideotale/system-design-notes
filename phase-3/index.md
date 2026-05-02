---
layout: default
title: "Phase 3: Distributed Systems"
nav_order: 4
has_children: true
permalink: /phase-3/
---

# Phase 3: Distributed Systems
{: .no_toc }

**Duration:** Weeks 11–16
{: .label .label-yellow }

This phase is where senior engineers separate themselves. Distributed systems introduce failures, partial order, and impossibility results that don't exist on a single machine. Mastering this phase means understanding why Kafka, Cassandra, and etcd make the design choices they do — and being able to defend your own choices under pressure.
{: .fs-5 }

---

## Topics

| # | Topic | Key Concepts |
|:--|:------|:-------------|
| 1 | [Distributed Consensus](consensus/) | Paxos, Raft, ZAB, Leader Election, Distributed Locks |
| 2 | [Distributed Transactions](distributed-transactions/) | 2PC, 3PC, SAGA, Outbox Pattern, TCC |
| 3 | [Data Consistency Patterns](data-consistency/) | Event Sourcing, CQRS, CDC, Idempotency, CRDTs |
| 4 | [Distributed Coordination](coordination/) | ZooKeeper, etcd, Consul, Service Discovery |
| 5 | [Clock Synchronization](clocks/) | NTP, TrueTime, Lamport Clocks, Vector Clocks, HLC |

---

## Why Phase 3 Matters

Phase 2 gave you the patterns for scaling a single service. Phase 3 is about what happens when you have **many services that must coordinate**. The classic problems:

- How do you elect a leader without a split-brain? (**Consensus**)
- How do you commit a transaction across Payment + Inventory + Shipping atomically? (**Distributed Transactions**)
- How do you keep read models in sync with write models without a distributed lock? (**Data Consistency**)
- How do you discover services and manage configuration without a single point of failure? (**Coordination**)
- How do you order events when every server has a slightly different clock? (**Clock Synchronization**)

Every large-scale system — Kafka, Cassandra, Kubernetes, Google Spanner — has explicit answers to all five questions. Being able to articulate those answers is what Staff Engineer-level system design looks like.

---

## Phase 3 Checklist

- [ ] Explain the Raft consensus algorithm: leader election, log replication, and safety guarantees
- [ ] Describe what happens in 2PC when the coordinator fails after Phase 1
- [ ] Explain why 3PC still fails under network partitions
- [ ] Design a SAGA for an order flow with payment, inventory, and shipping — including all compensating transactions
- [ ] Explain the Outbox Pattern and why it solves the dual-write problem
- [ ] Describe Event Sourcing vs CQRS and when to use each independently vs together
- [ ] Explain Change Data Capture (CDC) with Debezium and its use cases
- [ ] Define idempotency and design an idempotent payment API
- [ ] Explain how vector clocks detect causality vs concurrency between events
- [ ] Describe what a CRDT is and give two concrete examples
- [ ] Explain ZooKeeper's recipe for distributed locking with ephemeral sequential nodes
- [ ] Describe how etcd is used in Kubernetes and what happens if etcd goes down
- [ ] Compare client-side vs server-side service discovery — trade-offs for each
- [ ] Explain Lamport timestamps and their limitation
- [ ] Describe Google's TrueTime API and why it enables external consistency in Spanner
