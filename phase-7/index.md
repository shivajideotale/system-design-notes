---
layout: default
title: "Phase 7: Observability & Reliability"
nav_order: 8
has_children: true
permalink: /phase-7/
---

# Phase 7: Observability & Reliability
{: .no_toc }

**Duration:** Weeks 29–32
{: .label .label-yellow }

You can't manage what you can't measure. This phase covers the operational discipline that separates systems that survive production from systems that merely pass CI. Observability (logs, metrics, traces) tells you what your system is doing right now. SLOs and error budgets give you a principled way to balance reliability against feature velocity. Chaos engineering tests your assumptions before production does. Performance engineering closes the loop — profiling and GC tuning translate observations into improvements.
{: .fs-5 }

---

## Topics

| # | Topic | Key Concepts |
|:--|:------|:-------------|
| 1 | [Observability](observability/) | Structured Logging, ELK Stack, Prometheus, Micrometer, OpenTelemetry, Jaeger, SLI/SLO/SLA, Error Budgets |
| 2 | [Reliability Engineering](reliability/) | Chaos Engineering, FMEA, Game Days, RTO/RPO, Backup Strategies, Multi-Region Architecture |
| 3 | [Performance Engineering](performance/) | async-profiler, JFR, Flame Graphs, Gatling, k6, GC Tuning, JVM Heap Sizing |

---

## Why Phase 7 Matters

Observability and reliability are where senior engineers earn their credibility in production systems:

- A service has 99.9% uptime on paper but users still complain about latency spikes — how do you find the root cause across 8 services? (**Distributed tracing + correlation IDs**)
- Your SLO says 99.95% availability — how do you prove it, and when does a new deployment get blocked? (**Error budget burn rate alerts**)
- An incident takes 4 hours to resolve because nobody can correlate logs across services — what should have been in place? (**Structured logging + trace IDs**)
- You want to know if your Order Service can survive a Kafka broker outage before it happens in production — how? (**Chaos experiment with steady-state hypothesis**)
- A failover takes 45 minutes but your SLA promises 15-minute recovery — what specifically needs to change? (**RTO analysis, automated failover, runbook gaps**)
- Your JVM GC is pausing for 800ms every 2 minutes under load — how do you diagnose it and which GC collector should you switch to? (**JFR + flame graphs + GC tuning**)

These are Staff Engineer conversations — not architecture diagrams, but operational maturity under pressure.

---

## Phase 7 Checklist

- [ ] Explain the three pillars of observability and what question each pillar answers
- [ ] Describe how MDC correlation IDs flow from an HTTP request through all downstream service logs
- [ ] Sketch the ELK stack data flow — from application log to Kibana dashboard
- [ ] Explain the RED method and the USE method — when each applies
- [ ] Write a Micrometer `Timer` and `Counter` registration and explain how they surface in Prometheus
- [ ] Write a PromQL query that computes availability SLI (percentage of non-5xx responses)
- [ ] Explain what a 30-day error budget means and how a burn rate alert catches problems before the budget is exhausted
- [ ] Describe the difference between SLI, SLO, and SLA — give a concrete example of each for an API
- [ ] Explain how OpenTelemetry auto-instrumentation works without code changes (Java agent)
- [ ] Describe W3C TraceContext `traceparent` header format and what each field means
- [ ] Compare head-based vs tail-based sampling — when tail-based sampling is required
- [ ] Explain the chaos engineering hypothesis model — steady state, experiment, observation
- [ ] Define RTO and RPO — draw the timeline showing where each is measured
- [ ] Compare active-active vs active-passive multi-region — trade-offs on consistency and cost
- [ ] Explain the difference between full, incremental, and WAL-based backup strategies
- [ ] Describe how async-profiler avoids safepoint bias that afflicts JVMTI-based profilers
- [ ] Read a flame graph — identify a CPU hotspot and an allocation hotspot
- [ ] Compare G1GC, ZGC, and Shenandoah — which to choose for a latency-critical API
- [ ] Explain `MaxRAMPercentage` and why it matters in containerized Java deployments

---

## References

- *Site Reliability Engineering* — Google (free at [sre.google](https://sre.google/books/))
- *The Site Reliability Workbook* — Google (free at [sre.google](https://sre.google/workbook/table-of-contents/))
- [OpenTelemetry Java Documentation](https://opentelemetry.io/docs/instrumentation/java/)
- [Micrometer Documentation](https://micrometer.io/docs)
- [async-profiler](https://github.com/async-profiler/async-profiler)
- [Chaos Toolkit](https://chaostoolkit.org/)
- [Gatling](https://gatling.io/docs/gatling/)
