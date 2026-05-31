# 🔌 Microservices Communication Patterns: How Services Talk

> *"In a monolith, components talk via function calls (nanoseconds). In microservices, they talk over the network (milliseconds). This 1,000,000x difference changes EVERYTHING about how you design communication."*

**⏱️ Estimated Time**: 25 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Design Guidelines](./Designing_Guidline.md)

---

## 📋 Table of Contents
1. [Sync vs Async: The Fundamental Choice](#-sync-vs-async)
2. [Synchronous Patterns (REST, gRPC)](#-synchronous-patterns)
3. [Asynchronous Patterns (Events, Messages)](#-asynchronous-patterns)
4. [Pattern Comparison Matrix](#-pattern-comparison)
5. [Anti-Patterns (What NOT to Do)](#-anti-patterns)
6. [Interview Q&A](#-interview-qa)
7. [Boss Battle](#-boss-battle)

---

## ⚡ Sync vs Async

```
SYNCHRONOUS (Request/Response):
  Service A ──request──→ Service B
  Service A ←─response── Service B
  
  A WAITS for B to respond. If B is slow, A is slow.
  
  ✅ Simple, intuitive, immediate response
  ❌ Temporal coupling (both must be UP), cascading failures

ASYNCHRONOUS (Event/Message):
  Service A ──event──→ [Message Broker] ──→ Service B
  Service A continues working! (fire-and-forget or eventual response)
  
  ✅ Decoupled (A doesn't know/care about B), resilient, scalable
  ❌ Complex, eventual consistency, harder to debug

THE RULE OF THUMB:
  Need immediate answer? → Synchronous (query)
  Triggering an action?  → Asynchronous (command/event)
```

---

## 🔄 Synchronous Patterns

### REST (HTTP/JSON)

```
WHEN: External APIs, CRUD operations, simple request/response
WHERE: Browser→API, mobile→API, service→service (simple cases)

GET /api/users/42 → { "name": "Alice", "email": "..." }

PROS: Universal, human-readable, cacheable (GET), tooling everywhere
CONS: Verbose (JSON overhead), no streaming, HTTP/1.1 = one-at-a-time

BEST PRACTICE FOR SERVICE-TO-SERVICE:
  ✅ Use for queries (getting data)
  ✅ Set timeouts (500ms max!)
  ✅ Add circuit breaker
  ❌ Don't use for commands that trigger long processes
```

### gRPC (HTTP/2 + Protobuf)

```
WHEN: Internal service-to-service, high performance, streaming
WHERE: Backend microservices talking to each other

PROTO DEFINITION:
  service OrderService {
    rpc GetOrder(OrderId) returns (Order);
    rpc StreamUpdates(OrderId) returns (stream OrderUpdate); // Server streaming!
  }

PROS: 
  - 2-10x faster than REST/JSON (binary Protobuf, less bytes)
  - HTTP/2 multiplexing (parallel requests on 1 connection)
  - Streaming support (server, client, bidirectional)
  - Strong typing (proto schema = contract)
  
CONS:
  - Not browser-friendly (need gRPC-Web or gateway)
  - Binary = harder to debug (need tools)
  - Tighter coupling (shared proto files)

NETFLIX uses gRPC for ALL internal communication!
```

### Comparison: REST vs gRPC

| Aspect | REST | gRPC |
|--------|------|------|
| **Protocol** | HTTP/1.1 (or 2) | HTTP/2 only |
| **Format** | JSON (text) | Protobuf (binary) |
| **Speed** | Baseline | 2-10x faster |
| **Streaming** | SSE / WebSocket (separate) | Built-in (4 patterns) |
| **Browser** | Native | Needs proxy |
| **Contract** | OpenAPI (optional) | Proto (mandatory) |
| **Best For** | External, public APIs | Internal, high-perf |

---

## 📨 Asynchronous Patterns

### Pattern 1: Event-Driven (Pub/Sub)

```
CONCEPT: Services publish events. Interested services subscribe.
  Nobody needs to know WHO is listening!

  Order Service ──→ "OrderPlaced" ──→ [Kafka Topic]
                                         ├──→ Payment Service (charge)
                                         ├──→ Inventory Service (reserve)
                                         └──→ Notification Service (email)

  Order Service doesn't know about Payment, Inventory, or Notification!
  Adding a new subscriber = ZERO changes to publisher.

WHEN TO USE:
  ✅ Multiple consumers need the same event
  ✅ Producer shouldn't care about consumers
  ✅ Eventual consistency is acceptable
  ✅ Fan-out: one event triggers many reactions
```

### Pattern 2: Command Queue (Point-to-Point)

```
CONCEPT: Send a command to exactly ONE consumer via a queue.

  API Gateway ──→ [SendEmail Queue] ──→ Email Worker (one of N instances)
  
  Only ONE worker processes each message (competing consumers).
  If worker dies, message returns to queue → another worker picks it up.

WHEN TO USE:
  ✅ Task must be processed exactly once
  ✅ Work distribution across workers
  ✅ Rate limiting (workers pull at their pace)
  ✅ Guaranteed delivery (message persisted in queue)
```

### Pattern 3: Event Sourcing + CQRS

```
CONCEPT: Store events, not current state. Separate read/write models.

  WRITE: UserService ──→ store event "NameChanged(id=42, name='Bob')"
  READ:  Read Model ←── materialized view (rebuilt from events)
  
  Benefits: Complete audit trail, time travel, different read optimizations
  
See: EventSourcing.md for deep dive!
```

### Pattern 4: Saga (Distributed Transactions)

```
CONCEPT: Multi-step process across services with compensation on failure.

  1. Create Order    ✅
  2. Reserve Stock   ✅  
  3. Charge Payment  ❌ FAILED!
  4. → Compensate: Unreserve Stock, Cancel Order

  Two flavors:
    Orchestration: Central coordinator tells each step what to do
    Choreography: Each service reacts to events and triggers next step

See: Saga_Pattern_Deep_Dive.md for complete implementation!
```

---

## 📊 Pattern Comparison

| Pattern | Coupling | Speed | Reliability | Complexity | Use Case |
|---------|----------|-------|-------------|------------|----------|
| REST | High | Medium | Medium | Low | CRUD, queries |
| gRPC | High | Fast | Medium | Medium | Internal RPC |
| Event (Pub/Sub) | Low | Async | High | Medium | Notifications, fan-out |
| Command Queue | Low | Async | Very High | Medium | Task processing |
| Saga | Medium | Async | High | High | Distributed transactions |

---

## 🚫 Anti-Patterns

```
1. DISTRIBUTED MONOLITH 💀
   Problem: Microservices that MUST deploy together → not really independent
   Sign: Changing one service requires changing 3 others
   Fix: Clear bounded contexts, async events between domains

2. SYNC CHAIN OF DEATH 💀
   Problem: A→B→C→D synchronous chain. D is slow → everything is slow
   Sign: Cascading timeouts, all services slow when one is slow
   Fix: Break chains with async where possible, circuit breakers

3. CHATTY SERVICES 💀
   Problem: 50 HTTP calls to render one page
   Sign: N+1 calls, high network overhead, low throughput
   Fix: Batch endpoints, BFF (Backend for Frontend), GraphQL

4. SHARED DATABASE 💀
   Problem: Multiple services write to same table
   Sign: Schema changes break multiple services, lock contention
   Fix: Database per service, async data replication via events

5. TEMPORAL COUPLING 💀
   Problem: Service A requires Service B to be UP at the same moment
   Sign: One service down → dependent services fail immediately
   Fix: Async messaging, caching, graceful degradation with fallbacks
```

---

## 🎓 Interview Q&A

### Q1: "How do microservices communicate? Which pattern would you choose?"
**A**: It depends on the use case. For queries needing immediate responses, synchronous (REST for simplicity, gRPC for performance). For commands triggering actions, asynchronous (event-driven via Kafka). For distributed transactions, Saga pattern. The key principle: prefer async for inter-domain communication to reduce coupling.

### Q2: "How do you handle failures in synchronous communication?"
**A**: Three layers: 1) Timeouts (never wait forever — 500ms max), 2) Circuit breaker (stop calling a failed service), 3) Fallbacks (return cached/default data). Plus retries with exponential backoff and jitter for transient errors.

### Q3: "Event-driven vs request-driven — trade-offs?"
**A**: Event-driven: loosely coupled, resilient to failures, scales independently, but eventual consistency and harder to debug (need distributed tracing). Request-driven: simple, immediate consistency, easy to reason about, but tight coupling and cascading failure risk. Most production systems use BOTH — sync for queries, async for commands.

---

## 🎲 Boss Battle: Design the Communication 🏗️

> **Scenario**: Design communication patterns for an online banking system:
> - Transfer money between accounts
> - Check balance (real-time)
> - Send transaction notification (email/push)
> - Generate monthly statements
> - Fraud detection on every transaction
>
> For each, specify: sync/async, protocol, why.

<details>
<summary>🔓 Click to reveal answer</summary>

| Operation | Pattern | Protocol | Why |
|-----------|---------|----------|-----|
| **Check balance** | Sync | REST/gRPC | User needs immediate answer |
| **Transfer money** | Saga (async) | Events (Kafka) | Multi-step, needs compensation |
| **Send notification** | Async | Event → Queue | Non-critical, can be delayed |
| **Monthly statements** | Async | Scheduled + Queue | Batch, not time-sensitive |
| **Fraud detection** | Async (real-time stream) | Kafka Streams | Process in parallel, don't block transfer |

**Money Transfer Saga:**
1. Debit source account (sync within Account Service)
2. Credit destination account (async event)
3. If credit fails → compensate: re-credit source
4. Emit TransactionCompleted event → triggers notification + fraud check

**Key Design Decisions:**
- Balance check is the ONLY sync call (user waiting)
- Money transfer looks sync to user but is actually saga internally
- Fraud runs in parallel (doesn't block the transfer, flags suspicious ones)
- Notifications are fire-and-forget via queue (guaranteed delivery, no rush)
</details>

---

👉 **[Next: Saga Pattern →](./Saga_Pattern_Deep_Dive.md)**  
👉 **[Back to Microservices Overview →](./README.md)**
