---
layout: default
title: "Phase 9: Cloud-Native & Infrastructure"
nav_order: 10
has_children: true
permalink: /phase-9/
---

# Phase 9: Cloud-Native & Infrastructure
{: .no_toc }

**Duration:** Weeks 36–39
{: .label .label-yellow }

Cloud-native infrastructure is the operational foundation that everything else runs on. Kubernetes schedules, scales, and recovers your services. Terraform and GitOps pipelines provision and reconcile your infrastructure declaratively. Serverless fills the gaps where managed compute beats containers. Cost optimization turns cloud spend from an unmanaged surprise into a predictable, right-sized investment. Senior engineers who understand infrastructure make better architectural decisions — because they know the constraints their code runs under.
{: .fs-5 }

---

## Topics

| # | Topic | Key Concepts |
|:--|:------|:-------------|
| 1 | [Kubernetes Deep Dive](kubernetes/) | Control Plane, Workload Resources, HPA/KEDA, Networking, Storage, Helm, Operators |
| 2 | [Cloud Architecture Patterns](cloud-architecture/) | Well-Architected Framework, Serverless, Terraform, GitOps, Container Security |
| 3 | [Cost Optimization](cost-optimization/) | Right-Sizing, Spot Instances, Reserved Capacity, S3 Lifecycle, FinOps |

---

## Why Phase 9 Matters

Cloud-native decisions surface in Staff Engineer interviews because they constrain every other design decision:

- Your Order Service needs to survive a node failure — how does Kubernetes guarantee pod rescheduling with no data loss for a stateful database? (**StatefulSet + PVC binding**)
- Traffic spikes 10× during flash sales — how does your autoscaling react to Kafka consumer lag rather than CPU? (**KEDA with Kafka trigger**)
- A new microservice takes 45 seconds to deploy because the container pulls a 1.2 GB image — what went wrong and how do you fix it? (**Distroless image + multi-stage build**)
- A developer `kubectl apply`-ed a config directly to production — how does your system detect and revert the drift? (**ArgoCD GitOps reconciliation**)
- Your cloud bill doubled unexpectedly — what is your first diagnostic step? (**Cost allocation tags, Cost Explorer by service and tag**)
- You need to run a Java Lambda that must respond in under 100ms — how do you solve the JVM cold start problem? (**SnapStart, GraalVM native, or Quarkus**)

These are Staff/Principal Engineer conversations — not just "what is Kubernetes" but how infrastructure choices constrain and enable system design.

---

## Phase 9 Checklist

- [ ] Describe the Kubernetes control plane components and what each does
- [ ] Explain the pod scheduling flow — from `kubectl apply` to container running
- [ ] Compare Deployment, StatefulSet, and DaemonSet — give a concrete use case for each
- [ ] Explain the difference between CPU requests/limits and memory requests/limits — what happens when a pod exceeds each?
- [ ] Describe how HPA, VPA, and KEDA differ — when would you use KEDA instead of HPA?
- [ ] Explain the four Kubernetes Service types and when to use each
- [ ] Describe how an Ingress controller works — how does NGINX Ingress terminate TLS?
- [ ] Write a NetworkPolicy that isolates a pod to only accept traffic from a specific namespace
- [ ] Explain the PersistentVolume / PersistentVolumeClaim / StorageClass relationship
- [ ] Describe how Helm templating works — what problem does it solve over raw YAML?
- [ ] Explain the Operator pattern — what is a CRD and what does a controller do?
- [ ] Name the 6 AWS Well-Architected pillars and one key trade-off question per pillar
- [ ] Explain the Lambda cold start problem for JVM — three ways to mitigate it
- [ ] Compare Terraform and AWS CDK — when would you choose one over the other?
- [ ] Describe GitOps with ArgoCD — how does it detect and reconcile drift?
- [ ] Explain why distroless container images improve security — what is removed and why it matters
- [ ] Describe the 3-tier Spot + On-Demand + Reserved capacity model
- [ ] Explain S3 Intelligent-Tiering — how does it differ from configuring lifecycle rules manually?
- [ ] Describe a tagging strategy for cloud cost allocation — which tags are mandatory?

---

## References

- *Kubernetes in Action* — Marko Luksa (2nd Ed)
- [Kubernetes Official Documentation](https://kubernetes.io/docs/)
- [AWS Well-Architected Framework](https://docs.aws.amazon.com/wellarchitected/latest/framework/welcome.html)
- [Terraform Documentation](https://developer.hashicorp.com/terraform/docs)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [KEDA Documentation](https://keda.sh/docs/)
- *Cloud Native Patterns* — Cornelia Davis (Manning)
