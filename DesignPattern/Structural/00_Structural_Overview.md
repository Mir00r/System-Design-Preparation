# 🏗️ Structural Design Patterns: The Art of Object Composition 🎯

> **"It's not about what objects ARE — it's about how they're PUT TOGETHER."**

---

## 📋 Table of Contents

1. [What Are Structural Patterns?](#-what-are-structural-patterns)
2. [The 7 Structural Patterns](#-the-7-structural-patterns-at-a-glance)
3. [When to Use Which?](#-when-to-use-which)
4. [Comparison Matrix](#-comparison-matrix)
5. [Pattern Combinations](#-real-world-combinations)
6. [Interview Quick-Fire](#-interview-quick-fire)

---

## 🤔 What Are Structural Patterns?

Structural patterns explain how to assemble objects and classes into **larger structures** while keeping these structures **flexible and efficient**.

### 🎬 The LEGO Analogy

```
🧱 STRUCTURAL PATTERNS = LEGO BUILDING TECHNIQUES

Without structural patterns:
→ You have a pile of random bricks 🧱🧱🧱🧱
→ No plan for how to connect them
→ The structure is fragile and can't grow

With structural patterns:
→ You have TECHNIQUES for connecting bricks 🏰
→ Adapter: Connect bricks that don't normally fit
→ Bridge: Separate the blueprint from the bricks
→ Composite: Small builds combine into big builds
→ Decorator: Add accessories without rebuilding
→ Facade: Put a pretty exterior on complex internals
→ Flyweight: Share common bricks across builds
→ Proxy: Put a protective layer around delicate parts
```

### 💡 Core Theme

```
ALL structural patterns answer ONE question:

"How do I organize my classes/objects into larger 
 structures WITHOUT creating tight coupling?"

KEY TECHNIQUES THEY USE:
━━━━━━━━━━━━━━━━━━━━━━━
1. COMPOSITION → Wrap objects inside other objects
2. INHERITANCE → Extend to adapt interfaces
3. DELEGATION → Pass work to contained objects
4. ABSTRACTION → Hide complexity behind simple interfaces
```

---

## 🎯 The 7 Structural Patterns at a Glance

```
🏗️ STRUCTURAL PATTERNS FAMILY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. 🔌 ADAPTER (Wrapper)
   "Make incompatible things work together"
   → Like a USB-C to HDMI adapter

2. 🌉 BRIDGE
   "Separate what from how"
   → Decouple abstraction from implementation

3. 🌳 COMPOSITE
   "Treat one and many the same way"
   → Tree structures, file systems

4. 🎀 DECORATOR (Wrapper)
   "Add features dynamically"
   → Like adding toppings to coffee

5. 🏨 FACADE
   "Simple door to complex room"
   → Like a hotel concierge

6. 🪶 FLYWEIGHT
   "Share to save memory"
   → Like shared fonts in a document

7. 🛡️ PROXY
   "Control access through a representative"
   → Like a security guard or cache
```

---

## 🔄 When to Use Which?

### The Decision Flowchart

```
Structure problem?
│
├─ Two things have INCOMPATIBLE INTERFACES?
│  └─ YES → 🔌 ADAPTER
│
├─ Need to vary ABSTRACTION and IMPLEMENTATION independently?
│  └─ YES → 🌉 BRIDGE
│
├─ Need TREE/HIERARCHY where leaf and branch are treated the same?
│  └─ YES → 🌳 COMPOSITE
│
├─ Need to ADD BEHAVIOR without modifying the class?
│  └─ YES → 🎀 DECORATOR
│
├─ Need to SIMPLIFY a complex subsystem?
│  └─ YES → 🏨 FACADE
│
├─ Have THOUSANDS of similar objects eating memory?
│  └─ YES → 🪶 FLYWEIGHT
│
└─ Need to CONTROL ACCESS (lazy load, cache, security)?
   └─ YES → 🛡️ PROXY
```

### Quick Comparison Table

| Scenario | Pattern | Example |
|----------|---------|---------|
| Old API → New API translation | **Adapter** | JDBC drivers wrapping vendor APIs |
| Remote + Shape have independent variants | **Bridge** | Shape drawing on different renderers |
| Menu items can be single OR group | **Composite** | File system (files + folders) |
| Add logging/caching without modifying code | **Decorator** | Java I/O streams |
| Simplify complex library usage | **Facade** | JPA hiding SQL complexity |
| 10,000 tree objects in a forest | **Flyweight** | Character formatting in text editors |
| Lazy-load expensive image | **Proxy** | Hibernate lazy loading |

---

## 📊 Comparison Matrix

| Feature | Adapter | Bridge | Composite | Decorator | Facade | Flyweight | Proxy |
|---------|:-------:|:------:|:---------:|:---------:|:------:|:---------:|:-----:|
| **Intent** | Convert interface | Decouple abs/impl | Tree structure | Add behavior | Simplify | Save memory | Control access |
| **Wraps** | 1 object | N/A | Many objects | 1 object | Many objects | Shared state | 1 object |
| **Interface** | Changes | Same | Uniform | Same | New simple | Same | Same |
| **Complexity** | 🟢 | 🟡 | 🟡 | 🟡 | 🟢 | 🔴 | 🟡 |
| **Interview Freq** | ⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ |

### 🤯 The "Wrapper" Confusion Cleared Up

Three patterns wrap objects — here's how they differ:

```
ADAPTER vs DECORATOR vs PROXY — All wrap, but WHY?
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

🔌 ADAPTER:
   Purpose: Change the INTERFACE
   Before:  RoundPeg → (can't fit) → SquareHole
   After:   RoundPeg → [Adapter] → SquareHole ✅
   Analogy: Translator (same meaning, different language)

🎀 DECORATOR:
   Purpose: Add new BEHAVIOR/RESPONSIBILITY
   Before:  Coffee (plain)
   After:   Coffee → [MilkDecorator] → [SugarDecorator]
   Analogy: Adding layers (each adds something new)

🛡️ PROXY:
   Purpose: Control ACCESS to the original
   Before:  Direct database access
   After:   Client → [CacheProxy] → Database
   Analogy: Bodyguard (same person, controlled access)
```

---

## 🤝 Real-World Combinations

```
🌐 SPRING MVC (Adapter + Facade + Proxy):
├── DispatcherServlet = Facade (simplifies request handling)
├── HandlerAdapter = Adapter (adapts different controller types)
└── AOP Proxies = Proxy (add cross-cutting concerns)

📱 ANDROID UI (Composite + Decorator + Adapter):
├── ViewGroup = Composite (contains Views or other ViewGroups)
├── RecyclerView.Adapter = Adapter (adapts data to views)
└── ItemDecoration = Decorator (adds dividers without modifying items)

💻 JAVA I/O (Decorator):
InputStream                           → Base component
├── FileInputStream                   → Concrete component
├── BufferedInputStream               → Decorator (adds buffering)
├── DataInputStream                   → Decorator (adds typed reads)
└── GZIPInputStream                   → Decorator (adds decompression)
```

---

## 🎯 Interview Quick-Fire

### Q1: "Adapter vs Bridge — both connect things?"

```
ADAPTER:
• Works with EXISTING incompatible interfaces
• Applied AFTER design (retrofit)
• "Make things work together"

BRIDGE:
• Designed UP FRONT for flexibility
• Prevents class explosion
• "Prevent tight coupling before it happens"

Timeline: Bridge = preventive medicine 💊
          Adapter = emergency surgery 🏥
```

### Q2: "Decorator vs Inheritance — why not just subclass?"

```
INHERITANCE for adding features:
class CoffeeWithMilk extends Coffee { }
class CoffeeWithSugar extends Coffee { }
class CoffeeWithMilkAndSugar extends Coffee { }
class CoffeeWithMilkAndSugarAndWhip extends Coffee { }
// 💥 EXPLOSION! 2^n subclasses for n features!

DECORATOR:
Coffee c = new Whip(new Sugar(new Milk(new SimpleCoffee())));
// ✅ Any combination, no new classes needed!
```

### Q3: "When is Facade an anti-pattern?"

**A:** When it becomes a "God Object" — if the Facade does too much or knows too much about internals, it defeats the purpose. A good Facade delegates, it doesn't implement business logic itself.

---

## 🎮 Pattern Identification Game

```
Match the scenario to the pattern:

1. "Netflix shows thumbnails that load full images only when scrolled into view"
   → ________________

2. "Java's Arrays.asList() returns a List backed by an array"
   → ________________

3. "Company org chart where departments contain teams that contain people"
   → ________________

4. "Starbucks adds milk, syrup, whip — each adds cost to base coffee"
   → ________________

5. "AWS SDK provides simple methods hiding complex HTTP/auth logic"
   → ________________
```

<details>
<summary>🔑 Answers</summary>

```
1. Proxy (Virtual Proxy — lazy loading)
2. Adapter (Array interface → List interface)
3. Composite (tree hierarchy, uniform treatment)
4. Decorator (add features dynamically)
5. Facade (simple interface to complex subsystem)
```
</details>

---

## 📚 Deep Dive into Each Pattern

| # | Pattern | Tutorial Link | Estimated Time |
|---|---------|---------------|:--------------:|
| 6 | Adapter | [Deep Dive →](./Adapter.md) | ⏱️ 25 min |
| 7 | Bridge | [Deep Dive →](./Bridge.md) | ⏱️ 30 min |
| 8 | Composite | [Deep Dive →](./Composite.md) | ⏱️ 30 min |
| 9 | Decorator | [Deep Dive →](./Decorator.md) | ⏱️ 30 min |
| 10 | Facade | [Deep Dive →](./Facade.md) | ⏱️ 20 min |
| 11 | Flyweight | [Deep Dive →](./Flyweight.md) | ⏱️ 25 min |
| 12 | Proxy | [Deep Dive →](./Proxy.md) | ⏱️ 30 min |

---

*← [Creational Patterns](../Creational/00_Creational_Overview.md) | [Behavioral Patterns →](../Behavioral/00_Behavioral_Overview.md)*

#DesignPatterns #Structural #Java #InterviewPrep 🚀
