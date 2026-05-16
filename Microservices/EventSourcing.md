# 📜 Event Sourcing: Storing State as a Sequence of Events

> *"Instead of storing 'account balance = $500', store every event: 'deposited $1000', 'withdrew $300', 'transferred $200'. You can always derive the current state, but you can never derive the history from just the current state."*

**⏱️ Estimated Time**: 35 minutes | **🎯 Difficulty**: 🔴 Advanced | **🔗 Prerequisites**: [Event Streaming vs Message Queue](../MessagingQ/EventStreaming_vs_MessageQueue.md), [CQRS](../Architectures/CQRS.md)

---

## 📋 Table of Contents
1. [What is Event Sourcing](#-what-is-event-sourcing)
2. [Traditional vs Event-Sourced](#-traditional-vs-event-sourced)
3. [Event Store](#-event-store)
4. [Projections & Read Models](#-projections--read-models)
5. [Snapshots](#-snapshots)
6. [Spring Boot Implementation](#-spring-boot-implementation)
7. [When to Use / Not Use](#-when-to-use--not-use)
8. [Common Pitfalls](#-common-pitfalls)
9. [Mini Challenge](#-mini-challenge)
10. [Interview Q&A](#-interview-qa)

---

## 🤔 What is Event Sourcing

```
TRADITIONAL (State-based):
  Database stores CURRENT STATE only:
  ┌────────────────────────────────────┐
  │ accounts table                     │
  │ id=1, balance=500, name="Alice"    │  ← What's the balance? $500.
  └────────────────────────────────────┘     But HOW did it get to $500? 🤷

EVENT SOURCING (Event-based):
  Database stores ALL EVENTS that led to current state:
  ┌────────────────────────────────────────────────────┐
  │ event_store                                        │
  │ 1. AccountCreated { id=1, name="Alice" }           │
  │ 2. MoneyDeposited { id=1, amount=1000 }            │
  │ 3. MoneyWithdrawn { id=1, amount=300 }             │
  │ 4. MoneyTransferred { id=1, to=2, amount=200 }    │
  └────────────────────────────────────────────────────┘
  
  Current state: replay all events → balance = 1000 - 300 - 200 = $500
  PLUS: full audit trail, can answer "what happened at 2pm yesterday?"

KEY PRINCIPLES:
  1. Events are IMMUTABLE facts (never updated or deleted)
  2. Events are the SINGLE SOURCE OF TRUTH
  3. Current state is DERIVED by replaying events
  4. New events are APPENDED (append-only log)
```

---

## ⚖️ Traditional vs Event-Sourced

```
TRADITIONAL CRUD:
  Create: INSERT INTO orders (id, status, total) VALUES (1, 'pending', 99.99)
  Update: UPDATE orders SET status = 'confirmed' WHERE id = 1
  Update: UPDATE orders SET status = 'shipped' WHERE id = 1
  
  Result: orders table shows { id=1, status='shipped', total=99.99 }
  Lost info: When was it confirmed? Who confirmed it? Was it ever cancelled and re-confirmed?

EVENT SOURCED:
  Append: OrderCreated { id=1, items=[...], total=99.99, customer="alice" }
  Append: OrderConfirmed { id=1, confirmed_by="bob", timestamp="2024-01-15T10:30" }
  Append: PaymentReceived { id=1, payment_id="pay_123", amount=99.99 }
  Append: OrderShipped { id=1, tracking="UPS123", carrier="UPS" }
  
  Result: Full history preserved! Current state derived by replaying.
  Bonus: Can rebuild any past state (state at any point in time)

COMPARISON:
  ┌───────────────────┬──────────────────┬───────────────────────┐
  │                   │ Traditional CRUD │ Event Sourcing         │
  ├───────────────────┼──────────────────┼───────────────────────┤
  │ Storage           │ Current state    │ All events (append)   │
  │ History           │ Lost on update   │ Complete audit trail   │
  │ Debugging         │ "Why is it $500?"│ Replay events to see  │
  │ Schema evolution  │ ALTER TABLE      │ Add new event types   │
  │ Read performance  │ Fast (direct)    │ Slower (replay/project)│
  │ Write performance │ May need locks   │ Append-only (fast)    │
  │ Complexity        │ Simple           │ Higher                │
  │ Storage size      │ Current only     │ Grows over time       │
  └───────────────────┴──────────────────┴───────────────────────┘
```

---

## 🗃️ Event Store

```
EVENT STORE = Append-only database optimized for event sourcing

STRUCTURE:
  ┌──────────────────────────────────────────────────────────────────┐
  │ stream_id    │ version │ event_type       │ data          │ time │
  ├──────────────┼─────────┼──────────────────┼───────────────┼──────┤
  │ order-123    │ 1       │ OrderCreated     │ {items, total}│ T1   │
  │ order-123    │ 2       │ OrderConfirmed   │ {by: "bob"}   │ T2   │
  │ order-123    │ 3       │ PaymentReceived  │ {amount: 99}  │ T3   │
  │ order-456    │ 1       │ OrderCreated     │ {items, total}│ T4   │
  │ order-123    │ 4       │ OrderShipped     │ {tracking}    │ T5   │
  └──────────────────────────────────────────────────────────────────┘

OPERATIONS:
  Append event to stream: 
    append("order-123", event, expectedVersion=3)
    // Optimistic concurrency: fails if current version != 3
    // Prevents concurrent conflicting writes

  Read stream (load aggregate):
    events = readStream("order-123", fromVersion=0)
    order = new Order()
    for event in events: order.apply(event)
    // Now order has current state

IMPLEMENTATIONS:
  EventStoreDB:   Purpose-built (gRPC API, projections, subscriptions)
  PostgreSQL:     events table + optimistic locking (simple, proven)
  Kafka:          Topic per aggregate type, compaction for snapshots
  DynamoDB:       Stream-id as partition key, version as sort key
```

---

## 📊 Projections & Read Models

```
PROBLEM: Reading by replaying events is slow for complex queries
  "Show me all orders > $100 shipped to NYC this month"
  → Can't efficiently query an event stream for this!

SOLUTION: PROJECTIONS (pre-computed read models)

  Events → [Projection Handler] → Read Model (optimized for queries)

  ┌─────────────────────────────────────────────────────────────────┐
  │ EVENT STORE                    READ MODELS                      │
  │ (write side)                   (read side — CQRS)               │
  │                                                                 │
  │ OrderCreated ──────────┐                                        │
  │ OrderConfirmed ────────┤                                        │
  │ OrderShipped ──────────┼──▶ [Order Summary Table]               │
  │ OrderCancelled ────────┘    id | status | total | customer      │
  │                             1  | shipped| 99.99 | alice         │
  │                                                                 │
  │ Same events ───────────────▶ [Revenue Dashboard]                │
  │                              month    | revenue | order_count    │
  │                              2024-01  | $50000  | 523           │
  │                                                                 │
  │ Same events ───────────────▶ [Customer Order History]           │
  │                              customer | orders | total_spent     │
  │                              alice    | 15     | $2340          │
  └─────────────────────────────────────────────────────────────────┘

  KEY INSIGHT: Multiple read models from same events!
  Each optimized for different query patterns.
  Can rebuild any read model by replaying all events.
  Can create NEW read models for new requirements (without schema migration!)
```

---

## 📸 Snapshots

```
PROBLEM: Aggregate with 10,000 events → slow to replay on every read

SOLUTION: SNAPSHOTS (periodic state capture)

  Events:    [e1] [e2] [e3] ... [e5000] [SNAPSHOT] [e5001] ... [e5050]
                                            │
                                    { balance: 4500,
                                      status: "active",
                                      version: 5000 }

  To load current state:
    1. Load latest snapshot (version 5000, state = {...})
    2. Replay only events AFTER snapshot (e5001 to e5050)
    3. Apply 50 events instead of 5050 → much faster!

  Snapshot strategy:
    - Every N events (e.g., snapshot every 100 events)
    - Periodically (e.g., snapshot daily)
    - On read if stale (lazy snapshotting)
```

---

## 💻 Spring Boot Implementation

```java
// Domain Events
public sealed interface OrderEvent {
    UUID orderId();
    Instant occurredAt();
    
    record OrderCreated(UUID orderId, UUID customerId, List<Item> items, 
                        BigDecimal total, Instant occurredAt) implements OrderEvent {}
    record OrderConfirmed(UUID orderId, String confirmedBy, 
                          Instant occurredAt) implements OrderEvent {}
    record OrderShipped(UUID orderId, String trackingNumber, 
                        Instant occurredAt) implements OrderEvent {}
    record OrderCancelled(UUID orderId, String reason, 
                          Instant occurredAt) implements OrderEvent {}
}

// Aggregate (applies events to build state)
public class Order {
    private UUID id;
    private OrderStatus status;
    private BigDecimal total;
    private List<OrderEvent> uncommittedEvents = new ArrayList<>();
    
    // Command: create order
    public static Order create(UUID customerId, List<Item> items) {
        Order order = new Order();
        BigDecimal total = items.stream().map(Item::price).reduce(BigDecimal.ZERO, BigDecimal::add);
        order.raise(new OrderCreated(UUID.randomUUID(), customerId, items, total, Instant.now()));
        return order;
    }
    
    // Command: confirm order
    public void confirm(String confirmedBy) {
        if (status != OrderStatus.PENDING) throw new IllegalStateException("Can't confirm " + status);
        raise(new OrderConfirmed(id, confirmedBy, Instant.now()));
    }
    
    // Apply event (state transition — NO side effects!)
    private void apply(OrderEvent event) {
        switch (event) {
            case OrderCreated e -> { this.id = e.orderId(); this.status = OrderStatus.PENDING; this.total = e.total(); }
            case OrderConfirmed e -> { this.status = OrderStatus.CONFIRMED; }
            case OrderShipped e -> { this.status = OrderStatus.SHIPPED; }
            case OrderCancelled e -> { this.status = OrderStatus.CANCELLED; }
        }
    }
    
    private void raise(OrderEvent event) {
        apply(event);
        uncommittedEvents.add(event);
    }
    
    // Reconstitute from history
    public static Order fromHistory(List<OrderEvent> events) {
        Order order = new Order();
        events.forEach(order::apply);
        return order;
    }
}

// Event Store (PostgreSQL implementation)
@Repository
public class PostgresEventStore {
    private final JdbcTemplate jdbc;
    
    public void append(UUID streamId, List<OrderEvent> events, int expectedVersion) {
        for (int i = 0; i < events.size(); i++) {
            int version = expectedVersion + i + 1;
            int rows = jdbc.update("""
                INSERT INTO events (stream_id, version, event_type, data, created_at)
                VALUES (?, ?, ?, ?::jsonb, ?)
                ON CONFLICT (stream_id, version) DO NOTHING
                """, streamId, version, events.get(i).getClass().getSimpleName(),
                serialize(events.get(i)), events.get(i).occurredAt());
            
            if (rows == 0) throw new OptimisticLockingException(
                "Concurrent modification on stream " + streamId);
        }
    }
    
    public List<OrderEvent> readStream(UUID streamId) {
        return jdbc.query(
            "SELECT event_type, data FROM events WHERE stream_id = ? ORDER BY version",
            (rs, row) -> deserialize(rs.getString("event_type"), rs.getString("data")),
            streamId);
    }
}

// Application Service
@Service
@Transactional
public class OrderService {
    private final PostgresEventStore eventStore;
    private final ApplicationEventPublisher publisher;
    
    public UUID createOrder(CreateOrderCommand cmd) {
        Order order = Order.create(cmd.customerId(), cmd.items());
        eventStore.append(order.getId(), order.getUncommittedEvents(), 0);
        order.getUncommittedEvents().forEach(publisher::publishEvent); // for projections
        return order.getId();
    }
    
    public void confirmOrder(UUID orderId, String confirmedBy) {
        List<OrderEvent> history = eventStore.readStream(orderId);
        Order order = Order.fromHistory(history);
        order.confirm(confirmedBy);
        eventStore.append(orderId, order.getUncommittedEvents(), history.size());
        order.getUncommittedEvents().forEach(publisher::publishEvent);
    }
}
```

---

## ⚖️ When to Use / Not Use

| Use Event Sourcing When | Avoid Event Sourcing When |
|---|---|
| Audit trail is a business requirement (finance, healthcare) | Simple CRUD with no audit needs |
| Need to answer "what happened?" not just "what is?" | Team is small / unfamiliar with the pattern |
| Complex domain with many state transitions | Read-heavy with simple queries |
| Need to rebuild/replay state for debugging | Data has no meaningful lifecycle (static reference data) |
| Multiple read models needed (CQRS) | Strict consistency between write and read is required |
| Temporal queries ("state at time T") | GDPR deletion requirements (events are immutable!) |

---

## ⚠️ Common Pitfalls

1. **Storing too much in events** — Events should capture what HAPPENED, not derived data. Store `PriceChanged{new_price: 50}`, not `PriceChanged{new_price: 50, old_price: 40, difference: 10, percent_change: 25}`. Derived data can be computed from the event.

2. **Not handling event schema evolution** — Events are immutable and stored forever. When you change event structure, old events still exist. Use versioning (`OrderCreatedV2`) or upcasters (transform old events to new schema on read).

3. **GDPR "right to erasure"** conflicts with immutability — Events are supposed to be immutable, but GDPR requires data deletion. Solutions: crypto-shredding (encrypt personal data with per-user key, delete key to "erase"), or store personal data in a separate mutable store referenced by ID.

4. **Using event sourcing everywhere** — Not every aggregate needs event sourcing. A simple "UserPreferences" entity that gets updated rarely doesn't benefit from full event sourcing. Use it where the audit trail and temporal queries provide real business value.

---

## 🧩 Mini Challenge

**An e-commerce order goes through: Created → PaymentReceived → Confirmed → Shipped → Delivered. A customer disputes the charge. Show the events needed and how you'd determine the order state at the time of dispute.**

<details>
<summary>💡 Click to reveal answer</summary>

```
EVENT STREAM for order-789:
  [1] OrderCreated     { items: [{sku: "ABC", qty: 2, price: 49.99}], total: 99.98, t: Jan 10 }
  [2] PaymentReceived  { payment_id: "pay_456", amount: 99.98, method: "visa_4242", t: Jan 10 }
  [3] OrderConfirmed   { confirmed_by: "system", t: Jan 10 }
  [4] OrderShipped     { tracking: "UPS123", carrier: "UPS", t: Jan 12 }
  [5] OrderDelivered   { signed_by: "Alice", delivered_at: "Jan 14 2:30pm", t: Jan 14 }
  [6] DisputeOpened    { reason: "unauthorized", dispute_id: "disp_001", t: Jan 20 }

INVESTIGATING THE DISPUTE:
  
  Q: "Was the order actually delivered?"
  → Replay to event 5: yes, signed by "Alice" on Jan 14
  
  Q: "What payment method was used?"
  → Event 2: Visa ending in 4242
  
  Q: "Timeline of events?"
  → Complete audit trail from events 1-6
  
  Q: "State at time of dispute (Jan 20)?"
  → Replay events 1-5: status=DELIVERED, total=$99.98
  → Then apply event 6: status=DISPUTED
  
  Q: "What if we need to refund?"
  → Append: RefundIssued { amount: 99.98, reason: "dispute resolution", t: Jan 25 }
  → Append: DisputeResolved { resolution: "refunded", t: Jan 25 }
  
  BENEFIT: Complete, immutable record for compliance and dispute resolution.
  With CRUD, we'd only know current status ("refunded") — not the full story.
```

</details>

---

## 📝 Interview Q&A

**Q: How does Event Sourcing relate to CQRS? Are they always used together?**
> A: They're complementary but independent. **CQRS** (Command Query Responsibility Segregation) separates read and write models. **Event Sourcing** stores state as events instead of current values. They pair well because: event sourcing makes the write side complex (replay events), so you build optimized read models (projections) for queries — which IS CQRS. However, you can have CQRS without event sourcing (separate read replicas), and event sourcing without CQRS (replay events for every read — simple but slow). In practice, event sourcing almost always implies CQRS because querying raw events is impractical for most read patterns.

---

## 🔗 What to Read Next

1. **[Architectures/CQRS.md](../Architectures/CQRS.md)** — Command Query separation (natural companion)
2. **[MessagingQ/Kafka.md](../MessagingQ/Kafka.md)** — Kafka as an event store
3. **[Microservices/Saga_Pattern_Deep_Dive.md](./Saga_Pattern_Deep_Dive.md)** — Distributed transactions with events

---

*[← Service Mesh](./ServiceMesh.md) | [Back to Index](../INDEX.md) | [Next: Saga Pattern →](./Saga_Pattern_Deep_Dive.md)*
