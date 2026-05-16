# 🎯 Strategy Pattern: Swap Algorithms on the Fly! 🎯

> **"Don't hardcode the HOW. Let the client CHOOSE the algorithm at runtime."**

---

## 🎬 The Story

### 🗺️ Google Maps — Multiple Route Strategies

```
You open Google Maps for directions. It offers:
🚗 DRIVING strategy  → Fastest roads, considers traffic
🚶 WALKING strategy  → Pedestrian paths, no highways  
🚴 CYCLING strategy  → Bike lanes, flat routes
🚌 TRANSIT strategy  → Bus/train schedules

SAME destination, DIFFERENT algorithms!
The app doesn't have a giant if-else. Each routing 
algorithm is a separate STRATEGY that's plugged in.
```

---

## 🤔 The Problem

```java
// ❌ BAD: Hard-coded algorithms with if-else
public class Navigator {
    public Route findRoute(String from, String to, String mode) {
        if (mode.equals("driving")) {
            // 100 lines of driving algorithm 😰
        } else if (mode.equals("walking")) {
            // 80 lines of walking algorithm
        } else if (mode.equals("cycling")) {
            // 90 lines of cycling algorithm
        }
        // Adding scooter? Modify this GIANT class again! 💀
        // Testing walking separately? IMPOSSIBLE!
    }
}
```

---

## 💡 The Solution

```
STRATEGY PATTERN:
1. Define a family of algorithms (interface)
2. Encapsulate each one (separate classes)  
3. Make them interchangeable (same interface)
4. Client picks which strategy to use at RUNTIME

Result: Adding new algorithms = new class. ZERO changes to existing code!
```

---

## 🏗️ The Structure

```
┌────────────────┐       ┌──────────────────────┐
│    Context      │       │  Strategy (Interface) │
│ ──────────────  │──────▶│ ────────────────────  │
│ - strategy      │       │ + execute(data)       │
│ + setStrategy() │       └──────────┬───────────┘
│ + doWork()      │                  │ implements
│  → strategy.    │         ┌────────┼────────┐
│    execute()    │         │        │        │
└────────────────┘    StrategyA  StrategyB  StrategyC
```

---

## 💻 Java Implementation

```java
// 🎯 Strategy Interface
@FunctionalInterface  // Can use lambdas! 🎉
public interface PaymentStrategy {
    boolean pay(double amount);
}

// 💳 Concrete Strategies
public class CreditCardPayment implements PaymentStrategy {
    private final String cardNumber;
    
    public CreditCardPayment(String cardNumber) { this.cardNumber = cardNumber; }
    
    @Override
    public boolean pay(double amount) {
        System.out.printf("💳 Paid $%.2f with Credit Card ending %s%n", 
                          amount, cardNumber.substring(cardNumber.length() - 4));
        return true;
    }
}

public class PayPalPayment implements PaymentStrategy {
    private final String email;
    
    public PayPalPayment(String email) { this.email = email; }
    
    @Override
    public boolean pay(double amount) {
        System.out.printf("🅿️ Paid $%.2f via PayPal (%s)%n", amount, email);
        return true;
    }
}

public class CryptoPayment implements PaymentStrategy {
    @Override
    public boolean pay(double amount) {
        System.out.printf("₿ Paid $%.2f in Bitcoin%n", amount);
        return true;
    }
}

// 🛒 Context — uses the strategy
public class ShoppingCart {
    private final List<Item> items = new ArrayList<>();
    private PaymentStrategy paymentStrategy; // 👈 Strategy reference
    
    public void addItem(Item item) { items.add(item); }
    
    public void setPaymentStrategy(PaymentStrategy strategy) {
        this.paymentStrategy = strategy; // Swap at runtime!
    }
    
    public boolean checkout() {
        double total = items.stream().mapToDouble(Item::getPrice).sum();
        return paymentStrategy.pay(total); // Delegate to strategy!
    }
}

// 🎮 Usage — client picks strategy!
ShoppingCart cart = new ShoppingCart();
cart.addItem(new Item("Laptop", 999.99));
cart.addItem(new Item("Mouse", 29.99));

// User selects payment method at checkout:
cart.setPaymentStrategy(new CreditCardPayment("4111-1111-1111-1234"));
cart.checkout(); // 💳 Paid $1029.98 with Credit Card ending 1234

// Or with lambdas! (since it's @FunctionalInterface)
cart.setPaymentStrategy(amount -> {
    System.out.println("🎁 Paid with Gift Card: $" + amount);
    return true;
});
```

---

## 🌍 Real-World Examples

```java
// ☕ Java's Comparator IS a Strategy!
List<String> names = Arrays.asList("Charlie", "Alice", "Bob");
names.sort(Comparator.naturalOrder());         // Strategy 1
names.sort(Comparator.reverseOrder());         // Strategy 2
names.sort(Comparator.comparingInt(String::length)); // Strategy 3

// 🌱 Spring Validators — each is a strategy!
// 🎮 Game AI: AggressiveStrategy, DefensiveStrategy, PassiveStrategy
// 📦 Compression: GzipStrategy, ZipStrategy, Brotli Strategy
// 🔐 Authentication: JWTStrategy, OAuth2Strategy, BasicAuthStrategy
```

---

## ⚡ When to Use

```
✅ Multiple algorithms for the same task (sort, validate, compress, route)
✅ Want to switch algorithm at runtime based on user choice / config
✅ Eliminate large if-else/switch for algorithm selection
✅ Need to isolate algorithm logic for testing

❌ Only 2 simple alternatives — if-else is fine!
❌ Algorithm never changes — just hardcode it
❌ Client doesn't need to know about strategies — maybe Template Method instead
```

---

## 🎯 Interview Key: Strategy vs State

```
STRATEGY: Client CHOOSES which algorithm to use (conscious selection)
STATE: Object AUTOMATICALLY changes behavior based on internal state

Strategy = YOU pick the music playlist
State = Traffic light changes automatically (red→green→yellow→red)
```

---

## 🏆 Achievement: Progress: 13/23 patterns █████████████░

---

*← [Behavioral Overview](./00_Behavioral_Overview.md) | [Observer →](./Observer.md)*
