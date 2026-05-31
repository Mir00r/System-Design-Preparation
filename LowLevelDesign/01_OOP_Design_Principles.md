# 🏛️ OOP Design Principles: SOLID, DRY, YAGNI, and KISS 🚀

**⏱️ 30 minutes** | **🎯 🟡 Intermediate** | **🔗 Prerequisites**: [LowLevelDesign README](./README.md)

> **"Good design is obvious. Great design is transparent."** — Joe Sparano

---

## 📋 Table of Contents

1. [Why Principles Matter](#1-why-principles-matter)
2. [SOLID Principles](#2-solid-principles)
3. [DRY — Don't Repeat Yourself](#3-dry--dont-repeat-yourself)
4. [YAGNI — You Aren't Gonna Need It](#4-yagni--you-arent-gonna-need-it)
5. [KISS — Keep It Simple Stupid](#5-kiss--keep-it-simple-stupid)
6. [LoD — Law of Demeter](#6-lod--law-of-demeter)
7. [Principle Violations Cheat Sheet](#7-principle-violations-cheat-sheet)
8. [Java 17+ Code Examples](#8-java-17-code-examples)
9. [Interview Q&A](#9-interview-qa)

---

## 1. Why Principles Matter

```
Without Principles          With Principles
────────────────────        ──────────────────────────
God classes (5000 lines)    Small, focused classes
Hidden dependencies         Explicit, injected deps
Copy-paste everywhere       Single source of truth
"It works, don't touch"     Confident to refactor
Hard to unit test           Easy to mock and test
```

These principles are **not rules** — they are heuristics. Know when to apply them and when pragmatism wins.

---

## 2. SOLID Principles

### S — Single Responsibility Principle (SRP)

> *"A class should have only one reason to change."* — Robert C. Martin

```java
// ❌ VIOLATION: OrderProcessor does too much
public class OrderProcessor {
    public void processOrder(Order order) {
        // Validates order
        if (order.getItems().isEmpty()) throw new IllegalArgumentException("Empty order");
        
        // Saves to database
        jdbcTemplate.update("INSERT INTO orders ...", order);
        
        // Sends email
        emailService.send(order.getCustomerEmail(), "Order confirmed");
        
        // Generates PDF invoice
        pdfGenerator.generate(order);
    }
}

// ✅ CORRECT: Each class has one reason to change
public class OrderValidator {
    public void validate(Order order) {
        if (order.getItems().isEmpty()) 
            throw new IllegalArgumentException("Order cannot be empty");
    }
}

public class OrderRepository {
    public void save(Order order) {
        jdbcTemplate.update("INSERT INTO orders ...", order);
    }
}

public class OrderNotificationService {
    public void notifyCustomer(Order order) {
        emailService.send(order.getCustomerEmail(), "Order confirmed");
    }
}

public class OrderProcessor {
    private final OrderValidator validator;
    private final OrderRepository repository;
    private final OrderNotificationService notifier;

    public void processOrder(Order order) {
        validator.validate(order);
        repository.save(order);
        notifier.notifyCustomer(order);
    }
}
```

---

### O — Open/Closed Principle (OCP)

> *"Software entities should be open for extension, closed for modification."*

```java
// ❌ VIOLATION: Add new payment type = modify existing class
public class PaymentProcessor {
    public void pay(String type, double amount) {
        if (type.equals("CREDIT_CARD")) {
            // credit card logic
        } else if (type.equals("PAYPAL")) {
            // paypal logic
        } else if (type.equals("CRYPTO")) {  // new: must modify class!
            // crypto logic
        }
    }
}

// ✅ CORRECT: Add new type = add new class (never touch existing code)
@FunctionalInterface
public interface PaymentStrategy {
    void pay(double amount);
}

@Component("CREDIT_CARD")
public class CreditCardPayment implements PaymentStrategy {
    public void pay(double amount) { /* credit card logic */ }
}

@Component("PAYPAL")
public class PaypalPayment implements PaymentStrategy {
    public void pay(double amount) { /* paypal logic */ }
}

// New payment type: just add a new class, zero existing code changes
@Component("CRYPTO")
public class CryptoPayment implements PaymentStrategy {
    public void pay(double amount) { /* crypto logic */ }
}

@Service
public class PaymentProcessor {
    private final Map<String, PaymentStrategy> strategies;

    public PaymentProcessor(Map<String, PaymentStrategy> strategies) {
        this.strategies = strategies;
    }

    public void pay(String type, double amount) {
        strategies.getOrDefault(type, 
            amt -> { throw new IllegalArgumentException("Unknown payment: " + type); })
            .pay(amount);
    }
}
```

---

### L — Liskov Substitution Principle (LSP)

> *"Objects of a supertype should be replaceable by objects of a subtype without breaking the program."*

```java
// ❌ VIOLATION: Rectangle/Square classic violation
public class Rectangle {
    protected int width, height;
    public void setWidth(int w) { this.width = w; }
    public void setHeight(int h) { this.height = h; }
    public int area() { return width * height; }
}

public class Square extends Rectangle {
    @Override
    public void setWidth(int w) { 
        this.width = w; 
        this.height = w; // Square breaks Rectangle's contract!
    }
}

// Client code breaks when Square is substituted for Rectangle:
Rectangle r = new Square();
r.setWidth(5);
r.setHeight(3);
System.out.println(r.area()); // Expected 15, got 9!

// ✅ CORRECT: Separate abstractions
public sealed interface Shape permits Rectangle, Square, Circle {
    int area();
}

public record Rectangle(int width, int height) implements Shape {
    public int area() { return width * height; }
}

public record Square(int side) implements Shape {
    public int area() { return side * side; }
}
```

---

### I — Interface Segregation Principle (ISP)

> *"Clients should not be forced to depend on interfaces they don't use."*

```java
// ❌ VIOLATION: Fat interface forces empty implementations
public interface Worker {
    void work();
    void eat();       // Robots don't eat!
    void sleep();     // Robots don't sleep!
}

public class Robot implements Worker {
    public void work() { /* robot works */ }
    public void eat() { throw new UnsupportedOperationException(); }  // forced!
    public void sleep() { throw new UnsupportedOperationException(); } // forced!
}

// ✅ CORRECT: Segregated interfaces
public interface Workable {
    void work();
}

public interface Eatable {
    void eat();
}

public interface Sleepable {
    void sleep();
}

public class Human implements Workable, Eatable, Sleepable {
    public void work() { /* works */ }
    public void eat()  { /* eats */ }
    public void sleep() { /* sleeps */ }
}

public class Robot implements Workable {
    public void work() { /* robot works, no eating/sleeping needed */ }
}
```

---

### D — Dependency Inversion Principle (DIP)

> *"Depend on abstractions, not concretions."*

```java
// ❌ VIOLATION: High-level module depends on low-level detail
public class OrderService {
    private MySQLOrderRepository repository = new MySQLOrderRepository(); // tight coupling!

    public void createOrder(Order order) {
        repository.save(order);
    }
}

// ✅ CORRECT: Depend on interface (abstraction)
public interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(String id);
}

@Repository
public class MySQLOrderRepository implements OrderRepository {
    public void save(Order order) { /* MySQL impl */ }
    public Optional<Order> findById(String id) { /* MySQL impl */ return Optional.empty(); }
}

@Repository
public class InMemoryOrderRepository implements OrderRepository {
    private final Map<String, Order> store = new ConcurrentHashMap<>();
    public void save(Order order) { store.put(order.getId(), order); }
    public Optional<Order> findById(String id) { return Optional.ofNullable(store.get(id)); }
}

@Service
public class OrderService {
    private final OrderRepository repository; // depends on abstraction

    public OrderService(OrderRepository repository) { // injected
        this.repository = repository;
    }

    public void createOrder(Order order) {
        repository.save(order); // works with MySQL, InMemory, or any future impl
    }
}
```

---

## 3. DRY — Don't Repeat Yourself

> *"Every piece of knowledge must have a single, unambiguous, authoritative representation within a system."*

```java
// ❌ VIOLATION: Same validation repeated in 3 services
public class UserService {
    public void createUser(String email) {
        if (email == null || !email.contains("@")) throw new InvalidEmailException(email);
        // ...
    }
}

public class AuthService {
    public void authenticate(String email, String pwd) {
        if (email == null || !email.contains("@")) throw new InvalidEmailException(email);
        // ...
    }
}

public class InviteService {
    public void sendInvite(String email) {
        if (email == null || !email.contains("@")) throw new InvalidEmailException(email);
        // ...
    }
}

// ✅ CORRECT: Single authoritative source
public record Email(String value) {
    public Email {
        Objects.requireNonNull(value, "Email cannot be null");
        if (!value.matches("^[\\w.-]+@[\\w.-]+\\.[a-zA-Z]{2,}$"))
            throw new InvalidEmailException(value);
    }
}

// Now all services use Email type — validation is in one place
public class UserService {
    public void createUser(Email email) { /* already validated */ }
}
```

**DRY applies to**: logic, data, documentation, build steps — not just code.

---

## 4. YAGNI — You Aren't Gonna Need It

> *"Always implement things when you actually need them, never when you just foresee that you need them."* — Ron Jeffries

```java
// ❌ VIOLATION: Building a "flexible" framework nobody asked for
public abstract class AbstractRepositoryFactory<T, ID, R extends Repository<T, ID>> {
    public abstract R create();
    public abstract R createWithCaching();
    public abstract R createWithAudit();
    // 50 more abstract methods "just in case"
}

// ✅ CORRECT: Build only what is needed today
@Repository
public class UserRepository {
    public void save(User user) { /* straightforward impl */ }
    public Optional<User> findById(UUID id) { return Optional.empty(); }
}
// Add caching WHEN the profiler shows it's needed, not before.
```

**YAGNI Violations to avoid:**
- Building plugin systems for one use case
- Adding parameters "for future flexibility"
- Creating abstract base classes for a single concrete class
- Over-engineering configuration for hardcoded values

---

## 5. KISS — Keep It Simple Stupid

> *"Complexity is the enemy of execution."* — Tony Robbins

```java
// ❌ VIOLATION: Over-engineered date formatting
public class DateFormatterStrategy {
    private final List<DateFormatterChain> chains;
    
    public String format(LocalDate date, FormatterContext ctx) {
        return chains.stream()
            .filter(c -> c.supports(ctx.getLocale()))
            .findFirst()
            .orElse(chains.get(0))
            .format(date, ctx);
    }
}

// ✅ CORRECT: Simple is better
public String formatDate(LocalDate date, Locale locale) {
    return DateTimeFormatter
        .ofLocalizedDate(FormatStyle.SHORT)
        .withLocale(locale)
        .format(date);
}
```

---

## 6. LoD — Law of Demeter

> *"Talk only to your immediate friends, not to strangers."* (Principle of Least Knowledge)

```java
// ❌ VIOLATION: Method chain — depends on internal structure of many classes
public class OrderService {
    public void process(Order order) {
        String city = order.getCustomer().getAddress().getCity().getName();
        // Breaks if any intermediate object changes
    }
}

// ✅ CORRECT: Tell, don't ask
public class Order {
    public String getCustomerCity() {  // Encapsulate the navigation
        return customer.getCity();
    }
}

public class Customer {
    public String getCity() {
        return address.getCity();
    }
}

public class OrderService {
    public void process(Order order) {
        String city = order.getCustomerCity(); // only talks to Order
    }
}
```

---

## 7. Principle Violations Cheat Sheet

| Code Smell | Likely Violated Principle | Fix |
|-----------|--------------------------|-----|
| Class > 300 lines | SRP | Split into focused classes |
| Method with `if instanceof` | OCP, LSP | Polymorphism / pattern |
| `UnsupportedOperationException` | LSP, ISP | Redesign interface hierarchy |
| Fat interface (10+ methods) | ISP | Split into role interfaces |
| `new ConcreteClass()` inside business logic | DIP | Inject via constructor |
| Copy-pasted validation/logic | DRY | Extract to shared class/method |
| "For future use" parameters | YAGNI | Remove them |
| Deep method chains `a.b().c().d()` | LoD | Add delegation methods |
| 500-line switch statement | OCP | Strategy pattern |

---

## 8. Java 17+ Code Examples

### Using Records for Value Objects (DRY + DIP)

```java
// Value objects with built-in equals/hashCode/toString
public record CustomerId(UUID value) {
    public CustomerId {
        Objects.requireNonNull(value, "CustomerId cannot be null");
    }
    
    public static CustomerId generate() {
        return new CustomerId(UUID.randomUUID());
    }
    
    public static CustomerId from(String s) {
        return new CustomerId(UUID.fromString(s));
    }
}

public record Money(BigDecimal amount, Currency currency) {
    public Money {
        if (amount.compareTo(BigDecimal.ZERO) < 0) 
            throw new IllegalArgumentException("Money cannot be negative");
        Objects.requireNonNull(currency);
    }
    
    public Money add(Money other) {
        if (!this.currency.equals(other.currency))
            throw new IllegalArgumentException("Cannot add different currencies");
        return new Money(this.amount.add(other.amount), this.currency);
    }
}
```

### Using Sealed Classes for Type Hierarchies (LSP-safe)

```java
// Sealed hierarchy ensures no unexpected subtypes break LSP
public sealed interface PaymentResult 
    permits PaymentResult.Success, PaymentResult.Failure, PaymentResult.Pending {
    
    record Success(String transactionId, Money amount) implements PaymentResult {}
    record Failure(String errorCode, String message) implements PaymentResult {}
    record Pending(String trackingId, Duration expectedIn) implements PaymentResult {}
}

// Pattern matching makes it exhaustive (compiler checks all cases)
public String describe(PaymentResult result) {
    return switch (result) {
        case PaymentResult.Success s -> "Paid: " + s.transactionId();
        case PaymentResult.Failure f -> "Failed: " + f.message();
        case PaymentResult.Pending p -> "Pending: " + p.trackingId();
    };
}
```

---

## 9. Interview Q&A

<details>
<summary>💡 Q: Which SOLID principle is most commonly violated in enterprise code?</summary>

**SRP** — Most commonly violated. God classes that handle persistence, business logic, and presentation grow naturally over time ("it's easier to add here"). Fix with extracting services, repositories, and mappers.

**DIP** is a close second — `new ServiceImpl()` inside business methods creates hidden dependencies that make testing and evolution difficult.

</details>

<details>
<summary>💡 Q: How do you explain DIP vs DI (Dependency Injection)?</summary>

- **DIP** (Dependency Inversion *Principle*) — Design principle: depend on abstractions, not concretions
- **DI** (Dependency Injection) — Design *pattern/technique* for achieving DIP: inject dependencies from outside

DIP is the **why**. DI (and IoC containers like Spring) is the **how**.

```java
// DIP: OrderService depends on OrderRepository interface (abstraction)
// DI: Spring injects MySQLOrderRepository at runtime
@Service
public class OrderService {
    private final OrderRepository repo; // DIP — abstraction
    
    public OrderService(OrderRepository repo) { // DI — injected
        this.repo = repo;
    }
}
```

</details>

<details>
<summary>💡 Q: When is it OK to violate YAGNI?</summary>

YAGNI can be relaxed when:
1. **Security** — build auth/authz properly even if "not needed yet"
2. **Extensibility in known hot-spots** — e.g., payment types are known to vary
3. **High refactor cost** — database schema changes are expensive, plan ahead
4. **Team alignment** — agreed-upon abstractions (repositories, services) even for simple code

Rule of thumb: defer flexibility until the *second* time you need to change something. First time: implement directly. Second time: refactor to be extensible.

</details>

<details>
<summary>💡 Q: What's the difference between SRP and ISP?</summary>

- **SRP** — *classes* should have one reason to change (cohesion at class level)
- **ISP** — *interfaces* should not force clients to depend on methods they don't use (granularity at interface level)

They're related: an interface that violates ISP often results in classes that violate SRP when implementing it.

</details>

---

*Previous: [← README](./README.md) | Next: [Design A Parking Lot →](./02_Design_A_Parking_Lot.md)*
