# 🏗️ System Design Case Studies

> *"In interviews, they don't care if your design is perfect. They care if your thinking is structured."*

---

## 🎯 What This Directory Contains

This directory contains **complete, interview-ready system design walkthroughs** for the most commonly asked problems at top tech companies. Every case study follows the **RADIO framework** — the same structured approach used by engineers at Google, Meta, Amazon, and Netflix.

**Each case study gives you:**
- ✅ Clarifying questions to ask the interviewer
- ✅ Functional + non-functional requirements with realistic numbers
- ✅ Full API design
- ✅ Data model with table schemas
- ✅ High-level + detailed architecture diagrams (ASCII)
- ✅ Bottleneck analysis and trade-off discussion
- ✅ Scaling strategies used by real companies
- ✅ Interview Q&A and common mistakes

---

## 🧭 The RADIO Framework

Every system design interview can be structured with **RADIO**:

```
R — Requirements        Clarify WHAT you're building
A — API Design          Define HOW clients interact
D — Data Model          Decide WHERE data lives and HOW it's structured
I — Infrastructure      Draw HOW the system is built
O — Optimization        Find bottlenecks and trade-offs
```

> Use this as your checklist in every interview. Never skip a step.

---

## 📚 Case Studies Index

### 🟢 Beginner (Great starting points)

| # | Design | Core Concepts | Difficulty | Time |
|---|---|---|---|---|
| 1 | [How to Approach System Design](./How_To_Approach_System_Design.md) | RADIO framework, estimation, trade-offs | 🟢 Easy | 30 min |
| 2 | [Design a URL Shortener](./DesignURLShortener.md) | Hashing, caching, redirects, DB schema | 🟢 Easy | 45 min |
| 3 | [Design a Notification System](./DesignNotificationSystem.md) | Message queues, push/pull, fan-out | 🟢 Easy | 45 min |

### 🟡 Intermediate

| # | Design | Core Concepts | Difficulty | Time |
|---|---|---|---|---|
| 4 | [Design Twitter / X](./DesignTwitter.md) | Feed generation, fan-out, eventual consistency | 🟡 Medium | 60 min |
| 5 | [Design WhatsApp](./DesignWhatsApp.md) | WebSocket, message delivery, E2E encryption | 🟡 Medium | 60 min |
| 6 | [Design a Rate Limiter](./DesignRateLimiter.md) | Token bucket, Redis, distributed systems | 🟡 Medium | 45 min |
| 7 | [Design a Key-Value Store](./DesignKeyValueStore.md) | Consistent hashing, replication, conflict resolution | 🟡 Medium | 60 min |

### 🔴 Advanced

| # | Design | Core Concepts | Difficulty | Time |
|---|---|---|---|---|
| 8 | [Design Netflix](./DesignNetflix.md) | Video streaming, CDN, encoding pipeline | 🔴 Hard | 75 min |
| 9 | [Design Uber](./DesignUber.md) | Geospatial indexing, matching, real-time tracking | 🔴 Hard | 75 min |
| 10 | [Design a Search Engine](./DesignSearchEngine.md) | Inverted index, crawling, ranking | 🔴 Hard | 90 min |
| 11 | [Design a Distributed Cache](./DesignDistributedCache.md) | Consistent hashing, eviction, hot keys | 🔴 Hard | 75 min |

> ✅ = Available | 📋 = Planned

---

## ⏱️ Back-of-Envelope Estimation Cheat Sheet

Memorize these numbers — interviewers WILL ask you to estimate.

```
Storage Units:
  1 KB = 10^3 bytes
  1 MB = 10^6 bytes
  1 GB = 10^9 bytes
  1 TB = 10^12 bytes

Time:
  1 ms = 10^-3 seconds
  1 μs = 10^-6 seconds
  1 ns = 10^-9 seconds

Operations per second (rough):
  L1 cache access:      ~0.5 ns
  L2 cache access:      ~7 ns
  RAM access:           ~100 ns
  SSD random read:      ~150 μs
  HDD random read:      ~10 ms
  Network roundtrip:    ~150 ms (CA → EU)
  Disk sequential read: ~1 GB/s
  Network bandwidth:    ~10 Gbps (datacenter)

Useful numbers:
  1 million seconds ≈ 11.5 days
  1 day = 86,400 seconds ≈ 100K seconds
  400 million tweets/day ÷ 86400 = ~5000 tweets/sec

Sizing:
  1 char = 1 byte (UTF-8 ASCII)
  1 tweet (280 chars) ≈ 300 bytes
  1 photo ≈ 200 KB
  1 minute of video (1080p) ≈ 100 MB
```

---

## 🗺️ Learning Path

```
Start Here → How_To_Approach_System_Design.md
    ↓
URL Shortener (understand hashing, caching, simple schema)
    ↓
Notification System (understand queues, fan-out)
    ↓
Twitter (understand feed generation, read-heavy scaling)
    ↓
WhatsApp (understand real-time, WebSockets, reliability)
    ↓
Netflix (understand CDN, video pipeline, encoding)
    ↓
Uber (understand geospatial, matching algorithms)
```

---

## 🔗 Related Domains

- [BuildingBlocks/](../BuildingBlocks/) — Core components used in every design
- [KeyConcepts/](../KeyConcepts/) — CAP theorem, consistency, availability
- [Database/](../Database/) — SQL, NoSQL, sharding, indexing
- [MessagingQ/](../MessagingQ/) — Kafka, message queues
- [Microservices/](../Microservices/) — Decomposition, patterns

---

*[← Back to Index](../INDEX.md)*
