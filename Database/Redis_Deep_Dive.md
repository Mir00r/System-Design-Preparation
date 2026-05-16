# 🔴 Redis Deep Dive: The In-Memory Data Structure Store That Powers Real-Time Systems

> *"Redis isn't just a cache. It's a Swiss Army knife of data structures — strings, hashes, sorted sets, streams, HyperLogLog — all in-memory, all sub-millisecond. When LinkedIn needs to show 'who viewed your profile' in real-time across 900M users, Redis is the answer."*

**⏱️ Estimated Time**: 45 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Caching](../BuildingBlocks/Caching.md), [Caching Strategies](../BuildingBlocks/CachingStrategies.md)

---

## 📋 Table of Contents
1. [What is Redis](#-what-is-redis)
2. [Data Structures](#-data-structures)
3. [Persistence Models](#-persistence-models)
4. [Replication & High Availability](#-replication--high-availability)
5. [Redis Cluster (Sharding)](#-redis-cluster-sharding)
6. [Use Case Patterns](#-use-case-patterns)
7. [Spring Boot Integration](#-spring-boot-integration)
8. [Performance Tuning](#-performance-tuning)
9. [Common Pitfalls](#-common-pitfalls)
10. [Mini Challenge](#-mini-challenge)
11. [Interview Q&A](#-interview-qa)

---

## 🤔 What is Redis

```
Redis = REmote DIctionary Server

┌─────────────────────────────────────────────────────────┐
│                    REDIS AT A GLANCE                     │
│                                                         │
│  Type:        In-memory data structure store            │
│  Speed:       100K-1M+ ops/sec (single thread!)        │
│  Latency:     < 1ms for most operations                │
│  Persistence: Optional (RDB snapshots / AOF log)       │
│  Replication: Master-Replica with async replication     │
│  Clustering:  Redis Cluster (auto-sharding, 16384 slots)│
│  Protocol:    RESP (Redis Serialization Protocol)       │
│  License:     Source-available (SSPL since Redis 7.4)   │
└─────────────────────────────────────────────────────────┘

Why single-threaded is FAST:
  - No lock contention (no mutexes, no context switches)
  - All data in RAM (no disk I/O on reads)
  - I/O multiplexing (epoll) handles 10K+ connections
  - Operations are O(1) or O(log N) — microsecond execution
  - Network I/O is the bottleneck, not CPU
```

---

## 🏗️ Data Structures

```
┌─────────────────────────────────────────────────────────────────┐
│  DATA STRUCTURE    │  USE CASE                │  COMPLEXITY      │
├────────────────────┼──────────────────────────┼──────────────────┤
│  String            │  Cache, counters, flags  │  O(1) get/set    │
│  Hash              │  Object storage, sessions│  O(1) per field  │
│  List              │  Message queue, timeline │  O(1) push/pop   │
│  Set               │  Tags, unique visitors   │  O(1) add/remove │
│  Sorted Set (ZSet) │  Leaderboards, rankings  │  O(log N) add    │
│  Stream            │  Event log, Kafka-lite   │  O(1) append     │
│  HyperLogLog       │  Unique count (~0.81% err)│ O(1) add/count  │
│  Bitmap            │  Feature flags, presence │  O(1) bit ops    │
│  Geospatial        │  Nearby search, distance │  O(log N) add    │
└─────────────────────────────────────────────────────────────────┘
```

### Key Examples

```
STRING — Simple key-value, counters, rate limiting
  SET user:123:name "John"
  GET user:123:name → "John"
  INCR page:home:views → 1, 2, 3... (atomic counter)
  SETEX session:abc 3600 "user_data" (auto-expire in 1 hour)

HASH — Object-like storage (user profiles, product info)
  HSET user:123 name "John" email "john@ex.com" plan "pro"
  HGET user:123 email → "john@ex.com"
  HGETALL user:123 → {name: "John", email: "...", plan: "pro"}

SORTED SET — Leaderboards, time-based feeds, rate limiting
  ZADD leaderboard 1500 "player_A"
  ZADD leaderboard 2300 "player_B"
  ZADD leaderboard 1800 "player_C"
  ZREVRANGE leaderboard 0 9 WITHSCORES → top 10 players (descending)
  ZRANK leaderboard "player_A" → rank position

LIST — Message queues, recent activity feeds
  LPUSH notifications:user:123 "You have a new follower"
  RPOP notifications:user:123 → oldest notification (FIFO queue)
  LRANGE notifications:user:123 0 9 → last 10 notifications

STREAM — Event log (like Kafka topics, with consumer groups)
  XADD orders * user_id 123 amount 99.99 → "1705312800000-0"
  XREADGROUP GROUP payment-svc consumer-1 COUNT 10 BLOCK 5000 STREAMS orders >
  XACK orders payment-svc "1705312800000-0"

HYPERLOGLOG — Count unique items with minimal memory (12KB per counter!)
  PFADD unique_visitors:2024-01-15 "user_123" "user_456" "user_123"
  PFCOUNT unique_visitors:2024-01-15 → 2 (approximate, ±0.81% error)
  // 12KB to count billions of unique visitors!
```

---

## 💾 Persistence Models

```
OPTION 1: RDB (Point-in-time snapshots)
  How: Fork process → child writes all data to .rdb file
  Config: save 60 10000 (snapshot if 10000 keys changed in 60s)
  Pros: compact file, fast restart, good for backups
  Cons: data loss between snapshots (up to N minutes)
  Use when: cache (some loss acceptable), periodic backups

OPTION 2: AOF (Append-Only File — write log)
  How: Every write command appended to log file
  Config: appendfsync everysec (flush to disk every second)
  Pros: minimal data loss (≤1 second), human-readable log
  Cons: larger file, slower restart (replay all commands)
  Use when: persistent store (minimal loss required)

OPTION 3: RDB + AOF (recommended for production)
  How: Use AOF for durability + periodic RDB for fast restarts
  On restart: Redis loads AOF (most complete) if available
  Best of both worlds

OPTION 4: No persistence (pure cache)
  How: No disk writes at all
  Pros: maximum performance
  Cons: all data lost on restart
  Use when: pure caching layer (data reconstructable from DB)

DURABILITY COMPARISON:
  No persistence:  data loss = all data on crash
  RDB only:        data loss = up to last snapshot interval
  AOF everysec:    data loss = up to 1 second
  AOF always:      data loss = 0 (but 10x slower writes)
```

---

## 🔄 Replication & High Availability

```
MASTER-REPLICA REPLICATION:
  [Master] ──async replicate──▶ [Replica 1] (read-only)
       │                        [Replica 2] (read-only)
       │                        [Replica 3] (read-only)
       │
       └── Writes go to master only
           Reads can be served by any replica (read scaling)

  Replication is ASYNCHRONOUS:
    - Master doesn't wait for replicas to acknowledge
    - Possible stale reads from replicas (eventual consistency)
    - If master crashes before replication → data loss (small window)

REDIS SENTINEL (automatic failover):
  [Sentinel 1] ←monitor→ [Master]
  [Sentinel 2] ←monitor→ [Master]  ← if master fails...
  [Sentinel 3] ←monitor→ [Master]

  Failover process:
    1. Sentinels detect master is down (quorum: 2/3 agree)
    2. Sentinel promotes a replica to new master
    3. Other replicas reconfigure to follow new master
    4. Clients are notified of new master address
    Failover time: 10-30 seconds typically

REDIS CLUSTER (sharding + HA):
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ Master 1 │  │ Master 2 │  │ Master 3 │
  │ slots 0- │  │ slots    │  │ slots    │
  │   5460   │  │5461-10922│  │10923-16383│
  │ Replica  │  │ Replica  │  │ Replica  │
  └──────────┘  └──────────┘  └──────────┘

  - 16384 hash slots distributed across masters
  - Key → CRC16(key) mod 16384 → determines which master
  - Each master has 1+ replica for failover
  - Automatic failover within the cluster (no Sentinel needed)
```

---

## 🎯 Use Case Patterns

```
1. SESSION STORE
   SETEX session:token123 3600 '{"user_id":123,"role":"admin"}'
   // Fast lookup, auto-expiry, scales across app servers

2. RATE LIMITER (sliding window)
   MULTI
     ZADD rate:user:123 {now_ms} {request_id}
     ZREMRANGEBYSCORE rate:user:123 0 {now_ms - 60000}
     ZCARD rate:user:123
   EXEC
   // If count > 100 → rate limited

3. DISTRIBUTED LOCK (Redlock)
   SET lock:order:456 {uuid} NX PX 30000
   // NX = only if not exists, PX = expire in 30s
   // Release: check uuid matches, then DEL

4. REAL-TIME LEADERBOARD
   ZADD game:leaderboard {score} {player_id}
   ZREVRANK game:leaderboard {player_id} → player's rank
   ZREVRANGE game:leaderboard 0 9 WITHSCORES → top 10

5. PUB/SUB (real-time notifications)
   SUBSCRIBE channel:chat:room1
   PUBLISH channel:chat:room1 "Hello everyone!"
   // All subscribers receive instantly

6. CACHE-ASIDE PATTERN
   GET product:123
   if null → fetch from DB → SET product:123 {data} EX 300
```

---

## 💻 Spring Boot Integration

```java
// application.yml
spring:
  data:
    redis:
      host: redis-cluster.internal
      port: 6379
      password: ${REDIS_PASSWORD}
      lettuce:
        pool:
          max-active: 16
          max-idle: 8
          min-idle: 4

// 1. Cache-aside with Spring Cache abstraction
@Service
public class ProductService {
    
    @Cacheable(value = "products", key = "#id", unless = "#result == null")
    public Product getById(Long id) {
        return productRepository.findById(id).orElse(null);
    }
    
    @CacheEvict(value = "products", key = "#product.id")
    public Product update(Product product) {
        return productRepository.save(product);
    }
}

// 2. Direct RedisTemplate usage (fine-grained control)
@Service
public class RateLimiterService {
    private final StringRedisTemplate redis;
    
    public boolean isAllowed(String userId, int maxRequests, Duration window) {
        String key = "rate:" + userId;
        long now = Instant.now().toEpochMilli();
        
        List<Object> results = redis.executePipelined((RedisCallback<Object>) conn -> {
            conn.zAdd(key.getBytes(), now, UUID.randomUUID().toString().getBytes());
            conn.zRemRangeByScore(key.getBytes(), 0, now - window.toMillis());
            conn.zCard(key.getBytes());
            conn.pExpire(key.getBytes(), window.toMillis());
            return null;
        });
        
        Long count = (Long) results.get(2);
        return count <= maxRequests;
    }
}

// 3. Distributed lock with Redisson
@Service
public class OrderService {
    private final RedissonClient redisson;
    
    public void processOrder(Long orderId) {
        RLock lock = redisson.getLock("lock:order:" + orderId);
        try {
            if (lock.tryLock(5, 30, TimeUnit.SECONDS)) {
                // Critical section — only one instance processes this order
                doProcessOrder(orderId);
            }
        } finally {
            if (lock.isHeldByCurrentThread()) {
                lock.unlock();
            }
        }
    }
}
```

---

## ⚡ Performance Tuning

```
MEMORY OPTIMIZATION:
  - Use hash encoding for small objects (< 128 fields)
    hash-max-ziplist-entries 128
    hash-max-ziplist-value 64
  - Use short key names: "u:123:n" instead of "user:123:name"
  - Set maxmemory + eviction policy:
    maxmemory 4gb
    maxmemory-policy allkeys-lru (evict least-recently-used)

EVICTION POLICIES:
  noeviction:     return error when memory full (safe but fails writes)
  allkeys-lru:    evict any key using LRU (best for cache use case)
  volatile-lru:   evict only keys with TTL using LRU
  allkeys-lfu:    evict least-frequently-used (Redis 4.0+)
  volatile-ttl:   evict keys closest to expiry

PIPELINE (batch commands — reduce network round-trips):
  Without pipeline: 100 commands × 1ms RTT = 100ms
  With pipeline:    100 commands batched in 1 RTT = 1ms
  
  // Java
  redis.executePipelined(ops -> {
      for (int i = 0; i < 1000; i++) {
          ops.opsForValue().set("key:" + i, "value:" + i);
      }
  });

AVOID:
  ❌ KEYS * (scans ALL keys — blocks server, O(N))
  ✅ SCAN 0 MATCH pattern* COUNT 100 (cursor-based, non-blocking)
  ❌ Large values (> 1MB — causes latency spikes)
  ❌ Hot keys (one key getting 100K ops/sec — single-threaded bottleneck)
```

---

## ⚠️ Common Pitfalls

1. **Using Redis as primary database without persistence** — Redis with no persistence loses ALL data on crash/restart. If your data isn't reconstructable from another source, enable AOF (`appendfsync everysec`) at minimum. For critical data, use RDB+AOF.

2. **Hot key problem** — One key receiving disproportionate traffic (celebrity profile, viral content) bottlenecks the single thread handling that shard. Solutions: client-side caching, key replication to multiple shards with random suffix (`hot_key:1`, `hot_key:2`), or local in-process cache (Caffeine) in front of Redis.

3. **Thundering herd on cache miss** — When a popular cache key expires, 1000 concurrent requests all miss cache and hit the database simultaneously. Solutions: `SETNX` lock (only one request fetches from DB), jitter on TTL (randomize expiry ±10%), or cache warming before expiry.

4. **KEYS command in production** — `KEYS *` blocks the entire Redis server for seconds (O(N) scan). Use `SCAN` with cursor for production key iteration. Better yet, maintain a tracking set if you need to enumerate keys.

5. **Not setting maxmemory** — Without memory limits, Redis grows until OOM-killed by the OS (no graceful degradation). Always set `maxmemory` and an eviction policy. Monitor memory usage and alert at 80% threshold.

---

## 🧩 Mini Challenge

**Design a "trending topics" feature using Redis. Requirements: track hashtag usage in the last hour, return the top 10 trending hashtags, update in real-time.**

<details>
<summary>💡 Click to reveal answer</summary>

**Approach: Sliding window with Sorted Sets**

```
When a post with #SystemDesign is created:
  ZINCRBY trending:current_minute 1 "#SystemDesign"
  // Each minute gets its own sorted set

Every minute, rotate the window:
  // Union the last 60 minutes' sorted sets into a combined result
  ZUNIONSTORE trending:last_hour 60 
    trending:minute:1 trending:minute:2 ... trending:minute:60

Get top 10:
  ZREVRANGE trending:last_hour 0 9 WITHSCORES
  → [("#SystemDesign", 1523), ("#Redis", 890), ("#Kafka", 654), ...]

Cleanup:
  // Delete minute buckets older than 60 minutes
  DEL trending:minute:{old_minute}
```

**Optimized approach (lower memory):**
```
Use a single sorted set with decay:
  ZINCRBY trending 1 "#SystemDesign"
  
Every minute, decay all scores by 1/60th:
  // Lua script to multiply all scores by 0.983 (≈ e^(-1/60))
  EVAL "local members = redis.call('ZRANGEBYSCORE', KEYS[1], 1, '+inf')
        for _, m in ipairs(members) do
          redis.call('ZINCRBY', KEYS[1], -0.017 * redis.call('ZSCORE', KEYS[1], m), m)
        end" 1 trending

Result: recent hashtags have higher scores, old ones decay naturally.
Top 10: ZREVRANGE trending 0 9 WITHSCORES
```

</details>

---

## 📝 Interview Q&A

**Q: Redis is single-threaded. How does it handle 100K+ operations per second?**
> A: Redis's single thread handles only command execution, not network I/O. It uses I/O multiplexing (epoll/kqueue) to handle thousands of connections concurrently on a single thread. Each command executes in microseconds (in-memory, no disk I/O, no locks). The bottleneck is network latency, not CPU. For multi-core utilization: Redis 6+ uses I/O threads for network read/write (command execution remains single-threaded), and Redis Cluster shards data across multiple instances on different cores.

**Q: When would you choose Redis over Memcached?**
> A: Choose Redis when you need: (1) Rich data structures (sorted sets for leaderboards, streams for event logs, geo for location). (2) Persistence (data survives restarts). (3) Replication and high availability (Sentinel/Cluster). (4) Pub/Sub or Streams (real-time messaging). (5) Lua scripting (atomic multi-step operations). Choose Memcached when: simple key-value cache only, multi-threaded performance on a single node (Memcached uses all cores), or you need the simplest possible cache with no features beyond GET/SET.

**Q: How do you handle cache invalidation in a distributed system with Redis?**
> A: Multiple strategies: (1) **TTL-based** — set expiry on all keys; stale data is bounded by TTL. Simplest but can serve stale data. (2) **Event-driven invalidation** — on database write, publish to a Kafka/Redis Pub/Sub topic; cache service subscribes and deletes the key. Near real-time, but complex. (3) **Write-through** — update cache AND database atomically on every write. Consistent but adds write latency. (4) **Cache-aside with versioning** — store a version number; on read, check version against source of truth. Most common pattern: cache-aside with TTL + event-driven invalidation for frequently-updated data.

---

## 🔗 What to Read Next

1. **[BuildingBlocks/Caching.md](../BuildingBlocks/Caching.md)** — Caching patterns and strategies that Redis implements
2. **[BuildingBlocks/RateLimiting.md](../BuildingBlocks/RateLimiting.md)** — Redis-powered rate limiting algorithms
3. **[Database/Database_Selection_Guide.md](./Database_Selection_Guide.md)** — When to choose Redis vs other databases

---

*[← Query Optimization](./QueryOptimization.md) | [Back to Database](../INDEX.md) | [Next: MongoDB Deep Dive →](./MongoDB_Deep_Dive.md)*
