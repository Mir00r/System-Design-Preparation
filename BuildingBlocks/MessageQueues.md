# 📬 Message Queues: Asynchronous Communication Between Services

> *"If microservices talk to each other synchronously, you've built a distributed monolith. One slow service brings down everything. Message queues break this coupling — the sender fires and forgets, the receiver processes at its own pace."*

**⏱️ Estimated Time**: 35 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Microservices Basics](../Microservices/DistributedSystem.md)

---

## 📋 Table of Contents
1. [The Problem](#-the-problem)
2. [How Message Queues Work](#-how-message-queues-work)
3. [Message Queue vs Event Stream](#-message-queue-vs-event-stream)
4. [Delivery Guarantees](#-delivery-guarantees)
5. [Popular Message Queues Compared](#-popular-message-queues-compared)
6. [When to Use What](#-when-to-use-what)
7. [Spring Boot Integration](#-spring-boot-integration)
8. [Common Pitfalls](#-common-pitfalls)
9. [Mini Challenge](#-mini-challenge)
10. [Interview Q&A](#-interview-qa)

---

## 🤔 The Problem

```
Synchronous architecture (direct HTTP calls):

  [Order Service] ──HTTP──▶ [Payment Service] ──HTTP──▶ [Notification Service]
                                                              ──HTTP──▶ [Inventory Service]

  Problems:
    1. Temporal coupling: all services must be UP simultaneously
    2. Cascading failure: payment-service slow → order-service times out → user sees error
    3. No retry logic: if notification-service is down, notification is LOST
    4. Scaling mismatch: order-service handles 1000 TPS but email-service handles only 50 TPS
    5. Latency: total = sum of all service latencies (200ms + 500ms + 100ms = 800ms)


Asynchronous architecture (message queue):

  [Order Service] ──publish──▶ [Message Queue] ──consume──▶ [Payment Service]
                                      │        ──consume──▶ [Notification Service]
                                      └──────────consume──▶ [Inventory Service]

  Benefits:
    1. Decoupled: services don't need to know about each other
    2. Resilient: queue buffers messages if consumer is down
    3. Load leveling: consumers process at their own speed
    4. Retry: failed messages are retried automatically
    5. Latency: order-service returns in 50ms (just writes to queue)
```

---

## 🏗️ How Message Queues Work

```
┌─────────────────────────────────────────────────────────────────┐
│                    MESSAGE QUEUE COMPONENTS                     │
│                                                                 │
│  Producer          Queue/Topic              Consumer            │
│  (sender)         (buffer)                 (receiver)           │
│                                                                 │
│  [Order Svc] ─▶ ┌─────────────────┐ ─▶ [Payment Svc]          │
│                  │ msg1 │ msg2 │...│                            │
│                  └─────────────────┘                            │
│                                                                 │
│  Key concepts:                                                  │
│    Producer: creates and sends messages                         │
│    Consumer: receives and processes messages                    │
│    Queue: FIFO buffer that stores messages until consumed       │
│    Topic: pub/sub — one message delivered to ALL subscribers    │
│    Broker: the server managing queues/topics (Kafka, RabbitMQ)  │
│    Dead Letter Queue (DLQ): where failed messages go            │
└─────────────────────────────────────────────────────────────────┘
```

### Two Messaging Patterns

```
PATTERN 1: POINT-TO-POINT (Queue)
  One message → one consumer
  Use: task distribution (send email, process payment)

  Producer → [Queue: ████████] → Consumer A gets msg1
                                → Consumer B gets msg2
                                → Consumer A gets msg3
  (work is distributed across consumers — each message processed once)

PATTERN 2: PUBLISH/SUBSCRIBE (Topic)
  One message → all subscribers
  Use: event notification (order.created → payment, inventory, analytics ALL get it)

  Producer → [Topic: order.created]
              ├──▶ Payment Service (subscriber 1) gets the message
              ├──▶ Inventory Service (subscriber 2) gets the message
              └──▶ Analytics Service (subscriber 3) gets the message
  (every subscriber receives every message)
```

---

## ⚖️ Message Queue vs Event Stream

| Feature | Message Queue (RabbitMQ, SQS) | Event Stream (Kafka) |
|---|---|---|
| **Message lifecycle** | Deleted after consumption | Retained for configurable period (days/forever) |
| **Replay** | ❌ Cannot re-read consumed messages | ✅ Can replay from any offset |
| **Ordering** | Per-queue FIFO (mostly) | Per-partition ordering guarantee |
| **Consumer model** | Push (broker pushes to consumer) | Pull (consumer pulls at own pace) |
| **Throughput** | ~10K-50K msg/sec | 1M+ msg/sec per cluster |
| **Use case** | Task queues, work distribution | Event sourcing, streaming, log aggregation |
| **Complexity** | 🟢 Simpler to operate | 🔴 Complex (ZooKeeper/KRaft, partitions, offsets) |

---

## 📦 Delivery Guarantees

```
AT-MOST-ONCE (fire and forget):
  Producer sends → broker ACKs immediately → if consumer crashes, message lost
  Use: metrics, non-critical analytics
  Pro: fastest, simplest
  Con: messages can be lost

AT-LEAST-ONCE (guaranteed delivery, possible duplicates):
  Producer sends → broker ACKs after persisting → consumer processes → ACKs back
  If consumer crashes before ACK → broker redelivers → DUPLICATE processing possible
  Use: most business events (combine with idempotent consumers)
  Pro: no message loss
  Con: consumers must handle duplicates (idempotency key)

EXACTLY-ONCE (holy grail — complex):
  Kafka transactional produce + consume with exactly-once semantics (EOS)
  Or: at-least-once delivery + idempotent consumer (simulated exactly-once)
  Use: financial transactions, payment processing
  Pro: no loss, no duplicates
  Con: highest latency, most complex
```

---

## 📊 Popular Message Queues Compared

| Feature | **Kafka** | **RabbitMQ** | **AWS SQS** | **Redis Streams** |
|---|---|---|---|---|
| **Model** | Event stream (log) | Message queue (AMQP) | Managed queue | Stream (append-only) |
| **Throughput** | 1M+ msg/sec | 10-50K msg/sec | Unlimited (managed) | 100K+ msg/sec |
| **Ordering** | Per-partition | Per-queue | Best-effort (FIFO optional) | Per-stream |
| **Retention** | Days/weeks/forever | Until consumed | 14 days max | Configurable |
| **Replay** | ✅ Yes | ❌ No | ❌ No | ✅ Yes |
| **Delivery** | At-least-once (exactly-once possible) | At-least-once | At-least-once | At-least-once |
| **Operational cost** | 🔴 High (cluster mgmt) | 🟡 Medium | 🟢 Zero (managed) | 🟢 Low (if already using Redis) |
| **Best for** | Event sourcing, streaming, high throughput | Task queues, routing, low latency | Simple queues, serverless | Lightweight streaming needs |

---

## 🎯 When to Use What

```
Choose KAFKA when:
  ✅ High throughput (> 100K msg/sec)
  ✅ Event sourcing / event-driven architecture
  ✅ Need to replay events (rebuild state)
  ✅ Multiple consumers need the same events (pub/sub at scale)
  ✅ Log aggregation / streaming analytics

Choose RABBITMQ when:
  ✅ Complex routing (topic exchange, headers exchange, routing keys)
  ✅ Task queues (distribute work across workers)
  ✅ Low latency messaging (< 1ms broker latency)
  ✅ Priority queues (process VIP orders first)
  ✅ Request-reply pattern (RPC over queues)

Choose AWS SQS when:
  ✅ Zero operational overhead (fully managed)
  ✅ Simple queue semantics (no complex routing)
  ✅ Lambda integration (serverless event processing)
  ✅ Don't want to manage broker infrastructure

Choose REDIS STREAMS when:
  ✅ Already using Redis (no new infrastructure)
  ✅ Lightweight streaming needs
  ✅ Sub-millisecond latency required
  ✅ Consumer groups with simple semantics
```

---

## ⚙️ Spring Boot Integration

### RabbitMQ (Task Queue Pattern)

```java
// Producer: send order event to queue
@Service
public class OrderEventPublisher {
    private final RabbitTemplate rabbitTemplate;

    public void publishOrderCreated(Order order) {
        rabbitTemplate.convertAndSend(
            "order-exchange",           // exchange name
            "order.created",            // routing key
            new OrderCreatedEvent(order.getId(), order.getUserId(), order.getAmount())
        );
    }
}

// Consumer: process order events
@Component
public class PaymentConsumer {

    @RabbitListener(queues = "payment-queue")
    public void processPayment(OrderCreatedEvent event) {
        paymentService.charge(event.userId(), event.amount());
        // If this throws an exception → message is requeued (retry)
        // After 3 retries → sent to Dead Letter Queue
    }
}
```

### Kafka (Event Streaming Pattern)

```java
// Producer: publish domain event
@Service
public class OrderEventPublisher {
    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;

    public void publish(OrderEvent event) {
        kafkaTemplate.send("order-events", event.orderId(), event)
            .whenComplete((result, ex) -> {
                if (ex != null) log.error("Failed to publish: {}", event.orderId(), ex);
            });
    }
}

// Consumer: multiple services subscribe to the same topic
@Component
public class InventoryConsumer {

    @KafkaListener(topics = "order-events", groupId = "inventory-service")
    public void updateInventory(OrderEvent event) {
        if (event.type() == EventType.ORDER_CREATED) {
            inventoryService.reserve(event.items());
        }
    }
}
```

---

## ⚠️ Common Pitfalls

1. **Not handling duplicate messages** — At-least-once delivery means consumers WILL receive duplicates (network retry, consumer crash before ACK). Every consumer must be idempotent: use an `idempotency_key` or `message_id` to detect and skip duplicates.

2. **Unbounded queue growth** — If consumers are slower than producers, the queue grows indefinitely until the broker runs out of disk. Set TTL on messages, configure max queue length, and alert when queue depth exceeds thresholds.

3. **Message ordering assumptions** — Most queues guarantee ordering only within a single partition/queue. If you need ordering for a specific entity (all messages for order-123 in sequence), route by entity ID to the same partition.

4. **No Dead Letter Queue** — Messages that fail processing repeatedly (poison messages) block the queue forever. Always configure a DLQ: after N retries, move the message to a DLQ for manual investigation.

5. **Synchronous patterns over queues** — Using request-reply pattern over a queue (send message, wait for response) negates the benefits of async messaging. If you need a synchronous response, use HTTP. Queues are for fire-and-forget or eventual processing.

---

## 🧩 Mini Challenge

**You're designing a food delivery app.** When a user places an order, these things must happen:
1. Payment must be charged (critical — must succeed)
2. Restaurant must be notified (critical — must succeed)
3. Analytics event must be logged (non-critical)
4. Promotional email must be sent (non-critical)

**Design the messaging architecture. Which operations should be synchronous and which asynchronous?**

<details>
<summary>💡 Click to reveal answer</summary>

```
SYNCHRONOUS (must succeed before returning to user):
  1. Payment charging — user needs immediate feedback (success/failure)
     [Order Service] ──HTTP──▶ [Payment Service]
     If payment fails → return error to user, don't proceed

ASYNCHRONOUS (via message queue, after order is confirmed):
  2. Restaurant notification ← critical, at-least-once delivery
     Publish to "order.confirmed" topic → restaurant-service consumes
     Retry policy: 5 retries with exponential backoff
     DLQ: if all retries fail → alert operations team for manual intervention

  3. Analytics event ← non-critical, at-most-once is acceptable
     Publish to "order.analytics" topic → analytics-service consumes
     No DLQ needed; if lost, analytics is slightly inaccurate (acceptable)

  4. Promotional email ← non-critical, eventual delivery OK
     Publish to "notifications.email" queue → email-service consumes
     Low priority queue; processed when email-service has capacity
     DLQ: failed emails are retried later but never block other processing

Architecture:
  [User] → [Order Service] ──sync──▶ [Payment Service]
                │
                │ (after payment succeeds)
                ▼
  [Kafka Topic: order.confirmed]
    ├──▶ Restaurant Service (consumer group: restaurant-svc)
    ├──▶ Analytics Service (consumer group: analytics-svc)
    └──▶ Notification Service (consumer group: notification-svc)
```

**Key insight**: Only the payment is synchronous because the user needs immediate feedback. Everything else happens asynchronously after the order is confirmed — the user doesn't wait for the restaurant notification or email to be sent.

</details>

---

## 📝 Interview Q&A

**Q: What's the difference between a message queue and an event stream?**
> A: A message queue (RabbitMQ, SQS) removes messages after consumption — once processed, the message is gone. It's designed for task distribution where each message is processed by exactly one consumer. An event stream (Kafka) retains messages for a configurable period regardless of consumption — consumers track their own position (offset). Multiple consumer groups can independently read the same stream, and consumers can replay from any point in history. Use queues for work distribution, streams for event sourcing and analytics.

**Q: How do you ensure exactly-once processing with at-least-once delivery?**
> A: Implement idempotent consumers. Each message carries a unique `idempotency_key`. Before processing, the consumer checks if this key has already been processed (lookup in a database or Redis). If yes, skip it. If no, process it and record the key atomically with the side effect (within the same database transaction). This turns at-least-once delivery into effectively exactly-once processing at the application level — the most practical approach in distributed systems.

**Q: When would you choose RabbitMQ over Kafka?**
> A: Choose RabbitMQ when: (1) you need complex message routing — RabbitMQ's exchange types (topic, headers, fanout) route messages based on patterns, headers, or broadcast; (2) you need low-latency messaging (< 1ms broker latency vs Kafka's typical 5-10ms); (3) you need priority queues (process urgent messages first); (4) your throughput is < 50K msg/sec; (5) you want simpler operations (single node RabbitMQ is much simpler than a Kafka cluster with ZooKeeper/KRaft).

---

## 🔗 What to Read Next

1. **[MessagingQ/Kafka.md](../MessagingQ/Kafka.md)** — Deep dive into Kafka partitions, consumer groups, and exactly-once semantics
2. **[BuildingBlocks/CircuitBreaker.md](./CircuitBreaker.md)** — When downstream services are slow, combine with async messaging for resilience
3. **[Architectures/Event_Driven.md](../Architectures/Event_Driven.md)** — Event-driven architecture patterns built on message queues

---

*[← Search Index](./SearchIndex.md) | [Back to BuildingBlocks](../INDEX.md)*
