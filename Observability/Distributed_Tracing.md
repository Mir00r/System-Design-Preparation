# 🕵️ Distributed Tracing
## OpenTelemetry, Jaeger, and Following Requests Across Microservices

> *"In a monolith, debugging is like following a single road. In microservices, a request bounces through 10 services like a pinball. Distributed tracing is the replay button — showing you exactly where the ball went and how long it spent at each bumper."*

**⏱️ Estimated Time**: 40 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Logging Best Practices](./Logging_Best_Practices.md), [Metrics & Monitoring](./Metrics_Monitoring.md)

---

## 📋 Table of Contents
1. [The Problem: Debugging Without Traces](#-the-problem-debugging-without-traces)
2. [Core Concepts: Traces, Spans, Context](#-core-concepts-traces-spans-context)
3. [OpenTelemetry Standard](#-opentelemetry-standard)
4. [Jaeger — Trace Visualization](#-jaeger--trace-visualization)
5. [Spring Boot Auto-Instrumentation](#-spring-boot-auto-instrumentation)
6. [Manual Span Creation](#-manual-span-creation)
7. [Sampling Strategies](#-sampling-strategies)
8. [Common Pitfalls](#-common-pitfalls)
9. [Mini Challenge](#-mini-challenge)
10. [Interview Q&A](#-interview-qa)

---

## 🤔 The Problem: Debugging Without Traces

A user reports: "My checkout is taking 8 seconds."

```
Architecture:
  Browser → API Gateway → order-service → inventory-service
                                       → payment-service
                                       → notification-service

Without tracing:
  You search logs on all 5 services.
  Each service reports "200 OK" — no errors.
  How long did each step take? You don't know.
  Which service is slow? You can't tell.
  Is it a network issue between services? Unknown.
  Time to diagnose: 2+ hours.

With distributed tracing:
  One search by traceId in Jaeger:
  
  ┌────────────────────────────────────────────────────────────────────┐
  │ Trace x7f2b3c  Total: 8.2s                                        │
  ├────────────────────────────────────────────────────────────────────┤
  │ ▌api-gateway      (0ms - 8200ms)                                   │
  │  ▌ order-service    (10ms - 8190ms)                                │
  │  │  ▌ inventory-service  (20ms - 120ms) ← 100ms ✅                 │
  │  │  ▌ payment-service    (130ms - 8100ms) ← 7970ms ❌ ROOT CAUSE   │
  │  │  ▌ notification-service (8110ms - 8190ms) ← 80ms ✅             │
  └────────────────────────────────────────────────────────────────────┘
  
  Root cause: payment-service took 8 seconds.
  Time to diagnose: 30 seconds.
```

---

## 🧩 Core Concepts: Traces, Spans, Context

### Trace

A trace is the complete record of a single request's journey through the system. It has a unique **traceId** (e.g., `x7f2b3c4d5e6f7a8b`).

### Span

A span is a single unit of work within a trace — one operation in one service. It has:
- `spanId`: unique to this span
- `parentSpanId`: the span that called this one (null for root span)
- `traceId`: the overall trace this span belongs to
- `start_time` and `end_time`: wall clock timestamps
- `service` and `operation` name
- `tags`: key-value metadata (HTTP method, status code, DB query)
- `logs/events`: timestamped events within the span (errors, checkpoints)

### Span Relationships

```
                   Trace x7f2b3c
                         │
        ┌────────────────▼─────────────────┐
        │ span-1: api-gateway /checkout     │ (root span)
        │ 0ms ─────────────────────── 8200ms│
        └─────────┬───────────────────┬─────┘
                  │  child spans      │
       ┌──────────▼──────┐    ┌───────▼──────────┐
       │ span-2:         │    │ span-3:           │
       │ order-service   │    │ auth-service      │
       │ 10ms ─── 8190ms │    │ 5ms ─── 30ms     │
       └──┬──────────────┘    └──────────────────┘
          │
    ┌─────┴──────────┬────────────────────┐
    │                │                    │
┌───▼────┐     ┌─────▼──────┐      ┌──────▼─────┐
│ span-4 │     │  span-5    │      │  span-6    │
│invent. │     │ payment    │      │ notif.     │
│100ms   │     │ 7970ms ❌  │      │ 80ms       │
└────────┘     └────────────┘      └────────────┘
```

### Trace Context Propagation

For spans to be linked across services, the traceId and parentSpanId must be passed in HTTP headers:

```
Service A calls Service B:

HTTP Request headers:
  traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01
               │  │                                │               │
               │  └─ traceId (128-bit hex)         └─ spanId       └─ flags
               └─ version

Service B reads these headers, creates its child span with:
  traceId = 4bf92f3577b34da6a3ce929d0e0e4736  (same as parent)
  parentSpanId = 00f067aa0ba902b7             (Service A's spanId)
  spanId = [new unique ID for this span]
```

W3C `traceparent` is the standard header (used by OpenTelemetry). Legacy: Zipkin uses `X-B3-TraceId`, Jaeger uses `uber-trace-id`.

---

## 🌐 OpenTelemetry Standard

OpenTelemetry (OTel) is a CNCF standard for instrumentation — one SDK, multiple backends.

```
┌─────────────────────────────────────────────────────────────────┐
│                    OpenTelemetry Architecture                   │
│                                                                 │
│  Your Application (annotated with OTel SDK)                     │
│  ↓ telemetry data (traces, metrics, logs)                       │
│                                                                 │
│  [OTel Collector]  — receives, processes, exports               │
│  ↓ routes to backends                                           │
│                                                                 │
│  [Jaeger] [Zipkin] [Tempo] [Datadog] [Honeycomb] [X-Ray]       │
│  (you can switch backends without changing app code)            │
└─────────────────────────────────────────────────────────────────┘

Before OpenTelemetry: Each vendor had their own SDK.
  Changing from Zipkin to Datadog = rewrite all instrumentation code.

After OpenTelemetry: Instrument once with OTel SDK.
  Change only the OTel Collector config to switch backends.
```

---

## 🔍 Jaeger — Trace Visualization

Jaeger is the most popular open-source distributed tracing backend.

```
Jaeger UI features:
  - Search traces by service, operation, duration, tags, time range
  - Visualize span timelines (waterfall view)
  - Compare traces side-by-side
  - Find root cause with critical path analysis
  - Service dependency graph (which service calls which)

Jaeger architecture (modern):
  App ──→ [OTel Collector] ──→ [Jaeger Backend] ──→ [Elasticsearch/Cassandra]
                                      ↑
                               [Jaeger Query UI]
```

---

## ⚙️ Spring Boot Auto-Instrumentation

Spring Boot 3 + Micrometer Tracing + OTel provides auto-instrumentation with zero code changes:

```xml
<!-- pom.xml -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
```

```yaml
# application.yaml
management:
  tracing:
    sampling:
      probability: 1.0      # 100% sampling (reduce in high-traffic production)
    propagation:
      type: w3c             # Use W3C traceparent header standard

# OpenTelemetry exporter config
otel:
  exporter:
    otlp:
      endpoint: http://otel-collector:4317  # OTel Collector endpoint
  service:
    name: order-service
```

**Auto-instrumented by Spring Boot + Micrometer**:
- All incoming HTTP requests (creates root span)
- All outgoing HTTP calls via `RestTemplate` / `WebClient` (creates child span, injects headers)
- All Spring Data / JPA queries (creates DB span with query info)
- Kafka publish/consume
- gRPC calls
- MDC integration: `traceId` and `spanId` automatically added to every log line

---

## ✏️ Manual Span Creation

For business logic that auto-instrumentation doesn't cover:

```java
@Service
public class OrderService {

    private final Tracer tracer;  // Micrometer's Tracer abstraction

    public Order createOrder(OrderRequest request) {
        // Create a span for the entire order creation process
        Span orderSpan = tracer.nextSpan()
                .name("order.create")
                .tag("orderId", request.getOrderId())
                .tag("userId", request.getUserId())
                .tag("itemCount", String.valueOf(request.getItems().size()))
                .start();

        try (Tracer.SpanInScope ws = tracer.withSpan(orderSpan.start())) {
            // 1. Validate inventory (child span created by HTTP client automatically)
            validateInventory(request);

            // 2. Process payment (manual span for important sub-step)
            Span paymentSpan = tracer.nextSpan().name("payment.process").start();
            try (Tracer.SpanInScope paymentScope = tracer.withSpan(paymentSpan)) {
                processPayment(request);
            } finally {
                paymentSpan.end();
            }

            // 3. Add custom event (timestamped annotation within a span)
            orderSpan.event("order.saved.to.db");

            Order order = saveOrder(request);
            orderSpan.tag("orderStatus", order.getStatus());
            return order;

        } catch (Exception e) {
            orderSpan.tag("error", "true")
                     .tag("error.message", e.getMessage())
                     .event("order.failed");
            throw e;
        } finally {
            orderSpan.end();  // MUST always end the span
        }
    }
}
```

### Context Propagation in Async Code

```java
// PROBLEM: Async methods lose trace context
@Async
public void sendNotification(String userId, String message) {
    // If called without context propagation, this runs in a different thread
    // with no traceId — the span is disconnected from the parent trace
    emailService.send(userId, message);
}

// SOLUTION: Spring's @Async + Micrometer Tracing auto-propagates context
// As long as you use Spring's TaskExecutor (not raw threads), context propagates.
// For raw threads/CompletableFuture, wrap with context:

CompletableFuture<Void> future = CompletableFuture.runAsync(
        tracer.currentTraceContext().wrap(  // wraps runnable with current context
            () -> emailService.send(userId, message)
        )
);
```

---

## 🎯 Sampling Strategies

100% trace sampling is expensive. At 10,000 req/sec, you'd store 864 million traces/day.

```
Sampling strategies:

1. PROBABILITY SAMPLING (simplest)
   Sample 1% of all requests uniformly.
   Problem: Rare errors (0.01% rate) are almost never captured.

2. RATE LIMITING SAMPLING
   Always sample up to N traces per second (e.g., 10/sec per service).
   Better for low-traffic services.

3. TAIL-BASED SAMPLING (best, but complex)
   Collect ALL spans. Make the sampling decision AFTER the trace completes.
   Always sample:
     - Traces with errors (status=ERROR)
     - Traces with high latency (duration > P99 threshold)
     - Traces with specific user IDs (for debugging specific accounts)
   Drop: fast, successful traces (statistically sample these)
   
   Requires an OTel Collector or Jaeger Collector with buffering.

4. ADAPTIVE SAMPLING
   Automatically adjust sampling rate based on traffic volume.
   Low traffic: 100% sampling. High traffic: 0.1% sampling.
```

```yaml
# application.yaml: Spring Boot sampling config
management:
  tracing:
    sampling:
      probability: 0.1    # sample 10% of requests (for high-traffic production)
      # For tail-based sampling, set to 1.0 here and configure sampling in OTel Collector
```

---

## ⚠️ Common Pitfalls

1. **Not propagating context in async code** — When you create a thread pool thread, `CompletableFuture`, or `@Async` method without propagating trace context, the child span loses its traceId. Use `tracer.currentTraceContext().wrap()` or Spring's `TaskDecorator`.

2. **High-cardinality span tags** — `span.tag("userId", userId)` creates unique values for every user. Jaeger/Elasticsearch can handle this, but it increases storage significantly. Tag wisely; use indexed tags for values you'll search by.

3. **Sampling too aggressively** — 0.1% sampling means you'll never see a bug that affects 0.05% of requests. Tail-based sampling (always sample errors) is much better than probability sampling.

4. **Not ending spans** — A span without `span.end()` is never exported. Use try/finally to guarantee spans are always ended, even on exceptions.

5. **Trusting only the happy path** — Auto-instrumentation traces HTTP calls but may miss: database stored procedures, Redis pipelining, Kafka batch processing. Profile what auto-instrumentation covers and manually instrument gaps.

---

## 🧩 Mini Challenge

**Your team is adding tracing to a microservice that processes Kafka messages.** Auto-instrumentation traces the Kafka `consume` operation, but the trace ends there. The message handling code calls 3 downstream services — those spans appear as separate orphaned traces.

**What's the problem and how do you fix it?**

<details>
<summary>💡 Click to reveal answer</summary>

**Problem**: The Kafka message producer (upstream service) encoded trace context in the Kafka message headers when publishing. The consumer needs to **extract** that context and use it as the parent context for all subsequent operations.

If you don't extract the context, each Kafka message handler creates a new root span — disconnected from the trace that produced the message.

**Fix: Extract trace context from Kafka headers**:

```java
@KafkaListener(topics = "order-events")
public void handleOrderEvent(ConsumerRecord<String, OrderEvent> record) {
    // Extract W3C traceparent from Kafka headers
    TextMapGetter<Headers> getter = new TextMapGetter<>() {
        @Override
        public Iterable<String> keys(Headers headers) {
            return StreamSupport.stream(headers.spliterator(), false)
                    .map(Header::key).collect(Collectors.toList());
        }

        @Override
        public String get(Headers headers, String key) {
            Header header = headers.lastHeader(key);
            return header != null ? new String(header.value()) : null;
        }
    };

    // Create context from Kafka headers
    Context extractedContext = openTelemetry.getPropagators()
            .getTextMapPropagator()
            .extract(Context.current(), record.headers(), getter);

    // Use extracted context as parent for all spans in this handler
    try (Scope scope = extractedContext.makeCurrent()) {
        Span span = tracer.spanBuilder("kafka.process.order-event")
                .setParent(extractedContext)
                .startSpan();
        try (Scope spanScope = span.makeCurrent()) {
            processOrderEvent(record.value());  // all downstream calls now inherit this context
        } finally {
            span.end();
        }
    }
}
```

**Result**: The entire flow — from HTTP request that produced the Kafka message, through the Kafka consumer, to all downstream service calls — appears in one unified trace in Jaeger.

</details>

---

## 📝 Interview Q&A

**Q: Explain the relationship between a trace, a span, and a traceId.**
> A: A trace represents the complete end-to-end journey of a single request across all services — it's the big picture. A trace consists of one or more spans. A span is a single unit of work: one operation in one service (e.g., "payment-service processing charge"). Every span in the same trace shares the same traceId (a 128-bit ID). Spans are linked in a parent-child tree: the API gateway creates the root span, which is the parent of spans in downstream services it calls, which are parents of their own downstream calls.

**Q: How does trace context propagate between microservices?**
> A: Via HTTP headers, following the W3C traceparent standard: `traceparent: 00-{traceId}-{parentSpanId}-{flags}`. When Service A calls Service B, it injects these headers into the outgoing HTTP request. Service B extracts the headers and uses the traceId and parentSpanId to link its spans as children of Service A's span. OpenTelemetry auto-instruments this for RestTemplate, WebClient, and gRPC — developers don't write this propagation code manually.

**Q: Why is tail-based sampling better than probability sampling?**
> A: Probability sampling randomly keeps N% of traces, regardless of their value. This means most kept traces are boring successful requests, while the rare error traces you actually need for debugging are likely to be dropped (a 0.1% error rate with 1% sampling means you keep only 1 in 1000 error traces). Tail-based sampling defers the keep/drop decision until after the trace is complete. You can then apply rules: "always keep traces with errors, high latency, or specific tags; drop routine fast successful traces." This maximizes diagnostic value while minimizing storage costs.

---

## 🔗 What to Read Next

1. **[Observability/Alerting_SLO_SLA_SLI.md](./Alerting_SLO_SLA_SLI.md)** — Use trace latency data to define and monitor SLOs
2. **[Observability/Metrics_Monitoring.md](./Metrics_Monitoring.md)** — Correlate high-P99 metric alerts with traces to find root cause
3. **[Microservices/](../Microservices/)** — Distributed tracing is most valuable (and most complex) in microservice architectures

---

*[← Metrics & Monitoring](./Metrics_Monitoring.md) | [Back to Observability](./README.md) | [Next: SLO, SLA, SLI & Alerting →](./Alerting_SLO_SLA_SLI.md)*
