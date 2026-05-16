# 📊 Metrics & Monitoring
## Prometheus, Grafana, RED/USE Methods, and What to Measure

> *"Logs tell you what happened. Metrics tell you how your system is behaving over time — and they're the only way to spot a degradation trend before it becomes an outage."*

**⏱️ Estimated Time**: 40 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Logging Best Practices](./Logging_Best_Practices.md)

---

## 📋 Table of Contents
1. [Metric Types](#-metric-types)
2. [What to Measure: RED and USE Methods](#-what-to-measure-red-and-use-methods)
3. [Prometheus — Collection & Storage](#-prometheus--collection--storage)
4. [Grafana — Visualization](#-grafana--visualization)
5. [Spring Boot Metrics with Micrometer](#-spring-boot-metrics-with-micrometer)
6. [Custom Business Metrics](#-custom-business-metrics)
7. [Alerting with AlertManager](#-alerting-with-alertmanager)
8. [Common Pitfalls](#-common-pitfalls)
9. [Mini Challenge](#-mini-challenge)
10. [Interview Q&A](#-interview-qa)

---

## 🔢 Metric Types

Understanding the four core metric types is essential for Prometheus and most monitoring systems:

### Counter — Only Goes Up

```
What it is: A cumulative count that only increases (resets to 0 on restart)
Examples:   http_requests_total, errors_total, orders_processed_total

Use for:    Counting events. Prometheus RATE() function gives you rate of change.

PromQL:
  rate(http_requests_total[5m])           → requests per second (averaged over 5m)
  increase(errors_total[1h])              → total errors in last hour
  rate(errors_total[5m]) / rate(http_requests_total[5m])  → error rate %
```

### Gauge — Can Go Up or Down

```
What it is: A snapshot of a current value
Examples:   memory_used_bytes, active_connections, queue_depth, cpu_temperature

Use for:    Current state measurements.

PromQL:
  memory_used_bytes                       → current memory
  memory_used_bytes / memory_total_bytes  → memory utilization %
  active_db_connections > 80             → alert if connections high
```

### Histogram — Distribution of Values

```
What it is: Counts observations in configurable buckets (e.g., request durations)
Examples:   http_request_duration_seconds, db_query_duration_ms

Structure:  {le="0.1"}: 2450  (2450 requests completed in ≤100ms)
            {le="0.5"}: 3100  (3100 requests in ≤500ms)
            {le="1.0"}: 3195  (3195 requests in ≤1s)
            {le="+Inf"}: 3200 (3200 total requests)

Use for:    Latency percentiles, response size distributions.

PromQL:
  histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
  → P99 latency over last 5 minutes — the most important latency metric
```

### Summary — Pre-computed Percentiles

```
What it is: Like histogram, but percentiles are computed at the client (not PromQL)
Examples:   Used by some libraries; generally prefer histograms

Downside:   Cannot aggregate across multiple instances (histogram can)
Recommendation: Use histograms in most cases.
```

---

## 🎯 What to Measure: RED and USE Methods

Two mental models for deciding what metrics to track:

### RED Method (for Services / APIs)

**R**ate · **E**rrors · **D**uration — for every service endpoint

```
Rate:     How many requests per second is this service handling?
          metric: rate(http_requests_total[1m])

Errors:   What fraction of requests are failing?
          metric: rate(http_requests_total{status=~"5.."}[1m]) / rate(http_requests_total[1m])
          Target: < 0.1% error rate (for most services)

Duration: How long are requests taking? (P50, P95, P99)
          metric: histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
          Target: P99 < 1 second (define per service)
```

### USE Method (for Resources / Infrastructure)

**U**tilization · **S**aturation · **E**rrors — for every resource

```
Resource: CPU, Memory, Disk, Network, DB Connections, Thread Pool

Utilization:  How busy is the resource? (% of time it's being used)
              CPU: rate(cpu_seconds_total{mode!="idle"}[5m])
              DB:  active_connections / max_connections

Saturation:   How much work is queued waiting for the resource?
              Thread pool queue depth
              Run queue length (CPU waiting tasks)
              DB connection wait time

Errors:       How many errors is the resource producing?
              Disk read/write errors
              Network packet drops
              DB deadlocks
```

### Golden Signals (Google SRE's Version of RED)

```
Latency:       Time to service a request (both successful AND failed)
Traffic:       Demand on the system (requests/sec, transactions/sec)
Errors:        Rate of failed requests (HTTP 5xx, exception rate)
Saturation:    How "full" the service is (CPU %, memory %, queue depth)
```

---

## 🔭 Prometheus — Collection & Storage

```
Architecture:

  [Service A]   [Service B]   [Service C]
  /metrics ←──  /metrics ←──  /metrics ←──  ← Prometheus scrapes every 15s
       ↑              ↑              ↑
       └──────────────┴──────────────┘
                      │
               [Prometheus Server]
               (TSDB — time-series DB)
                      │
              PromQL queries
                      │
               [Grafana / AlertManager]
```

### Prometheus Scraping

```yaml
# prometheus.yml — tells Prometheus where to scrape metrics
global:
  scrape_interval: 15s      # how often to pull metrics
  evaluation_interval: 15s  # how often to evaluate alert rules

scrape_configs:
  - job_name: 'order-service'
    static_configs:
      - targets: ['order-service:8080']   # /actuator/prometheus endpoint
    metrics_path: '/actuator/prometheus'

  # Kubernetes service discovery — auto-discovers pods with prometheus annotations
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
```

### PromQL Essentials

```promql
# Rate of HTTP requests per second (5-minute window)
rate(http_requests_total[5m])

# Error rate as percentage
sum(rate(http_requests_total{status=~"5.."}[5m]))
  / sum(rate(http_requests_total[5m])) * 100

# P99 latency
histogram_quantile(0.99,
  sum(rate(http_request_duration_seconds_bucket[5m])) by (le, service))

# Memory utilization by pod (Kubernetes)
container_memory_usage_bytes{container!="POD"}
  / container_spec_memory_limit_bytes * 100

# Alert: DB connection pool nearly exhausted
hikaricp_connections_active / hikaricp_connections_max > 0.8
```

---

## 📈 Grafana — Visualization

```
Key dashboards every production service should have:

1. SERVICE OVERVIEW DASHBOARD
   ┌─────────────────┬─────────────────┬─────────────────┐
   │   Request Rate  │   Error Rate    │  P99 Latency    │
   │   3,240 req/s   │   0.08%         │   145ms         │
   └─────────────────┴─────────────────┴─────────────────┘
   ┌──────────────────────────────────────────────────────┐
   │  Request Rate over time (line graph, 6h window)      │
   │  ████████████████████████████████████████████████   │
   └──────────────────────────────────────────────────────┘

2. INFRASTRUCTURE DASHBOARD
   CPU % │ Memory % │ Disk I/O │ Network I/O │ DB Connections

3. BUSINESS METRICS DASHBOARD
   Orders/min │ Revenue/min │ Failed Payments │ Cart Abandonment
```

---

## ⚙️ Spring Boot Metrics with Micrometer

Micrometer is Spring Boot's metrics facade — it sends metrics to Prometheus, Datadog, CloudWatch, etc. via a unified API.

```xml
<!-- pom.xml dependencies -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

```yaml
# application.yaml: expose Prometheus metrics endpoint
management:
  endpoints:
    web:
      exposure:
        include: health, info, prometheus, metrics
  endpoint:
    prometheus:
      enabled: true
  metrics:
    tags:
      service: order-service         # add service tag to all metrics
      environment: production
    distribution:
      percentiles-histogram:
        http.server.requests: true   # enable histograms for HTTP latency
      percentiles:
        http.server.requests: 0.5,0.95,0.99  # P50/P95/P99
```

```java
// Spring Boot auto-instruments:
//   - HTTP request rates and latency (http.server.requests)
//   - JVM memory, GC, thread pools
//   - HikariCP connection pool
//   - Tomcat/Netty connection stats
// All available at /actuator/prometheus
```

---

## 📐 Custom Business Metrics

Auto-instrumentation covers infrastructure. You must manually add business metrics.

```java
@Service
public class OrderService {

    // Define meters once (inject MeterRegistry)
    private final Counter ordersCreated;
    private final Counter ordersFailed;
    private final Timer orderProcessingTime;
    private final Gauge pendingOrders;

    public OrderService(MeterRegistry registry, OrderRepository orderRepo) {
        // Counter: monotonically increasing
        this.ordersCreated = Counter.builder("orders.created")
                .description("Total orders successfully created")
                .tag("region", "us-east-1")
                .register(registry);

        this.ordersFailed = Counter.builder("orders.failed")
                .description("Total orders that failed")
                .register(registry);

        // Timer: records duration and count
        this.orderProcessingTime = Timer.builder("orders.processing.duration")
                .description("Time to process an order end-to-end")
                .publishPercentiles(0.5, 0.95, 0.99)
                .register(registry);

        // Gauge: snapshot of current state (use supplier to get live value)
        this.pendingOrders = Gauge.builder("orders.pending", orderRepo,
                        repo -> repo.countByStatus("PENDING"))
                .description("Current number of pending orders")
                .register(registry);
    }

    public Order createOrder(OrderRequest request) {
        return orderProcessingTime.record(() -> {
            try {
                Order order = processOrder(request);  // business logic
                ordersCreated.increment();
                return order;
            } catch (Exception e) {
                ordersFailed.increment(
                        Tags.of("reason", e.getClass().getSimpleName()));
                throw e;
            }
        });
    }
}
```

---

## 🚨 Alerting with AlertManager

```yaml
# alert_rules.yml — Prometheus alerting rules
groups:
  - name: order-service
    rules:
      # Alert: high error rate
      - alert: HighErrorRate
        expr: |
          sum(rate(http_requests_total{job="order-service",status=~"5.."}[5m]))
          / sum(rate(http_requests_total{job="order-service"}[5m])) > 0.01
        for: 2m           # must be true for 2 minutes before alerting (avoids flapping)
        labels:
          severity: critical
          team: backend
        annotations:
          summary: "High error rate on order-service"
          description: "Error rate is {{ $value | humanizePercentage }} (threshold: 1%)"
          runbook: "https://wiki.mycompany.com/runbooks/order-service-errors"

      # Alert: high latency
      - alert: HighP99Latency
        expr: |
          histogram_quantile(0.99,
            rate(http_request_duration_seconds_bucket{job="order-service"}[5m])) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "P99 latency above 2 seconds on order-service"

      # Alert: DB connections nearly exhausted
      - alert: DBConnectionsHigh
        expr: |
          hikaricp_connections_active{pool="orderServicePool"}
          / hikaricp_connections_max > 0.85
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "DB connection pool {{ $value | humanizePercentage }} utilized"
```

---

## ⚠️ Common Pitfalls

1. **High cardinality labels** — Never use user IDs, order IDs, or URLs as metric labels. `http_requests_total{userId="12345"}` creates one metric series per user — millions of series crash Prometheus. Labels should have low cardinality (< 100 unique values): `status`, `method`, `service`.

2. **Alerting on averages** — P50 (average) latency looks fine while P99 is catastrophic. Always alert on P99 (or P95). "Average response time: 150ms" can coexist with "1% of users wait > 10 seconds."

3. **Alert fatigue** — Too many alerts → on-call engineers ignore them. Every alert must be actionable (someone must do something when it fires) and have a runbook link. Silence noisy alerts rather than letting them cry wolf.

4. **Not measuring business metrics** — Infrastructure looks green but revenue is down. Instrument business events: `orders.created.per.minute`, `payment.failure.rate`, `cart.abandonment.rate`. These are the metrics that actually matter.

5. **Missing the `for` clause in alerts** — Without `for: 2m`, a 1-second CPU spike triggers PagerDuty at 3 AM. Always require the condition to persist for a meaningful duration before alerting.

---

## 🧩 Mini Challenge

**You're on-call.** A Grafana alert fires: P99 latency on `order-service` is 8 seconds. The dashboard shows:
- CPU: 25% (normal)
- Memory: 60% (normal)
- Error rate: 0.3% (slightly high)
- DB connection pool: 98% utilized 🔴
- JVM Heap: 45% (normal)

**What is the likely root cause, and what are your next 3 actions?**

<details>
<summary>💡 Click to reveal answer</summary>

**Root cause**: DB connection pool exhaustion. At 98% utilization, nearly all connections are in use. New requests queue waiting for a free connection — this explains the high P99 latency (requests wait up to 8 seconds for a connection) while average CPU/memory are fine.

**Next 3 actions**:

1. **Immediate (next 2 minutes)**: Check if there are long-running/stuck queries. Query: `SHOW PROCESSLIST` in MySQL or `SELECT * FROM pg_stat_activity WHERE state = 'active' ORDER BY duration DESC`. If there are queries running for > 30 seconds — kill them.

2. **Short-term (next 10 minutes)**: Temporarily increase HikariCP max pool size in the running service (`maximumPoolSize: 20 → 30`) and hot-reload the config if possible. Monitor if connection utilization drops and latency recovers. Also scale horizontally — deploy 2 additional order-service instances to spread the connection load.

3. **Investigation (next hour)**: Find what caused connection leaks/overload. Check: (a) was there a traffic spike? (look at request rate metric), (b) are there connection leaks? (check `hikaricp_connections_pending` trend — if it's steadily increasing, connections aren't being returned), (c) are there slow queries? (check `db_query_duration_seconds` histogram for P99 spikes).

**Why this is good**: You diagnosed from metrics (not guessing), took immediate action to restore service, and planned root cause investigation rather than only firefighting.

</details>

---

## 📝 Interview Q&A

**Q: What's the difference between a Counter and a Gauge in Prometheus?**
> A: A Counter only ever increases (it resets to 0 on restart). It's for counting events: total requests, total errors, total bytes sent. You use `rate()` to compute the rate of change. A Gauge is a point-in-time snapshot that can go up or down: CPU usage, memory consumption, current connections, queue depth. You query Gauges directly without `rate()`.

**Q: How do you compute P99 latency in Prometheus?**
> A: Use a Histogram metric (e.g., `http_request_duration_seconds_bucket`) and the `histogram_quantile` function: `histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))`. The histogram records counts of observations in pre-defined duration buckets. PromQL uses linear interpolation across buckets to estimate the percentile. The key advantage over a Summary is that histograms can be aggregated across multiple instances.

**Q: What is the RED method and when do you use it?**
> A: RED stands for Rate, Errors, Duration. It's a framework for monitoring request-driven microservices. Rate = requests per second (is demand changing?). Errors = fraction of requests failing (is the service healthy?). Duration = latency percentiles, especially P99 (is performance acceptable?). You apply RED to every HTTP endpoint or gRPC method. These three signals cover the most common failure modes and are the first things to check when an alert fires.

---

## 🔗 What to Read Next

1. **[Observability/Distributed_Tracing.md](./Distributed_Tracing.md)** — Connect metrics to traces to find which specific requests are causing high P99
2. **[Observability/Alerting_SLO_SLA_SLI.md](./Alerting_SLO_SLA_SLI.md)** — Turn metrics into SLOs and define error budgets
3. **[BuildingBlocks/CircuitBreaker.md](../BuildingBlocks/CircuitBreaker.md)** — Use metrics from circuit breakers to trigger alerting

---

*[← Logging Best Practices](./Logging_Best_Practices.md) | [Back to Observability](./README.md) | [Next: Distributed Tracing →](./Distributed_Tracing.md)*
