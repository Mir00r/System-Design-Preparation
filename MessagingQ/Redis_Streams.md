# 🌊 Redis Streams: Lightweight Event Streaming Without Kafka's Complexity

> *"Need a message log with consumer groups, acknowledgments, and persistence — but don't want to deploy a Kafka cluster for a few thousand messages per second? Redis Streams gives you 80% of Kafka's features at 10% of the operational cost."*

**⏱️ Estimated Time**: 25 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Redis Deep Dive](../Database/Redis_Deep_Dive.md), [Message Queues](../BuildingBlocks/MessageQueues.md)

---

## 📋 Table of Contents
1. [What Are Redis Streams](#-what-are-redis-streams)
2. [Core Operations](#-core-operations)
3. [Consumer Groups](#-consumer-groups)
4. [vs Kafka vs Pub/Sub](#-vs-kafka-vs-pubsub)
5. [Spring Boot Integration](#-spring-boot-integration)
6. [Use Cases](#-use-cases)
7. [Common Pitfalls](#-common-pitfalls)
8. [Mini Challenge](#-mini-challenge)
9. [Interview Q&A](#-interview-qa)

---

## 🤔 What Are Redis Streams

```
┌─────────────────────────────────────────────────────────┐
│              REDIS STREAMS AT A GLANCE                   │
│                                                         │
│  Type:        Append-only log data structure            │
│  Model:       Kafka-like (persistent, consumer groups)  │
│  Persistence: Yes (follows Redis persistence config)    │
│  Ordering:    Strict (timestamp-based IDs)              │
│  Consumer:    Groups with per-consumer acknowledgment   │
│  Throughput:  100K-500K messages/sec (single node)      │
│  Retention:   Configurable (MAXLEN or MINID trimming)   │
│  Since:       Redis 5.0 (2018)                          │
└─────────────────────────────────────────────────────────┘

STREAM = Append-only log of entries (like Kafka topic with 1 partition):
  
  Stream: "orders"
  ┌──────────────────────────────────────────────────────────┐
  │ ID: 1705312800000-0  │ user_id: "u123" │ amount: 99.99  │
  │ ID: 1705312800001-0  │ user_id: "u456" │ amount: 29.99  │
  │ ID: 1705312800001-1  │ user_id: "u789" │ amount: 149.99 │
  │ ID: 1705312800002-0  │ user_id: "u123" │ amount: 49.99  │
  └──────────────────────────────────────────────────────────┘
       ↑ oldest                                    newest ↑

  Each entry has:
    - Unique ID: <timestamp_ms>-<sequence> (auto-generated or custom)
    - Fields: key-value pairs (like a hash map per entry)
```

---

## ⚡ Core Operations

```
WRITING (append to stream):
  XADD orders * user_id "u123" amount "99.99" product "laptop"
  → "1705312800000-0" (auto-generated ID)
  
  XADD orders MAXLEN ~1000 * user_id "u456" amount "29.99"
  → Keeps stream approximately 1000 entries (auto-trim old)

READING (multiple patterns):

  1. Read from beginning:
     XRANGE orders - + COUNT 10
     → first 10 entries

  2. Read from specific ID onwards:
     XRANGE orders 1705312800001-0 + COUNT 10
     → entries after that ID

  3. Read new entries (blocking — like tail -f):
     XREAD BLOCK 5000 STREAMS orders $
     → blocks up to 5s waiting for new entries

  4. Read last N entries:
     XREVRANGE orders + - COUNT 5
     → last 5 entries (newest first)

TRIMMING (retention):
  XTRIM orders MAXLEN 10000        → keep exactly 10000 entries
  XTRIM orders MAXLEN ~10000       → keep approximately 10000 (faster)
  XTRIM orders MINID 1705312800000 → delete entries older than this ID

STREAM INFO:
  XLEN orders        → number of entries in stream
  XINFO STREAM orders → detailed stream metadata
```

---

## 👥 Consumer Groups

```
CONSUMER GROUPS = Kafka-like distributed processing

  Stream: "orders"
  ┌─────────────────────────────────────────────────────────┐
  │ entry-1 │ entry-2 │ entry-3 │ entry-4 │ entry-5 │ ... │
  └─────────────────────────────────────────────────────────┘
       │         │         │         │         │
       ▼         ▼         ▼         ▼         ▼
  [Consumer Group: "payment-svc"]
       Consumer A gets: entry-1, entry-3, entry-5
       Consumer B gets: entry-2, entry-4
       (work distributed, each entry goes to ONE consumer)
  
  [Consumer Group: "analytics-svc"]  (independent group!)
       Consumer C gets: entry-1, entry-2, entry-3, entry-4, entry-5
       (sees ALL entries — groups are independent)

COMMANDS:
  -- Create consumer group (start reading from beginning)
  XGROUP CREATE orders payment-svc 0
  
  -- Create group starting from new messages only
  XGROUP CREATE orders analytics-svc $ MKSTREAM
  
  -- Consumer reads from group (gets undelivered messages)
  XREADGROUP GROUP payment-svc consumer-1 COUNT 10 BLOCK 5000 STREAMS orders >
  
  -- Acknowledge processed message
  XACK orders payment-svc 1705312800000-0
  
  -- Check pending (unacknowledged) messages
  XPENDING orders payment-svc - + 10
  
  -- Claim stuck messages (consumer died without ACK)
  XCLAIM orders payment-svc consumer-2 60000 1705312800000-0
  (claim messages idle > 60s from dead consumer)

PENDING ENTRIES LIST (PEL):
  Tracks messages delivered but not yet acknowledged
  If consumer crashes → messages stay in PEL → another consumer can XCLAIM them
  Like Kafka's "uncommitted offsets" but per-message
```

---

## ⚖️ vs Kafka vs Pub/Sub

```
┌──────────────────────────────────────────────────────────────────┐
│ Feature            │ Redis Streams     │ Kafka         │ Pub/Sub  │
├────────────────────┼───────────────────┼───────────────┼──────────┤
│ Persistence        │ ✅ (Redis AOF/RDB)│ ✅ (disk log) │ ❌ (fire │
│                    │                   │               │  & forget)│
│ Consumer groups    │ ✅                │ ✅            │ ❌        │
│ Message replay     │ ✅ (by ID)        │ ✅ (by offset)│ ❌        │
│ Ordering           │ ✅ (single stream)│ ✅ (per part.)│ ❌        │
│ Acknowledgment     │ ✅ (per message)  │ ✅ (offset)   │ ❌        │
│ Throughput/node    │ 100-500K msg/s    │ 1-5M msg/s   │ 1M+ msg/s│
│ Partitioning       │ ❌ (manual sharding)│ ✅ (native) │ N/A      │
│ Retention          │ Memory-limited    │ Disk-based    │ None     │
│ Ops complexity     │ 🟢 Low (Redis)    │ 🔴 High      │ 🟢 Low   │
│ Best for           │ Moderate scale,   │ High scale,   │ Real-time│
│                    │ already have Redis│ event sourcing│ ephemeral│
└──────────────────────────────────────────────────────────────────┘

WHEN TO CHOOSE REDIS STREAMS:
  ✅ Already running Redis (no new infrastructure)
  ✅ Moderate throughput (< 500K msg/sec)
  ✅ Need consumer groups but Kafka is overkill
  ✅ Data fits in memory (or acceptable retention limit)
  ✅ Want simplicity of Redis operations

WHEN TO CHOOSE KAFKA:
  ✅ Need millions of messages/sec
  ✅ Long retention (days/weeks of replay)
  ✅ Multiple partitions for parallel consumers
  ✅ Event sourcing (immutable event log)
  ✅ Data larger than available memory
```

---

## 💻 Spring Boot Integration

```java
// Using Spring Data Redis for Streams
@Configuration
public class RedisStreamConfig {
    
    @Bean
    public StreamMessageListenerContainer<String, MapRecord<String, String, String>> 
            streamListenerContainer(RedisConnectionFactory factory) {
        
        var options = StreamMessageListenerContainer.StreamMessageListenerContainerOptions
            .builder()
            .pollTimeout(Duration.ofSeconds(2))
            .batchSize(10)
            .targetType(MapRecord.class)
            .build();
        
        var container = StreamMessageListenerContainer.create(factory, options);
        
        // Subscribe to consumer group
        container.receive(
            Consumer.from("payment-group", "consumer-1"),
            StreamOffset.create("orders", ReadOffset.lastConsumed()),
            new OrderStreamListener()
        );
        
        container.start();
        return container;
    }
}

// Producer
@Service
public class OrderStreamProducer {
    private final StringRedisTemplate redisTemplate;
    
    public String publishOrder(Order order) {
        Map<String, String> fields = Map.of(
            "order_id", order.getId(),
            "user_id", order.getUserId(),
            "amount", order.getAmount().toString(),
            "status", "CREATED"
        );
        
        // Add to stream with auto-generated ID, keep last 100K entries
        StringRecord record = StreamRecords.string(fields).withStreamKey("orders");
        RecordId id = redisTemplate.opsForStream().add(record);
        
        // Trim to keep memory bounded
        redisTemplate.opsForStream().trim("orders", 100000, true);
        
        return id.getValue();
    }
}

// Consumer (with acknowledgment)
@Component
public class OrderStreamListener implements StreamListener<String, MapRecord<String, String, String>> {
    
    private final StringRedisTemplate redisTemplate;
    
    @Override
    public void onMessage(MapRecord<String, String, String> message) {
        try {
            Map<String, String> fields = message.getValue();
            String orderId = fields.get("order_id");
            
            // Process the order
            paymentService.processPayment(orderId, new BigDecimal(fields.get("amount")));
            
            // Acknowledge successful processing
            redisTemplate.opsForStream()
                .acknowledge("orders", "payment-group", message.getId());
                
        } catch (Exception e) {
            log.error("Failed to process order: {}", message.getId(), e);
            // Don't acknowledge — message stays in PEL for retry/claim
        }
    }
}

// Dead letter handling: claim old pending messages
@Scheduled(fixedDelay = 60000)
public void claimStuckMessages() {
    // Find messages pending > 5 minutes (consumer probably dead)
    PendingMessages pending = redisTemplate.opsForStream()
        .pending("orders", "payment-group", Range.unbounded(), 100);
    
    for (PendingMessage msg : pending) {
        if (msg.getElapsedTimeSinceLastDelivery().toMinutes() > 5) {
            // Claim to this consumer and reprocess
            redisTemplate.opsForStream()
                .claim("orders", "payment-group", "consumer-recovery", 
                       Duration.ofMinutes(5), msg.getId());
        }
    }
}
```

---

## ⚠️ Common Pitfalls

1. **Memory exhaustion** — Redis Streams live in memory. Without `MAXLEN` trimming, streams grow until Redis OOM. Always set `MAXLEN ~N` on XADD or schedule periodic `XTRIM`. Size your Redis instance for expected stream size.

2. **Not acknowledging messages** — Unacknowledged messages stay in the Pending Entries List (PEL) forever. If consumers process but forget to XACK, the PEL grows unboundedly, causing memory pressure and making XPENDING queries slow.

3. **Single stream as bottleneck** — Redis Streams don't have native partitioning like Kafka. A single stream on one Redis node caps at ~500K msg/sec. For higher throughput, manually shard across multiple streams/keys (e.g., `orders:{hash(order_id) % 4}`).

4. **Using Redis Streams for long retention** — If you need weeks of message retention, Redis Streams is wrong (memory cost). Use Kafka for long-term retention and Redis Streams only for short-lived processing (hours to 1-2 days max).

---

## 🧩 Mini Challenge

**You have 3 microservices consuming from a Redis Stream. Consumer B crashes and doesn't recover for 10 minutes. Design the recovery mechanism to ensure no messages are lost.**

<details>
<summary>💡 Click to reveal answer</summary>

```
RECOVERY MECHANISM:

1. Consumer B crashes → its pending messages stay in PEL
   XPENDING orders my-group - + 10 → shows B's unacked messages

2. Monitoring service (or surviving consumers) periodically checks PEL:
   @Scheduled(fixedDelay = 30000)  // every 30s
   public void recoverStuckMessages() {
       // Find messages idle > 2 minutes (consumer B probably dead)
       PendingMessagesSummary summary = redis.opsForStream()
           .pending("orders", "my-group");
       
       // For each consumer with old pending messages:
       for (var consumerPending : summary.getConsumerPendingMessages()) {
           if (consumerPending.getConsumerName().equals(myName)) continue;
           
           // Check if consumer is alive (heartbeat in Redis key)
           Boolean alive = redis.hasKey("consumer:heartbeat:" + consumerPending.getConsumerName());
           
           if (!alive) {
               // XCLAIM: transfer messages from dead consumer to me
               List<RecordId> ids = getOldPendingIds(consumerPending.getConsumerName());
               redis.opsForStream().claim("orders", "my-group", myName,
                   Duration.ofMinutes(2), ids.toArray(new RecordId[0]));
               
               // These messages will now be delivered to me for processing
           }
       }
   }

3. When B comes back online:
   - First reads its own pending messages: XREADGROUP ... STREAMS orders 0
     (0 = read pending messages, not > which reads new only)
   - Processes and ACKs them
   - Then switches to reading new messages: XREADGROUP ... STREAMS orders >

4. Prevent double-processing:
   - Idempotent consumers (check if order already processed before acting)
   - Each message has unique ID — use as deduplication key

RESULT: No messages lost. Max delay = claim interval (30s-2min).
```

</details>

---

## 📝 Interview Q&A

**Q: Why would you use Redis Streams instead of Kafka?**
> A: Redis Streams when: (1) Already have Redis in your stack — no new infrastructure needed. (2) Moderate throughput sufficient (< 500K msg/sec). (3) Short retention (hours to days, data fits in memory). (4) Simpler operations — Redis is simpler than Kafka (no ZooKeeper/KRaft, no topic partitioning, no ISR management). (5) Need both cache AND streams in one system (session cache + event stream). Kafka when: you need millions msg/sec, multi-day retention for replay, native partitioning for parallelism, or the data exceeds memory.

---

## 🔗 What to Read Next

1. **[Database/Redis_Deep_Dive.md](../Database/Redis_Deep_Dive.md)** — Redis fundamentals
2. **[MessagingQ/Kafka.md](./Kafka.md)** — When you outgrow Redis Streams
3. **[MessagingQ/MessageQueue_Comparison.md](./MessageQueue_Comparison.md)** — Full comparison

---

*[← AWS SQS/SNS](./AWS_SQS_SNS.md) | [Back to Index](../INDEX.md) | [Next: Message Queue Comparison →](./MessageQueue_Comparison.md)*
