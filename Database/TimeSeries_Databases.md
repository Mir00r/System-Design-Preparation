# ⏰ TimeSeries Databases: Purpose-Built Storage for Time-Stamped Data

> *"When Uber ingests 1 trillion events per day from 5 million drivers or when Cloudflare monitors 25 million HTTP requests per second, generic databases crumble. TimeSeries databases are designed for one thing: efficiently storing and querying data points indexed by time."*

**⏱️ Estimated Time**: 30 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Database Selection](./SQL_Vs_NoSQL.md), [Metrics & Monitoring](../Observability/Metrics_Monitoring.md)

---

## 📋 Table of Contents
1. [What Are TimeSeries Databases](#-what-are-timeseries-databases)
2. [Key Concepts](#-key-concepts)
3. [Popular Solutions](#-popular-solutions-compared)
4. [Data Model & Schema Design](#-data-model--schema-design)
5. [Write Optimization](#-write-optimization)
6. [Query Patterns](#-query-patterns)
7. [Spring Boot Integration](#-spring-boot-integration)
8. [Common Pitfalls](#-common-pitfalls)
9. [Mini Challenge](#-mini-challenge)
10. [Interview Q&A](#-interview-qa)

---

## 🤔 What Are TimeSeries Databases

```
TIMESERIES DATA CHARACTERISTICS:
  ✅ Time-stamped (every data point has a timestamp)
  ✅ Append-only (data is written once, rarely updated)
  ✅ High write throughput (millions of points per second)
  ✅ Recent data queried most often
  ✅ Aggregation-heavy (avg, sum, percentile over time windows)
  ✅ Natural expiry (old data can be downsampled or deleted)

EXAMPLES OF TIMESERIES DATA:
  📈 Server metrics:    CPU=72%, memory=8.2GB, disk=45% at 10:30:05
  📊 Application metrics: request_count=1523, latency_p99=45ms at 10:30:00
  🌡️ IoT sensors:       temperature=23.5°C, humidity=67% at 10:30:02
  💹 Financial:         AAPL price=$185.23, volume=1.2M at 10:30:01
  🚗 Vehicle telemetry: speed=65mph, fuel=72%, location=(...) at 10:30:03

WHY NOT JUST USE PostgreSQL?
  PostgreSQL at 100K writes/sec with 1 year retention = 3 trillion rows
  - B-tree indexes grow to terabytes (slow writes)
  - No automatic downsampling (manual cron jobs)
  - No built-in retention policies
  - Range queries on time are expensive without special partitioning
  - Compression is generic (not optimized for sequential numeric data)

  TimeSeries DB at same scale:
  - 10-50x better compression (gorilla encoding, delta-of-delta)
  - Automatic downsampling (1s → 1min → 1hr over time)
  - Built-in retention (auto-delete data older than N days)
  - O(1) time-range queries (time-based partitioning)
  - Purpose-built for this exact workload
```

---

## 🔑 Key Concepts

```
TERMINOLOGY:
  Metric:       what you're measuring ("cpu_usage", "http_requests")
  Tag/Label:    metadata for filtering (host="web-01", region="us-east")
  Field/Value:  the actual measurement (72.5, 1523)
  Timestamp:    when the measurement was taken
  Series:       unique combination of metric + tags
  Point:        single measurement (metric + tags + value + timestamp)

DATA POINT STRUCTURE:
  cpu_usage{host="web-01", region="us-east"} 72.5 1705312800000
  │          │                                │     │
  metric     tags (indexed, filterable)       value timestamp (epoch ms)

TIME-BASED PARTITIONING:
  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐
  │ Jan 1-7 │ │ Jan 8-14│ │Jan 15-21│ │Jan 22-28│  ← time chunks
  └─────────┘ └─────────┘ └─────────┘ └─────────┘
  
  Query "last 24 hours" → only reads latest chunk (fast!)
  Drop old data → delete entire chunk file (O(1), no scan!)

COMPRESSION TECHNIQUES:
  Delta-of-delta (timestamps):
    Raw: 1000, 1010, 1020, 1030, 1040
    Delta: 10, 10, 10, 10
    Delta-of-delta: 0, 0, 0 → compresses to nearly nothing!

  Gorilla encoding (float values):
    XOR consecutive values → if values are similar, most bits are 0
    72.5, 72.6, 72.4, 72.5 → XOR produces very few non-zero bits
    
  Result: 10-50x compression ratio for typical metrics data
```

---

## 📊 Popular Solutions Compared

| Feature | InfluxDB | TimescaleDB | Prometheus | ClickHouse |
|---------|----------|-------------|------------|------------|
| **Type** | Purpose-built TSDB | PostgreSQL extension | Monitoring TSDB | Column-oriented OLAP |
| **Query Language** | InfluxQL / Flux | SQL (full PostgreSQL) | PromQL | SQL |
| **Best For** | IoT, metrics, events | SQL + time-series hybrid | Kubernetes monitoring | Analytics at scale |
| **Write Speed** | 1M+ points/sec | 100K-1M rows/sec | 100K samples/sec | 1M+ rows/sec |
| **Compression** | Excellent (gorilla) | Good (PostgreSQL + custom) | Excellent (gorilla) | Excellent (column) |
| **Retention** | Built-in policies | Continuous aggregates | Built-in | TTL per table |
| **Clustering** | Enterprise only | PostgreSQL replication | Federation/Thanos | Native distributed |
| **Ecosystem** | Telegraf, Grafana | Full PostgreSQL ecosystem | Grafana, AlertManager | dbt, Grafana |
| **Ideal Scale** | 10K-100K series | 10K-10M series | 1K-100K series | 1M+ series |

---

## 📐 Data Model & Schema Design

```
INFLUXDB DATA MODEL:
  // Line protocol
  weather,city=NYC,station=central temperature=72.5,humidity=65 1705312800000000000
  weather,city=SF,station=downtown temperature=58.2,humidity=78 1705312800000000000

  measurement = "weather"
  tags        = city, station (indexed, string only)
  fields      = temperature, humidity (not indexed, any type)
  timestamp   = nanosecond precision

TIMESCALEDB (SQL-based):
  CREATE TABLE metrics (
    time        TIMESTAMPTZ NOT NULL,
    host        TEXT NOT NULL,
    metric_name TEXT NOT NULL,
    value       DOUBLE PRECISION
  );
  
  -- Convert to hypertable (auto-partitions by time)
  SELECT create_hypertable('metrics', 'time', chunk_time_interval => INTERVAL '1 day');
  
  -- Add compression policy (compress chunks older than 7 days)
  ALTER TABLE metrics SET (timescaledb.compress);
  SELECT add_compression_policy('metrics', INTERVAL '7 days');
  
  -- Add retention policy (drop data older than 90 days)
  SELECT add_retention_policy('metrics', INTERVAL '90 days');

DOWNSAMPLING (reduce resolution over time):
  Raw data:     every 1 second  (keep for 24 hours)
  1-minute avg: aggregate       (keep for 7 days)
  1-hour avg:   aggregate       (keep for 1 year)
  1-day avg:    aggregate       (keep forever)

  Result: recent data = high resolution, old data = low resolution
  Storage savings: 99%+ reduction for data older than 7 days
```

---

## ⚡ Write Optimization

```
WHY TSDB WRITES ARE FAST:

  Traditional DB:
    1. Parse SQL
    2. Find row in B-tree index
    3. Random disk I/O to update page
    4. Update all secondary indexes
    → 1000-10000 writes/sec per node

  TimeSeries DB:
    1. Batch incoming points by time chunk
    2. Append to in-memory buffer (WAL for durability)
    3. Flush buffer as compressed column file when full
    4. No random I/O, no index updates
    → 100K-1M+ writes/sec per node

WRITE OPTIMIZATION PATTERNS:

  1. BATCHING (send many points at once)
     ❌ One HTTP request per data point
     ✅ Batch 1000-5000 points per request
     
  2. PRE-SORTING (sort by time before writing)
     Points arriving in time order → best compression
     Out-of-order writes → back-fill creates overhead
     
  3. APPROPRIATE PRECISION
     ❌ Nanosecond precision for temperature sensors (overkill)
     ✅ Second precision for infrastructure metrics
     
  4. CARDINALITY CONTROL
     ❌ Tag with user_id (millions of unique values)
     ✅ Tag with region, host, service (bounded cardinality)
```

---

## 💻 Spring Boot Integration

```java
// InfluxDB integration
// build.gradle: implementation 'com.influxdb:influxdb-client-java:6.10.0'

@Configuration
public class InfluxDbConfig {
    @Bean
    public InfluxDBClient influxDBClient() {
        return InfluxDBClientFactory.create(
            "http://influxdb:8086",
            "my-token".toCharArray(),
            "my-org",
            "my-bucket"
        );
    }
}

@Service
public class MetricsService {
    private final WriteApiBlocking writeApi;
    private final QueryApi queryApi;
    
    // Write metrics
    public void recordMetric(String host, double cpuUsage, double memoryUsage) {
        Point point = Point.measurement("system_metrics")
            .addTag("host", host)
            .addTag("region", "us-east-1")
            .addField("cpu_usage", cpuUsage)
            .addField("memory_usage", memoryUsage)
            .time(Instant.now(), WritePrecision.MS);
        
        writeApi.writePoint(point);
    }
    
    // Batch write for high throughput
    public void recordBatch(List<MetricPoint> points) {
        List<Point> influxPoints = points.stream()
            .map(p -> Point.measurement(p.name())
                .addTag("host", p.host())
                .addField("value", p.value())
                .time(p.timestamp(), WritePrecision.MS))
            .toList();
        
        writeApi.writePoints(influxPoints);
    }
    
    // Query: average CPU per host over last hour
    public List<HostCpuAvg> getAvgCpuLastHour() {
        String flux = """
            from(bucket: "my-bucket")
              |> range(start: -1h)
              |> filter(fn: (r) => r._measurement == "system_metrics")
              |> filter(fn: (r) => r._field == "cpu_usage")
              |> aggregateWindow(every: 5m, fn: mean)
              |> group(columns: ["host"])
            """;
        
        return queryApi.query(flux).stream()
            .flatMap(table -> table.getRecords().stream())
            .map(record -> new HostCpuAvg(
                record.getValueByKey("host").toString(),
                (Double) record.getValue()))
            .toList();
    }
}

// TimescaleDB with Spring Data JPA (just SQL!)
@Repository
public interface MetricsRepository extends JpaRepository<Metric, Long> {
    
    @Query(value = """
        SELECT time_bucket('5 minutes', time) AS bucket,
               host,
               AVG(cpu_usage) AS avg_cpu,
               MAX(cpu_usage) AS max_cpu
        FROM metrics
        WHERE time > NOW() - INTERVAL '1 hour'
        GROUP BY bucket, host
        ORDER BY bucket DESC
        """, nativeQuery = true)
    List<Object[]> getAvgCpuByHost();
}
```

---

## ⚠️ Common Pitfalls

1. **High cardinality tags** — Using user IDs, request IDs, or UUIDs as tags creates millions of time series (one per unique tag combination). This explodes memory usage and slows queries. Keep tags to bounded values (host, region, service, status_code).

2. **Querying without time bounds** — A query without a time range scans ALL data. Always include `WHERE time > X` or `range(start: -1h)`. Most TSDB can't efficiently query "all data for host X ever" without time constraints.

3. **Not setting up retention and downsampling** — Without retention policies, data grows indefinitely. A metric at 1-second resolution = 86,400 points/day = 31.5M points/year per series. Set up automatic downsampling (1s → 1m → 1h) and retention (delete raw data after N days).

4. **Using a TSDB as a general-purpose database** — TSDBs are optimized for append-only time-ordered data. They're terrible at: random updates, JOINs, complex transactions, ad-hoc queries on non-time dimensions. Keep your application state in PostgreSQL/MongoDB and use TSDB only for metrics/events.

---

## 🧩 Mini Challenge

**Your microservice emits 50 metrics per second per instance. You have 200 instances across 3 regions. Design the data ingestion pipeline and calculate storage requirements for 1 year retention with downsampling.**

<details>
<summary>💡 Click to reveal answer</summary>

**Calculations:**
```
Write rate: 50 metrics × 200 instances = 10,000 points/second
Per day: 10,000 × 86,400 = 864 million points/day
Per year (raw): 864M × 365 = 315 billion points

Storage (raw, ~16 bytes per point compressed):
  315B × 16 bytes = ~5 TB/year (raw only)

With downsampling:
  Raw (1s):   keep 7 days  → 864M × 7 = 6B points → ~96 GB
  1-min avg:  keep 30 days → 14.4M × 30 = 432M points → ~7 GB
  1-hour avg: keep 1 year  → 240K × 365 = 87.6M points → ~1.4 GB
  Total: ~105 GB (vs 5 TB raw — 98% savings!)
```

**Architecture:**
```
[200 Microservices] → [Telegraf/Agent per host]
       │                      │ (batch 5s, buffer overflow to disk)
       ▼                      ▼
[Regional Kafka topic: metrics-raw]
       │
       ▼
[InfluxDB / TimescaleDB Cluster]
  ├── Hot tier (SSD): raw data, 7 days
  ├── Warm tier (HDD): 1-min aggregates, 30 days
  └── Cold tier (S3): 1-hour aggregates, 1 year

[Grafana] → queries hot tier for real-time dashboards
[Alertmanager] → queries last 5-15 minutes for alerts
```

</details>

---

## 📝 Interview Q&A

**Q: Why can't you just use PostgreSQL with a timestamp column for time-series data?**
> A: PostgreSQL works for small-scale time-series (< 1M rows/day) using table partitioning by time range. However, at scale it struggles because: (1) B-tree index updates on every INSERT slow writes; (2) no built-in compression for sequential numeric data (TSDB achieves 10-50x compression); (3) no automatic retention/downsampling; (4) VACUUM overhead on high-write tables is significant; (5) analytical queries (avg over 1M points) are slower without columnar storage. TimescaleDB solves this by extending PostgreSQL with hypertables (automatic time partitioning), native compression, continuous aggregates, and retention policies — giving you SQL familiarity with TSDB performance.

**Q: What is "high cardinality" in time-series and why is it a problem?**
> A: Cardinality = number of unique time series (unique combinations of metric name + all tag values). Example: `http_requests{method, path, status, host}` with 5 methods × 10,000 paths × 5 statuses × 200 hosts = 50 million series. Each series requires memory for metadata, active buffers, and index entries. High cardinality causes: (1) memory explosion (each series consumes ~1-4KB in memory); (2) slow queries (must scan millions of series); (3) slow compaction. Solutions: avoid unbounded tag values (no user IDs), use histograms instead of per-endpoint metrics, pre-aggregate at collection time.

---

## 🔗 What to Read Next

1. **[Observability/Prometheus_Grafana.md](../Observability/Prometheus_Grafana.md)** — Prometheus TSDB in the monitoring stack
2. **[Observability/Metrics_Monitoring.md](../Observability/Metrics_Monitoring.md)** — What metrics to collect and why
3. **[Database/Database_Selection_Guide.md](./Database_Selection_Guide.md)** — When to choose a TSDB vs other databases

---

*[← Elasticsearch Deep Dive](./Elasticsearch_Deep_Dive.md) | [Back to Database](../INDEX.md) | [Next: Database Selection Guide →](./Database_Selection_Guide.md)*
