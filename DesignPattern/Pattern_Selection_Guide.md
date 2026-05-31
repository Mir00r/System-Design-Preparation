# 🗺️ Design Pattern Selection Guide: Choosing the Right Pattern 🚀

**⏱️ 25 minutes** | **🎯 🟡 Intermediate** | **🔗 Prerequisites**: [Design Patterns Overview](./README.md)

> **"A pattern is not a finished design that can be transformed directly into code. It is a description or template for how to solve a problem that can be used in many different situations."** — Gang of Four

---

## 📋 Table of Contents

1. [Pattern Categories Overview](#1-pattern-categories-overview)
2. [Decision Tree: Which Pattern to Use?](#2-decision-tree-which-pattern-to-use)
3. [Creational Patterns: Object Creation Problems](#3-creational-patterns-object-creation-problems)
4. [Structural Patterns: Object Composition Problems](#4-structural-patterns-object-composition-problems)
5. [Behavioral Patterns: Object Communication Problems](#5-behavioral-patterns-object-communication-problems)
6. [When Modern Java Replaces Patterns](#6-when-modern-java-replaces-patterns)
7. [FAANG Frequency Table](#7-faang-frequency-table)
8. [Quick Code Examples](#8-quick-code-examples)
9. [Anti-Pattern Warnings](#9-anti-pattern-warnings)
10. [Interview Q&A](#10-interview-qa)

---

## 1. Pattern Categories Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    DESIGN PATTERNS (23 GoF)                     │
├─────────────────┬───────────────────┬───────────────────────────┤
│   CREATIONAL    │    STRUCTURAL     │       BEHAVIORAL          │
│  (HOW created) │  (HOW composed)   │   (HOW communicate)       │
├─────────────────┼───────────────────┼───────────────────────────┤
│ • Singleton     │ • Adapter         │ • Chain of Responsibility │
│ • Factory       │ • Bridge          │ • Command                 │
│ • Abstract Fac  │ • Composite       │ • Iterator                │
│ • Builder       │ • Decorator       │ • Mediator                │
│ • Prototype     │ • Facade          │ • Memento                 │
│                 │ • Flyweight       │ • Observer                │
│                 │ • Proxy           │ • State                   │
│                 │                   │ • Strategy                │
│                 │                   │ • Template Method         │
│                 │                   │ • Visitor                 │
│                 │                   │ • Interpreter             │
└─────────────────┴───────────────────┴───────────────────────────┘
```

---

## 2. Decision Tree: Which Pattern to Use?

```
What is your problem?
│
├── OBJECT CREATION
│   ├── Need exactly ONE instance?           → Singleton
│   ├── Need to create family of objects?    → Abstract Factory
│   ├── Subclass decides type to create?     → Factory Method
│   ├── Complex multi-step construction?     → Builder
│   └── Clone an existing object?            → Prototype
│
├── OBJECT STRUCTURE / COMPOSITION
│   ├── Incompatible interfaces?             → Adapter
│   ├── Add behavior without subclassing?    → Decorator
│   ├── Simplify complex subsystem?          → Facade
│   ├── Tree-like hierarchy (part-whole)?    → Composite
│   ├── Separate abstraction from impl?      → Bridge
│   ├── Control access to object?            → Proxy
│   └── Sharing many fine-grained objects?   → Flyweight
│
└── OBJECT BEHAVIOR / COMMUNICATION
    ├── Algorithms interchangeable?          → Strategy
    ├── Object changes behavior by state?    → State
    ├── Notify multiple objects on change?   → Observer
    ├── Encapsulate request as object?       → Command
    ├── Reduce dependencies between objects? → Mediator
    ├── Template with overridable steps?     → Template Method
    ├── Process chain of handlers?           → Chain of Responsibility
    ├── Traverse collection uniformly?       → Iterator
    ├── Save/restore object state?           → Memento
    ├── Add ops to object hierarchy?         → Visitor
    └── Interpret a grammar/language?        → Interpreter
```

---

## 3. Creational Patterns: Object Creation Problems

| Pattern | Problem | Solution | Java Example |
|---------|---------|----------|-------------|
| **Singleton** | Need single shared instance | Private constructor + static accessor | `ApplicationContext`, `Logger` |
| **Factory Method** | Subclass controls object type | Abstract factory method in parent | `DocumentParser.create(type)` |
| **Abstract Factory** | Family of related objects | Factory of factories | `UIFactory → WindowsUI / MacUI` |
| **Builder** | Complex object with many params | Fluent builder with validate() | `HttpRequest.newBuilder()` |
| **Prototype** | Copy complex object | `clone()` / copy constructor | Cache of pre-built configs |

**Key Question**: *"Who creates it, and how complex is the creation?"*

---

## 4. Structural Patterns: Object Composition Problems

| Pattern | Problem | Solution | Java Example |
|---------|---------|----------|-------------|
| **Adapter** | Incompatible interfaces | Wrapper translating calls | Legacy XML → JSON adapter |
| **Decorator** | Add features without inheritance | Wrapping objects at runtime | `BufferedInputStream(FileInputStream)` |
| **Facade** | Complex subsystem too hard to use | Single simplified entry point | `OrderFacade` wrapping inventory, payment, shipping |
| **Composite** | Tree structure of objects | Common interface for leaf/branch | File system: File + Directory |
| **Bridge** | Abstraction + implementation vary | Separate hierarchies linked by composition | `Shape(color)` → `Circle(RedColor)` |
| **Proxy** | Control/enhance object access | Same interface, extra behavior | Lazy-loading, security proxy, cache proxy |
| **Flyweight** | Many identical small objects | Share common state | Characters in text editor, particles in game |

**Key Question**: *"How are my objects related and composed?"*

---

## 5. Behavioral Patterns: Object Communication Problems

| Pattern | Problem | Solution | Java Example |
|---------|---------|----------|-------------|
| **Strategy** | Multiple algorithms, interchangeable | Interface + concrete implementations | Sort strategies, payment methods |
| **Observer** | One-to-many state change notification | Subscribe/notify mechanism | Event bus, Spring `@EventListener` |
| **Command** | Encapsulate request, support undo | Command object with execute()/undo() | Menu actions, transaction rollback |
| **State** | Behavior depends on state | State objects instead of if-chains | Order lifecycle, connection state |
| **Template Method** | Algorithm skeleton, vary some steps | Abstract class with hook methods | `JdbcTemplate`, report generators |
| **Chain of Responsibility** | Multiple handlers, dynamic routing | Linked chain of handlers | Filter chains, logging levels |
| **Mediator** | Objects know too much about each other | Central coordinator | Chat room, air traffic control |
| **Iterator** | Traverse collection without knowing type | Unified `next()`/`hasNext()` | `Iterable<T>`, stream pipelines |
| **Memento** | Save and restore object state | Snapshot object externally | Undo in text editors, game saves |
| **Visitor** | Add operations to object hierarchy | Visitor + accept() double dispatch | AST operations, export formats |
| **Interpreter** | Parse and evaluate language/grammar | Grammar rules as classes | SQL parser, regex engine |

**Key Question**: *"How do objects collaborate and who's responsible for what?"*

---

## 6. When Modern Java Replaces Patterns

Modern Java 17+ features can replace or simplify several classic patterns:

```java
// ❌ CLASSIC Command Pattern (verbose)
interface Command { void execute(); }
class SaveCommand implements Command {
    public void execute() { /* save logic */ }
}

// ✅ MODERN Java — Lambda replaces Command
Runnable save = () -> { /* save logic */ };
save.run();
```

| Classic Pattern | Modern Java Alternative | Benefit |
|----------------|------------------------|---------|
| **Command** | Lambdas / `Runnable` / `Supplier` | No boilerplate classes |
| **Strategy** | Functional interfaces + lambdas | `Comparator.comparing(...)` |
| **Iterator** | Streams + `forEach` | Declarative, composable |
| **Singleton** | Enum singleton | Thread-safe by spec |
| **Template Method** | Default interface methods | More flexible than inheritance |
| **Value Object** | `record` (Java 16+) | Immutable, auto-equals/hashCode |
| **Factory** | Static factory methods on interfaces | `List.of()`, `Map.of()` |
| **Builder** | `record` with builders / Lombok | Less boilerplate |
| **Null Object** | `Optional<T>` | Explicit null handling |
| **Observer** | Reactive Streams (`Flow.Publisher`) | Backpressure support |

```java
// ❌ CLASSIC Strategy (pre-Java 8)
interface SortStrategy { void sort(List<?> list); }
class BubbleSort implements SortStrategy { ... }
class QuickSort implements SortStrategy { ... }

// ✅ MODERN Strategy — Functional Interface
List<String> names = Arrays.asList("Charlie", "Alice", "Bob");
names.sort(Comparator.naturalOrder());          // built-in strategy
names.sort((a, b) -> a.length() - b.length()); // lambda strategy

// ❌ CLASSIC Singleton
public class Config {
    private static Config INSTANCE;
    private Config() {}
    public static synchronized Config getInstance() {
        if (INSTANCE == null) INSTANCE = new Config();
        return INSTANCE;
    }
}

// ✅ MODERN Singleton — Enum (thread-safe, serialization-safe)
public enum Config {
    INSTANCE;
    public String getDb() { return System.getenv("DB_URL"); }
}
```

---

## 7. FAANG Frequency Table

| Pattern | Frequency | Where Used | Interview Focus |
|---------|-----------|------------|----------------|
| **Singleton** | ⭐⭐⭐⭐⭐ | Configuration, connection pools | Thread safety issues |
| **Observer** | ⭐⭐⭐⭐⭐ | Event systems, notifications | Loose coupling, memory leaks |
| **Strategy** | ⭐⭐⭐⭐⭐ | Payment, routing, algorithms | Open/Closed Principle |
| **Factory** | ⭐⭐⭐⭐⭐ | Object creation, plugins | Extensibility |
| **Builder** | ⭐⭐⭐⭐⭐ | Request/config building | Fluent API design |
| **Decorator** | ⭐⭐⭐⭐ | I/O streams, middleware | Composition over inheritance |
| **Facade** | ⭐⭐⭐⭐ | Service layers, APIs | Simplifying complexity |
| **Proxy** | ⭐⭐⭐⭐ | AOP, lazy loading, security | Spring AOP internals |
| **Command** | ⭐⭐⭐ | Undo/redo, request queuing | Encapsulating requests |
| **Template Method** | ⭐⭐⭐ | Frameworks, JdbcTemplate | Inversion of Control |
| **State** | ⭐⭐⭐ | Workflow engines, UI | State machine design |
| **Composite** | ⭐⭐⭐ | File systems, UI trees | Recursive structures |
| **Chain of Resp.** | ⭐⭐⭐ | Filter chains, logging | Decoupled handlers |
| **Adapter** | ⭐⭐⭐ | Third-party integration | Interface compatibility |
| **Iterator** | ⭐⭐ | Collection traversal | Built into Java |
| **Mediator** | ⭐⭐ | Chat systems, workflow | Reducing coupling |
| **Flyweight** | ⭐⭐ | String pool, caching | Memory optimization |
| **Visitor** | ⭐⭐ | Compiler/AST, reports | Double dispatch |
| **Memento** | ⭐⭐ | Undo/redo, snapshots | State persistence |

---

## 8. Quick Code Examples

### Builder Pattern (most common in Java)

```java
// Effective Java - Builder for complex objects
public final class HttpRequest {
    private final String url;
    private final String method;
    private final Map<String, String> headers;
    private final Duration timeout;

    private HttpRequest(Builder builder) {
        this.url = Objects.requireNonNull(builder.url, "url required");
        this.method = builder.method;
        this.headers = Map.copyOf(builder.headers);
        this.timeout = builder.timeout;
    }

    public static Builder newBuilder(String url) {
        return new Builder(url);
    }

    public static final class Builder {
        private final String url;
        private String method = "GET";
        private Map<String, String> headers = new HashMap<>();
        private Duration timeout = Duration.ofSeconds(30);

        private Builder(String url) { this.url = url; }

        public Builder method(String method) {
            this.method = method;
            return this;
        }

        public Builder header(String name, String value) {
            this.headers.put(name, value);
            return this;
        }

        public Builder timeout(Duration timeout) {
            this.timeout = timeout;
            return this;
        }

        public HttpRequest build() {
            return new HttpRequest(this);
        }
    }
}

// Usage
HttpRequest request = HttpRequest.newBuilder("https://api.example.com/users")
    .method("POST")
    .header("Content-Type", "application/json")
    .timeout(Duration.ofSeconds(10))
    .build();
```

### Observer Pattern (Spring-style)

```java
// Event (Value Object with Java record)
public record OrderPlacedEvent(String orderId, BigDecimal total, Instant timestamp) {}

// Publisher
@Service
public class OrderService {
    private final ApplicationEventPublisher eventPublisher;

    public Order placeOrder(OrderRequest request) {
        Order order = processOrder(request);
        eventPublisher.publishEvent(new OrderPlacedEvent(
            order.getId(), order.getTotal(), Instant.now()
        ));
        return order;
    }
}

// Subscribers (multiple, decoupled)
@Component
public class EmailNotificationListener {
    @EventListener
    public void onOrderPlaced(OrderPlacedEvent event) {
        emailService.sendConfirmation(event.orderId());
    }
}

@Component
public class InventoryListener {
    @EventListener
    @Async
    public void onOrderPlaced(OrderPlacedEvent event) {
        inventoryService.reserve(event.orderId());
    }
}
```

### Strategy Pattern (with lambdas)

```java
// Strategy as functional interface
@FunctionalInterface
public interface PricingStrategy {
    BigDecimal calculate(BigDecimal basePrice, Customer customer);
}

@Service
public class PricingService {
    private final Map<CustomerTier, PricingStrategy> strategies = Map.of(
        CustomerTier.REGULAR,  (price, c) -> price,
        CustomerTier.SILVER,   (price, c) -> price.multiply(BigDecimal.valueOf(0.95)),
        CustomerTier.GOLD,     (price, c) -> price.multiply(BigDecimal.valueOf(0.90)),
        CustomerTier.PLATINUM, (price, c) -> price.multiply(BigDecimal.valueOf(0.80))
    );

    public BigDecimal getPrice(BigDecimal base, Customer customer) {
        return strategies.getOrDefault(customer.getTier(),
            strategies.get(CustomerTier.REGULAR))
            .calculate(base, customer);
    }
}
```

### Decorator Pattern (I/O style)

```java
// Component interface
public interface TextProcessor {
    String process(String text);
}

// Base implementation
public class PlainTextProcessor implements TextProcessor {
    public String process(String text) { return text; }
}

// Decorators (composable)
public class UpperCaseDecorator implements TextProcessor {
    private final TextProcessor wrapped;
    public UpperCaseDecorator(TextProcessor wrapped) { this.wrapped = wrapped; }
    public String process(String text) { return wrapped.process(text).toUpperCase(); }
}

public class TrimDecorator implements TextProcessor {
    private final TextProcessor wrapped;
    public TrimDecorator(TextProcessor wrapped) { this.wrapped = wrapped; }
    public String process(String text) { return wrapped.process(text).trim(); }
}

// Composing decorators at runtime
TextProcessor processor = new UpperCaseDecorator(
                              new TrimDecorator(
                                  new PlainTextProcessor()));
processor.process("  hello world  "); // → "HELLO WORLD"
```

---

## 9. Anti-Pattern Warnings

> **⚠️ Common mistakes in design pattern interviews:**

| Anti-Pattern | Problem | Correct Approach |
|-------------|---------|-----------------|
| **Singleton Overuse** | Tight coupling, hard to test, global mutable state | Use dependency injection instead |
| **Premature Abstraction** | Factory for objects that never vary | YAGNI — create directly if no variation needed |
| **Pattern Soup** | Using patterns everywhere to seem smart | Apply only when problem matches pattern intent |
| **Inheritance over Composition** | Deep hierarchies with Template Method everywhere | Prefer Strategy + composition |
| **Observer Memory Leaks** | Forgot to unsubscribe listeners | Always `removeListener()` or use weak references |
| **Thread-unsafe Singleton** | Classic double-checked locking bugs | Use enum singleton or initialization-on-demand holder |
| **Visitor Rigidity** | Adding new elements requires updating all visitors | Use when element hierarchy is stable, operations vary |

---

## 10. Interview Q&A

<details>
<summary>💡 Q: What's the difference between Factory Method and Abstract Factory?</summary>

**Factory Method**: One factory method, subclasses decide which class to instantiate. Single product hierarchy.

```java
abstract class Dialog {
    abstract Button createButton(); // Factory Method
    void render() { createButton().render(); }
}
class WindowsDialog extends Dialog {
    Button createButton() { return new WindowsButton(); }
}
```

**Abstract Factory**: Multiple factory methods for creating a *family* of related objects:

```java
interface UIFactory {
    Button createButton();
    Checkbox createCheckbox();
    TextField createTextField();
}
class WindowsUIFactory implements UIFactory { ... }
class MacUIFactory implements UIFactory { ... }
```

</details>

<details>
<summary>💡 Q: When would you use Proxy vs Decorator?</summary>

**Proxy** — controls *access* to the real object:
- Same interface, but adds access control, lazy loading, caching, remote access
- Client doesn't know it's a proxy

**Decorator** — *enhances* an object with additional behavior:
- Wraps and extends functionality
- Can be stacked: `new LoggingDecorator(new CachingDecorator(new Service()))`

Key difference: Proxy manages lifecycle of the subject; Decorator receives subject from caller.

</details>

<details>
<summary>💡 Q: How does Spring use design patterns internally?</summary>

| Pattern | Spring Usage |
|---------|-------------|
| Singleton | Default `@Bean` scope |
| Factory | `BeanFactory`, `ApplicationContext` |
| Proxy | AOP (`@Transactional`, `@Cacheable`) |
| Observer | `ApplicationEventPublisher` |
| Template Method | `JdbcTemplate`, `RestTemplate` |
| Decorator | `HttpServletRequestWrapper` |
| Strategy | `ResourceLoader`, `TransactionManager` |
| Composite | `CompositeValidator` |
| Chain of Resp. | `HandlerInterceptor` chain |

</details>

<details>
<summary>💡 Q: Explain the Open/Closed Principle with a design pattern example</summary>

**Open for extension, closed for modification.**

Strategy pattern is the canonical example:

```java
// CLOSED for modification (never touch this)
public class ReportGenerator {
    private final ExportStrategy strategy;

    public ReportGenerator(ExportStrategy strategy) {
        this.strategy = strategy;
    }

    public void generate(Report report) {
        strategy.export(report);
    }
}

// OPEN for extension (add new formats without changing ReportGenerator)
public class PdfExportStrategy implements ExportStrategy { ... }
public class ExcelExportStrategy implements ExportStrategy { ... }
public class CsvExportStrategy implements ExportStrategy { ... }
```

</details>

<details>
<summary>💡 Q: When should you NOT use design patterns?</summary>

- **Simple one-off code** — don't factory-ize a single object type
- **Small scripts/utilities** — overhead outweighs benefit
- **When Java features suffice** — use lambda instead of Command pattern
- **When team doesn't know the pattern** — reduces readability
- **When there's only one concrete case** — YAGNI applies

> Rule of thumb: Apply patterns when you see **variation** (multiple strategies, multiple products, multiple observers). If there's only one, you're over-engineering.

</details>

---

*Previous: [← SUMMARY](./SUMMARY.md) | Next: [Creational Patterns →](./Creational/)*
