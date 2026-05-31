# 📈 Performance Testing: JMeter, Gatling & Load Testing Strategies

> *"'It works on my machine with 1 user' means nothing. Performance testing answers: What happens with 10,000 concurrent users? Where does the system break? Can we handle Black Friday traffic?'"*

**⏱️ Estimated Time**: 25 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Testing Pyramid](./Testing_Pyramid.md)

---

## 📋 Types of Performance Tests

```
┌────────────────────┬────────────────────────────────────────────────────┐
│ Type               │ Purpose                                            │
├────────────────────┼────────────────────────────────────────────────────┤
│ Load Test          │ Can system handle EXPECTED peak load?              │
│ Stress Test        │ What happens BEYOND expected load? Where breaks?   │
│ Spike Test         │ Sudden traffic surge (flash sale, viral tweet)     │
│ Soak/Endurance     │ Run for hours — memory leaks? resource exhaustion? │
│ Scalability Test   │ Does adding resources improve performance linearly?│
│ Breakpoint Test    │ At what load does system fail?                     │
└────────────────────┴────────────────────────────────────────────────────┘

LOAD PATTERNS:
  Load test:    ──────────────────── (steady at expected peak)
  Stress test:  ──────╱╱╱╱╱╱╱╱╱╱╱╱╱ (increasing until failure)
  Spike test:   ──────│████│──────── (sudden burst, then back down)
  Soak test:    ──────────────────── (steady for 8-24 hours)
```

---

## 🛠️ Tools Comparison

```
┌────────────────────┬────────────────┬─────────────────┬──────────────────┐
│                    │ JMeter         │ Gatling         │ k6               │
├────────────────────┼────────────────┼─────────────────┼──────────────────┤
│ Language           │ XML/GUI        │ Scala/Java      │ JavaScript       │
│ Protocol           │ HTTP, JDBC,    │ HTTP, WebSocket │ HTTP, WebSocket, │
│                    │ JMS, LDAP...   │                 │ gRPC             │
│ Scripting          │ GUI + Groovy   │ Code-first (DSL)│ Code-first (JS)  │
│ Reports            │ Built-in       │ Beautiful HTML  │ Cloud dashboard  │
│ CI/CD integration  │ Maven plugin   │ Maven/Gradle    │ CLI + cloud      │
│ Resource usage     │ Heavy (Java)   │ Efficient (Akka)│ Very light (Go)  │
│ Learning curve     │ Low (GUI)      │ Medium          │ Low              │
│ Best for           │ Enterprise     │ Java/Scala teams│ DevOps, modern   │
│ Max users/machine  │ ~5,000         │ ~30,000         │ ~30,000+         │
└────────────────────┴────────────────┴─────────────────┴──────────────────┘
```

---

## 💻 Gatling Example (Java DSL)

```java
public class OrderLoadSimulation extends Simulation {
    
    HttpProtocolBuilder httpProtocol = http
        .baseUrl("https://staging-api.myapp.com")
        .acceptHeader("application/json")
        .contentTypeHeader("application/json");
    
    // Scenario: User browses products, adds to cart, checks out
    ScenarioBuilder purchaseFlow = scenario("Purchase Flow")
        .exec(http("Get Products")
            .get("/api/products")
            .check(status().is(200))
            .check(jsonPath("$[0].id").saveAs("productId")))
        .pause(2, 5)  // think time: 2-5 seconds
        .exec(http("Add to Cart")
            .post("/api/cart/items")
            .body(StringBody("{\"productId\": \"#{productId}\", \"quantity\": 1}"))
            .check(status().is(201)))
        .pause(1, 3)
        .exec(http("Checkout")
            .post("/api/orders")
            .body(StringBody("{\"paymentMethod\": \"card_test\"}"))
            .check(status().is(201))
            .check(responseTimeInMillis().lt(2000)));  // assert < 2s
    
    // Load profile
    {
        setUp(
            purchaseFlow.injectOpen(
                // Ramp up to 100 users over 1 minute
                rampUsers(100).during(Duration.ofMinutes(1)),
                // Stay at 100 users for 5 minutes
                constantUsersPerSec(20).during(Duration.ofMinutes(5)),
                // Spike: 500 users in 10 seconds
                rampUsers(500).during(Duration.ofSeconds(10))
            )
        ).protocols(httpProtocol)
         .assertions(
             global().responseTime().percentile3().lt(2000),  // p95 < 2s
             global().successfulRequests().percent().gt(99.0), // >99% success
             global().requestsPerSec().gt(100.0)              // >100 RPS
         );
    }
}
```

---

## 📊 Key Metrics to Measure

```
LATENCY (response time):
  p50 (median): 50% of requests faster than this
  p95:          95% of requests faster (good SLA target)
  p99:          99% of requests faster (tail latency)
  
  Example target:
    p50 < 200ms, p95 < 500ms, p99 < 2000ms

THROUGHPUT:
  Requests per second (RPS): how many requests system handles
  Transactions per second (TPS): completed business transactions

ERROR RATE:
  % of requests that return 4xx/5xx
  Target: < 0.1% under normal load, < 1% under stress

RESOURCE UTILIZATION:
  CPU: should stay below 70% at peak (headroom for spikes)
  Memory: watch for steady growth (memory leak indicator)
  Network: bandwidth saturation?
  DB connections: pool exhaustion?
  
SATURATION POINT:
  The load level where adding more users DEGRADES performance
  
  RPS ↑                    ┌─── saturation (system degrading)
       │               ╱───┘
       │          ╱───╱
       │     ╱───╱
       │╱───╱
       └────────────────── Concurrent Users →
```

---

## ⚠️ Common Pitfalls

1. **Testing against production** — Load testing production can cause outages for real users. Always test against a staging environment that mirrors production (same DB size, same instance types). Use traffic replay tools for realistic patterns.

2. **Not warming up** — JVM apps need warm-up (JIT compilation, connection pool filling). Start load tests with a gradual ramp-up (1-5 min), then measure steady-state. Ignore first few minutes in results.

3. **Unrealistic test data** — Testing with 100 products when production has 1M changes query performance completely. Seed staging with production-scale data. Use realistic user behavior patterns.

4. **Only testing happy path** — Real traffic includes errors, retries, timeouts. Include: invalid requests (5-10%), slow consumers, connection drops.

---

## 📝 Interview Q&A

**Q: How would you performance test a new microservice before production?**
> A: (1) **Define SLOs**: target latency (p95 < 500ms), throughput (1000 RPS), error rate (<0.1%). (2) **Baseline**: single-user test to establish minimum latency. (3) **Load test**: ramp to expected peak (from traffic estimates/analytics). Verify SLOs are met. (4) **Stress test**: push beyond expected peak (2x, 5x) to find breaking point. Know your limits. (5) **Soak test**: run at expected load for 8+ hours — catch memory leaks, connection pool exhaustion. (6) **Profile bottlenecks**: when SLOs are missed, profile (CPU? DB queries? external service?). Fix and re-test. (7) **Automate**: run performance tests in CI (nightly or pre-release) with automatic SLO assertions.

---

## 🏢 Industry Performance Testing Practices

| Company | Tool / Approach | Scale | Key Insight |
|---------|----------------|-------|-------------|
| Netflix | ChAOS + JMeter | 100M+ users | Tests resilience under degraded conditions, not just peak load |
| Amazon | Internal tooling | Prime Day: 300K TPS | Separate perf tests per microservice; automated regression alerts |
| Google | Borg-level load generation | Search: millions QPS | Every major service has latency SLOs enforced in CI |
| Stripe | Continuous load testing | 100K+ TPS | Canary deploys with real-time latency comparison |
| LinkedIn | Gatling + custom tooling | 1B+ members | Introduced "latency budgets" per API |

---

## 🎲 Mini Challenge

> 🎲 **CHALLENGE** (5 minutes):
> You deploy a new version of your search service. After deployment, p99 latency
> jumps from 200ms to 800ms but p50 latency stays the same (50ms).
> What does this tell you, and what would you investigate first?

<details>
<summary>💡 Click to reveal answer</summary>

**What it tells you:**
- p50 unchanged → the median case is fine (most requests are OK)
- p99 jumped 4x → ~1% of requests are very slow (tail latency problem)

**Root causes to investigate (in order):**
1. **GC pauses** — Java STW GC pauses affect ~1% of requests. Profile heap allocation. Look for large object creation, G1GC tuning.
2. **Database query plans** — A new code path may hit a slow query only for certain data patterns. Check slow query log for queries >200ms.
3. **Lock contention** — New code might contend with existing concurrent operations. Thread dump during slow period.
4. **External service timeout** — If your service calls another service, tail latency from that service adds to yours. Trace with distributed tracing (Zipkin/Jaeger).
5. **Cold cache paths** — New feature may have a cache miss path that's slow. Check cache hit rate metrics.

**Investigation tool**: Flame graph profiling (async-profiler for Java) + distributed tracing + percentile latency breakdown by endpoint.

</details>

---

## 🔗 What to Read Next

1. **[Testing/TDD_BDD.md](./TDD_BDD.md)** — Test-driven development
2. **[Performance/Profiling_and_Optimization.md](../Performance/Profiling_and_Optimization.md)** — Profiling bottlenecks
3. **[Observability/Metrics_Monitoring.md](../Observability/Metrics_Monitoring.md)** — Monitoring in production

---

*[← E2E Testing](./E2E_Testing.md) | [Back to Index](../INDEX.md) | [Next: TDD & BDD →](./TDD_BDD.md)*
