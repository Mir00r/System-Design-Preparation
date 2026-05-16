# ⚖️ Message Queue Comparison: Kafka vs RabbitMQ vs SQS vs Redis Streams

> *"Choosing the wrong message queue is one of the most expensive mistakes in distributed system design — it's deeply embedded in your architecture and extremely painful to migrate. This guide helps you choose right the first time."*

**⏱️ Estimated Time**: 20 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Message Queues](../BuildingBlocks/MessageQueues.md), [Kafka](./Kafka.md)

---

## 📊 Complete Comparison Matrix

| Feature | Kafka | RabbitMQ | AWS SQS | Redis Streams |
|---------|-------|----------|---------|---------------|
| **Model** | Distributed log | Message broker | Managed queue | In-memory log |
| **Delivery** | Pull-based | Push-based | Pull-based | Pull-based |
| **Ordering** | Per-partition | Per-queue | FIFO queues only | Per-stream |
| **Throughput** | 1-5M msg/sec | 20-50K msg/sec | Unlimited (managed) | 100-500K msg/sec |
| **Latency** | 5-50ms | ~1ms | 1-10ms | <1ms |
| **Retention** | Days/weeks (disk) | Until consumed | 14 days max | Memory-limited |
| **Replay** | ✅ (offset reset) | ❌ (consumed = gone) | ❌ | ✅ (by ID) |
| **Consumer groups** | ✅ Native | ❌ (competing consumers) | ❌ | ✅ Native |
| **Exactly-once** | ✅ (idempotent producer) | ❌ (at-least-once) | ✅ (FIFO only) | ❌ (at-least-once) |
| **Routing** | Topic + partition key | Exchanges (rich routing) | Basic (SNS for fan-out) | Manual |
| **Dead letter** | ❌ (manual) | ✅ Native | ✅ Native | ❌ (manual) |
| **Multi-DC** | ✅ (MirrorMaker) | ✅ (Shovel/Federation) | ❌ (per-region) | ✅ (Redis Enterprise) |
| **Ops complexity** | 🔴 High | 🟡 Medium | 🟢 Zero (managed) | 🟢 Low |
| **Cost model** | Infrastructure | Infrastructure | Pay-per-request | Infrastructure |
| **Protocol** | Custom (TCP) | AMQP, MQTT, STOMP | HTTP/SQS API | RESP (Redis) |

---

## 🎯 Decision Framework

```
START HERE — What's your PRIMARY need?

┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  "High throughput event streaming + replay"                     │
│  → KAFKA (millions/sec, days of retention, event sourcing)      │
│                                                                 │
│  "Complex routing + request-reply + low latency"                │
│  → RABBITMQ (exchanges, priority queues, RPC patterns)          │
│                                                                 │
│  "Zero ops, serverless, AWS-native"                             │
│  → SQS/SNS (fully managed, infinite scale, pay-per-use)        │
│                                                                 │
│  "Already have Redis, moderate scale, simplicity"               │
│  → REDIS STREAMS (no new infra, consumer groups, fast)          │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘

SECONDARY FACTORS:
  Team size:     < 5 engineers → SQS (no ops) or Redis Streams (simple)
  Scale:         > 500K msg/sec → Kafka or SQS
  Budget:        Infrastructure-constrained → Redis Streams (reuse existing)
  Cloud vendor:  AWS-locked → SQS/SNS; Multi-cloud → Kafka or RabbitMQ
  Compliance:    Data sovereignty → self-hosted (Kafka, RabbitMQ)
```

---

## 🏢 Use Case → Technology Mapping

```
EVENT SOURCING / EVENT STREAMING:
  → Kafka (immutable log, replay, multiple consumers)
  
MICROSERVICE ASYNC COMMUNICATION:
  → RabbitMQ (routing, DLQ, varied patterns)
  → SQS (if on AWS, simpler)

TASK QUEUE (background job processing):
  → SQS (simple, reliable, auto-scaling with Lambda)
  → RabbitMQ (priority queues, complex routing)
  → Redis Streams (if already have Redis, low latency)

REAL-TIME NOTIFICATIONS:
  → Redis Pub/Sub (lowest latency, ephemeral)
  → SNS → Lambda (serverless)
  → RabbitMQ (if need persistence)

LOG AGGREGATION:
  → Kafka (high throughput, retention for reprocessing)
  
IOT DATA INGESTION:
  → Kafka (scale) or MQTT → RabbitMQ (protocol support)

EXACTLY-ONCE FINANCIAL TRANSACTIONS:
  → Kafka (idempotent producer + transactional API)
  → SQS FIFO (exactly-once deduplication)

CHAT / MESSAGING:
  → Redis Streams (low latency, consumer groups per chat room)
  → Kafka (at scale, message history/replay)
```

---

## 📈 Scaling Characteristics

```
KAFKA SCALING:
  Add partitions → parallel consumers (but can't reduce partitions!)
  Add brokers → rebalance partitions across brokers
  Throughput scales linearly with partitions × brokers
  Bottleneck: single partition = single consumer (ordering constraint)

RABBITMQ SCALING:
  Add consumers to queue → competing consumers (work distribution)
  Add nodes to cluster → distribute queues across nodes
  Bottleneck: single queue maxes at ~50K msg/sec
  Solution: consistent hash exchange → spread across multiple queues

SQS SCALING:
  Automatic and invisible! No tuning needed.
  Standard: "nearly unlimited" throughput
  FIFO: 3000 msg/sec (batch) per queue, per message group ID
  Scale FIFO: multiple queues with application-level routing

REDIS STREAMS SCALING:
  Single stream = single Redis thread = 100-500K msg/sec
  Scale: manually shard streams across Redis Cluster nodes
  streams:{0}, streams:{1}, streams:{2} → hash to different nodes
```

---

## 💰 Cost Comparison (1M messages/day)

```
APPROXIMATE MONTHLY COST FOR 1M messages/day:

  Kafka (self-hosted, 3 brokers on m5.large):
    ~$300-500/month (EC2 + EBS storage)
    + operational effort (significant)

  Kafka (Confluent Cloud):
    ~$200-400/month (depending on throughput + storage)
    + reduced ops

  RabbitMQ (self-hosted, 3 nodes on m5.large):
    ~$300-400/month (EC2)
    + moderate operational effort

  AWS SQS:
    ~$12/month (1M messages × 30 days × $0.40/million)
    + zero operational effort!

  Redis Streams (existing Redis cluster):
    $0 incremental (already paying for Redis)
    + minimal ops (Redis is already managed)

  Note: At 100M+ messages/day, Kafka and self-hosted become more
  cost-effective than SQS (per-request pricing adds up).
```

---

## ⚠️ Migration Considerations

```
MIGRATING BETWEEN MESSAGE SYSTEMS:

  Most painful migrations:
    Kafka → RabbitMQ: lose replay, lose consumer groups
    RabbitMQ → Kafka: rewrite all routing logic, lose RPC patterns
    
  Least painful migrations:
    Self-hosted Kafka → Confluent Cloud: same protocol
    Self-hosted RabbitMQ → CloudAMQP: same protocol
    Redis Pub/Sub → Redis Streams: moderate (add ACK logic)

  MIGRATION PATTERN (parallel running):
    1. Deploy new system alongside old
    2. Dual-write: publish to BOTH old and new
    3. Migrate consumers one by one to new system
    4. Verify: compare messages in both systems
    5. Cut over: stop writing to old system
    6. Decommission old system after drain period
```

---

## 🧩 Mini Challenge

**You're building a ride-sharing app. Choose the messaging technology for: (1) real-time driver location updates, (2) ride request matching, (3) payment processing, (4) analytics pipeline. Justify each choice.**

<details>
<summary>💡 Click to reveal answer</summary>

```
RIDE-SHARING MESSAGING ARCHITECTURE:

1. REAL-TIME DRIVER LOCATION UPDATES:
   → Redis Pub/Sub (NOT Streams)
   Reasoning:
   - Locations are ephemeral (old location = useless)
   - Ultra-low latency needed (< 1ms)
   - No persistence needed (latest location only matters)
   - High frequency (every 3-5 seconds per driver)
   - Fire-and-forget is fine (missing one update is OK)

2. RIDE REQUEST MATCHING:
   → Redis Streams (with consumer group)
   Reasoning:
   - Need acknowledgment (can't lose ride requests!)
   - Low latency (users expect match in seconds)
   - Moderate throughput (100K rides/day, not millions)
   - Consumer group: matching service instances compete
   - Already have Redis for locations (no new infra)

3. PAYMENT PROCESSING:
   → SQS FIFO (or Kafka with transactions)
   Reasoning:
   - MUST NOT lose payment events
   - MUST NOT duplicate charge (exactly-once needed)
   - Order matters (charge before refund)
   - SQS FIFO: exactly-once + ordering + DLQ
   - Low-medium volume (1 payment per ride)
   - Zero ops needed for payment-critical path

4. ANALYTICS PIPELINE:
   → Kafka
   Reasoning:
   - High volume (every location update, every event = millions/day)
   - Need replay (reprocess after fixing analytics bug)
   - Multiple consumers (real-time dashboard, batch ML, audit)
   - Long retention (keep 7 days for reprocessing)
   - Throughput matters more than latency
   
SUMMARY:
  Real-time + ephemeral → Redis Pub/Sub
  Reliable + low latency → Redis Streams
  Exactly-once + managed → SQS FIFO
  High-volume + replay  → Kafka
```

</details>

---

## 📝 Interview Q&A

**Q: How would you decide between Kafka and RabbitMQ for a new microservices project?**
> A: I'd ask: (1) **Do you need message replay?** If yes → Kafka. Event sourcing, reprocessing after bugs, or multiple independent consumers reading the same events all need Kafka's immutable log. (2) **What's your throughput?** Over 100K msg/sec sustained → Kafka. Under that → RabbitMQ is simpler. (3) **Do you need complex routing?** RabbitMQ excels at routing messages based on patterns, headers, priorities. Kafka routing is limited to topic+partition. (4) **Do you need request-reply?** RabbitMQ has built-in RPC support. Kafka doesn't. (5) **Team expertise?** Kafka has higher operational complexity (ZooKeeper/KRaft, partition rebalancing, ISR). If the team is small, RabbitMQ or SQS might be more practical. My default: start with SQS/RabbitMQ for simplicity, adopt Kafka only when you have a clear need for event streaming/replay.

---

## 🔗 What to Read Next

1. **[MessagingQ/Kafka.md](./Kafka.md)** — Kafka deep dive
2. **[MessagingQ/RabbitMQ.md](./RabbitMQ.md)** — RabbitMQ deep dive
3. **[MessagingQ/EventStreaming_vs_MessageQueue.md](./EventStreaming_vs_MessageQueue.md)** — Fundamental paradigm differences

---

*[← Redis Streams](./Redis_Streams.md) | [Back to Index](../INDEX.md) | [Next: Event Streaming vs Message Queue →](./EventStreaming_vs_MessageQueue.md)*
