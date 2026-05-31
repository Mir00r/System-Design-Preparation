# 🗺️ System Design Repository: Concept Map & Knowledge Graph 🚀

**⏱️ 10 minutes** | **🎯 🟢 Beginner** | **🔗 Prerequisites**: None — start here!

> **"You can't connect the dots looking forward; you can only connect them looking backward."** — Steve Jobs

This map shows how all 27 domains in this repository relate to each other, helping you navigate from fundamentals to advanced topics.

---

## 📋 Table of Contents

1. [Repository Overview](#1-repository-overview)
2. [Domain Relationship Map](#2-domain-relationship-map)
3. [Learning Paths](#3-learning-paths)
4. [Domain Index](#4-domain-index)
5. [Knowledge Clusters](#5-knowledge-clusters)
6. [Where to Start](#6-where-to-start)

---

## 1. Repository Overview

```
┌─────────────────────────────────────────────────────────────────────┐
│            SYSTEM DESIGN PREPARATION REPOSITORY                     │
│                    27 Domains | ~500 Files                          │
├──────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  FOUNDATIONS ────────→ CORE SYSTEMS ────────→ INTERVIEW MASTERY     │
│                                                                      │
│  • Data Structures      • System Design          • Behavioral        │
│  • Algorithms           • Architecture           • CompanyProblems   │
│  • Java Core            • Database               • AI/ML             │
│  • Design Patterns      • Microservices                              │
│                         • Messaging                                  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. Domain Relationship Map

```
                     ┌─────────────────────────────────────────┐
                     │          MATHEMATICAL FOUNDATION         │
                     │  Algorithms/ ─── DataStructures/         │
                     └──────────────────┬──────────────────────┘
                                        │ builds on
                     ┌──────────────────▼──────────────────────┐
                     │            PROGRAMMING CORE              │
                     │  javaTpoints/ ─── SpringBoot/            │
                     │       └──── DataStructures/              │
                     │              JavaCollections/            │
                     └──────────────────┬──────────────────────┘
                                        │ applies to
        ┌───────────────────────────────▼───────────────────────────────┐
        │                      DESIGN LAYER                             │
        │   DesignPattern/ ─────────── LowLevelDesign/                  │
        │         └── Pattern_Selection_Guide                           │
        │   Principles/ ────────────── Engineering/                     │
        └───────┬──────────────────────────────┬───────────────────────┘
                │ shapes                        │ guides
   ┌────────────▼──────────┐      ┌─────────────▼─────────────────┐
   │   ARCHITECTURE LAYER  │      │     DATA & STORAGE LAYER       │
   │  Architectures/       │      │  Database/                     │
   │  Microservices/       │      │  KeyConcepts/                  │
   │    (CQRS, EventDriven)│      │    (CAP, ACID, Hashing)        │
   └────────────┬──────────┘      └─────────────┬─────────────────┘
                │                               │
   ┌────────────▼──────────────────────────────▼──────────────────┐
   │                   INFRASTRUCTURE LAYER                        │
   │  MessagingQ/ (Kafka) ─── BuildingBlocks/ (Caching)           │
   │  DevOps/ ─────────────── Security/ ─── APIs/                 │
   └────────────────────────────────┬─────────────────────────────┘
                                    │ deployed via
                     ┌──────────────▼──────────────────────────┐
                     │           CLOUD & OPERATIONS             │
                     │   Cloud/ ─────────── DevOps/             │
                     │   Testing/ ──────── Tradeoffs/           │
                     └──────────────────────────────────────────┘
                                    │
                     ┌──────────────▼──────────────────────────┐
                     │        INTERVIEW PREPARATION             │
                     │  Behavioral/ ─── CompanyCodingProblems/ │
                     │  AI_Core_Concepts/ ── InterviewPrep/     │
                     └─────────────────────────────────────────┘
```

---

## 3. Learning Paths

### 🎯 Path A: Backend Engineer at FAANG (12 weeks)

```
Week 1-2:   DataStructures/ → Algorithms/ → javaTpoints/
Week 3-4:   DesignPattern/ → Principles/ → Database/
Week 5-6:   Architectures/ → Microservices/ → APIs/
Week 7-8:   KeyConcepts/ → MessagingQ/ → BuildingBlocks/
Week 9-10:  Security/ → DevOps/ → Cloud/
Week 11:    Behavioral/ + CompanyCodingProblems/
Week 12:    Tradeoffs/ + InterviewPrep/ (mock interviews)
```

**See**: [ROADMAPS.md](./ROADMAPS.md) for detailed week-by-week plan

### 🎯 Path B: System Design Focus (8 weeks)

```
Week 1:  KeyConcepts/ (CAP, ACID, Hashing, Availability)
Week 2:  Database/ (Indexing, Sharding, ACID, SQL vs NoSQL)
Week 3:  BuildingBlocks/ (Caching strategies) + MessagingQ/
Week 4:  Architectures/ (Microservices, Event-Driven, CQRS)
Week 5:  APIs/ (REST, GraphQL, gRPC, Rate Limiting, Security)
Week 6:  Microservices/ + Security/ + DevOps/ basics
Week 7:  Cloud/ + Tradeoffs/ (consistency vs availability)
Week 8:  CompanyCodingProblems/ system design files
```

### 🎯 Path C: Java Deep Dive (6 weeks)

```
Week 1:  javaTpoints/ root files (versions, access, static, lambdas)
Week 2:  javaTpoints/threads/ (concurrency, virtual threads)
Week 3:  DataStructures/JavaCollections/ (all 12 files)
Week 4:  DesignPattern/ (all 3 categories + Pattern_Selection_Guide)
Week 5:  SpringBoot/ + Microservices/
Week 6:  Algorithms/ applied to Java problems
```

### 🎯 Path D: ML/AI Engineering (4 weeks)

```
Week 1:  AI_Core_Concepts/Foundations/ + MachineLearning/
Week 2:  AI_Core_Concepts/DeepLearning/ + NLP_LLMs/
Week 3:  AI_Core_Concepts/AI_System_Design/ + AI_Design_Patterns/
Week 4:  AI_Core_Concepts/AI_Java_Developers/ + relevant system design
```

---

## 4. Domain Index

| # | Domain | Files | Description | Entry Point |
|---|--------|-------|-------------|------------|
| 1 | **Algorithms/** | ~60 | Sorting, searching, graph, DP, bit manipulation | [README](./Algorithms/README.md) |
| 2 | **DataStructures/** | ~50 | Arrays, trees, graphs, heaps, linked lists | [README](./DataStructures/README.md) |
| 3 | **DataStructures/JavaCollections/** | 12 | Complete Java collections tutorial series | [Overview](./DataStructures/JavaCollections/00_Overview_and_Architecture.md) |
| 4 | **javaTpoints/** | 15+11 | Java language features, concurrency, threads | [Versions](./javaTpoints/JavaDifferentVersionInfo.md) |
| 5 | **SpringBoot/** | ~15 | Spring ecosystem, beans, AOP, data, security | [SpringBoot/](./SpringBoot/) |
| 6 | **DesignPattern/** | ~30 | 23 GoF patterns + selection guide | [README](./DesignPattern/README.md) |
| 7 | **LowLevelDesign/** | 3 | OOP principles, LLD interviews, system modeling | [README](./LowLevelDesign/README.md) |
| 8 | **Architectures/** | 10 | Clean, Hexagonal, CQRS, Microservices, Event-Driven | [Overview](./Architectures/Overview.md) |
| 9 | **Microservices/** | ~15 | Service design, communication, resilience | [Microservices/](./Microservices/) |
| 10 | **Database/** | 11 | ACID, indexing, sharding, SQL optimization | [SQL Q&A](./Database/SQL_Q&A.md) |
| 11 | **KeyConcepts/** | 6 | CAP theorem, consistency, caching, hashing | [CAP](./KeyConcepts/CAPTheorem.md) |
| 12 | **APIs/** | 7 | REST, GraphQL, gRPC, rate limiting, security | [Overview](./APIs/RESTful.md) |
| 13 | **Security/** | ~8 | Auth, encryption, OWASP, JWT | [Security/](./Security/) |
| 14 | **MessagingQ/** | 1+ | Kafka, message queues, event streaming | [Kafka](./MessagingQ/Kafka.md) |
| 15 | **BuildingBlocks/** | 2 | Caching, caching strategies | [Caching](./BuildingBlocks/Caching.md) |
| 16 | **DevOps/** | ~25 | CI/CD, Docker, Kubernetes, Git, Linux | [DevOps/](./DevOps/) |
| 17 | **Cloud/** | ~10 | AWS/GCP/Azure comparison, cloud patterns | [Cloud/](./Cloud/) |
| 18 | **Testing/** | ~5 | Test pyramid, performance testing, TDD | [Testing/](./Testing/) |
| 19 | **Tradeoffs/** | ~10 | Consistency vs availability, SQL vs NoSQL | [Tradeoffs/](./Tradeoffs/) |
| 20 | **Principles/** | ~10 | SOLID, DRY, YAGNI, Clean Code | [Principles/](./Principles/) |
| 21 | **Behavioral/** | 12+12 | STAR method, leadership, communication | [README](./Behavioral/README.md) |
| 22 | **Behavioral/Engineering/** | 12 | Behavioral prep for engineers | [Foundations](./Behavioral/Engineering/01_Foundations_Of_Behavioral_Interviews.md) |
| 23 | **InterviewPrep/** | ~10 | 30-day plan, coding patterns, system design | [InterviewPrep/](./InterviewPrep/) |
| 24 | **CompanyCodingProblems/** | 19 | Intercom interview prep problems | [README](./CompanyCodingProblems/Intercom/README.md) |
| 25 | **AI_Core_Concepts/** | 23 | ML, DL, NLP, LLMs, AI system design | [README](./AI_Core_Concepts/README.md) |
| 26 | **Engineering/** | 1 | Software engineering pillars | [Engineering](./Engineering/The_5_Pillars_of_Modern_Software_Engineering.md) |
| 27 | **Algorithms/ProblemSets/** | ~20 | Categorized LeetCode-style problems | [ProblemSets](./Algorithms/ProblemSets/) |

---

## 5. Knowledge Clusters

### Cluster A: Computer Science Foundation
```
DataStructures/ ←──────→ Algorithms/
      ↑                        ↑
      └──── javaTpoints/ ──────┘
            (Collections)
```
These are the prerequisites for everything else. Master these first.

### Cluster B: Software Design
```
Principles/
    ↓
DesignPattern/ ←──→ LowLevelDesign/
    ↓
Architectures/ ←──→ Microservices/
```
Understanding patterns before architecture makes system design intuitive.

### Cluster C: Data & Infrastructure
```
Database/ ←──→ KeyConcepts/
    ↓               ↓
BuildingBlocks/ ←──→ MessagingQ/
    ↓
Security/ ←──→ APIs/
```
Every production system touches all of these.

### Cluster D: Platform & Operations
```
DevOps/ ←──→ Cloud/
    ↓
Testing/ ←──→ Tradeoffs/
```
Operations knowledge separates senior from junior engineers.

### Cluster E: Interview Mastery
```
Behavioral/ ←──→ Behavioral/Engineering/
    ↓
InterviewPrep/ ←──→ CompanyCodingProblems/
    ↓
AI_Core_Concepts/ (differentiation)
```

---

## 6. Where to Start

Based on your background:

**🆕 New to System Design:**
→ Start at [KeyConcepts/CAPTheorem.md](./KeyConcepts/CAPTheorem.md) then follow Path B above

**🧑‍💻 Experienced Developer:**
→ Start at [Tradeoffs/](./Tradeoffs/) and [Architectures/Overview.md](./Architectures/Overview.md)

**☕ Java Developer:**
→ Start at [javaTpoints/JavaDifferentVersionInfo.md](./javaTpoints/JavaDifferentVersionInfo.md) then Path C

**📐 Preparing for a specific company:**
→ Jump to [CompanyCodingProblems/](./CompanyCodingProblems/) and [Behavioral/Engineering/](./Behavioral/Engineering/)

**🤖 AI/ML Track:**
→ Start at [AI_Core_Concepts/README.md](./AI_Core_Concepts/README.md) then Path D

---

*See also: [ROADMAPS.md](./ROADMAPS.md) | [MASTER_ENHANCEMENT_GUIDE.md](./MASTER_ENHANCEMENT_GUIDE.md) | [README.md](./README.md)*
