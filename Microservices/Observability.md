# 🔭 Observability in Microservices: Seeing Through the Distributed Fog

> *"In a monolith, you read ONE log file. In microservices with 100 services, a single user request generates logs across 15 services. Without observability, debugging is like finding a needle in 15 haystacks — blindfolded."*

**⏱️ Estimated Time**: 25 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Distributed System](./DistributedSystem.md)

---

## 📋 Table of Contents
1. [Three Pillars of Observability](#-three-pillars)
2. [Distributed Tracing](#-distributed-tracing)
3. [Centralized Logging](#-centralized-logging)
4. [Metrics & Dashboards](#-metrics--dashboards)
5. [Implementation (Spring Boot)](#-implementation)
6. [How Big Tech Does It](#-big-tech-observability)
7. [Interview Q&A](#-interview-qa)
8. [Boss Battle](#-boss-battle)

---

## 🏛️ Three Pillars

```
THE THREE PILLARS OF OBSERVABILITY:

┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  📊 METRICS        📝 LOGS           🔗 TRACES                 │
│  "What's broken?"  "Why is it broken?" "Where is it broken?"   │
│                                                                 │
│  Numeric time-     Detailed text       Request flow across      │
│  series data       events & errors     multiple services        │
│                                                                 │
│  Examples:         Examples:            Example:                 │
│  - Request rate    - Error stacktrace   - User→Gateway→Auth     │
│  - Error %         - Request payload      →Product→DB           │
│  - p99 latency     - Debug details      - 45ms total, 30ms DB  │
│                                                                 │
│  Tools:            Tools:              Tools:                    │
│  Prometheus        ELK Stack           Jaeger                   │
│  Grafana           Loki                Zipkin                   │
│  Datadog           Datadog             OpenTelemetry            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

RELATIONSHIP:
  1. Alert fires (METRICS): "p99 latency > 500ms!"
  2. Where? (TRACES): "It's slow in the Payment → Fraud service call"
  3. Why? (LOGS): "Fraud service: DB connection pool exhausted"
```

---

## 🔗 Distributed Tracing

```
THE PROBLEM:
  User clicks "Place Order" → request touches 8 services → 3 seconds later, fails
  WHICH service caused the failure? WHERE was the bottleneck?

THE SOLUTION: TRACES
  Assign a unique Trace ID to the request at entry point.
  Every service passes this ID along. Every span records timing.

  Trace ID: abc-123
  ├── Span 1: API Gateway (2ms)
  │   ├── Span 2: Auth Service (15ms)
  │   └── Span 3: Order Service (2800ms) ← BOTTLENECK!
  │       ├── Span 4: Inventory Service (50ms) ✅
  │       ├── Span 5: Payment Service (200ms) ✅
  │       └── Span 6: Fraud Detection (2500ms) ← ROOT CAUSE!
  │           └── Span 7: ML Model inference (2400ms) 💀
  └── Total: 3017ms

  NOW YOU KNOW: Fraud Detection's ML model is the bottleneck!
```

### Key Concepts

| Term | Definition | Example |
|------|-----------|---------|
| **Trace** | Complete journey of one request | User's checkout flow |
| **Span** | One operation within a trace | "Query database" (30ms) |
| **Trace ID** | Unique ID for the entire request | `abc-123-def-456` |
| **Span ID** | Unique ID for one operation | `span-789` |
| **Parent Span** | The span that triggered this one | Order → Payment |
| **Baggage** | Key-value pairs propagated across services | `user-id=42` |

---

## 📝 Centralized Logging

```
THE PROBLEM:
  100 services × 3 instances each = 300 log files
  Where do you look when something fails?

THE SOLUTION: Ship ALL logs to ONE place!

  ┌─────────────────────────────────────────────────────────┐
  │ Service A ──┐                                           │
  │ Service B ──┤──→ [Log Shipper] ──→ [Log Store] ──→ UI  │
  │ Service C ──┤    (Fluentd/      (Elasticsearch/   (Kibana/│
  │ Service D ──┘     Filebeat)      Loki/CloudWatch)  Grafana)│
  └─────────────────────────────────────────────────────────┘
  
  Search: "Show me all logs with traceId=abc-123"
  Result: Complete story across ALL services for that request!
```

### Structured Logging Best Practices

```java
// ❌ BAD: Unstructured (can't search/filter!)
log.error("Payment failed for user");

// ✅ GOOD: Structured with context
log.error("Payment failed", 
    Map.of(
        "userId", userId,
        "orderId", orderId,
        "amount", amount,
        "errorCode", "INSUFFICIENT_FUNDS",
        "traceId", MDC.get("traceId")  // Correlate with trace!
    ));

// Output (JSON format → searchable!):
// {"timestamp":"2024-01-15T10:30:00Z","level":"ERROR",
//  "message":"Payment failed","userId":"42","orderId":"ORDER-789",
//  "amount":99.99,"errorCode":"INSUFFICIENT_FUNDS",
//  "traceId":"abc-123","service":"payment-service"}
```

### Log Levels Strategy

```
TRACE  → Development only (method entry/exit)
DEBUG  → Development/staging (variable values)
INFO   → Normal operations (request received, completed)  ← Production baseline
WARN   → Degraded but working (retry succeeded, slow response)
ERROR  → Failed operation (exception, timeout, business rule violation)
FATAL  → System cannot continue (out of memory, config missing)

PRODUCTION: INFO and above (WARN, ERROR, FATAL)
DEBUGGING AN ISSUE: Temporarily enable DEBUG for specific service
```

---

## 📊 Metrics & Dashboards

### RED Method (for services)

```
R — Rate:     Requests per second (is traffic normal?)
E — Errors:   Error rate (are things failing?)
D — Duration: Latency percentiles (are things slow?)

EXAMPLE DASHBOARD:
  ┌────────────────────────────────────────────────────────────┐
  │ 📊 Order Service Dashboard                                 │
  ├────────────────────────────────────────────────────────────┤
  │ Request Rate: ████████████████ 1,250 RPS (↑ 5%)           │
  │ Error Rate:   ██ 0.3% (✅ within SLO)                     │
  │ p50 Latency:  ████ 45ms                                    │
  │ p95 Latency:  ████████ 120ms                               │
  │ p99 Latency:  ████████████████████ 480ms (⚠️ near SLO)    │
  └────────────────────────────────────────────────────────────┘
```

---

## ☕ Implementation

### OpenTelemetry + Spring Boot 3

```java
// 1. Dependencies (Spring Boot 3.2+)
// spring-boot-starter-actuator
// micrometer-tracing-bridge-otel
// opentelemetry-exporter-otlp

// 2. Configuration (application.yml)
management:
  tracing:
    sampling:
      probability: 1.0  # 100% in dev, 10% in prod (cost!)
  otlp:
    tracing:
      endpoint: http://jaeger:4318/v1/traces

// 3. Automatic instrumentation — IT JUST WORKS!
// Spring Boot auto-instruments:
//   - HTTP requests (incoming + outgoing via RestTemplate/WebClient)
//   - Database queries (JDBC)
//   - Kafka messages
//   - Scheduled tasks

// 4. Custom spans for important business logic
@Service
public class FraudDetectionService {
    
    private final Tracer tracer;
    
    public FraudResult checkFraud(Transaction tx) {
        Span span = tracer.nextSpan().name("fraud-ml-inference").start();
        try (Tracer.SpanInScope ws = tracer.withSpan(span)) {
            span.tag("transaction.amount", String.valueOf(tx.getAmount()));
            span.tag("transaction.country", tx.getCountry());
            
            FraudResult result = mlModel.predict(tx); // Expensive!
            
            span.tag("fraud.score", String.valueOf(result.getScore()));
            return result;
        } finally {
            span.end();
        }
    }
}
```

### Correlation: Tying It All Together

```java
// The MAGIC: Trace ID in every log line!
// Spring Boot + Micrometer does this automatically via MDC

// Log output automatically includes traceId + spanId:
// 2024-01-15 10:30:00 INFO [order-service,abc123,span456] Processing order

// Now in Kibana/Loki, search: traceId="abc123"
// → See ALL logs from ALL services for this one request!
```

---

## 🏢 Big Tech Observability

| Company | Tracing | Logging | Metrics | Special Sauce |
|---------|---------|---------|---------|---------------|
| **Google** | Dapper (internal) → OpenTelemetry | Stackdriver | Monarch | Invented distributed tracing! |
| **Uber** | Jaeger (open-sourced by Uber!) | ELK | M3 (custom) | Created Jaeger for their 4000+ services |
| **Netflix** | Zipkin (contributed to) | Atlas + Kibana | Atlas | Pioneered chaos + observability combo |
| **Amazon** | X-Ray | CloudWatch | CloudWatch | Deep AWS integration |
| **Spotify** | OpenTelemetry | GCP Logging | Prometheus | Cloud-native approach |

---

## 🎓 Interview Q&A

### Q1: "How do you debug a slow request in a microservices system?"
**A**: 1) Check metrics dashboard — which service has elevated latency? 2) Find the trace for the slow request using trace ID from the gateway. 3) The trace shows exactly which span (service call) is slow. 4) Look at logs for that service/trace ID to find the root cause (slow query, downstream timeout, etc.).

### Q2: "What's the difference between monitoring and observability?"
**A**: Monitoring tells you WHEN something is wrong (alerts on known metrics). Observability lets you ask arbitrary questions about WHY it's wrong — even for problems you didn't predict. You need all three pillars (metrics for detection, traces for localization, logs for explanation) to achieve true observability.

### Q3: "How do you handle the cost of distributed tracing at scale?"
**A**: Sampling! In production, you don't trace 100% of requests. Strategies: 1) Head-based sampling: randomly trace 10% of requests. 2) Tail-based sampling: decide AFTER the request (always keep errors and slow requests, sample normal ones). 3) Adaptive sampling: increase rate during incidents. Cost vs visibility trade-off.

---

## 🎲 Boss Battle: Build the Observability Stack 🔭

> **Scenario**: You have 50 microservices, 3 environments (dev/staging/prod), 10M requests/day in production. Design the observability strategy.
>
> **Requirements:**
> - Detect issues within 2 minutes
> - Debug root cause within 15 minutes
> - Cost-effective (not trace everything)
> - Developers can self-serve (no SRE bottleneck)

<details>
<summary>🔓 Click to reveal answer</summary>

**Architecture:**
```
Services → OpenTelemetry Collector → Fan out:
  ├── Traces → Jaeger (10% sampling, 100% for errors)
  ├── Metrics → Prometheus → Grafana
  └── Logs → Loki → Grafana
```

**Sampling Strategy:**
- Dev: 100% traces (low volume)
- Staging: 50% traces  
- Prod: 10% head-based + 100% tail-based for errors/slow (>2s)
- Always trace: payment paths, error responses, SLO-breaching requests

**Alerting Layers:**
1. **P1 (page on-call)**: Error rate > 1% OR p99 > SLO for 5 minutes
2. **P2 (Slack alert)**: Error budget burn rate high
3. **P3 (ticket)**: Gradual performance degradation trend

**Self-Service for Devs:**
- Grafana dashboards per service (auto-generated from labels)
- "Trace Explorer" UI: paste request ID → see full trace
- Log search: filter by service + traceId + time range
- Runbooks linked from every alert

**Cost Control:**
- Logs: Retain 7 days hot (Loki), 90 days cold (S3)
- Traces: Retain 3 days hot, 30 days cold (sampled only)
- Metrics: 15s resolution for 24h, 1m for 30d, 5m for 1y

**Estimated cost**: ~$3-5K/month for 50 services at 10M req/day
</details>

---

👉 **[Next: Exception Tracking →](./Microservice_Exception_Tracking.md)**  
👉 **[Related: Slower Service Detection →](./Slower_Microservice_Detection.md)**  
👉 **[Back to Microservices Overview →](./README.md)**
