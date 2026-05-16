# 🎭 Behavioral Design Patterns: The Art of Object Communication 🎯

> **"It's not enough to have objects — they need to TALK to each other intelligently."**

---

## 🤔 What Are Behavioral Patterns?

Behavioral patterns are concerned with **algorithms** and **assignment of responsibilities** between objects. They describe how objects communicate, interact, and distribute work.

```
CREATIONAL  = How objects are BORN
STRUCTURAL  = How objects are ORGANIZED
BEHAVIORAL  = How objects COMMUNICATE & BEHAVE
```

---

## 🎯 The 11 Behavioral Patterns at a Glance

```
🎭 BEHAVIORAL PATTERNS FAMILY
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

 1. 🎯 STRATEGY        → "Swap algorithms at runtime"
 2. 👀 OBSERVER        → "Notify when something changes"
 3. 📜 COMMAND         → "Encapsulate requests as objects"
 4. 📐 TEMPLATE METHOD → "Define skeleton, let kids fill in"
 5. 🔄 ITERATOR        → "Traverse without exposing internals"
 6. 🚦 STATE           → "Behavior changes with state"
 7. ⛓️ CHAIN OF RESP.  → "Pass until someone handles it"
 8. 📡 MEDIATOR        → "Central hub for communication"
 9. 💾 MEMENTO         → "Save and restore state"
10. 🚶 VISITOR         → "Add operations externally"
11. 📖 INTERPRETER     → "Parse a mini language"
```

---

## 🔄 Quick Decision Guide

| I need to... | Pattern |
|-------------|---------|
| Switch algorithms without changing client | **Strategy** |
| Notify multiple objects of changes | **Observer** |
| Queue, undo, or log operations | **Command** |
| Define algorithm skeleton, vary steps | **Template Method** |
| Walk through a collection generically | **Iterator** |
| Change behavior based on object state | **State** |
| Let multiple handlers try a request | **Chain of Responsibility** |
| Reduce direct dependencies between components | **Mediator** |
| Implement undo/restore/snapshots | **Memento** |
| Add new operations without modifying classes | **Visitor** |
| Parse expressions or simple grammars | **Interpreter** |

---

## 📊 Comparison Matrix

| Pattern | Complexity | Interview Freq | Spring Uses It? | Java SDK Uses It? |
|---------|:----------:|:--------------:|:---------------:|:-----------------:|
| Strategy | 🟢 Easy | ⭐⭐⭐⭐⭐ | ✅ Validators | ✅ Comparator |
| Observer | 🟢 Easy | ⭐⭐⭐⭐⭐ | ✅ Events | ✅ PropertyChangeListener |
| Command | 🟡 Med | ⭐⭐⭐⭐ | ✅ Spring Batch | ✅ Runnable |
| Template Method | 🟢 Easy | ⭐⭐⭐⭐ | ✅ JdbcTemplate | ✅ AbstractList |
| Iterator | 🟢 Easy | ⭐⭐⭐ | ✅ | ✅ Iterator<T> |
| State | 🟡 Med | ⭐⭐⭐⭐ | ✅ StateMachine | — |
| Chain of Resp | 🟡 Med | ⭐⭐⭐ | ✅ Filters | ✅ Servlet Filters |
| Mediator | 🟡 Med | ⭐⭐⭐ | ✅ MediatR equiv | — |
| Memento | 🟡 Med | ⭐⭐⭐ | — | ✅ Serialization |
| Visitor | 🔴 Hard | ⭐⭐ | ✅ BeanDefinitionVisitor | — |
| Interpreter | 🔴 Hard | ⭐ | ✅ SpEL | ✅ Pattern/Regex |

---

## 🤝 Common Combinations

```
🎯 Strategy + Factory:
   Factory creates the right Strategy based on context

👀 Observer + Mediator:
   Mediator uses Observer to notify colleagues

📜 Command + Memento:
   Command executes, Memento stores state for undo

🚦 State + Strategy:
   State delegates behavior to Strategy objects

📐 Template Method + Strategy:
   Template defines flow, Strategy handles variable steps
```

---

## 📚 Deep Dive Links

| # | Pattern | Tutorial | Time |
|---|---------|----------|:----:|
| 13 | Strategy | [→](./Strategy.md) | ⏱️ 25min |
| 14 | Observer | [→](./Observer.md) | ⏱️ 25min |
| 15 | Command | [→](./Command.md) | ⏱️ 30min |
| 16 | Template Method | [→](./Template_Method.md) | ⏱️ 20min |
| 17 | Iterator | [→](./Iterator.md) | ⏱️ 20min |
| 18 | State | [→](./State.md) | ⏱️ 30min |
| 19 | Chain of Responsibility | [→](./Chain_of_Responsibility.md) | ⏱️ 25min |
| 20 | Mediator | [→](./Mediator.md) | ⏱️ 25min |
| 21 | Memento | [→](./Memento.md) | ⏱️ 20min |
| 22 | Visitor | [→](./Visitor.md) | ⏱️ 30min |
| 23 | Interpreter | [→](./Interpreter.md) | ⏱️ 25min |

---

*← [Structural Patterns](../Structural/00_Structural_Overview.md) | [Strategy (first behavioral) →](./Strategy.md)*

#DesignPatterns #Behavioral #Java #InterviewPrep 🚀
