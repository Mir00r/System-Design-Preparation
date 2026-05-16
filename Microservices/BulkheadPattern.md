# 🚧 Bulkhead Pattern: Isolating Failures to Prevent Cascading Collapse

> *"On a ship, bulkheads are watertight compartments. If one compartment floods, the others stay dry — the ship doesn't sink. In software, bulkheads isolate resource pools so one failing dependency can't consume all your capacity."*

**⏱️ Estimated Time**: 20 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Circuit Breaker](../BuildingBlocks/CircuitBreaker.md), [Fault Tolerance](../KeyConcepts/FaultTolerance.md)

---

## 🤔 The Problem

```
WITHOUT BULKHEAD:
  [Thread Pool: 200 threads shared by all requests]
  
  Payment service becomes slow (5s response time instead of 50ms)
  → Payment requests consume all 200 threads (waiting for responses)
  → Product listing requests can't get a thread
  → Cart service requests can't get a thread
  → ENTIRE SYSTEM DOWN because of ONE slow dependency!

WITH BULKHEAD:
  [Pool: Payment — 30 threads]     ← Payment slow? Only 30 threads affected
  [Pool: Products — 80 threads]    ← Still serving product pages ✅
  [Pool: Cart — 40 threads]        ← Still serving cart operations ✅
  [Pool: General — 50 threads]     ← Everything else still works ✅
  
  Payment goes down → only payment-related requests fail
  Other 170 threads continue serving users normally!
```

---

## 🏗️ Implementation Patterns

```
1. THREAD POOL BULKHEAD:
   Separate thread pool per dependency/operation
   Each pool has fixed max size
   When pool is full → reject immediately (fail fast)
   
2. SEMAPHORE BULKHEAD:
   Limit concurrent calls (no separate threads)
   Lighter weight than thread pools
   Better for reactive/async systems
   
3. CONNECTION POOL BULKHEAD:
   Separate DB/HTTP connection pools per service
   Prevents one service from exhausting all connections
   
4. INFRASTRUCTURE BULKHEAD:
   Separate clusters/instances for different workloads
   Critical path on dedicated infrastructure
   Blast radius limited to one workload
```

---

## 💻 Spring Boot with Resilience4j

```java
// Resilience4j Bulkhead configuration
@Configuration
public class BulkheadConfig {
    
    @Bean
    public BulkheadRegistry bulkheadRegistry() {
        // Thread pool bulkhead for payment calls
        ThreadPoolBulkheadConfig paymentConfig = ThreadPoolBulkheadConfig.custom()
            .maxThreadPoolSize(10)        // max 10 concurrent payment calls
            .coreThreadPoolSize(5)
            .queueCapacity(20)            // buffer 20 more in queue
            .keepAliveDuration(Duration.ofSeconds(30))
            .build();
        
        // Semaphore bulkhead for product catalog (lighter weight)
        BulkheadConfig productConfig = BulkheadConfig.custom()
            .maxConcurrentCalls(25)       // max 25 concurrent catalog calls
            .maxWaitDuration(Duration.ofMillis(500))  // wait 500ms for slot
            .build();
        
        return BulkheadRegistry.of(Map.of(
            "payment", paymentConfig,
            "product", productConfig
        ));
    }
}

// Service with bulkhead annotations
@Service
public class OrderService {
    
    @Bulkhead(name = "payment", type = Bulkhead.Type.THREADPOOL,
              fallbackMethod = "paymentFallback")
    public CompletableFuture<PaymentResult> processPayment(Order order) {
        return CompletableFuture.supplyAsync(() -> 
            paymentClient.charge(order.getCustomerId(), order.getTotal()));
    }
    
    @Bulkhead(name = "product", type = Bulkhead.Type.SEMAPHORE)
    public Product getProduct(String id) {
        return productClient.getById(id);
    }
    
    // Fallback when bulkhead is full
    private CompletableFuture<PaymentResult> paymentFallback(Order order, Exception e) {
        log.warn("Payment bulkhead full, queuing order {}", order.getId());
        // Queue for later processing instead of failing
        paymentQueue.enqueue(order);
        return CompletableFuture.completedFuture(PaymentResult.queued());
    }
}
```

---

## ⚠️ Common Pitfalls

1. **Bulkhead too large** — A bulkhead with 200 threads doesn't protect much. Size it to what the dependency can actually handle. If Payment handles 50 TPS, set bulkhead to ~50 concurrent calls, not 200.

2. **No fallback when bulkhead rejects** — When the bulkhead is full and rejects a request, return a meaningful response (cached data, graceful error, queue for later). Don't just throw a 500 error.

3. **Shared thread pools undermining isolation** — If your bulkheads share the same executor service or connection pool underneath, you haven't actually isolated anything. Verify each bulkhead has truly independent resources.

---

## 📝 Interview Q&A

**Q: How do Bulkhead, Circuit Breaker, and Retry patterns work together?**
> A: They form complementary layers: **Retry** handles transient failures (try again 2-3 times). **Circuit Breaker** detects sustained failures (stop trying after N failures, wait before retrying). **Bulkhead** isolates resource consumption (limit concurrency per dependency so failures don't spread). Typical composition order: `Bulkhead → CircuitBreaker → Retry → actual call`. The bulkhead ensures limited threads are used; within those threads, the circuit breaker decides whether to even attempt the call; if it does, retry handles transient failures. Without bulkhead: a slow service with retries could consume ALL your threads (3 retries × 200 requests = 600 threads blocked).

---

## 🔗 What to Read Next

1. **[BuildingBlocks/CircuitBreaker.md](../BuildingBlocks/CircuitBreaker.md)** — Complementary pattern for detecting failures
2. **[KeyConcepts/FaultTolerance.md](../KeyConcepts/FaultTolerance.md)** — Overall resilience strategy
3. **[Microservices/Sidecar_Pattern.md](./Sidecar_Pattern.md)** — Infrastructure-level isolation

---

*[← Saga Pattern](./Saga_Pattern_Deep_Dive.md) | [Back to Index](../INDEX.md) | [Next: Sidecar Pattern →](./Sidecar_Pattern.md)*
