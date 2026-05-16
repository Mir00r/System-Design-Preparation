# 🛡️ Fault Tolerance: Building Systems That Survive Failures

> *"Everything fails, all the time." — Werner Vogels, CTO of Amazon. The question isn't whether your system will experience failures — it's whether your users will notice when they happen.*

**⏱️ Estimated Time**: 30 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Availability](./Availability.md), [Replication](./Replication.md), [Circuit Breaker](../BuildingBlocks/CircuitBreaker.md)

---

## 📋 Table of Contents
1. [What is Fault Tolerance](#-what-is-fault-tolerance)
2. [Types of Failures](#-types-of-failures)
3. [Redundancy Patterns](#-redundancy-patterns)
4. [Failover Strategies](#-failover-strategies)
5. [Graceful Degradation](#-graceful-degradation)
6. [Chaos Engineering](#-chaos-engineering)
7. [Code Example](#-code-example)
8. [Common Pitfalls](#-common-pitfalls)
9. [Mini Challenge](#-mini-challenge)
10. [Interview Q&A](#-interview-qa)

---

## 🤔 What is Fault Tolerance

```
DEFINITIONS:
  Fault:            a component deviates from its spec (disk fails, bug triggered)
  Error:            the system enters an incorrect state (from a fault)
  Failure:          the system can't deliver its intended service (user-visible)
  
  Fault Tolerance:  the system continues operating correctly despite faults

RELIABILITY TARGETS:
  99.9% (three 9s):   8.76 hours downtime/year   → acceptable for internal tools
  99.99% (four 9s):   52.6 minutes downtime/year → standard for production services
  99.999% (five 9s):  5.26 minutes downtime/year → required for critical infra

  To achieve 99.99%: every component must fail less than 52 min/year
  OR: components can fail more, but redundancy masks the failures
```

---

## ⚡ Types of Failures

```
┌─────────────────────────────────────────────────────────────────────┐
│ FAILURE TYPE     │ DESCRIPTION             │ EXAMPLE                 │
├──────────────────┼─────────────────────────┼─────────────────────────┤
│ Crash failure    │ Node stops completely    │ Server OOM-killed       │
│ Omission failure │ Fails to send/receive   │ Dropped network packets │
│ Timing failure   │ Response too slow        │ GC pause causes timeout │
│ Byzantine failure│ Node behaves arbitrarily │ Corrupted data, hacked  │
│ Performance deg. │ Works but degraded       │ CPU at 100%, slow disk  │
└──────────────────┴─────────────────────────┴─────────────────────────┘

FAILURE DOMAINS (blast radius):
  Process:     one service instance crashes
  Machine:     hardware failure (disk, memory, NIC)
  Rack:        top-of-rack switch failure, power strip failure
  Datacenter:  fire, flood, power outage
  Region:      entire geographic region offline
  Global:      DNS failure, certificate expiry, bad deployment

DESIGN FOR EACH LEVEL:
  Process → restart (systemd, Kubernetes pod restart)
  Machine → replicate to other machines (replica sets)
  Rack → spread replicas across racks (rack-aware placement)
  Datacenter → multi-DC replication (active-active or active-passive)
  Region → multi-region with failover (DNS-based or global LB)
  Global → minimize blast radius (canary deployments, feature flags)
```

---

## 🔁 Redundancy Patterns

```
ACTIVE-PASSIVE (Hot Standby):
  ┌─────────┐          ┌─────────┐
  │ Active  │─replicate─▶│ Passive │  (not serving traffic)
  │ (serves │          │(standby)│
  │ traffic)│          │         │
  └─────────┘          └─────────┘
       │ fails              │
       └────────────────────┘ promote to active (failover)
  
  Used: RDS Multi-AZ, Redis Sentinel, traditional HA databases
  Pros: simple, no split-brain risk
  Cons: standby wastes resources, failover takes 10-30s

ACTIVE-ACTIVE:
  ┌─────────┐          ┌─────────┐
  │ Active  │◄─sync───▶│ Active  │  (both serving traffic)
  │(region1)│          │(region2)│
  └─────────┘          └─────────┘
  
  Used: Global apps, CDN, DNS round-robin
  Pros: no wasted resources, instant failover (just stop routing)
  Cons: conflict resolution needed, more complex

N+1 REDUNDANCY:
  N instances handle normal load + 1 spare
  If any instance fails, remaining N handle 100% traffic
  Example: 3 app servers normally handle 33% each
           If one dies, other 2 handle 50% each (pre-provisioned capacity)

N+2 REDUNDANCY:
  Used for critical systems where simultaneous failures are expected
  Example: during deployments, one instance is being updated + one might crash
```

---

## 🔄 Failover Strategies

```
DNS-BASED FAILOVER:
  health check fails → update DNS record → traffic routes elsewhere
  Pros: simple to implement
  Cons: DNS TTL means 30-300s stale routing (clients cache old IP)

LOAD BALANCER FAILOVER:
  LB health checks backends every 5-10s
  Unhealthy backend → removed from pool → traffic to healthy nodes
  Pros: instant (no DNS delay), seamless for clients
  Cons: LB itself is SPOF (need redundant LBs)

DATABASE FAILOVER:
  Automatic: Sentinel/orchestrator detects failure → promotes replica
  Manual: DBA triggers failover (safer for planned maintenance)
  Semi-automatic: system proposes failover, human approves

MULTI-REGION FAILOVER:
  Normal:  US users → US region, EU users → EU region
  Failure: US region down → all traffic → EU region (with higher latency)
  
  Requirements:
    - Data replicated to all regions (async or sync)
    - Each region can handle 100% of global traffic (capacity planning)
    - Health checks from external monitoring (not self-assessment)
```

---

## 📉 Graceful Degradation

```
PRINCIPLE: Better to serve partial results than no results

EXAMPLES:
  Netflix homepage (region failure):
    Normal: personalized recommendations from ML pipeline
    Degraded: show popular titles (cached, static, always available)
    Result: users see content (maybe less relevant) instead of error page

  E-commerce search (Elasticsearch down):
    Normal: full-text search with facets, relevance, suggestions
    Degraded: redirect to category browsing (works without search)
    Result: users can still find products

  Payment service (fraud detection timeout):
    Normal: real-time fraud scoring on every transaction
    Degraded: allow transactions below $50, queue large ones for review
    Result: most purchases proceed, high-risk ones delayed (not blocked)

IMPLEMENTATION PATTERNS:
  1. Circuit breaker: stop calling failed service, use fallback
  2. Bulkhead: isolate failures to one component (don't cascade)
  3. Timeout + retry: bound waiting time, retry with backoff
  4. Feature flags: disable non-critical features under pressure
  5. Cache fallback: serve stale cached data when source is unavailable
  6. Queue buffering: accept requests, process when service recovers
```

---

## 🐒 Chaos Engineering

```
PRINCIPLE: "Don't wait for failures in production to discover you're not resilient"

NETFLIX SIMIAN ARMY:
  Chaos Monkey:    randomly kills production instances
  Chaos Kong:      simulates entire region failure
  Latency Monkey:  injects artificial network delays
  Conformity Monkey: finds instances not following best practices

CHAOS ENGINEERING PROCESS:
  1. Define "steady state" (normal system behavior, SLOs)
  2. Hypothesize: "system will maintain steady state if X fails"
  3. Inject failure in production (controlled blast radius)
  4. Observe: did system maintain steady state?
  5. If not: fix the weakness, repeat

COMMON EXPERIMENTS:
  - Kill a database primary → does failover work?
  - Block network between services → does circuit breaker trigger?
  - Fill disk to 100% → does system alert and handle gracefully?
  - Spike CPU to 100% on a node → does auto-scaling respond?
  - Inject 5-second latency to dependency → does timeout/fallback work?
  - Corrupt a response from dependency → does validation catch it?

TOOLS:
  Chaos Monkey (Netflix):     random instance termination
  Gremlin:                    enterprise chaos platform
  LitmusChaos:               Kubernetes-native chaos
  Toxiproxy:                 inject network faults in test environments
  AWS Fault Injection Simulator: managed chaos for AWS
```

---

## 💻 Code Example

```java
// Fault-tolerant service with multiple resilience patterns
@Service
public class ProductService {
    private final ProductClient productClient;
    private final Cache<String, Product> localCache;
    private final CircuitBreaker circuitBreaker;
    
    public Product getProduct(String id) {
        // Pattern 1: Circuit breaker wraps remote call
        return circuitBreaker.executeSupplier(() -> {
            try {
                // Pattern 2: Timeout on remote call
                Product product = productClient.getById(id)
                    .timeout(Duration.ofMillis(500))
                    .block();
                
                // Update cache on success
                localCache.put(id, product);
                return product;
                
            } catch (TimeoutException e) {
                // Pattern 3: Fallback to cache on timeout
                return getCachedOrThrow(id);
            }
        }, throwable -> {
            // Pattern 4: Circuit breaker open — use cache
            return getCachedOrThrow(id);
        });
    }
    
    private Product getCachedOrThrow(String id) {
        Product cached = localCache.getIfPresent(id);
        if (cached != null) {
            log.warn("Serving stale cache for product {}", id);
            return cached;
        }
        throw new ServiceUnavailableException("Product service unavailable");
    }
}

// Retry with exponential backoff
@Configuration
public class RetryConfig {
    @Bean
    public RetryTemplate retryTemplate() {
        return RetryTemplate.builder()
            .maxAttempts(3)
            .exponentialBackoff(100, 2.0, 2000)  // 100ms, 200ms, 400ms...
            .retryOn(TransientException.class)   // only retry transient failures
            .build();
    }
}

// Bulkhead: isolate thread pools per dependency
@Configuration
public class BulkheadConfig {
    @Bean
    public ThreadPoolBulkhead paymentBulkhead() {
        return ThreadPoolBulkhead.of("payment",
            ThreadPoolBulkheadConfig.custom()
                .maxThreadPoolSize(10)   // max 10 concurrent calls to payment
                .coreThreadPoolSize(5)
                .queueCapacity(20)       // buffer 20 more
                .build());
    }
    // If payment service is slow, only 10 threads are stuck
    // Other services (product, user) have their own pools — unaffected
}
```

---

## ⚠️ Common Pitfalls

1. **Single points of failure hidden in "redundant" architectures** — You have 3 app servers but one shared database, one load balancer, or one DNS provider. True fault tolerance requires redundancy at EVERY layer. Map your architecture and identify every SPOF.

2. **Untested failover** — Having a standby database doesn't help if you've never tested the failover process. Run failover drills regularly. Netflix's Chaos Monkey works because they test continuously, not annually.

3. **Retry storms** — When a service goes down, all clients retry simultaneously (exponential increase in traffic). The service recovers briefly, gets overwhelmed by retries, and crashes again. Use: exponential backoff + jitter, circuit breakers, and retry budgets (max 10% of requests can be retries).

4. **Cascading failures** — Service A depends on B depends on C. C slows down → B's thread pool fills up → A times out → user sees errors. Solution: timeouts at every boundary, circuit breakers, bulkheads (isolated thread pools per dependency).

---

## 🧩 Mini Challenge

**Your e-commerce checkout service depends on 4 microservices: Inventory, Payment, Shipping, and Notification. Design the fault tolerance strategy — which failures should block checkout vs. which can be handled gracefully?**

<details>
<summary>💡 Click to reveal answer</summary>

```
CRITICAL PATH (must succeed for checkout):
  ✅ Inventory — MUST verify stock (prevent overselling)
  ✅ Payment — MUST charge customer (no free products!)

BEST-EFFORT (can degrade gracefully):
  🟡 Shipping — can QUEUE for later processing
  🟡 Notification — can RETRY async (email confirmation can be delayed)

IMPLEMENTATION:
┌─────────────────────────────────────────────────────────────┐
│ Step 1: Reserve Inventory (SYNC, fail = abort checkout)     │
│   - Timeout: 2 seconds                                      │
│   - Retry: 1 retry with 500ms backoff                       │
│   - Failure: "Item no longer available" error to user        │
│                                                             │
│ Step 2: Process Payment (SYNC, fail = release inventory)    │
│   - Timeout: 5 seconds (payment gateways are slow)          │
│   - Retry: 0 retries (idempotency key prevents double-charge)│
│   - Failure: "Payment failed" error, release inventory      │
│                                                             │
│ Step 3: Create Shipping Order (ASYNC via queue)             │
│   - Publish to message queue (Kafka/SQS)                    │
│   - If queue publish fails: store in local DB, retry later   │
│   - Shipping service processes when available                │
│                                                             │
│ Step 4: Send Notification (ASYNC, fire-and-forget)          │
│   - Publish notification event to queue                      │
│   - If fails: log warning, don't block checkout             │
│   - Retry from dead letter queue later                       │
└─────────────────────────────────────────────────────────────┘

Result: Checkout succeeds even if shipping/notification are down.
User gets: "Order confirmed! Shipping details coming soon."
```

</details>

---

## 📝 Interview Q&A

**Q: How would you design a system to achieve 99.99% availability?**
> A: (1) **Eliminate SPOFs**: replicate everything — databases (multi-AZ replicas), app servers (auto-scaling groups across AZs), load balancers (AWS ALB is inherently redundant). (2) **Automated failover**: database auto-failover (RDS Multi-AZ), Kubernetes pod restart (health checks + readiness probes), circuit breakers for dependencies. (3) **Graceful degradation**: serve cached/partial results when non-critical services fail. (4) **Rolling deployments**: deploy to 1 instance first (canary), monitor errors, then roll out. Never deploy to all instances simultaneously. (5) **Monitoring + alerting**: detect failures before users do (< 1 minute detection time). (6) **Chaos testing**: regularly inject failures to validate resilience. 99.99% = 52 minutes downtime/year, which means no single outage can last more than ~5 minutes.

**Q: What's the difference between fault tolerance and high availability?**
> A: **High availability** means the system is operational a high percentage of time (measured in 9s). It might briefly fail but recovers quickly (e.g., 30-second failover). **Fault tolerance** means the system continues operating WITHOUT interruption even when faults occur — users don't notice the failure at all (e.g., one server crashes, others instantly absorb the load with no dropped requests). Fault tolerance is a stronger guarantee than HA. Example: A system with 30-second database failover is highly available (99.99%) but not truly fault-tolerant (users experience 30s of errors during failover). A system with synchronous multi-master replication and instant load-balancer rerouting is fault-tolerant (zero user-visible impact).

---

## 🔗 What to Read Next

1. **[BuildingBlocks/CircuitBreaker.md](../BuildingBlocks/CircuitBreaker.md)** — Preventing cascading failures
2. **[KeyConcepts/Availability.md](./Availability.md)** — Measuring and achieving high availability
3. **[Observability/Alerting_SLO_SLA_SLI.md](../Observability/Alerting_SLO_SLA_SLI.md)** — Defining reliability targets

---

*[← Consensus](./Consensus.md) | [Back to Index](../INDEX.md) | [Next: Latency vs Throughput →](./LatencyVsThroughput.md)*
