# 📈 Database Scaling: From One Server to Planet Scale

> *"Facebook's MySQL infrastructure serves 5 BILLION queries per second across tens of thousands of shards. They didn't start there — they started with a single MySQL server in a dorm room. Every database scaling story follows the same path: optimize → read replicas → caching → sharding. Skip steps, and you'll pay in downtime."*

**⏱️ Estimated Time**: 35 minutes | **🎯 Difficulty**: 🔴 Hard | **🔗 Prerequisites**: [SQL vs NoSQL](./SQL_Vs_NoSQL.md), [Sharding](./Sharding.md), [Indexing](./Indexing.md)

---

## 📋 Table of Contents
1. [The Scaling Journey](#-the-scaling-journey)
2. [Vertical Scaling (Scale Up)](#-vertical-scaling-scale-up)
3. [Read Replicas (Scale Reads)](#-read-replicas-scale-reads)
4. [Caching Layer](#-caching-layer)
5. [Horizontal Sharding (Scale Writes)](#-horizontal-sharding-scale-writes)
6. [Partitioning Strategies](#-partitioning-strategies)
7. [Multi-Region Databases](#-multi-region-databases)
8. [Scaling Patterns Summary](#-scaling-patterns-summary)
9. [Java/Spring Boot Examples](#-javaspring-boot-examples)
10. [Mini Challenge](#-mini-challenge)
11. [Interview Q&A](#-interview-qa)

---

## 🗺️ The Scaling Journey

```
╔══════════════════════════════════════════════════════════════════╗
║  Every database scaling journey follows this progression:      ║
║                                                                ║
║  Single DB → Optimize queries → Read replicas → Cache layer   ║
║  → Vertical scale → Sharding → Multi-region                   ║
║                                                                ║
║  Don't jump to sharding! Exhaust simpler solutions first.     ║
╚══════════════════════════════════════════════════════════════════╝

TRAFFIC THRESHOLDS (rough guidelines):
  ┌───────────────────┬──────────────────────────────────────────┐
  │  Stage            │  When You Need It                        │
  ├───────────────────┼──────────────────────────────────────────┤
  │  Single DB        │  < 1,000 QPS                             │
  │  + Indexing       │  1,000 - 5,000 QPS                       │
  │  + Read Replicas  │  5,000 - 50,000 QPS (read-heavy)         │
  │  + Caching        │  50,000 - 500,000 QPS                    │
  │  + Sharding       │  > 500,000 QPS or > 1TB data             │
  │  + Multi-region   │  Global users, < 100ms latency worldwide │
  └───────────────────┴──────────────────────────────────────────┘
```

### The Evolution

```
Stage 1: Single Database
  ┌──────────────┐
  │  App Server  │──────►┌──────────┐
  └──────────────┘       │  Single  │
                         │    DB    │
                         └──────────┘

Stage 2: Read Replicas
  ┌──────────────┐ writes ►┌──────────┐
  │  App Server  │─────────│  Primary │
  └──────────────┘ reads  ►│          │
        │                  └─────┬────┘
        │                        │ replication
        │ reads               ┌──┴───┐
        └────────────────────►│Replica│
                              └──────┘

Stage 3: Caching
  ┌──────────────┐  reads  ┌───────┐  miss  ┌──────────┐
  │  App Server  │────────►│ Cache │────────►│  DB      │
  └──────────────┘         │(Redis)│         └──────────┘
                           └───────┘

Stage 4: Sharding
  ┌──────────────┐
  │  App Server  │
  └──────┬───────┘
         │ route by shard key
    ┌────┼────────────┐
    ▼    ▼            ▼
  ┌────┐┌────┐    ┌────┐
  │ S1 ││ S2 │... │ SN │
  └────┘└────┘    └────┘
  Users  Users     Users
  A-H    I-P       Q-Z
```

---

## ⬆️ Vertical Scaling (Scale Up)

```
Add more resources to EXISTING server:
  • More RAM (16GB → 256GB)
  • Faster CPU (8 cores → 64 cores)
  • Better storage (HDD → NVMe SSD)
  • More IOPS

LIMITS:
  ┌─────────────────────────────────────────────────────────┐
  │  AWS largest instance (x2idn.24xlarge):                 │
  │  • 96 vCPUs                                             │
  │  • 1,536 GB RAM                                         │
  │  • 3.8TB NVMe SSD                                       │
  │  • Cost: ~$16/hour ($140K/year)                         │
  │                                                         │
  │  When THIS isn't enough... you MUST go horizontal.      │
  └─────────────────────────────────────────────────────────┘

QUICK WINS BEFORE VERTICAL SCALING:
  1. Add indexes (10x-1000x improvement!)
  2. Optimize queries (EXPLAIN ANALYZE)
  3. Connection pooling (reduce connection overhead)
  4. Query caching (materialized views)
  5. Archive old data (smaller working set)
```

---

## 📖 Read Replicas (Scale Reads)

```
PRINCIPLE: Most apps are 90% reads, 10% writes.
  → Replicate data to multiple read-only copies!

  ┌────────────────────────────────────────────────────┐
  │  Writes → Primary (single source of truth)         │
  │  Reads  → Any replica (load balanced!)             │
  │                                                    │
  │  1 Primary + 5 Replicas = 6x read throughput!      │
  └────────────────────────────────────────────────────┘

REPLICATION TYPES:
  Synchronous: Primary waits for replica ACK (strong consistency, slow)
  Asynchronous: Primary doesn't wait (fast, but replication lag!)
  Semi-synchronous: Wait for at least 1 replica (compromise)

REPLICATION LAG PROBLEM:
  User writes: UPDATE profile SET name='Alice' → Primary ✅
  User reads (0.1s later): SELECT name FROM profile → Replica: "Bob" 😱
  
  Replica hasn't received the update yet! (lag = 100ms typically)
  
  Solutions:
  • Read-your-own-writes: After write, read from Primary (for that user)
  • Causal consistency: Track write timestamp, route to up-to-date replica
  • Sticky sessions: Same user always hits same replica
```

---

## 🗄️ Caching Layer

```
Put frequently-read data in MEMORY (Redis/Memcached):
  Read from cache: 0.1ms (vs 5-50ms from DB!)
  
  ┌──────────────┐
  │  App Server  │
  └──────┬───────┘
         │ 1. Check cache
         ▼
  ┌──────────────┐  cache HIT (99%)   ┌──────────┐
  │  Redis Cache │─────────────────────►│ Response │
  │              │                      └──────────┘
  │  cache MISS  │
  │  (1%)        │
  └──────┬───────┘
         │ 2. Query DB
         ▼
  ┌──────────────┐  3. Store in cache
  │  Database    │──────────────────────► Redis
  └──────────────┘

CACHE-ASIDE PATTERN:
  Hit rate 99% = only 1% of requests reach database!
  100K QPS → only 1K QPS to DB. Massive reduction!
  
  See: [Caching Strategies](../BuildingBlocks/CachingStrategies.md)
```

---

## 🔀 Horizontal Sharding (Scale Writes)

```
PROBLEM: Read replicas don't help with WRITE scaling!
  All writes still go to ONE primary. Single machine limit!

SOLUTION: Split data across multiple databases (shards).
  Each shard handles a SUBSET of the data.
  
  ┌─────────────────────────────────────────────────────────┐
  │  Shard 1: Users A-H     (1/4 of all writes)            │
  │  Shard 2: Users I-P     (1/4 of all writes)            │
  │  Shard 3: Users Q-W     (1/4 of all writes)            │
  │  Shard 4: Users X-Z     (1/4 of all writes)            │
  │                                                         │
  │  4 shards = 4x write throughput!                        │
  │  Each shard can have its own read replicas!             │
  └─────────────────────────────────────────────────────────┘

CHALLENGES:
  • Cross-shard queries (JOIN across shards = very expensive!)
  • Rebalancing (adding new shards, moving data)
  • Distributed transactions (ACID across shards is HARD)
  • Hotspots (uneven data distribution)
  • Auto-increment IDs (can't use sequential IDs across shards!)
```

---

## 📊 Partitioning Strategies

```
1. RANGE-BASED PARTITIONING:
   Shard by range of shard key values
   
   Shard 1: user_id 1 - 1,000,000
   Shard 2: user_id 1,000,001 - 2,000,000
   Shard 3: user_id 2,000,001 - 3,000,000
   
   ✅ Range queries efficient (users 500-600 → one shard)
   ❌ Hotspots! New users always hit latest shard!

2. HASH-BASED PARTITIONING:
   shard = hash(user_id) % num_shards
   
   hash(123) % 4 = 3 → Shard 3
   hash(456) % 4 = 0 → Shard 0
   
   ✅ Even distribution (no hotspots)
   ❌ Range queries need to hit ALL shards
   ❌ Adding shards = massive data movement!

3. CONSISTENT HASHING:
   Ring-based hashing (minimal redistribution when adding shards)
   
   ✅ Add/remove shard → only ~1/N data moves
   ❌ Complex to implement
   Used by: DynamoDB, Cassandra, Redis Cluster

4. DIRECTORY-BASED:
   Lookup table: user_id → shard_id
   
   ✅ Flexible (can move any user to any shard)
   ❌ Lookup table becomes bottleneck/SPOF
```

---

## 🌍 Multi-Region Databases

```
For global applications with users worldwide:

  ┌─────────────┐         ┌─────────────┐         ┌─────────────┐
  │  US-East    │◄───────►│  EU-West    │◄───────►│  Asia       │
  │  Primary    │  sync   │  Primary    │  sync   │  Primary    │
  │  (US data)  │  or     │  (EU data)  │  or     │  (Asia data)│
  └─────────────┘  async  └─────────────┘  async  └─────────────┘
  
  Options:
  • Active-Passive: One region primary, others read replicas
  • Active-Active: Multiple regions accept writes (conflict resolution!)
  • Geo-partitioned: Each region owns its data (no conflicts!)

TECHNOLOGIES:
  • CockroachDB: Multi-region SQL (Spanner-like)
  • Google Spanner: Global strong consistency (TrueTime!)
  • DynamoDB Global Tables: Multi-region, eventual consistency
  • Vitess (YouTube): MySQL sharding + geo-distribution
```

---

## 📋 Scaling Patterns Summary

```
┌──────────────────────┬────────────────────┬─────────────────────────┐
│  Technique           │  Scales            │  Complexity             │
├──────────────────────┼────────────────────┼─────────────────────────┤
│  Query optimization  │  Reads + Writes    │  🟢 Low                │
│  Connection pooling  │  Connections       │  🟢 Low                │
│  Read replicas       │  Reads only        │  🟡 Medium             │
│  Caching (Redis)     │  Reads mostly      │  🟡 Medium             │
│  Vertical scaling    │  Both (limited!)   │  🟢 Low (just $$$)     │
│  Sharding            │  Both              │  🔴 High               │
│  Multi-region        │  Global latency    │  🔴 Very High          │
└──────────────────────┴────────────────────┴─────────────────────────┘
```

---

## 💻 Java/Spring Boot Examples

### Read-Write Splitting with Spring

```java
@Configuration
public class DataSourceConfig {
    
    @Bean
    @Primary
    public DataSource routingDataSource() {
        Map<Object, Object> targetDataSources = new HashMap<>();
        targetDataSources.put("PRIMARY", primaryDataSource());
        targetDataSources.put("REPLICA", replicaDataSource());
        
        RoutingDataSource routingDS = new RoutingDataSource();
        routingDS.setTargetDataSources(targetDataSources);
        routingDS.setDefaultTargetDataSource(primaryDataSource());
        return routingDS;
    }
}

public class RoutingDataSource extends AbstractRoutingDataSource {
    @Override
    protected Object determineCurrentLookupKey() {
        return TransactionSynchronizationManager.isCurrentTransactionReadOnly()
            ? "REPLICA" : "PRIMARY";
    }
}

@Service
public class UserService {
    
    @Transactional(readOnly = true)  // → Routes to REPLICA!
    public User findById(Long id) {
        return userRepository.findById(id).orElseThrow();
    }
    
    @Transactional  // → Routes to PRIMARY!
    public User create(User user) {
        return userRepository.save(user);
    }
}
```

### Simple Sharding Router

```java
@Component
public class ShardRouter {
    
    private final List<DataSource> shards;
    private static final int NUM_SHARDS = 4;
    
    public DataSource getShardForUser(Long userId) {
        int shardIndex = (int) (userId % NUM_SHARDS);
        return shards.get(shardIndex);
    }
    
    public DataSource getShardForKey(String key) {
        int hash = Math.abs(key.hashCode());
        int shardIndex = hash % NUM_SHARDS;
        return shards.get(shardIndex);
    }
    
    // Fan-out query (hits ALL shards — expensive!)
    public <T> List<T> queryAllShards(Function<JdbcTemplate, List<T>> query) {
        return shards.parallelStream()
            .map(ds -> new JdbcTemplate(ds))
            .flatMap(jt -> query.apply(jt).stream())
            .collect(Collectors.toList());
    }
}
```

---

## 🎮 Mini Challenge

### 🧩 Design: Scale a Social Media Database

Your social media app has:
- 100M users, 1B posts, 10B likes
- 50K reads/sec, 5K writes/sec
- Read:Write ratio = 10:1
- Growing 20% per month

Design the database scaling strategy:

<details>
<summary>🔑 Answer</summary>

**Phase 1 (Current: 50K reads/sec):**
- Primary + 3 read replicas (handles 4x reads)
- Redis cache for user profiles and hot posts (reduces DB reads by 90%)
- Result: DB sees only ~5K reads/sec + 5K writes/sec ✅

**Phase 2 (3 months: 100K reads/sec, 10K writes/sec):**
- Add more replicas (5 total)
- Cache warm-up for trending content
- Shard likes table (most write-heavy) by post_id
- Result: Writes distributed, reads from cache

**Phase 3 (12 months: 500K reads/sec, 50K writes/sec):**
- Shard users table by user_id (hash-based)
- Shard posts by user_id (co-locate with user)
- Keep likes sharded by post_id
- Each shard: 1 primary + 2 replicas
- Result: 16 shards × 3 = 48 database instances

**Shard key choices:**
- Users: hash(user_id) — even distribution
- Posts: user_id — co-locate with author (avoids cross-shard for profile pages)
- Likes: post_id — co-locate with post (like count is hot query)
</details>

---

## ❓ Interview Q&A

**Q1: Walk me through the database scaling journey from startup to scale.**
> Start: single DB with proper indexes. As reads grow: add read replicas. Add caching layer (Redis) for hot data. When writes exceed single-server capacity: shard horizontally. For global users: multi-region deployment. Key: exhaust simpler solutions first. Premature sharding adds enormous complexity.

**Q2: How do read replicas work and what's the replication lag problem?**
> Primary handles all writes and asynchronously replicates to read replicas. Problem: replica may be milliseconds to seconds behind primary. A user writes data then immediately reads from a replica — sees stale data! Solutions: read-your-own-writes (route to primary after write), sticky sessions, or causal consistency tracking.

**Q3: What are the main challenges of database sharding?**
> (1) Cross-shard queries (JOINs become distributed operations), (2) Choosing the right shard key (bad choice = hotspots), (3) Rebalancing when adding shards (data migration), (4) Distributed transactions (no simple ACID), (5) Maintaining referential integrity, (6) Auto-increment IDs (need distributed ID generation like Snowflake).

**Q4: How do you choose a shard key?**
> The shard key should: (1) Distribute data evenly (avoid hotspots), (2) Minimize cross-shard queries (co-locate related data), (3) Be immutable (changing shard key = moving data), (4) Be frequently used in queries (routing efficiency). Example: user_id for user data, but NOT timestamp (all recent data hits one shard).

**Q5: When should you NOT shard?**
> Don't shard if: (1) Data fits on one machine (< 1TB), (2) Traffic manageable with replicas + caching (< 100K QPS), (3) Many cross-entity JOINs needed, (4) Strong ACID transactions across entities required, (5) Team lacks operational expertise. The complexity cost of sharding is enormous — exhaust vertical scaling, caching, and read replicas first.

---

## 🔗 Related Topics
- [Sharding](./Sharding.md) — Deep dive on sharding strategies
- [SQL vs NoSQL](./SQL_Vs_NoSQL.md) — Choosing database type
- [Consistent Hashing](../KeyConcepts/ConsistentHashing.md) — For shard routing
- [Caching Strategies](../BuildingBlocks/CachingStrategies.md) — Cache layer design
- [Connection Pooling](./ConnectionPooling.md) — Optimizing connections

---

*"There are only two hard problems in databases: cache invalidation, naming things, and knowing when to shard." — Every DBA Ever* 📈
