---
layout: default
title: "Phase 6: Microservices & API Design"
nav_order: 7
has_children: true
permalink: /phase-6/
---

# Phase 6: Microservices & API Design
{: .no_toc }

**Duration:** Weeks 25–28
{: .label .label-yellow }

Splitting a monolith into microservices solves some problems and introduces others. The services must communicate, the APIs must evolve without breaking clients, and identities must flow across trust boundaries. This phase covers the design principles behind service decomposition (Domain-Driven Design), the three communication protocols every senior engineer must know cold (REST, gRPC, GraphQL), the infrastructure that manages service-to-service traffic at scale (service mesh), and the security primitives that protect API boundaries (OAuth 2.0, JWT, mTLS).
{: .fs-5 }

---

## Topics

| # | Topic | Key Concepts |
|:--|:------|:-------------|
| 1 | [Microservices & DDD](microservices-ddd/) | Bounded Contexts, Aggregates, Service Decomposition, Strangler Fig, Service Mesh, Istio |
| 2 | [API Design](api-design/) | REST best practices, gRPC & Protocol Buffers, GraphQL & DataLoader, API Gateway, BFF |
| 3 | [API Security](api-security/) | OAuth 2.0, OIDC, JWT, Refresh Token Rotation, API Keys, mTLS |

---

## Why Phase 6 Matters

Microservices and API design decisions compound over years. A bad service boundary requires an expensive rewrite. An under-designed API breaks mobile clients when the schema changes. A weak token strategy becomes a critical vulnerability:

- When should Order and Payment be the same service vs separate? (**Bounded Context and aggregate invariants**)
- How do you migrate 3 million users off a monolith without a big-bang rewrite? (**Strangler Fig pattern**)
- Why does gRPC use 50–70% less bandwidth than JSON REST for the same data? (**Protocol Buffers wire format**)
- How do you solve the N+1 query problem where a GraphQL client causes 100 database queries per request? (**DataLoader batching**)
- How does Istio enforce mTLS between services without changing application code? (**Envoy sidecar injection**)
- Why does a leaked JWT remain valid even after the user changes their password? (**Short-lived access tokens + refresh token rotation**)

These questions come up in Staff and Principal Engineer interviews. The answers require knowing not just the API but the internals.

---

## Phase 6 Checklist

- [ ] Define Bounded Context, Aggregate, Entity, and Value Object — give an example from an e-commerce domain
- [ ] Explain why an Aggregate Root enforces invariants and what that implies for transaction scope
- [ ] Describe three Context Mapping patterns — Anti-Corruption Layer, Shared Kernel, Customer-Supplier
- [ ] Explain the Strangler Fig pattern and describe a concrete migration sequence for a monolith endpoint
- [ ] Compare synchronous vs asynchronous inter-service communication — when each introduces unacceptable coupling
- [ ] Explain what a Service Mesh (Istio) does that a load balancer does not
- [ ] Describe how Istio enforces mTLS without changes to application code
- [ ] Describe cursor-based pagination and explain why it outperforms offset pagination on large datasets
- [ ] Explain idempotency keys — how to implement them for a POST /payments endpoint
- [ ] Describe REST versioning strategies (URI, header, query param) and their trade-offs
- [ ] Explain Protocol Buffers field numbers and why removing a field without reserving its number causes corruption
- [ ] Describe the four gRPC streaming modes and give a use case for each
- [ ] Explain the GraphQL N+1 problem and how DataLoader solves it with batching + caching
- [ ] Explain the difference between OAuth 2.0 Authorization Code flow and Client Credentials flow
- [ ] Describe JWT structure and explain why you should validate `iss`, `aud`, `exp`, and signature
- [ ] Explain refresh token rotation and how it detects token theft
- [ ] Describe mTLS — how it differs from server-only TLS, and what it proves about the client
