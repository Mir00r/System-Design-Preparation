# 🏛️ Domain-Driven Design (DDD): Modeling Complex Business Logic

> *"The heart of software is its ability to solve domain-related problems for its user. All other features, vital though they may be, support this basic purpose." — Eric Evans*

**⏱️ Estimated Time**: 25 minutes | **🎯 Difficulty**: 🔴 Hard | **🔗 Prerequisites**: [Clean Code](./Clean_Code_Principles.md), [Microservices](../Microservices/)

---

## 🤔 Why DDD?

```
THE PROBLEM:
  Business experts speak: "When a customer places an order,
  we need to verify their credit limit and reserve inventory."
  
  Developers write: CustomerOrderProcessingServiceImpl.java
  → processOrderV2Final() → 500 lines of spaghetti logic
  → Business rules buried in if-else chains
  → Nobody understands what the code actually does

DDD SOLUTION:
  Code mirrors the business language:
  
  customer.placeOrder(order);        // Domain model speaks business
  creditPolicy.verify(customer);     // Named concepts
  inventory.reserve(order.items());  // Clear responsibilities
  
  Business expert reads the code → "Yes, that's exactly right!"
```

---

## 🏗️ Strategic DDD: Bounded Contexts

```
BOUNDED CONTEXT = A clear boundary around a domain model
  where terms have specific meaning.

EXAMPLE: E-commerce Platform
  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
  │  ORDERING        │  │  INVENTORY       │  │  SHIPPING        │
  │  Context         │  │  Context         │  │  Context         │
  │                  │  │                  │  │                  │
  │  "Product" =     │  │  "Product" =     │  │  "Product" =     │
  │  name, price,    │  │  SKU, quantity,   │  │  weight, dims,   │
  │  description     │  │  warehouse loc   │  │  fragile flag    │
  │                  │  │                  │  │                  │
  │  "Customer" =    │  │  (no Customer    │  │  "Customer" =    │
  │  name, email,    │  │   concept here)  │  │  address, phone  │
  │  payment method  │  │                  │  │                  │
  └────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘
           │                     │                     │
           └─── Events ──────────┴──── Events ─────────┘
  
  KEY INSIGHT: "Product" means DIFFERENT things in different contexts!
  Don't force one Product class to serve all needs.
  Each context has its own model, optimized for its concerns.
```

---

## 💻 Tactical DDD Building Blocks

```java
// ═══════════════════════════════════════════════════════════════
// ENTITY: Has identity, mutable, lifecycle
// ═══════════════════════════════════════════════════════════════
public class Order {
    private final OrderId id;           // Identity (unique)
    private OrderStatus status;         // Mutable state
    private List<OrderLine> lines;
    private Money totalAmount;
    
    public void addItem(Product product, int quantity) {
        // Business rule INSIDE the entity
        if (status != OrderStatus.DRAFT) {
            throw new IllegalStateException("Cannot modify confirmed order");
        }
        lines.add(new OrderLine(product, quantity));
        recalculateTotal();
    }
    
    public void confirm() {
        if (lines.isEmpty()) {
            throw new DomainException("Cannot confirm empty order");
        }
        this.status = OrderStatus.CONFIRMED;
        registerEvent(new OrderConfirmedEvent(this.id));
    }
}

// ═══════════════════════════════════════════════════════════════
// VALUE OBJECT: No identity, immutable, equality by value
// ═══════════════════════════════════════════════════════════════
public record Money(BigDecimal amount, Currency currency) {
    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new DomainException("Cannot add different currencies");
        }
        return new Money(this.amount.add(other.amount), this.currency);
    }
}

public record Address(String street, String city, String zip, String country) {
    // No ID! Two addresses with same values ARE the same address.
}

// ═══════════════════════════════════════════════════════════════
// AGGREGATE: Cluster of entities with one root, consistency boundary
// ═══════════════════════════════════════════════════════════════
// Order is the AGGREGATE ROOT
// OrderLine is an entity INSIDE the aggregate
// Access OrderLine ONLY through Order (never directly)
// One transaction = one aggregate

// ═══════════════════════════════════════════════════════════════
// DOMAIN SERVICE: Logic that doesn't belong to any single entity
// ═══════════════════════════════════════════════════════════════
public class PricingService {
    public Money calculatePrice(Order order, Customer customer) {
        Money basePrice = order.calculateSubtotal();
        Discount discount = discountPolicy.apply(customer, basePrice);
        Tax tax = taxCalculator.calculate(basePrice, customer.getAddress());
        return basePrice.subtract(discount.amount()).add(tax.amount());
    }
}

// ═══════════════════════════════════════════════════════════════
// REPOSITORY: Persistence abstraction for aggregates
// ═══════════════════════════════════════════════════════════════
public interface OrderRepository {
    Order findById(OrderId id);
    void save(Order order);
    // ONLY aggregate roots get repositories!
    // No OrderLineRepository!
}

// ═══════════════════════════════════════════════════════════════
// DOMAIN EVENT: Something that happened in the domain
// ═══════════════════════════════════════════════════════════════
public record OrderConfirmedEvent(OrderId orderId, Instant occurredAt) 
    implements DomainEvent {}
```

---

## 🌍 Ubiquitous Language

```
THE MOST IMPORTANT DDD CONCEPT:

  Developers and business experts use THE SAME vocabulary.
  This vocabulary is the "Ubiquitous Language."

  ❌ Developer says: "The UserEntity has a status flag set to 2"
  ✅ Both say:       "The Customer's account is suspended"

  ❌ Developer says: "We INSERT a row into order_items"
  ✅ Both say:       "The Customer adds a Product to their Cart"

HOW TO BUILD IT:
  1. Listen to domain experts — use THEIR words
  2. Name classes, methods, variables using those words
  3. If the code doesn't match the language, refactor the CODE
  4. If the language is ambiguous, clarify with experts
  5. Document the glossary — share it across the team
```

---

## 📊 When to Use DDD

| Scenario | Use DDD? | Why |
|---|---|---|
| Complex domain logic | ✅ Yes | DDD shines with complex rules |
| Simple CRUD app | ❌ No | Over-engineering, just use Spring Data |
| Many business rules | ✅ Yes | Rules belong in domain model |
| Data-centric (reports) | ❌ No | Use simple queries, no domain model |
| Multiple bounded contexts | ✅ Yes | DDD provides integration patterns |
| Startup MVP | ❌ No | Too much upfront investment |

---

## ⚠️ Common Pitfalls

1. **Anemic domain model** — Entities with only getters/setters and all logic in services. This is NOT DDD — it's a procedural program wearing OOP clothes. Put business rules INSIDE entities.

2. **One model to rule them all** — Using a single `Product` class across ordering, inventory, and shipping. Each bounded context should have its own model tailored to its needs.

3. **Aggregate too large** — If you put everything in one aggregate, you get lock contention and performance issues. Keep aggregates small; use domain events for cross-aggregate communication.

---

## 📝 Interview Q&A

**Q: How do bounded contexts map to microservices?**
> A: A bounded context is a logical boundary; a microservice is a deployment boundary. Ideally, one microservice = one bounded context. They share no database, communicate via events or APIs, and each has its own model. However, you CAN have multiple bounded contexts in one service (monolith) or split one context across services (rarely advisable). Start with bounded contexts in a monolith, then extract to microservices when needed.

---

## 🔗 What to Read Next

1. **[Microservices/EventSourcing.md](../Microservices/EventSourcing.md)** — Event sourcing pattern
2. **[Architectures/Clean.md](../Architectures/Clean.md)** — Clean architecture
3. **[Principles/TheoriesAndLaws.md](./TheoriesAndLaws.md)** — Engineering laws and theories

---

*[← Clean Code Principles](./Clean_Code_Principles.md) | [Back to Index](../INDEX.md) | [Next: Theories and Laws →](./TheoriesAndLaws.md)*
