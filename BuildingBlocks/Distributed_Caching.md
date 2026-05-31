# 🌍 Distributed Caching

> *"A single Redis instance can handle 100K ops/sec. Impressive! But what happens when you need 10 MILLION ops/sec? When your cache needs 500 GB but a single server has 64 GB RAM? When your cache must survive server failures without losing data? Welcome to distributed caching — where we spread our cache across many nodes to achieve scale, speed, and resilience that no single server could dream of."*

**⏱️ Estimated Time**: 30 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Caching](Caching.md), [Cache Eviction Policies](Cache_Eviction_Policies.md), [Consistent Hashing](../KeyConcepts/ConsistentHashing.md)

---

## 📋 Table of Contents
1. [Why Distributed Caching?](#-why-distributed-caching)
2. [Architecture Patterns](#-architecture-patterns)
3. [Data Distribution (Sharding)](#-data-distribution-sharding)
4. [Replication & HA](#-replication--ha)
5. [Consistency Challenges](#-consistency-challenges)
6. [Cache Stampede Prevention](#-cache-stampede-prevention)
7. [Java Implementation](#-java-implementation)
8. [Interview Q&A](#-interview-qa)

---

## 🎯 Why Distributed Caching?

```
SINGLE CACHE LIMITATIONS:
  ┌────────────────────────────────────────────────────────────────┐
  │  Problem           │ Single Node       │ Distributed Cache     │
  ├────────────────────────────────────────────────────────────────┤
  │  Memory capacity   │ Limited by 1 box  │ Sum of ALL nodes!     │
  │  Throughput        │ ~100K ops/sec     │ N × 100K ops/sec!     │
  │  Availability      │ SPOF!             │ Survives node failure! │
  │  Latency (global)  │ Far for some users│ Near for ALL users!   │
  └────────────────────────────────────────────────────────────────┘

DISTRIBUTED CACHE:
  ┌──────┐  ┌──────┐  ┌──────┐  ┌──────┐
  │Node 1│  │Node 2│  │Node 3│  │Node 4│  = 4 × capacity!
  │ 64GB │  │ 64GB │  │ 64GB │  │ 64GB │  = 256 GB total cache!
  │ 100K │  │ 100K │  │ 100K │  │ 100K │  = 400K ops/sec!
  └──────┘  └──────┘  └──────┘  └──────┘

REAL-WORLD SCALE:
  • Twitter: 100+ Redis clusters, petabytes of cached data
  • Facebook: Memcached with 1000+ nodes, trillions of items
  • Netflix: EVCache (on top of Memcached), 200+ clusters
  • Instagram: 12+ Redis clusters for feed, stories, likes
```

---

## 🏗️ Architecture Patterns

```
PATTERN 1: CLIENT-SIDE SHARDING
  ┌──────────┐
  │  Client  │──── hash(key) % N ────►┌────────┐
  │ (app)    │                         │ Node X │
  └──────────┘                         └────────┘
  
  • Client decides which node to query
  • Simple but: adding/removing nodes = MASSIVE reshuffling!
  • Used by: early Memcached deployments

PATTERN 2: PROXY-BASED
  ┌──────────┐     ┌───────┐     ┌────────┐
  │  Client  │────►│ Proxy │────►│ Node X │
  └──────────┘     │(Twemproxy)│  └────────┘
                   └───────┘
  
  • Proxy handles routing (clients are dumb!)
  • Proxy = potential bottleneck & SPOF
  • Used by: Twemproxy, Codis

PATTERN 3: CLUSTER MODE (Redis Cluster)
  ┌──────────┐     ┌────────┐─────►┌────────┐
  │  Client  │────►│ Node A │      │ Node B │  
  └──────────┘     └────────┘◄─────└────────┘
                        ↕               ↕
                   ┌────────┐      ┌────────┐
                   │ Node C │◄────►│ Node D │
                   └────────┘      └────────┘
  
  • Nodes gossip to share cluster topology
  • Client can query ANY node → MOVED redirect if wrong!
  • Smart clients cache slot mapping → direct routing!
  • Used by: Redis Cluster, Apache Ignite

PATTERN 4: MULTI-LAYER (L1 + L2)
  ┌──────────┐     ┌────────────┐     ┌────────────────┐
  │  Client  │────►│ L1: Local  │────►│ L2: Distributed │
  └──────────┘     │ (in-process)│     │ (Redis/Memcached)│
                   │ ~1000 items │     │ ~10M items      │
                   │ 0.01ms!     │     │ 1-2ms           │
                   └────────────┘     └────────────────┘
                         │                     │
                         └─────── Miss ────────└──► Database (50ms)
  
  L1 hit: microseconds! (HashMap in JVM)
  L2 hit: 1-2ms (network hop to Redis)
  DB: 50-200ms (only on double-miss!)
  
  Challenge: L1 invalidation across app instances!
  Solution: pub/sub (Redis PUBLISH) or events (Kafka/CDC)
```

---

## 🔑 Data Distribution (Sharding)

```
HOW TO DECIDE WHICH NODE STORES WHICH KEY?

METHOD 1: MODULO HASHING (naive!)
  node = hash(key) % num_nodes
  
  Problem: adding 1 node → ~100% of keys REMAP!
  3 nodes → 4 nodes: 75% of keys move! Cache goes cold! 🥶

METHOD 2: CONSISTENT HASHING (industry standard!)
  ┌─────────────────────────────────────────────────────┐
  │                                                      │
  │          N1 ●                                        │
  │        /      \                                      │
  │      /    HASH   \        Key "user:42"              │
  │     ●     RING    ●      → hashes to position X     │
  │    N4    (0 - 2^32)  N2   → walks clockwise          │
  │     \              /      → first node = N2!         │
  │      \          /                                    │
  │        ●──────●                                      │
  │        N3                                            │
  └─────────────────────────────────────────────────────┘
  
  Adding a node: only ~1/N keys remap! (not 75%!)
  Virtual nodes: each physical node = 100-200 virtual positions
  → Even distribution regardless of number of physical nodes!

METHOD 3: HASH SLOTS (Redis Cluster approach)
  16,384 hash slots distributed among nodes:
  
  Node A: slots 0 - 5460
  Node B: slots 5461 - 10922  
  Node C: slots 10923 - 16383
  
  key → CRC16(key) % 16384 → slot number → owning node!
  
  Rebalancing: move SLOTS (not individual keys) between nodes.
  Granularity: 16K slots gives fine-grained redistribution!

HASH TAGS (multi-key operations):
  Problem: "user:42:profile" and "user:42:session" → different nodes!
  Can't do multi-key operations across nodes!
  
  Solution: Hash tags! {user:42}:profile and {user:42}:session
  Only the part in {} is hashed → same node! ✅
  → Enables MGET, transactions, Lua scripts on related keys!
```

---

## 🔄 Replication & HA

```
REPLICATION: Copy data to multiple nodes for durability!

REDIS CLUSTER REPLICATION:
  ┌──────────┐         ┌──────────┐
  │ Master A │────────►│ Replica A│  (async replication!)
  │ slots 0-5K│        │ (standby)│
  └──────────┘         └──────────┘
  
  Master dies → Replica promoted to new master!
  Automatic failover! Clients get MOVED to new master.
  
  Caveat: ASYNC replication! Data acknowledged by master
  but NOT yet replicated → lost on crash! 
  (Redis chooses availability over consistency!)

MEMCACHED: NO BUILT-IN REPLICATION!
  Solution: replicate at application level (write to 2 pools)
  Or: accept cache miss on failure (just hit DB!)

EVCache (Netflix):
  • Writes to ALL replicas simultaneously! (write fan-out)
  • Reads from nearest replica (lowest latency!)
  • Zone-aware: replicas in different AZs
  • If one zone fails → read from another zone's replica!

HIGH AVAILABILITY PATTERNS:
  ┌───────────────────────────────────────────────────────────────┐
  │  Pattern          │  How It Works                             │
  ├───────────────────────────────────────────────────────────────┤
  │  Master-Replica   │  Write to master, read from replicas      │
  │  (Redis Sentinel) │  Auto-failover on master death            │
  ├───────────────────────────────────────────────────────────────┤
  │  Multi-Master     │  Write to any node, eventual consistency  │
  │  (Redis Cluster)  │  Each node owns subset of slots           │
  ├───────────────────────────────────────────────────────────────┤
  │  Cross-Region     │  Full replica in another region           │
  │  (ElastiCache)    │  Disaster recovery, read locally!         │
  └───────────────────────────────────────────────────────────────┘
```

---

## 🔒 Consistency Challenges

```
THE FUNDAMENTAL TENSION:
  Database is updated → cache has STALE data!
  How long until cache reflects the truth?

PROBLEM SCENARIOS:
  1. Write-through delay:
     App updates DB → updates cache → SUCCESS!
     But another app instance reads BEFORE cache update!
     
  2. Race condition:
     Thread A: reads DB (value=10)
     Thread B: updates DB to 20, invalidates cache
     Thread A: writes value=10 to cache (STALE!)
     → Cache now has 10, DB has 20! Inconsistent!

  3. Network partition:
     App → Redis: DELETE key (invalidate)
     Network drops the DELETE! Cache keeps stale value!

SOLUTIONS:
  ┌───────────────────────────────────────────────────────────────┐
  │  Strategy              │  Consistency │  Complexity │  Use     │
  ├───────────────────────────────────────────────────────────────┤
  │  Cache-aside + TTL     │  Eventual    │  Low        │  Most!   │
  │  Write-through         │  Strong*     │  Medium     │  Some    │
  │  CDC → invalidation    │  Eventual    │  High       │  Best!   │
  │  Version stamps        │  Eventual    │  Medium     │  Good    │
  └───────────────────────────────────────────────────────────────┘

CDC (Change Data Capture) INVALIDATION:
  DB transaction commits → Debezium captures change → 
  Kafka event → Cache invalidation service → DELETE from Redis!
  
  Why CDC is best:
  • Never miss an invalidation (DB is source of truth!)
  • Works even if app crashes between DB write and cache invalidate
  • Decoupled: any DB change (even direct SQL) triggers invalidation!

VERSION STAMPS:
  Cache entry: { value: "data", version: 5 }
  On write: increment version in DB (version: 6)
  On read: if cache.version < db.version → stale! Refresh!
  
  Simple but requires checking version on every read.
```

---

## ⚡ Cache Stampede Prevention

```
THE THUNDERING HERD PROBLEM:
  Popular key expires → 1000 requests simultaneously miss →
  1000 threads all hit the database → DB CRUSHED! 💀

  Timeline:
  t=0: Key "trending_post" expires (TTL!)
  t=0.001s: 1000 requests arrive, all get cache MISS
  t=0.002s: 1000 threads query DB for same data!
  t=0.5s: DB overwhelmed, latency spikes, cascading failure!

SOLUTION 1: MUTEX LOCK (Distributed Lock)
  First thread: acquires lock, queries DB, updates cache, releases lock
  Other threads: wait or return stale data (with grace period!)
  
  if (cache.get(key) == null) {
      if (redis.setnx("lock:" + key, "1", 5_SECONDS)) {
          // I won the lock! I'll refresh the cache.
          value = db.query(key);
          cache.set(key, value, TTL);
          redis.del("lock:" + key);
      } else {
          // Someone else is refreshing. Wait or return stale.
          Thread.sleep(50);
          return cache.get(key); // Retry
      }
  }

SOLUTION 2: EARLY REFRESH (Background Refresh)
  Don't wait for TTL to expire!
  At 80% of TTL: trigger background refresh!
  
  Entry: { value: "data", ttl: 60s, softTtl: 48s }
  After 48s: background thread refreshes (before true expiry!)
  Readers still get the current value (never see a miss!)

SOLUTION 3: PROBABILISTIC EARLY EXPIRATION
  Each reader has a small probability of refreshing early.
  As TTL approaches: probability increases!
  
  shouldRefresh = random() < beta * log(delta) * (now - fetchTime)
  
  Only ~1 request triggers refresh before actual expiry!
  Used by: Redis best practices, Google's paper on cache stampedes.
```

---

## 💻 Java Implementation

### Spring Boot + Redis Distributed Cache

```java
@Configuration
@EnableCaching
public class DistributedCacheConfig {
    
    @Bean
    public RedisCacheManager cacheManager(
            RedisConnectionFactory factory) {
        
        RedisCacheConfiguration config = RedisCacheConfiguration
            .defaultCacheConfig()
            .entryTtl(Duration.ofMinutes(10))
            .serializeValuesWith(
                SerializationPair.fromSerializer(
                    new GenericJackson2JsonRedisSerializer()));
        
        return RedisCacheManager.builder(factory)
            .cacheDefaults(config)
            .withCacheConfiguration("users", 
                config.entryTtl(Duration.ofMinutes(30)))
            .withCacheConfiguration("sessions",
                config.entryTtl(Duration.ofMinutes(5)))
            .build();
    }
}

@Service
public class UserService {
    
    @Autowired private UserRepository userRepo;
    @Autowired private RedisTemplate<String, Object> redis;
    
    /**
     * Cache-aside with stampede protection!
     */
    public User getUser(String userId) {
        String cacheKey = "user:" + userId;
        
        // 1. Try cache
        User cached = (User) redis.opsForValue().get(cacheKey);
        if (cached != null) return cached;
        
        // 2. Stampede protection: distributed lock!
        String lockKey = "lock:" + cacheKey;
        Boolean acquired = redis.opsForValue()
            .setIfAbsent(lockKey, "1", Duration.ofSeconds(5));
        
        if (Boolean.TRUE.equals(acquired)) {
            try {
                // Double-check after acquiring lock
                cached = (User) redis.opsForValue().get(cacheKey);
                if (cached != null) return cached;
                
                // 3. Load from DB
                User user = userRepo.findById(userId)
                    .orElseThrow();
                
                // 4. Populate cache
                redis.opsForValue().set(cacheKey, user, 
                    Duration.ofMinutes(30));
                return user;
            } finally {
                redis.delete(lockKey);
            }
        } else {
            // Another thread is loading. Wait and retry.
            try { Thread.sleep(50); } catch (Exception e) {}
            return getUser(userId); // Retry (with backoff!)
        }
    }
    
    /**
     * Invalidation on write (cache-aside pattern).
     */
    @Transactional
    public User updateUser(String userId, UserUpdateRequest req) {
        User user = userRepo.findById(userId).orElseThrow();
        user.setName(req.getName());
        User saved = userRepo.save(user);
        
        // Invalidate cache (don't update — avoid race conditions!)
        redis.delete("user:" + userId);
        
        return saved;
    }
}
```

### Multi-Layer Cache (L1 + L2)

```java
@Component
public class MultiLayerCache<V> {
    
    // L1: In-process (Caffeine — fastest!)
    private final Cache<String, V> l1Cache;
    
    // L2: Distributed (Redis — shared across instances!)
    @Autowired private RedisTemplate<String, V> redis;
    
    // Cross-instance invalidation
    @Autowired private RedisMessageListenerContainer listener;
    
    public MultiLayerCache() {
        this.l1Cache = Caffeine.newBuilder()
            .maximumSize(1_000)
            .expireAfterWrite(Duration.ofMinutes(1))
            .build();
        
        // Listen for invalidation events from other instances!
        subscribeToInvalidations();
    }
    
    public V get(String key, Function<String, V> loader) {
        // L1: local check (microseconds!)
        V value = l1Cache.getIfPresent(key);
        if (value != null) return value;
        
        // L2: Redis check (1-2ms)
        value = redis.opsForValue().get(key);
        if (value != null) {
            l1Cache.put(key, value); // Promote to L1!
            return value;
        }
        
        // Miss: load from source (DB)
        value = loader.apply(key);
        if (value != null) {
            redis.opsForValue().set(key, value, Duration.ofMinutes(10));
            l1Cache.put(key, value);
        }
        return value;
    }
    
    public void invalidate(String key) {
        l1Cache.invalidate(key);
        redis.delete(key);
        // Tell OTHER instances to invalidate their L1!
        redis.convertAndSend("cache:invalidate", key);
    }
    
    private void subscribeToInvalidations() {
        // When another instance invalidates → clear our L1 too!
        listener.addMessageListener((message, pattern) -> {
            String key = new String(message.getBody());
            l1Cache.invalidate(key);
        }, new PatternTopic("cache:invalidate"));
    }
}
```

---

## ❓ Interview Q&A

**Q1: How does consistent hashing help in distributed caching?**
> Without consistent hashing: adding/removing a node remaps ~100% of keys (cache goes completely cold!). With consistent hashing: only ~1/N keys remap when a node changes. Each node maps to multiple virtual positions on a hash ring. Keys map to the next clockwise node. Adding a node only "steals" keys from its clockwise neighbor. This minimizes cache misses during scaling events. Redis Cluster uses a similar concept with 16,384 hash slots.

**Q2: How would you handle cache invalidation across multiple services?**
> Best approach: CDC (Change Data Capture) via Debezium → Kafka → invalidation consumer. Why: catches ALL database changes (even direct SQL), survives app crashes, decoupled from write path. For L1 (in-process) caches across instances: use Redis pub/sub or Kafka topic — when any instance invalidates, broadcast to all others. For less critical data: just use short TTL (eventual consistency without explicit invalidation). Never do write-through to cache in a distributed system — race conditions make it unsafe.

**Q3: Explain the cache stampede problem and how to prevent it.**
> When a popular key expires, thousands of concurrent requests all miss the cache simultaneously and hit the database — potentially crashing it. Solutions: (1) Distributed lock — first thread acquires lock and refreshes; others wait or get stale data, (2) Early background refresh — at 80% of TTL, proactively refresh before expiry so no request ever misses, (3) Probabilistic early expiration — each request has increasing probability of triggering refresh as TTL approaches, spreading load over time, (4) "Never expire" + async refresh — key never truly expires, background job keeps it fresh.

**Q4: When would you use a multi-layer cache (L1 + L2)?**
> When you need both ultra-low latency AND large shared cache: L1 (Caffeine, in-process) gives microsecond access but is limited to JVM heap and per-instance. L2 (Redis) gives millisecond access but shared across all instances with huge capacity. Use L1 for hot items (top 1000 products), L2 for warm items (all recently accessed). Challenge: L1 consistency across instances — solved via Redis pub/sub invalidation broadcast. Used by: Netflix (EVCache + in-memory), most high-throughput Java services at scale.

---

## 🔗 Related Topics
- [Caching](Caching.md) — Caching fundamentals
- [Cache Eviction Policies](Cache_Eviction_Policies.md) — LRU, LFU, W-TinyLFU
- [Redis Deep Dive](../Database/Redis_Deep_Dive.md) — Redis internals
- [Consistent Hashing](../KeyConcepts/ConsistentHashing.md) — Data distribution
- [Distributed Cache Design](../SystemDesignCaseStudies/DesignDistributedCache.md) — Full case study

---

*"Distributed caching is the art of putting the RIGHT data, in the RIGHT place, at the RIGHT time, while handling the fact that multiple machines disagree about what 'right' means. Get it right and your system handles 10x traffic without breaking a sweat. Get it wrong and you have 10 machines all serving stale data confidently." — Platform Engineer* 🌍
