# 🎀 Decorator Pattern: Add Powers Without Changing Identity! 🎯

> **"Don't modify the coffee — decorate it! Add milk, sugar, whip, mocha — each a layer, each removable."**

---

## 🎬 The Story

### ☕ The Starbucks Problem

```
WITHOUT DECORATOR (Subclass Explosion! 💥):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Coffee
├── CoffeeWithMilk
├── CoffeeWithSugar
├── CoffeeWithMilkAndSugar
├── CoffeeWithMilkAndSugarAndWhip
├── CoffeeWithMocha
├── CoffeeWithMochaAndMilk
├── ... (2^n combinations = 💀)

WITH DECORATOR (Wrap and Stack! ✅):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
new Whip(new Mocha(new Milk(new SimpleCoffee())))
// Stack as many decorators as you want!
// Remove any layer without affecting others!
// Add NEW decorators without changing existing code!
```

---

## 💡 The Solution

```
DECORATOR wraps an object to add new behavior dynamically.
It has the SAME interface as the wrapped object.
Client doesn't know the difference!

Coffee   ($2) → [+Milk $0.5] → [+Sugar $0.3] → [+Whip $0.7]
                                                  Total: $3.50
Each decorator:
1. Holds reference to the wrapped component
2. Delegates calls to the wrapped component  
3. Adds its OWN behavior before/after
```

---

## 🏗️ The Structure

```
     ┌──────────────────────────┐
     │  Component (interface)    │◄──────────────────────┐
     │  + operation()            │                       │
     └──────────┬───────────────┘                       │
                │ implements                              │
     ┌──────────┼──────────────────┐                    │
     │                             │                    │has-a
┌────▼──────┐          ┌──────────▼──────────────┐     │
│  Concrete  │          │    Decorator (abstract)  │─────┘
│  Component │          │ ──────────────────────── │
│ (the base) │          │ - wrapped: Component     │
└────────────┘          │ + operation()             │
                        │   → wrapped.operation()   │
                        └──────────┬───────────────┘
                                   │ extends
                        ┌──────────┼──────────┐
                        │                     │
               ┌────────▼──────┐    ┌─────────▼─────┐
               │ DecoratorA     │    │ DecoratorB     │
               │ + operation()  │    │ + operation()  │
               │  → extra logic │    │  → extra logic │
               │  → super.op()  │    │  → super.op()  │
               └───────────────┘    └───────────────┘
```

---

## 💻 Java Implementation

```java
// ☕ Component interface
public interface Coffee {
    double getCost();
    String getDescription();
}

// Base coffee (concrete component)
public class SimpleCoffee implements Coffee {
    @Override public double getCost() { return 2.00; }
    @Override public String getDescription() { return "Simple coffee"; }
}

// 🎀 Base Decorator
public abstract class CoffeeDecorator implements Coffee {
    protected final Coffee wrapped;
    
    public CoffeeDecorator(Coffee coffee) { this.wrapped = coffee; }
    
    @Override public double getCost() { return wrapped.getCost(); }
    @Override public String getDescription() { return wrapped.getDescription(); }
}

// Concrete Decorators
public class MilkDecorator extends CoffeeDecorator {
    public MilkDecorator(Coffee coffee) { super(coffee); }
    
    @Override public double getCost() { return super.getCost() + 0.50; }
    @Override public String getDescription() { return super.getDescription() + " + Milk 🥛"; }
}

public class SugarDecorator extends CoffeeDecorator {
    public SugarDecorator(Coffee coffee) { super(coffee); }
    
    @Override public double getCost() { return super.getCost() + 0.30; }
    @Override public String getDescription() { return super.getDescription() + " + Sugar 🍬"; }
}

public class WhipDecorator extends CoffeeDecorator {
    public WhipDecorator(Coffee coffee) { super(coffee); }
    
    @Override public double getCost() { return super.getCost() + 0.70; }
    @Override public String getDescription() { return super.getDescription() + " + Whip 🍦"; }
}

// 🎮 Usage — stack decorators!
Coffee order = new WhipDecorator(
                   new SugarDecorator(
                       new MilkDecorator(
                           new SimpleCoffee())));

System.out.println(order.getDescription()); // Simple coffee + Milk 🥛 + Sugar 🍬 + Whip 🍦
System.out.println("$" + order.getCost());  // $3.50
```

---

## 🌍 Real-World Examples — Java I/O! 

```java
// ☕ Java I/O is THE classic Decorator example!
InputStream fileStream = new FileInputStream("data.txt");        // Component
InputStream buffered   = new BufferedInputStream(fileStream);     // + Buffering
DataInputStream data   = new DataInputStream(buffered);           // + Typed reads
// Each wraps the previous, adding new capability!

// 🌱 Spring Security — Filter chain = decorators!
// 🌐 HTTP middleware — each adds logging, auth, compression
// 📝 Logging wrappers — add timing, context to any logger
```

---

## ⚡ When to Use

```
✅ Add behavior to objects dynamically (at runtime)
✅ Combination explosion from inheritance
✅ Want to add/remove features independently
✅ Single Responsibility: each decorator = one feature

❌ Order of decorators matters and is hard to control
❌ Too many small decorator classes = confusion
❌ Hard to unwrap/identify specific decorator in the chain
```

---

## 🎯 Key Interview Point

**Decorator vs Inheritance:**
- Inheritance: static, compile-time, 2^n class explosion
- Decorator: dynamic, runtime, linear growth, composable

**Decorator vs Proxy:**
- Decorator: ADDS behavior (enhance)
- Proxy: CONTROLS access (restrict/lazy-load)

---

## 🏆 Achievement Unlocked!

```
Progress: 9/23 patterns █████████░
```

---

*← [Composite](./Composite.md) | [Facade →](./Facade.md)*
