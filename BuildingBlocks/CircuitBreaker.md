# 🔌 Circuit Breaker: Stop Cascading Failures Before They Destroy Your System

> **"In electrical engineering, a circuit breaker prevents a fault in one part from frying the whole grid. In software, it does exactly the same thing."**

---

## 🎯 What You'll Learn
- How one slow service can take down your entire system (cascading failure)
- The Circuit Breaker pattern — CLOSED, OPEN, HALF-OPEN states explained
- Resilience4j implementation from scratch
- Bulkhead pattern — thread pool isolation
- Retry with exponential backoff and jitter
- How Netflix, Amazon, and Google prevent cascading failures

**⏱️ Estimated Time**: 22 minutes | **🎯 Difficulty**: 🟡 Medium  
**🔗 Prerequisites**: [Service Discovery](./ServiceDiscovery.md) | [API Gateway](./APIGateway.md)  
**🔗 Related Topics**: [Rate Limiting](./RateLimiting.md) | [Microservices/DistributedSystem](../Microservices/DistributedSystem.md)

---

## 📋 Table of Contents
1. [The Cascading Failure Problem](#-the-cascading-failure-problem)
2. [What Is a Circuit Breaker?](#-what-is-a-circuit-breaker)
3. [The Three States](#-the-three-states)
4. [Configuration Parameters](#-configuration-parameters)
5. [Resilience4j Implementation](#-resilience4j-implementation)
6. [Retry with Exponential Backoff](#-retry-with-exponential-backoff)
7. [Bulkhead Pattern: Thread Pool Isolation](#-bulkhead-pattern-thread-pool-isolation)
8. [Fallback Strategies](#-fallback-strategies)
9. [Combined Resilience Strategy](#-combined-resilience-strategy)
10. [Tools & Libraries](#-tools--libraries)
11. [Industry Examples](#-industry-examples)
12. [Common Pitfalls](#-common-pitfalls)
13. [Mini Challenge](#-mini-challenge)
14. [Interview Q&A](#-interview-qa)

---

## 🤔 The Cascading Failure Problem

It's a regular Tuesday. Then:

```
CASCADING FAILURE TIMELINE:

14:30:00 - ProductService database gets slow (disk I/O spike)
14:30:05 - ProductService responses: 10s → 30s timeout
14:30:10 - OrderService calls ProductService, waits 30s
14:30:10 - All 200 OrderService threads are now WAITING for slow ProductService
14:30:15 - OrderService thread pool EXHAUSTED (all threads stuck waiting)
14:30:15 - New requests to OrderService: REJECTED (no threads available)
14:30:20 - CartService calls OrderService → OrderService is now also failing
14:30:25 - CartService thread pool EXHAUSTED
14:30:30 - Your entire application is DOWN
           One slow database took down the whole system
```

This is a **cascading failure** — a slow/failing dependency causes its callers to fail, which causes THEIR callers to fail, until everything is down.

```
❌ WITHOUT CIRCUIT BREAKER:

DatabaseSlow → ProductServiceSlow → OrderServiceDead → CartServiceDead → 💀

✅ WITH CIRCUIT BREAKER:

DatabaseSlow → ProductServiceSlow → Circuit OPENS
               → OrderService gets fast failure (not 30s wait)
               → Falls back to cached data or graceful error
               → CartService unaffected ✅
```

> 💡 **Key Insight**: A slow failure is worse than a fast failure. A Circuit Breaker converts "30-second timeout" into "instant failure" — which is paradoxically better for the overall system.

---

## 💡 What Is a Circuit Breaker?

A **Circuit Breaker** wraps remote calls and monitors for failures. When failures exceed a threshold, it "trips" (opens) and immediately returns errors without actually calling the failing service — giving it time to recover.

Named after the electrical circuit breaker: when too much current flows (failure), the breaker trips and cuts the circuit, preventing damage. When conditions normalize, the breaker resets.

```
CIRCUIT BREAKER WRAPPER:

Without CB:     OrderService ──────────────────────────────► ProductService
                                  waits 30s, gets timeout

With CB:        OrderService ──► [CircuitBreaker] ──► ProductService
                                  CB is CLOSED: passes through
                                  CB is OPEN: immediately returns error
                                  (no call to ProductService)
```

---

## 🌍 Real-World Analogy

```
HOME CIRCUIT BREAKER ANALOGY:

Your home has a circuit breaker panel.
Each circuit protects a set of outlets/appliances.

Normal state: Breaker CLOSED → electricity flows → appliances work.

You plug in a faulty appliance that draws too much current:
  → Breaker trips (OPENS)
  → Electricity stops flowing
  → Only THAT circuit is cut — rest of house works fine
  → You investigate and fix the problem
  → Reset the breaker (HALF-OPEN test)
  → If OK → CLOSED again → normal operation

Without the breaker: The fault would spread to adjacent circuits,
potentially burning down the whole house.

Software circuit breaker works exactly the same way.
```

---

## 🔄 The Three States

```
                ┌─────────────────────────────────┐
                │                                 │
         failures > threshold             success rate OK
                │                                 │
                ▼                                 │
┌──────────┐  TRIP  ┌─────────┐  ALLOW PROBE  ┌──────────┐
│  CLOSED  │───────►│  OPEN   │──────────────►│HALF-OPEN │
│(normal)  │        │(failing)│               │(testing) │
└──────────┘        └─────────┘               └──────────┘
                         │                         │
                         │ probe fails              │ probe succeeds
                         └─────────────────────────┘
                               stay OPEN           → CLOSED
```

### State 1: CLOSED (Normal Operation)

```
State: CLOSED
All requests pass through to the actual service.
Success/failure tracked in a sliding window.

If failures exceed threshold → TRIP → go to OPEN
```

### State 2: OPEN (Circuit Tripped)

```
State: OPEN
NO requests pass through to the actual service.
All calls immediately return a pre-defined error or fallback.

Benefits:
  - No more 30-second timeouts — instant failure
  - Failing service gets zero load → can recover
  - Caller threads freed up immediately

After wait duration (e.g., 60 seconds) → go to HALF-OPEN
```

### State 3: HALF-OPEN (Testing Recovery)

```
State: HALF-OPEN
Allow a LIMITED number of test requests through.

If test requests succeed → CLOSED (service recovered! 🎉)
If test requests fail    → OPEN again (still broken, wait more)
```

---

## ⚙️ Configuration Parameters

```
CIRCUIT BREAKER CONFIG:

FAILURE_RATE_THRESHOLD = 50%     ← If >50% of requests fail → OPEN
SLOW_CALL_RATE_THRESHOLD = 100%  ← If >100% of calls are slow → OPEN
SLOW_CALL_DURATION_THRESHOLD = 2s ← "Slow" = takes more than 2 seconds
MINIMUM_NUMBER_OF_CALLS = 10     ← Need at least 10 calls before evaluating
SLIDING_WINDOW_SIZE = 20         ← Evaluate last 20 calls
WAIT_DURATION_IN_OPEN_STATE = 60s ← Wait 60s before trying HALF-OPEN
PERMITTED_CALLS_IN_HALF_OPEN = 5  ← Allow 5 probe requests in HALF-OPEN
```

---

## 💻 Resilience4j Implementation

### Setup (Spring Boot)

```xml
<!-- pom.xml -->
<dependency>
    <groupId>io.github.resilience4j</groupId>
    <artifactId>resilience4j-spring-boot3</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId>
</dependency>
```

```yaml
# application.yml
resilience4j:
  circuitbreaker:
    instances:
      productService:
        failure-rate-threshold: 50          # Open when >50% fail
        slow-call-rate-threshold: 100
        slow-call-duration-threshold: 2000  # 2 seconds = "slow"
        minimum-number-of-calls: 10
        sliding-window-size: 20
        wait-duration-in-open-state: 60000  # 60 seconds in OPEN
        permitted-number-of-calls-in-half-open-state: 5
        automatic-transition-from-open-to-half-open-enabled: true
```

### Basic Usage with @CircuitBreaker

```java
@Service
public class OrderService {
    
    @Autowired
    private ProductServiceClient productClient;
    
    // Annotation-based — declarative and clean
    @CircuitBreaker(name = "productService", fallbackMethod = "getProductFallback")
    public Product getProduct(String productId) {
        return productClient.getProduct(productId);
    }
    
    // Fallback called when circuit is OPEN or exception occurs
    public Product getProductFallback(String productId, Exception e) {
        log.warn("Circuit open for product: {}, reason: {}", productId, e.getMessage());
        
        // Fallback strategies (pick appropriate one):
        // 1. Return cached data
        return productCache.getOrDefault(productId, Product.defaultProduct());
        // 2. Return partial data
        // 3. Return friendly error message
    }
}
```

### Programmatic Usage (More Control)

```java
@Service
public class ResilientOrderService {
    
    private final CircuitBreaker circuitBreaker;
    private final ProductServiceClient productClient;
    
    public ResilientOrderService(CircuitBreakerRegistry registry,
                                  ProductServiceClient productClient) {
        this.circuitBreaker = registry.circuitBreaker("productService");
        this.productClient = productClient;
        
        // Event listeners for monitoring
        this.circuitBreaker.getEventPublisher()
            .onStateTransition(event -> 
                log.info("Circuit state changed: {} → {}", 
                    event.getStateTransition().getFromState(),
                    event.getStateTransition().getToState()))
            .onError(event -> 
                log.warn("Circuit recorded failure: {}", event.getThrowable().getMessage()));
    }
    
    public Product getProduct(String productId) {
        // Wrap call in circuit breaker
        Supplier<Product> decoratedSupplier = CircuitBreaker
            .decorateSupplier(circuitBreaker, () -> productClient.getProduct(productId));
        
        // Execute with fallback
        return Try.ofSupplier(decoratedSupplier)
            .recover(CallNotPermittedException.class, 
                ex -> getFromCache(productId))   // Circuit OPEN
            .recover(Exception.class, 
                ex -> Product.defaultProduct())  // Any other failure
            .get();
    }
    
    private Product getFromCache(String productId) {
        // Redis or in-memory cache lookup
        return productCache.get(productId);
    }
}
```

---

## 🔁 Retry with Exponential Backoff

Retry is complementary to Circuit Breaker:
- **Retry**: Try again when a call fails (for transient errors)
- **Circuit Breaker**: Stop trying when failure rate is high (for systemic failures)

```
⚠️ IMPORTANT: Always combine with Circuit Breaker
If you retry 3x on every call and circuit is open:
3 retries × 1000 concurrent requests = 3000 calls to broken service → thundering herd!
```

```java
// application.yml
resilience4j:
  retry:
    instances:
      productService:
        max-attempts: 3
        wait-duration: 500ms
        enable-exponential-backoff: true
        exponential-backoff-multiplier: 2   # 500ms, 1000ms, 2000ms
        exponential-max-wait-duration: 10s
        retry-exceptions:
          - java.net.ConnectException
          - java.net.SocketTimeoutException
        ignore-exceptions:
          - com.example.OrderNotFoundException  # Don't retry business errors
```

```java
@CircuitBreaker(name = "productService", fallbackMethod = "fallback")
@Retry(name = "productService")   // Retry INSIDE circuit breaker
public Product getProduct(String productId) {
    return productClient.getProduct(productId);
}
```

### Exponential Backoff with Jitter

```
WITHOUT JITTER (thundering herd):
All 1000 clients fail at t=0
All retry at t=500ms simultaneously
All retry at t=1000ms simultaneously → same problem!

WITH JITTER (spread the load):
Each client adds random jitter to wait time:
  wait = min(base * 2^attempt, maxWait) + random(0, jitter)

Client 1: 500ms + 43ms = 543ms
Client 2: 500ms + 189ms = 689ms
Client 3: 500ms + 67ms = 567ms
...spread across a range → no thundering herd
```

---

## 🧱 Bulkhead Pattern: Thread Pool Isolation

Even with Circuit Breaker, one slow service can exhaust a shared thread pool.

```
❌ PROBLEM: Shared thread pool

Thread pool: 200 threads (shared)
ProductService calls: consuming 150 threads (slow)
OrderService calls: only 50 threads left
CartService calls: no threads left → rejected!

Even though CartService → its own dependency is fine,
it's starved by ProductService's slowness.

✅ SOLUTION: Isolated thread pools (Bulkhead)

ProductService: dedicated pool of 50 threads
OrderService:   dedicated pool of 50 threads
CartService:    dedicated pool of 50 threads

ProductService can max out its 50 threads — CartService unaffected.
```

```yaml
# application.yml
resilience4j:
  bulkhead:
    instances:
      productService:
        max-concurrent-calls: 50      # Max simultaneous calls
        max-wait-duration: 10ms       # How long to wait for a slot
```

```java
@CircuitBreaker(name = "productService", fallbackMethod = "fallback")
@Bulkhead(name = "productService", type = Bulkhead.Type.THREADPOOL)
@TimeLimiter(name = "productService")  // timeout per call
public CompletableFuture<Product> getProductAsync(String productId) {
    return CompletableFuture.supplyAsync(() -> 
        productClient.getProduct(productId)
    );
}
```

---

## 🛟 Fallback Strategies

When the Circuit is OPEN or a call fails, what do you return?

```
FALLBACK OPTIONS (choose by impact):

1. CACHED DATA (best when available)
   Return last known good data from Redis/in-memory cache
   User sees slightly stale data — acceptable in most cases

2. DEGRADED RESPONSE  
   Return partial data (product name only, no price)
   Better than nothing

3. DEFAULT RESPONSE
   Return a safe default value
   "Sorry, product details temporarily unavailable"

4. STATIC FALLBACK
   Return pre-built fallback response stored locally
   "Recommended products" → return hardcoded top-10

5. FAIL FAST
   Return HTTP 503 Service Unavailable immediately
   Better than making user wait 30 seconds for same result

NEVER:
   - Silently return null/empty (causes NullPointerException downstream)
   - Retry indefinitely in the fallback
   - Call ANOTHER slow service in the fallback
```

---

## 🏰 Combined Resilience Strategy

The full resilience stack for a production-grade service call:

```
ORDER OF DECORATORS (outside to inside):
                                             
Bulkhead → CircuitBreaker → RateLimiter → TimeLimiter → Retry → actual call

1. Bulkhead:        Don't exceed 50 concurrent calls
2. CircuitBreaker:  Don't call if failure rate > 50%
3. RateLimiter:     Don't exceed 100 calls/second
4. TimeLimiter:     Timeout if call takes > 2 seconds
5. Retry:           Retry up to 3x on transient errors
6. Actual call:     Make the real HTTP/gRPC call
```

```java
@Bean
public Function<String, Product> resilientProductCall(
        CircuitBreakerRegistry cbRegistry,
        RetryRegistry retryRegistry,
        BulkheadRegistry bulkheadRegistry) {
    
    CircuitBreaker cb = cbRegistry.circuitBreaker("productService");
    Retry retry = retryRegistry.retry("productService");
    Bulkhead bulkhead = bulkheadRegistry.bulkhead("productService");
    
    return Decorators.ofSupplier(productId -> productClient.getProduct(productId))
        .withCircuitBreaker(cb)
        .withRetry(retry)
        .withBulkhead(bulkhead)
        .withFallback(List.of(Exception.class), 
            ex -> Product.defaultProduct())
        .decorate();
}
```

---

## 🛠️ Tools & Libraries

| Library | Language | Features |
|---|---|---|
| **Resilience4j** | Java | Circuit Breaker, Retry, Bulkhead, Rate Limiter, TimeLimiter. Lightweight, functional |
| **Spring Cloud Circuit Breaker** | Java | Abstraction layer over Resilience4j or Sentinel |
| **Hystrix** (Netflix) | Java | Original! Now in maintenance mode. Resilience4j is successor |
| **Polly** | .NET | CB, Retry, Bulkhead for .NET ecosystem |
| **Hystrix-go / gobreaker** | Go | Circuit breaker for Go services |
| **Envoy Proxy** | Any | Circuit breaking at the service mesh/proxy level (no code changes) |
| **Istio** | Any | Service mesh with built-in circuit breaking via Envoy |

---

## 🏢 Industry Examples

```
NETFLIX — Created the Circuit Breaker pattern in software:
- Hystrix library (2012) — open-sourced, now in maintenance
- "Chaos Monkey" — randomly kills services to test resilience
- "Simian Army" — broader chaos engineering tools
- Every service call wrapped in circuit breaker
- Graceful degradation: missing personalization → show generic content

AMAZON:
- "Exponential backoff with jitter" pattern published by Amazon
- Used across all AWS services for retry logic
- DynamoDB SDK has built-in circuit breaker behavior
- "Availability Zone isolation" — CB pattern at infrastructure level

UBER:
- Uses circuit breakers to protect against downstream slowness
- "Driver location service" has CB protecting map service calls
- Fallback: show last known driver location (stale but better than nothing)

GOOGLE:
- "Fail open" vs "fail closed" decisions documented in SRE Book
- Circuit breaking at gRPC level built into service framework
- Load shedding (a related pattern) to protect critical services
```

---

## ⚠️ Common Pitfalls

### 1. Circuit Breaker with No Fallback
```
❌ BAD: Circuit opens → returns generic 500 error to user
✅ GOOD: Circuit opens → returns cached data or friendly degraded response
```

### 2. Too Aggressive Thresholds
```
❌ BAD: failure-rate-threshold: 10%
         Any normal transient blip opens the circuit
         Circuit oscillates open/closed constantly
✅ GOOD: failure-rate-threshold: 50% with minimum-calls: 20
         Give the service a fair chance before tripping
```

### 3. Retrying Without Circuit Breaker
```
❌ BAD: Retry 3x on every call, no circuit breaker
         Slow service × 3 retries × 1000 users = 3000 calls to broken service
✅ GOOD: Circuit breaker + retry
         Once circuit opens → retries stop → fast failure for everyone
```

### 4. Same Timeout for Fast and Slow Operations
```
❌ BAD: All calls: 5-second timeout
         Fast GET /products takes 50ms
         Slow POST /generate-report takes 4 seconds
         Users wait 5 seconds on failed reports
✅ GOOD: GET /products: 500ms timeout
         POST /generate-report: 10-second timeout
         Match timeout to expected operation time
```

### 5. Not Monitoring Circuit State
```
❌ BAD: Circuit opens → nobody knows → 30 minutes of degraded service
✅ GOOD: Emit metrics/events on state transitions
         Alert: "productService circuit OPENED" → PagerDuty
         Dashboard: real-time circuit breaker states
```

---

## 🧩 Mini Challenge

```
🎲 SCENARIO (4 minutes):

Black Friday. Your system:
  CartService → [calls] → InventoryService
  InventoryService → [checks] → Redis Cache → [falls back to] → Database

At 12:00 PM: Database suddenly slow (5-second queries).
Redis cache hit rate: 60% (40% cache misses go to slow DB).
CartService has 500 concurrent requests/second.

WITHOUT CIRCUIT BREAKER, what happens step by step?
WITH CIRCUIT BREAKER, what's the ideal configuration?
```

<details>
<summary>💡 Click to reveal answer</summary>

**WITHOUT Circuit Breaker:**
```
t=0: DB slows (5s per query)
t=1: 40% of InventoryService calls hit slow DB → take 5s
t=5: InventoryService thread pool filling up with waiting threads
t=10: InventoryService thread pool FULL (500 threads × 5s backlog)
t=10: CartService gets timeouts from InventoryService
t=15: CartService threads exhausted waiting for InventoryService
t=20: 💀 Entire checkout flow down
```

**WITH Circuit Breaker (ideal config):**
```yaml
circuitbreaker:
  inventoryService:
    failure-rate-threshold: 50       # Open at 50% failure
    slow-call-duration-threshold: 1000  # 1s = "slow" (not 5s)
    slow-call-rate-threshold: 80     # Open when 80% are slow
    minimum-number-of-calls: 20
    wait-duration-in-open-state: 30000  # 30s open (let DB recover)
    
timeLimiter:
  inventoryService:
    timeout-duration: 1s             # Never wait more than 1s
    
retry:
  inventoryService:
    max-attempts: 2                  # 1 retry only
    wait-duration: 200ms
```

**With this config:**
- `t=5`: Circuit detects 80%+ slow calls → OPENS
- `t=5+`: CartService gets instant failure from CB (not 5s wait)
- `t=5+`: CartService fallback: "Use last known inventory" from its own cache
- Users see: "Availability may be slightly delayed" (not "500 Error")
- InventoryService: gets zero traffic → DB pressure drops → recovers
- `t=35s`: CB HALF-OPENS → test request succeeds → CLOSES → normal operation
- **Total degraded window: ~35 seconds vs potentially hours**
</details>

---

## 📝 Interview Q&A

**Q1: What is the Circuit Breaker pattern and why is it needed?**
> Circuit Breaker prevents cascading failures in microservices. When a downstream service is failing, repeated calls waste threads and amplify the failure. The CB tracks failure rates and "trips" (opens) when failures exceed a threshold — returning fast errors instead of waiting for timeouts. This frees up threads and gives the failing service time to recover.

**Q2: Describe the three states of a Circuit Breaker.**
> CLOSED: Normal operation, all calls pass through, failures tracked. OPEN: All calls immediately fail (no actual service calls) — entered when failure threshold exceeded. HALF-OPEN: After a wait period, allows limited probe requests to test if service recovered — if probes succeed, closes; if fails, opens again.

**Q3: What is the Bulkhead pattern and how does it complement Circuit Breaker?**
> Bulkhead isolates thread pools per downstream service. Without it, one slow service can exhaust the shared thread pool, starving other services. With Bulkhead, each service dependency has its own pool — ServiceA slowness only affects ServiceA's thread pool; ServiceB is unaffected. Circuit Breaker + Bulkhead = defense in depth.

**Q4: What's the difference between Circuit Breaker and Retry?**
> Retry handles transient failures (network blip, momentary overload) by trying again. Circuit Breaker handles systemic failures (service is down) by fast-failing. They're complementary: Retry handles occasional failures, Circuit Breaker handles when failures become persistent. Apply Retry INSIDE Circuit Breaker so retries stop when circuit opens.

**Q5: When should you NOT use a Circuit Breaker?**
> Synchronous database calls within the same service (latency is local — use connection pooling instead). Very low-traffic services where statistical thresholds can't be reliably measured. Write operations where retries could cause duplicates (use idempotency keys instead). Simple point-to-point calls where the service is very reliable.

---

## 🔗 What to Read Next

| Topic | Why You Need It |
|---|---|
| [Rate Limiting](./RateLimiting.md) | Rate Limiting protects YOUR service from being overloaded; Circuit Breaker protects you from overloading others |
| [Microservices/Failure Handling](../Microservices/Microservice_Failure_Situation_Handle.md) | Broader failure handling strategies beyond just Circuit Breaker |
| [Observability/Metrics Monitoring](../Observability/Metrics_Monitoring.md) | Circuit Breaker is useless without alerting when it trips — set up monitoring |

---

*[← Back to Building Blocks](../BuildingBlocks/) | [← Index](../INDEX.md)*
