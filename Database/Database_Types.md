# 📊 Database Types & Selection Guide

> *"Choosing the wrong database is like choosing the wrong foundation for a building — you won't notice until it's too late, and by then, migration will cost you millions. A senior engineer's superpower isn't knowing one database deeply — it's knowing WHEN to use which type. This guide covers all 15 major database types and when each one shines."*

**⏱️ Estimated Time**: 35 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [SQL vs NoSQL](../../Database/SQL_Vs_NoSQL.md), [ACID](../../Database/ACID.md)

---

## 📋 Table of Contents
1. [The Database Landscape](#-the-database-landscape)
2. [Relational Databases (SQL)](#-relational-databases-sql)
3. [Document Databases](#-document-databases)
4. [Key-Value Stores](#-key-value-stores)
5. [Wide-Column Stores](#-wide-column-stores)
6. [Graph Databases](#-graph-databases)
7. [Time-Series Databases](#-time-series-databases)
8. [Search Engines](#-search-engines)
9. [In-Memory Databases](#-in-memory-databases)
10. [Vector Databases](#-vector-databases)
11. [Object Databases](#-object-databases)
12. [NewSQL Databases](#-newsql-databases)
13. [Decision Framework](#-decision-framework)
14. [Interview Q&A](#-interview-qa)

---

## 🗺️ The Database Landscape

```
┌──────────────────────────────────────────────────────────────────┐
│                    DATABASE TYPE DECISION TREE                     │
│                                                                    │
│  What's your PRIMARY access pattern?                              │
│                                                                    │
│  ├── Complex joins + transactions? → RELATIONAL (PostgreSQL)      │
│  ├── Flexible schema + nested docs? → DOCUMENT (MongoDB)         │
│  ├── Simple key→value lookups? → KEY-VALUE (Redis, DynamoDB)     │
│  ├── Massive write throughput + wide rows? → WIDE-COLUMN         │
│  ├── Relationship traversals? → GRAPH (Neo4j)                    │
│  ├── Time-ordered metrics? → TIME-SERIES (InfluxDB)              │
│  ├── Full-text search + fuzzy matching? → SEARCH (Elasticsearch) │
│  ├── Ultra-low latency + caching? → IN-MEMORY (Redis)            │
│  ├── AI/ML embeddings + similarity? → VECTOR (Pinecone)          │
│  └── SQL semantics + horizontal scale? → NEWSQL (CockroachDB)    │
│                                                                    │
│  REALITY: Most systems use 2-4 different databases!              │
│  "Polyglot persistence" — right tool for each job!               │
└──────────────────────────────────────────────────────────────────┘
```

---

## 📊 Relational Databases (SQL)

```
EXAMPLES: PostgreSQL, MySQL, Oracle, SQL Server

WHEN TO USE:
  ✅ Structured data with clear relationships
  ✅ ACID transactions required (banking, orders!)
  ✅ Complex queries with JOINs
  ✅ Data integrity is paramount
  ✅ Reporting & analytics
  ✅ < 10 TB data (beyond → consider sharding/NewSQL)

WHEN NOT TO USE:
  ❌ Highly variable/nested schema (use Document DB)
  ❌ Simple key-value access at massive scale (use KV store)
  ❌ Need horizontal scaling to 100+ nodes (use wide-column)
  ❌ Graph traversals (use Graph DB)

EXAMPLE USE CASES:
  • E-commerce: orders, products, customers
  • Banking: accounts, transactions (ACID critical!)
  • SaaS platforms: users, subscriptions, billing
  • CMS: articles, categories, tags

PostgreSQL vs MySQL:
  PostgreSQL: more features (JSONB, full-text, extensions)
  MySQL: simpler, more widely deployed, faster for reads
  For new projects: PostgreSQL (industry moving toward it)
```

---

## 📄 Document Databases

```
EXAMPLES: MongoDB, CouchDB, Amazon DocumentDB, Firestore

DATA MODEL: JSON/BSON documents (nested, flexible schema!)
  {
    "user_id": "123",
    "name": "John",
    "addresses": [         ← Nested array!
      {"city": "NYC", "zip": "10001"},
      {"city": "LA", "zip": "90001"}
    ],
    "preferences": {       ← Nested object!
      "theme": "dark",
      "notifications": true
    }
  }

WHEN TO USE:
  ✅ Schema evolves frequently (startups, MVPs!)
  ✅ Nested/hierarchical data (profiles, content)
  ✅ Read-heavy with denormalized data
  ✅ Need to scale horizontally (built-in sharding!)
  ✅ Each record has different fields

WHEN NOT TO USE:
  ❌ Heavy cross-document transactions (no JOINs!)
  ❌ Complex relational queries
  ❌ Need strict schema enforcement upfront
  ❌ Many-to-many relationships (use relational or graph)

EXAMPLE USE CASES:
  • User profiles (each user has different fields!)
  • Content management (articles, blog posts, metadata)
  • Product catalogs (shoes have size, electronics have specs)
  • Mobile app backends (flexible schema for rapid iteration)
```

---

## 🔑 Key-Value Stores

```
EXAMPLES: Redis, Amazon DynamoDB, etcd, Riak

DATA MODEL: Simple key → value mapping
  "session:abc123" → "{user_id: 456, expires: ...}"
  "cache:product:789" → "{name: ..., price: 99.99}"

WHEN TO USE:
  ✅ Simple access pattern: get/put by key
  ✅ Ultra-low latency required (< 1ms!)
  ✅ Caching layer
  ✅ Session storage
  ✅ Real-time leaderboards / counters
  ✅ Distributed locks

WHEN NOT TO USE:
  ❌ Need to query by value (no secondary indexes!)
  ❌ Complex relationships between entities
  ❌ Need to scan/filter across values
  ❌ Transactions across multiple keys (depends on implementation)

SUBTYPES:
  In-memory (Redis): ultra-fast, limited by RAM
  On-disk (RocksDB): larger capacity, still fast
  Distributed (DynamoDB): massive scale, tunable consistency
  Coordination (etcd, ZooKeeper): config, service discovery
```

---

## 📋 Wide-Column Stores

```
EXAMPLES: Apache Cassandra, HBase, Google Bigtable, ScyllaDB

DATA MODEL: 2D key-value (row key + column families)
  Row: "user:123"
    Column family "profile": {name: "John", age: 30}
    Column family "activity": {last_login: "...", posts: 42}
    Column family "metrics": {views: 1000, likes: 500}
  
  Each row can have DIFFERENT columns! (sparse!)

WHEN TO USE:
  ✅ Massive write throughput (millions/second!)
  ✅ Time-series-like patterns with wide rows
  ✅ Data naturally partitions by row key
  ✅ Eventual consistency is acceptable
  ✅ 100+ TB of data, multi-region
  ✅ High availability across data centers

WHEN NOT TO USE:
  ❌ Need JOINs or complex queries
  ❌ Strong consistency required (use relational!)
  ❌ Small dataset (< 10 TB — overkill!)
  ❌ Frequently changing access patterns (schema design is critical!)

EXAMPLE USE CASES:
  • IoT sensor data (millions of devices writing constantly)
  • User activity feeds (write-heavy, append-heavy)
  • Messaging: message history (partition by conversation_id)
  • Analytics: event tracking at massive scale
```

---

## 🕸️ Graph Databases

```
EXAMPLES: Neo4j, Amazon Neptune, JanusGraph, TigerGraph

DATA MODEL: Nodes + Edges (relationships are first-class!)
  (John)-[:FRIENDS_WITH]->(Jane)
  (Jane)-[:WORKS_AT]->(Google)
  (John)-[:INTERESTED_IN]->(SystemDesign)

WHEN TO USE:
  ✅ Relationship-heavy queries ("friends of friends who like X")
  ✅ Social networks, recommendation engines
  ✅ Fraud detection (suspicious connection patterns!)
  ✅ Knowledge graphs (entities + relationships)
  ✅ Path finding (shortest path, network analysis)
  ✅ Access control (role → permission → resource graphs)

WHEN NOT TO USE:
  ❌ Simple CRUD without relationship traversals
  ❌ Aggregation/analytics (better in columnar DBs!)
  ❌ High write throughput (graph updates are complex!)
  ❌ Data that doesn't have natural relationships

EXAMPLE QUERIES:
  "Find mutual friends": 2 hops → trivial in graph, brutal in SQL!
  SQL: 5-way JOIN (painful, slow!)
  Cypher: MATCH (a)-[:FRIENDS]-(mutual)-[:FRIENDS]-(b) RETURN mutual

EXAMPLE USE CASES:
  • Social network: LinkedIn connections, Facebook friends
  • Fraud: find circular payment patterns
  • Recommendations: "users who bought X also bought Y"
  • Identity resolution: link same person across systems
```

---

## ⏱️ Time-Series Databases

```
EXAMPLES: InfluxDB, TimescaleDB, Prometheus, QuestDB

DATA MODEL: Timestamp → metrics (optimized for time-ordered inserts!)
  2024-01-15T10:00:00 | server=web01 | cpu=85.2 | mem=72.1
  2024-01-15T10:00:01 | server=web01 | cpu=83.7 | mem=72.0
  2024-01-15T10:00:02 | server=web01 | cpu=87.1 | mem=72.3

WHEN TO USE:
  ✅ Metrics & monitoring (CPU, memory, request counts)
  ✅ IoT sensor data (temperature, pressure, location)
  ✅ Financial tick data (stock prices every millisecond!)
  ✅ Data with natural time ordering
  ✅ Downsampling (aggregate old data: 1s → 1min → 1hour)
  ✅ Retention policies (auto-delete data older than 30 days)

WHEN NOT TO USE:
  ❌ General-purpose CRUD (use relational!)
  ❌ Non-time-ordered data
  ❌ Complex relationships between data points
  ❌ Need random updates to historical data

OPTIMIZATIONS (why general DBs are worse):
  • Columnar storage: compress time series amazingly well!
  • Append-only: writes are always at "now" → sequential I/O!
  • Automatic rollup: minute data → hourly → daily
  • Specialized queries: "average CPU last 24 hours, grouped by host"
```

---

## 🔍 Search Engines

```
EXAMPLES: Elasticsearch, Apache Solr, Meilisearch, Typesense

DATA MODEL: Inverted index (word → documents containing it)
  "distributed" → [doc1, doc3, doc7, doc42]
  "system"      → [doc1, doc2, doc7, doc15]
  "design"      → [doc1, doc7, doc22]
  
  Search "distributed system design" → intersection → doc1, doc7!

WHEN TO USE:
  ✅ Full-text search (fuzzy matching, relevance ranking!)
  ✅ Autocomplete / typeahead suggestions
  ✅ Log aggregation & analysis (ELK stack!)
  ✅ Faceted search (filter by category, price range, etc.)
  ✅ Geospatial search with text
  ✅ Complex filtering + sorting + pagination

WHEN NOT TO USE:
  ❌ Source of truth for data (it's an INDEX, not primary storage!)
  ❌ Transactions / consistency guarantees
  ❌ Simple key-value lookups (overkill!)
  ❌ Write-heavy without search needs
```

---

## 💨 In-Memory Databases

```
EXAMPLES: Redis, Memcached, VoltDB, Apache Ignite

WHEN TO USE:
  ✅ Sub-millisecond latency required
  ✅ Caching (most common use!)
  ✅ Session management
  ✅ Real-time analytics (counters, leaderboards)
  ✅ Pub/Sub messaging
  ✅ Rate limiting, distributed locks

KEY INSIGHT: RAM is 1000x faster than SSD!
  RAM access: ~100 nanoseconds
  SSD access: ~100 microseconds
  HDD access: ~10 milliseconds
  
  For hot data accessed millions of times/second → in-memory wins!
```

---

## 🧠 Vector Databases

```
EXAMPLES: Pinecone, Weaviate, Milvus, Qdrant, pgvector

DATA MODEL: High-dimensional vectors + similarity search
  "What is system design?" → [0.23, -0.45, 0.78, ..., 0.12] (1536 dims)
  
  Query: "find similar vectors" → cosine similarity / ANN search

WHEN TO USE:
  ✅ AI/ML applications (LLM embeddings!)
  ✅ Semantic search ("find similar meaning, not exact words")
  ✅ Recommendation engines (similar users/products)
  ✅ Image/audio similarity search
  ✅ RAG (Retrieval Augmented Generation) for LLMs

WHEN NOT TO USE:
  ❌ Exact match queries (use traditional DB!)
  ❌ Structured data with clear schema
  ❌ Non-ML workloads
  ❌ Small datasets (< 100K vectors — just use brute force!)

THE RISE OF VECTORS:
  With ChatGPT/LLMs: vector DBs became critical!
  Every AI app needs: embed text → store vectors → similarity search
  This is a NEW database category (2021+)!
```

---

## 🆕 NewSQL Databases

```
EXAMPLES: CockroachDB, Google Spanner, TiDB, YugabyteDB

THE PROMISE: SQL semantics + horizontal scaling!
  "Best of both worlds: relational features + NoSQL scale"

WHEN TO USE:
  ✅ Need ACID transactions AND horizontal scaling
  ✅ Global distribution with strong consistency
  ✅ Migrating from PostgreSQL but need more scale
  ✅ Can't sacrifice SQL features for scale
  ✅ Multi-region with low-latency reads globally

WHEN NOT TO USE:
  ❌ Simple workloads that don't need distribution
  ❌ Cost-sensitive (more expensive than single PostgreSQL!)
  ❌ Don't need global distribution
  ❌ Very high write throughput (wide-column still wins here)

GOOGLE SPANNER: True global consistency!
  Uses GPS + atomic clocks for clock synchronization!
  Transactions span continents with strong consistency!
  (Only Google has the hardware for this approach...)
```

---

## 🎯 Decision Framework

```
CHOOSING THE RIGHT DATABASE:

  STEP 1: What's your data model?
    Structured tables → Relational
    Flexible JSON docs → Document
    Nodes & edges → Graph
    Time-ordered → Time-Series
    Vectors → Vector DB
    
  STEP 2: What's your access pattern?
    By primary key → Key-Value
    By many attributes → Relational or Search
    By time range → Time-Series
    By similarity → Vector DB
    By relationship path → Graph
    
  STEP 3: What are your scale requirements?
    < 1 TB, moderate traffic → PostgreSQL (covers 90% of needs!)
    1-100 TB, high write → Cassandra/ScyllaDB
    Need global ACID → CockroachDB/Spanner
    Need caching layer → Redis
    Need search → Elasticsearch
    
  STEP 4: What consistency do you need?
    Strong (banking, orders) → Relational, NewSQL
    Eventual (social, analytics) → Cassandra, DynamoDB
    Tunable → DynamoDB, Cassandra

COMMON COMBINATIONS (Polyglot Persistence):
  ┌──────────────────────────────────────────────────────────────┐
  │  System        │  Primary DB    │  Supporting DBs             │
  ├──────────────────────────────────────────────────────────────┤
  │  E-commerce    │  PostgreSQL    │  Redis (cache),             │
  │                │  (orders)      │  Elasticsearch (search),    │
  │                │                │  S3 (images)                │
  ├──────────────────────────────────────────────────────────────┤
  │  Social Media  │  PostgreSQL    │  Redis (feed cache),        │
  │                │  (users)       │  Cassandra (activity),      │
  │                │                │  Neo4j (connections)        │
  ├──────────────────────────────────────────────────────────────┤
  │  IoT Platform  │  TimescaleDB   │  Redis (real-time),         │
  │                │  (metrics)     │  PostgreSQL (device mgmt),  │
  │                │                │  S3 (raw data archive)      │
  ├──────────────────────────────────────────────────────────────┤
  │  AI/ML App     │  PostgreSQL    │  Pinecone (vectors),        │
  │                │  (metadata)    │  Redis (cache),             │
  │                │                │  S3 (model artifacts)       │
  └──────────────────────────────────────────────────────────────┘
```

---

## ❓ Interview Q&A

**Q1: How do you decide between SQL and NoSQL?**
> Start with these questions: (1) Do I need ACID transactions across multiple entities? → SQL. (2) Is my schema highly variable? → Document DB. (3) Is my access pattern simple key→value at massive scale? → KV store. (4) Do I need horizontal scaling to 100+ nodes? → NoSQL/NewSQL. Default: start with PostgreSQL — it handles 90% of use cases with JSONB for flexible parts. Only introduce specialized DBs when PostgreSQL becomes a bottleneck for a specific pattern.

**Q2: When would you use Cassandra over PostgreSQL?**
> When you need: (1) Write throughput > 100K/s consistently, (2) Data > 10 TB with linear scaling by adding nodes, (3) Multi-region active-active (all regions can write!), (4) High availability over strong consistency (AP in CAP). Example: messaging history (billions of messages, time-ordered, partition by conversation), IoT sensor data, user activity feeds. Don't use for: transactions, JOINs, ad-hoc queries, small datasets.

**Q3: What's the role of Redis in a typical architecture?**
> Redis serves as: (1) Cache layer (reduce DB load by 90%+), (2) Session store (fast user session lookups), (3) Rate limiter (sliding window counters), (4) Pub/Sub (real-time notifications), (5) Distributed locks (Redlock algorithm), (6) Leaderboards (sorted sets), (7) Job queues (lists). It's NOT a primary database — it's a supporting database that makes other systems faster. Most production systems have Redis alongside their primary DB.

**Q4: How do you handle multi-database consistency?**
> Challenge: data written to PostgreSQL must also appear in Elasticsearch and Redis. Solutions: (1) Change Data Capture (CDC): Debezium captures DB changes → publishes to Kafka → consumers update Elasticsearch/Redis. (2) Outbox pattern: write to DB + outbox table in same transaction → background job publishes events. (3) Application-level: write to DB, then update cache/search (risk: partial failure). Best: CDC + eventual consistency (accept seconds of lag between systems).

**Q5: What's the future trend in databases?**
> (1) Vector databases for AI/LLM applications (every AI app needs embeddings), (2) Serverless databases (Aurora Serverless, PlanetScale — scale to zero!), (3) Edge databases (SQLite at the edge, local-first with sync), (4) Multi-model databases (PostgreSQL doing everything: relational + JSON + vectors + full-text), (5) AI-powered query optimization (auto-indexing, auto-tuning). PostgreSQL is eating the world by adding every feature (pgvector, pg_trgm, JSONB, PostGIS...).

---

## 🔗 Related Topics
- [SQL vs NoSQL](../../Database/SQL_Vs_NoSQL.md) — Deep comparison
- [ACID](../../Database/ACID.md) — Transaction guarantees
- [Database Sharding](../../Database/Sharding.md) — Horizontal scaling
- [CAP Theorem](../../KeyConcepts/CAPTheorem.md) — Consistency trade-offs
- [Redis Deep Dive](../../Database/Redis_Deep_Dive.md) — In-memory DB internals

---

*"The best database is the one your team knows, until it isn't. Then the migration story becomes a 2-year project that makes everyone question their career choices." — Every Senior Engineer Who's Done a DB Migration* 📊
