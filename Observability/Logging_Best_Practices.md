# 📋 Logging Best Practices
## Structured Logs, ELK Stack, and What Not to Log

> *"Logs are your system's diary. When things go wrong at 3 AM, the quality of your logs is the difference between a 5-minute fix and a 3-hour debugging session."*

**⏱️ Estimated Time**: 35 minutes | **🎯 Difficulty**: 🟢 Easy | **🔗 Prerequisites**: None

---

## 📋 Table of Contents
1. [Log Levels — When to Use Each](#-log-levels--when-to-use-each)
2. [Structured vs Unstructured Logging](#-structured-vs-unstructured-logging)
3. [What to Log (and What NOT to)](#-what-to-log-and-what-not-to)
4. [Correlation IDs](#-correlation-ids)
5. [Log Aggregation — ELK Stack](#-log-aggregation--elk-stack)
6. [Spring Boot Logging Configuration](#-spring-boot-logging-configuration)
7. [Common Pitfalls](#-common-pitfalls)
8. [Mini Challenge](#-mini-challenge)
9. [Interview Q&A](#-interview-qa)

---

## 🤔 The Problem

It's 2 AM. Users report orders are failing. You SSH into the production server and grep the logs:

```
[INFO] Processing request
[INFO] User authenticated
[ERROR] Something went wrong
[INFO] Request completed
```

**Useless.** Which user? Which order? What went wrong? When exactly? 5 minutes later you're still no closer to the cause.

Now with good logging:
```json
{"timestamp":"2026-05-17T02:14:23Z","level":"ERROR","service":"order-service","traceId":"x7f2b3c","userId":"12345","orderId":"ORD-789","message":"Payment failed","error":"PaymentGatewayException","errorCode":"INSUFFICIENT_FUNDS","duration_ms":234}
```

Root cause found in 30 seconds.

---

## 🎚️ Log Levels — When to Use Each

| Level | When to Use | Volume | Alerting |
|---|---|---|---|
| **TRACE** | Extremely detailed: method entry/exit, variable values | Massive | Never |
| **DEBUG** | Developer-useful detail: SQL queries, cache hits/misses | High | Never |
| **INFO** | Normal business events: user logged in, order created | Medium | Rarely |
| **WARN** | Unexpected but recoverable: retry happened, cache miss, deprecated API used | Low | Sometimes |
| **ERROR** | An operation failed: exception caught, payment declined, DB query failed | Very Low | Always |
| **FATAL** | System cannot continue: DB connection lost, OOM, critical config missing | Rare | Always + PagerDuty |

### Rule of Thumb

```
Production environment: INFO level (or WARN for high-traffic services)
  → You see business events + all problems
  → You don't drown in debug noise

Debugging a specific issue: temporarily set to DEBUG for affected package
  log.setLevel("com.myapp.payment", Level.DEBUG)  // specific package only
  // Never set root logger to DEBUG in production — disk I/O will overwhelm you

Never in production:
  → log.trace(...)  — only for local development
  → Blanket log.debug(...) at root level
```

---

## 🏗️ Structured vs Unstructured Logging

### Unstructured (Plain Text) — Avoid

```
INFO  2026-05-17 02:14:23 - User 12345 created order ORD-789 for $29.99 in 234ms
ERROR 2026-05-17 02:14:45 - Payment failed for order ORD-789: insufficient funds
```

Problems:
- Parsing requires fragile regex
- Searching by field is slow (full-text scan)
- Adding fields breaks existing parsers
- No standard schema — every developer writes differently

### Structured Logging (JSON) — Use This

```json
{"timestamp":"2026-05-17T02:14:23.456Z","level":"INFO","service":"order-service","version":"2.1.4","environment":"production","traceId":"x7f2b3c4d","spanId":"a1b2c3","userId":"12345","orderId":"ORD-789","amount":29.99,"currency":"USD","duration_ms":234,"message":"Order created successfully"}

{"timestamp":"2026-05-17T02:14:45.123Z","level":"ERROR","service":"order-service","traceId":"x7f2b3c4d","userId":"12345","orderId":"ORD-789","errorType":"PaymentGatewayException","errorCode":"INSUFFICIENT_FUNDS","paymentProvider":"stripe","message":"Payment processing failed"}
```

Benefits:
- Machine-parseable: Elasticsearch indexes each field automatically
- Field-based searching: `orderId: "ORD-789"` finds all logs instantly
- Consistent schema across services
- Easy aggregation: "count errors by errorCode in last 1 hour"

---

## ✅ What to Log (and What NOT to)

### ✅ DO Log

```java
// Business events (INFO)
log.info("event=order_created orderId={} userId={} amount={} duration_ms={}",
        orderId, userId, amount, duration);

// All errors with context (ERROR)
log.error("event=payment_failed orderId={} errorCode={} provider={}",
        orderId, e.getCode(), provider, e);

// External service calls (INFO/WARN)
log.info("event=external_api_call service=stripe endpoint=/v1/charges duration_ms={} status={}",
        duration, response.getStatus());

// Security events (WARN/ERROR to dedicated security log)
log.warn("event=login_failed email={} ip={} attemptCount={}",
        maskEmail(email), clientIp, attempts);

// Performance boundaries (WARN if slow)
if (duration > 1000) {
    log.warn("event=slow_query query={} duration_ms={}", queryName, duration);
}
```

### ❌ NEVER Log

```java
// NEVER: Credentials / secrets
log.info("User logged in with password: {}", password);          // plaintext password in logs!
log.debug("API key: {}", apiKey);                                // secret in logs!
log.info("JWT token: {}", jwtToken);                             // token theft via logs!

// NEVER: PII without masking
log.info("Processing card: {}", creditCardNumber);               // PCI-DSS violation
log.info("SSN: {}", socialSecurityNumber);                       // HIPAA/GDPR violation
log.info("Full DOB: {}", dateOfBirth);                           // PII in logs

// NEVER: User-controlled data without sanitization
log.info("Search query: " + userSearchInput);                    // log injection attack
// Attacker input: "\n[ERROR] Admin password changed\n" → injects fake log entries
// Use structured logging (parameterized) which prevents this automatically

// AVOID: Verbose objects in production
log.debug("Full response: {}", response.toString());             // could be 10MB of JSON
```

### Masking Sensitive Data

```java
// Mask credit card: show only last 4 digits
public static String maskCard(String cc) {
    return "****-****-****-" + cc.substring(cc.length() - 4);
}

// Mask email: show first char + domain
public static String maskEmail(String email) {
    int at = email.indexOf('@');
    return email.charAt(0) + "***" + email.substring(at);
    // "alice@example.com" → "a***@example.com"
}

// Usage:
log.info("event=checkout userId={} cardLastFour={} email={}",
        userId, maskCard(cardNumber), maskEmail(email));
```

---

## 🔗 Correlation IDs

In microservices, a single user request touches 5-10 services. Without a correlation ID, you can't connect the logs across services.

```
Without correlation ID:
  order-service logs:    "Order created for user 12345"
  payment-service logs:  "Payment processed for $29.99"
  email-service logs:    "Confirmation email sent"
  
  → Which logs belong to the same request? You can't tell.

With correlation ID (traceId):
  order-service:   {"traceId": "x7f2b", "message": "Order created", "userId": "12345"}
  payment-service: {"traceId": "x7f2b", "message": "Payment processed", "amount": 29.99}
  email-service:   {"traceId": "x7f2b", "message": "Email sent", "template": "order_confirm"}
  
  → Search Kibana for traceId=x7f2b → see entire request journey instantly
```

### Spring Boot Correlation ID with MDC

```java
// MDC (Mapped Diagnostic Context) = per-thread key-value store for log context

// Filter: generate or propagate traceId on every request
@Component
public class TraceIdFilter extends OncePerRequestFilter {

    private static final String TRACE_HEADER = "X-Trace-Id";
    private static final String TRACE_KEY = "traceId";

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws IOException, ServletException {
        // Propagate incoming trace ID or generate new one
        String traceId = Optional.ofNullable(request.getHeader(TRACE_HEADER))
                .filter(s -> s.matches("[a-zA-Z0-9\\-]{8,36}"))  // sanitize!
                .orElse(UUID.randomUUID().toString().replace("-", "").substring(0, 12));

        MDC.put(TRACE_KEY, traceId);
        MDC.put("service", "order-service");
        response.setHeader(TRACE_HEADER, traceId);  // return in response too

        try {
            chain.doFilter(request, response);
        } finally {
            MDC.clear();  // MUST clear — threads are reused in thread pools
        }
    }
}

// logback-spring.xml: include MDC values in every log line
// <pattern>{"timestamp":"%d{ISO8601}","level":"%-5level","traceId":"%X{traceId}","service":"%X{service}","message":"%msg"}%n</pattern>
```

---

## 🗂️ Log Aggregation — ELK Stack

At scale, logs from hundreds of instances must be centralized for searching.

```
┌───────────────────────────────────────────────────────────────┐
│                     ELK STACK                                 │
│                                                               │
│  [App Instances] → [Filebeat] → [Logstash] → [Elasticsearch] │
│  (emit logs)        (collect)   (transform)  (index + store)  │
│                                              ↓                │
│                                          [Kibana]             │
│                                         (visualize + search)  │
└───────────────────────────────────────────────────────────────┘

Modern alternative: Grafana Loki (cheaper — indexes labels only, not full text)
  [App] → [Promtail/Alloy] → [Loki] → [Grafana]
```

### Elasticsearch Query (Kibana DevTools)

```json
// Find all errors for a specific order in the last hour
GET logs-*/_search
{
  "query": {
    "bool": {
      "must": [
        { "term":  { "orderId": "ORD-789" } },
        { "term":  { "level": "ERROR" } },
        { "range": { "timestamp": { "gte": "now-1h" } } }
      ]
    }
  },
  "sort": [{ "timestamp": { "order": "desc" } }]
}

// Aggregate: error rate by service over time
GET logs-*/_search
{
  "aggs": {
    "errors_over_time": {
      "date_histogram": { "field": "timestamp", "calendar_interval": "1m" },
      "aggs": {
        "by_service": {
          "terms": { "field": "service" }
        }
      }
    }
  }
}
```

---

## ⚙️ Spring Boot Logging Configuration

```xml
<!-- logback-spring.xml: structured JSON logging for production -->
<configuration>
  <springProfile name="production">
    <appender name="JSON_STDOUT" class="ch.qos.logback.core.ConsoleAppender">
      <encoder class="net.logstash.logback.encoder.LogstashEncoder">
        <!-- Adds MDC fields (traceId, service) automatically -->
        <includeMdcKeyName>traceId</includeMdcKeyName>
        <includeMdcKeyName>service</includeMdcKeyName>
        <customFields>{"environment":"production","app":"order-service"}</customFields>
      </encoder>
    </appender>
    <root level="INFO">
      <appender-ref ref="JSON_STDOUT"/>
    </root>
    <!-- Quieten noisy libraries -->
    <logger name="org.hibernate.SQL" level="WARN"/>
    <logger name="org.springframework.web" level="WARN"/>
  </springProfile>

  <springProfile name="development">
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
      <encoder>
        <pattern>%d{HH:mm:ss} [%X{traceId}] %-5level %logger{36} - %msg%n</pattern>
      </encoder>
    </appender>
    <root level="DEBUG">
      <appender-ref ref="CONSOLE"/>
    </root>
  </springProfile>
</configuration>
```

```yaml
# application.yaml: log level by package
logging:
  level:
    root: INFO
    com.myapp: DEBUG          # your code: debug level
    com.myapp.security: WARN  # security logs: warn only (less noise)
    org.springframework: WARN
    org.hibernate: WARN
```

---

## ⚠️ Common Pitfalls

1. **String concatenation in log statements** — `log.debug("User: " + user.toString())` evaluates `user.toString()` even if DEBUG is disabled. Use parameterized logging: `log.debug("User: {}", user)` — the string is only built if the level is active.

2. **Not clearing MDC** — `MDC.put("traceId", id)` in a thread pool without `MDC.clear()` after the request causes the next request on that thread to inherit the previous traceId. Always use try/finally.

3. **Logging inside tight loops** — `for (Item item : 1_000_000_items) { log.debug(...) }` generates millions of log entries, overwhelms disk I/O, and can crash the service. Log summaries, not individual iterations.

4. **Overly broad catch-and-log** — `catch (Exception e) { log.error("Error", e); }` without re-throwing swallows the exception silently. The caller doesn't know something went wrong. Either handle the exception properly, or log + re-throw.

5. **Same log level for everything** — Teams that put everything at INFO level create too much noise. Set appropriate levels so that searching for `level=ERROR` returns only genuine errors, not routine warnings.

---

## 🧩 Mini Challenge

**Review this logging code and identify 3 problems**:

```java
public void processPayment(String userId, String cardNumber, double amount) {
    logger.info("Processing payment for user " + userId + " with card " + cardNumber + " amount: " + amount);
    try {
        paymentGateway.charge(cardNumber, amount);
        logger.info("Payment successful");
    } catch (Exception e) {
        logger.error("Payment failed");
    }
}
```

<details>
<summary>💡 Click to reveal answer</summary>

**Problem 1: String concatenation (performance)**
`"Processing payment for user " + userId + ...` builds the string even if INFO is disabled. Use: `logger.info("event=payment_start userId={} amount={}", userId, amount)`

**Problem 2: Logging the full card number (security/PCI violation)**
`cardNumber` in plain text in logs is a critical PCI-DSS violation. Change to: `maskCard(cardNumber)` → `"****-****-****-1234"`

**Problem 3: Swallowing exception context**
`logger.error("Payment failed")` logs nothing about what went wrong. The `catch (Exception e)` swallows the exception. Change to: `logger.error("event=payment_failed userId={} amount={}", userId, amount, e)` — passing `e` as the last argument includes the full stack trace in the log.

**Bonus Problem 4: No traceId / correlation ID**
In a microservices environment, this log entry can't be correlated with logs from other services for the same request. Should use MDC to include traceId in every log line.

</details>

---

## 📝 Interview Q&A

**Q: What is structured logging and why is it preferred over plain text logs?**
> A: Structured logging emits logs as key-value pairs (typically JSON) rather than free-form text. Benefits: (1) machine-parseable — log aggregators (ELK, Loki) automatically index each field; (2) field-based searching — `orderId:ORD-789` is an indexed query vs full-text scan; (3) consistent schema — all services emit the same fields (traceId, userId, duration_ms); (4) prevents log injection — parameterized logging never concatenates user input directly into log strings.

**Q: How would you handle log volume at scale (100 services, 10K requests/sec)?**
> A: (1) **Sampling**: log 100% of errors, 10% of successful requests — reduces volume 10x without losing important signals. (2) **Async logging**: use async appenders (Logback's AsyncAppender) so log I/O doesn't block request threads. (3) **Log levels by environment**: DEBUG in dev, WARN in production for framework libs. (4) **Centralize and compress**: Loki compresses log streams by label, achieving 10:1+ compression vs Elasticsearch. (5) **Retention policies**: keep ERROR logs 90 days, INFO logs 30 days, DEBUG logs 7 days.

**Q: What's the difference between logs and traces?**
> A: Logs are discrete events emitted by a single service — "Order created", "Payment failed". They tell you what happened within one service. Traces are correlated records that follow a single request across multiple services, showing the full call graph and timing at each step. A trace says "User's checkout request spent 100ms in order-service, 800ms waiting for payment-service, 50ms in email-service." You need logs for detail, traces for end-to-end context.

---

## 🔗 What to Read Next

1. **[Observability/Metrics_Monitoring.md](./Metrics_Monitoring.md)** — Complement logs with time-series metrics for trends and alerting
2. **[Observability/Distributed_Tracing.md](./Distributed_Tracing.md)** — Connect logs across services with trace context propagation
3. **[Security/OWASP_Top10.md](../Security/OWASP_Top10.md)** — A09 (Logging Failures) covers the security angle of what you MUST log

---

*[← Back to Observability](./README.md) | [Next: Metrics & Monitoring →](./Metrics_Monitoring.md)*
