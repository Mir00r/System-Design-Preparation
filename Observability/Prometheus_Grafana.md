# 📈 Prometheus & Grafana: Metrics Collection, Storage, and Visualization

> *"Logs tell you WHAT happened. Metrics tell you HOW MUCH is happening. When your P99 latency spikes from 200ms to 5 seconds, you don't grep logs — you look at a Grafana dashboard that shows exactly when it started, which service is affected, and correlates it with CPU/memory/request-rate metrics."*

**⏱️ Estimated Time**: 40 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Metrics & Monitoring](./Metrics_Monitoring.md), [Alerting SLO/SLA/SLI](./Alerting_SLO_SLA_SLI.md)

---

## 📋 Table of Contents
1. [The Problem](#-the-problem)
2. [Architecture](#-architecture)
3. [Prometheus Deep Dive](#-prometheus-deep-dive)
4. [Metric Types](#-metric-types)
5. [PromQL (Query Language)](#-promql-query-language)
6. [Grafana Dashboards](#-grafana-dashboards)
7. [Alerting Pipeline](#-alerting-pipeline)
8. [Spring Boot Integration](#-spring-boot-integration)
9. [Scaling Prometheus](#-scaling-prometheus)
10. [Common Pitfalls](#-common-pitfalls)
11. [Mini Challenge](#-mini-challenge)
12. [Interview Q&A](#-interview-qa)

---

## 🤔 The Problem

```
Without metrics monitoring:

  PagerDuty alert: "Users reporting slow checkout"
  Engineer: *opens Kibana, searches logs*
  20 minutes later: "Payment service has errors"
  But WHY? CPU? Memory? Network? Database? Downstream dependency?
  Another 20 minutes correlating metrics manually...

With Prometheus + Grafana:

  PagerDuty alert: "P99 latency > 2s on payment-service" (auto-detected)
  Engineer opens Grafana dashboard:
    - Sees latency spike started at 14:32
    - Correlates with: DB connection pool exhausted at 14:31
    - Root cause: connection leak introduced in deploy at 14:30
  Resolution time: 3 minutes (vs 40+ minutes)
```

---

## 🏗️ Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│              PROMETHEUS + GRAFANA ARCHITECTURE                       │
│                                                                      │
│  TARGETS (expose /metrics endpoint)          PROMETHEUS              │
│                                                                      │
│  [Order Service :8080/metrics]  ◀─── scrape ───┐                    │
│  [Payment Service :8081/metrics]◀─── scrape ───┤                    │
│  [User Service :8082/metrics]   ◀─── scrape ───┤                    │
│  [Node Exporter :9100/metrics]  ◀─── scrape ───┤   ┌────────────┐  │
│  [PostgreSQL Exporter :9187]    ◀─── scrape ───┤   │ Prometheus │  │
│  [Redis Exporter :9121]         ◀─── scrape ───┘   │            │  │
│                                                     │ - Scrapes  │  │
│                                                     │   every 15s│  │
│  PULL MODEL:                                        │ - Stores   │  │
│    Prometheus PULLS metrics from targets            │   time-series│ │
│    (targets don't push — they just expose)          │ - Evaluates │  │
│                                                     │   alert rules│ │
│                                                     └─────┬──────┘  │
│                                                           │          │
│                              ┌─────────────────────────────┘          │
│                              │                                        │
│                 ┌────────────▼────────────┐                           │
│                 │      Grafana            │                           │
│                 │  - Visualize metrics    │                           │
│                 │  - Build dashboards     │                           │
│                 │  - Query with PromQL    │                           │
│                 └─────────────────────────┘                           │
│                                                                      │
│                 ┌─────────────────────────┐                           │
│                 │   Alertmanager          │                           │
│                 │  - Deduplicate alerts   │──▶ Slack / PagerDuty     │
│                 │  - Route by severity    │──▶ Email                 │
│                 │  - Silence / inhibit    │──▶ Webhook               │
│                 └─────────────────────────┘                           │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 🔍 Prometheus Deep Dive

```
PULL vs PUSH MODEL:

  PUSH (StatsD, Datadog Agent):
    Application pushes metrics → aggregator → storage
    Problem: if aggregator is down, metrics are lost

  PULL (Prometheus):
    Prometheus scrapes /metrics endpoint every 15 seconds
    Application just exposes current state — stateless!
    Benefits:
      - Prometheus controls scrape rate (no flood)
      - Easy to detect if target is DOWN (scrape fails)
      - Targets are simple (just expose a /metrics HTTP endpoint)

DATA MODEL (time-series):
  Every metric is a time-series identified by:
    metric_name{label1="value1", label2="value2"} value timestamp

  Example:
    http_requests_total{method="GET", path="/api/orders", status="200"} 15234 1705312800
    http_requests_total{method="POST", path="/api/orders", status="500"} 42 1705312800
    http_request_duration_seconds{method="GET", path="/api/orders", quantile="0.99"} 0.250

STORAGE:
  - Custom TSDB (Time-Series Database) optimized for append-only writes
  - Compressed on disk: ~1.5 bytes per sample
  - Default retention: 15 days (configurable)
  - For long-term: Thanos / Cortex / Mimir (extends to years)
```

---

## 📊 Metric Types

```
1. COUNTER (monotonically increasing)
   Use: total requests, total errors, bytes sent
   Only goes UP (resets on restart)
   
   http_requests_total{status="200"} → 15234 → 15235 → 15236...
   
   PromQL: rate(http_requests_total[5m])  → requests per second

2. GAUGE (arbitrary value, goes up and down)
   Use: temperature, queue size, active connections, memory usage
   
   process_memory_bytes → 524288000 → 612345000 → 498765000...
   
   PromQL: process_memory_bytes  → current value

3. HISTOGRAM (distribution of values in buckets)
   Use: request latency, response size
   Records: count of observations in each bucket
   
   http_request_duration_seconds_bucket{le="0.1"} 8000  (8000 requests < 100ms)
   http_request_duration_seconds_bucket{le="0.5"} 9500  (9500 requests < 500ms)
   http_request_duration_seconds_bucket{le="1.0"} 9900  (9900 requests < 1s)
   http_request_duration_seconds_bucket{le="+Inf"} 10000 (10000 total)
   
   PromQL: histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
           → P99 latency

4. SUMMARY (pre-calculated quantiles — calculated client-side)
   Use: similar to histogram but quantiles computed by the application
   Cheaper to query but can't aggregate across instances
   
   http_request_duration_seconds{quantile="0.99"} 0.250
   http_request_duration_seconds{quantile="0.95"} 0.180
```

---

## 🔎 PromQL (Query Language)

```
ESSENTIAL QUERIES:

# Request rate (requests per second, averaged over 5 minutes)
rate(http_requests_total[5m])

# Error rate (% of requests that are 5xx)
sum(rate(http_requests_total{status=~"5.."}[5m])) 
/ sum(rate(http_requests_total[5m])) * 100

# P99 latency
histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le))

# P99 latency per service
histogram_quantile(0.99, 
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service))

# CPU usage per pod (percentage)
rate(container_cpu_usage_seconds_total[5m]) * 100

# Memory usage (percentage of limit)
container_memory_usage_bytes / container_spec_memory_limit_bytes * 100

# Top 5 services by error rate
topk(5, sum(rate(http_requests_total{status=~"5.."}[5m])) by (service))

# Requests per second, grouped by endpoint
sum(rate(http_requests_total[5m])) by (path)

# Alert: P99 > 1 second for more than 5 minutes
http_request_duration_seconds{quantile="0.99"} > 1 # for > 5m
```

---

## 📊 Grafana Dashboards

```
THE FOUR GOLDEN SIGNALS (Google SRE) — one row each:

┌─────────────────────────────────────────────────────────────────┐
│  LATENCY                                                        │
│  ┌──────────────────────────────────────────────────┐           │
│  │ P50: 45ms | P95: 180ms | P99: 450ms             │           │
│  │         ───────────────────                       │           │
│  │    ──────                  ──────                 │           │
│  │ ──                              ──── (time →)    │           │
│  └──────────────────────────────────────────────────┘           │
├─────────────────────────────────────────────────────────────────┤
│  TRAFFIC (requests per second)                                  │
│  ┌──────────────────────────────────────────────────┐           │
│  │ Current: 1,234 req/s                              │           │
│  │     ████████████████████████                      │           │
│  │ ████                        ████                  │           │
│  └──────────────────────────────────────────────────┘           │
├─────────────────────────────────────────────────────────────────┤
│  ERRORS (error rate %)                                          │
│  ┌──────────────────────────────────────────────────┐           │
│  │ Current: 0.3% | Threshold: 1%                    │           │
│  │ ─────────────────────────────── (flat = good)    │           │
│  └──────────────────────────────────────────────────┘           │
├─────────────────────────────────────────────────────────────────┤
│  SATURATION (resource utilization)                              │
│  ┌──────────────────────────────────────────────────┐           │
│  │ CPU: 45% | Memory: 62% | DB Connections: 80/100  │           │
│  │ ████████░░ CPU   ████████████░░ MEM   ████████░░ DB│         │
│  └──────────────────────────────────────────────────┘           │
└─────────────────────────────────────────────────────────────────┘

DASHBOARD BEST PRACTICES:
  - USE (Utilization, Saturation, Errors) per service
  - RED (Rate, Errors, Duration) per endpoint
  - One overview dashboard + drill-down dashboards per service
  - Include deployment markers (vertical lines when deploys happen)
  - Link dashboards to runbooks ("click here for troubleshooting guide")
```

---

## 🚨 Alerting Pipeline

```
Prometheus evaluates alert rules → fires to Alertmanager → routes to channels

# prometheus-rules.yml
groups:
  - name: slo-alerts
    rules:
      # P99 latency > 1s for 5+ minutes
      - alert: HighLatency
        expr: histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service)) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "P99 latency > 1s on {{ $labels.service }}"
          dashboard: "https://grafana.internal/d/svc-overview?var-service={{ $labels.service }}"

      # Error rate > 5% for 2+ minutes
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{status=~"5.."}[5m])) by (service)
          / sum(rate(http_requests_total[5m])) by (service) > 0.05
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Error rate > 5% on {{ $labels.service }}"

# alertmanager.yml
route:
  receiver: 'slack-default'
  routes:
    - match:
        severity: critical
      receiver: 'pagerduty-oncall'
    - match:
        severity: warning
      receiver: 'slack-engineering'

receivers:
  - name: 'pagerduty-oncall'
    pagerduty_configs:
      - routing_key: '<pagerduty-key>'
  - name: 'slack-engineering'
    slack_configs:
      - channel: '#engineering-alerts'
        text: '{{ .Annotations.summary }}'
```

---

## 💻 Spring Boot Integration

```java
// pom.xml dependencies:
// spring-boot-starter-actuator + micrometer-registry-prometheus

// application.yml
management:
  endpoints:
    web:
      exposure:
        include: health, prometheus, metrics
  metrics:
    tags:
      service: order-service
      environment: production

// Exposes: http://localhost:8080/actuator/prometheus
// Output:
//   http_server_requests_seconds_count{method="GET",uri="/api/orders",status="200"} 15234
//   http_server_requests_seconds_sum{method="GET",uri="/api/orders",status="200"} 3421.5
//   jvm_memory_used_bytes{area="heap"} 524288000
//   hikaricp_connections_active{pool="HikariPool-1"} 8

// Custom business metrics
@Service
public class OrderService {
    private final Counter orderCounter;
    private final Timer orderProcessingTimer;
    private final Gauge activeOrdersGauge;

    public OrderService(MeterRegistry registry) {
        this.orderCounter = Counter.builder("orders_created_total")
            .description("Total orders created")
            .tag("type", "standard")
            .register(registry);

        this.orderProcessingTimer = Timer.builder("order_processing_duration_seconds")
            .description("Time to process an order")
            .publishPercentiles(0.5, 0.95, 0.99)  // P50, P95, P99
            .register(registry);

        this.activeOrdersGauge = Gauge.builder("orders_active", 
                orderRepository, repo -> repo.countByStatus("PROCESSING"))
            .description("Currently processing orders")
            .register(registry);
    }

    public Order createOrder(CreateOrderRequest request) {
        return orderProcessingTimer.record(() -> {
            Order order = processOrder(request);
            orderCounter.increment();
            return order;
        });
    }
}

// Custom health indicator affecting UP/DOWN
@Component
public class PaymentServiceHealth implements HealthIndicator {
    @Override
    public Health health() {
        boolean paymentReachable = checkPaymentService();
        if (paymentReachable) {
            return Health.up().withDetail("payment-service", "reachable").build();
        }
        return Health.down().withDetail("payment-service", "unreachable").build();
    }
}
```

### Prometheus scrape config (for Kubernetes):

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'spring-boot-apps'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
        target_label: __metrics_path__
        regex: (.+)

# Pod annotation to enable scraping:
# metadata:
#   annotations:
#     prometheus.io/scrape: "true"
#     prometheus.io/path: "/actuator/prometheus"
#     prometheus.io/port: "8080"
```

---

## 📐 Scaling Prometheus

```
SINGLE PROMETHEUS LIMITATIONS:
  - ~1M active time-series per instance
  - 15-day default retention
  - Single point of failure
  - Can't query across multiple Prometheus instances

SCALING SOLUTIONS:

1. FEDERATION (horizontal sharding):
   Prometheus A: scrapes services in namespace "orders"
   Prometheus B: scrapes services in namespace "payments"
   Global Prometheus: federates (pulls aggregated metrics from A and B)

2. THANOS (long-term storage + global view):
   [Prometheus] → [Thanos Sidecar] → [Object Storage (S3)]
                                            ↑
   [Thanos Query] ← queries across multiple Prometheus + S3
   
   Benefits: unlimited retention, global query, deduplication

3. GRAFANA MIMIR (Grafana's scalable alternative):
   [Prometheus] → remote_write → [Mimir cluster]
   
   Benefits: horizontally scalable, multi-tenant, long-term storage

CARDINALITY MANAGEMENT (critical for scaling):
  ❌ BAD: http_requests_total{user_id="12345"} → millions of series!
  ✅ GOOD: http_requests_total{service="order", method="GET", status="200"} → thousands
  
  Rule: labels should have LOW cardinality (< 100 unique values per label)
  High-cardinality dimensions (user_id, request_id) belong in LOGS, not metrics
```

---

## ⚠️ Common Pitfalls

1. **High cardinality labels** — Adding `user_id`, `request_id`, or `ip_address` as metric labels creates millions of time-series, exhausting Prometheus memory. Metrics labels should have bounded, low cardinality (< 100 values). Use logs for high-cardinality data.

2. **Alerting on symptoms, not causes** — Alert: "CPU > 80%". This is useless — CPU at 80% might be normal under load. Instead, alert on USER-FACING impact: "P99 latency > SLO" or "error rate > 1%". Then use CPU metric for root cause investigation on the dashboard.

3. **Missing `rate()` on counters** — Plotting a counter directly shows a monotonically increasing line (useless). Always wrap counters in `rate()` or `increase()` to see per-second rates. `rate(http_requests_total[5m])` shows requests/sec.

4. **Too many dashboards, no one watches them** — 50 dashboards that nobody looks at are worse than 3 focused ones with alerts. Build: one SERVICE OVERVIEW dashboard (golden signals) + per-service drill-downs opened only during incidents.

5. **No recording rules for expensive queries** — Complex PromQL queries (nested aggregations, histogram_quantile) are expensive to compute on every dashboard load. Use recording rules to pre-compute and store results: `record: job:http_requests:rate5m` evaluated every 15s.

---

## 🧩 Mini Challenge

**Your `payment-service` has an SLO of P99 latency < 500ms. Write a Prometheus alerting rule that fires when the SLO is violated for more than 5 minutes, and a Grafana dashboard panel query showing the latency trend.**

<details>
<summary>💡 Click to reveal answer</summary>

**Alerting rule** (prometheus-rules.yml):
```yaml
groups:
  - name: payment-slo
    rules:
      - alert: PaymentLatencySLOViolation
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_server_requests_seconds_bucket{service="payment-service"}[5m])) by (le)
          ) > 0.5
        for: 5m
        labels:
          severity: critical
          service: payment-service
          slo: "p99_latency"
        annotations:
          summary: "Payment service P99 latency {{ $value | humanizeDuration }} > 500ms SLO"
          dashboard: "https://grafana.internal/d/payment-svc"
          runbook: "https://wiki.internal/runbooks/payment-high-latency"
```

**Grafana panel query** (shows P50, P95, P99 trend lines):
```
# Panel 1: Latency percentiles over time
A: histogram_quantile(0.99, sum(rate(http_server_requests_seconds_bucket{service="payment-service"}[5m])) by (le))
   Legend: P99

B: histogram_quantile(0.95, sum(rate(http_server_requests_seconds_bucket{service="payment-service"}[5m])) by (le))
   Legend: P95

C: histogram_quantile(0.50, sum(rate(http_server_requests_seconds_bucket{service="payment-service"}[5m])) by (le))
   Legend: P50

# Add threshold line at 500ms (SLO boundary)
D: 0.5
   Legend: SLO Target (500ms)
```

**Dashboard config**:
- Y-axis: seconds (auto-format to ms)
- Threshold: red line at 0.5s
- Alert annotation: vertical red bar when alert fires
- Time range: last 6 hours (to see when degradation started)

</details>

---

## 📝 Interview Q&A

**Q: Why does Prometheus use a pull model instead of push?**
> A: Pull model advantages: (1) **Prometheus controls the scrape rate** — no risk of being overwhelmed by misbehaving targets that push too frequently. (2) **Target health detection** — if a scrape fails, Prometheus knows the target is DOWN (with push, silence is ambiguous — is the service healthy but idle, or crashed?). (3) **Simpler targets** — services just expose an HTTP endpoint; no need to configure a push destination. (4) **Easy development** — you can curl the /metrics endpoint to debug what's being exposed. The downside: pull doesn't work well for short-lived jobs (batch jobs that finish before the next scrape). For these, Prometheus provides the Pushgateway as an exception.

**Q: How do you decide between using a metric vs a log?**
> A: **Metrics** are for aggregatable numeric measurements: request rate, error percentage, latency percentiles, resource utilization. They answer "how many?" and "how fast?" across time. They're cheap to store (8 bytes per sample) and fast to query (PromQL aggregation). **Logs** are for discrete events with context: "user X got error Y with payload Z at time T." They answer "what happened?" for a specific event. They're expensive to store (1KB+ per entry) but provide full context for debugging. Rule of thumb: if you need `COUNT()`, `SUM()`, `AVG()`, or percentiles → metric. If you need to understand a specific event → log. Never put user_ids in metrics labels — that's a log use case.

---

## 🔗 What to Read Next

1. **[Observability/Metrics_Monitoring.md](./Metrics_Monitoring.md)** — The theory behind metrics and monitoring
2. **[Observability/ELK_Stack.md](./ELK_Stack.md)** — Complement metrics with log aggregation
3. **[Observability/OpenTelemetry.md](./OpenTelemetry.md)** — Unified framework that combines metrics, logs, and traces

---

*[← ELK Stack](./ELK_Stack.md) | [Back to Observability](./README.md) | [Next: OpenTelemetry →](./OpenTelemetry.md)*
