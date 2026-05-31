# 🔌 Circuit Breaker Pattern in Microservices: Failing Fast, Failing Smart

> *"In electrical circuits, a breaker trips to prevent fire. In microservices, a circuit breaker trips to prevent cascading failure. Both save the whole system by sacrificing one connection."*

**⏱️ Estimated Time**: 20 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Bulkhead Pattern](./BulkheadPattern.md), [Failure Handling](./Microservice_Failure_Situation_Handle.md)

---

## 📋 Table of Contents
1. [The Problem: Cascading Failures](#-the-problem)
2. [How Circuit Breaker Works](#-how-it-works)
3. [States Explained](#-states)
4. [Implementation (Resilience4j + Spring Boot)](#-implementation)
5. [Configuration Strategy](#-configuration)
6. [Real-World Usage at Big Tech](#-big-tech-usage)
7. [Interview Q&A](#-interview-qa)
8. [Boss Battle](#-boss-battle)

---

## 💥 The Problem

```
WITHOUT CIRCUIT BREAKER:

  Service A → Service B (DOWN! timeout = 30s)
  
  Thread 1: waiting... waiting... 30s timeout ❌
  Thread 2: waiting... waiting... 30s timeout ❌
  Thread 3: waiting... waiting... 30s timeout ❌
  ... (200 threads all blocked waiting for B!)
  
  Result: Service A runs out of threads → Service A is now DOWN too!
  Result: Service C (which calls A) → also goes down!
  Result: ENTIRE SYSTEM CASCADE FAILURE from one service! 💀

WITH CIRCUIT BREAKER:

  Service A → [Circuit Breaker] → Service B (DOWN!)
  
  First 5 failures: "B seems down, let me count..."
  After 5 failures: CIRCUIT OPEN! 🔴
  
  Thread 6:  → Circuit Breaker → INSTANT FAIL (no waiting!) → Fallback
  Thread 7:  → Circuit Breaker → INSTANT FAIL → Fallback
  Thread 8:  → Circuit Breaker → INSTANT FAIL → Fallback
  
  Result: Fail in 1ms instead of 30s! A stays healthy! No cascade! ✅
  After 60s: Try B again (HALF-OPEN) → if B recovered, resume normal
```

---

## ⚙️ How It Works

```
STATE MACHINE:

         ┌────────────────────────────────────────────────────┐
         │                                                    │
         ▼                                                    │
    ┌─────────┐    failure rate     ┌──────────┐    timeout   │
    │ CLOSED  │ ───exceeds threshold──→│  OPEN  │────expires──→│
    │(normal) │                     │ (reject) │             │
    └─────────┘                     └──────────┘             │
         ▲                                                    │
         │           success                    ┌───────────┐ │
         └──────────────────────────────────────│HALF-OPEN  │─┘
                                                │(test call)│
                              failure           └───────────┘
                              ────────────────────────→ OPEN
```

---

## 🚦 States

| State | Behavior | Analogy |
|-------|----------|---------|
| **🟢 CLOSED** | All requests pass through normally | Highway open — drive freely |
| **🔴 OPEN** | All requests REJECTED instantly (no call to B) | Highway closed — detour immediately |
| **🟡 HALF-OPEN** | Limited test requests pass through | One lane open — testing if safe |

### Transitions

```
CLOSED → OPEN:
  When: Failure rate > threshold (e.g., >50% of last 10 calls failed)
  What happens: Stop sending requests, return fallback immediately
  Why: Protect from wasting resources on a dead service

OPEN → HALF-OPEN:
  When: Wait duration expires (e.g., 60 seconds since circuit opened)
  What happens: Allow N test requests through
  Why: Check if the failing service has recovered

HALF-OPEN → CLOSED:
  When: Test requests succeed (e.g., 3 out of 5 pass)
  What happens: Resume normal traffic
  Why: Service recovered! Back to business.

HALF-OPEN → OPEN:
  When: Test requests fail
  What happens: Go back to rejecting everything
  Why: Still broken, wait more before trying again
```

---

## ☕ Implementation

### Resilience4j (Modern, recommended for Spring Boot)

```java
// 1. Add dependency
// implementation 'io.github.resilience4j:resilience4j-spring-boot3:2.1.0'

// 2. Configuration (application.yml)
resilience4j:
  circuitbreaker:
    instances:
      paymentService:
        sliding-window-size: 10          # Evaluate last 10 calls
        failure-rate-threshold: 50       # Open if >50% fail
        wait-duration-in-open-state: 60s # Wait 60s before half-open
        permitted-number-of-calls-in-half-open-state: 3  # Test with 3 calls
        slow-call-rate-threshold: 80     # Also open if 80% are slow
        slow-call-duration-threshold: 2s # "Slow" = >2 seconds

// 3. Usage with annotation
@Service
public class OrderService {
    
    @CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
    public PaymentResult processPayment(Order order) {
        return paymentClient.charge(order.getAmount()); // May fail!
    }
    
    // Fallback when circuit is OPEN or call fails
    private PaymentResult paymentFallback(Order order, Throwable ex) {
        log.warn("Payment service unavailable, queuing order: {}", order.getId());
        paymentQueue.enqueue(order); // Queue for retry later
        return PaymentResult.pending("Payment will be processed shortly");
    }
}
```

### Combining with Retry + Timeout

```java
// The Resilience Stack (order matters!):
// Retry → CircuitBreaker → TimeLimiter → Bulkhead → Actual Call

@CircuitBreaker(name = "inventoryService", fallbackMethod = "inventoryFallback")
@Retry(name = "inventoryService")            // Retry 3 times BEFORE circuit counts failure
@TimeLimiter(name = "inventoryService")       // Timeout after 2s per call
@Bulkhead(name = "inventoryService")          // Max 10 concurrent calls
public CompletableFuture<Stock> checkStock(String productId) {
    return CompletableFuture.supplyAsync(
        () -> inventoryClient.getStock(productId)
    );
}

// Configuration
resilience4j:
  retry:
    instances:
      inventoryService:
        max-attempts: 3
        wait-duration: 500ms
        exponential-backoff-multiplier: 2  # 500ms, 1s, 2s
  timelimiter:
    instances:
      inventoryService:
        timeout-duration: 2s
  bulkhead:
    instances:
      inventoryService:
        max-concurrent-calls: 10
```

---

## 🎛️ Configuration

### How to Choose Settings

```
FAILURE RATE THRESHOLD:
  50% = balanced (recommended default)
  25% = aggressive (trip faster, more protective)
  80% = lenient (only trip when really broken)
  
SLIDING WINDOW SIZE:
  10 calls = responsive but may false-trip on brief blips
  100 calls = stable but slower to detect real failures
  
WAIT DURATION (OPEN STATE):
  10s = aggressive retry (good for transient issues)
  60s = conservative (good for service restarts taking ~30s)
  5min = very conservative (external dependencies with slow recovery)

RULE OF THUMB:
  - Internal services (same datacenter): 50% threshold, 10 window, 30s wait
  - External APIs (third-party): 25% threshold, 20 window, 60s wait
  - Critical path (payment): 30% threshold, 5 window, 10s wait (fail fast!)
```

---

## 🏢 Big Tech Usage

| Company | Implementation | Notable Approach |
|---------|---------------|------------------|
| **Netflix** | Hystrix (now deprecated → Resilience4j) | Pioneered the pattern! |
| **Amazon** | Custom + AWS ALB circuit breaking | Per-endpoint, per-AZ |
| **Uber** | Custom (Go-based) | Per-method granularity |
| **Spotify** | Envoy proxy (service mesh level) | Infrastructure-level, not code |
| **Google** | gRPC built-in + Envoy | Automatic hedging + circuit break |

```
NETFLIX'S CIRCUIT BREAKER PHILOSOPHY:
  "Everything will fail. The question is: will your system 
   handle it gracefully or catastrophically?"
   
  They circuit-break EVERY external dependency:
  - Other microservices (1000+ services)
  - Databases (separate breaker per DB)
  - Caches (Redis failures shouldn't kill the app)
  - Third-party APIs (CDN, payment providers)
```

---

## 🎓 Interview Q&A

### Q1: "Explain the circuit breaker pattern"
**A**: It's a proxy that monitors call failures to a dependency. After a threshold of failures (e.g., 50% of last 10 calls), it "opens" and immediately rejects requests without calling the failing service — preventing resource exhaustion and cascading failures. After a timeout, it allows test requests (half-open) to check if the service recovered.

### Q2: "Circuit breaker vs retry — what's the difference?"
**A**: Retry attempts the same call again (for transient failures like network blips). Circuit breaker STOPS calling entirely (for sustained failures like a service being down). They complement each other: retry first (3 attempts), and if all retries fail, that counts as ONE failure toward the circuit breaker threshold.

### Q3: "Where should the circuit breaker live — in code or infrastructure?"
**A**: Both are valid. In code (Resilience4j): more control, custom fallbacks, business-aware. In infrastructure (service mesh/Envoy): automatic for all services, no code changes, but less customizable fallbacks. Many companies use infrastructure-level breaking for basic protection + code-level for critical paths with custom fallbacks.

---

## 🎲 Boss Battle: Design the Resilience Layer 🛡️

> **Scenario**: Your e-commerce checkout calls 4 services:
> 1. Inventory Service (check stock)
> 2. Payment Service (charge card)
> 3. Notification Service (send email)
> 4. Recommendation Service (suggest add-ons)
>
> If any is slow/down, checkout shouldn't fail completely.
> Design the circuit breaker + fallback strategy for each.

<details>
<summary>🔓 Click to reveal answer</summary>

| Service | Criticality | CB Config | Fallback | Why |
|---------|-------------|-----------|----------|-----|
| **Inventory** | CRITICAL | 50% / 5 calls / 10s wait | Return "available" with async verification | Can't checkout without stock check, but optimistic OK |
| **Payment** | CRITICAL | 30% / 5 calls / 5s wait | Queue payment for retry, show "processing" | Must charge eventually, but don't block the user |
| **Notification** | LOW | 70% / 20 calls / 120s wait | Skip silently, retry from queue later | User doesn't need email instantly |
| **Recommendation** | ZERO | 80% / 10 calls / 300s wait | Show static "popular items" list | Nice-to-have, never block checkout! |

**Design Principles:**
1. **Payment**: Most aggressive breaker (fail fast! money is critical)
2. **Recommendation**: Most lenient (who cares if it's slow? hide it)
3. **Notification**: Fire-and-forget via queue (guaranteed delivery, not speed)
4. **Inventory**: Optimistic fallback + async reconciliation

**The checkout ALWAYS succeeds** from the user's perspective — even if 3 of 4 dependencies are down!
</details>

---

## 🏆 Achievement Unlocked!

```
┌──────────────────────────────────────────────────────────────┐
│  🛡️ ACHIEVEMENT: Resilience Engineer Level 1                 │
│                                                              │
│  You now understand:                                         │
│  ✅ Why cascading failures happen                            │
│  ✅ Circuit breaker states (Closed → Open → Half-Open)       │
│  ✅ How to implement with Resilience4j                       │
│  ✅ How to combine with retry, timeout, bulkhead             │
│  ✅ How big tech companies use it                             │
│                                                              │
│  NEXT: → Service Mesh (circuit breaking at infrastructure)   │
└──────────────────────────────────────────────────────────────┘
```

👉 **[Next: Service Mesh →](./ServiceMesh.md)**  
👉 **[Related: Bulkhead Pattern →](./BulkheadPattern.md)**  
👉 **[Back to Microservices Overview →](./README.md)**
