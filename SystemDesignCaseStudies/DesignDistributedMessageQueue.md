# 📨 Design a Distributed Message Queue

> *"Every modern distributed system has a message queue at its core. It's the 'shock absorber' between services — decoupling producers from consumers, smoothing traffic spikes, and enabling reliable async processing. Designing one from scratch reveals deep knowledge of distributed consensus, persistence, ordering guarantees, and the fundamental trade-offs that every queue (Kafka, RabbitMQ, SQS) makes differently."*

**⏱️ Estimated Time**: 40 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Message Queues](../BuildingBlocks/MessageQueues.md), [Pub/Sub](../MessagingQ/PubSub.md), [Replication](../Database/Sharding.md)

---

## 📋 Table of Contents
1. [Requirements](#-requirements)
2. [Core Concepts](#-core-concepts)
3. [High-Level Architecture](#-high-level-architecture)
4. [Storage Engine](#-storage-engine)
5. [Ordering Guarantees](#-ordering-guarantees)
6. [Delivery Semantics](#-delivery-semantics)
7. [Replication & Fault Tolerance](#-replication--fault-tolerance)
8. [Consumer Groups](#-consumer-groups)
9. [Java Implementation](#-java-implementation)
10. [Interview Q&A](#-interview-qa)

---

## 📝 Requirements

```
FUNCTIONAL:
  • Producers publish messages to named topics
  • Consumers subscribe and receive messages
  • Consumer groups: parallel consumption with load balancing
  • Message ordering: guaranteed within a partition
  • At-least-once delivery (with exactly-once option!)
  • Message retention (configurable: 7 days default)
  • Dead letter queue (failed messages after N retries)
  
NON-FUNCTIONAL:
  • Throughput: 1M messages/second per topic!
  • Latency: < 10ms P99 for publish
  • Durability: no message loss (replicated, persisted!)
  • Availability: 99.99% (survive broker failures!)
  • Scalability: add partitions/brokers without downtime

SCALE:
  • 100K topics
  • 1M messages/second aggregate
  • Messages: 1 KB average, up to 1 MB max
  • Retention: 7 days (configurable to infinite!)
  • Consumer groups: 10K+ active groups
```

---

## 🧱 Core Concepts

```
TERMINOLOGY:
  ┌───────────────────────────────────────────────────────────────────┐
  │  Concept         │  Definition                                    │
  ├───────────────────────────────────────────────────────────────────┤
  │  Topic           │  Named category of messages (like "orders")    │
  │  Partition       │  Ordered, immutable sequence within a topic    │
  │  Message/Record  │  Key + Value + Timestamp + Headers             │
  │  Offset          │  Position of message in partition (sequential!)│
  │  Broker          │  Server that stores and serves messages        │
  │  Producer        │  Publishes messages to topics                  │
  │  Consumer        │  Reads messages from topics                    │
  │  Consumer Group  │  Set of consumers sharing partition assignment │
  │  Replication     │  Copies of partitions on different brokers     │
  └───────────────────────────────────────────────────────────────────┘

TOPIC vs QUEUE SEMANTICS:
  QUEUE: message consumed by ONE consumer → deleted!
  TOPIC: message consumed by MANY consumers independently!
         (each consumer group tracks its own offset!)

HOW IT WORKS:
  Topic "orders" has 4 partitions:
  
  ┌─────────────────────────────────────────────────────────────┐
  │  Partition 0: [msg0][msg1][msg2][msg3]... → append-only!    │
  │  Partition 1: [msg0][msg1][msg2]... → append-only!          │
  │  Partition 2: [msg0][msg1][msg2][msg3][msg4]... → offset=4  │
  │  Partition 3: [msg0][msg1]... → newest partition!            │
  └─────────────────────────────────────────────────────────────┘
  
  Messages are APPENDED (never modified/deleted until retention expires!)
  Consumers track their own offset per partition!
  
  Consumer Group A: reading partition 0 at offset=2, partition 1 at offset=1...
  Consumer Group B: reading partition 0 at offset=0 (independent!)
```

---

## 🏗️ High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DISTRIBUTED MESSAGE QUEUE                              │
│                                                                           │
│  Producers                    Broker Cluster              Consumers       │
│  ┌────────┐                  ┌─────────────────┐        ┌────────────┐  │
│  │Producer│──── Publish ────►│  Broker 1       │◄─Fetch─│Consumer A1 │  │
│  │  App1  │                  │  Partitions:    │        │(Group A)   │  │
│  └────────┘                  │  orders-P0(L)  │        └────────────┘  │
│  ┌────────┐                  │  orders-P1(F)  │        ┌────────────┐  │
│  │Producer│──── Publish ────►│  payments-P0(L)│◄─Fetch─│Consumer A2 │  │
│  │  App2  │                  └────────┬────────┘        │(Group A)   │  │
│  └────────┘                           │                 └────────────┘  │
│                              ┌────────▼────────┐        ┌────────────┐  │
│                              │  Broker 2       │◄─Fetch─│Consumer B1 │  │
│                              │  orders-P0(F)  │        │(Group B)   │  │
│                              │  orders-P1(L)  │        └────────────┘  │
│                              │  payments-P1(L)│                         │
│                              └────────┬────────┘                         │
│                                       │                                   │
│                              ┌────────▼────────┐                         │
│                              │  Broker 3       │    L = Leader           │
│                              │  orders-P2(L)  │    F = Follower         │
│                              │  payments-P0(F)│    (replication!)        │
│                              └─────────────────┘                         │
│                                                                           │
│  ┌──────────────────────────────────────────────┐                        │
│  │  Controller / Metadata Service               │                        │
│  │  (ZooKeeper / Raft-based)                     │                        │
│  │  • Broker membership                          │                        │
│  │  • Partition → Broker mapping                 │                        │
│  │  • Leader election!                           │                        │
│  │  • Consumer group coordination                │                        │
│  └──────────────────────────────────────────────┘                        │
└─────────────────────────────────────────────────────────────────────────┘

WRITE PATH (Produce):
  1. Producer hashes message key → determines partition!
     hash("order-123") % 4 = partition 2
  2. Producer sends to LEADER of partition 2 (Broker 3)
  3. Broker 3 appends to local log (WAL!)
  4. Followers (Broker 1, 2) replicate the message
  5. Once min.insync.replicas acknowledged → success!
  6. Return offset to producer (confirmation!)

READ PATH (Consume):
  1. Consumer requests: "partition 2, offset 150, max 1000 messages"
  2. Broker reads from log file (sequential I/O — FAST!)
  3. Returns batch of messages to consumer
  4. Consumer processes messages
  5. Consumer commits offset (stores progress!)
  6. On restart: resume from last committed offset!
```

---

## 💾 Storage Engine

```
THE SECRET TO HIGH THROUGHPUT: Sequential I/O!

  Messages stored in APPEND-ONLY LOG FILES (segments):
  
  Partition directory: /data/orders-P0/
  ├── 00000000000000000000.log  (segment 0: offsets 0-999)
  ├── 00000000000000001000.log  (segment 1: offsets 1000-1999)
  ├── 00000000000000002000.log  (segment 2: offsets 2000-2847) ← active!
  ├── 00000000000000000000.index (offset → file position mapping!)
  └── 00000000000000000000.timeindex (timestamp → offset mapping!)

WHY SEQUENTIAL I/O IS FAST:
  Random I/O: HDD = 100 IOPS, SSD = 100K IOPS
  Sequential I/O: HDD = 100 MB/s, SSD = 3 GB/s! 🚀
  
  Append-only = always sequential! No seeks needed!
  Reading = also sequential (consumers read in order!)
  
  Linux page cache: OS caches file data in RAM automatically!
  Warm partition → reads served from RAM (no disk I/O at all!)

MESSAGE FORMAT:
  ┌──────────────────────────────────────────────────────────────┐
  │  Field          │  Size      │  Description                   │
  ├──────────────────────────────────────────────────────────────┤
  │  Length          │  4 bytes   │  Total message size            │
  │  CRC             │  4 bytes   │  Checksum for integrity!       │
  │  Magic           │  1 byte    │  Format version                │
  │  Timestamp       │  8 bytes   │  Producer timestamp            │
  │  Key length      │  4 bytes   │  -1 if null                    │
  │  Key             │  variable  │  Partition routing key          │
  │  Value length    │  4 bytes   │  Message body size             │
  │  Value           │  variable  │  The actual message!           │
  │  Headers         │  variable  │  Key-value metadata            │
  └──────────────────────────────────────────────────────────────┘

RETENTION & COMPACTION:
  Time-based: delete segments older than 7 days!
  Size-based: delete oldest segments when topic exceeds 100 GB!
  
  Log compaction (for changelog topics):
  Keep only LATEST value per key! (like a database snapshot!)
  Key "user:42" was updated 100 times → keep only the latest!
```

---

## 📊 Ordering Guarantees

```
ORDERING IS PER-PARTITION, NOT PER-TOPIC!

  Topic with 4 partitions:
  Producer sends: M1, M2, M3, M4, M5, M6
  
  If all have same key → same partition → ORDERED! ✅
  If different keys → spread across partitions → NO GLOBAL ORDER!
  
  M1(key=A) → P0     Consumer sees P0: M1, M4 (ordered!)
  M2(key=B) → P1     Consumer sees P1: M2, M5 (ordered!)
  M3(key=C) → P2     Consumer sees P2: M3, M6 (ordered!)
  M4(key=A) → P0     But across partitions? No guarantee!
  M5(key=B) → P1
  M6(key=C) → P2

HOW TO ENSURE ORDER:
  Same entity → same key → same partition → same consumer → ORDERED!
  
  Example: "All events for order-123 must be processed in order"
  → Use "order-123" as message key!
  → hash("order-123") always routes to same partition!
  → One consumer handles that partition → processes in order!

TRADE-OFF:
  More partitions → more parallelism → higher throughput!
  But: ordering only within partition!
  
  If you need GLOBAL ordering (all messages in one sequence):
  → Use 1 partition! (but: throughput limited to single broker!)
  → Usually unnecessary — per-entity ordering is sufficient!
```

---

## 📬 Delivery Semantics

```
THREE DELIVERY GUARANTEES:

AT-MOST-ONCE (fire and forget):
  Producer sends → don't wait for ACK → might be lost!
  Consumer reads → commit offset BEFORE processing → might skip!
  
  Use case: metrics, logs (losing one data point is OK!)
  Throughput: HIGHEST (no waiting!)

AT-LEAST-ONCE (most common!):
  Producer sends → wait for ACK → retry on failure → DUPLICATES possible!
  Consumer reads → process → THEN commit offset → on crash: reprocess!
  
  Use case: most systems! (handle duplicates with idempotency!)
  Throughput: high (ACK adds small latency)

EXACTLY-ONCE (hardest!):
  Producer: idempotent writes (broker deduplicates by producer ID + sequence!)
  Consumer: transactional processing (process + commit offset atomically!)
  
  HOW EXACTLY-ONCE WORKS (Kafka):
  1. Producer assigns sequence number to each message
  2. Broker tracks: {producerId: lastSequence}
  3. If broker sees sequence ≤ lastSequence → DUPLICATE! Skip!
  4. Consumer: processes message + commits offset in same transaction!
     (Using transactional outbox or Kafka transactions!)
  
  Use case: financial transactions, inventory updates!
  Throughput: lower (transactional overhead)

IDEMPOTENT CONSUMER PATTERN:
  Even with at-least-once, make your consumer SAFE for duplicates!
  
  // Track processed message IDs
  if (redis.setnx("processed:" + messageId, "1")) {
      processMessage(message); // First time!
  } else {
      // Already processed — skip! (idempotent!)
  }
```

---

## 🔄 Replication & Fault Tolerance

```
REPLICATION FACTOR = 3 (industry standard):
  Each partition has 1 LEADER + 2 FOLLOWERS!
  
  Topic: "orders", Partition 0, RF=3:
  ┌──────────┐     ┌──────────┐     ┌──────────┐
  │ Broker 1 │     │ Broker 2 │     │ Broker 3 │
  │ P0(LEADER)│───►│ P0(follower)│   │ P0(follower)│
  │ Writes go │     │ Replicates! │   │ Replicates! │
  │ here!     │     │             │   │             │
  └──────────┘     └──────────┘     └──────────┘

IN-SYNC REPLICAS (ISR):
  ISR = set of replicas that are "caught up" with leader!
  Leader: offset 1000
  Follower A: offset 998 → in ISR (< 10s behind) ✅
  Follower B: offset 500 → NOT in ISR (too far behind!) ❌
  
  min.insync.replicas = 2:
  Write succeeds only if at least 2 replicas (including leader) have it!
  → Guarantees data survives loss of 1 broker!

LEADER ELECTION:
  Leader dies → Controller detects (heartbeat timeout!)
  → Pick new leader from ISR! (must be caught up!)
  → Update metadata → clients route to new leader!
  → Downtime: 1-5 seconds (during election!)

UNCLEAN ELECTION (data loss trade-off):
  All ISR replicas dead? Only out-of-sync replica available?
  
  unclean.leader.election = true: 
    Allow non-ISR replica to become leader!
    RISK: messages that weren't replicated are LOST!
    BENEFIT: partition is available! (availability over consistency!)
  
  unclean.leader.election = false:
    Partition UNAVAILABLE until ISR replica recovers!
    BENEFIT: no data loss! (consistency over availability!)
    RISK: downtime until recovery!
```

---

## 👥 Consumer Groups

```
CONSUMER GROUPS: Parallel consumption with load balancing!

  Topic "orders" has 4 partitions.
  Consumer Group "order-processing" has 3 consumers.
  
  ASSIGNMENT (automatic rebalancing!):
  Consumer A → Partition 0, Partition 1
  Consumer B → Partition 2
  Consumer C → Partition 3
  
  Each partition assigned to EXACTLY ONE consumer in the group!
  → No duplicate processing! (within a group)
  → Parallelism = min(partitions, consumers)!
  
  If Consumer B crashes:
  → Rebalance! Partitions redistributed among A and C!
  Consumer A → Partition 0, Partition 1, Partition 2 ← picked up B's work!
  Consumer C → Partition 3

MULTIPLE CONSUMER GROUPS (independent!):
  Group "analytics": reads ALL messages (for reporting!)
  Group "email": reads ALL messages (for notifications!)
  
  Each group tracks its own offset independently!
  Same message consumed by BOTH groups! (fan-out!)

CONSUMER LAG:
  Lag = latest offset - consumer's committed offset
  
  Partition 0: latest offset = 10,000
  Consumer committed: 9,500
  Lag: 500 messages behind!
  
  Monitor lag! If growing → consumer too slow! Scale up!
  Alert if lag > 10,000 (processing falling behind!)
```

---

## 💻 Java Implementation

### Message Broker Core

```java
/**
 * Simplified message broker partition implementation.
 * Demonstrates the core concepts of append-only log + offset tracking.
 */
public class Partition {
    
    private final String topic;
    private final int partitionId;
    private final List<Message> log = new ArrayList<>(); // Append-only!
    private final ReadWriteLock lock = new ReentrantReadWriteLock();
    private final AtomicLong nextOffset = new AtomicLong(0);
    
    /**
     * Append message to partition log.
     * Returns the assigned offset.
     */
    public long append(Message message) {
        lock.writeLock().lock();
        try {
            long offset = nextOffset.getAndIncrement();
            message.setOffset(offset);
            message.setTimestamp(System.currentTimeMillis());
            log.add(message);
            return offset;
        } finally {
            lock.writeLock().unlock();
        }
    }
    
    /**
     * Read messages starting from offset.
     * Consumer calls this to fetch a batch!
     */
    public List<Message> read(long fromOffset, int maxMessages) {
        lock.readLock().lock();
        try {
            if (fromOffset >= log.size()) return Collections.emptyList();
            
            int endIdx = (int) Math.min(fromOffset + maxMessages, log.size());
            return new ArrayList<>(log.subList((int) fromOffset, endIdx));
        } finally {
            lock.readLock().unlock();
        }
    }
    
    public long getLatestOffset() {
        return nextOffset.get() - 1;
    }
}
```

### Producer with Partitioning

```java
@Service
public class MessageProducer {
    
    private final Map<String, List<Partition>> topicPartitions;
    
    /**
     * Publish message to topic.
     * Partition determined by key hash (or round-robin if no key).
     */
    public PublishResult publish(String topic, String key, byte[] value) {
        List<Partition> partitions = topicPartitions.get(topic);
        if (partitions == null) throw new TopicNotFoundException(topic);
        
        // Determine target partition
        int partitionIdx;
        if (key != null) {
            // Key-based: same key → same partition (ordering!)
            partitionIdx = Math.abs(key.hashCode() % partitions.size());
        } else {
            // Round-robin for keyless messages
            partitionIdx = roundRobinCounter.getAndIncrement() % partitions.size();
        }
        
        Partition partition = partitions.get(partitionIdx);
        
        Message message = new Message();
        message.setKey(key);
        message.setValue(value);
        message.setId(UUID.randomUUID().toString());
        
        long offset = partition.append(message);
        
        // Replicate to followers (async for throughput!)
        replicateToFollowers(topic, partitionIdx, message);
        
        return new PublishResult(topic, partitionIdx, offset);
    }
}
```

### Consumer with Offset Tracking

```java
@Service
public class MessageConsumer {
    
    private final Map<String, Long> committedOffsets = new ConcurrentHashMap<>();
    private final RedisTemplate<String, String> redis;
    
    /**
     * Poll for new messages (long-polling!).
     */
    public List<Message> poll(String topic, int partition, 
                              String consumerGroup, int maxMessages) {
        String offsetKey = consumerGroup + ":" + topic + ":" + partition;
        
        // Get last committed offset
        long fromOffset = getCommittedOffset(offsetKey) + 1;
        
        // Fetch from partition
        Partition p = getPartition(topic, partition);
        return p.read(fromOffset, maxMessages);
    }
    
    /**
     * Commit offset after successful processing!
     * (at-least-once: commit AFTER processing!)
     */
    public void commitOffset(String topic, int partition, 
                             String consumerGroup, long offset) {
        String offsetKey = consumerGroup + ":" + topic + ":" + partition;
        redis.opsForValue().set(offsetKey, String.valueOf(offset));
        committedOffsets.put(offsetKey, offset);
    }
    
    private long getCommittedOffset(String offsetKey) {
        Long cached = committedOffsets.get(offsetKey);
        if (cached != null) return cached;
        
        String stored = redis.opsForValue().get(offsetKey);
        return stored != null ? Long.parseLong(stored) : -1;
    }
}
```

### Spring Boot Kafka Producer/Consumer

```java
// Production usage with Spring Kafka
@Service
public class OrderEventProducer {
    
    @Autowired private KafkaTemplate<String, OrderEvent> kafkaTemplate;
    
    public void publishOrderCreated(Order order) {
        OrderEvent event = new OrderEvent("CREATED", order);
        
        // Key = orderId → all events for same order go to same partition!
        kafkaTemplate.send("orders", order.getId(), event)
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    log.error("Failed to publish order event!", ex);
                    // Retry or dead letter!
                } else {
                    log.info("Published to partition {} offset {}",
                        result.getRecordMetadata().partition(),
                        result.getRecordMetadata().offset());
                }
            });
    }
}

@Service
public class OrderEventConsumer {
    
    @KafkaListener(topics = "orders", groupId = "order-processing")
    public void handleOrderEvent(
            @Payload OrderEvent event,
            @Header(KafkaHeaders.RECEIVED_PARTITION) int partition,
            @Header(KafkaHeaders.OFFSET) long offset) {
        
        // Idempotency check!
        String dedupeKey = "processed:" + event.getEventId();
        if (!redis.opsForValue().setIfAbsent(dedupeKey, "1", 
                Duration.ofDays(7))) {
            log.info("Duplicate event, skipping: {}", event.getEventId());
            return;
        }
        
        // Process the event
        switch (event.getType()) {
            case "CREATED" -> orderService.processNewOrder(event.getOrder());
            case "PAID" -> orderService.fulfillOrder(event.getOrder());
            case "CANCELLED" -> orderService.cancelOrder(event.getOrder());
        }
        
        // Offset committed automatically (enable.auto.commit=true)
        // Or manually: acknowledgment.acknowledge();
    }
}
```

---

## ❓ Interview Q&A

**Q1: How does a message queue achieve high throughput?**
> Five key techniques: (1) Sequential I/O — append-only log, no random writes (HDD: 100 MB/s sequential vs 100 IOPS random!), (2) Batching — producers batch messages before sending, brokers batch writes to disk, consumers fetch in batches, (3) Zero-copy — sendfile() syscall transfers data directly from page cache to network socket (no user-space copy!), (4) Compression — batch-level compression (messages compressed together = better ratio!), (5) Partitioning — horizontal scaling across brokers (N partitions = N× throughput!).

**Q2: How do you ensure no message loss?**
> Producer side: acks=all (wait for ALL ISR replicas to acknowledge!). Broker side: replication factor=3 + min.insync.replicas=2 (data exists on at least 2 brokers before confirming!). Consumer side: commit offset AFTER processing (at-least-once). If broker dies before replication: the un-replicated messages are lost ONLY if we don't retry. With acks=all + retries + idempotent producer: effectively no loss! Trade-off: higher latency (waiting for replication) vs lower latency (acks=1, risk of loss).

**Q3: Kafka vs RabbitMQ — when to use which?**
> Kafka: high-throughput streaming, event sourcing, log aggregation, multiple consumers reading same data independently, replay capability (retention!). RabbitMQ: traditional message queue, complex routing (exchanges, bindings), lower latency for individual messages, when you need "exactly-once processed-and-deleted" semantics. Rule of thumb: Kafka for event streaming & big data, RabbitMQ for task distribution & complex routing. Kafka stores messages (log); RabbitMQ delivers and deletes (queue).

**Q4: How do you handle a slow consumer that's falling behind?**
> Monitor consumer lag (latest offset - committed offset). Solutions: (1) Scale consumers — add more instances to the consumer group (up to number of partitions!), (2) Increase batch size — process more messages per poll, (3) Async processing within consumer — process messages in parallel (careful with ordering!), (4) Back-pressure — if consumer can't keep up, pause consumption temporarily (Kafka pause/resume), (5) Increase retention — give consumer more time to catch up! If permanently slow: re-architecture needed (faster processing or fewer messages).

---

## 🎮 Mini Challenge

Design a Dead Letter Queue (DLQ) system:
- After 3 failed processing attempts → move to DLQ
- DLQ messages can be manually inspected and retried
- Track failure reason and attempt count
- Alert when DLQ depth exceeds threshold

*How do you implement retry with exponential backoff in a queue-based system?*

---

## 🔗 Related Topics
- [Message Queues](../BuildingBlocks/MessageQueues.md) — Fundamentals
- [Pub/Sub](../MessagingQ/PubSub.md) — Pub/Sub patterns
- [CDC](../MessagingQ/CDC.md) — Change Data Capture
- [Event-Driven Architecture](../Architectures/Event_Driven.md) — Async patterns

---

*"A distributed message queue is essentially a distributed, replicated, append-only log with consumer group coordination. Once you understand it's just a LOG — everything else (ordering, retention, replay, compaction) follows naturally." — Jay Kreps, creator of Kafka* 📨
