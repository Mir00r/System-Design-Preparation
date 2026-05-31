# 🔄 Batch vs Stream Processing: The Data Processing Spectrum

> *"LinkedIn processes 7 TRILLION messages per day through Kafka Streams. Meanwhile, their analytics team runs nightly batch jobs on Hadoop that take 4 hours to compute recommendations. Both are critical. The question isn't 'which is better?' — it's 'what latency can your business tolerate?' If the answer is 'milliseconds,' you need streams. If it's 'hours,' batch is simpler and cheaper."*

**⏱️ Estimated Time**: 30 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Message Queues](../BuildingBlocks/MessageQueues.md), [Event-Driven Architecture](../Architectures/Event_Driven.md)

---

## 📋 Table of Contents
1. [Batch vs Stream: The Core Difference](#-batch-vs-stream-the-core-difference)
2. [When to Use Each](#-when-to-use-each)
3. [Batch Processing Deep Dive](#-batch-processing-deep-dive)
4. [Stream Processing Deep Dive](#-stream-processing-deep-dive)
5. [Lambda Architecture](#-lambda-architecture)
6. [Kappa Architecture](#-kappa-architecture)
7. [Java Examples](#-java-examples)
8. [Technology Comparison](#-technology-comparison)
9. [Mini Challenge](#-mini-challenge)
10. [Interview Q&A](#-interview-qa)

---

## 🤔 Batch vs Stream: The Core Difference

```
╔══════════════════════════════════════════════════════════════════╗
║  BATCH: Process data in large chunks at scheduled intervals.   ║
║  STREAM: Process data one record at a time, as it arrives.     ║
║                                                                ║
║  Batch = "Let's do laundry once a week" 🧺                    ║
║  Stream = "Wash each item as soon as it's dirty" 👕            ║
╚══════════════════════════════════════════════════════════════════╝
```

### 🎮 The Analogy

```
BATCH PROCESSING (bank statement):
  📊 At end of month: process ALL transactions → generate statement
  Latency: Days to hours
  Throughput: Very high (optimized for bulk)
  
  T1 T2 T3 T4 T5 .... T1000 → [BATCH JOB] → Monthly Report

STREAM PROCESSING (fraud detection):
  🚨 EACH transaction → immediately analyzed → alert if suspicious
  Latency: Milliseconds to seconds
  Throughput: Record by record
  
  T1 → [PROCESS] → ✅
  T2 → [PROCESS] → ✅
  T3 → [PROCESS] → 🚨 FRAUD ALERT!

MICRO-BATCH (Spark Streaming):
  📦 Every 1 second: collect all events → process as tiny batch
  Latency: Seconds
  Throughput: High (mini-batches)
  
  [T1,T2,T3] → [PROCESS] → Results
  [T4,T5,T6] → [PROCESS] → Results
```

---

## 🎯 When to Use Each

```
┌──────────────────────────────────────────────────────────────────┐
│  USE BATCH WHEN:                                                 │
│  ✅ Results can wait hours/days (monthly reports, ML training)   │
│  ✅ Need to process historical data (backfill, migration)        │
│  ✅ Complex computations (joins across large datasets)           │
│  ✅ Cost optimization (run overnight on cheap compute)           │
│  ✅ Data completeness matters (wait for all data to arrive)      │
│                                                                  │
│  USE STREAM WHEN:                                                │
│  ✅ Real-time response needed (fraud, monitoring, alerts)        │
│  ✅ Continuous data flow (IoT sensors, clickstream)              │
│  ✅ Low latency required (< seconds)                            │
│  ✅ Infinite data (can't store it all, must process on-the-fly) │
│  ✅ Event-driven actions (new user → send welcome email)         │
└──────────────────────────────────────────────────────────────────┘
```

---

## 📦 Batch Processing Deep Dive

```
CHARACTERISTICS:
  • Bounded data (known start and end)
  • High throughput (optimized for bulk operations)
  • High latency (minutes to hours)
  • Fault tolerant (restart from checkpoint)
  • Simple mental model (input → transform → output)

TYPICAL FLOW:
  ┌─────────────┐    ┌──────────────┐    ┌─────────────┐
  │  Data Lake  │───►│  Batch Job   │───►│  Data       │
  │  (S3/HDFS)  │    │  (Spark/MR)  │    │  Warehouse  │
  └─────────────┘    └──────────────┘    └─────────────┘
       Input              Process             Output
     (TB of data)       (hours)           (aggregated)

EXAMPLES:
  • Daily sales report aggregation
  • Monthly billing calculation  
  • ML model training on historical data
  • ETL: Extract-Transform-Load to data warehouse
  • Nightly recommendation engine update
  • Log analysis and reporting
```

### MapReduce Pattern

```
  INPUT (1TB of web logs):
  
  MAP phase (parallelize):
    Worker 1: logs[0-250GB]  → {url: count} pairs
    Worker 2: logs[250-500GB] → {url: count} pairs
    Worker 3: logs[500-750GB] → {url: count} pairs
    Worker 4: logs[750GB-1TB] → {url: count} pairs
    
  SHUFFLE (redistribute by key):
    All "google.com" counts → same reducer
    All "amazon.com" counts → same reducer
    
  REDUCE (aggregate):
    Reducer 1: sum all "google.com" counts → 1,000,000
    Reducer 2: sum all "amazon.com" counts → 500,000
    
  OUTPUT: {google.com: 1M, amazon.com: 500K, ...}
```

---

## 🌊 Stream Processing Deep Dive

```
CHARACTERISTICS:
  • Unbounded data (never-ending stream)
  • Low latency (ms to seconds)
  • Process one record (or micro-batch) at a time
  • Ordering and time semantics matter
  • Exactly-once is HARD (at-least-once + idempotency)

TYPICAL FLOW:
  ┌──────────┐    ┌──────────────┐    ┌──────────────┐
  │  Source  │───►│  Stream      │───►│  Sink        │
  │  (Kafka) │    │  Processor   │    │  (DB/Alert)  │
  └──────────┘    └──────────────┘    └──────────────┘
  Continuous       Transform/Filter     Real-time output
  events           Aggregate/Join       

WINDOWING (how to aggregate infinite data):

  Tumbling Window (fixed, non-overlapping):
  ──[0-5min]──[5-10min]──[10-15min]──►
    count=42    count=38    count=51
    
  Sliding Window (overlapping):
  ──[0-5min]──
     ──[1-6min]──
        ──[2-7min]──►
        
  Session Window (gap-based):
  ──[events]──gap──[events]──gap──[events]──►
    session 1        session 2     session 3
```

---

## 🏗️ Lambda Architecture

```
  Combines BOTH batch and stream for best of both worlds:
  
                         ┌─────────────────────────┐
                         │    Batch Layer          │
                         │    (Hadoop/Spark)       │
  ┌──────────┐          │    Complete, accurate   │───┐
  │  All     │─────────►│    but SLOW (hours)     │   │
  │  Data    │          └─────────────────────────┘   │ MERGE
  │  Stream  │                                         ▼
  │          │          ┌─────────────────────────┐ ┌──────┐
  │          │─────────►│    Speed Layer          │►│Query │
  └──────────┘          │    (Storm/Flink)        │ │Layer │
                        │    Fast, approximate    │ └──────┘
                        │    (seconds)            │   ▲
                        └─────────────────────────┘   │
                                                       │
  Query Layer merges:                                  │
    Batch view (accurate but old) + Speed view (recent but approx)
    = Complete, relatively fresh result!

  PROS: Accuracy of batch + speed of stream
  CONS: Two codebases to maintain! Complex! 😰
```

---

## 🏗️ Kappa Architecture

```
  EVERYTHING is a stream. No separate batch layer!
  
  ┌──────────┐    ┌──────────────────────┐    ┌──────────┐
  │  Event   │───►│  Stream Processor    │───►│  Serving  │
  │  Log     │    │  (Kafka Streams/     │    │  Layer    │
  │  (Kafka) │    │   Flink)             │    │  (DB)     │
  └──────────┘    └──────────────────────┘    └──────────┘
  
  Need to reprocess? Replay the event log from beginning!
  
  Kafka retains ALL events (configurable retention).
  "Reprocessing" = deploy new consumer, replay from offset 0.
  
  PROS: Single codebase! Simpler! Events are source of truth!
  CONS: Reprocessing can be slow for huge event logs
  
  Used by: LinkedIn, Uber, most modern event-driven systems
```

---

## 💻 Java Examples

### Batch: Spring Batch Job

```java
@Configuration
@EnableBatchProcessing
public class DailySalesReportJob {
    
    @Bean
    public Job salesReportJob(JobBuilderFactory jobs, Step processStep) {
        return jobs.get("dailySalesReport")
            .start(processStep)
            .build();
    }
    
    @Bean
    public Step processStep(StepBuilderFactory steps) {
        return steps.get("processTransactions")
            .<Transaction, SalesSummary>chunk(1000)  // Process 1000 at a time
            .reader(transactionReader())
            .processor(aggregationProcessor())
            .writer(reportWriter())
            .build();
    }
    
    @Bean
    public JdbcCursorItemReader<Transaction> transactionReader() {
        return new JdbcCursorItemReaderBuilder<Transaction>()
            .name("transactionReader")
            .dataSource(dataSource)
            .sql("SELECT * FROM transactions WHERE date = CURRENT_DATE - 1")
            .rowMapper(new TransactionRowMapper())
            .build();
    }
}
```

### Stream: Kafka Streams

```java
@Configuration
public class FraudDetectionStream {
    
    @Bean
    public KStream<String, Transaction> fraudDetectionTopology(
            StreamsBuilder builder) {
        
        KStream<String, Transaction> transactions = 
            builder.stream("transactions");
        
        // Real-time fraud detection
        KStream<String, FraudAlert> alerts = transactions
            .filter((key, txn) -> txn.getAmount() > 10000)
            .mapValues(txn -> assessFraudRisk(txn))
            .filter((key, risk) -> risk.getScore() > 0.8)
            .mapValues(risk -> new FraudAlert(risk));
        
        // Output alerts in real-time
        alerts.to("fraud-alerts");
        
        // Windowed aggregation: transactions per user per 5 minutes
        transactions
            .groupByKey()
            .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(5)))
            .count()
            .toStream()
            .filter((window, count) -> count > 10)  // More than 10 txns in 5min
            .mapValues(count -> new VelocityAlert(count))
            .to("velocity-alerts");
        
        return transactions;
    }
}
```

---

## 📊 Technology Comparison

```
┌────────────────────┬───────────────┬──────────────┬──────────────┐
│  Technology        │  Type         │  Latency     │  Best For    │
├────────────────────┼───────────────┼──────────────┼──────────────┤
│  Hadoop MapReduce  │  Batch        │  Hours       │  Huge data   │
│  Apache Spark      │  Batch/Micro  │  Min-Sec     │  General     │
│  Apache Flink      │  True Stream  │  Millisec    │  Low latency │
│  Kafka Streams     │  Stream       │  Millisec    │  Simple apps │
│  Apache Storm      │  Stream       │  Millisec    │  Legacy      │
│  AWS Lambda        │  Event        │  Seconds     │  Serverless  │
│  Google Dataflow   │  Unified      │  Varies      │  Cloud native│
│  Spring Batch      │  Batch        │  Min-Hours   │  Enterprise  │
└────────────────────┴───────────────┴──────────────┴──────────────┘
```

---

## 🎮 Mini Challenge

### 🧩 Design: Real-time Analytics Dashboard

You're building a dashboard showing:
- Total sales today (accurate to the second)
- Top 10 products this hour
- Sales by region (updated every 5 minutes)
- Comparison with same day last year

Which should be batch and which should be stream?

<details>
<summary>🔑 Answer</summary>

- **Total sales today** → Stream (Kafka Streams, tumbling window of 1 day, running sum)
- **Top 10 products this hour** → Stream (sliding window of 1 hour, maintain top-K heap)
- **Sales by region every 5 min** → Stream (tumbling window of 5 minutes, group by region)
- **Same day last year** → Batch (pre-computed daily, stored in cache. This data never changes!)

Architecture: Kappa style — all sales events flow through Kafka. Stream processors compute real-time aggregates. Historical comparisons pre-computed in batch, cached in Redis.
</details>

---

## ❓ Interview Q&A

**Q1: What's the fundamental difference between batch and stream processing?**
> Batch processes bounded, finite datasets at scheduled intervals with high throughput but high latency (hours). Stream processes unbounded, continuous data one record at a time with low latency (milliseconds) but more complexity around ordering, windowing, and exactly-once guarantees.

**Q2: Explain the Lambda Architecture. What problem does it solve?**
> Lambda combines batch and stream layers. The batch layer provides complete, accurate results (but delayed). The speed layer provides real-time approximate results. The serving layer merges both. It solves the problem of needing both accuracy and timeliness, but at the cost of maintaining two separate processing pipelines.

**Q3: Why is exactly-once processing hard in stream processing?**
> Because failures can happen between processing a record and committing the offset. If you process then crash before commit: reprocessed on restart (at-least-once). If you commit then crash before processing: lost (at-most-once). Solutions: transactional processing (Kafka exactly-once), idempotent writes, or two-phase commit — all add complexity.

**Q4: What is windowing in stream processing?**
> Windows group infinite stream data into finite chunks for aggregation. Tumbling windows: fixed, non-overlapping time buckets (every 5 min). Sliding windows: overlapping (every minute, looking back 5 min). Session windows: activity-based gaps (user session = events with < 30min gap between them).

**Q5: When would you choose Kappa architecture over Lambda?**
> When you can represent ALL processing as stream processing and don't need separate batch logic. Kappa uses a single stream processing pipeline with an immutable event log (Kafka). To reprocess, replay from the beginning. Simpler than Lambda (one codebase), but requires that your event log stores enough history.

---

## 🔗 Related Topics
- [Message Queues](../BuildingBlocks/MessageQueues.md) — Stream processing infrastructure
- [Event-Driven Architecture](../Architectures/Event_Driven.md) — Architectural pattern for streams
- [CQRS](../Architectures/CQRS.md) — Often combined with event streams
- [Change Data Capture](../MessagingQ/CDC.md) — Turning DB changes into streams

---

*"Batch is for looking back. Streaming is for looking forward. Great systems do both." — Jay Kreps, Creator of Apache Kafka* 🌊

---

*Previous: [← REST vs RPC](./REST_vs_RPC.md) | Next: [Long Polling vs WebSockets →](./Long_Polling_vs_WebSockets.md)*
