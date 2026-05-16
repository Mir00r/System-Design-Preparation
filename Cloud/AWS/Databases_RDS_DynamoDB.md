# 🗄️ AWS Databases: RDS, Aurora & DynamoDB

> *"The database you choose defines your system's capabilities and constraints. RDS gives you familiar SQL with managed operations. DynamoDB gives you single-digit millisecond performance at any scale — if you model your data correctly."*

**⏱️ Estimated Time**: 25 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [SQL vs NoSQL](../../Database/SQL_Vs_NoSQL.md), [AWS Core Services](./Core_Services.md)

---

## 📋 Table of Contents
1. [Database Options Overview](#-database-options)
2. [RDS (Relational Database Service)](#-rds)
3. [Aurora](#-aurora)
4. [DynamoDB](#-dynamodb)
5. [ElastiCache](#-elasticache)
6. [Decision Matrix](#-decision-matrix)
7. [Interview Q&A](#-interview-qa)

---

## 🗺️ Database Options

```
┌─────────────────────────────────────────────────────────────────────┐
│                   AWS DATABASE SERVICES                              │
├────────────────┬─────────────────┬──────────────────────────────────┤
│ Service        │ Type            │ Best For                         │
├────────────────┼─────────────────┼──────────────────────────────────┤
│ RDS            │ Relational (SQL)│ Traditional apps, complex queries│
│ Aurora         │ Relational      │ High-perf MySQL/PostgreSQL       │
│ DynamoDB       │ Key-Value/Doc   │ Any scale, single-digit ms       │
│ ElastiCache    │ In-memory       │ Caching, sessions, leaderboards  │
│ DocumentDB     │ Document (JSON) │ MongoDB-compatible workloads     │
│ Neptune        │ Graph           │ Social networks, fraud detection │
│ Keyspaces      │ Wide-column     │ Cassandra-compatible workloads   │
│ Timestream     │ Time-series     │ IoT, metrics, monitoring         │
│ MemoryDB       │ Redis-compatible│ Durable in-memory (Redis + disk) │
└────────────────┴─────────────────┴──────────────────────────────────┘
```

---

## 🐘 RDS

```
WHAT: Managed relational databases (you choose the engine)
  Engines: PostgreSQL, MySQL, MariaDB, Oracle, SQL Server
  
  AWS manages: patching, backups, failover, monitoring
  You manage: schema design, queries, performance tuning, scaling decisions

KEY FEATURES:
  Multi-AZ:    Synchronous standby in another AZ (automatic failover <60s)
  Read Replicas: Async copies for read scaling (up to 15 replicas)
  Automated Backups: Point-in-time recovery (up to 35 days)
  Encryption: At-rest (KMS) + in-transit (SSL/TLS)
  
  ┌─────────────────────────────────────────────────────┐
  │ Multi-AZ Architecture:                              │
  │                                                     │
  │   AZ-A              AZ-B                            │
  │  ┌─────────┐       ┌──────────┐                    │
  │  │ Primary │──sync─▶│ Standby  │                    │
  │  │  (R/W)  │       │ (no access)│                   │
  │  └─────────┘       └──────────┘                    │
  │       ▲                  ▲                          │
  │       │                  │ (promoted on failure)    │
  │   [App writes]      [Auto failover]                │
  │                                                     │
  │ Read Replicas (separate from Multi-AZ):             │
  │  ┌─────────┐  async  ┌──────────┐                  │
  │  │ Primary │────────▶│ Replica 1│ (reads)           │
  │  │  (R/W)  │────────▶│ Replica 2│ (reads)           │
  │  └─────────┘         └──────────┘                  │
  └─────────────────────────────────────────────────────┘

INSTANCE SIZES:
  db.t3.micro:   1 vCPU, 1 GB RAM (~$15/month) — dev/test
  db.r6g.large:  2 vCPU, 16 GB RAM (~$175/month) — small production
  db.r6g.4xlarge: 16 vCPU, 128 GB RAM (~$1400/month) — large production
```

---

## 🚀 Aurora

```
WHAT: AWS-built MySQL/PostgreSQL-compatible database (5x faster than MySQL)
  - Same SQL, drivers, tools as MySQL/PostgreSQL
  - Cloud-native storage architecture (shared distributed storage)
  - Up to 128 TB auto-scaling storage
  - 6 copies of data across 3 AZs (extremely durable)

WHY AURORA OVER RDS:
  ┌──────────────────────┬────────────────────┬──────────────────────┐
  │                      │ RDS MySQL          │ Aurora MySQL         │
  ├──────────────────────┼────────────────────┼──────────────────────┤
  │ Storage              │ EBS (single volume)│ Distributed (6 copies)│
  │ Failover time        │ 60-120 seconds     │ ~30 seconds          │
  │ Read replicas        │ 5 max              │ 15 max               │
  │ Replica lag          │ Seconds            │ <100ms (shared storage)│
  │ Storage scaling      │ Manual (resize EBS)│ Auto (10GB→128TB)    │
  │ Throughput           │ Baseline           │ Up to 5x MySQL       │
  │ Cost                 │ Lower              │ ~20% more            │
  └──────────────────────┴────────────────────┴──────────────────────┘

AURORA SERVERLESS v2:
  - Auto-scales compute (0.5 ACU to 128 ACU)
  - Pay per second of compute used
  - Scales in seconds (not minutes like adding read replicas)
  - Great for: variable workloads, dev/test, infrequent access
```

---

## ⚡ DynamoDB

```
WHAT: Serverless NoSQL key-value and document database
  - Single-digit millisecond latency at ANY scale
  - No servers to manage, no capacity planning (on-demand mode)
  - Automatic scaling to millions of requests per second
  - Global Tables: multi-region, active-active replication

DATA MODEL:
  Table → Items (rows) → Attributes (columns)
  Primary Key: Partition Key (hash) + optional Sort Key (range)
  
  Example: Orders table
    PK: customer_id (partition key)
    SK: order_date#order_id (sort key)
    
  Query patterns:
    Get all orders for customer: PK = "cust_123"
    Get orders in date range: PK = "cust_123" AND SK BETWEEN "2024-01-01" AND "2024-12-31"

CAPACITY MODES:
  On-Demand:   Pay per request ($1.25 per million writes, $0.25 per million reads)
               Best for: unpredictable traffic, new tables
  Provisioned: Set RCU/WCU (Read/Write Capacity Units)
               Best for: predictable traffic (cheaper at scale)
               + Auto-scaling adjusts capacity based on usage

KEY FEATURES:
  DynamoDB Streams: CDC — captures all changes (for Lambda triggers, replication)
  DAX (DynamoDB Accelerator): In-memory cache, microsecond reads
  Global Tables: Multi-region active-active (sub-second replication)
  TTL: Auto-delete expired items (sessions, temp data)
  
LIMITATIONS:
  - Max item size: 400 KB
  - No joins (denormalize or use multiple queries)
  - Query must include partition key (no full-table scans for queries)
  - Complex queries need GSIs (Global Secondary Indexes) — max 20 per table
```

---

## 🏎️ ElastiCache

```
WHAT: Managed Redis or Memcached for in-memory caching

  Redis:     Rich data structures, persistence, pub/sub, Lua scripting
  Memcached: Simple key-value, multi-threaded, no persistence

USE CASES:
  - Session storage (fast read/write, TTL expiration)
  - Database query caching (reduce DB load by 80%+)
  - Leaderboards (Redis sorted sets)
  - Rate limiting (Redis INCR + EXPIRE)
  - Real-time analytics (Redis HyperLogLog, Streams)

ARCHITECTURE:
  [App] → check cache → [ElastiCache Redis]
                              │ miss
                              ▼
                         [RDS/DynamoDB] → store result in cache
```

---

## 📊 Decision Matrix

| Question | RDS/Aurora | DynamoDB | ElastiCache |
|---|---|---|---|
| Need complex JOINs? | ✅ Yes | ❌ No | ❌ No |
| Need ACID transactions? | ✅ Full | ✅ Limited (25 items) | ❌ No |
| Predictable access patterns? | Any | ✅ Required | Key-based |
| Scale to millions TPS? | ❌ Hard | ✅ Easy | ✅ Easy |
| Latency requirement? | Low ms | Single-digit ms | Sub-ms |
| Schema flexibility? | Fixed | Flexible | Key-value |
| Cost at low scale? | Higher (instances run 24/7) | Lower (on-demand) | Higher |

---

## ⚠️ Common Pitfalls

1. **DynamoDB hot partitions** — If one partition key gets disproportionate traffic (e.g., celebrity user), that partition throttles. Design partition keys for even distribution. Use write sharding (append random suffix) for hot keys.

2. **RDS without connection pooling** — Each RDS connection uses memory. Lambda can create hundreds of connections simultaneously. Use RDS Proxy for connection pooling between Lambda and RDS.

3. **Not using read replicas** — If 80% of your traffic is reads, route reads to replicas (accepting eventual consistency) and writes to primary. Reduces primary load by 80%.

---

## 📝 Interview Q&A

**Q: When would you choose DynamoDB over Aurora for a new microservice?**
> A: Choose **DynamoDB** when: (1) Access patterns are known and key-based (get user by ID, get orders by user+date). (2) Need single-digit ms latency at any scale without tuning. (3) Serverless architecture (pairs with Lambda). (4) Simple data model without complex relationships. (5) Need global replication (Global Tables). Choose **Aurora** when: (1) Complex queries with JOINs and aggregations. (2) Data has many relationships (relational model fits better). (3) Need ad-hoc queries (analytics, reporting). (4) Existing application uses SQL. (5) ACID transactions across many entities. Many systems use BOTH: Aurora for complex transactional data, DynamoDB for high-speed simple lookups (sessions, user preferences, activity feeds).

---

## 🔗 What to Read Next

1. **[Cloud/AWS/Networking_VPC.md](./Networking_VPC.md)** — Network your databases
2. **[Database/Sharding.md](../../Database/Sharding.md)** — When to shard
3. **[BuildingBlocks/Caching.md](../../BuildingBlocks/Caching.md)** — Caching strategies with ElastiCache

---

*[← Storage](./Storage_S3_EBS_EFS.md) | [Back to Index](../../INDEX.md) | [Next: Networking →](./Networking_VPC.md)*
