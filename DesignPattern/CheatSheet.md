# 📋 Design Patterns CheatSheet

> **One-liner for each pattern. Print this. Tape it to your wall.**

---

## 🏗️ Creational Patterns (Object Creation)

| Pattern | One-Liner | Remember As |
|---------|-----------|-------------|
| **Singleton** | One instance, global access | "There can be only ONE" |
| **Factory Method** | Subclass decides which class to instantiate | "Pizza shop — each branch makes its own pizza" |
| **Abstract Factory** | Family of related objects without specifying classes | "IKEA furniture sets — matching table + chair + sofa" |
| **Builder** | Step-by-step construction of complex objects | "Subway sandwich — bread, then meat, then veggies" |
| **Prototype** | Clone existing objects instead of creating new | "Dolly the sheep — copy, don't build from scratch" |

---

## 🧱 Structural Patterns (Object Composition)

| Pattern | One-Liner | Remember As |
|---------|-----------|-------------|
| **Adapter** | Convert interface to one client expects | "Power plug adapter — EU plug → US socket" |
| **Bridge** | Separate abstraction from implementation | "Remote + Device — any remote works with any device" |
| **Composite** | Treat single and group objects uniformly | "File system — file & folder both have size()" |
| **Decorator** | Add behavior dynamically by wrapping | "Coffee + milk + sugar — each wraps the previous" |
| **Facade** | Simple interface to complex subsystem | "Hotel concierge — one call handles everything" |
| **Flyweight** | Share common state between many objects | "Characters in Word — 'a' glyph shared 1000x" |
| **Proxy** | Placeholder that controls access to real object | "Credit card = proxy for bank account" |

---

## 🎭 Behavioral Patterns (Object Communication)

| Pattern | One-Liner | Remember As |
|---------|-----------|-------------|
| **Strategy** | Swap algorithms at runtime | "Google Maps — driving / walking / cycling" |
| **Observer** | Notify dependents when state changes | "YouTube subscribe — new video → all notified" |
| **Command** | Encapsulate request as object (undo/redo) | "Restaurant order slip — waiter carries it to chef" |
| **Template Method** | Skeleton in base class, steps in subclass | "Recipe template — steps fixed, ingredients vary" |
| **Iterator** | Sequential access without exposing internals | "TV remote — next channel, don't care how channels stored" |
| **State** | Behavior changes when state changes | "Vending machine — behavior depends on: has coin? has item?" |
| **Chain of Responsibility** | Pass request along chain until handled | "Tech support — bot → junior → senior" |
| **Mediator** | Central hub coordinates communication | "Air traffic control — pilots don't talk to each other" |
| **Memento** | Save/restore object state | "Ctrl+Z — save snapshot, restore later" |
| **Visitor** | Add operations without modifying classes | "Health inspector visits all restaurants" |
| **Interpreter** | Grammar rules as class hierarchy | "Calculator expression tree — 3 + 5 * 2" |

---

## ⚡ Quick Decision Guide

```
Need to CREATE objects?          → Creational patterns
Need to COMPOSE/STRUCTURE?       → Structural patterns  
Need to COMMUNICATE/COORDINATE?  → Behavioral patterns
```

### Creational — Which One?

```
One instance only?               → Singleton
Object type decided at runtime?  → Factory Method
Family of objects?               → Abstract Factory
Complex object, many params?     → Builder
Copy existing object?            → Prototype
```

### Structural — Which One?

```
Incompatible interface?          → Adapter
Vary abstraction & impl?         → Bridge
Tree/hierarchy of objects?       → Composite
Add features dynamically?        → Decorator
Simplify complex API?            → Facade
Tons of similar objects?         → Flyweight
Control access / lazy load?      → Proxy
```

### Behavioral — Which One?

```
Swap algorithm at runtime?       → Strategy
Notify on state change?          → Observer
Undo/redo, queue requests?       → Command
Same steps, different details?   → Template Method
Traverse collection?             → Iterator
Object acts differently per state? → State
Multiple handlers, pass along?   → Chain of Responsibility
Reduce coupling many-to-many?    → Mediator
Save/restore snapshots?          → Memento
New ops without modifying class? → Visitor
Parse a grammar/DSL?             → Interpreter
```

---

## 🏢 Big Tech Favorites

| Company | Top Patterns |
|---------|-------------|
| **Google** | Observer, Strategy, Builder, Flyweight |
| **Netflix** | Chain of Responsibility, Strategy, Proxy, Command |
| **Amazon** | Factory, Builder, Observer, Strategy |
| **Meta** | Observer, Mediator, Command, Composite |
| **Apple** | Singleton, Observer, Delegate (Strategy), Composite |

---

## 🚫 Common Confusions

```
Adapter vs Decorator:    Adapter CHANGES interface. Decorator ADDS behavior, SAME interface.
Strategy vs State:       Strategy = client CHOOSES. State = object TRANSITIONS automatically.
Proxy vs Decorator:      Proxy CONTROLS access. Decorator ADDS features.
Factory vs Builder:      Factory = one step. Builder = many steps.
Observer vs Mediator:    Observer = 1-to-many broadcast. Mediator = many-to-many coordination.
Command vs Memento:      Command stores OPERATION (undo = inverse). Memento stores STATE (undo = restore).
```

---

*[← Back to README](./README.md)*
