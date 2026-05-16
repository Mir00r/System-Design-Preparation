# 🧭 Database Selection Guide: Choosing the Right Database for Every Use Case

> *"There is no 'best database.' There's only the best database for YOUR specific access patterns, consistency requirements, scale targets, and team expertise. This guide gives you a decision framework used by FAANG architects."*

**⏱️ Estimated Time**: 25 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [SQL vs NoSQL](./SQL_Vs_NoSQL.md), [CAP Theorem](../KeyConcepts/CAPTheorem.md)

---

## 📋 Table of Contents
1. [Decision Framework](#-decision-framework)
2. [Database Categories](#-database-categories)
3. [Selection Matrix](#-selection-matrix)
4. [Use Case Mapping](#-use-case-mapping)
5. [FAANG Database Choices](#-faang-database-choices)
6. [Decision Flowchart](#-decision-flowchart)
7. [Anti-Patterns](#-anti-patterns)
8. [Mini Challenge](#-mini-challenge)
9. [Interview Q&A](#-interview-qa)

---

## 🎯 Decision Framework

```
ASK THESE QUESTIONS (in order):

1. DATA MODEL: What shape is your data?
   → Tabular with relationships? → Relational (PostgreSQL, MySQL)
   → Nested documents with varying schema? → Document (MongoDB)
   → Simple key-value lookups? → Key-Value (Redis, DynamoDB)
   → Highly connected (graph traversals)? → Graph (Neo4j)
   → Time-stamped metrics? → TimeSeries (InfluxDB, TimescaleDB)
   → Full-text search? → Search (Elasticsearch)

2. ACCESS PATTERNS: How do you read/write?
   → Complex JOINs, aggregations? → Relational
   → Point lookups by ID? → Key-Value / Document
   → Range scans on time? → TimeSeries
   → Full-text with relevance? → Search engine
   → Graph traversals (friends-of-friends)? → Graph DB

3. CONSISTENCY REQUIREMENTS:
   → Strong ACID (financial)? → PostgreSQL, MySQL, CockroachDB
   → Eventual consistency OK? → Cassandra, DynamoDB, MongoDB
   → Tunable per-query? → Cassandra, MongoDB

4. SCALE REQUIREMENTS:
   → Single server sufficient (< 1TB)? → PostgreSQL (simplest)
   → Read-heavy scaling? → Read replicas, Redis cache
   → Write-heavy scaling? → Cassandra, DynamoDB, Kafka
   → Both reads and writes at scale? → DynamoDB, CockroachDB, Vitess

5. OPERATIONAL COMPLEXITY BUDGET:
   → Minimal ops team? → Managed services (RDS, DynamoDB, Atlas)
   → Self-hosted expertise? → PostgreSQL, Cassandra, Elasticsearch
```

---

## 📂 Database Categories

```
┌─────────────────────────────────────────────────────────────────┐
│ CATEGORY        │ EXAMPLES            │ STRENGTH                 │
├─────────────────┼─────────────────────┼──────────────────────────┤
│ Relational      │ PostgreSQL, MySQL   │ ACID, JOINs, SQL         │
│                 │ CockroachDB         │ ecosystem, consistency   │
├─────────────────┼─────────────────────┼──────────────────────────┤
│ Document        │ MongoDB, Couchbase  │ Flexible schema, nested  │
│                 │ Firestore           │ data, developer velocity │
├─────────────────┼─────────────────────┼──────────────────────────┤
│ Key-Value       │ Redis, DynamoDB     │ Lowest latency, simple   │
│                 │ Memcached           │ model, extreme throughput│
├─────────────────┼─────────────────────┼──────────────────────────┤
│ Wide-Column     │ Cassandra, HBase    │ Write throughput, linear │
│                 │ ScyllaDB            │ scale, time-series       │
├─────────────────┼─────────────────────┼──────────────────────────┤
│ Search          │ Elasticsearch       │ Full-text, relevance     │
│                 │ OpenSearch, Solr    │ ranking, aggregations    │
├─────────────────┼─────────────────────┼──────────────────────────┤
│ Graph           │ Neo4j, Amazon       │ Relationship traversal,  │
│                 │ Neptune, Dgraph     │ pattern matching         │
├─────────────────┼─────────────────────┼──────────────────────────┤
│ TimeSeries      │ InfluxDB,           │ Time-range queries,      │
│                 │ TimescaleDB,        │ compression, retention   │
│                 │ Prometheus          │ policies, downsampling   │
├─────────────────┼─────────────────────┼──────────────────────────┤
│ Message/Stream  │ Kafka, Pulsar       │ Event streaming, replay, │
│                 │ Redis Streams       │ pub/sub, ordering        │
└─────────────────┴─────────────────────┴──────────────────────────┘
```

---

## 📊 Selection Matrix

| Requirement | PostgreSQL | MongoDB | Redis | Cassandra | Elasticsearch | DynamoDB |
|---|---|---|---|---|---|---|
| **ACID Transactions** | ✅ Full | ✅ Single-doc (multi since 4.0) | ❌ Atomic ops only | ❌ No | ❌ No | ✅ Per-item |
| **Complex Queries/JOINs** | ✅ Excellent | 🟡 $lookup (limited) | ❌ No | ❌ No JOINs | 🟡 Aggregations | ❌ No |
| **Schema Flexibility** | ❌ Strict (JSONB helps) | ✅ Schema-less | 🟡 Structure-aware | ❌ Rigid tables | ✅ Dynamic mapping | 🟡 Flexible |
| **Write Throughput** | 🟡 10-50K/s | 🟡 50-100K/s | ✅ 1M+/s | ✅ 500K+/s | 🟡 50-100K/s | ✅ 1M+/s |
| **Read Latency** | 🟡 1-10ms | 🟡 1-10ms | ✅ <1ms | 🟡 2-10ms | 🟡 5-50ms | ✅ <5ms |
| **Horizontal Scale** | 🟡 Read replicas | ✅ Auto-sharding | 🟡 Cluster | ✅ Linear | ✅ Sharding | ✅ Automatic |
| **Full-Text Search** | 🟡 tsvector (basic) | 🟡 Text indexes | ❌ No | ❌ No | ✅ Excellent | ❌ No |
| **Operational Complexity** | 🟢 Low | 🟡 Medium | 🟢 Low | 🔴 High | 🔴 High | 🟢 Low (managed) |
| **Cost** | 🟢 Open source | 🟢 Open source | 🟢 Open source | 🟢 Open source | 🟡 License change | 🟡 Pay-per-use |

---

## 🏢 Use Case Mapping

```
E-COMMERCE PLATFORM:
  ├── Product catalog → MongoDB (flexible attributes per category)
  ├── Orders & payments → PostgreSQL (ACID, complex queries)
  ├── Product search → Elasticsearch (full-text, facets, relevance)
  ├── Shopping cart → Redis (session, fast reads, auto-expiry)
  ├── Recommendations → Neo4j or Redis (collaborative filtering)
  └── Analytics → ClickHouse (OLAP, aggregations)

SOCIAL MEDIA (Twitter/Instagram):
  ├── User profiles → PostgreSQL (relational, ACID)
  ├── Posts/tweets → Cassandra (write-heavy, timeline reads)
  ├── Timeline feed → Redis (pre-computed, fast reads)
  ├── Search → Elasticsearch (full-text, hashtags, trends)
  ├── Social graph → Neo4j (friends, followers, recommendations)
  ├── Notifications → Redis Streams (real-time, ordered)
  └── Metrics → InfluxDB (engagement, performance monitoring)

RIDE-SHARING (Uber/Lyft):
  ├── User/driver profiles → PostgreSQL (ACID, relational)
  ├── Real-time location → Redis Geo (geospatial queries, <1ms)
  ├── Trip history → Cassandra (write-heavy, time-series-like)
  ├── Payment → PostgreSQL (ACID, audit trail)
  ├── ETA calculation → Redis + precomputed (low latency)
  ├── Surge pricing → Redis (real-time counters per zone)
  └── Trip events → Kafka → InfluxDB (streaming analytics)

FINTECH (Banking/Trading):
  ├── Accounts & ledger → PostgreSQL (strict ACID, double-entry)
  ├── Transaction history → PostgreSQL with partitioning
  ├── Session/auth → Redis (fast token validation)
  ├── Fraud detection → Elasticsearch (pattern matching, real-time)
  ├── Market data → TimescaleDB (tick data, time-range queries)
  └── Audit log → Cassandra (immutable, append-only, never delete)
```

---

## 🏢 FAANG Database Choices

```
GOOGLE:
  Spanner         → global ACID transactions (strongest consistency + global scale)
  Bigtable        → wide-column for analytics, Gmail indexing
  Firestore       → document DB for mobile/web apps
  AlloyDB         → managed PostgreSQL-compatible

AMAZON:
  DynamoDB        → key-value / document at any scale (single-digit ms)
  Aurora          → PostgreSQL/MySQL compatible (5x faster)
  Redshift        → OLAP data warehouse
  ElastiCache     → managed Redis/Memcached
  Neptune         → graph database
  Timestream      → managed time-series

META (Facebook):
  MySQL (heavily modified) → social graph, messages
  TAO                      → graph-aware caching layer over MySQL
  RocksDB                  → embedded KV store (basis for many systems)
  Cassandra                → Instagram messages (modified)

NETFLIX:
  Cassandra       → user activity, viewing history (billions of events)
  EVCache (Redis) → session, recommendations, homepage
  CockroachDB     → billing, account management (global ACID)
  Elasticsearch   → title search, content discovery

UBER:
  MySQL (Schemaless)  → trip data, custom sharding layer
  Cassandra           → driver location history
  Redis               → geospatial (nearby drivers), caching
  Elasticsearch       → search, logs
  Apache Pinot        → real-time analytics (Uber's internal OLAP)
```

---

## 🔀 Decision Flowchart

```
                    START
                      │
                      ▼
          ┌─── Need ACID transactions? ───┐
          │ YES                            │ NO
          ▼                                ▼
    Need global scale?            Need sub-millisecond latency?
    │ YES        │ NO             │ YES           │ NO
    ▼            ▼                ▼               ▼
  CockroachDB  PostgreSQL       Redis          What's your data shape?
  / Spanner                                    │
                                    ┌──────────┼──────────┐
                                    ▼          ▼          ▼
                              Documents?    Time-based?  Graph?
                                    │          │          │
                                    ▼          ▼          ▼
                                 MongoDB    InfluxDB/   Neo4j
                                           TimescaleDB
                                    │
                              Write-heavy, linear scale needed?
                              │ YES              │ NO
                              ▼                  ▼
                           Cassandra        Need full-text search?
                                            │ YES        │ NO
                                            ▼            ▼
                                      Elasticsearch   MongoDB or
                                                      PostgreSQL
```

---

## ❌ Anti-Patterns

1. **"Use MongoDB for everything"** — Document databases struggle with highly relational data requiring complex JOINs. A social network with friends-of-friends queries needs a graph DB or relational DB, not `$lookup` pipelines in MongoDB.

2. **"We need Cassandra for 10GB of data"** — Cassandra's operational complexity (ring management, repair, compaction tuning) only pays off at hundreds of GBs to TBs. For small datasets, PostgreSQL is simpler and faster.

3. **"Redis is our primary database"** — Without persistence configuration, Redis loses all data on restart. Even with AOF, it's limited by RAM size. Use Redis as cache/session store, not as source of truth for critical business data.

4. **"We'll use Elasticsearch for everything"** — ES is not designed for strong consistency, ACID transactions, or primary data storage. Always keep a source of truth (PostgreSQL, MongoDB) and sync to ES for search functionality.

5. **"One database to rule them all"** — Polyglot persistence (multiple databases for different needs) is standard practice at scale. The key is keeping the architecture manageable — don't use 10 databases when 3 would suffice.

---

## 🧩 Mini Challenge

**You're designing a food delivery app (like DoorDash). Identify which database technology you'd use for each component: user accounts, restaurant menus, real-time driver locations, order tracking, search, and analytics.**

<details>
<summary>💡 Click to reveal answer</summary>

```
FOOD DELIVERY APP — DATABASE ARCHITECTURE:

┌─────────────────────────────────────────────────────────────┐
│ Component              │ Database        │ Reasoning          │
├────────────────────────┼─────────────────┼────────────────────┤
│ User accounts/auth     │ PostgreSQL      │ ACID, relational   │
│                        │                 │ (user↔payment↔addr)│
├────────────────────────┼─────────────────┼────────────────────┤
│ Restaurant menus       │ MongoDB         │ Flexible schema    │
│                        │                 │ (pizza vs sushi    │
│                        │                 │ have diff fields)  │
├────────────────────────┼─────────────────┼────────────────────┤
│ Real-time driver       │ Redis (Geo)     │ <1ms geospatial    │
│ locations              │                 │ queries, GEORADIUS │
│                        │                 │ for nearest driver │
├────────────────────────┼─────────────────┼────────────────────┤
│ Order state machine    │ PostgreSQL      │ ACID transitions   │
│ (placed→confirmed→    │                 │ (payment + status  │
│  preparing→delivered)  │                 │ must be atomic)    │
├────────────────────────┼─────────────────┼────────────────────┤
│ Order history          │ Cassandra       │ Write-heavy, time- │
│ (completed orders)     │                 │ sorted, append-only│
│                        │                 │ (millions/day)     │
├────────────────────────┼─────────────────┼────────────────────┤
│ Restaurant/food search │ Elasticsearch   │ Full-text, geo     │
│                        │                 │ filtering, facets  │
│                        │                 │ (cuisine, price,   │
│                        │                 │ rating, distance)  │
├────────────────────────┼─────────────────┼────────────────────┤
│ Real-time analytics    │ ClickHouse or   │ Fast aggregations  │
│ (dashboards, reports)  │ Apache Pinot    │ on millions of     │
│                        │                 │ orders/events      │
├────────────────────────┼─────────────────┼────────────────────┤
│ Session / cart         │ Redis           │ Fast, ephemeral,   │
│                        │                 │ auto-expiry (TTL)  │
└─────────────────────────────────────────────────────────────┘

Data flow:
  Order placed → PostgreSQL (ACID) → Kafka event → Cassandra (history)
                                                  → Elasticsearch (analytics)
                                                  → Push notification (Redis pub/sub)
```

</details>

---

## 📝 Interview Q&A

**Q: You're building a new social media app. Walk me through your database selection process.**
> A: (1) **User profiles + relationships**: PostgreSQL — relational data (user follows user), ACID for account operations, proven at scale with read replicas. (2) **Posts/feed**: For write path, Cassandra — time-sorted writes, eventual consistency is OK. For read path, pre-computed feeds in Redis (fan-out-on-write for non-celebrities). (3) **Search** (users, hashtags, content): Elasticsearch with async sync from primary stores. (4) **Real-time features** (online status, typing indicators): Redis pub/sub or Redis Streams. (5) **Media metadata**: MongoDB — flexible schema for different media types (photo/video/story have different attributes). (6) **Analytics**: ClickHouse for engagement metrics and dashboards. Key principle: start with PostgreSQL for most things, add specialized databases only when PostgreSQL becomes a bottleneck for specific access patterns.

**Q: What's polyglot persistence and what are its trade-offs?**
> A: Polyglot persistence means using different database technologies for different parts of your application based on each component's specific needs. **Benefits**: each database excels at its intended workload (Redis for cache speed, ES for search, PostgreSQL for transactions). **Trade-offs**: (1) Operational complexity — your team must master multiple technologies; (2) Data consistency — keeping data in sync across databases requires event-driven patterns (CDC, dual-writes, saga); (3) Cross-cutting queries are hard — can't JOIN across databases easily; (4) More infrastructure to monitor and maintain. **Rule of thumb**: Start with 1-2 databases. Add a specialized database only when you have a clear performance or functional gap that can't be solved by your existing stack.

---

## 🔗 What to Read Next

1. **[Database/Redis_Deep_Dive.md](./Redis_Deep_Dive.md)** — Deep dive into Redis
2. **[Database/MongoDB_Deep_Dive.md](./MongoDB_Deep_Dive.md)** — Deep dive into MongoDB
3. **[Database/Cassandra_Deep_Dive.md](./Cassandra_Deep_Dive.md)** — Deep dive into Cassandra
4. **[KeyConcepts/CAPTheorem.md](../KeyConcepts/CAPTheorem.md)** — Consistency vs availability trade-offs

---

*[← TimeSeries Databases](./TimeSeries_Databases.md) | [Back to Index](../INDEX.md)*
