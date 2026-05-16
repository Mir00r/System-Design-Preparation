# 🐰 RabbitMQ: The Enterprise Message Broker for Complex Routing

> *"When your microservices need smart routing — send purchase orders to billing AND shipping, retry failed payments 3 times with exponential backoff, and prioritize VIP customer requests — RabbitMQ's exchange model gives you that control out of the box."*

**⏱️ Estimated Time**: 35 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Message Queues](../BuildingBlocks/MessageQueues.md), [Kafka](./Kafka.md)

---

## 📋 Table of Contents
1. [What is RabbitMQ](#-what-is-rabbitmq)
2. [AMQP Model](#-amqp-model)
3. [Exchange Types](#-exchange-types)
4. [Reliability Guarantees](#-reliability-guarantees)
5. [Clustering & High Availability](#-clustering--high-availability)
6. [Spring Boot Integration](#-spring-boot-integration)
7. [Advanced Patterns](#-advanced-patterns)
8. [When to Use](#-when-to-use--not-use)
9. [Common Pitfalls](#-common-pitfalls)
10. [Mini Challenge](#-mini-challenge)
11. [Interview Q&A](#-interview-qa)

---

## 🤔 What is RabbitMQ

```
┌─────────────────────────────────────────────────────────┐
│                  RABBITMQ AT A GLANCE                    │
│                                                         │
│  Type:        Message broker (AMQP 0-9-1)              │
│  Model:       Smart broker, dumb consumers              │
│  Routing:     Exchanges + binding keys (flexible)       │
│  Delivery:    Push-based (broker pushes to consumers)   │
│  Ordering:    Per-queue FIFO guaranteed                 │
│  Protocol:    AMQP, MQTT, STOMP, HTTP                   │
│  Language:    Erlang (built for high concurrency)        │
│  Throughput:  20K-50K messages/sec per node              │
│  Latency:     ~1ms message delivery                     │
└─────────────────────────────────────────────────────────┘
```

---

## 🏗️ AMQP Model

```
MESSAGE FLOW:

  [Producer] → [Exchange] → [Binding] → [Queue] → [Consumer]
                   │                       │
                   │ routing rules          │ competing consumers
                   │                       │ (work distribution)

COMPONENTS:
  Producer:   publishes messages to an Exchange (NOT directly to queue)
  Exchange:   receives messages, routes to queues based on rules
  Binding:    rule connecting Exchange → Queue (with routing key pattern)
  Queue:      buffer that stores messages until consumed
  Consumer:   subscribes to queue, processes messages

  Key insight: Producers don't know about queues!
  They publish to an exchange with a routing key.
  The exchange decides which queues receive the message.

EXAMPLE:
  Producer publishes: exchange="orders", routing_key="order.created"
  
  Bindings:
    exchange="orders" → queue="billing" (key="order.created")
    exchange="orders" → queue="shipping" (key="order.created")  
    exchange="orders" → queue="analytics" (key="order.*")
  
  Result: message goes to billing, shipping, AND analytics queues
```

---

## 📬 Exchange Types

```
1. DIRECT EXCHANGE (routing key exact match):
   
   Producer → Exchange → routing_key="payment.success"
                │
                ├── binding key="payment.success" → [billing queue] ✅
                ├── binding key="payment.failed"  → [retry queue]   ❌ (no match)
                └── binding key="payment.success" → [notification]  ✅

2. FANOUT EXCHANGE (broadcast to ALL bound queues):
   
   Producer → Exchange → (routing key ignored)
                │
                ├── → [queue 1] ✅
                ├── → [queue 2] ✅
                └── → [queue 3] ✅
   
   Use case: broadcast events (user signup → email, analytics, recommendations)

3. TOPIC EXCHANGE (pattern matching with wildcards):
   
   Routing key: "order.us.electronics"
   
   Bindings:
     "order.us.*"         → [us-fulfillment]    ✅ (* = one word)
     "order.*.electronics"→ [electronics-team]   ✅
     "order.#"            → [audit-log]          ✅ (# = zero or more words)
     "payment.#"          → [payment-service]    ❌ (doesn't match "order.*")

4. HEADERS EXCHANGE (match on message headers, not routing key):
   
   Message headers: { "format": "pdf", "type": "report" }
   Binding args: { "format": "pdf", "x-match": "any" }
   → Match! Route to queue.
   
   Use case: routing based on metadata, not content path
```

---

## ✅ Reliability Guarantees

```
MESSAGE DURABILITY (survive broker restart):
  1. Durable exchange:  AMQP.exchangeDeclare(durable=true)
  2. Durable queue:     AMQP.queueDeclare(durable=true)
  3. Persistent message: properties.deliveryMode(2)
  
  All THREE needed for messages to survive restart!

PUBLISHER CONFIRMS (producer knows message is safe):
  channel.confirmSelect();
  channel.basicPublish(exchange, routingKey, props, body);
  channel.waitForConfirms(); // blocks until broker confirms receipt
  
  Without confirms: fire-and-forget (message might be lost)

CONSUMER ACKNOWLEDGMENTS:
  autoAck=false: consumer must explicitly ACK after processing
  
  channel.basicConsume(queue, false, (tag, delivery) -> {
      try {
          processMessage(delivery.getBody());
          channel.basicAck(tag, false);        // success → remove from queue
      } catch (Exception e) {
          channel.basicNack(tag, false, true);  // failure → requeue
      }
  });
  
  If consumer crashes without ACK → message redelivered to another consumer

DELIVERY GUARANTEES:
  At-most-once:  autoAck=true, no publisher confirms (fast, lossy)
  At-least-once: manual ACK + publisher confirms (safe, may duplicate)
  Exactly-once:  not natively supported (use idempotent consumers)
```

---

## 🏢 Clustering & High Availability

```
CLUSTERING (multiple nodes act as one broker):
  ┌─────────┐   ┌─────────┐   ┌─────────┐
  │  Node 1 │───│  Node 2 │───│  Node 3 │
  │ (queues │   │ (queues │   │ (queues │
  │  A, B)  │   │  C, D)  │   │  E, F)  │
  └─────────┘   └─────────┘   └─────────┘
  
  Metadata (exchanges, bindings, users) replicated to ALL nodes.
  Queue DATA lives on ONE node by default (not replicated).

QUORUM QUEUES (replicated queues for HA — recommended since 3.8):
  Based on Raft consensus protocol
  Queue replicated across N nodes (configurable)
  Tolerates (N-1)/2 node failures
  
  x-queue-type: quorum
  
  ┌─────────────────────────────────────────┐
  │ Queue "orders" (quorum, RF=3)           │
  │                                         │
  │  Node 1: Leader (accepts writes)        │
  │  Node 2: Follower (replica)             │
  │  Node 3: Follower (replica)             │
  │                                         │
  │  If Node 1 fails → Node 2 elected leader│
  │  Message confirmed ONLY after majority   │
  │  acknowledges (Raft commit)             │
  └─────────────────────────────────────────┘

  vs CLASSIC MIRRORED QUEUES (deprecated):
    Synchronous replication to all mirrors (slow)
    Split-brain issues during network partitions
    → Use quorum queues instead!
```

---

## 💻 Spring Boot Integration

```java
// application.yml
spring:
  rabbitmq:
    host: rabbitmq-cluster
    port: 5672
    username: ${RABBIT_USER}
    password: ${RABBIT_PASS}
    listener:
      simple:
        acknowledge-mode: manual
        prefetch: 10
        retry:
          enabled: true
          max-attempts: 3
          initial-interval: 1000
          multiplier: 2.0

// Configuration
@Configuration
public class RabbitConfig {
    
    @Bean
    public TopicExchange orderExchange() {
        return new TopicExchange("order.exchange", true, false);
    }
    
    @Bean
    public Queue billingQueue() {
        return QueueBuilder.durable("billing.queue")
            .withArgument("x-queue-type", "quorum")  // quorum for HA
            .withArgument("x-dead-letter-exchange", "dlx.exchange")
            .build();
    }
    
    @Bean
    public Binding billingBinding() {
        return BindingBuilder.bind(billingQueue())
            .to(orderExchange())
            .with("order.created.#");  // topic pattern
    }
}

// Producer
@Service
public class OrderEventPublisher {
    private final RabbitTemplate rabbitTemplate;
    
    public void publishOrderCreated(Order order) {
        rabbitTemplate.convertAndSend(
            "order.exchange",              // exchange
            "order.created." + order.getRegion(),  // routing key
            order,                          // message body (auto-serialized)
            message -> {
                message.getMessageProperties().setDeliveryMode(MessageDeliveryMode.PERSISTENT);
                message.getMessageProperties().setMessageId(order.getId().toString());
                return message;
            }
        );
    }
}

// Consumer with manual acknowledgment
@Component
public class BillingConsumer {
    
    @RabbitListener(queues = "billing.queue")
    public void handleOrderCreated(Order order, Channel channel, 
                                    @Header(AmqpHeaders.DELIVERY_TAG) long tag) {
        try {
            billingService.createInvoice(order);
            channel.basicAck(tag, false);  // success
        } catch (TransientException e) {
            channel.basicNack(tag, false, true);  // requeue for retry
        } catch (PermanentException e) {
            channel.basicNack(tag, false, false);  // send to dead letter queue
        }
    }
}
```

---

## 🔧 Advanced Patterns

```
DEAD LETTER QUEUE (DLQ) — handle poison messages:
  Queue → message fails 3 times → Dead Letter Exchange → DLQ
  
  Operations team monitors DLQ, investigates/replays failed messages

DELAYED MESSAGE (retry with backoff):
  Plugin: rabbitmq_delayed_message_exchange
  OR: TTL + DLX trick:
    retry.1s queue (TTL=1s) → DLX → retry.5s queue (TTL=5s) → DLX → main queue

PRIORITY QUEUES:
  x-max-priority: 10
  VIP orders get priority=9, normal orders get priority=1
  Higher priority consumed first

REQUEST-REPLY (RPC over RabbitMQ):
  Client → request queue → Server processes → reply queue → Client
  Use correlation_id to match responses to requests
```

---

## ⚖️ When to Use / Not Use

| Use RabbitMQ When | Avoid RabbitMQ When |
|---|---|
| Complex routing logic (topic, headers) | Simple pub/sub at massive scale (use Kafka) |
| Request-reply / RPC patterns | Need message replay / reprocessing (Kafka) |
| Priority queues needed | >100K msg/sec sustained (Kafka scales better) |
| Varied messaging patterns in one system | Event sourcing (need immutable log) |
| Team knows AMQP / needs protocol diversity | Long-term message retention (Kafka) |
| Low-latency message delivery (~1ms) | Need exactly-once semantics built-in |

---

## ⚠️ Common Pitfalls

1. **Not using publisher confirms** — Without confirms, messages can be lost between producer and broker (network failure, broker restart). Always enable publisher confirms for important messages.

2. **Unbounded queue growth** — If consumers are slower than producers, queues grow until disk is full and broker crashes. Set `x-max-length` or `x-overflow: reject-publish` to bound queues. Monitor queue depth and alert.

3. **Using classic mirrored queues** — Deprecated in favor of quorum queues. Classic mirrored queues have split-brain issues during network partitions and synchronous replication that degrades performance. Migrate to quorum queues.

4. **Single consumer for ordering** — If you need strict ordering, use a single consumer per queue. Multiple consumers process in parallel but ordering is lost. If you need parallelism + ordering, use consistent hashing exchange to route related messages to the same queue.

---

## 🧩 Mini Challenge

**Design a notification system using RabbitMQ: when a user gets a new follower, send email, push notification, and in-app notification. Email has retry with backoff, push is fire-and-forget, in-app must be acknowledged.**

<details>
<summary>💡 Click to reveal answer</summary>

```
Architecture:
  Producer: "FollowerService" publishes to fanout exchange

  [FollowerService]
       │ publish: { user_id, follower_id, follower_name }
       ▼
  [fanout exchange: "notification.new_follower"]
       │
       ├──→ [queue: "email.notifications"] (durable, quorum, DLQ)
       │      └── Consumer: EmailService
       │          - Manual ACK
       │          - Retry 3x with exponential backoff (1s, 4s, 16s)
       │          - After 3 failures → DLQ for manual review
       │
       ├──→ [queue: "push.notifications"] (durable, classic)  
       │      └── Consumer: PushService
       │          - Auto ACK (fire-and-forget)
       │          - If push gateway is down, message lost (acceptable)
       │          - Prefetch=100 for throughput
       │
       └──→ [queue: "inapp.notifications"] (durable, quorum)
              └── Consumer: InAppService
                  - Manual ACK
                  - Stores in DB (notification center)
                  - If DB write fails → NACK with requeue
                  - x-max-length: 1000000 (bound growth)

  Monitoring:
    - Alert if email queue depth > 10000 (consumer too slow)
    - Alert if DLQ has any messages (investigate failures)
    - Dashboard: messages published/consumed rates per queue
```

</details>

---

## 📝 Interview Q&A

**Q: Compare RabbitMQ's exchange model to Kafka's topic/partition model.**
> A: **RabbitMQ**: Smart broker — exchange routes messages to queues based on rules (routing keys, patterns, headers). Messages are deleted after consumption. Push-based delivery. Best for: complex routing, request-reply, varied messaging patterns, low-latency delivery. **Kafka**: Dumb broker — messages appended to topic partitions (immutable log). Consumers track their own offset. Pull-based. Messages retained for configurable duration (replayable). Best for: event streaming, high throughput, event sourcing, multiple consumers reading same data independently. Key difference: RabbitMQ routes messages AT delivery time; Kafka stores everything and consumers decide what to read. RabbitMQ deletes after consumption; Kafka retains for replay.

---

## 🔗 What to Read Next

1. **[MessagingQ/Kafka.md](./Kafka.md)** — Compare with Kafka's event streaming model
2. **[MessagingQ/MessageQueue_Comparison.md](./MessageQueue_Comparison.md)** — Full comparison matrix
3. **[BuildingBlocks/MessageQueues.md](../BuildingBlocks/MessageQueues.md)** — Messaging fundamentals

---

*[← Kafka](./Kafka.md) | [Back to Index](../INDEX.md) | [Next: AWS SQS/SNS →](./AWS_SQS_SNS.md)*
