# 🧪 Performance Testing & SRE Practices: Proving It at Scale

> *"Hope is not a strategy. If you haven't load tested it, you don't know if it works under load — you just BELIEVE it does. And production will prove you wrong at 3 AM."*

**⏱️ Estimated Time**: 25 minutes | **🎯 Difficulty**: 🔴 Advanced | **🔗 Prerequisites**: [Metrics & SLOs](./Metrics_and_SLOs.md)

---

## 📋 Table of Contents
1. [Types of Performance Tests](#-types-of-performance-tests)
2. [Load Testing with JMeter & Gatling](#-load-testing-tools)
3. [Chaos Engineering](#-chaos-engineering)
4. [SRE Practices for Performance](#-sre-practices)
5. [Performance in CI/CD](#-performance-in-cicd)
6. [Incident Response for Performance](#-incident-response)
7. [Interview Q&A](#-interview-qa)
8. [Boss Battle](#-boss-battle)

---

## 🧪 Types of Performance Tests

```
┌──────────────────────────────────────────────────────────────────────┐
│ TEST TYPE       │ GOAL                │ PATTERN              │ WHEN   │
├─────────────────┼─────────────────────┼──────────────────────┼────────┤
│ Load Test       │ Can it handle       │ Ramp to expected     │ Every  │
│                 │ expected traffic?   │ peak, sustain 1hr    │ release│
├─────────────────┼─────────────────────┼──────────────────────┼────────┤
│ Stress Test     │ Where does it       │ Increase until       │ Quart- │
│                 │ break?              │ failure              │ erly   │
├─────────────────┼─────────────────────┼──────────────────────┼────────┤
│ Spike Test      │ Can it handle       │ 0 → max → 0         │ Before │
│                 │ sudden bursts?      │ in seconds           │ events │
├─────────────────┼─────────────────────┼──────────────────────┼────────┤
│ Soak Test       │ Memory leaks?       │ Moderate load for    │ Monthly│
│ (Endurance)     │ Resource exhaustion?│ 24-72 hours          │        │
├─────────────────┼─────────────────────┼──────────────────────┼────────┤
│ Capacity Test   │ How much can it     │ Find max throughput  │ Plann- │
│                 │ handle per $?       │ at acceptable latency│ ing    │
├─────────────────┼─────────────────────┼──────────────────────┼────────┤
│ Benchmark       │ Compare before/     │ Fixed workload,      │ Every  │
│                 │ after changes       │ measure delta        │ PR     │
└─────────────────┴─────────────────────┴──────────────────────┴────────┘

REAL STORIES:
  Amazon: 100ms latency → 1% revenue loss
  Google: 500ms slower → 20% fewer searches
  Twitter: Load tested before Super Bowl (10x normal traffic spike)
```

---

## 🔧 Load Testing Tools

### Gatling (Recommended for Java developers)

```scala
// Gatling simulation: 100 users ramping up over 60s
class ProductSearchSimulation extends Simulation {
  
  val httpProtocol = http
    .baseUrl("https://api.myapp.com")
    .acceptHeader("application/json")
    .shareConnections  // Realistic connection pooling

  val searchScenario = scenario("Product Search")
    .exec(
      http("Search Products")
        .get("/api/products?q=laptop")
        .check(status.is(200))
        .check(responseTimeInMillis.lt(500))  // Assert < 500ms!
    )
    .pause(1, 3)  // Think time: 1-3 seconds between requests
    .exec(
      http("Get Product Details")
        .get("/api/products/${productId}")
        .check(status.is(200))
    )

  setUp(
    searchScenario.inject(
      rampUsers(100).during(60.seconds),  // Ramp up
      constantUsersPerSec(50).during(300.seconds)  // Sustain
    )
  ).protocols(httpProtocol)
   .assertions(
      global.responseTime.percentile(95).lt(500),  // p95 < 500ms
      global.successfulRequests.percent.gt(99.0)   // >99% success
   )
}
```

### k6 (Modern, developer-friendly)

```javascript
// k6 load test script
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '1m', target: 100 },   // Ramp up to 100 users
    { duration: '5m', target: 100 },   // Stay at 100 for 5 min
    { duration: '1m', target: 0 },     // Ramp down
  ],
  thresholds: {
    http_req_duration: ['p(95)<500'],   // p95 < 500ms
    http_req_failed: ['rate<0.01'],     // <1% errors
  },
};

export default function () {
  const res = http.get('https://api.myapp.com/products');
  check(res, { 'status 200': (r) => r.status === 200 });
  sleep(1);
}
```

---

## 🔥 Chaos Engineering

```
"EVERYTHING FAILS, ALL THE TIME" — Werner Vogels (AWS CTO)

CHAOS ENGINEERING = Intentionally break things to prove resilience!

EXPERIMENTS:
  1. Kill a service instance → Does load balancer redirect?
  2. Inject 500ms latency → Do timeouts + circuit breakers work?
  3. Fill disk to 95% → Does alerting fire? Does app gracefully degrade?
  4. Kill a database replica → Does failover happen automatically?
  5. Simulate network partition → Do retries + fallbacks work?
  
TOOLS:
  - Chaos Monkey (Netflix) → randomly kills instances
  - Litmus Chaos (Kubernetes) → pod/node/network chaos
  - Gremlin → enterprise chaos platform
  - Toxiproxy → inject latency/errors in network
  - tc (Linux) → network delay/packet loss at OS level

THE PROCESS:
  1. Define steady state (SLOs are met)
  2. Hypothesize: "If X fails, system still meets SLOs"
  3. Introduce failure (in staging first!)
  4. Observe: Did hypothesis hold?
  5. If not: fix, add resilience, re-test
```

---

## 📈 SRE Practices

### Error Budgets

```
ERROR BUDGET = How much unreliability you can afford

SLO = 99.9% availability (Three nines)
Error Budget = 0.1% = 43 minutes of downtime per month

WHAT THIS MEANS:
  Budget remaining > 0: Ship fast! Take risks! Move features!
  Budget remaining = 0: FREEZE! Fix reliability before new features!
  
  This aligns engineering velocity with reliability.

ERROR BUDGET POLICY (Google-style):
  ┌─────────────────────────────────────────────────┐
  │ Budget Status    │ Actions                       │
  ├──────────────────┼───────────────────────────────┤
  │ > 50% remaining  │ Ship features normally        │
  │ 20-50% remaining │ Add extra testing/review      │
  │ 0-20% remaining  │ Only reliability work         │
  │ 0% (exhausted)   │ Feature freeze! Fix everything│
  └──────────────────┴───────────────────────────────┘
```

### On-Call & Runbooks

```
RUNBOOK TEMPLATE FOR PERFORMANCE INCIDENTS:

## High Latency Alert (p99 > SLO)

### 1. ASSESS (first 2 minutes)
  - Check: which endpoint? All or specific?
  - Check: traffic spike? (Grafana → request rate)
  - Check: recent deployment? (last deploy time)

### 2. DIAGNOSE (next 5 minutes)
  - Dashboard: CPU, memory, GC, thread count
  - Dependencies: are downstream services slow?
  - Database: slow queries? connection pool exhausted?
  - Look at: thread dumps, recent error logs

### 3. MITIGATE (immediate actions)
  - Scale horizontally (add instances) if CPU-bound
  - Rollback last deployment if correlates
  - Enable circuit breaker for slow dependency
  - Shed load (rate limit) if overwhelmed
  
### 4. RESOLVE (root cause fix)
  - Fix the actual bug/query/config
  - Add test to prevent regression
  - Update this runbook with learnings
```

---

## 🔄 Performance in CI/CD

```
SHIFT-LEFT PERFORMANCE: Catch issues BEFORE production!

┌─────────────────────────────────────────────────────────────────────┐
│  PR Check:                                                          │
│    ✅ Microbenchmark (JMH): "This method got 20% slower"           │
│    ✅ Query analysis: "New query missing index"                     │
│                                                                     │
│  Staging Deploy:                                                    │
│    ✅ Load test (Gatling): "p95 < 500ms at 1000 RPS"              │
│    ✅ Regression check: "Latency didn't increase vs main branch"   │
│                                                                     │
│  Canary Deploy (1% traffic):                                       │
│    ✅ Real user metrics: "Canary latency ≈ stable latency"         │
│    ✅ Error rate: "Canary errors ≤ stable errors"                   │
│    🚨 Auto-rollback if SLO violated!                                │
│                                                                     │
│  Production (100% traffic):                                         │
│    ✅ Continuous SLO monitoring                                      │
│    ✅ Alerting on error budget burn rate                             │
│    ✅ Weekly performance review                                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 🚨 Incident Response

### The OODA Loop for Performance Incidents

```
O — OBSERVE: What metrics are anomalous?
O — ORIENT:  Is this a known pattern? What changed recently?
D — DECIDE:  Mitigate now (scale/rollback) or investigate first?
A — ACT:     Execute the decision, communicate status

TIMELINE OF A GOOD INCIDENT RESPONSE:
  T+0:    Alert fires (PagerDuty/OpsGenie)
  T+2min: On-call acknowledges, opens incident channel
  T+5min: Initial assessment shared: "p99 latency 3x SLO"
  T+10min: Root cause identified: "New query missing index"
  T+15min: Fix applied (add index / rollback deploy)
  T+20min: Metrics recovering, confirm SLO restored
  T+30min: Incident closed, postmortem scheduled

WORST ANTI-PATTERNS:
  ❌ "Let's wait and see" (while users suffer)
  ❌ Debugging in production without communicating
  ❌ Skipping the postmortem ("it won't happen again")
```

---

## 🎓 Interview Q&A

### Q1: "How do you prevent performance regressions?"
**A**: Performance gates in CI/CD: automated load tests on staging, benchmark comparisons in PRs (JMH), canary deployments with automated rollback, SLO-based alerting. The key is catching issues before they reach 100% of users.

### Q2: "What's your approach to chaos engineering?"
**A**: Start in staging with one hypothesis (e.g., "if Service A goes down, the fallback serves cached data"). Define steady state via SLOs, inject failure, observe. Only promote to production after staging confidence. Tools: Chaos Monkey, Litmus, Gremlin. Always have a kill switch.

### Q3: "Describe your ideal performance testing strategy for a new feature."
**A**: 1) Benchmark critical code paths (JMH) during development. 2) Load test in staging at 2x expected peak before release. 3) Canary to 1% of traffic, compare metrics. 4) Gradual rollout with auto-rollback if SLO degrades. 5) Monitor error budget burn rate for 7 days post-launch.

---

## 🎲 Boss Battle: Black Friday Preparation 🛒

> **Scenario**: You're the performance lead for an e-commerce site expecting 10x normal traffic on Black Friday (in 3 weeks). Current peak is 5,000 RPS; expecting 50,000 RPS.
>
> **Challenge**: Create a preparation plan covering testing, scaling, and incident readiness.
>
> <details>
> <summary>🔓 Click to reveal answer</summary>
>
> **Week 1: Assess & Test**
> - Run load test at 50K RPS in staging → identify bottlenecks
> - Profile hot paths: product search, cart, checkout, payment
> - Capacity test: find actual breaking point (maybe 30K RPS today)
> - Database: identify slow queries at scale, add indexes
>
> **Week 2: Optimize & Scale**
> - Scale bottlenecks: add Redis caching for product catalog (cache-aside)
> - Pre-warm caches: load popular products before event
> - Auto-scaling rules: scale at 60% CPU (not default 80%)
> - Circuit breakers: timeout on non-critical services (recommendations)
> - Rate limiting: protect checkout/payment APIs
> - CDN: cache static assets and product images aggressively
>
> **Week 3: Validate & Prepare**
> - Full dress rehearsal: 50K RPS load test + chaos (kill nodes mid-test)
> - Runbooks: updated for every failure scenario
> - War room: scheduled on-call rotation for the event
> - Kill switches: feature flags to disable non-critical features
> - Communication plan: status page, Slack channel, escalation path
>
> **On The Day:**
> - Monitor error budget burn rate
> - Auto-scale + manual override ready
> - Degrade gracefully: disable recommendations if overloaded
> - Celebrate when it works! 🎉
>
> **Key Principle**: Test at 2x expected peak. If you can handle 100K RPS in staging, 50K in production is comfortable.
> </details>

---

## 🏆 Achievement Unlocked!

```
┌──────────────────────────────────────────────────────────────┐
│  🏗️ ACHIEVEMENT: Performance Engineering Master             │
│                                                              │
│  You've completed all Performance topics!                    │
│  ✅ Metrics & SLOs       ✅ Profiling & Optimization        │
│  ✅ Database Performance  ✅ JVM Tuning                      │
│  ✅ Memory Management     ✅ Benchmarking                    │
│  ✅ Caching Performance   ✅ Concurrency Performance         │
│  ✅ Network & I/O         ✅ Performance Testing & SRE       │
│                                                              │
│  NEXT: → Microservices (apply performance at scale!)         │
└──────────────────────────────────────────────────────────────┘
```

👉 **[Back to Performance Overview →](./README.md)**  
👉 **[Next Journey: Microservices →](../Microservices/README.md)**
