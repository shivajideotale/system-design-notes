---
layout: default
title: "2.4 High Availability Patterns"
parent: "Phase 2: Core System Design"
nav_order: 4
---

# High Availability Patterns
{: .no_toc }

<details open markdown="block">
  <summary>Table of Contents</summary>
  {: .text-delta }
1. TOC
{:toc}
</details>

High availability is measured in **nines**: 99.9% availability means 8.7 hours of downtime per year. 99.99% means 52 minutes. 99.999% means 5 minutes. Every nine costs significantly more than the previous one. The goal is choosing the right trade-off for the business requirement.

---

## Availability Numbers

| SLA | Downtime/year | Downtime/month | Downtime/week |
|:----|:-------------|:--------------|:-------------|
| 99% ("two nines") | 3.65 days | 7.2 hours | 1.68 hours |
| 99.9% ("three nines") | 8.76 hours | 43.8 minutes | 10.1 minutes |
| 99.99% ("four nines") | 52.6 minutes | 4.4 minutes | 1.01 minutes |
| 99.999% ("five nines") | 5.26 minutes | 25.9 seconds | 6.05 seconds |

**Compounding:** If your service depends on 3 services each at 99.9%, your overall availability is `0.999³ = 99.7%` — almost three nines, not three nines. This is why microservice architectures need careful reliability design.

---

## Replication

Replication maintains **copies of data on multiple nodes** to handle node failure and distribute read load.

### Primary-Replica (Master-Slave)

One node accepts writes (primary), others replicate and accept reads (replicas).

```
Write:   Client → Primary (write committed)
                     ↓ replicate (async)
Replica 1         Replica 2

Read:    Client → Replica (stale by replication lag)
      or Client → Primary (always fresh)
```

**Replication lag:** The time between a write on the primary and it being visible on replicas. Usually milliseconds, but can grow to seconds under high load.

**Synchronous vs Asynchronous replication:**

| | Synchronous | Asynchronous |
|:-|:-----------|:------------|
| Write latency | Higher (waits for replica ack) | Lower (fire and forget) |
| Data loss on primary failure | Zero (replica is up-to-date) | Up to replication lag |
| Availability | Lower (write blocked if replica is down) | Higher |
| Default in MySQL | Available but rarely used | Default |
| Default in PostgreSQL | Available (synchronous_standby_names) | Default |

**Semi-synchronous (MySQL default for HA):** Wait for at least one replica to acknowledge before committing. Balance between safety and latency.

**Read-your-writes consistency problem:**

```
User updates profile → writes to primary
User immediately reads profile → routed to replica
Replica hasn't replicated yet → user sees stale profile
```

**Solutions:**
- Route reads of recently written data to the primary
- Use sticky routing for the current user after any write
- Wait until replication lag < threshold before allowing reads

### Primary-Primary (Multi-Primary)

Both nodes accept reads and writes. Requires **conflict resolution**.

```
Client A → Primary 1: UPDATE balance = 100
Client B → Primary 2: UPDATE balance = 200
              ↕ (conflict!)
Which value wins?
```

**Conflict resolution strategies:**
- **Last write wins (LWW):** Use timestamp. Risk: clock skew means correct write loses.
- **Application-level merge:** Custom logic per data type.
- **CRDTs:** Data types that merge automatically (counters, sets).

**Use cases:** Multi-region active-active writes (e.g., accept writes in US-East and EU-West). Complex to operate correctly.

### Quorum-Based Replication

Used in distributed databases (Cassandra, DynamoDB, Riak).

```
N = total replicas
W = write quorum (writes must succeed on W nodes)
R = read quorum (reads must return from R nodes)

Strong consistency: W + R > N

Common config: N=3, W=2, R=2 (QUORUM)
  W + R = 4 > 3 → strong consistency guaranteed
  
Availability-focused: N=3, W=1, R=1
  W + R = 2 ≤ 3 → eventual consistency
```

```bash
# Cassandra: per-query consistency level
# In Java (DataStax driver):
SimpleStatement stmt = SimpleStatement.builder("SELECT * FROM users WHERE id = ?")
    .setConsistencyLevel(ConsistencyLevel.QUORUM)  // read from majority
    .build();

// Or ONE for low-latency reads that tolerate stale data
SimpleStatement stmt = SimpleStatement.builder("SELECT * FROM users WHERE id = ?")
    .setConsistencyLevel(ConsistencyLevel.ONE)
    .build();
```

---

## Failover

What happens when the primary fails?

### Active-Passive (Warm Standby)

One node is active; the standby is ready but idle (or only receives replication).

```
Normal:   Client → Primary (handles all traffic)
                       ↓ replication
                    Standby (idle, receiving data)

Failure:  Primary dies
          Standby detected by monitoring
          Standby promoted → becomes new Primary
          Clients reconnect (DNS TTL update or VIP failover)
```

**RTO (Recovery Time Objective):** Time to detect failure + promote standby + clients reconnect. Typically 30 seconds to 2 minutes.

**RPO (Recovery Point Objective):** With async replication, RPO = replication lag at time of failure. With sync replication, RPO ≈ 0.

**Pros:** Simple, no write conflicts, predictable behavior.  
**Cons:** Standby resources are wasted during normal operation. Failover takes tens of seconds to minutes.

### Active-Active

Both nodes accept traffic simultaneously. Requires conflict handling.

```
Normal:   Client A → Primary 1 (handles writes)
          Client B → Primary 2 (handles writes)
          Both replicate to each other

Benefit: 2× write capacity, no downtime on single node failure
```

**Pros:** Full utilization of all nodes. Instant failover (no promotion needed).  
**Cons:** Write conflicts, complex replication, harder to reason about.

**Use case:** Multi-region active-active databases (DynamoDB global tables, CockroachDB, Cassandra multi-datacenter).

---

## Circuit Breaker

Without a circuit breaker, one slow downstream service causes cascading failures: threads pile up waiting, thread pools exhaust, upstream services also degrade.

**States:**

```
CLOSED → normal operation, requests flow through
         ↓ error rate exceeds threshold
OPEN   → fail fast, return error immediately (no downstream call)
         ↓ after timeout (e.g., 30 seconds)
HALF-OPEN → allow one test request through
         ↓ test request succeeds
CLOSED → resume normal operation
         ↓ test request fails
OPEN   → back to open state
```

### Resilience4j in Spring Boot

```java
// pom.xml: resilience4j-spring-boot3

@Configuration
public class ResilienceConfig {
    @Bean
    public CircuitBreakerConfig paymentCircuitBreakerConfig() {
        return CircuitBreakerConfig.custom()
            .failureRateThreshold(50)          // Open if 50%+ calls fail
            .slowCallRateThreshold(80)          // Also open if 80%+ calls are slow
            .slowCallDurationThreshold(Duration.ofSeconds(2))
            .waitDurationInOpenState(Duration.ofSeconds(30))
            .slidingWindowType(COUNT_BASED)
            .slidingWindowSize(10)              // Last 10 calls
            .minimumNumberOfCalls(5)            // Need at least 5 calls before evaluating
            .permittedNumberOfCallsInHalfOpenState(3)
            .build();
    }
}

@Service
public class PaymentService {
    @Autowired private PaymentClient paymentClient;

    @CircuitBreaker(name = "payment", fallbackMethod = "paymentFallback")
    public PaymentResult processPayment(PaymentRequest request) {
        return paymentClient.process(request); // may throw or be slow
    }
    
    // Fallback: called when circuit is open or call fails
    public PaymentResult paymentFallback(PaymentRequest request, Exception e) {
        log.warn("Payment circuit open, using fallback: {}", e.getMessage());
        // Queue for async retry, or return partial response
        return PaymentResult.queued(request.getId());
    }
}
```

**Metrics to monitor:**
- `resilience4j_circuitbreaker_state` — is it open?
- `resilience4j_circuitbreaker_failure_rate` — is it near threshold?
- `resilience4j_circuitbreaker_calls_total` — volume

---

## Retry with Exponential Backoff + Jitter

### Why Naive Retry Is Dangerous

```
Service A fails
  → 1000 callers retry immediately
  → Service A gets 1000× traffic spike
  → Service A fails harder
  → Retry storm amplifies the outage
```

### Exponential Backoff

Double the wait time on each retry:

```
Attempt 1: wait 1s
Attempt 2: wait 2s
Attempt 3: wait 4s
Attempt 4: wait 8s
Attempt 5: give up (max retries reached)
```

```java
// Formula: min(cap, base × 2^attempt)
long delay = Math.min(MAX_DELAY_MS, BASE_DELAY_MS * (1L << attempt));
```

### Jitter (Essential)

Without jitter, 1000 callers all back off to the same delay — they all retry at the same time.

```
With full jitter: delay = random(0, min(cap, base × 2^attempt))

"Equal jitter" (AWS recommendation):
  v = min(cap, base × 2^attempt)
  delay = v/2 + random(0, v/2)
```

```java
// Resilience4j retry with exponential backoff + jitter
@Bean
public RetryConfig retryConfig() {
    return RetryConfig.custom()
        .maxAttempts(3)
        .intervalFunction(
            IntervalFunction.ofExponentialRandomBackoff(
                Duration.ofMillis(100),  // initial interval
                2.0,                     // multiplier
                0.5,                     // randomization factor (jitter)
                Duration.ofSeconds(10)   // max interval
            )
        )
        .retryExceptions(ConnectException.class, SocketTimeoutException.class)
        .ignoreExceptions(ValidationException.class) // don't retry 400s
        .build();
}

@Service
public class OrderService {
    @Retry(name = "orderService", fallbackMethod = "orderFallback")
    @CircuitBreaker(name = "orderService") // combine CB + retry
    public Order createOrder(OrderRequest request) {
        return orderClient.create(request);
    }
}
```

**Never retry non-idempotent operations without an idempotency key.** A payment retry without an idempotency key can charge a customer twice.

---

## Bulkhead Pattern

Isolate failures — prevent one failing consumer from consuming all resources.

Named after ship compartments: if one floods, the others stay dry.

### Thread Pool Bulkhead

Separate thread pools for different services. Service B's slow calls can't starve Service A's threads.

```
Without bulkhead:
  [Thread Pool: 200 threads]
  Payment service: 150 threads blocked (slow)
  Order service:    50 threads left (starved)
  
With bulkhead:
  [Payment Thread Pool: 100 threads]  ← payment slowness contained here
  [Order Thread Pool:   100 threads]  ← orders unaffected
```

```java
// Resilience4j ThreadPoolBulkhead
@Bean
public ThreadPoolBulkheadConfig bulkheadConfig() {
    return ThreadPoolBulkheadConfig.custom()
        .maxThreadPoolSize(20)
        .coreThreadPoolSize(10)
        .queueCapacity(100)
        .keepAliveDuration(Duration.ofSeconds(60))
        .build();
}

@Service
public class PaymentService {
    @Bulkhead(name = "payment", type = Bulkhead.Type.THREADPOOL)
    @CircuitBreaker(name = "payment")
    public CompletableFuture<PaymentResult> processPaymentAsync(PaymentRequest req) {
        return CompletableFuture.supplyAsync(() -> paymentClient.process(req));
    }
}
```

### Semaphore Bulkhead

Limit concurrent calls without a separate thread pool (for reactive/virtual thread stacks).

```java
// Max 25 concurrent calls to the inventory service
@Bulkhead(name = "inventory", type = Bulkhead.Type.SEMAPHORE)
public InventoryResponse checkInventory(Long productId) {
    return inventoryClient.check(productId);
}
```

---

## Health Checks

### Liveness vs Readiness (Kubernetes)

| | Liveness Probe | Readiness Probe |
|:-|:--------------|:----------------|
| **Question** | Is the app alive? | Is the app ready to serve traffic? |
| **On failure** | Kubernetes restarts the pod | Kubernetes removes pod from load balancer |
| **Use case** | Detect deadlocks, stuck threads | Warm-up period, circuit breaker open, DB connection lost |
| **Risk of being too aggressive** | Restart loops | Traffic loss |

**Liveness:** Must always return 200 unless the pod is truly broken. Don't add DB connection checks — a DB outage shouldn't restart healthy pods.

**Readiness:** Can be strict. If your app can't reach the DB, mark unready. Traffic will be drained to other pods that can reach the DB.

```java
// Spring Boot Actuator + Kubernetes probes
@Component
public class DatabaseReadinessIndicator implements HealthIndicator {
    @Autowired private DataSource dataSource;
    
    @Override
    public Health health() {
        try (Connection conn = dataSource.getConnection()) {
            conn.isValid(1);
            return Health.up().build();
        } catch (SQLException e) {
            return Health.down().withDetail("error", e.getMessage()).build();
        }
    }
}
```

```yaml
# application.yml
management:
  health:
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true
  endpoint:
    health:
      probes:
        enabled: true

# Kubernetes pod spec
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 3
```

---

## Graceful Degradation

When a dependency fails, serve a **degraded but still useful response** rather than erroring entirely.

### Patterns

**Stub response:**
```java
// If recommendation engine is down, return empty list (not 500)
@CircuitBreaker(name = "recommendations", fallbackMethod = "emptyRecommendations")
public List<Product> getRecommendations(Long userId) {
    return recommendationEngine.forUser(userId);
}

public List<Product> emptyRecommendations(Long userId, Throwable t) {
    return Collections.emptyList(); // page still loads, just no recommendations
}
```

**Cached stale data:**
```java
// If user profile service is down, return last known profile from cache
public UserProfile getProfile(Long userId) {
    try {
        return profileService.getProfile(userId);
    } catch (ServiceUnavailableException e) {
        UserProfile stale = cache.getStale(userId); // ignore TTL expiry
        if (stale != null) return stale;
        throw e;
    }
}
```

**Feature flags:**
```java
// Disable expensive feature under load
if (featureFlags.isEnabled("SHOW_LIVE_INVENTORY") && !circuitOpen) {
    product.setInventory(inventoryService.getCount(product.getId()));
} else {
    product.setInventory(null); // show "Check availability" instead
}
```

**Read-only mode:**
```java
// If DB is degraded, serve cached reads but reject writes with 503
if (dbHealthIndicator.getStatus() == DOWN) {
    if (request.getMethod().equals("GET")) {
        return serveFromCache(request);
    } else {
        throw new ServiceUnavailableException("Service in read-only mode");
    }
}
```

---

## Key Takeaways for Interviews

1. **Calculate availability impact of dependencies.** 3 dependencies at 99.9% → 99.7% overall. Always mention this.
2. **Async replication = possible data loss.** The question is how much data loss is acceptable (RPO).
3. **Circuit breaker prevents cascading failures.** Without it, one slow service can take down the entire system.
4. **Jitter is not optional.** Without it, exponential backoff causes synchronized retry storms.
5. **Liveness ≠ Readiness.** This distinction trips up many engineers. Liveness fails → restart. Readiness fails → drain traffic.
6. **Graceful degradation is an architecture decision.** Decide in advance what the fallback is for every external dependency.

---

## References

- [Resilience4j documentation](https://resilience4j.readme.io/)
- [AWS Architecture Blog: Exponential Backoff and Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)
- [Google SRE Book — Chapter 22: Addressing Cascading Failures](https://sre.google/sre-book/cascading-failures/)
- [Spring Boot Actuator + Kubernetes probes](https://docs.spring.io/spring-boot/docs/current/reference/html/actuator.html#actuator.endpoints.kubernetes-probes)
- *Release It!* — Michael Nygard (Circuit Breaker, Bulkhead, Timeouts chapters)
