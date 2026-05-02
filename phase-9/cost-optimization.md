---
layout: default
title: "9.3 Cost Optimization"
parent: "Phase 9: Cloud-Native & Infrastructure"
nav_order: 3
---

# Cost Optimization
{: .no_toc }

<details open markdown="block">
  <summary>Table of Contents</summary>
  {: .text-delta }
1. TOC
{:toc}
</details>

Cloud cost optimization is applied financial engineering. Unmanaged cloud spend grows proportionally with engineering velocity — teams ship fast, provision liberally, and rarely revisit resource sizing. The result is cloud bills that scale faster than the business. FinOps practices close this loop: right-size what you run, commit to capacity you can predict, buy spot capacity for fault-tolerant workloads, and tier storage to match access patterns to cost.

---

## Right-Sizing Instances

### Instance Family Selection

AWS EC2 instance families are optimized for different resource profiles. Picking the right family before tuning size avoids paying for the wrong resource entirely.

| Family | Optimized for | vCPU:RAM ratio | Example instances | Java workload examples |
|:-------|:-------------|:---------------|:------------------|:----------------------|
| **General Purpose (M)** | Balanced CPU and memory | 1:4 | m7g, m6i, m6a | Most Spring Boot services |
| **Compute Optimized (C)** | CPU-bound, low memory | 1:2 | c7g, c6i, c6a | CPU-intensive data processing, media encoding |
| **Memory Optimized (R)** | Memory-bound, large heap | 1:8 | r7g, r6i, x2idn | Large JVM heap (>16GB), in-memory caching (Redis), OLAP |
| **Storage Optimized (I)** | High IOPS local NVMe | — | i4i, im4gn | Elasticsearch, Cassandra, time-series databases |
| **Accelerated (P/G)** | GPU | — | p4d, g5 | ML training/inference |

**Graviton (g-suffix) instances:** AWS ARM-based processors (m7g, c7g, r7g) typically deliver 10–40% better price-performance than equivalent x86 instances. Java workloads run natively on ARM — no code changes required.

### How to Right-Size

Right-sizing from gut feel leads to over-provisioning. Right-size from data:

```bash
# 1. Observe actual resource utilization for 2+ weeks (capture both normal and peak traffic)
# AWS CloudWatch Metrics Insights: find instances running below 20% CPU average

aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-0abc1234 \
  --start-time 2026-04-01T00:00:00Z \
  --end-time 2026-05-01T00:00:00Z \
  --period 86400 \
  --statistics Average Maximum
```

```yaml
# 2. In Kubernetes: use VPA in recommendation mode to see suggested requests/limits
kubectl get vpa order-service-vpa -o yaml

# VPA recommendation output:
# status:
#   recommendation:
#     containerRecommendations:
#       - containerName: order-service
#         lowerBound:
#           cpu: 200m
#           memory: 350Mi
#         target:
#           cpu: 400m          # currently configured: 500m → can reduce by 20%
#           memory: 512Mi      # currently configured: 512Mi → already correct
#         upperBound:
#           cpu: 1500m
#           memory: 900Mi
```

```java
// 3. Profile heap usage under load to right-size JVM heap
// -Xmx should be ~70% of total container memory limit
// For a 1Gi container: -Xmx=717m (70%)
// This leaves ~300Mi for Metaspace, JIT code cache, thread stacks, DirectByteBuffer

// Or use MaxRAMPercentage (adaptive, works regardless of total memory):
// JAVA_OPTS="-XX:MaxRAMPercentage=70.0 -XX:InitialRAMPercentage=50.0"
```

**Right-sizing decision process:**

```
Observe (2 weeks of metrics)
  → p99 CPU > 80% for 30+ min/day? Scale up or scale out
  → avg CPU < 20%? Downsize or use Spot for burst
  → OOMKill events? Increase memory limit or fix heap leak
  → Memory always < 30% used? Downsize memory, check heap settings
```

---

## Spot Instances and Preemptible VMs

### How Spot Works

Spot instances (AWS) / Preemptible VMs (GCP) use spare cloud capacity at discounts of 60–90% vs On-Demand pricing. The trade-off: the cloud provider can reclaim the instance with a **2-minute warning**. Your workload must handle interruption gracefully.

| Workload characteristic | Spot-safe? |
|:------------------------|:-----------|
| Stateless, horizontally scalable | ✅ Yes |
| Can be restarted mid-work (batch with checkpointing) | ✅ Yes |
| CI/CD build runners | ✅ Yes |
| Stateful database primary | ❌ No |
| Long-running transaction coordinator | ❌ No |
| Real-time low-latency API (single instance) | ❌ No (interruption = outage) |

### Spot Best Practices

**Diversify across instance types and AZs.** If you request only `m5.2xlarge` in `us-east-1a`, a capacity event in that pool interrupts all your instances simultaneously. Request a pool of equivalent types:

```json
// EC2 Auto Scaling Mixed Instances Policy
{
  "MixedInstancesPolicy": {
    "InstancesDistribution": {
      "OnDemandBaseCapacity": 1,                 // always keep 1 On-Demand (floor)
      "OnDemandPercentageAboveBaseCapacity": 20, // 20% On-Demand + 80% Spot above floor
      "SpotAllocationStrategy": "capacity-optimized"  // pick pools with most available capacity
    },
    "LaunchTemplate": {
      "Overrides": [
        { "InstanceType": "m6i.2xlarge" },
        { "InstanceType": "m5.2xlarge" },
        { "InstanceType": "m5a.2xlarge" },
        { "InstanceType": "m6a.2xlarge" },
        { "InstanceType": "m7g.2xlarge" }        // diversify across 5 equivalent types
      ]
    }
  }
}
```

**Kubernetes Spot node groups:**

```yaml
# EKS Managed Node Group: Spot capacity for stateless workloads
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig
metadata:
  name: production-cluster
  region: us-east-1
nodeGroups:
  # On-Demand: for stateful, latency-sensitive workloads
  - name: on-demand-m6i
    instanceType: m6i.2xlarge
    desiredCapacity: 3
    minSize: 3
    maxSize: 10
    labels:
      capacity-type: on-demand

  # Spot: for stateless services that can tolerate interruption
  - name: spot-mixed
    instancesDistribution:
      maxPrice: 0.20          # price cap (optional; capacity-optimized already prefers cheap pools)
      instanceTypes: [m6i.2xlarge, m5.2xlarge, m5a.2xlarge, m6a.2xlarge]
      onDemandBaseCapacity: 0
      onDemandPercentageAboveBaseCapacity: 0
      spotAllocationStrategy: capacity-optimized
    desiredCapacity: 5
    minSize: 0
    maxSize: 50
    labels:
      capacity-type: spot
    taints:
      - key: spot
        value: "true"
        effect: NoSchedule      # only pods with a matching toleration land here
```

```yaml
# Application deployment: tolerate spot, prefer spot with weight
spec:
  tolerations:
    - key: spot
      value: "true"
      effect: NoSchedule
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 80
          preference:
            matchExpressions:
              - key: capacity-type
                operator: In
                values: [spot]
```

**Handle interruption in the application:**

```java
// Register a shutdown hook to drain in-flight requests on SIGTERM
// (Kubernetes sends SIGTERM 30 seconds before SIGKILL on eviction)
@Bean
public GracefulShutdown gracefulShutdown() {
    return new GracefulShutdown();
}

@Bean
public WebServerFactoryCustomizer<ConfigurableWebServerFactory> webServerCustomizer(
        GracefulShutdown gracefulShutdown) {
    return factory -> factory.addShutdownHook(gracefulShutdown);
}
```

```yaml
# application.yml: Spring Boot graceful shutdown
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 25s  # drain for up to 25s (within the 30s SIGTERM window)
```

---

## Reserved Capacity

Spot is for workloads that can be interrupted. Reserved capacity is for workloads that run 24/7 and whose size is predictable.

### AWS Savings Plans and Reserved Instances

**Savings Plans (recommended over RIs):** Commit to a consistent spend ($/hour) for 1 or 3 years. More flexible — the discount applies to any instance type or family within the committed category.

| Type | Flexibility | Discount vs On-Demand |
|:-----|:-----------|:----------------------|
| **EC2 Instance Savings Plan** | Locked to instance family + region | Up to 72% |
| **Compute Savings Plan** | Any EC2, Fargate, or Lambda | Up to 66% |
| **Reserved Instance (Standard)** | Specific instance type + AZ | Up to 72% |
| **Reserved Instance (Convertible)** | Can exchange to different type | Up to 54% |

**RDS Reserved Instances:** Databases run 24/7 and are the largest single item on most cloud bills. A 3-year RDS Reserved Instance for the order service database can reduce the DB cost by 60%.

### Three-Tier Capacity Strategy

```
                  Baseline load (predictable)
   ──────────────────────────────────────────────── Reserved / Savings Plan
   
                  Normal traffic variation
   ─── ─── ─── ─── ─── ─── ─── ─── ─── ─── ─── ── On-Demand (Auto Scaling)
   
                  Burst / batch / CI
   ──  ──  ──  ──  ──  ──  ──  ──  ──  ──  ──  ─── Spot Instances
```

```
Total cost example (100 m6i.2xlarge hours/day baseline):

On-Demand only:          100 × $0.384/hr × 24h × 365d = $337,000/yr
Reserved (3yr all-upfront): 100 × $0.145/hr × 24h × 365d = $127,000/yr
Savings: $210,000/yr (62% reduction)
```

**Commitment sizing:** Commit to your P10 utilization (the lowest 10% of daily usage). This ensures the reserved capacity is always consumed — you never pay for unused reserved hours. Let On-Demand and Spot cover the rest.

---

## S3 Storage Tiers and Lifecycle Policies

S3 storage costs accumulate silently. Objects written once and never accessed again continue to incur Standard tier costs indefinitely unless lifecycle policies move or delete them.

### Storage Classes

| Class | Use case | Min storage duration | Retrieval | Cost vs Standard |
|:------|:---------|:---------------------|:----------|:----------------|
| **S3 Standard** | Frequently accessed data | None | Milliseconds | Baseline |
| **S3 Intelligent-Tiering** | Unknown or changing access patterns | 30 days | Milliseconds | Same + monitoring fee |
| **S3 Standard-IA** | Infrequently accessed, but needs fast retrieval | 30 days | Milliseconds | ~46% cheaper storage; retrieval fee |
| **S3 One Zone-IA** | Non-critical infrequent data (single AZ) | 30 days | Milliseconds | ~59% cheaper; single AZ risk |
| **S3 Glacier Instant** | Archive with occasional access | 90 days | Milliseconds | ~68% cheaper storage |
| **S3 Glacier Flexible** | Long-term archive | 90 days | Minutes to hours | ~83% cheaper storage |
| **S3 Glacier Deep Archive** | Regulatory archive (7–10 years) | 180 days | 12–48 hours | ~95% cheaper storage |

**S3 Intelligent-Tiering:** Automatically moves objects between access tiers based on observed access patterns. No retrieval fee for the Standard and Infrequent Access tiers. Charges a small per-object monitoring fee (~$0.0025/1,000 objects/month). Ideal when you don't know — or can't predict — access patterns.

### Lifecycle Policies

```json
{
  "Rules": [
    {
      "ID": "OrderServiceLogs",
      "Status": "Enabled",
      "Filter": { "Prefix": "logs/order-service/" },
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "STANDARD_IA"    // move to IA after 30 days
        },
        {
          "Days": 90,
          "StorageClass": "GLACIER_IR"     // move to Glacier Instant after 90 days
        },
        {
          "Days": 365,
          "StorageClass": "DEEP_ARCHIVE"   // move to Deep Archive after 1 year
        }
      ],
      "Expiration": {
        "Days": 2557                        // delete after 7 years (regulatory retention)
      }
    },
    {
      "ID": "AbortIncompleteMultipartUploads",
      "Status": "Enabled",
      "Filter": { "Prefix": "" },
      "AbortIncompleteMultipartUpload": {
        "DaysAfterInitiation": 7            // clean up failed multipart uploads (often overlooked)
      }
    }
  ]
}
```

```java
// Terraform: S3 lifecycle rule for application artifacts
resource "aws_s3_bucket_lifecycle_configuration" "artifacts" {
  bucket = aws_s3_bucket.artifacts.id

  rule {
    id     = "expire-old-builds"
    status = "Enabled"

    filter { prefix = "builds/" }

    noncurrent_version_transition {
      noncurrent_days = 30
      storage_class   = "STANDARD_IA"
    }
    noncurrent_version_expiration {
      noncurrent_days = 90    // keep last 90 days of build artifacts; delete older versions
    }
    expiration {
      expired_object_delete_marker = true
    }
  }
}
```

---

## FinOps: Cost Visibility and Governance

Cost optimization without visibility is guesswork. FinOps practices create the feedback loop between engineering decisions and cost outcomes.

### Tagging Strategy

Tags are the foundation of cost allocation. Without tags, AWS Cost Explorer shows costs by service, not by team or product.

```
Mandatory tags for all resources:
  env:          production | staging | development
  team:         platform | orders | payments | analytics
  service:      order-service | inventory-service | auth-service
  cost-center:  CC-1234 (maps to finance system)

Optional but useful:
  owner:        alice@example.com
  managed-by:   terraform | cdk | manual
```

```hcl
# Terraform: enforce tags via provider default_tags
provider "aws" {
  region = "us-east-1"
  default_tags {
    tags = {
      env         = var.environment
      team        = "platform"
      managed-by  = "terraform"
      cost-center = "CC-1234"
    }
  }
}
```

**AWS Tag Policies (Organizations):** Enforce mandatory tags across all accounts in an AWS Organization. Resources missing required tags can be blocked from creation via SCPs (Service Control Policies).

### Cost Anomaly Detection

```bash
# AWS Cost Anomaly Detection: alert when a service spends significantly above normal

aws ce create-anomaly-monitor \
  --anomaly-monitor '{"MonitorName":"ServiceMonitor","MonitorType":"DIMENSIONAL","MonitorDimension":"SERVICE"}'

aws ce create-anomaly-subscription \
  --anomaly-subscription '{
    "SubscriptionName": "DailyAnomalyAlert",
    "MonitorArnList": ["arn:aws:ce::123456789:anomalymonitor/abc123"],
    "Subscribers": [{"Address": "infra-oncall@example.com", "Type": "EMAIL"}],
    "Threshold": 20,
    "Frequency": "DAILY"
  }'
# Alert when a service's daily cost exceeds the expected baseline by more than $20
```

### Cost Per Unit Metrics

Tracking absolute cloud spend is necessary but insufficient. Cloud cost should be normalized to a business unit to distinguish "we're spending more because we're growing" from "we're spending more because we're inefficient."

```java
// Emit cost-relevant business metrics alongside infrastructure metrics
// Prometheus metric: orders processed per dollar spent
// Numerator: business throughput (orders/hour from application metrics)
// Denominator: cloud spend rate (from AWS Cost Explorer API, updated hourly)

@Scheduled(fixedRate = 3600000)
public void recordCostEfficiencyMetrics() {
    double ordersLastHour = orderRepository.countOrdersInLastHour();
    double cloudCostLastHour = awsCostClient.getServiceCostLastHour("order-service");

    if (cloudCostLastHour > 0) {
        costEfficiencyGauge.set(ordersLastHour / cloudCostLastHour);
        // Alert if orders per dollar falls below historical baseline → efficiency regression
    }
}
```

### Cost Optimization Checklist

| Category | Check | Typical saving |
|:---------|:------|:---------------|
| **Compute** | Right-size over-provisioned instances (VPA recommendations) | 20–40% |
| **Compute** | Migrate compatible workloads to Graviton (ARM) | 10–20% |
| **Commitment** | Purchase Savings Plans for predictable baseline | 40–60% |
| **Spot** | Move batch, CI, and stateless burst to Spot | 60–80% |
| **Database** | Purchase RDS Reserved Instances | 40–60% |
| **Database** | Downsize RDS to right-sized class (use Performance Insights) | 20–40% |
| **Storage** | Add S3 lifecycle policies (move to IA/Glacier) | 50–80% on old data |
| **Networking** | Enable S3/DynamoDB VPC Endpoints (avoid NAT Gateway charges) | Variable |
| **Idle** | Delete unused EBS volumes, idle ELBs, unattached EIPs | 100% of wasted spend |
| **Containers** | Enable right-sized Fargate Spot for batch ECS tasks | 60–80% |

---

## Key Takeaways for Interviews

1. **Right-size from data, not intuition.** Two weeks of CloudWatch or VPA recommendations will reveal that most production workloads run at 20–30% of provisioned capacity. Profile heap usage before setting `-Xmx`.
2. **The 3-tier strategy is the canonical answer for cloud capacity:** Reserved for predictable baseline (buy at P10 utilization), On-Demand for normal variation, Spot for fault-tolerant burst. Never buy reserved capacity above your P10.
3. **Spot is a design pattern, not just a procurement decision.** To use Spot safely, workloads must be stateless, handle SIGTERM with a graceful shutdown, and run across diverse instance pools and AZs. Bolt-on interruption handling is unreliable.
4. **S3 lifecycle policies are almost always missing.** Objects written once (logs, backups, build artifacts) default to Standard tier storage forever. A simple lifecycle policy moving to IA at 30 days and Glacier at 90 days cuts log storage costs by 80% with no access impact.
5. **Intelligent-Tiering removes the need to predict access patterns.** For any bucket where you cannot confidently predict access frequency, enable Intelligent-Tiering. The monitoring fee is negligible compared to over-provisioning storage tier.
6. **Tags are cost infrastructure.** Without mandatory tags on every resource, cost allocation by team/product is impossible, anomaly detection has no context, and engineers have no feedback loop between their provisioning decisions and the bill. Enforce tags at creation time via Tag Policies, not after the fact.
7. **Cost per unit is more informative than absolute spend.** A rising cloud bill during rapid growth is expected. What matters is cost per order, cost per user, or cost per API call. If cost per unit is rising, that is the efficiency regression to investigate.

---

## References

- [AWS Well-Architected: Cost Optimization Pillar](https://docs.aws.amazon.com/wellarchitected/latest/cost-optimization-pillar/welcome.html)
- [AWS EC2 Spot Best Practices](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-best-practices.html)
- [AWS Savings Plans Documentation](https://docs.aws.amazon.com/savingsplans/latest/userguide/what-is-savings-plans.html)
- [S3 Storage Class Comparison](https://aws.amazon.com/s3/storage-classes/)
- [Kubernetes VPA Documentation](https://github.com/kubernetes/autoscaler/tree/master/vertical-pod-autoscaler)
- [FinOps Foundation: FinOps Framework](https://www.finops.org/framework/)
- [AWS Cost Explorer](https://docs.aws.amazon.com/cost-management/latest/userguide/ce-what-is.html)
