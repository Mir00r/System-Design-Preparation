# 🧠 How to Approach System Design Interviews
## The RADIO Framework + 45-Minute Playbook

> *"The interviewer isn't testing if you can build a perfect system. They're testing if you can think like a senior engineer — structured, pragmatic, and trade-off aware."*

**⏱️ Estimated Read Time**: 30 minutes | **🎯 Difficulty**: 🟢 Easy | **🔗 Prerequisites**: None — read this first

---

## 📋 Table of Contents
1. [Why System Design Interviews Fail](#-why-most-candidates-fail)
2. [The RADIO Framework](#-the-radio-framework)
3. [45-Minute Interview Playbook](#️-45-minute-interview-playbook)
4. [Back-of-Envelope Estimation](#-back-of-envelope-estimation)
5. [How to Draw Architecture Diagrams](#-how-to-draw-architecture-diagrams)
6. [Trade-Off Language](#-trade-off-language)
7. [Common Mistakes](#-common-mistakes)
8. [Mini Challenge](#-mini-challenge)
9. [Interview Q&A](#-interview-qa)

---

## ❌ Why Most Candidates Fail

**Scenario**: Interviewer says "Design YouTube."

**Bad candidate**: Immediately draws boxes — Client → Load Balancer → App Server → Database. Done in 5 minutes. Says "we can scale horizontally."

**Why this fails**:
- No clarification — are we designing upload, playback, recommendations, comments?
- No estimation — how many users? How much storage? What's the read:write ratio?
- No trade-offs discussed — why SQL vs NoSQL? Why this queue vs that one?
- No bottleneck analysis — what breaks at 10M users? At 100M?

**The fix**: Use **RADIO** to structure every single answer.

---

## 📡 The RADIO Framework

```
┌─────────────────────────────────────────────────────────────┐
│                    RADIO Framework                          │
│                                                             │
│  R ─── Requirements      "What are we building?"           │
│  A ─── API Design        "How do clients interact?"         │
│  D ─── Data Model        "What does the data look like?"   │
│  I ─── Infrastructure    "How does the system work?"       │
│  O ─── Optimization      "Where does it break? Fix it."    │
└─────────────────────────────────────────────────────────────┘
```

---

### R — Requirements

**Spend 5 minutes here. Never skip this.**

Ask the interviewer clarifying questions to scope the problem. You're not expected to design everything — you're expected to focus on the most important parts.

**Functional Requirements** (what the system must DO):
```
"Should I focus on the core happy path first?"
"Are we designing the read path, write path, or both?"
"What are the top 3 features we must support?"
```

**Non-Functional Requirements** (how the system must BEHAVE):
```
Scale:        "How many DAU? How many requests/sec?"
Latency:      "What's the acceptable response time?"
Availability: "5 nines? 3 nines? Is eventual consistency OK?"
Consistency:  "Can users see slightly stale data?"
Durability:   "What's the data loss tolerance?"
```

**Example — Design Twitter:**
```
Functional:
  ✅ Post tweets (280 chars, optional media)
  ✅ Follow/unfollow users
  ✅ View home timeline (tweets from people you follow)
  ✅ Like and retweet
  ❌ (out of scope) DMs, trending, ads

Non-Functional:
  - 300M DAU
  - 500M tweets/day (write: ~6K/sec, peak ~50K/sec)
  - Read:Write ratio = 100:1 (read-heavy)
  - Timeline load < 200ms
  - Eventual consistency acceptable (you don't need to see tweets instantly)
  - 99.99% availability
```

---

### A — API Design

Design the REST (or GraphQL/gRPC) endpoints. Keep it simple.

**Pattern**:
```
HTTP_VERB /resource/{id}?params
Request body: { key: type }
Response: { key: type } or HTTP status
```

**Example — Twitter:**
```
POST   /v1/tweets               → Create tweet
GET    /v1/tweets/{tweetId}     → Get single tweet
DELETE /v1/tweets/{tweetId}     → Delete tweet
GET    /v1/users/{userId}/timeline  → Get home timeline
POST   /v1/users/{userId}/follow    → Follow user
POST   /v1/tweets/{tweetId}/like    → Like tweet
```

**Don't over-engineer** the API at this point. Use sensible REST conventions and move on.

---

### D — Data Model

Choose your storage type and define schemas.

**Decision tree**:
```
Does the data have complex relationships?   → SQL (PostgreSQL)
Is it key-value / document / flexible?     → NoSQL (MongoDB, DynamoDB)
Is it a social graph?                      → Graph DB (Neo4j)
Is it time-series data?                    → InfluxDB / Prometheus
Is it a cache / session?                   → Redis
Is it blob data (files, videos)?           → Object storage (S3)
Is it search?                              → Elasticsearch
```

**Example — Twitter:**
```sql
-- Core tables
users        (user_id PK, username, email, bio, follower_count, created_at)
tweets       (tweet_id PK, user_id FK, content, media_url, created_at, like_count)
follows      (follower_id, followee_id, created_at)  -- composite PK
likes        (user_id, tweet_id, created_at)          -- composite PK

-- Feed table (materialized, pre-computed)
feed         (user_id, tweet_id, created_at)  -- denormalized for fast reads
```

**Key question to answer**: "What are the access patterns?" — design the schema around HOW it's queried, not just how it's stored.

---

### I — Infrastructure

This is the main architecture diagram. Draw it in layers.

**Template**:
```
Client (web/mobile)
    ↓ HTTPS
DNS / CDN  (Cloudflare, CloudFront)
    ↓
Load Balancer  (L7, Nginx / ALB)
    ↓
[Service Layer]  (stateless, horizontally scalable)
    ↓         ↓         ↓
[Cache]    [Queue]   [Storage]
(Redis)   (Kafka)   (PostgreSQL / S3)
```

Start with the **happy path** — what happens when a user makes a request? Walk the interviewer through each component and WHY it's there.

---

### O — Optimization

This is where senior candidates shine. Identify bottlenecks and propose solutions.

**Common bottlenecks and fixes**:

| Bottleneck | Symptoms | Solutions |
|---|---|---|
| Database read overload | High latency, connection errors | Read replicas, caching layer |
| Database write bottleneck | Write queue building up | Sharding, CQRS, async writes |
| Single service hotspot | One service overwhelmed | Horizontal scaling, rate limiting |
| Cache miss storm | DB spike on cold start | Cache warm-up, consistent hashing |
| Large file uploads | Timeout, memory pressure | Presigned URLs, chunked upload |
| Celebrity problem | One user has 100M followers | Hybrid fan-out, lazy loading |

---

## ⏰ 45-Minute Interview Playbook

```
Minute  0-5:   Requirements (R)
               → Ask 3-5 clarifying questions
               → Write down functional + non-functional requirements
               → Estimate scale (DAU, QPS, storage)

Minute  5-10:  API Design (A)
               → Define 3-5 core endpoints
               → Request/response format

Minute 10-15:  Data Model (D)
               → Choose storage type(s) and justify
               → Draw key table schemas or document structure

Minute 15-35:  Infrastructure (I)
               → High-level diagram first (5 min)
               → Dive deep into most critical component (15 min)

Minute 35-45:  Optimization (O)
               → Identify 2-3 bottlenecks proactively
               → Propose and evaluate solutions
               → Discuss trade-offs
```

> **Tip**: Talk while you draw. Never go silent for more than 30 seconds. The interviewer wants to hear your thought process, not just your conclusion.

---

## 🔢 Back-of-Envelope Estimation

Always estimate before designing. These numbers show you can think at scale.

### The Magic Formula
```
QPS (queries/sec) = DAU × actions_per_day ÷ 86,400
Storage/day       = writes_per_day × data_size_per_write
Bandwidth         = QPS × response_size
```

### Example — Twitter Scale
```
Given: 300M DAU, 5 tweets/user/day reads, 0.1 tweets/user/day writes

Write QPS = 300M × 0.1 / 86400 ≈ 350 writes/sec
Read QPS  = 300M × 5   / 86400 ≈ 17,000 reads/sec
Peak QPS  = ~10x average → 170,000 reads/sec

Storage:
  1 tweet = 300 bytes text + avg 10% have 200KB photo
  Text storage = 300M × 0.1 × 300 bytes/day = ~9 GB/day
  Photo storage = 300M × 0.01 × 200KB/day = ~600 GB/day
  5 years = ~1 PB total
```

---

## 🎨 How to Draw Architecture Diagrams

On a whiteboard or in a text interview, use boxes and arrows:

```
Level 1 — High-Level (always start here):

  [Client]──→[CDN]──→[Load Balancer]──→[App Server]──→[Database]
                                             ↓
                                           [Cache]

Level 2 — Service decomposition:

  [App Server] = [API Gateway] → [Tweet Service]
                               → [User Service]
                               → [Feed Service]

Level 3 — Data flow (for the critical path):

  POST /tweet
    → API Gateway (auth + rate limit)
    → Tweet Service (validate + persist)
    → PostgreSQL (write tweet)
    → Kafka (publish tweet_created event)
    → Fan-out Service (push to follower feeds)
    → Redis (update feed cache)
```

---

## 🗣️ Trade-Off Language

Use these phrases to show senior-engineer thinking:

| Situation | What to say |
|---|---|
| Choosing SQL vs NoSQL | "I'd choose PostgreSQL here because we need ACID guarantees and the data is relational. We can always migrate to Cassandra later if we need horizontal writes at massive scale." |
| Consistency vs Availability | "For a social media feed, I'd accept eventual consistency — users don't need to see tweets in perfect real-time. This lets us use a simpler replication model and stay available during network partitions." |
| Caching strategy | "I'd use a write-through cache here so reads are always fast. The trade-off is slightly higher write latency, but for a read-heavy system that's acceptable." |
| Synchronous vs async | "I'd make this fan-out asynchronous via Kafka — the tweet is durably stored immediately, and the feed update happens eventually. This keeps the write path fast and decouples the two concerns." |

---

## ⚠️ Common Mistakes

1. **Jumping to solutions** — Always ask clarifying questions for the first 5 minutes. Never start drawing immediately.

2. **Ignoring non-functional requirements** — "Design Twitter" for a startup vs 300M DAU requires completely different architectures. Always establish scale first.

3. **Over-engineering** — Adding microservices, sharding, and multi-region replication for a system with 1,000 users. Start simple, evolve to complex.

4. **Not discussing trade-offs** — Every choice has pros and cons. "I chose Redis over Memcached because of data persistence and richer data structures, though Memcached can be simpler for pure caching."

5. **Forgetting failure modes** — What happens when the database goes down? When the cache is cold? When a celebrity user has 100M followers and posts a tweet? Senior engineers think about failure proactively.

---

## 🧩 Mini Challenge

**You have 1 minute**: A user asks you to "Design Instagram." What are your first 3 clarifying questions?

<details>
<summary>💡 Click to reveal answer</summary>

**Great clarifying questions**:

1. **Scope**: "Should I focus on the photo upload flow, the feed generation, or both? What about Stories, Reels, DMs?"

2. **Scale**: "How many DAU are we targeting? How many photo uploads per day? What's the read:write ratio?"

3. **Constraints**: "What's the acceptable latency for feed loading? Is eventual consistency OK — can users see posts a few seconds after posting? What's our uptime requirement?"

**Why these are good**: They establish scope (you can't design everything in 45 min), they establish scale (1K users vs 1B users = completely different systems), and they surface hidden constraints that change the entire architecture.

</details>

---

## 📝 Interview Q&A

**Q: How do you handle the "celebrity problem" — a user with 100M followers posts a tweet?**

> A: For most users (< 10K followers), fan-out on write works — the tweet is pushed to all follower feeds synchronously or async via queue. For celebrities, fan-out on read is used — the feed is assembled at read time by checking if the user follows that celebrity and fetching their recent tweets. Twitter uses a hybrid approach called "lazy fan-out with celebrity detection."

**Q: When would you choose eventual consistency over strong consistency?**

> A: Eventual consistency is acceptable when slight data staleness doesn't harm user experience or business logic — social media feeds, product recommendations, analytics dashboards, non-critical notifications. Strong consistency is required for financial transactions, inventory counts, access control, and anything where two systems must agree on a value simultaneously.

**Q: How do you estimate storage requirements?**

> A: `(writes/day) × (bytes/write) × (retention years) × (replication factor)`. Always add 20% overhead for indexes and metadata.

**Q: What's the difference between horizontal and vertical scaling?**

> A: Vertical (scale-up): bigger machine, more RAM/CPU. Simple, no code changes, but has a hard limit and single point of failure. Horizontal (scale-out): more machines, stateless services, load balancer in front. Infinite scaling potential, but requires stateless design, consistent hashing for distributed data.

**Q: How do you handle database bottlenecks?**

> A: Read bottleneck → add read replicas + cache. Write bottleneck → shard by user_id or another natural partition key, or use CQRS to separate read/write models. Both → consider DynamoDB or Cassandra with built-in horizontal scaling.

---

## 🔗 What to Read Next

1. **[Design a URL Shortener](./DesignURLShortener.md)** — Perfect first case study: simple enough to finish in 30 min, covers hashing, caching, and scaling
2. **[BuildingBlocks/](../BuildingBlocks/)** — Master the components before combining them
3. **[KeyConcepts/CAP Theorem](../KeyConcepts/CAPTheorem.md)** — The fundamental trade-off in all distributed systems

---

*[← Back to Case Studies](./README.md) | [Next: Design URL Shortener →](./DesignURLShortener.md)*
