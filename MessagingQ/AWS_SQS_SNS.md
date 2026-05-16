# ☁️ AWS SQS & SNS: Managed Messaging for Serverless and Cloud-Native

> *"When you need a queue that scales to billions of messages with zero operational overhead — no cluster management, no disk provisioning, no replication setup — SQS just works. Pair it with SNS for fan-out and you have a serverless messaging backbone."*

**⏱️ Estimated Time**: 30 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Message Queues](../BuildingBlocks/MessageQueues.md)

---

## 📋 Table of Contents
1. [SQS Overview](#-sqs-overview)
2. [SNS Overview](#-sns-overview)
3. [SQS + SNS Fan-Out Pattern](#-sqs--sns-fan-out-pattern)
4. [FIFO vs Standard Queues](#-fifo-vs-standard-queues)
5. [Lambda Integration](#-lambda-integration)
6. [Spring Boot Integration](#-spring-boot-integration)
7. [Best Practices](#-best-practices)
8. [Common Pitfalls](#-common-pitfalls)
9. [Mini Challenge](#-mini-challenge)
10. [Interview Q&A](#-interview-qa)

---

## 📬 SQS Overview

```
┌─────────────────────────────────────────────────────────┐
│                  AWS SQS AT A GLANCE                     │
│                                                         │
│  Type:        Fully managed message queue               │
│  Model:       Pull-based (consumers poll for messages)  │
│  Throughput:  Nearly unlimited (auto-scales)            │
│  Retention:   1 minute to 14 days (default: 4 days)    │
│  Message size:256KB max (use S3 for larger)            │
│  Delivery:    At-least-once (standard) / Exactly-once (FIFO)│
│  Pricing:     Per-request ($0.40 per million requests)  │
│  Latency:     Standard: ~ms, FIFO: ~ms                  │
└─────────────────────────────────────────────────────────┘

HOW SQS WORKS:
  [Producer] → SendMessage → [SQS Queue] → ReceiveMessage → [Consumer]
  
  1. Producer sends message to queue
  2. Message stored redundantly across multiple AZs
  3. Consumer polls queue (long polling recommended)
  4. Message becomes "invisible" (visibility timeout)
  5. Consumer processes and deletes message
  6. If not deleted within visibility timeout → reappears for retry

VISIBILITY TIMEOUT:
  ┌──────────────────────────────────────────────────────┐
  │ Timeline:                                            │
  │                                                      │
  │ [Msg in queue] → [Consumer receives] → [Invisible]  │
  │                   │                     (30s default)│
  │                   │                     │            │
  │                   │    Consumer processes│            │
  │                   │    and DELETEs   ────┘            │
  │                   │                                  │
  │                   │    OR: timeout expires            │
  │                   │    → message visible again        │
  │                   │    → another consumer gets it     │
  └──────────────────────────────────────────────────────┘
```

---

## 📢 SNS Overview

```
┌─────────────────────────────────────────────────────────┐
│                  AWS SNS AT A GLANCE                     │
│                                                         │
│  Type:        Pub/Sub notification service              │
│  Model:       Push-based (SNS pushes to subscribers)    │
│  Subscribers: SQS, Lambda, HTTP/S, Email, SMS, Mobile   │
│  Throughput:  Nearly unlimited                          │
│  Ordering:    FIFO topics available                     │
│  Filtering:   Message attribute-based filtering         │
│  Fanout:      One message → many subscribers            │
└─────────────────────────────────────────────────────────┘

HOW SNS WORKS:
  [Publisher] → SNS Topic → [Subscriber 1: SQS Queue]
                           → [Subscriber 2: Lambda]
                           → [Subscriber 3: HTTP endpoint]
                           → [Subscriber 4: Email]

MESSAGE FILTERING (subscribe to subset):
  SNS Topic: "orders"
  
  Subscriber 1 filter: { "order_type": ["premium"] }
    → Only receives premium orders
  
  Subscriber 2 filter: { "region": ["us-east", "us-west"] }
    → Only receives US orders
  
  Subscriber 3: no filter
    → Receives ALL orders
```

---

## 🔀 SQS + SNS Fan-Out Pattern

```
THE MOST COMMON AWS MESSAGING PATTERN:

  [OrderService]
       │
       │ publish order event
       ▼
  [SNS Topic: "order-events"]
       │
       ├──▶ [SQS: billing-queue] → [BillingService]
       │     (processes invoices)
       │
       ├──▶ [SQS: shipping-queue] → [ShippingService]
       │     (creates shipment)
       │
       ├──▶ [SQS: analytics-queue] → [AnalyticsService]
       │     (tracks metrics)
       │
       └──▶ [Lambda: notification] → sends email/push
             (instant notification)

WHY SNS + SQS (not just SNS alone):
  - SQS provides BUFFERING (if consumer is slow/down, messages wait)
  - SQS provides RETRY (failed messages become visible again)
  - SQS provides DLQ (poison messages go to dead letter queue)
  - SNS alone is fire-and-forget (if subscriber is down, message lost)

  SNS = fan-out (one → many)
  SQS = reliable delivery (buffer + retry + DLQ)
  Together = reliable fan-out
```

---

## 🔢 FIFO vs Standard Queues

```
┌─────────────────────────────────────────────────────────────────┐
│ Feature          │ Standard Queue       │ FIFO Queue             │
├──────────────────┼──────────────────────┼────────────────────────┤
│ Ordering         │ Best-effort          │ Strict FIFO            │
│ Delivery         │ At-least-once        │ Exactly-once           │
│ Throughput       │ Unlimited            │ 3,000 msg/sec (batch)  │
│                  │                      │ 300 msg/sec (no batch) │
│ Deduplication    │ No                   │ 5-min dedup window     │
│ Queue name       │ Any name             │ Must end in .fifo      │
│ Use case         │ High throughput,     │ Order matters,         │
│                  │ duplicates OK        │ no duplicates          │
└──────────────────┴──────────────────────┴────────────────────────┘

FIFO MESSAGE GROUPS (parallelism within FIFO):
  MessageGroupId = "user-123" → all messages for user-123 in order
  MessageGroupId = "user-456" → all messages for user-456 in order
  
  Different group IDs processed in parallel!
  Same group ID processed in strict order.
  
  Use case: process each user's events in order, but users are independent
```

---

## ⚡ Lambda Integration

```
SQS → Lambda (event-driven, serverless processing):

  [SQS Queue] ──event source mapping──▶ [Lambda Function]
  
  How it works:
    1. Lambda service polls SQS on your behalf
    2. When messages available, invokes your Lambda
    3. Batch of messages (1-10000) passed to one invocation
    4. On success: messages auto-deleted
    5. On failure: messages become visible again (retry)
    6. After N retries: sent to DLQ

  Configuration:
    BatchSize: 10 (process 10 messages per Lambda invocation)
    MaximumBatchingWindowInSeconds: 5 (wait up to 5s to fill batch)
    MaximumConcurrency: 50 (max 50 Lambda instances in parallel)

SNS → Lambda (instant push):
  [SNS Topic] ──subscription──▶ [Lambda Function]
  
  Invoked immediately when message published (no polling).
  No retry/DLQ unless you configure Lambda destination.
  Best for: real-time notifications, lightweight processing.
```

---

## 💻 Spring Boot Integration

```java
// build.gradle
implementation 'io.awspring.cloud:spring-cloud-aws-messaging:3.0.0'
implementation 'software.amazon.awssdk:sqs:2.20.0'
implementation 'software.amazon.awssdk:sns:2.20.0'

// Configuration
@Configuration
public class AwsMessagingConfig {
    @Bean
    public SqsClient sqsClient() {
        return SqsClient.builder()
            .region(Region.US_EAST_1)
            .build();
    }
    
    @Bean
    public SnsClient snsClient() {
        return SnsClient.builder()
            .region(Region.US_EAST_1)
            .build();
    }
}

// Producer: Publish to SNS Topic
@Service
public class OrderEventPublisher {
    private final SnsClient snsClient;
    private final String topicArn = "arn:aws:sns:us-east-1:123456:order-events";
    
    public void publishOrderCreated(Order order) {
        PublishRequest request = PublishRequest.builder()
            .topicArn(topicArn)
            .message(objectMapper.writeValueAsString(order))
            .messageAttributes(Map.of(
                "event_type", MessageAttributeValue.builder()
                    .dataType("String").stringValue("ORDER_CREATED").build(),
                "region", MessageAttributeValue.builder()
                    .dataType("String").stringValue(order.getRegion()).build()
            ))
            .build();
        
        snsClient.publish(request);
    }
}

// Consumer: Listen to SQS Queue
@Component
public class BillingConsumer {
    
    @SqsListener(value = "billing-queue", deletionPolicy = SqsMessageDeletionPolicy.ON_SUCCESS)
    public void handleOrderEvent(@Payload Order order, 
                                  @Header("ApproximateReceiveCount") int receiveCount) {
        if (receiveCount > 3) {
            log.error("Message failed {} times, sending to DLQ", receiveCount);
            throw new PermanentFailureException("Max retries exceeded");
        }
        
        billingService.createInvoice(order);
        // Message auto-deleted on success (deletionPolicy = ON_SUCCESS)
    }
}

// Direct SQS operations (fine-grained control)
@Service
public class QueueService {
    private final SqsClient sqsClient;
    
    // Send with delay
    public void sendDelayedMessage(String queueUrl, String body, int delaySeconds) {
        sqsClient.sendMessage(SendMessageRequest.builder()
            .queueUrl(queueUrl)
            .messageBody(body)
            .delaySeconds(delaySeconds)  // 0-900 seconds
            .build());
    }
    
    // Long polling (efficient — waits up to 20s for messages)
    public List<Message> receiveMessages(String queueUrl) {
        return sqsClient.receiveMessage(ReceiveMessageRequest.builder()
            .queueUrl(queueUrl)
            .maxNumberOfMessages(10)
            .waitTimeSeconds(20)  // long polling!
            .visibilityTimeout(60)
            .build()).messages();
    }
}
```

---

## ✅ Best Practices

```
SQS BEST PRACTICES:
  1. Long polling (waitTimeSeconds=20): reduces empty responses & cost
  2. Batch operations: send/receive/delete up to 10 messages at once
  3. Set appropriate visibility timeout: > max processing time
  4. Always configure DLQ: catch poison messages (maxReceiveCount=3)
  5. Use message deduplication: idempotency key in message ID
  6. Large messages: store body in S3 + SQS carries S3 pointer

SNS BEST PRACTICES:
  1. Use message filtering: don't send everything to every subscriber
  2. SNS + SQS for reliable fan-out (not raw SNS → HTTP)
  3. Set delivery retry policies for HTTP endpoints
  4. Use FIFO topics with FIFO queues for ordered fan-out
  5. Enable server-side encryption (SSE) for sensitive data
```

---

## ⚠️ Common Pitfalls

1. **Short polling wasting money** — Default short polling returns immediately (often with no messages). You pay per request. Long polling (20s wait) reduces empty responses by 90%+. Always set `WaitTimeSeconds=20`.

2. **Visibility timeout too short** — If processing takes 45 seconds but timeout is 30 seconds, the message becomes visible and another consumer processes it (duplicate processing!). Set timeout to 2-3x your expected processing time.

3. **Not handling duplicates (Standard queues)** — Standard SQS delivers at-least-once. Your consumer MUST be idempotent. Use a deduplication store (Redis SETNX with message ID, or database unique constraint).

4. **FIFO queue throughput limit** — FIFO queues cap at 3,000 msg/sec with batching. If you need higher throughput, use multiple FIFO queues with consistent hashing, or switch to Standard queues with application-level ordering.

---

## 🧩 Mini Challenge

**Design an order processing system using SQS and SNS. Requirements: order events fan out to 4 services, payment processing must be exactly-once, and failed messages should be retried 3 times before going to a dead letter queue.**

<details>
<summary>💡 Click to reveal answer</summary>

```
Architecture:

[OrderService] → [SNS Topic: "orders" (FIFO)] 
                    │
                    ├──▶ [SQS FIFO: payment.fifo]
                    │     - MessageGroupId: order_id
                    │     - ContentBasedDeduplication: true
                    │     - Exactly-once delivery
                    │     - maxReceiveCount: 3
                    │     - DLQ: payment-dlq.fifo
                    │     └── [PaymentService] (idempotent with order_id)
                    │
                    ├──▶ [SQS Standard: inventory-queue]
                    │     - At-least-once (idempotent consumer)
                    │     - maxReceiveCount: 3
                    │     - DLQ: inventory-dlq
                    │     └── [InventoryService]
                    │
                    ├──▶ [SQS Standard: notification-queue]
                    │     - DelaySeconds: 5 (slight delay OK)
                    │     - maxReceiveCount: 5 (more retries for email)
                    │     - DLQ: notification-dlq
                    │     └── [NotificationService]
                    │
                    └──▶ [Lambda: analytics-processor]
                          - Direct SNS → Lambda (real-time)
                          - Writes to analytics DB
                          - onFailure → SQS DLQ

DLQ Processing:
  CloudWatch Alarm on DLQ message count > 0
  → Alert on-call engineer
  → Manual investigation or automated replay
  
  DLQ replay: move messages from DLQ back to source queue after fix
```

</details>

---

## 📝 Interview Q&A

**Q: When would you choose SQS over Kafka?**
> A: Choose SQS when: (1) You want zero operational overhead (no cluster management, no Zookeeper, fully managed). (2) You're building on AWS with Lambda (native integration, event source mapping). (3) Message volume is variable (auto-scales to any load, pay per request). (4) You don't need message replay (messages deleted after processing). (5) Team is small and doesn't want to manage Kafka clusters. Choose Kafka when: (1) Need message replay/reprocessing (immutable log). (2) Need very high throughput with ordering (millions/sec). (3) Multiple consumers need to read the same messages independently. (4) Event sourcing architecture. (5) Need to process messages in exact order across partitions.

---

## 🔗 What to Read Next

1. **[MessagingQ/Kafka.md](./Kafka.md)** — Compare with Kafka's event streaming
2. **[MessagingQ/MessageQueue_Comparison.md](./MessageQueue_Comparison.md)** — Full comparison matrix
3. **[Cloud/AWS/Core_Services.md](../Cloud/AWS/Core_Services.md)** — AWS service ecosystem

---

*[← RabbitMQ](./RabbitMQ.md) | [Back to Index](../INDEX.md) | [Next: Redis Streams →](./Redis_Streams.md)*
