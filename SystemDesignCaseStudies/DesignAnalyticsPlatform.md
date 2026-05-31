# 📊 Design an Analytics Platform (Real-Time + Batch)

> *"Every click, every scroll, every purchase generates an event. Netflix knows exactly when you paused a show. Spotify knows which song you skipped at 0:03. Amazon knows what you almost bought but didn't. This isn't magic — it's an analytics platform ingesting billions of events, processing them in real-time AND batch, and serving insights to dashboards in milliseconds. This design combines streaming architectures, data lakes, OLAP databases, and the Lambda/Kappa architectures."*

**⏱️ Estimated Time**: 40 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Message Queues](../BuildingBlocks/MessageQueues.md), [Batch vs Stream](../Tradeoffs/Batch_vs_Stream_Processing.md), [Database Types](../Database/Database_Types.md)

---

## 📋 Table of Contents
1. [Requirements](#-requirements)
2. [Lambda vs Kappa Architecture](#-lambda-vs-kappa-architecture)
3. [High-Level Architecture](#-high-level-architecture)
4. [Data Ingestion](#-data-ingestion)
5. [Stream Processing](#-stream-processing)
6. [Batch Processing](#-batch-processing)
7. [Storage & Query Layer](#-storage--query-layer)
8. [Java Implementation](#-java-implementation)
9. [Interview Q&A](#-interview-qa)

---

## 📝 Requirements

```
FUNCTIONAL:
  • Ingest events from web, mobile, backend services (clicks, views, purchases)
  • Real-time dashboards (live metrics: DAU, revenue, errors!)
  • Batch analytics (daily reports, trends, ML training data!)
  • Custom event queries (ad-hoc SQL over event data!)
  • Alerting (error rate spikes, anomaly detection!)
  • User-level analytics (funnel analysis, cohorts!)
  • A/B test measurement (statistical significance!)
  
NON-FUNCTIONAL:
  • Ingestion: 1M events/second sustained, 10M burst!
  • Real-time latency: < 30 seconds from event to dashboard!
  • Batch: daily jobs complete within 4 hours!
  • Query latency: < 5 seconds for dashboard queries
  • Retention: raw events for 1 year, aggregated forever!
  • Accuracy: batch = exact, real-time = approximate (OK!)

DATA SCALE:
  • 1M events/second = 86B events/day!
  • Event size: ~500 bytes average
  • Daily ingestion: ~43 TB raw data!
  • 1 year retention: ~15 PB!
```

---

## ⚖️ Lambda vs Kappa Architecture

```
LAMBDA ARCHITECTURE (traditional, proven!):
  ┌─────────────────────────────────────────────────────────────────┐
  │                                                                  │
  │  Events ──┬──► [BATCH LAYER] ──► Batch Views (exact!)           │
  │           │    (Spark, daily)    (data warehouse!)               │
  │           │                                                      │
  │           └──► [SPEED LAYER] ──► Real-time Views (approximate!) │
  │                (Flink, live)     (Redis counters!)               │
  │                                                                  │
  │  Query = merge(batch_view, speed_view)                          │
  │                                                                  │
  │  ✅ Exact batch results (re-process entire dataset!)             │
  │  ✅ Real-time for dashboards                                     │
  │  ❌ Maintain TWO code paths (batch + stream!)                   │
  │  ❌ Complexity: merging batch + speed results!                   │
  └─────────────────────────────────────────────────────────────────┘

KAPPA ARCHITECTURE (modern, streaming-first!):
  ┌─────────────────────────────────────────────────────────────────┐
  │                                                                  │
  │  Events ──► [STREAM LAYER ONLY] ──► Views                       │
  │             (Kafka + Flink)         (both real-time & batch!)    │
  │                                                                  │
  │  "Batch" = replay the stream from beginning!                    │
  │  Need to recompute? Replay Kafka from offset 0!                 │
  │                                                                  │
  │  ✅ Single code path (simpler!)                                  │
  │  ✅ Lower operational complexity                                 │
  │  ❌ Replay takes time (re-process ALL events for correction!)   │
  │  ❌ Kafka retention must be long enough! (or archived!)         │
  └─────────────────────────────────────────────────────────────────┘

WHEN TO USE WHICH:
  Lambda: When you need guaranteed accuracy for historical data
          (financial reports, compliance!) AND real-time monitoring.
  Kappa:  When streaming can handle all use cases (logs, metrics,
          user events) and occasional recomputation is acceptable.
  
  Most companies: Lambda for critical metrics + Kappa for logs/events!
```

---

## 🏗️ High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                     ANALYTICS PLATFORM                                     │
│                                                                            │
│  Data Sources           Ingestion        Processing        Serving        │
│  ┌─────────┐           ┌────────┐       ┌──────────┐     ┌──────────┐  │
│  │Web/Mobile│──events──►│        │       │  Stream  │────►│Real-time │  │
│  │SDK       │           │ Kafka  │──────►│  (Flink) │     │Views     │  │
│  └─────────┘           │        │       │  agg in  │     │(Redis/   │  │
│  ┌─────────┐           │  (100  │       │  30s!    │     │ ClickHouse)│ │
│  │Backend  │──events──►│  parti-│       └──────────┘     └──────────┘  │
│  │Services │           │  tions)│                                        │
│  └─────────┘           │        │       ┌──────────┐     ┌──────────┐  │
│  ┌─────────┐           │        │──────►│  Batch   │────►│Batch     │  │
│  │IoT/     │──events──►│        │       │  (Spark) │     │Views     │  │
│  │Sensors  │           └────────┘       │  daily   │     │(Data     │  │
│  └─────────┘                │           │  jobs!   │     │ Lake/    │  │
│                             │           └──────────┘     │ Warehouse)│ │
│                             │                            └──────────┘  │
│                             │                                           │
│                             ▼ (sink to storage)                         │
│                      ┌──────────────┐                                   │
│                      │  Data Lake    │  (S3/HDFS — raw events!)        │
│                      │  Parquet/ORC  │  (columnar for analytics!)      │
│                      └──────────────┘                                   │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │  QUERY LAYER                                                        │ │
│  │  ┌─────────────┐  ┌─────────────┐  ┌──────────────┐              │ │
│  │  │ ClickHouse  │  │ Presto/     │  │  Grafana/    │              │ │
│  │  │ (OLAP!)     │  │ Trino       │  │  Superset    │              │ │
│  │  │ Fast aggr.  │  │ (Ad-hoc SQL │  │  (Dashboards)│              │ │
│  │  │ on columns! │  │  over lake) │  │              │              │ │
│  │  └─────────────┘  └─────────────┘  └──────────────┘              │ │
│  └────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 📥 Data Ingestion

```
INGESTION AT 1M EVENTS/SECOND:

CLIENT SDK (what websites/apps use):
  // Track event from browser
  analytics.track("product_viewed", {
    product_id: "SKU-123",
    category: "electronics",
    price: 49.99,
    user_id: "user-42",
    session_id: "sess-abc",
    timestamp: 1700000000,
    device: "mobile",
    country: "US"
  });
  
  SDK batches events (every 1 second or 100 events) → HTTP POST!

INGESTION SERVICE:
  ┌────────────────────────────────────────────────────────────┐
  │  1. RECEIVE: HTTP endpoint (or gRPC for higher throughput!) │
  │     POST /v1/events (batch of 100 events)                   │
  │                                                             │
  │  2. VALIDATE: schema check, required fields, sanitize!      │
  │     Reject invalid events (don't pollute data lake!)        │
  │                                                             │
  │  3. ENRICH: add server-side data                            │
  │     + IP → country (GeoIP lookup!)                          │
  │     + user agent → device/browser parsing                   │
  │     + server timestamp (don't trust client clock!)          │
  │                                                             │
  │  4. PUBLISH: write to Kafka topic!                          │
  │     Topic: "raw-events" (100 partitions!)                   │
  │     Key: user_id (all events for same user → same partition)│
  │                                                             │
  │  5. ACK: return 202 Accepted to client                      │
  │     (async processing! Don't block client!)                 │
  └────────────────────────────────────────────────────────────┘

KAFKA TOPICS:
  • raw-events: all events as received (100 partitions!)
  • enriched-events: after enrichment pipeline
  • aggregated-events: pre-aggregated (for dashboards!)
  • dead-letter-events: failed processing (inspect later!)
```

---

## ⚡ Stream Processing

```
REAL-TIME PROCESSING (Apache Flink / Kafka Streams):

USE CASES:
  • Live DAU counter (updated every 10 seconds!)
  • Real-time revenue dashboard
  • Error rate alerting (spike detection!)
  • Live A/B test metrics
  • Fraud detection (unusual patterns!)

WINDOWED AGGREGATIONS:
  ┌────────────────────────────────────────────────────────────────┐
  │  Tumbling Window (non-overlapping!):                            │
  │  |--- 1 min ---|--- 1 min ---|--- 1 min ---|                   │
  │  [events]       [events]       [events]                        │
  │  → count per minute (page views per minute!)                   │
  │                                                                 │
  │  Sliding Window (overlapping!):                                 │
  │  |--- 5 min ---|                                                │
  │     |--- 5 min ---|                                             │
  │        |--- 5 min ---|                                          │
  │  → moving average (smoothed metrics!)                           │
  │                                                                 │
  │  Session Window (gap-based!):                                   │
  │  [events...gap>30min...][events...gap>30min...][events...]     │
  │  → user session duration (gap = session boundary!)              │
  └────────────────────────────────────────────────────────────────┘

STREAM PROCESSING PIPELINE:
  Kafka(raw-events) 
    → Filter (valid events only!)
    → Map (enrich with user segments)
    → KeyBy (user_id or event_type)
    → Window (1 minute tumbling!)
    → Aggregate (count, sum, avg!)
    → Sink (Redis for dashboard, ClickHouse for queries!)

EXACTLY-ONCE IN STREAMING:
  Flink checkpoints: periodic snapshot of stream state!
  If failure: restart from last checkpoint → no duplicates!
  Kafka source + Flink + Kafka/DB sink = exactly-once end-to-end!
```

---

## 🗄️ Storage & Query Layer

```
STORAGE TIERS:

  HOT (real-time, < 1 hour old):
  • Redis: live counters, rate metrics
  • ClickHouse: recent events (fast OLAP queries!)
  
  WARM (1 hour - 30 days):
  • ClickHouse: detailed events, fast aggregations
  • Pre-aggregated tables (hourly/daily rollups!)
  
  COLD (30 days - 1 year):
  • Data Lake (S3): raw events in Parquet format
  • Query via Presto/Trino (slower but unlimited scale!)
  
  ARCHIVE (> 1 year):
  • Glacier/Cold storage (cheap, slow retrieval!)

CLICKHOUSE (OLAP Database — perfect for analytics!):
  WHY: Columnar storage → aggregation queries are BLAZING fast!
  
  "How many page_views per country in the last hour?"
  → ClickHouse: 200ms (scans only 'country' and 'timestamp' columns!)
  → PostgreSQL: 30 seconds (scans entire rows!)
  
  Table:
  CREATE TABLE events (
    event_date Date,
    event_time DateTime,
    event_type LowCardinality(String),
    user_id String,
    country LowCardinality(String),
    device LowCardinality(String),
    value Float64
  ) ENGINE = MergeTree()
  PARTITION BY toYYYYMM(event_date)
  ORDER BY (event_type, event_date, user_id);
  
  → Partitioned by month (drop old partitions = instant deletion!)
  → Ordered by event_type + date (fast range scans!)
  → LowCardinality: dictionary encoding for low-cardinality strings!
  
  Query speed: billions of rows aggregated in seconds! 🚀

PRE-AGGREGATION (materialized views!):
  Raw: 86B events/day (too many to query interactively!)
  
  Pre-aggregate:
  • Per-minute: page_views by country by device (60× fewer rows!)
  • Per-hour: revenue by product category
  • Per-day: DAU, WAU, MAU, retention cohorts
  
  Dashboard queries hit AGGREGATED tables (fast!).
  Deep-dive queries hit RAW tables (slower but detailed!).
```

---

## 💻 Java Implementation

### Event Ingestion Service

```java
@RestController
@RequestMapping("/v1/events")
public class EventIngestionController {
    
    @Autowired private KafkaTemplate<String, AnalyticsEvent> kafka;
    @Autowired private EventValidator validator;
    @Autowired private GeoIpService geoIpService;
    
    /**
     * Ingest batch of analytics events.
     * Returns 202 Accepted (async processing!).
     */
    @PostMapping
    public ResponseEntity<Void> ingestEvents(
            @RequestBody List<RawEvent> events,
            HttpServletRequest request) {
        
        String clientIp = request.getRemoteAddr();
        String country = geoIpService.getCountry(clientIp);
        
        for (RawEvent raw : events) {
            // Validate
            if (!validator.isValid(raw)) {
                kafka.send("dead-letter-events", raw.getUserId(), raw);
                continue;
            }
            
            // Enrich
            AnalyticsEvent enriched = AnalyticsEvent.builder()
                .eventId(UUID.randomUUID().toString())
                .eventType(raw.getEventType())
                .userId(raw.getUserId())
                .properties(raw.getProperties())
                .clientTimestamp(raw.getTimestamp())
                .serverTimestamp(Instant.now())
                .country(country)
                .device(parseDevice(request.getHeader("User-Agent")))
                .build();
            
            // Publish to Kafka (key = userId for ordering!)
            kafka.send("raw-events", enriched.getUserId(), enriched);
        }
        
        return ResponseEntity.accepted().build();
    }
}
```

### Stream Processing (Kafka Streams)

```java
@Configuration
public class StreamProcessingConfig {
    
    /**
     * Real-time event counting with tumbling windows!
     */
    @Bean
    public KStream<String, AnalyticsEvent> processEvents(
            StreamsBuilder builder) {
        
        KStream<String, AnalyticsEvent> events = builder
            .stream("raw-events", Consumed.with(Serdes.String(), eventSerde));
        
        // Count events per type per minute (tumbling window!)
        events
            .groupBy((key, event) -> event.getEventType())
            .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(1)))
            .count(Materialized.as("event-counts-per-minute"))
            .toStream()
            .foreach((windowedKey, count) -> {
                String eventType = windowedKey.key();
                long windowEnd = windowedKey.window().end();
                
                // Push to Redis for real-time dashboard!
                redis.opsForZSet().add("metrics:counts:" + eventType,
                    String.valueOf(windowEnd), count);
            });
        
        // Revenue aggregation (per minute!)
        events
            .filter((key, event) -> "purchase".equals(event.getEventType()))
            .groupBy((key, event) -> "revenue")
            .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(1)))
            .aggregate(
                () -> BigDecimal.ZERO,
                (key, event, total) -> total.add(event.getRevenue()),
                Materialized.with(Serdes.String(), bigDecimalSerde))
            .toStream()
            .foreach((windowedKey, revenue) -> {
                redis.opsForValue().set("metrics:revenue:current_minute", 
                    revenue.toString());
            });
        
        return events;
    }
}
```

---

## ❓ Interview Q&A

**Q1: How do you handle late-arriving events in stream processing?**
> Events can arrive late (mobile goes offline, network delays). Solutions: (1) Watermarks: declare "I believe all events before time T have arrived" — process window when watermark passes. Allow configurable lateness (e.g., 5 minutes). (2) Late events update previously-emitted results (retraction + correction!). (3) For dashboards: show "approximate" with asterisk, update when late events arrive. (4) For batch: re-run affected time windows daily (correct any stream approximations!). Lambda architecture handles this naturally: stream = approximate, batch = final truth!

**Q2: How would you design a funnel analysis system?**
> Track ordered sequence of events per user: view_product → add_to_cart → checkout → purchase. Implementation: (1) Session window per user (Flink!), (2) Within session: check if events match funnel steps IN ORDER, (3) Record how far each user got (drop-off point!). Storage: pre-compute funnels daily (Spark job) → store in ClickHouse: {funnel_step, date, count}. Query: "What % of product viewers reached checkout?" = (checkout_count / view_count) × 100. Real-time: approximate using stream, exact using batch!

**Q3: ClickHouse vs PostgreSQL for analytics — why the massive speed difference?**
> ClickHouse is COLUMNAR: stores each column separately on disk. For "COUNT(*) WHERE country='US'": reads ONLY the 'country' column (few bytes per row!). PostgreSQL is ROW-based: reads entire rows (hundreds of bytes per row!) even if you only need one column. For 1 billion rows: ClickHouse reads ~1 GB (country column), PostgreSQL reads ~500 GB (all columns!). Also: ClickHouse uses vectorized processing (SIMD), aggressive compression per column, and no transactional overhead (no MVCC, no locks!). Trade-off: no efficient single-row updates/deletes — append-only!

**Q4: How do you ensure exactly-once event processing?**
> End-to-end exactly-once requires: (1) Producer deduplication: idempotent Kafka producer (sequence numbers per partition), (2) Stream processing: Flink checkpoints (periodic state snapshots, replay from checkpoint on failure), (3) Sink deduplication: upsert with event_id as key (if same event_id arrives twice → overwrite with same data = idempotent!). For counters: use Flink's exactly-once sinks (2PC with Kafka/databases). For data lake: write to staging → atomically move to final location (no partial writes visible!).

---

## 🔗 Related Topics
- [Batch vs Stream Processing](../Tradeoffs/Batch_vs_Stream_Processing.md) — Core trade-off
- [Pub/Sub](../MessagingQ/PubSub.md) — Event ingestion
- [CDC](../MessagingQ/CDC.md) — Database event streaming
- [Database Types](../Database/Database_Types.md) — OLAP vs OLTP

---

*"Analytics is not about collecting data — it's about collecting the RIGHT data, processing it FAST enough to act on, and presenting it in a way that drives decisions. A petabyte of unqueried data is worthless. A single metric on a dashboard that prevents a production outage is priceless." — Data Platform Engineer* 📊
