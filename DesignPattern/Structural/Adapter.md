# 🔌 Adapter Pattern: Making Incompatible Things Work Together! 🎯

> **"You don't throw away your European appliances when you move to the US. You get an adapter."**

---

## 📋 Table of Contents

1. [The Story](#-the-story)
2. [The Problem](#-the-problem)
3. [The Solution](#-the-solution)
4. [The Structure](#-the-structure)
5. [Java Implementation](#-java-implementation)
6. [Real-World Examples](#-real-world-examples)
7. [When to Use & When NOT](#-when-to-use--when-not)
8. [Puzzle Time](#-puzzle-time)
9. [Interview Q&A](#-interview-qa)
10. [Achievement](#-achievement-unlocked)

---

## 🎬 The Story

### 🔌 The Travel Adapter

You're an American developer traveling to Europe. Your laptop charger has a US plug (two flat prongs), but European outlets have round holes.

```
🇺🇸 US PLUG ──╲   ╱── 🇪🇺 EU OUTLET
(flat prongs)   ╳       (round holes)
              ╱   ╲     
         INCOMPATIBLE! 💥

🇺🇸 US PLUG ──→ [🔌 ADAPTER] ──→ 🇪🇺 EU OUTLET
(flat prongs)    (converts shape)   (round holes)
         NOW IT WORKS! ✅
```

**The adapter doesn't change your plug or the outlet. It sits BETWEEN them, translating one interface to another.**

---

## 🤔 The Problem

```java
// 🎯 Your system expects this interface:
interface MediaPlayer {
    void play(String filename);
}

// 🎵 You have this working player:
class MP3Player implements MediaPlayer {
    @Override
    public void play(String filename) {
        System.out.println("🎵 Playing MP3: " + filename);
    }
}

// 🎬 But now you got a THIRD-PARTY library with a DIFFERENT interface:
class VLCPlayer {
    // ❌ DIFFERENT method name!
    // ❌ DIFFERENT parameter types!
    public void playMedia(File file, String codec, int volume) {
        System.out.println("🎬 VLC playing: " + file.getName());
    }
}

// 😱 VLCPlayer doesn't implement MediaPlayer!
// 😱 You CAN'T modify VLCPlayer (it's a third-party library)!
// 😱 Your code ONLY works with MediaPlayer interface!
// 🤔 NOW WHAT?!
```

---

## 💡 The Solution

```
ADAPTER = The bridge between two incompatible interfaces

It wraps the "foreign" object and translates calls from
YOUR interface to the foreign object's interface.

YOUR CODE → [Adapter] → THIRD-PARTY CODE
(speaks English)  (translator)  (speaks French)
```

---

## 🏗️ The Structure

### Object Adapter (Composition — Preferred ✅)

```
┌──────────────────┐          ┌────────────────────┐
│     Client        │          │  Target Interface   │
│                   │─────────▶│ (what client wants) │
│ Uses Target       │          │  + request()        │
└──────────────────┘          └──────────┬─────────┘
                                         │ implements
                              ┌──────────▼─────────┐
                              │      Adapter        │
                              │ ─────────────────── │
                              │ - adaptee: Adaptee  │ ← HAS-A
                              │ + request()         │
                              │   → adaptee.        │
                              │     specificReq()   │
                              └──────────┬─────────┘
                                         │ has reference to
                              ┌──────────▼─────────┐
                              │      Adaptee        │
                              │ (incompatible class)│
                              │ + specificRequest() │
                              └────────────────────┘
```

### Class Adapter (Inheritance — Less Common)

```
┌──────────────────┐     ┌──────────────────┐
│  Target Interface │     │     Adaptee       │
│  + request()      │     │ + specificReq()   │
└────────┬─────────┘     └────────┬─────────┘
         │ implements              │ extends
         └───────────┬─────────────┘
                     │
           ┌─────────▼─────────┐
           │      Adapter       │
           │ ───────────────── │
           │ + request()        │
           │   → specificReq()  │  ← Calls inherited method
           └───────────────────┘
```

---

## 💻 Java Implementation

### Scenario: Integrate Third-Party Payment Library

```java
// 🎯 YOUR system's interface (what your code expects)
public interface PaymentProcessor {
    boolean processPayment(String customerId, double amount, String currency);
    PaymentStatus getStatus(String transactionId);
}

// 🏦 THIRD-PARTY library (can't modify!)
// Maybe it's Stripe's SDK with a different API...
public class StripeAPI {
    public StripeCharge createCharge(long amountInCents, String curr, 
                                     String customerToken) {
        System.out.println("💳 Stripe: Charging " + amountInCents + " cents");
        return new StripeCharge("ch_" + System.currentTimeMillis(), "succeeded");
    }
    
    public StripeCharge retrieveCharge(String chargeId) {
        return new StripeCharge(chargeId, "succeeded");
    }
}

// 🔌 THE ADAPTER — bridges YOUR interface with Stripe's API
public class StripePaymentAdapter implements PaymentProcessor {
    
    private final StripeAPI stripe; // Composition: HAS-A Adaptee
    
    public StripePaymentAdapter(StripeAPI stripe) {
        this.stripe = stripe;
    }
    
    @Override
    public boolean processPayment(String customerId, double amount, String currency) {
        // 🔄 TRANSLATE: Our interface → Stripe's interface
        long amountInCents = Math.round(amount * 100); // dollars → cents
        String customerToken = "cus_" + customerId;     // our ID → Stripe token
        
        StripeCharge charge = stripe.createCharge(amountInCents, currency, customerToken);
        return "succeeded".equals(charge.getStatus());
    }
    
    @Override
    public PaymentStatus getStatus(String transactionId) {
        StripeCharge charge = stripe.retrieveCharge(transactionId);
        // 🔄 TRANSLATE: Stripe's status → Our status enum
        return switch (charge.getStatus()) {
            case "succeeded" -> PaymentStatus.COMPLETED;
            case "pending"   -> PaymentStatus.PROCESSING;
            case "failed"    -> PaymentStatus.FAILED;
            default          -> PaymentStatus.UNKNOWN;
        };
    }
}
```

### Client Code — Clean! 😍

```java
public class OrderService {
    private final PaymentProcessor paymentProcessor;
    
    // 🎯 Doesn't know or care it's Stripe underneath!
    public OrderService(PaymentProcessor paymentProcessor) {
        this.paymentProcessor = paymentProcessor;
    }
    
    public void checkout(String customerId, double total) {
        boolean success = paymentProcessor.processPayment(
            customerId, total, "USD"
        );
        if (success) {
            System.out.println("✅ Payment successful!");
        }
    }
}

// 🔧 Wiring (could be done by Spring DI)
StripeAPI stripe = new StripeAPI();
PaymentProcessor processor = new StripePaymentAdapter(stripe);
OrderService orderService = new OrderService(processor);
orderService.checkout("user123", 99.99);

// 🔄 Switch to PayPal? Just create a PayPalAdapter!
// PaymentProcessor processor = new PayPalAdapter(new PayPalSDK());
// ZERO changes to OrderService! ✅
```

### Two-Way Adapter (Advanced)

```java
// 🔄 When you need to adapt BOTH directions
public class TwoWayAdapter implements MediaPlayer, AdvancedMediaPlayer {
    private MediaPlayer basicPlayer;
    private AdvancedMediaPlayer advancedPlayer;
    
    public TwoWayAdapter(MediaPlayer player) {
        this.basicPlayer = player;
    }
    
    public TwoWayAdapter(AdvancedMediaPlayer player) {
        this.advancedPlayer = player;
    }
    
    @Override
    public void play(String file) {
        if (basicPlayer != null) basicPlayer.play(file);
        else advancedPlayer.playAdvanced(file, "auto");
    }
    
    @Override
    public void playAdvanced(String file, String codec) {
        if (advancedPlayer != null) advancedPlayer.playAdvanced(file, codec);
        else basicPlayer.play(file); // Ignores codec for basic player
    }
}
```

---

## 🌍 Real-World Examples

### 1. Java's Arrays.asList()

```java
// 📋 Array → List adapter!
String[] array = {"A", "B", "C"};
List<String> list = Arrays.asList(array);
// Array interface ADAPTED to List interface!
```

### 2. InputStreamReader (Byte stream → Character stream)

```java
// 🔤 Adapts InputStream (bytes) to Reader (characters)
InputStream byteStream = new FileInputStream("file.txt");
Reader charStream = new InputStreamReader(byteStream, "UTF-8");
// Different interfaces, adapter makes them compatible!
```

### 3. Spring MVC's HandlerAdapter

```java
// 🌱 Spring adapts various handler types to a uniform interface
// @Controller methods, HttpRequestHandler, Servlet — all adapted!
public interface HandlerAdapter {
    boolean supports(Object handler);
    ModelAndView handle(HttpServletRequest req, HttpServletResponse res, Object handler);
}
```

### 4. SLF4J (Logging Facade)

```java
// 📝 SLF4J adapts different logging frameworks to ONE interface
Logger logger = LoggerFactory.getLogger(MyClass.class);
// Could be Log4j2, Logback, java.util.logging underneath
// SLF4J acts as an adapter to all of them!
```

---

## ⚡ When to Use & When NOT

### ✅ USE Adapter When:

```
1. 🔌 Third-party library has incompatible interface
2. 🏗️ Legacy code needs to work with new code
3. 🧪 You want to test against a mock of the adapted interface
4. 🔄 You need to swap implementations without changing clients
5. 📦 Multiple classes need unified interface (normalize APIs)
```

### ❌ DON'T Use Adapter When:

```
1. 🎯 You CAN modify the source class — just change the interface!
2. 🤏 The mismatch is trivial — a simple method call is fine
3. 🏗️ The interfaces are too different — adapter becomes too complex
4. ⚡ Performance-critical path — extra indirection hurts
```

---

## 🧩 Puzzle Time

### Puzzle 1: Design an Adapter

```
Your system uses Celsius temperatures throughout.
You got a new WeatherAPI that returns Fahrenheit.

YOUR interface:    ThermometerService.getTemperature() → double (Celsius)
THEIR interface:   WeatherAPI.getFahrenheit() → float

Design the adapter!
```

<details>
<summary>🔑 Solution</summary>

```java
public class WeatherAPIAdapter implements ThermometerService {
    private final WeatherAPI api;
    
    public WeatherAPIAdapter(WeatherAPI api) { this.api = api; }
    
    @Override
    public double getTemperature() {
        float fahrenheit = api.getFahrenheit();
        return (fahrenheit - 32) * 5.0 / 9.0; // Convert to Celsius!
    }
}
```
</details>

---

## 🎯 Interview Q&A

### Q1: "Adapter vs Decorator?"

```
ADAPTER:  Changes the INTERFACE (makes X look like Y)
DECORATOR: Keeps the SAME interface, adds BEHAVIOR

Adapter   = Power plug converter (different shape, same electricity)
Decorator = Phone case (same phone interface, adds protection)
```

### Q2: "Object Adapter vs Class Adapter?"

```
Object Adapter (Composition):     Class Adapter (Inheritance):
✅ More flexible                  ❌ Less flexible (single inheritance)
✅ Can adapt subclasses too       ❌ Tied to one specific class
✅ Follows composition over       ✅ Can override adaptee behavior
   inheritance                    ❌ Not available in many languages
✅ Preferred in Java!             ❌ Rarely used in practice
```

---

## 🏆 Achievement Unlocked!

```
╔══════════════════════════════════════╗
║  🔌 ADAPTER PATTERN MASTER          ║
║                                      ║
║  You now understand:                 ║
║  ✅ Interface translation            ║
║  ✅ Object vs Class adapter          ║
║  ✅ Third-party integration          ║
║  ✅ Adapter vs Decorator vs Proxy    ║
║                                      ║
║  Progress: 6/23 patterns ██████░░   ║
╚══════════════════════════════════════╝
```

---

*← [Prototype](../Creational/Prototype.md) | [Bridge →](./Bridge.md)*

#DesignPatterns #Adapter #Java #InterviewPrep 🚀
