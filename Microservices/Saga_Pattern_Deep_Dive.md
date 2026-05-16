# 🎭 Saga Pattern Deep Dive: Managing Distributed Transactions Across Microservices

> *"In a monolith, a single database transaction ensures 'either all happens or nothing happens.' In microservices, each service has its own database — you can't use a single transaction. Sagas coordinate multi-service operations through a sequence of local transactions with compensating actions."*

**⏱️ Estimated Time**: 35 minutes | **🎯 Difficulty**: 🔴 Advanced | **🔗 Prerequisites**: [Event Sourcing](./EventSourcing.md), [ACID](../Database/ACID.md)

---

## 📋 Table of Contents
1. [The Distributed Transaction Problem](#-the-distributed-transaction-problem)
2. [Choreography vs Orchestration](#-choreography-vs-orchestration)
3. [Choreography Saga](#-choreography-saga)
4. [Orchestration Saga](#-orchestration-saga)
5. [Compensating Transactions](#-compensating-transactions)
6. [Spring Boot Implementation](#-spring-boot-implementation)
7. [Failure Scenarios](#-failure-scenarios)
8. [Common Pitfalls](#-common-pitfalls)
9. [Mini Challenge](#-mini-challenge)
10. [Interview Q&A](#-interview-qa)

---

## 🤔 The Distributed Transaction Problem

```
MONOLITH (single DB transaction):
  BEGIN TRANSACTION
    deduct_inventory(item_id, quantity)
    charge_payment(customer_id, amount)
    create_shipment(order_id, address)
  COMMIT  ← all-or-nothing guaranteed!

MICROSERVICES (each service has own DB):
  [Order Service] → [Inventory Service] → [Payment Service] → [Shipping Service]
       DB1               DB2                  DB3                  DB4
  
  Problem: No single transaction spans all 4 databases!
  
  What if Payment succeeds but Shipping fails?
    → Inventory deducted ✅
    → Payment charged ✅
    → Shipment failed ❌
    → INCONSISTENT STATE! Customer charged but no delivery!

SAGA SOLUTION:
  Execute a SEQUENCE of local transactions
  If any step fails → execute COMPENSATING transactions to undo previous steps
  
  T1: Reserve inventory    C1: Release inventory (compensate)
  T2: Charge payment       C2: Refund payment (compensate)
  T3: Create shipment      C3: Cancel shipment (compensate)
  
  If T3 fails: execute C2 (refund), then C1 (release inventory)
  Result: system returns to consistent state
```

---

## ⚖️ Choreography vs Orchestration

```
CHOREOGRAPHY (decentralized — services react to events):
  Each service publishes an event after its local transaction
  Next service subscribes and reacts
  No central coordinator
  
  [Order] →event→ [Inventory] →event→ [Payment] →event→ [Shipping]
  
  Like a DANCE: each dancer knows their moves, reacts to music/others

ORCHESTRATION (centralized — one coordinator directs):
  A Saga Orchestrator tells each service what to do
  Orchestrator tracks state and handles failures
  
  [Orchestrator] → command → [Inventory]
  [Orchestrator] ← reply
  [Orchestrator] → command → [Payment]
  [Orchestrator] ← reply
  [Orchestrator] → command → [Shipping]
  
  Like an ORCHESTRA: conductor directs each musician when to play

COMPARISON:
  ┌────────────────────┬───────────────────┬───────────────────────┐
  │                    │ Choreography      │ Orchestration          │
  ├────────────────────┼───────────────────┼───────────────────────┤
  │ Coupling           │ Loose (events)    │ Tighter (commands)     │
  │ Visibility         │ Hard to trace     │ Central state machine  │
  │ Complexity (simple)│ Lower             │ Higher (orchestrator)  │
  │ Complexity (many)  │ Spaghetti events  │ Manageable             │
  │ Single failure     │ No SPOF           │ Orchestrator is SPOF   │
  │ Adding services    │ Easy (subscribe)  │ Modify orchestrator    │
  │ Testing            │ Hard (distributed)│ Easier (unit test orch)│
  │ Best for           │ 3-5 simple steps  │ 5+ steps or complex    │
  └────────────────────┴───────────────────┴───────────────────────┘
```

---

## 💃 Choreography Saga

```
HAPPY PATH (Order Creation):

  [OrderService]                    [InventoryService]
       │                                  │
       │── OrderCreated event ──────────▶│
       │                                  │── reserve inventory
       │                                  │
       │                 InventoryReserved event
       │                                  │──────────────────▶ [PaymentService]
       │                                  │                        │
       │                                  │                   charge card
       │                                  │                        │
       │                    PaymentCompleted event                  │
       │◀─────────────────────────────────────────────────────────│
       │                                                           │
       │── mark order confirmed                                    │

FAILURE PATH (Payment fails):

  [OrderService]                    [InventoryService]
       │                                  │
       │── OrderCreated event ──────────▶│
       │                                  │── reserve inventory
       │                                  │
       │                 InventoryReserved event
       │                                  │──────────────────▶ [PaymentService]
       │                                  │                        │
       │                                  │                   PAYMENT FAILS!
       │                                  │                        │
       │                    PaymentFailed event                     │
       │                                  │◀───────────────────────│
       │                                  │
       │                                  │── COMPENSATE: release inventory
       │                                  │
       │              InventoryReleased event
       │◀─────────────────────────────────│
       │
       │── COMPENSATE: mark order failed
```

---

## 🎼 Orchestration Saga

```
SAGA ORCHESTRATOR (state machine):

  States: STARTED → INVENTORY_RESERVED → PAYMENT_CHARGED → SHIPPED → COMPLETED
  
  ┌─────────────────────────────────────────────────────────────────────┐
  │                      SAGA STATE MACHINE                             │
  │                                                                     │
  │  [STARTED] ──reserve inventory──▶ [INVENTORY_PENDING]              │
  │                                        │                            │
  │                         success ──▶ [INVENTORY_RESERVED]            │
  │                         failure ──▶ [FAILED] (no compensation needed)│
  │                                        │                            │
  │              ──charge payment──▶ [PAYMENT_PENDING]                  │
  │                                        │                            │
  │                         success ──▶ [PAYMENT_CHARGED]               │
  │                         failure ──▶ [COMPENSATING_INVENTORY]        │
  │                                        │                            │
  │              ──create shipment──▶ [SHIPPING_PENDING]                │
  │                                        │                            │
  │                         success ──▶ [COMPLETED] ✅                   │
  │                         failure ──▶ [COMPENSATING_PAYMENT]          │
  │                                   then [COMPENSATING_INVENTORY]      │
  │                                   then [FAILED] ❌                   │
  └─────────────────────────────────────────────────────────────────────┘

ORCHESTRATOR COMMANDS:
  Step 1: Send "ReserveInventory" command → wait for reply
  Step 2: Send "ChargePayment" command → wait for reply
  Step 3: Send "CreateShipment" command → wait for reply
  
  On failure at step N: send compensation commands for steps N-1 to 1 (reverse order)
```

---

## ↩️ Compensating Transactions

```
DESIGNING COMPENSATIONS:

  Action                    │ Compensation
  ──────────────────────────┼─────────────────────────────
  Reserve inventory (hold)  │ Release inventory (unhold)
  Charge credit card        │ Refund credit card
  Create shipment           │ Cancel shipment
  Send confirmation email   │ Send cancellation email
  Create account            │ Deactivate/delete account
  Grant access              │ Revoke access

IMPORTANT RULES:
  1. Compensations must be IDEMPOTENT (safe to retry)
  2. Compensations must ALWAYS SUCCEED (can't fail to undo!)
  3. Compensations are SEMANTIC undo (not database rollback)
     - "Refund payment" is not the same as "delete payment record"
     - The charge happened, the refund is a NEW transaction
  4. Some actions are NOT compensatable (sent email, physical shipment)
     → Design to be "pivot transactions" (point of no return)

PIVOT TRANSACTION:
  Steps before pivot: can be compensated (reversible)
  Pivot step: the go/no-go decision point
  Steps after pivot: can RETRY but not compensate (only forward)
  
  Example:
    [Reserve inventory] → [Charge payment (PIVOT)] → [Ship]
    Before pivot: if payment fails, release inventory
    After pivot: payment succeeded, shipping MUST complete (retry until success)
```

---

## 💻 Spring Boot Implementation

```java
// Orchestration Saga with state machine

// Saga State
public enum OrderSagaState {
    STARTED, INVENTORY_PENDING, INVENTORY_RESERVED,
    PAYMENT_PENDING, PAYMENT_CHARGED,
    SHIPPING_PENDING, COMPLETED,
    COMPENSATING_PAYMENT, COMPENSATING_INVENTORY, FAILED
}

// Saga Entity (persisted)
@Entity
public class OrderSaga {
    @Id
    private UUID sagaId;
    @Enumerated(EnumType.STRING)
    private OrderSagaState state;
    private UUID orderId;
    private UUID paymentId;
    private String failureReason;
    private Instant createdAt;
    private Instant updatedAt;
}

// Saga Orchestrator
@Service
public class OrderSagaOrchestrator {
    private final SagaRepository sagaRepo;
    private final InventoryClient inventoryClient;
    private final PaymentClient paymentClient;
    private final ShippingClient shippingClient;
    
    @Transactional
    public UUID startSaga(CreateOrderCommand cmd) {
        OrderSaga saga = new OrderSaga(UUID.randomUUID(), OrderSagaState.STARTED, cmd.orderId());
        sagaRepo.save(saga);
        
        // Step 1: Reserve inventory
        advanceSaga(saga);
        return saga.getSagaId();
    }
    
    @Transactional
    public void handleReply(UUID sagaId, SagaReply reply) {
        OrderSaga saga = sagaRepo.findById(sagaId).orElseThrow();
        
        switch (saga.getState()) {
            case INVENTORY_PENDING -> {
                if (reply.isSuccess()) {
                    saga.setState(OrderSagaState.INVENTORY_RESERVED);
                    advanceSaga(saga);
                } else {
                    saga.setState(OrderSagaState.FAILED);
                    saga.setFailureReason("Inventory unavailable");
                }
            }
            case PAYMENT_PENDING -> {
                if (reply.isSuccess()) {
                    saga.setPaymentId(reply.getPaymentId());
                    saga.setState(OrderSagaState.PAYMENT_CHARGED);
                    advanceSaga(saga);
                } else {
                    saga.setState(OrderSagaState.COMPENSATING_INVENTORY);
                    compensate(saga);
                }
            }
            case SHIPPING_PENDING -> {
                if (reply.isSuccess()) {
                    saga.setState(OrderSagaState.COMPLETED);
                } else {
                    saga.setState(OrderSagaState.COMPENSATING_PAYMENT);
                    compensate(saga);
                }
            }
        }
        sagaRepo.save(saga);
    }
    
    private void advanceSaga(OrderSaga saga) {
        switch (saga.getState()) {
            case STARTED, INVENTORY_RESERVED -> {
                saga.setState(saga.getState() == OrderSagaState.STARTED 
                    ? OrderSagaState.INVENTORY_PENDING : OrderSagaState.PAYMENT_PENDING);
                if (saga.getState() == OrderSagaState.INVENTORY_PENDING)
                    inventoryClient.reserveAsync(saga.getSagaId(), saga.getOrderId());
                else
                    paymentClient.chargeAsync(saga.getSagaId(), saga.getOrderId());
            }
            case PAYMENT_CHARGED -> {
                saga.setState(OrderSagaState.SHIPPING_PENDING);
                shippingClient.createShipmentAsync(saga.getSagaId(), saga.getOrderId());
            }
        }
    }
    
    private void compensate(OrderSaga saga) {
        switch (saga.getState()) {
            case COMPENSATING_PAYMENT -> {
                paymentClient.refundAsync(saga.getSagaId(), saga.getPaymentId());
                saga.setState(OrderSagaState.COMPENSATING_INVENTORY);
            }
            case COMPENSATING_INVENTORY -> {
                inventoryClient.releaseAsync(saga.getSagaId(), saga.getOrderId());
                saga.setState(OrderSagaState.FAILED);
            }
        }
    }
}
```

---

## ⚠️ Common Pitfalls

1. **Non-idempotent compensations** — If a compensation command is delivered twice (network retry), it must produce the same result. "Refund $99" executed twice = $198 refunded! Use idempotency keys: "Refund payment pay_123" — if already refunded, no-op.

2. **Missing compensations for edge cases** — Every forward action MUST have a defined compensation. If you add a step to the saga, you must also add the compensation. Review: "What if this step succeeds but the NEXT step fails?"

3. **Saga state not persisted** — If the orchestrator crashes mid-saga, you lose track of which step you're on. Always persist saga state to a database. On restart, resume from last persisted state.

4. **Mixing choreography and orchestration without clarity** — In complex systems, some sagas use choreography and others use orchestration. Document clearly which pattern each business process uses, or developers will create inconsistent flows.

---

## 🧩 Mini Challenge

**Design a saga for a travel booking: Book Flight → Book Hotel → Book Car Rental. The flight is non-refundable after confirmation. How do you handle a car rental failure?**

<details>
<summary>💡 Click to reveal answer</summary>

```
SAGA DESIGN:

Step 1: RESERVE Flight (tentative hold, not confirmed)
  Compensation: Cancel flight reservation

Step 2: RESERVE Hotel (tentative hold)  
  Compensation: Cancel hotel reservation

Step 3: RESERVE Car Rental
  Compensation: Cancel car reservation

Step 4: CONFIRM Flight (PIVOT — non-refundable after this!)
  No compensation possible!

Step 5: CONFIRM Hotel
  Compensation: Cancel hotel (before check-in date)

Step 6: CONFIRM Car Rental
  Compensation: Cancel car rental

SCENARIO: Car RESERVATION fails (Step 3):
  → Compensate Step 2: cancel hotel reservation
  → Compensate Step 1: cancel flight reservation
  → Saga state: FAILED
  → Notify user: "Car rental unavailable for these dates"

SCENARIO: Car CONFIRMATION fails (Step 6, after flight confirmed):
  → Flight already confirmed (non-refundable) — can't undo!
  → Options:
    a) Retry car confirmation (exponential backoff, different provider)
    b) Book alternative car rental
    c) Accept partial booking (flight + hotel only)
    d) Notify user: "Car couldn't be booked, flight + hotel confirmed"
  
  KEY INSIGHT: The PIVOT point (Step 4: confirm flight) means:
    - Steps 1-3 are all RESERVATIONS (fully compensatable)
    - Only after ALL reservations succeed, we start CONFIRMING
    - Confirmations should retry rather than compensate

REVISED STRATEGY (two phases):
  Phase 1 (all reversible): Reserve flight, hotel, car
  Decision point: all reserved? → proceed : compensate all
  Phase 2 (retry-forward): Confirm flight, hotel, car
  If confirmation fails: RETRY (not compensate), alert on-call
```

</details>

---

## 📝 Interview Q&A

**Q: When would you choose Choreography over Orchestration for a saga?**
> A: **Choreography** (event-driven) works best when: (1) Few steps (3-5 services) — easy to trace mentally. (2) Services are truly independent teams (loose coupling preferred). (3) Simple flows with minimal branching logic. (4) You want to avoid a single point of failure (no orchestrator). **Orchestration** works best when: (1) Many steps or complex branching/retry logic. (2) Need clear visibility of saga progress (single state machine). (3) Business process has many conditional paths. (4) Easier testing (unit test the orchestrator). In practice, **most production systems with > 5 saga steps use orchestration** because debugging "why did this order fail?" is much harder when events are scattered across 8 services vs. checking one orchestrator's state machine.

---

## 🔗 What to Read Next

1. **[Database/ACID.md](../Database/ACID.md)** — The transaction guarantees sagas replace
2. **[Microservices/EventSourcing.md](./EventSourcing.md)** — Event-driven state that powers choreography sagas
3. **[Architectures/CQRS.md](../Architectures/CQRS.md)** — Separating read/write often accompanies sagas

---

*[← Event Sourcing](./EventSourcing.md) | [Back to Index](../INDEX.md) | [Next: Bulkhead Pattern →](./BulkheadPattern.md)*
