# 🏗️ Low-Level Design: From Concepts to Code 🚀

> **"Low-level design is where good intentions meet reality — where principles become class diagrams and object interactions."**

Low-Level Design (LLD) bridges the gap between system architecture (HLD) and working code. This domain teaches you to design individual components, classes, and their interactions — a core skill tested at every senior/staff engineering interview.

---

## 📖 What is Low-Level Design?

```
┌────────────────────────────────────────────────────────┐
│          Software Design Levels                        │
├────────────────────────────────────────────────────────┤
│  HIGH-LEVEL DESIGN (HLD)                               │
│  • Distributed systems, services, databases            │
│  • How components interact across machines             │
│  • Example: Design a URL shortener (entire system)     │
├────────────────────────────────────────────────────────┤
│  LOW-LEVEL DESIGN (LLD)  ← This domain                 │
│  • Classes, interfaces, methods, relationships         │
│  • How objects interact within a single service        │
│  • Example: Design the URL entity + storage classes    │
├────────────────────────────────────────────────────────┤
│  IMPLEMENTATION                                        │
│  • Actual code, data structures, algorithms            │
│  • Unit tests, edge cases                              │
└────────────────────────────────────────────────────────┘
```

---

## 📂 What's In This Domain?

| File | Topic | Level |
|------|-------|-------|
| [01_OOP_Design_Principles.md](./01_OOP_Design_Principles.md) | SOLID, DRY, YAGNI, KISS with Java 17+ | 🟡 Intermediate |
| [02_Design_A_Parking_Lot.md](./02_Design_A_Parking_Lot.md) | Classic LLD problem: full class design | 🟡 Intermediate |

---

## 🛣️ Reading Order

```
README (this file)
    ↓
01_OOP_Design_Principles.md  — Learn the principles
    ↓
02_Design_A_Parking_Lot.md   — Apply them to a real problem
```

---

## 🔑 LLD vs HLD Quick Reference

| Aspect | LLD | HLD |
|--------|-----|-----|
| Focus | Classes, interfaces, methods | Services, databases, APIs |
| Tools | UML class diagrams, sequence diagrams | Architecture diagrams |
| Key Concepts | SOLID, design patterns, OOP | CAP theorem, scalability, sharding |
| Interview Style | "Design this class/module" | "Design this entire system" |
| Duration | 30-45 min | 45-60 min |
| Output | Class diagram + Java code | Architecture diagram + tech choices |

---

## 🏆 Most Common LLD Interview Questions

| System | Core Classes | Key Patterns |
|--------|-------------|-------------|
| Parking Lot | `ParkingLot`, `ParkingSpot`, `Vehicle`, `Ticket` | Factory, Strategy |
| Library System | `Book`, `Member`, `Loan`, `Catalog` | Observer, Builder |
| ATM Machine | `ATM`, `Card`, `Account`, `Transaction` | State, Command |
| Elevator System | `Elevator`, `Floor`, `Request`, `Scheduler` | State, Strategy |
| Chess Game | `Board`, `Piece`, `Move`, `Player` | Factory, Strategy |
| Hotel Booking | `Room`, `Reservation`, `Guest`, `Payment` | Builder, Observer |
| Snake & Ladder | `Board`, `Player`, `Snake`, `Ladder` | Strategy |
| Vending Machine | `VendingMachine`, `Item`, `Inventory` | State, Factory |
| Log System | `Logger`, `Appender`, `Formatter` | Singleton, Chain of Resp. |
| File System | `File`, `Directory`, `Permission` | Composite, Decorator |

---

## 🔗 Related Domains

- **[DesignPattern/](../DesignPattern/)** — Master patterns before LLD problems
- **[Principles/](../Principles/)** — SOLID and Clean Code principles
- **[DataStructures/JavaCollections/](../DataStructures/JavaCollections/)** — Core data structures you'll use in LLD
- **[Architectures/](../Architectures/)** — From LLD to full system design

---

*Next: [OOP Design Principles →](./01_OOP_Design_Principles.md)*
