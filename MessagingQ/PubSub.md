# 📢 Pub/Sub (Publish-Subscribe): Decoupling at Scale

> *"YouTube uploads 500 hours of video every minute. When a video is uploaded, 15+ systems need to react: transcoding, thumbnail generation, content moderation, notification service, search indexing, recommendation engine, analytics, monetization... If the upload service called each one synchronously, a single slow system would block ALL uploads. Pub/Sub makes each system independent — they subscribe to 'VideoUploaded' events and react at their own pace."*

**⏱️ Estimated Time**: 30 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Message Queues](../BuildingBlocks/MessageQueues.md), [Event-Driven Architecture](../Architectures/Event_Driven.md)

---

## 📋 Table of Contents
1. [What is Pub/Sub?](#-what-is-pubsub)
2. [Pub/Sub vs Message Queues](#-pubsub-vs-message-queues)
3. [How Pub/Sub Works](#-how-pubsub-works)
4. [Topic Design Patterns](#-topic-design-patterns)
5. [Delivery Guarantees](#-delivery-guarantees)
6. [Real-World Systems](#-real-world-systems)
7. [Java/Spring Boot Examples](#-javaspring-boot-examples)
8. [Scaling Pub/Sub](#-scaling-pubsub)
9. [Mini Challenge](#-mini-challenge)
10. [Interview Q&A](#-interview-qa)

---

## 🤔 What is Pub/Sub?

```
╔══════════════════════════════════════════════════════════════════╗
║  PUB/SUB = A messaging pattern where senders (publishers) do   ║
║  NOT send messages directly to receivers (subscribers).        ║
║  Instead, messages are categorized into TOPICS.                ║
║  Subscribers express interest in topics. Everyone is decoupled.║
╚══════════════════════════════════════════════════════════════════╝
```

### 🎮 The Radio Station Analogy

```
  📻 RADIO STATION (Publisher):
    "Here's the news!" → broadcasts on 98.5 FM (topic)
    Doesn't know or care WHO is listening!
    
  🎧 LISTENERS (Subscribers):
    Tune into 98.5 FM (subscribe to topic)
    Can listen or not — station doesn't care!
    Multiple listeners hear the SAME broadcast!
    
  KEY PROPERTIES:
    • Station doesn't know its listeners (decoupled!)
    • Multiple listeners get same message (fan-out!)
    • New listener? Just tune in! (no station change needed!)
    • Listener on vacation? Messages are missed (or buffered)
```

---

## ⚖️ Pub/Sub vs Message Queues

```
MESSAGE QUEUE (Point-to-Point):
  ┌──────────┐         ┌───────┐         ┌──────────┐
  │ Producer │────────►│ Queue │────────►│ Consumer │ (ONE consumer gets msg!)
  └──────────┘         └───────┘         └──────────┘
  
  • ONE message → ONE consumer (competing consumers)
  • Message removed after consumption
  • Used for: Work distribution, task processing

PUB/SUB (Fan-out):
  ┌──────────┐         ┌───────┐         ┌──────────┐ Subscriber A
  │Publisher │────────►│ Topic │────────►│          │ (gets msg!)
  └──────────┘         └───────┘    ├───►│          │ Subscriber B
                                    │    │          │ (ALSO gets msg!)
                                    └───►│          │ Subscriber C
                                         └──────────┘ (ALSO gets msg!)
  
  • ONE message → ALL subscribers get a COPY
  • Message stays until TTL (all subscribers can read)
  • Used for: Event notification, broadcasting, decoupling

KAFKA COMBINES BOTH:
  • Consumer Group = Queue behavior (one consumer per group gets msg)
  • Multiple Consumer Groups = Pub/Sub (each group gets a copy!)
```

---

## ⚙️ How Pub/Sub Works

```
STEP 1: Define Topics (categories of messages)
  Topics: "order-events", "user-events", "payment-events"

STEP 2: Publishers send messages to topics (not to subscribers!)
  OrderService.publish("order-events", {orderId: 123, status: "created"})

STEP 3: Subscribers register interest in topics
  NotificationService.subscribe("order-events")
  AnalyticsService.subscribe("order-events")
  InventoryService.subscribe("order-events")

STEP 4: Message broker delivers messages to ALL subscribers

  ┌─────────────────────────────────────────────────────────────┐
  │                      MESSAGE BROKER                          │
  │                                                             │
  │   Topic: "order-events"                                     │
  │   ┌─────────────────────────────────────────────┐          │
  │   │  msg1  │  msg2  │  msg3  │  msg4  │  msg5  │          │
  │   └──────┬─────────────────────────────┬────────┘          │
  │          │                             │                    │
  │          ▼                             ▼                    │
  │   Subscriber A (offset: 3)    Subscriber B (offset: 5)     │
  │   [behind by 2 messages]      [caught up!]                  │
  └─────────────────────────────────────────────────────────────┘
  
  Each subscriber tracks its OWN position (offset/cursor).
  Subscribers are INDEPENDENT — slow subscriber doesn't block others!
```

---

## 🏗️ Topic Design Patterns

```
1. EVENT-CARRIED STATE TRANSFER:
   Message contains all data needed (no callback required!)
   
   Topic: "order-created"
   Message: {
     orderId: 123,
     userId: 456,
     items: [{...}],     ← Full data included!
     totalAmount: 99.99,
     shippingAddress: {...}
   }
   
   Subscriber has everything needed to act. No API call back to publisher!

2. EVENT NOTIFICATION (thin events):
   Message just notifies "something happened" — subscriber fetches details.
   
   Topic: "order-created"
   Message: {orderId: 123, timestamp: "..."}
   
   Subscriber: "Order 123 created? Let me fetch details from Order API..."
   
   Pro: Small messages, publisher doesn't leak internal state
   Con: Subscriber must call back (coupling!)

3. TOPIC HIERARCHY:
   orders.created
   orders.updated
   orders.cancelled
   orders.shipped
   
   Subscribe to "orders.*" = get ALL order events!
   Subscribe to "orders.cancelled" = only cancellations!
```

---

## 🎯 Delivery Guarantees

```
┌───────────────────┬──────────────────────────────────────────────┐
│  Guarantee        │  What It Means                               │
├───────────────────┼──────────────────────────────────────────────┤
│  At-most-once     │  Message may be lost, never duplicated       │
│                   │  (fire and forget)                           │
├───────────────────┼──────────────────────────────────────────────┤
│  At-least-once    │  Message delivered 1+ times (may duplicate!) │
│                   │  Most common. Requires idempotent consumers! │
├───────────────────┼──────────────────────────────────────────────┤
│  Exactly-once     │  Message delivered exactly once              │
│                   │  Very hard! (Kafka achieves with transactions)│
└───────────────────┴──────────────────────────────────────────────┘

WHY AT-LEAST-ONCE IS MOST PRACTICAL:
  
  Publisher → Broker: "Here's message X"
  Broker → Publisher: "ACK, stored!"
  Broker → Subscriber: "Here's message X"
  Subscriber: processes message
  Subscriber → Broker: "ACK, done!"
  
  What if subscriber crashes AFTER processing but BEFORE ACK?
  Broker: "No ACK received... re-deliver!"
  Subscriber: processes message AGAIN (duplicate!)
  
  Solution: Make subscriber IDEMPOTENT
  (See: [Idempotency](../APIs/Idempotency.md))
```

---

## 🏢 Real-World Systems

```
┌───────────────┬──────────────────────────────────────────────────┐
│  System       │  Pub/Sub Usage                                   │
├───────────────┼──────────────────────────────────────────────────┤
│  Apache Kafka │  Distributed log with consumer groups            │
│               │  Millions of messages/sec, persistent storage    │
├───────────────┼──────────────────────────────────────────────────┤
│  Google Pub/Sub│  Fully managed, global, exactly-once delivery  │
│               │  Used by Google internally for YouTube, Gmail    │
├───────────────┼──────────────────────────────────────────────────┤
│  AWS SNS      │  Simple notification service, push-based         │
│               │  Fans out to SQS, Lambda, HTTP, email, SMS       │
├───────────────┼──────────────────────────────────────────────────┤
│  Redis Pub/Sub│  In-memory, fire-and-forget (no persistence!)    │
│               │  Good for real-time, bad for reliability         │
├───────────────┼──────────────────────────────────────────────────┤
│  RabbitMQ     │  Exchange + binding routing, flexible patterns   │
│               │  Fanout exchange = pure pub/sub                  │
└───────────────┴──────────────────────────────────────────────────┘
```

---

## 💻 Java/Spring Boot Examples

### Kafka Publisher

```java
@Service
public class OrderEventPublisher {
    
    @Autowired private KafkaTemplate<String, OrderEvent> kafkaTemplate;
    
    public void publishOrderCreated(Order order) {
        OrderCreatedEvent event = OrderCreatedEvent.builder()
            .orderId(order.getId())
            .userId(order.getUserId())
            .items(order.getItems())
            .totalAmount(order.getTotal())
            .timestamp(Instant.now())
            .build();
        
        // Publish to "order-events" topic, keyed by orderId (for ordering)
        kafkaTemplate.send("order-events", order.getId(), event)
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    log.error("Failed to publish order event: {}", ex.getMessage());
                    // Dead letter queue or retry
                } else {
                    log.info("Published OrderCreated to partition {}, offset {}",
                        result.getRecordMetadata().partition(),
                        result.getRecordMetadata().offset());
                }
            });
    }
}
```

### Multiple Subscribers (Different Consumer Groups)

```java
// Subscriber 1: Notification Service
@Service
public class NotificationSubscriber {
    
    @KafkaListener(topics = "order-events", groupId = "notification-service")
    public void handleOrderEvent(OrderEvent event) {
        if (event instanceof OrderCreatedEvent created) {
            emailService.sendOrderConfirmation(created.getUserId(), created.getOrderId());
            pushNotificationService.notify(created.getUserId(), "Order placed!");
        }
    }
}

// Subscriber 2: Analytics Service (DIFFERENT consumer group = gets same messages!)
@Service
public class AnalyticsSubscriber {
    
    @KafkaListener(topics = "order-events", groupId = "analytics-service")
    public void handleOrderEvent(OrderEvent event) {
        if (event instanceof OrderCreatedEvent created) {
            metricsService.incrementOrderCount();
            metricsService.addRevenue(created.getTotalAmount());
        }
    }
}

// Subscriber 3: Inventory Service
@Service
public class InventorySubscriber {
    
    @KafkaListener(topics = "order-events", groupId = "inventory-service")
    public void handleOrderEvent(OrderEvent event) {
        if (event instanceof OrderCreatedEvent created) {
            for (OrderItem item : created.getItems()) {
                inventoryService.decrementStock(item.getProductId(), item.getQuantity());
            }
        }
    }
}
```

### Spring Cloud Stream (Abstraction over Kafka/RabbitMQ)

```java
@Configuration
public class PubSubConfig {
    
    @Bean
    public Function<OrderEvent, Void> processOrder() {
        return event -> {
            log.info("Processing order event: {}", event.getOrderId());
            // Process...
            return null;
        };
    }
    
    @Bean
    public Consumer<OrderEvent> auditLog() {
        return event -> {
            auditRepository.save(new AuditEntry(event));
        };
    }
}
```

---

## 📈 Scaling Pub/Sub

```
SCALING PUBLISHERS:
  Easy! Multiple publishers write to same topic independently.
  No coordination needed between publishers.

SCALING SUBSCRIBERS:
  KAFKA: Partition-based parallelism
    Topic "orders" with 12 partitions:
    Consumer Group "notification-service" with 4 instances:
      Instance 1: partitions [0, 1, 2]
      Instance 2: partitions [3, 4, 5]
      Instance 3: partitions [6, 7, 8]
      Instance 4: partitions [9, 10, 11]
    
    Max parallelism = number of partitions!
    Adding Instance 5 with only 12 partitions → one sits idle!

ORDERING GUARANTEES:
  Within a partition: STRICT ordering guaranteed ✅
  Across partitions: NO ordering guarantee ❌
  
  Key insight: Same key → same partition → ordered!
  Different keys → different partitions → no order between them
```

---

## 🎮 Mini Challenge

### 🧩 Design: Event-Driven E-commerce

When a customer places an order, these things must happen:
1. Charge payment
2. Reserve inventory
3. Send confirmation email
4. Update analytics dashboard
5. Notify warehouse for shipping
6. Update recommendation engine

Design the Pub/Sub topic structure and subscriber grouping.

<details>
<summary>🔑 Answer</summary>

**Topics:**
- `order.created` — published when order is placed
- `payment.completed` / `payment.failed` — published after payment attempt
- `inventory.reserved` / `inventory.insufficient` — published after inventory check
- `order.confirmed` — published when payment + inventory both succeed

**Flow:**
1. OrderService publishes `order.created`
2. PaymentService subscribes to `order.created` → processes → publishes `payment.completed`
3. InventoryService subscribes to `order.created` → reserves → publishes `inventory.reserved`
4. OrderService subscribes to BOTH payment + inventory results → if both succeed → publishes `order.confirmed`
5. NotificationService subscribes to `order.confirmed` → sends email
6. AnalyticsService subscribes to `order.confirmed` → updates metrics
7. WarehouseService subscribes to `order.confirmed` → creates shipping order
8. RecommendationService subscribes to `order.confirmed` → updates model

**Key design decisions:**
- Payment and Inventory are CRITICAL (must succeed) → use Saga pattern for compensation
- Email, Analytics, Warehouse are NON-CRITICAL → fire-and-forget subscribers
- Each service = separate consumer group (all get the message independently)
</details>

---

## ❓ Interview Q&A

**Q1: What is Pub/Sub and how does it differ from a message queue?**
> Pub/Sub: one message is delivered to ALL subscribers (fan-out). Each subscriber gets a copy. Used for event broadcasting. Message Queue: one message goes to ONE consumer (competing consumers). Used for work distribution. Kafka supports both: multiple consumer groups = pub/sub (each group gets all messages), consumers within a group = queue (messages distributed among members).

**Q2: How does Kafka implement Pub/Sub?**
> Kafka organizes messages into topics split into partitions. Each partition is an ordered, immutable log. Consumer groups subscribe to topics. Each consumer in a group reads from assigned partitions. Different consumer groups independently read ALL messages (pub/sub). Within a group, messages are distributed (queue semantics). This enables both patterns simultaneously.

**Q3: How do you ensure ordering in Pub/Sub systems?**
> In Kafka: ordering is guaranteed within a partition, not across partitions. To ensure ordering for related events: use the same message key (e.g., order_id). Same key → same partition → strict ordering. Trade-off: more unique keys = better parallelism but weaker ordering; fewer keys = stricter ordering but less parallelism.

**Q4: What happens when a subscriber is slower than the publisher?**
> In persistent pub/sub (Kafka): messages accumulate in the topic. Subscriber's offset falls behind. This is called "consumer lag." The topic acts as a buffer. Subscriber eventually catches up. If lag grows too large: (1) scale subscriber (more instances), (2) increase partitions, (3) or accept eventual processing. In non-persistent pub/sub (Redis Pub/Sub): messages are LOST if subscriber can't keep up!

**Q5: How do you handle poison messages (messages that always fail processing)?**
> Dead Letter Queue (DLQ) pattern: after N failed attempts, move the message to a special "dead letter" topic. Benefits: (1) unblocks the subscriber, (2) preserves the message for investigation, (3) can be reprocessed after fixing the bug. Implementation: configure retry count, exponential backoff, and DLQ topic per consumer group.

---

## 🔗 Related Topics
- [Message Queues](../BuildingBlocks/MessageQueues.md) — Point-to-point messaging
- [Event-Driven Architecture](../Architectures/Event_Driven.md) — Architectural pattern using pub/sub
- [Idempotency](../APIs/Idempotency.md) — Making subscribers safe for redelivery
- [CQRS](../Architectures/CQRS.md) — Often uses pub/sub for event propagation

---

*"Pub/Sub is the nervous system of microservices. Without it, you have a distributed monolith that talks too much." — Event-Driven Architecture Wisdom* 📢
