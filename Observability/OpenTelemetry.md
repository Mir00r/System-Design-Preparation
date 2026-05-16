# 🔭 OpenTelemetry: The Universal Observability Framework

> *"Before OpenTelemetry, you had Prometheus for metrics, Jaeger for traces, and ELK for logs — three separate SDKs, three different instrumentation libraries, three vendor lock-ins. OpenTelemetry unifies all three pillars into one standard: instrument once, send anywhere."*

**⏱️ Estimated Time**: 40 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Distributed Tracing](./Distributed_Tracing.md), [Metrics & Monitoring](./Metrics_Monitoring.md), [Logging Best Practices](./Logging_Best_Practices.md)

---

## 📋 Table of Contents
1. [The Problem](#-the-problem)
2. [What is OpenTelemetry](#-what-is-opentelemetry)
3. [Three Pillars United](#-three-pillars-united)
4. [Architecture](#-architecture)
5. [Key Components](#-key-components)
6. [Spring Boot Integration](#-spring-boot-integration)
7. [The Collector](#-the-collector)
8. [Context Propagation](#-context-propagation)
9. [Common Pitfalls](#-common-pitfalls)
10. [Mini Challenge](#-mini-challenge)
11. [Interview Q&A](#-interview-qa)

---

## 🤔 The Problem

```
BEFORE OpenTelemetry (observability fragmentation):

  [Your Service]
       │
       ├── Micrometer SDK → Prometheus (metrics)
       ├── Zipkin/Brave SDK → Jaeger (traces)
       ├── Logback + Logstash encoder → ELK (logs)
       └── Datadog SDK → Datadog (APM)  ← vendor lock-in!

  Problems:
    1. Multiple SDKs: 4 different libraries, 4 sets of configuration
    2. Vendor lock-in: switching from Jaeger to Zipkin = re-instrument everything
    3. No correlation: logs don't link to traces, metrics don't link to logs
    4. Inconsistent: trace context might not propagate to log context
    5. Overhead: 4 SDKs running = 4× performance impact

AFTER OpenTelemetry (unified standard):

  [Your Service]
       │
       └── OpenTelemetry SDK (ONE SDK for everything)
              │
              ▼
       [OTel Collector] (receives all telemetry)
              │
              ├──▶ Prometheus (metrics)
              ├──▶ Jaeger (traces)
              ├──▶ Elasticsearch (logs)
              └──▶ Datadog / New Relic / Any backend

  Benefits:
    1. ONE SDK: single instrumentation for metrics, traces, AND logs
    2. Vendor agnostic: switch backends without code changes
    3. Correlated: trace_id in logs, exemplars in metrics → click from metric to trace
    4. Industry standard: CNCF graduated project, backed by every major vendor
```

---

## 🌟 What is OpenTelemetry

```
OpenTelemetry (OTel) is:
  - A STANDARD for generating, collecting, and exporting telemetry data
  - An open-source project under CNCF (graduated — same level as Kubernetes)
  - NOT a backend: it doesn't store or visualize data
  - A universal ADAPTER between your code and ANY observability platform

Components:
  1. SPECIFICATION: defines what metrics/traces/logs look like (the "protocol")
  2. SDKs: language-specific libraries (Java, Go, Python, Node.js, .NET...)
  3. AUTO-INSTRUMENTATION: automatic tracing for common frameworks (Spring, gRPC, HTTP)
  4. COLLECTOR: receives, processes, and exports telemetry data
  5. OTLP: OpenTelemetry Protocol (the wire format — gRPC or HTTP)

Supported languages: Java, Go, Python, JavaScript, .NET, Ruby, PHP, Rust, C++, Swift
Supported backends: Jaeger, Zipkin, Prometheus, Datadog, New Relic, Splunk, Grafana Cloud, AWS X-Ray...
```

---

## 🏛️ Three Pillars United

```
┌─────────────────────────────────────────────────────────────────┐
│         THE THREE PILLARS OF OBSERVABILITY                      │
│                                                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   METRICS    │  │   TRACES     │  │    LOGS      │          │
│  │              │  │              │  │              │          │
│  │ "How much?"  │  │ "What path?" │  │ "What event?"│          │
│  │              │  │              │  │              │          │
│  │ • Counters   │  │ • Spans      │  │ • Structured │          │
│  │ • Gauges     │  │ • Trace ID   │  │ • Timestamp  │          │
│  │ • Histograms │  │ • Parent ID  │  │ • Severity   │          │
│  │              │  │ • Duration   │  │ • Body       │          │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘          │
│         │                  │                  │                  │
│         └──────────────────┼──────────────────┘                  │
│                            │                                     │
│                   ┌────────▼────────┐                            │
│                   │ CORRELATION     │                            │
│                   │                 │                            │
│                   │ trace_id links  │                            │
│                   │ all three:      │                            │
│                   │                 │                            │
│                   │ Metric spike →  │                            │
│                   │ Find traces →   │                            │
│                   │ See logs        │                            │
│                   └─────────────────┘                            │
│                                                                 │
│  EXEMPLARS: metrics point to specific trace IDs                 │
│    http_request_duration{...} 2.5  trace_id=abc123              │
│    → Click on the spike → jump directly to the slow trace!      │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🏗️ Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                  OPENTELEMETRY ARCHITECTURE                          │
│                                                                      │
│  APPLICATION LAYER                                                   │
│                                                                      │
│  [Service A]                    [Service B]                          │
│   │ OTel SDK                     │ OTel SDK                          │
│   │ - Auto-instrumentation       │ - Auto-instrumentation            │
│   │ - Manual spans               │ - Manual spans                    │
│   │ - Custom metrics             │ - Custom metrics                  │
│   │ - Structured logs            │ - Structured logs                 │
│   │                              │                                   │
│   │  OTLP (gRPC/HTTP)           │  OTLP                            │
│   └──────────┬───────────────────┘──────────┘                       │
│              │                                                       │
│              ▼                                                       │
│  ┌───────────────────────────────────────────┐                      │
│  │        OTEL COLLECTOR                      │                      │
│  │                                            │                      │
│  │  Receivers    Processors    Exporters      │                      │
│  │  ┌────────┐  ┌──────────┐  ┌───────────┐  │                      │
│  │  │ OTLP   │  │ Batch    │  │ Prometheus│  │                      │
│  │  │ Jaeger │→│ Filter   │→│ Jaeger    │  │                      │
│  │  │ Zipkin │  │ Sampling │  │ OTLP/gRPC│  │                      │
│  │  │ Kafka  │  │ Enrich   │  │ Loki     │  │                      │
│  │  └────────┘  └──────────┘  └───────────┘  │                      │
│  └───────────────────────────────────────────┘                      │
│              │              │              │                          │
│              ▼              ▼              ▼                          │
│  ┌──────────────┐  ┌────────────┐  ┌──────────────┐                │
│  │  Prometheus  │  │   Jaeger   │  │  Grafana Loki│                │
│  │  (metrics)   │  │  (traces)  │  │   (logs)     │                │
│  └──────────────┘  └────────────┘  └──────────────┘                │
│                                                                      │
│  ALL accessible via Grafana (unified dashboards)                     │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 🧩 Key Components

### Traces & Spans

```
A TRACE = the entire journey of a request across services
A SPAN = one unit of work within a trace

Example: User places an order

Trace ID: abc-123-def-456
├─ Span: API Gateway (12ms)
│  └─ Span: Order Service - createOrder (45ms)
│     ├─ Span: Validate Request (2ms)
│     ├─ Span: PostgreSQL - INSERT order (8ms)
│     ├─ Span: Payment Service - charge (200ms)  ← slow!
│     │  ├─ Span: Stripe API call (180ms)        ← root cause
│     │  └─ Span: PostgreSQL - UPDATE payment (5ms)
│     └─ Span: Kafka - publish order.created (3ms)
└─ Total: 250ms

Span attributes:
  {
    "trace_id": "abc123def456",
    "span_id": "span_789",
    "parent_span_id": "span_456",
    "name": "POST /api/payments/charge",
    "kind": "CLIENT",
    "start_time": "2024-01-15T10:30:00.000Z",
    "end_time": "2024-01-15T10:30:00.200Z",
    "attributes": {
      "http.method": "POST",
      "http.url": "https://payment-service/charge",
      "http.status_code": 200,
      "payment.amount": 99.99,
      "payment.currency": "USD"
    },
    "status": "OK"
  }
```

### Metrics

```
OpenTelemetry metric instruments (mapped to Prometheus types):

  Counter → Prometheus counter
    api.http.requests.total (labels: method, path, status)

  UpDownCounter → Prometheus gauge
    api.active_connections (goes up and down)

  Histogram → Prometheus histogram
    api.http.request.duration (buckets: 10ms, 50ms, 100ms, 500ms, 1s, 5s)

  Gauge → Prometheus gauge
    system.memory.usage (current value)

OTLP metric format:
  {
    "name": "http.server.request.duration",
    "unit": "s",
    "histogram": {
      "data_points": [{
        "start_time": "...",
        "time": "...",
        "count": 1500,
        "sum": 450.5,
        "bucket_counts": [800, 400, 200, 70, 25, 5],
        "explicit_bounds": [0.01, 0.05, 0.1, 0.5, 1.0, 5.0],
        "attributes": { "http.method": "GET", "http.route": "/api/orders" }
      }]
    }
  }
```

---

## 💻 Spring Boot Integration

### Zero-Code Auto-Instrumentation (Java Agent)

```bash
# Download the OTel Java agent
curl -LO https://github.com/open-telemetry/opentelemetry-java-instrumentation/releases/latest/download/opentelemetry-javaagent.jar

# Run your Spring Boot app with the agent attached
java -javaagent:opentelemetry-javaagent.jar \
  -Dotel.service.name=order-service \
  -Dotel.exporter.otlp.endpoint=http://otel-collector:4317 \
  -Dotel.metrics.exporter=otlp \
  -Dotel.logs.exporter=otlp \
  -jar order-service.jar

# That's it! Auto-instruments:
#   - Spring MVC/WebFlux (HTTP server spans)
#   - RestTemplate/WebClient (HTTP client spans)
#   - JDBC (database query spans)
#   - Kafka producer/consumer (messaging spans)
#   - gRPC, Redis, MongoDB, and 100+ more libraries
```

### Manual Instrumentation (Custom Spans + Metrics)

```java
// build.gradle / pom.xml
// io.opentelemetry:opentelemetry-api
// io.opentelemetry:opentelemetry-sdk
// io.opentelemetry.instrumentation:opentelemetry-spring-boot-starter

@Service
public class OrderService {
    private final Tracer tracer;
    private final Meter meter;
    private final LongCounter orderCounter;
    private final DoubleHistogram orderValueHistogram;

    public OrderService(OpenTelemetry openTelemetry) {
        this.tracer = openTelemetry.getTracer("order-service");
        this.meter = openTelemetry.getMeter("order-service");
        
        this.orderCounter = meter.counterBuilder("orders.created.total")
            .setDescription("Total orders created")
            .build();
            
        this.orderValueHistogram = meter.histogramBuilder("orders.value")
            .setDescription("Order value distribution")
            .setUnit("USD")
            .build();
    }

    public Order createOrder(CreateOrderRequest request) {
        // Create a custom span for business logic
        Span span = tracer.spanBuilder("createOrder")
            .setAttribute("order.items_count", request.getItems().size())
            .setAttribute("order.user_id", request.getUserId())
            .startSpan();

        try (Scope scope = span.makeCurrent()) {
            // Validate
            validateOrder(request);
            
            // Process payment (auto-instrumented HTTP call → child span created automatically)
            PaymentResult payment = paymentClient.charge(request.getAmount());
            span.setAttribute("payment.transaction_id", payment.getTransactionId());
            
            // Save order
            Order order = orderRepository.save(toEntity(request, payment));
            
            // Record metrics
            orderCounter.add(1, Attributes.of(
                AttributeKey.stringKey("order.type"), request.getType()));
            orderValueHistogram.record(request.getAmount().doubleValue());
            
            span.setStatus(StatusCode.OK);
            return order;
            
        } catch (Exception e) {
            span.setStatus(StatusCode.ERROR, e.getMessage());
            span.recordException(e);
            throw e;
        } finally {
            span.end();
        }
    }
    
    private void validateOrder(CreateOrderRequest request) {
        // Child span (nested under createOrder)
        Span validationSpan = tracer.spanBuilder("validateOrder").startSpan();
        try (Scope scope = validationSpan.makeCurrent()) {
            // validation logic...
            validationSpan.setStatus(StatusCode.OK);
        } finally {
            validationSpan.end();
        }
    }
}
```

### Correlated Logs (trace_id in every log line)

```java
// With OTel Java agent, trace context is automatically added to MDC:
// logback-spring.xml
<encoder class="net.logstash.logback.encoder.LogstashEncoder">
    <provider class="net.logstash.logback.composite.logback.LoggingEventJsonProviders">
        <mdc/>  <!-- Automatically includes trace_id, span_id from OTel -->
    </provider>
</encoder>

// Log output (trace_id automatically included):
// {
//   "@timestamp": "2024-01-15T10:30:00Z",
//   "level": "INFO",
//   "message": "Order created successfully",
//   "trace_id": "abc123def456",  ← automatically injected by OTel agent
//   "span_id": "span789",
//   "service": "order-service",
//   "order_id": "ord-123"
// }

// In Grafana: click trace_id in logs → jumps to Jaeger trace view
// In Jaeger: click "View Logs" → filters Loki logs by this trace_id
```

---

## 🔄 The Collector

```
The OTel Collector is a PROXY that receives, processes, and exports telemetry:

WHY USE A COLLECTOR (instead of exporting directly from apps)?
  1. Decoupling: apps don't know about backends (change backend = change collector config)
  2. Batching: collector batches data before sending (reduces network calls)
  3. Processing: filter PII, sample traces, add metadata — without app changes
  4. Reliability: collector buffers if backend is down (prevents data loss)
  5. Multi-export: send same data to multiple backends simultaneously

COLLECTOR CONFIGURATION (otel-collector-config.yaml):

receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 5s
    send_batch_size: 1000
  
  # Remove sensitive attributes
  attributes:
    actions:
      - key: user.email
        action: delete
      - key: db.statement
        action: hash  # hash SQL queries (keep structure, hide data)
  
  # Tail-based sampling: keep 100% of errors, 10% of success
  tail_sampling:
    policies:
      - name: errors-always
        type: status_code
        status_code: { status_codes: [ERROR] }
      - name: sample-success
        type: probabilistic
        probabilistic: { sampling_percentage: 10 }

exporters:
  prometheus:
    endpoint: "0.0.0.0:8889"
  
  jaeger:
    endpoint: jaeger-collector:14250
    tls:
      insecure: true
  
  loki:
    endpoint: http://loki:3100/loki/api/v1/push
  
  otlp:
    endpoint: "tempo:4317"  # Grafana Tempo for traces

service:
  pipelines:
    metrics:
      receivers: [otlp]
      processors: [batch]
      exporters: [prometheus]
    traces:
      receivers: [otlp]
      processors: [batch, tail_sampling]
      exporters: [jaeger, otlp]
    logs:
      receivers: [otlp]
      processors: [batch, attributes]
      exporters: [loki]
```

---

## 🔗 Context Propagation

```
HOW TRACE CONTEXT FLOWS ACROSS SERVICES:

  [Service A] ─── HTTP Request ───▶ [Service B] ─── gRPC ───▶ [Service C]

  Service A creates trace: trace_id=abc123, span_id=span_A
  
  HTTP Header injected automatically:
    traceparent: 00-abc123-span_A-01
    (W3C Trace Context standard)

  Service B reads header → continues the SAME trace:
    Creates span_B with parent=span_A, same trace_id=abc123

  Service B calls Service C via gRPC:
    gRPC metadata: traceparent: 00-abc123-span_B-01

  Service C reads metadata → creates span_C (parent=span_B, trace=abc123)

  Result: all three services share ONE trace_id
  → Searching "abc123" in Jaeger shows the complete distributed trace

PROPAGATION FORMATS:
  - W3C Trace Context (standard, recommended): traceparent header
  - B3 (Zipkin format): X-B3-TraceId, X-B3-SpanId headers
  - Jaeger format: uber-trace-id header
  
  OTel SDK supports ALL formats and can convert between them
  (important for gradually migrating from Zipkin/Jaeger to OTel)
```

---

## ⚠️ Common Pitfalls

1. **Over-instrumenting** — Creating a span for every method call produces trace noise and significant overhead (10-30% CPU). Only create custom spans for meaningful operations: DB calls, HTTP calls, business logic that takes > 1ms. Let auto-instrumentation handle the rest.

2. **Ignoring sampling** — Collecting 100% of traces at high traffic (100K req/sec) produces terabytes of trace data daily. Implement tail-based sampling: keep 100% of errors and slow requests, sample 1-10% of successful requests. This captures all interesting traces while reducing cost by 90%.

3. **Not correlating signals** — Using OTel for traces but separate tools for logs/metrics defeats the purpose. Ensure trace_id appears in log lines (MDC injection), and use exemplars to link metrics to specific traces. The power is in clicking from a metric spike → trace → logs.

4. **Collector as single point of failure** — If the OTel Collector crashes, all telemetry is lost. Deploy as a DaemonSet (one per node) with a gateway collector behind it. SDK has retry/buffer for short outages, but prolonged collector downtime = data loss.

5. **High-cardinality span attributes** — Adding `user.id` or `request.body` as span attributes explodes storage costs in trace backends. Use span attributes for low-cardinality classification (service, method, status), and span events/logs for high-cardinality debugging data.

---

## 🧩 Mini Challenge

**You have 50 microservices, currently using Zipkin for tracing and Prometheus for metrics. Design a migration plan to OpenTelemetry without any observability downtime.**

<details>
<summary>💡 Click to reveal answer</summary>

**Phase 1: Deploy OTel Collector (no app changes)**
```
Current: [Apps] → Zipkin SDK → [Zipkin]
         [Apps] → Micrometer → [Prometheus scrape]

Add:     [OTel Collector] configured to:
           - Receive: Zipkin format (zipkin receiver)
           - Export: Zipkin (existing backend) + Jaeger (new backend, shadow)
```
At this point: nothing changes for apps. Collector is just a passthrough.

**Phase 2: Migrate apps one-by-one (gradual)**
```
For each service:
  1. Add OTel Java agent (-javaagent:opentelemetry-javaagent.jar)
  2. Configure OTLP exporter → OTel Collector
  3. Remove Zipkin SDK dependency
  4. Remove Micrometer Prometheus endpoint (OTel agent exposes /metrics in Prometheus format)
  
  The agent automatically:
    - Exports traces via OTLP → Collector
    - Exports metrics via OTLP → Collector (or continues Prometheus scrape)
    - Injects trace_id into logs
```

**Phase 3: Collector processes and exports**
```
Collector config (handles both old and new):
  receivers:
    zipkin: {}          # old services still using Zipkin format
    otlp: {}            # new services using OTel
  exporters:
    zipkin: {}          # keep sending to Zipkin (until fully migrated)
    jaeger: {}          # new trace backend
    prometheus: {}      # expose metrics endpoint (Prometheus scrapes collector)
```

**Phase 4: Cut over backends**
```
Once all 50 services migrated:
  - Remove Zipkin receiver and exporter from Collector
  - Point Grafana dashboards to new backends (Jaeger, Prometheus via Collector)
  - Decommission old Zipkin
```

**Key principle**: OTel Collector acts as a translation layer. Both old (Zipkin) and new (OTLP) formats are accepted simultaneously. No "big bang" migration — services migrate incrementally over weeks.

</details>

---

## 📝 Interview Q&A

**Q: What is OpenTelemetry and why should we use it?**
> A: OpenTelemetry is a CNCF-graduated open-source project that provides a vendor-neutral standard for generating, collecting, and exporting telemetry data (metrics, traces, and logs). You should use it because: (1) **Single instrumentation** — one SDK for all three signals instead of separate libraries for each. (2) **Vendor agnostic** — switch from Jaeger to Datadog by changing a config file, not re-instrumenting code. (3) **Correlation** — trace_id connects metrics, traces, and logs (click from a latency spike to the exact slow trace and its logs). (4) **Auto-instrumentation** — Java agent automatically traces HTTP, DB, gRPC, Kafka without code changes. (5) **Industry standard** — every major observability vendor (Datadog, New Relic, Splunk, Grafana) supports OTLP natively.

**Q: How does OpenTelemetry handle the performance overhead of tracing?**
> A: Multiple strategies: (1) **Head-based sampling** — decide at trace creation whether to sample (e.g., 10% of requests). Fast but can miss interesting traces. (2) **Tail-based sampling** — collect all spans, decide at the Collector whether to keep the trace (keep errors, slow requests, drop fast successful ones). Better quality, higher collector cost. (3) **Batching** — SDK batches spans/metrics and sends periodically (every 5s or 1000 items), reducing network overhead from once-per-span to once-per-batch. (4) **Async export** — telemetry export happens on background threads, not on the request hot path. Typical overhead with sampling: 1-3% CPU increase. Without sampling at 100K req/sec: up to 10-15% (hence why sampling is essential at scale).

---

## 🔗 What to Read Next

1. **[Observability/Distributed_Tracing.md](./Distributed_Tracing.md)** — Tracing concepts that OTel implements
2. **[Observability/Prometheus_Grafana.md](./Prometheus_Grafana.md)** — Prometheus as a metrics backend for OTel
3. **[Observability/ELK_Stack.md](./ELK_Stack.md)** — ELK as a logs backend for OTel

---

*[← Prometheus & Grafana](./Prometheus_Grafana.md) | [Back to Observability](./README.md)*
