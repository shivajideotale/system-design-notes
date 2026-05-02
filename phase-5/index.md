---
layout: default
title: "Phase 5: Messaging & Event-Driven Architecture"
nav_order: 6
has_children: true
permalink: /phase-5/
---

# Phase 5: Messaging & Event-Driven Architecture
{: .no_toc }

**Duration:** Weeks 21–24
{: .label .label-yellow }

Messaging systems are the connective tissue of distributed architectures. Once services are split, they must communicate — and synchronous HTTP between every pair of services creates tight coupling, cascading failures, and deployment dependencies. Async messaging decouples producers from consumers in time and space. This phase covers the internals of Kafka, RabbitMQ, and managed cloud queues; the patterns that govern event-driven systems; and the stream processing engines that derive real-time insight from event streams.
{: .fs-5 }

---

## Topics

| # | Topic | Key Concepts |
|:--|:------|:-------------|
| 1 | [Apache Kafka](kafka/) | Log-based architecture, partitioning, ISR, consumer groups, exactly-once semantics, Kafka Streams, Schema Registry |
| 2 | [Messaging Patterns & Brokers](messaging-patterns/) | RabbitMQ exchanges, DLQ, SQS/SNS, Pulsar, pub-sub vs point-to-point, backpressure |
| 3 | [Stream Processing](stream-processing/) | Flink stateful processing, Kafka Streams DSL, Spark Structured Streaming, windowing, watermarks |

---

## Why Phase 5 Matters

The shift from synchronous to asynchronous changes everything about how systems fail, scale, and evolve:

- How does Kafka guarantee ordering while also scaling to thousands of producers? (**Partition model + leader replication**)
- Why does RabbitMQ have four exchange types while SQS has one? (**Push-model routing vs pull-model polling**)
- How does a payment service ensure a charge is processed exactly once despite retries and failures? (**Idempotency + EOS transactions**)
- How does a fraud detection system make a decision in under 10 ms on a stream of 100K events/second? (**Stateful stream processing with Flink**)
- What happens to events that arrive 30 minutes late in a windowed aggregation? (**Watermarks and allowed lateness**)
- How do Kafka Streams applications scale without a dedicated cluster? (**Embedded processing, changelog topics, standby replicas**)

Understanding messaging at this depth separates engineers who wire up a Kafka topic from those who design the partition key, retention policy, consumer group topology, and DLQ strategy for a production system.

---

## Phase 5 Checklist

- [ ] Explain the difference between a log-based message broker (Kafka) and a queue-based broker (RabbitMQ) — when is each appropriate?
- [ ] Describe Kafka's replication model: leader, followers, ISR, and what happens when a follower falls out of ISR
- [ ] Explain the trade-off between producer `acks=1` and `acks=all` — and when you'd choose each
- [ ] Describe exactly-once semantics (EOS) in Kafka: idempotent producer + transactional API — what problem each solves
- [ ] Design a consumer group for a topic with 12 partitions and 4 consumers — what happens if a 5th consumer joins?
- [ ] Explain consumer offset management: auto-commit vs manual commit — when does each cause duplicates or data loss?
- [ ] Describe the Schema Registry + Avro workflow: how does backward/forward compatibility work?
- [ ] Explain RabbitMQ's four exchange types and give a concrete use case for each
- [ ] Design a Dead Letter Queue strategy: when to retry, when to DLQ, when to alert
- [ ] Compare Amazon SQS Standard vs FIFO — trade-offs for ordering and throughput
- [ ] Explain the competing consumers pattern and why it requires idempotent message handlers
- [ ] Describe the backpressure problem and how Project Reactor handles it with reactive streams
- [ ] Explain Kafka Streams topology: KStream vs KTable, GlobalKTable, state stores, and changelog topics
- [ ] Describe the three window types — tumbling, sliding, session — and give a use case for each
- [ ] Explain watermarks: what they are, why they're needed, and the trade-off between lateness tolerance and result freshness
- [ ] Compare Kafka Streams vs Apache Flink vs Spark Structured Streaming — when to choose each
