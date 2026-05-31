# 🔄 Change Data Capture (CDC): Streaming Database Changes

> *"LinkedIn needed to synchronize data across 1,000+ microservices without tight coupling. Their solution? They built Databus (now open-sourced as Debezium works similarly) — it reads the database transaction log and publishes every INSERT, UPDATE, and DELETE as an event stream. Services subscribe to changes they care about. The database's own change log becomes the source of truth for the entire system."*

**⏱️ Estimated Time**: 30 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Pub/Sub](./PubSub.md), [Database Indexing](../Database/Indexing.md)

---

## 📋 Table of Contents
1. [What is CDC?](#-what-is-cdc)
2. [Why CDC Matters](#-why-cdc-matters)
3. [CDC Approaches](#-cdc-approaches)
4. [Log-Based CDC (The Gold Standard)](#-log-based-cdc-the-gold-standard)
5. [CDC Architecture Patterns](#-cdc-architecture-patterns)
6. [Real-World Use Cases](#-real-world-use-cases)
7. [Tools & Technologies](#-tools--technologies)
8. [Java/Spring Boot Examples](#-javaspring-boot-examples)
9. [Mini Challenge](#-mini-challenge)
10. [Interview Q&A](#-interview-qa)

---

## 🤔 What is CDC?

```
╔══════════════════════════════════════════════════════════════════╗
║  CDC = Capturing every change made to a database and streaming ║
║  those changes as events to downstream systems.                ║
║                                                                ║
║  Every INSERT, UPDATE, DELETE → becomes an event you can react  ║
║  to in real-time!                                              ║
╚══════════════════════════════════════════════════════════════════╝
```

### 🎮 The Security Camera Analogy

```
  Your database is like a building:
  
  WITHOUT CDC (asking "what changed?"):
    🕵️ Every hour, you check every room: "Anything different since last time?"
    Expensive! Slow! What if you miss something between checks?
    
  WITH CDC (security cameras):
    📷 Cameras record EVERY movement in real-time.
    Any change → immediately captured → can react instantly!
    Nothing is missed! Complete audit trail!
    
  The database's transaction log IS the security camera!
  CDC = reading that camera feed and broadcasting it.
```

---

## 💡 Why CDC Matters

```
THE PROBLEM WITHOUT CDC:

  ┌──────────┐                    ┌────────────┐
  │  Order   │ ← How does Search │  Search    │
  │  Service │   know when a     │  Service   │
  │  (MySQL) │   new order is    │ (Elastic)  │
  │          │   created?        │            │
  └──────────┘                    └────────────┘
  
  Option A: Dual-write (write to MySQL AND Elasticsearch)
    DANGEROUS! What if one write succeeds and other fails? 💀
    Data inconsistency!
    
  Option B: Polling ("SELECT * WHERE updated_at > last_check")
    Slow (polling interval delay), expensive (constant queries),
    misses deletes, requires updated_at on every table
    
  Option C: Application events (publish event after write)
    What if app crashes AFTER writing DB but BEFORE publishing event?
    Lost events! Inconsistency! 💀

THE CDC SOLUTION:
  ┌──────────┐   WAL/binlog    ┌───────┐   events    ┌────────────┐
  │  Order   │────────────────►│  CDC  │────────────►│  Search    │
  │  Service │                 │(Debezium)│           │ (Elastic)  │
  │  (MySQL) │                 └───────┘             └────────────┘
  └──────────┘
  
  CDC reads the database's OWN transaction log (WAL/binlog).
  If it's in the DB → it's in the event stream. GUARANTEED!
  No dual-write risk! No lost events! No polling!
```

---

## 🔧 CDC Approaches

```
┌──────────────────┬────────────────────────────────────────────────┐
│  Approach        │  How It Works                                  │
├──────────────────┼────────────────────────────────────────────────┤
│  Log-based ⭐    │  Read database transaction log (WAL/binlog)    │
│                  │  Zero impact on source DB. Real-time. Complete!│
├──────────────────┼────────────────────────────────────────────────┤
│  Trigger-based   │  DB triggers fire on INSERT/UPDATE/DELETE      │
│                  │  Write changes to shadow table. Reliable but   │
│                  │  adds write overhead (2x writes!)              │
├──────────────────┼────────────────────────────────────────────────┤
│  Polling-based   │  Periodically query: WHERE updated > last_run │
│                  │  Simple but slow, misses deletes, requires     │
│                  │  timestamp columns everywhere                  │
├──────────────────┼────────────────────────────────────────────────┤
│  Dual-write      │  App writes to DB + event stream simultaneously│
│                  │  DANGEROUS! Race conditions! Avoid! ❌         │
└──────────────────┴────────────────────────────────────────────────┘

WHY LOG-BASED IS THE GOLD STANDARD:
  ✅ Zero overhead on source database (reads existing log)
  ✅ Captures ALL changes (including deletes!)
  ✅ Real-time (milliseconds delay)
  ✅ Correct ordering (follows transaction order)
  ✅ Complete (every committed transaction captured)
```

---

## 📜 Log-Based CDC (The Gold Standard)

```
Every database maintains a transaction log for crash recovery:
  • PostgreSQL: WAL (Write-Ahead Log)
  • MySQL: Binary Log (binlog)
  • MongoDB: Oplog (Operations Log)
  • SQL Server: Transaction Log

CDC READS THIS LOG:

  MySQL binlog entries:
  ┌──────────────────────────────────────────────────────────────┐
  │  Position │ Type   │ Table   │ Data                          │
  ├──────────────────────────────────────────────────────────────┤
  │  1001     │ INSERT │ orders  │ {id:1, user:"alice", amt:99}  │
  │  1002     │ UPDATE │ orders  │ {id:1, status:"shipped"}      │
  │  1003     │ DELETE │ users   │ {id:42}                       │
  └──────────────────────────────────────────────────────────────┘
  
  CDC connector (Debezium) reads position 1001, 1002, 1003...
  Converts each to a Kafka event:
  
  Topic: "dbserver1.orders"
  {
    "op": "c",              // c=create, u=update, d=delete
    "before": null,         // previous state (null for inserts)
    "after": {              // new state
      "id": 1,
      "user_id": "alice",
      "amount": 99.00,
      "status": "pending"
    },
    "source": {
      "db": "ecommerce",
      "table": "orders",
      "pos": 1001,
      "ts_ms": 1706123456000
    }
  }
```

---

## 🏗️ CDC Architecture Patterns

### Pattern 1: Database Sync (Replicate to another store)
```
  MySQL ──CDC──► Kafka ──► Elasticsearch
  
  Use case: Real-time search indexing
  Every INSERT/UPDATE/DELETE in MySQL → immediately reflected in ES!
```

### Pattern 2: Event Sourcing Bridge
```
  Legacy DB (CRUD) ──CDC──► Event Stream (Kafka)
  
  Use case: Migrate from CRUD to event-driven WITHOUT rewriting!
  Existing app writes to DB as before (no changes needed!)
  CDC captures changes and publishes events for new services!
```

### Pattern 3: Outbox Pattern
```
  ┌────────────────────────────────────────┐
  │  Transaction:                           │
  │    INSERT INTO orders VALUES (...)      │
  │    INSERT INTO outbox VALUES (event...) │ ← In SAME transaction!
  │  COMMIT;                                │
  └────────────────────────────────────────┘
  
  CDC reads outbox table → publishes to Kafka → marks as published
  
  GUARANTEE: If order exists in DB, event exists in outbox.
  Both in same transaction = atomic! No lost events!
```

### Pattern 4: Cache Invalidation
```
  DB ──CDC──► Cache Invalidator ──► Delete from Redis
  
  Use case: Automatic cache invalidation without TTL!
  When DB row changes → immediately remove stale cache entry.
  Next read hits DB → populates fresh cache.
```

---

## 🏢 Real-World Use Cases

| Company | CDC Use Case |
|---------|-------------|
| LinkedIn | Cross-service data synchronization (Databus → Brooklin) |
| Netflix | Database replication across regions |
| Uber | Event sourcing from existing databases |
| Airbnb | Real-time search index updates |
| Pinterest | Analytics pipeline from MySQL |
| Shopify | Order processing event streams |
| Wepay | Fraud detection from payment DB changes |

---

## 🛠️ Tools & Technologies

```
DEBEZIUM (most popular open-source CDC):
  Supports: MySQL, PostgreSQL, MongoDB, SQL Server, Oracle, Cassandra
  Runs as: Kafka Connect connector
  Output: Kafka topics (one per table)
  
  Architecture:
  ┌──────────┐       ┌───────────────────┐       ┌──────────┐
  │  Source   │──────►│  Debezium         │──────►│  Kafka   │
  │  Database │ binlog│  (Kafka Connect)  │ events│          │
  └──────────┘       └───────────────────┘       └──────────┘

OTHER TOOLS:
  • Maxwell's Daemon — MySQL only, simpler than Debezium
  • AWS DMS (Database Migration Service) — AWS managed CDC
  • Google Datastream — GCP managed CDC
  • Striim — Enterprise CDC with transformations
  • Confluent CDC Connectors — Managed Debezium
```

---

## 💻 Java/Spring Boot Examples

### Debezium Embedded (In-App CDC)

```java
@Configuration
public class CdcConfig {
    
    @Bean
    public io.debezium.engine.DebeziumEngine<?> debeziumEngine() {
        Properties props = new Properties();
        props.setProperty("name", "order-cdc-engine");
        props.setProperty("connector.class", "io.debezium.connector.mysql.MySqlConnector");
        props.setProperty("database.hostname", "localhost");
        props.setProperty("database.port", "3306");
        props.setProperty("database.user", "cdc_user");
        props.setProperty("database.password", "***");
        props.setProperty("database.server.id", "1");
        props.setProperty("database.include.list", "ecommerce");
        props.setProperty("table.include.list", "ecommerce.orders");
        props.setProperty("topic.prefix", "dbserver1");
        
        return DebeziumEngine.create(Json.class)
            .using(props)
            .notifying(this::handleChangeEvent)
            .build();
    }
    
    private void handleChangeEvent(RecordChangeEvent<SourceRecord> event) {
        SourceRecord record = event.record();
        Struct value = (Struct) record.value();
        
        String operation = value.getString("op"); // c, u, d, r
        Struct after = value.getStruct("after");
        Struct before = value.getStruct("before");
        
        switch (operation) {
            case "c" -> handleInsert(after);
            case "u" -> handleUpdate(before, after);
            case "d" -> handleDelete(before);
        }
    }
    
    private void handleInsert(Struct after) {
        Long orderId = after.getInt64("id");
        String status = after.getString("status");
        log.info("New order created: {} with status {}", orderId, status);
        
        // Update search index
        searchService.indexOrder(orderId);
        // Invalidate cache
        cacheService.evict("order:" + orderId);
    }
}
```

### Outbox Pattern Implementation

```java
@Service
@Transactional
public class OrderService {
    
    @Autowired private OrderRepository orderRepo;
    @Autowired private OutboxRepository outboxRepo;
    
    public Order createOrder(CreateOrderRequest request) {
        // 1. Create order in same transaction as outbox event
        Order order = new Order(request);
        order = orderRepo.save(order);
        
        // 2. Write event to outbox table (SAME TRANSACTION!)
        OutboxEvent event = OutboxEvent.builder()
            .aggregateType("Order")
            .aggregateId(order.getId().toString())
            .eventType("OrderCreated")
            .payload(objectMapper.writeValueAsString(
                new OrderCreatedEvent(order.getId(), order.getTotal())))
            .build();
        outboxRepo.save(event);
        
        // Both committed atomically! If one fails, both rollback!
        return order;
    }
}

// Debezium CDC reads outbox table → publishes to Kafka
// Then marks events as published (or deletes them)

@Entity
@Table(name = "outbox_events")
public class OutboxEvent {
    @Id @GeneratedValue private Long id;
    private String aggregateType;
    private String aggregateId;
    private String eventType;
    @Column(columnDefinition = "TEXT")
    private String payload;
    private Instant createdAt = Instant.now();
}
```

---

## 🎮 Mini Challenge

### 🧩 Design: Real-time Analytics from an E-commerce Database

Your MySQL database has tables: orders, products, users, payments. Design a CDC pipeline that:
- Updates Elasticsearch search index in real-time (< 5s delay)
- Feeds a real-time analytics dashboard
- Maintains a denormalized read model in Redis
- Must handle 10K transactions/second
- Must survive CDC connector restart without data loss

<details>
<summary>🔑 Answer</summary>

**Architecture:**
```
MySQL (binlog) → Debezium → Kafka → Consumers
```

**Topic per table:**
- `cdc.ecommerce.orders` → Order changes
- `cdc.ecommerce.products` → Product changes
- `cdc.ecommerce.payments` → Payment changes

**Consumers (separate consumer groups):**
1. **ES Indexer** — Subscribes to orders + products. On change: upsert/delete in Elasticsearch. Uses order_id as ES document ID (natural idempotency).
2. **Analytics Aggregator** — Subscribes to orders + payments. Maintains running counters in Redis (revenue, order count, conversion rate). Uses Kafka Streams for windowed aggregations.
3. **Read Model Builder** — Subscribes to all tables. Builds denormalized views in Redis (e.g., order + user + products combined document for API reads).

**Restart safety:**
- Debezium stores its position (binlog offset) in Kafka's internal topic (`connect-offsets`). On restart: resumes from last committed offset. No data loss!
- Each consumer commits Kafka offsets after processing. On consumer restart: replays from last committed offset.

**10K TPS:**
- Kafka topic with 12 partitions (parallelism for consumers)
- Each consumer group: 4-6 instances (3 partitions each)
- Debezium handles 10K TPS easily (just reading binlog)
</details>

---

## ❓ Interview Q&A

**Q1: What is Change Data Capture and why is it important?**
> CDC captures every data change (insert, update, delete) from a database and streams it as events. Important because it enables real-time data synchronization, event-driven architectures, and data pipelines WITHOUT dual-writes (which risk inconsistency) or polling (which is slow and expensive). It uses the database's own transaction log, so nothing is missed.

**Q2: Why is log-based CDC preferred over trigger-based or polling?**
> Log-based: (1) Zero overhead on source DB (reads existing WAL/binlog), (2) Captures ALL changes including deletes, (3) Real-time (ms latency), (4) Preserves transaction ordering. Triggers: add write overhead (2x writes per operation), can be complex. Polling: misses deletes, requires timestamp columns, has interval delay, and adds load to source DB.

**Q3: Explain the Outbox Pattern and why it's used with CDC.**
> Problem: You need to write to a DB AND publish an event atomically. If you do both separately, one can fail leaving inconsistency. Solution: Write the event to an "outbox" table in the SAME database transaction as your business data. CDC reads the outbox table and publishes events to Kafka. Since both the data and event are in one transaction: if committed → event will definitely be published. Guaranteed consistency!

**Q4: How does Debezium handle connector restarts?**
> Debezium stores its position in the database's transaction log (binlog offset, WAL LSN) in Kafka's internal topic. On restart: reads last stored position, resumes reading the log from that point. The database retains its log for a configurable period. As long as the connector doesn't fall behind beyond log retention, no data is lost. For safety: monitor lag and ensure binlog retention > max downtime.

**Q5: What's the difference between CDC and Event Sourcing?**
> CDC: Captures changes FROM an existing database (DB is primary, events are derived). Works with legacy CRUD applications without code changes. Event Sourcing: Events ARE the primary data model. State is derived from replaying events. Requires building the application event-first. CDC is great for brownfield (existing systems). Event Sourcing is for greenfield (new designs).

---

## 🔗 Related Topics
- [Pub/Sub](./PubSub.md) — CDC publishes to pub/sub systems
- [Event-Driven Architecture](../Architectures/Event_Driven.md) — CDC enables event-driven patterns
- [CQRS](../Architectures/CQRS.md) — CDC maintains read models
- [Database Scaling](../Database/Database_Scaling.md) — CDC for cross-DB sync

---

*"The database's transaction log is the most underrated data structure in computing. CDC just gives it the audience it deserves." — Martin Kleppmann (probably)* 🔄
