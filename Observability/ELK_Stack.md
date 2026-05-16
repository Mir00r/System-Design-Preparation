# 📊 ELK Stack: Elasticsearch, Logstash, and Kibana for Centralized Logging

> *"In a microservice architecture with 200 services, each producing thousands of log lines per second, you can't SSH into 200 servers to grep for an error. The ELK stack aggregates, indexes, and visualizes all your logs in one place — turning chaos into searchable, alertable insights."*

**⏱️ Estimated Time**: 40 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Logging Best Practices](./Logging_Best_Practices.md), [Distributed Tracing](./Distributed_Tracing.md)

---

## 📋 Table of Contents
1. [The Problem](#-the-problem)
2. [ELK Stack Architecture](#-elk-stack-architecture)
3. [Each Component Deep Dive](#-each-component-deep-dive)
4. [Data Pipeline](#-data-pipeline)
5. [Index Management](#-index-management)
6. [Spring Boot Integration](#-spring-boot-integration)
7. [Kibana Dashboards](#-kibana-dashboards)
8. [Scaling ELK](#-scaling-elk)
9. [Common Pitfalls](#-common-pitfalls)
10. [Mini Challenge](#-mini-challenge)
11. [Interview Q&A](#-interview-qa)

---

## 🤔 The Problem

```
200 microservices × 1000 log lines/sec = 200,000 log lines per second

Without centralized logging:
  Developer: "There's a 500 error in production"
  Ops: "Which of the 200 services caused it?"
  Developer: *SSHs into 50 servers, greps through 200 log files*
  30 minutes later: "Found it! Order-service threw NullPointerException"
  
  But wait — the root cause was in payment-service (3 hops upstream)
  Another 30 minutes of cross-referencing logs by timestamp...

With ELK Stack:
  Developer: searches Kibana: "error AND trace_id:abc-123"
  2 seconds later: sees the full error chain across all services
  Clicks on trace_id → sees the complete request flow
  Root cause identified in < 1 minute
```

---

## 🏗️ ELK Stack Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                     ELK STACK ARCHITECTURE                           │
│                                                                      │
│  DATA SOURCES                    INGESTION         STORAGE + SEARCH  │
│                                                                      │
│  [Service A] ──log──▶ ┌────────────────┐                            │
│  [Service B] ──log──▶ │  Filebeat      │     ┌─────────────────┐    │
│  [Service C] ──log──▶ │  (lightweight  │────▶│  Logstash       │    │
│  [Nginx]     ──log──▶ │   shipper)     │     │  (transform +   │    │
│  [K8s pods]  ──log──▶ └────────────────┘     │   enrich)       │    │
│                                               └────────┬────────┘    │
│                                                        │             │
│        Optional: Kafka buffer                          │             │
│        (absorb bursts, prevent data loss)              │             │
│        [Filebeat] → [Kafka] → [Logstash]              │             │
│                                                        ▼             │
│                                            ┌─────────────────────┐   │
│                                            │   Elasticsearch     │   │
│  VISUALIZATION                             │   (index + search)  │   │
│                                            │                     │   │
│  ┌──────────────────┐                      │   - Full-text search│   │
│  │   Kibana         │◀─── query ──────────│   - Aggregations    │   │
│  │                  │                      │   - 100K+ docs/sec  │   │
│  │  - Dashboards    │                      │     ingestion       │   │
│  │  - Search        │                      └─────────────────────┘   │
│  │  - Alerts        │                                                │
│  │  - Visualizations│                                                │
│  └──────────────────┘                                                │
└──────────────────────────────────────────────────────────────────────┘
```

---

## 🔍 Each Component Deep Dive

### Elasticsearch (Storage + Search Engine)

```
What it is: Distributed search and analytics engine (built on Apache Lucene)

How it stores logs:
  Index: logs-2024.01.15 (one index per day)
  Document: single log entry (JSON)
    {
      "@timestamp": "2024-01-15T10:30:00Z",
      "level": "ERROR",
      "service": "order-service",
      "trace_id": "abc-123",
      "message": "Payment failed: timeout after 5000ms",
      "host": "pod-order-7b4f8d",
      "exception": "java.net.SocketTimeoutException..."
    }

Key capabilities:
  - Full-text search: find "timeout" across 1B log entries in < 100ms
  - Structured queries: level:ERROR AND service:payment-service AND @timestamp > now-1h
  - Aggregations: count errors per service per minute (for dashboards)
  - Near real-time: logs searchable within 1 second of ingestion

Architecture:
  Cluster: 3+ nodes (master + data nodes)
  Shards: each index split into shards (parallel search)
  Replicas: each shard has 1+ replica (fault tolerance)
```

### Logstash (Transform + Enrich)

```
What it does: Receives logs, transforms them, sends to Elasticsearch

Pipeline: INPUT → FILTER → OUTPUT

Example pipeline (logstash.conf):
  input {
    kafka {
      topics => ["app-logs"]
      bootstrap_servers => "kafka:9092"
    }
  }

  filter {
    # Parse JSON log
    json { source => "message" }
    
    # Extract fields from unstructured logs
    grok {
      match => { "message" => "%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:level} %{GREEDYDATA:msg}" }
    }
    
    # Enrich with GeoIP (for access logs)
    geoip { source => "client_ip" }
    
    # Remove sensitive fields
    mutate { remove_field => ["password", "credit_card", "ssn"] }
    
    # Add derived fields
    mutate { add_field => { "environment" => "production" } }
  }

  output {
    elasticsearch {
      hosts => ["elasticsearch:9200"]
      index => "logs-%{service}-%{+YYYY.MM.dd}"
    }
  }
```

### Filebeat (Lightweight Log Shipper)

```
What it does: Lightweight agent that reads log files and ships to Logstash/Elasticsearch

Why not ship directly from app to Elasticsearch?
  - Applications shouldn't have network dependency on ELK
  - If Elasticsearch is down, Filebeat buffers locally (no log loss)
  - Minimal CPU/memory footprint (unlike Logstash agent)

Deployment: one Filebeat per host/pod (DaemonSet in K8s)

filebeat.yml:
  filebeat.inputs:
    - type: container
      paths:
        - /var/log/containers/*.log
      processors:
        - add_kubernetes_metadata: ~

  output.kafka:
    hosts: ["kafka:9092"]
    topic: "app-logs"
```

### Kibana (Visualization + Search UI)

```
What it provides:
  1. Discover: free-text search across all logs
  2. Dashboards: real-time charts (error rates, latency percentiles)
  3. Alerts: notify Slack/PagerDuty when error rate > threshold
  4. Lens: drag-and-drop visualization builder
  5. APM: application performance monitoring (if using Elastic APM)

Common queries:
  - "Find all errors for trace abc-123": trace_id:"abc-123" AND level:ERROR
  - "Show 500 errors in last hour": status:500 AND @timestamp > now-1h
  - "Payment timeouts": service:payment AND message:*timeout*
```

---

## 🔄 Data Pipeline

```
RECOMMENDED PRODUCTION PIPELINE:

  [App: structured JSON logs to stdout]
       │
       ▼
  [Filebeat DaemonSet] (collects container logs)
       │
       ▼
  [Kafka] (buffer — absorbs bursts, prevents data loss)
       │
       ▼
  [Logstash] (parse, transform, enrich, filter PII)
       │
       ▼
  [Elasticsearch] (index and store)
       │
       ▼
  [Kibana] (search, visualize, alert)

WHY KAFKA IN THE MIDDLE?
  Without Kafka:
    Log burst (10x normal) → Logstash overwhelmed → drops logs or crashes ES
  
  With Kafka:
    Log burst → Kafka absorbs (disk-backed, infinite buffer)
    Logstash consumes at its own pace → steady ingestion to ES
    Result: zero log loss, even during traffic spikes
```

---

## 📂 Index Management

```
INDEX LIFECYCLE MANAGEMENT (ILM):
  Problem: 200K logs/sec × 86,400 sec/day = 17 billion logs/day
           At 1KB/log = 17TB/day of raw log data
           Can't store forever!

  Solution: Tiered storage with automatic rollover

  Hot (0-3 days): SSD nodes, fast search, recent logs
    └── logs-2024-01-15 (today)
    └── logs-2024-01-14 (yesterday)
    └── logs-2024-01-13

  Warm (3-30 days): HDD nodes, slower but cheaper
    └── logs-2024-01-12
    └── logs-2024-01-10
    └── ... (force-merged, read-only, compressed)

  Cold (30-90 days): frozen/archive, very slow queries
    └── logs-2024-12-* (searchable snapshots on S3)

  Delete (>90 days): automatically purged

  ILM Policy:
    PUT _ilm/policy/log-lifecycle
    {
      "phases": {
        "hot": { "actions": { "rollover": { "max_size": "50GB", "max_age": "1d" } } },
        "warm": { "min_age": "3d", "actions": { "shrink": { "number_of_shards": 1 } } },
        "cold": { "min_age": "30d", "actions": { "searchable_snapshot": {} } },
        "delete": { "min_age": "90d", "actions": { "delete": {} } }
      }
    }
```

---

## 💻 Spring Boot Integration

```java
// 1. Structured JSON logging (Logback + logstash-encoder)
// pom.xml dependency:
// <dependency>
//   <groupId>net.logstash.logback</groupId>
//   <artifactId>logstash-logback-encoder</artifactId>
//   <version>7.4</version>
// </dependency>

// logback-spring.xml
<configuration>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <customFields>{"service":"order-service","env":"production"}</customFields>
            <fieldNames>
                <timestamp>@timestamp</timestamp>
                <version>[ignore]</version>
            </fieldNames>
        </encoder>
    </appender>
    
    <root level="INFO">
        <appender-ref ref="STDOUT" />
    </root>
</configuration>

// Output (structured JSON, ready for ELK):
// {"@timestamp":"2024-01-15T10:30:00.123Z","level":"ERROR","logger_name":"OrderService",
//  "message":"Payment failed","service":"order-service","trace_id":"abc-123",
//  "span_id":"def-456","exception":"java.net.SocketTimeoutException..."}

// 2. Add trace context to all logs (MDC)
@Component
public class TraceFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain chain) throws Exception {
        try {
            MDC.put("trace_id", request.getHeader("X-Trace-Id"));
            MDC.put("user_id", extractUserId(request));
            MDC.put("request_path", request.getRequestURI());
            chain.doFilter(request, response);
        } finally {
            MDC.clear();
        }
    }
}

// 3. Structured exception logging
@ControllerAdvice
public class GlobalExceptionHandler {
    private static final Logger log = LoggerFactory.getLogger(GlobalExceptionHandler.class);
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleException(Exception ex, HttpServletRequest req) {
        log.error("Unhandled exception on {} {}",
            req.getMethod(), req.getRequestURI(),
            StructuredArguments.kv("status", 500),
            StructuredArguments.kv("error_type", ex.getClass().getSimpleName()),
            ex);  // full stack trace included automatically
        
        return ResponseEntity.status(500).body(new ErrorResponse("Internal error"));
    }
}
```

---

## 📈 Scaling ELK

```
SCALING CHALLENGES:
  - 200K events/sec ingestion
  - 17TB/day new data
  - Sub-second search across billions of documents

ELASTICSEARCH CLUSTER SIZING:

  Small (startup): 3 nodes, 16GB RAM, 1TB SSD each
    - Handles: ~5K events/sec, 30-day retention
    
  Medium (scale-up): 9 nodes (3 master + 6 data), 64GB RAM, 4TB SSD
    - Handles: ~50K events/sec, 30-day retention
    
  Large (enterprise): 30+ nodes (3 master + 27 data), 128GB RAM, 8TB NVMe
    - Handles: ~200K events/sec, 90-day hot + cold tier retention

SCALING TECHNIQUES:
  1. Shard sizing: aim for 30-50GB per shard (sweet spot)
  2. Index-per-day: easy to delete old data (drop entire index)
  3. Hot-warm-cold architecture: expensive SSDs only for recent data
  4. Searchable snapshots: cold data on S3, loaded on-demand
  5. Data streams: automatic index rollover when size/time threshold hit

ALTERNATIVES (when ELK becomes too expensive):
  - Grafana Loki: like ELK but only indexes metadata (10x cheaper)
  - ClickHouse: columnar DB optimized for log analytics
  - AWS OpenSearch: managed Elasticsearch (operational burden reduced)
  - Datadog/Splunk: fully managed SaaS (most expensive, zero ops)
```

---

## ⚠️ Common Pitfalls

1. **No Kafka buffer** — Direct Filebeat → Elasticsearch works for small scale, but a traffic spike (10x normal) overwhelms ES and causes log loss or cluster instability. Always buffer through Kafka in production.

2. **Too many shards** — Each index defaults to 1 primary shard. With index-per-service-per-day (200 services × 30 days = 6000 indices × shards), the cluster collapses under shard overhead. Use index templates with appropriate shard counts, and rollover policies.

3. **Not filtering sensitive data** — Logs often contain PII (emails, IPs, session tokens). Without Logstash filters to redact sensitive fields, your log cluster becomes a compliance liability. Implement PII detection and masking in the Logstash pipeline.

4. **Searching unindexed fields** — Querying a field that isn't mapped (keyword/text) forces Elasticsearch to do expensive full-scan aggregations. Define index mappings explicitly for fields you query frequently (level, service, trace_id, status_code).

5. **Ignoring index lifecycle** — Without ILM, indices grow forever until disk fills up and the cluster goes red (read-only). Configure retention policies from day one: hot (3d) → warm (30d) → delete (90d).

---

## 🧩 Mini Challenge

**Your ELK cluster handles 50K events/sec normally. During Black Friday, traffic spikes to 500K events/sec. Elasticsearch starts rejecting writes (bulk queue full). How do you prevent log loss?**

<details>
<summary>💡 Click to reveal answer</summary>

**Immediate fix (prevent loss)**: Kafka buffer between Filebeat and Logstash

```
Normal flow: Filebeat → Kafka → Logstash → Elasticsearch

During spike:
  Filebeat pushes 500K events/sec → Kafka absorbs ALL of them (disk-backed)
  Logstash pulls at 50K/sec (ES capacity) → Kafka queue grows
  After spike ends: Logstash drains the backlog (catches up in hours)
  Result: ZERO log loss, just delayed indexing during spike
```

**Medium-term (handle the load)**:
1. **Scale Elasticsearch data nodes** (auto-scaling group): add 5 more data nodes for Black Friday
2. **Reduce indexing overhead**: disable replicas during bulk ingestion, re-enable after spike
3. **Sampling**: during extreme spikes, sample DEBUG/INFO logs at 10% (keep 100% of WARN/ERROR)
4. **Separate indices by priority**: critical logs (ERROR) go to a dedicated ES cluster that never gets overwhelmed; INFO/DEBUG go to best-effort cluster

**Long-term (architecture)**:
- Use Kafka with 72h retention as the "source of truth" for logs
- ES is just one consumer (for search/dashboards)
- Can replay from Kafka if ES needs reindexing
- Add a second consumer: stream logs to S3 for long-term archive (cheap, infinite)

</details>

---

## 📝 Interview Q&A

**Q: Why use ELK instead of just storing logs in a database?**
> A: Traditional databases (PostgreSQL, MySQL) are optimized for transactional workloads (ACID, joins, small reads/writes). Logs have completely different access patterns: (1) **Write-heavy** — 200K inserts/sec with no updates. (2) **Full-text search** — need to search "timeout" across billions of entries. (3) **Time-series** — always query by time range + filter. (4) **Aggregations** — count errors per service per minute. Elasticsearch's inverted index makes full-text search O(1) instead of O(n). Its columnar aggregation engine handles time-series analytics. And its shard-based architecture scales horizontally for ingestion. A relational DB would collapse under this workload pattern.

**Q: How would you reduce ELK costs for a large deployment?**
> A: (1) **Hot-warm-cold tiers**: expensive NVMe SSDs only for 3-day hot data; HDD for 30-day warm; S3 searchable snapshots for 90-day cold. (2) **Log level filtering**: don't index DEBUG logs in production (they're 70% of volume). Ship DEBUG to S3 only (for replay if needed). (3) **Sampling**: for high-volume services, index only 10% of INFO logs (100% of ERROR/WARN). (4) **Field reduction**: strip verbose fields (full stack traces stored as hash → lookup separately). (5) **Consider Loki**: Grafana Loki indexes only labels (not full text), reducing storage by 10x. Trade-off: can't full-text search, but label-based filtering covers 80% of use cases.

---

## 🔗 What to Read Next

1. **[Observability/Logging_Best_Practices.md](./Logging_Best_Practices.md)** — What to log and how to structure it (feeds into ELK)
2. **[Observability/Prometheus_Grafana.md](./Prometheus_Grafana.md)** — Metrics pipeline (complements ELK for monitoring)
3. **[Observability/OpenTelemetry.md](./OpenTelemetry.md)** — Unified observability framework (logs + metrics + traces)

---

*[← Alerting SLO/SLA/SLI](./Alerting_SLO_SLA_SLI.md) | [Back to Observability](./README.md) | [Next: Prometheus & Grafana →](./Prometheus_Grafana.md)*
