# 💾 Design a Distributed Cache (Redis at Scale)

> *"Facebook's Memcached cluster serves 5 BILLION requests per second — yes, BILLION with a B. Each request returns in under 1 millisecond. Without caching, Facebook would need 1000x more database servers. A distributed cache is the difference between $1M and $1B in infrastructure costs. But building one that's consistent, available, and handles cache stampedes? That's where the engineering gets real."*

**⏱️ Estimated Time**: 40 minutes | **🎯 Difficulty**: 🟢 Easy-Medium | **🔗 Prerequisites**: [Caching](../../BuildingBlocks/Caching.md), [Caching Strategies](../../BuildingBlocks/CachingStrategies.md), [Consistent Hashing](../../KeyConcepts/ConsistentHashing.md)

---

## 📋 Table of Contents
1. [Requirements](#-requirements)
2. [Why Distributed Cache?](#-why-distributed-cache)
3. [Architecture Design](#-architecture-design)
4. [Data Partitioning](#-data-partitioning)
5. [Cache Eviction Policies](#-cache-eviction-policies)
6. [Cache Patterns](#-cache-patterns)
7. [Common Problems & Solutions](#-common-problems--solutions)
8. [High Availability](#-high-availability)
9. [Java Implementation](#-java-implementation)
10. [Mini Challenge](#-mini-challenge)
11. [Interview Q&A](#-interview-qa)

---

## 📝 Requirements

### Functional Requirements
```
✅ put(key, value, TTL) — Store with optional expiration
✅ get(key) — Retrieve (< 1ms latency!)
✅ delete(key) — Invalidate
✅ Support various data types (strings, hashes, lists, sets)
✅ Atomic operations (increment, CAS)
✅ Bulk operations (multi-get for efficiency)
✅ TTL-based automatic expiration
✅ LRU/LFU eviction when memory full
```

### Non-Functional Requirements
```
✅ Sub-millisecond latency (< 1ms for p99!)
✅ High throughput (millions of ops/second)
✅ High availability (cache down = database overload!)
✅ Scalable (add nodes to increase capacity)
✅ Consistent hashing (minimal data movement on scale)
✅ Fault tolerant (node failure doesn't lose ALL data)
✅ Memory efficient (maximize useful data per GB RAM)
```

### Scale Numbers
```
Total cached data: 100 TB (across all nodes)
Individual node memory: 64 GB RAM
Nodes needed: 100TB / 64GB ≈ 1,600 nodes (without replication)
With replication (3x): ~5,000 nodes

QPS: 10 million reads/second, 1 million writes/second
Latency: p50 < 0.5ms, p99 < 2ms
```

---

## ❓ Why Distributed Cache?

```
SINGLE-NODE CACHE LIMITATIONS:

  Problem 1: MEMORY LIMIT
    Single machine: 64-256 GB RAM
    Need to cache: 10 TB+
    → MUST spread across multiple machines!
    
  Problem 2: THROUGHPUT LIMIT
    Single Redis: ~100K ops/second (single-threaded!)
    Need: 10M+ ops/second
    → MUST parallelize across machines!
    
  Problem 3: SINGLE POINT OF FAILURE
    Server crashes → cache gone → DB overwhelmed → cascading failure!
    → MUST have replicas!
    
  Problem 4: NETWORK DISTANCE
    Users in Tokyo → cache in US = 150ms RTT!
    → MUST have geographically distributed caches!

DISTRIBUTED CACHE COMPARISON:
  ┌─────────────────────────────────────────────────────────────┐
  │  Solution    │  Latency │  Scale     │  Complexity          │
  ├─────────────────────────────────────────────────────────────┤
  │  Local cache │  ~1 μs   │  Per-JVM   │  Low (Caffeine)     │
  │  Redis       │  ~0.5ms  │  Cluster   │  Medium             │
  │  Memcached   │  ~0.3ms  │  Cluster   │  Low (simpler)      │
  │  Custom      │  Varies  │  Unlimited │  High (build it!)   │
  └─────────────────────────────────────────────────────────────┘
  
  BEST PRACTICE: L1 (local) + L2 (distributed) cache!
  Local cache: 100 MB, 5s TTL (ultra-fast, per-JVM)
  Distributed: 64 GB per node (shared, longer TTL)
```

---

## 🏗️ Architecture Design

```
TWO MAIN ARCHITECTURES:

ARCHITECTURE 1: CLIENT-SIDE SHARDING (Memcached-style)
  Client knows all cache nodes. Client does consistent hashing.
  
  ┌─────────────────┐
  │  Application    │  ← Client library has hash ring!
  │  (has hash ring)│
  └────────┬────────┘
           │ hash(key) → pick node
     ┌─────┼──────┬──────────┐
     ▼     ▼      ▼          ▼
  [Node1] [Node2] [Node3] [Node4]
  
  ✅ Simple, fast (one hop!)
  ❌ Client must know all nodes
  ❌ Hard to add/remove nodes (must update ALL clients)
  ❌ No automatic failover

ARCHITECTURE 2: PROXY-BASED (Redis Cluster-style)
  Clients talk to any node. Nodes redirect if wrong.
  
  ┌─────────────────┐
  │  Application    │  ← Talks to any node!
  └────────┬────────┘
           │
     ┌─────▼──────┐
     │  Any Node  │ → "This key belongs to Node3!"
     └─────┬──────┘   MOVED redirect
           │
     ┌─────▼──────┐
     │   Node 3   │ → Has the actual data!
     └────────────┘
  
  ✅ Client is simple (no hash ring knowledge)
  ✅ Cluster manages itself (add/remove nodes)
  ✅ Automatic failover (replica promotion!)
  ❌ Extra hop on miss (MOVED redirect)
  ❌ More complex cluster protocol

ARCHITECTURE 3: PROXY LAYER (Twemproxy/Envoy)
  Dedicated proxy handles routing.
  
  ┌─────────────────┐
  │  Application    │
  └────────┬────────┘
           │
  ┌────────▼────────┐
  │  Cache Proxy    │  ← Knows all nodes, routes requests
  │  (Twemproxy)    │
  └────────┬────────┘
     ┌─────┼──────┬──────────┐
     ▼     ▼      ▼          ▼
  [Node1] [Node2] [Node3] [Node4]
  
  ✅ Application is simple
  ✅ Centralized routing logic
  ❌ Proxy is a bottleneck & SPOF
  ❌ Extra network hop (added latency)
```

---

## 🔄 Data Partitioning

```
CONSISTENT HASHING FOR CACHE:

  Same as Distributed KV Store! But with cache-specific twists:
  
  Hash ring with virtual nodes:
  
  Node A (64GB) → 150 virtual nodes
  Node B (128GB) → 300 virtual nodes (more capacity = more vnodes!)
  Node C (64GB) → 150 virtual nodes
  
  hash("user:123") → walks clockwise → lands on Node B
  
  ADDING A NEW NODE:
  New Node D added → only keys in D's range move!
  
  But for CACHE: we don't need to move data! 🎉
  Why? Because it's a CACHE! 
  
  Old keys on wrong node → cache miss → loaded from DB → cached on new node!
  This is called "lazy migration" — data moves organically on access!
  
  Only problem: cold cache! Lots of misses right after adding a node.
  Solution: Warm up! Pre-load popular keys from DB when node joins.

HOTKEY PROBLEM:
  What if one key is accessed 1M times/second?
  (e.g., "trending_topics" or "celebrity_profile_taylorswift")
  
  One node handles ALL requests for that key → overloaded!
  
  Solutions:
  1. READ REPLICAS: Replicate hot key to multiple nodes
     "user:taylorswift" → on Node B + Node C + Node D
     Client randomly picks one → load distributed!
     
  2. LOCAL CACHE: Cache hot keys in application memory
     L1 (local, 5s TTL) → L2 (Redis) → DB
     10 app servers each cache locally → 10x less Redis load!
     
  3. KEY SPLITTING: Split into sub-keys
     "trending:0", "trending:1", "trending:2" (same data!)
     hash differently → different nodes → distribute load!
```

---

## 🗑️ Cache Eviction Policies

```
WHEN MEMORY IS FULL, WHAT DO WE EVICT?

  ┌────────────────────────────────────────────────────────────────┐
  │  Policy    │  Algorithm          │  Best For                   │
  ├────────────────────────────────────────────────────────────────┤
  │  LRU       │  Least Recently     │  General purpose            │
  │            │  Used: evict oldest  │  (good default!)            │
  │            │  access              │                             │
  ├────────────────────────────────────────────────────────────────┤
  │  LFU       │  Least Frequently   │  Popularity-based           │
  │            │  Used: evict lowest  │  workloads (CDN, profiles)  │
  │            │  access count        │                             │
  ├────────────────────────────────────────────────────────────────┤
  │  TTL       │  Time-To-Live       │  Time-sensitive data        │
  │            │  Expire after fixed  │  (sessions, tokens)         │
  │            │  duration            │                             │
  ├────────────────────────────────────────────────────────────────┤
  │  Random    │  Evict random entry  │  Uniform access patterns    │
  │            │  (surprisingly good!)│  (simpler to implement)     │
  ├────────────────────────────────────────────────────────────────┤
  │  W-TinyLFU │  Windowed Tiny LFU  │  Mixed workloads            │
  │  (Caffeine)│  Best hit ratio!    │  (industry state-of-art!)   │
  └────────────────────────────────────────────────────────────────┘

  LRU IMPLEMENTATION (Doubly-Linked List + HashMap):
  
  ┌─────────────────────────────────────────────┐
  │  HEAD (most recent) ←→ ... ←→ TAIL (oldest)│
  │                                             │
  │  Access "B": move B to HEAD                 │
  │  Evict: remove TAIL                         │
  │  Insert: add at HEAD, if full → evict TAIL  │
  └─────────────────────────────────────────────┘
  
  HashMap: O(1) lookup by key
  LinkedList: O(1) move-to-front, O(1) evict-tail
  Combined: O(1) for all cache operations!

  REDIS APPROACH (Approximated LRU):
  Don't maintain actual linked list (too expensive for millions of keys!)
  Instead: sample 5 random keys, evict the oldest among them.
  Surprisingly close to perfect LRU with much less overhead!
```

---

## 🔧 Cache Patterns

```
PATTERN 1: CACHE-ASIDE (Lazy Loading)
  Most common! Application manages cache explicitly.
  
  Read:
  1. Check cache → hit? Return!
  2. Cache miss → read from DB
  3. Write to cache (with TTL)
  4. Return to caller
  
  Write:
  1. Write to DB
  2. Invalidate cache (delete key!)
  3. Next read will re-populate cache
  
  ✅ Only caches what's actually requested
  ❌ Cache miss = slower (DB + cache write)
  ❌ Stale data possible (between DB write and cache invalidation)

PATTERN 2: WRITE-THROUGH
  Every write goes through cache to DB.
  
  Write:
  1. Write to cache
  2. Cache writes to DB (synchronous!)
  3. Return to caller after both succeed
  
  ✅ Cache always consistent with DB!
  ❌ Higher write latency (cache + DB on every write)
  ❌ Caches data that might never be read

PATTERN 3: WRITE-BEHIND (Write-Back)
  Write to cache immediately, async write to DB.
  
  Write:
  1. Write to cache → return immediately!
  2. Background: batch writes to DB (every 5 seconds)
  
  ✅ Lowest write latency!
  ✅ Batch writes = fewer DB operations
  ❌ Risk of data loss if cache crashes before DB write!
  ❌ Complex consistency guarantees

PATTERN 4: READ-THROUGH
  Cache itself handles DB reads (transparent to app).
  
  Read:
  1. Ask cache for key
  2. Cache miss → cache itself reads from DB
  3. Cache stores result and returns
  
  ✅ Application code is simpler
  ❌ Cache must know about DB (coupled)
```

---

## 🚨 Common Problems & Solutions

```
PROBLEM 1: CACHE STAMPEDE (Thundering Herd)
  Popular key expires → 10,000 requests simultaneously miss →
  10,000 DB queries for same data! 💀
  
  Solutions:
  a) LOCKING: First request acquires lock, fetches from DB, populates cache.
     Others wait or get stale data.
  b) EARLY REFRESH: Refresh cache BEFORE TTL expires (background)
  c) STALE-WHILE-REVALIDATE: Return stale data immediately,
     refresh in background.
  d) JITTERED TTL: Instead of all keys expiring at same time,
     TTL = base + random(0, 60s). Spread expiration!

PROBLEM 2: CACHE PENETRATION
  Query for key that DOESN'T EXIST in DB either!
  Every request: miss cache → miss DB → waste resources!
  (e.g., spam requests for fake user IDs)
  
  Solutions:
  a) CACHE NULL: Store null/empty result with short TTL (5 min)
     "user:fake123" → NULL (cached! Don't hit DB again!)
  b) BLOOM FILTER: Before cache check, verify key EXISTS in Bloom filter
     Bloom says "no" → definitely doesn't exist → return 404!

PROBLEM 3: CACHE AVALANCHE
  Many keys expire at SAME TIME → massive DB load!
  (e.g., all keys cached at server startup with same TTL)
  
  Solutions:
  a) JITTERED TTL: TTL = base + random(0, 120s)
  b) WARM UP: Pre-populate cache before traffic hits
  c) CIRCUIT BREAKER: If DB is overloaded → serve stale cache data!
  d) MULTI-LEVEL CACHE: L1 (local) shields L2 (distributed)

PROBLEM 4: INCONSISTENCY
  DB updated → cache still has old value → users see stale data!
  
  Solutions:
  a) TTL: Accept eventual consistency (expire in 5-60s)
  b) EVENT-DRIVEN: DB change → event → cache invalidation
  c) DOUBLE DELETE: Delete cache, update DB, wait 500ms, delete again
     (catches race condition where stale read re-populates between writes)
```

---

## 🔒 High Availability

```
REPLICATION FOR CACHE:

  Primary-Replica Model (Redis Sentinel / Redis Cluster):
  
  ┌──────────────────────────────────────────┐
  │  Shard 1                                  │
  │  ┌────────┐ ──repl──► ┌────────┐        │
  │  │Primary │            │Replica │        │
  │  │(writes)│ ──repl──► ┌────────┐        │
  │  └────────┘            │Replica │        │
  │                        └────────┘        │
  └──────────────────────────────────────────┘
  
  Writes → Primary only
  Reads → Primary OR any Replica (distributed load!)
  
  If Primary dies:
  1. Sentinel/Cluster detects failure (heartbeat timeout)
  2. Promote one Replica to new Primary
  3. Other replicas re-point to new Primary
  4. Clients reconnect (< 5 second failover!)
  
  DATA LOSS WINDOW:
  Replication is ASYNC! (sync would add latency)
  If Primary dies with unreplicated writes → lost!
  Window: ~100ms of writes (acceptable for cache!)
  
  For CRITICAL cache (sessions): use synchronous replication
  (slower but no data loss)

MULTI-REGION CACHE:
  US cluster ←─ async repl ─→ EU cluster
  
  User in US writes → US cluster → async → EU cluster (200ms later)
  User in EU reads → EU cluster (local, fast!)
  
  Consistency: eventual (acceptable for most cache use cases)
  For strong consistency: route to primary region (higher latency)
```

---

## 💻 Java Implementation

### Distributed Cache Client

```java
@Service
public class DistributedCacheService {
    
    @Autowired private StringRedisTemplate redis;
    @Autowired private CaffeineCache localCache; // L1
    
    private static final Duration DEFAULT_TTL = Duration.ofMinutes(30);
    private static final Duration JITTER_MAX = Duration.ofSeconds(60);
    
    /**
     * Two-level cache: L1 (local) → L2 (Redis) → DB
     */
    public <T> T get(String key, Class<T> type, Supplier<T> dbFallback) {
        // L1: Check local cache (Caffeine, ~1μs)
        T localResult = localCache.getIfPresent(key, type);
        if (localResult != null) return localResult;
        
        // L2: Check distributed cache (Redis, ~0.5ms)
        String json = redis.opsForValue().get(key);
        if (json != null) {
            T value = deserialize(json, type);
            localCache.put(key, value); // Populate L1
            return value;
        }
        
        // Cache miss! Load from DB with stampede protection
        return loadWithLock(key, type, dbFallback);
    }
    
    /**
     * Stampede protection: only ONE thread loads from DB.
     * Others wait or get stale data.
     */
    private <T> T loadWithLock(String key, Class<T> type, 
                               Supplier<T> dbFallback) {
        String lockKey = "lock:" + key;
        
        // Try to acquire distributed lock
        Boolean locked = redis.opsForValue()
            .setIfAbsent(lockKey, "1", Duration.ofSeconds(10));
        
        if (Boolean.TRUE.equals(locked)) {
            try {
                // I won the lock! Load from DB.
                T value = dbFallback.get();
                
                if (value != null) {
                    put(key, value);
                } else {
                    // Cache null to prevent penetration!
                    redis.opsForValue().set(key, "NULL", 
                        Duration.ofMinutes(5));
                }
                
                return value;
            } finally {
                redis.delete(lockKey);
            }
        } else {
            // Another thread is loading. Wait briefly then retry.
            sleep(50); // 50ms
            String json = redis.opsForValue().get(key);
            return json != null ? deserialize(json, type) : null;
        }
    }
    
    /**
     * Write with jittered TTL (prevents cache avalanche!)
     */
    public <T> void put(String key, T value) {
        Duration jitteredTTL = DEFAULT_TTL.plus(
            Duration.ofSeconds(ThreadLocalRandom.current()
                .nextLong(JITTER_MAX.toSeconds())));
        
        String json = serialize(value);
        redis.opsForValue().set(key, json, jitteredTTL);
        localCache.put(key, value);
    }
    
    /**
     * Invalidate across all levels.
     */
    public void invalidate(String key) {
        redis.delete(key);
        localCache.invalidate(key);
        // Broadcast invalidation to other app instances!
        redis.convertAndSend("cache-invalidation", key);
    }
}
```

### LRU Cache Implementation (From Scratch)

```java
public class LRUCache<K, V> {
    
    private final int capacity;
    private final Map<K, Node<K, V>> map;
    private final Node<K, V> head; // Most recently used
    private final Node<K, V> tail; // Least recently used
    
    static class Node<K, V> {
        K key;
        V value;
        Node<K, V> prev, next;
        long expireAt; // TTL support
        
        Node(K key, V value, long expireAt) {
            this.key = key;
            this.value = value;
            this.expireAt = expireAt;
        }
    }
    
    public LRUCache(int capacity) {
        this.capacity = capacity;
        this.map = new ConcurrentHashMap<>(capacity);
        this.head = new Node<>(null, null, 0); // Sentinel
        this.tail = new Node<>(null, null, 0); // Sentinel
        head.next = tail;
        tail.prev = head;
    }
    
    public synchronized V get(K key) {
        Node<K, V> node = map.get(key);
        if (node == null) return null;
        
        // Check TTL
        if (node.expireAt > 0 && System.currentTimeMillis() > node.expireAt) {
            remove(key);
            return null; // Expired!
        }
        
        // Move to front (most recently used!)
        moveToHead(node);
        return node.value;
    }
    
    public synchronized void put(K key, V value, Duration ttl) {
        Node<K, V> existing = map.get(key);
        long expireAt = ttl != null 
            ? System.currentTimeMillis() + ttl.toMillis() : 0;
        
        if (existing != null) {
            existing.value = value;
            existing.expireAt = expireAt;
            moveToHead(existing);
        } else {
            if (map.size() >= capacity) {
                evict(); // Remove LRU (tail)!
            }
            Node<K, V> newNode = new Node<>(key, value, expireAt);
            map.put(key, newNode);
            addToHead(newNode);
        }
    }
    
    private void evict() {
        Node<K, V> lru = tail.prev; // Least recently used!
        removeNode(lru);
        map.remove(lru.key);
    }
    
    private void moveToHead(Node<K, V> node) {
        removeNode(node);
        addToHead(node);
    }
    
    private void addToHead(Node<K, V> node) {
        node.next = head.next;
        node.prev = head;
        head.next.prev = node;
        head.next = node;
    }
    
    private void removeNode(Node<K, V> node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }
}
```

### Cache Invalidation via Events

```java
@Service
public class CacheInvalidationListener {
    
    @Autowired private DistributedCacheService cache;
    
    /**
     * Listen for database change events (CDC / domain events)
     * and invalidate corresponding cache entries.
     */
    @KafkaListener(topics = "db-changes")
    public void onDatabaseChange(ChangeEvent event) {
        String entityType = event.getTable();
        String entityId = event.getRecordId();
        
        // Determine which cache keys to invalidate
        List<String> keysToInvalidate = switch (entityType) {
            case "users" -> List.of(
                "user:" + entityId,
                "user-profile:" + entityId,
                "user-preferences:" + entityId
            );
            case "products" -> List.of(
                "product:" + entityId,
                "product-details:" + entityId,
                "category-products:" + event.getField("category_id")
            );
            default -> List.of(entityType + ":" + entityId);
        };
        
        keysToInvalidate.forEach(cache::invalidate);
        
        log.info("Invalidated {} cache keys for {}.{}", 
            keysToInvalidate.size(), entityType, entityId);
    }
    
    /**
     * Listen for cross-instance cache invalidation broadcasts.
     * When one app instance invalidates, all others must too!
     */
    @RedisListener(channel = "cache-invalidation")
    public void onCacheInvalidation(String key) {
        // Only invalidate LOCAL cache (Redis already handled!)
        localCache.invalidate(key);
    }
}
```

---

## 🎮 Mini Challenge

### 🧩 Design: Cache Warming Strategy

Your application deploys a new version with empty cache. How do you prevent a "cold cache" stampede where thousands of requests hit the database simultaneously?

<details>
<summary>🔑 Answer</summary>

```java
@Service
public class CacheWarmer {
    
    /**
     * Pre-warm cache before traffic hits (during deployment!)
     */
    @EventListener(ApplicationReadyEvent.class)
    public void warmUpCache() {
        log.info("Starting cache warm-up...");
        
        // 1. Load top-N most accessed keys from access log
        List<String> hotKeys = analyticsService.getTopKeys(10_000);
        
        // 2. Parallel warm-up (but don't overwhelm DB!)
        Semaphore dbLimiter = new Semaphore(50); // Max 50 concurrent DB reads
        
        hotKeys.parallelStream().forEach(key -> {
            try {
                dbLimiter.acquire();
                Object value = loadFromDatabase(key);
                if (value != null) {
                    cache.put(key, value);
                }
            } finally {
                dbLimiter.release();
            }
        });
        
        // 3. Gradual traffic ramp-up
        // Use load balancer to send 10% traffic initially
        // Increase to 100% over 5 minutes as cache fills
        
        log.info("Cache warm-up complete: {} keys loaded", hotKeys.size());
    }
}
```

**Additional strategies:**
- **Shadow traffic:** Before cutover, replay production traffic to new cache
- **Redis persistence:** Use RDB/AOF so cache survives restarts
- **Blue-green cache:** Keep old cache running until new one is warm
- **Request coalescing:** Multiple threads waiting for same key → only one loads from DB
</details>

---

## ❓ Interview Q&A

**Q1: How do you handle cache invalidation in a distributed system?**
> Three approaches: (1) TTL-based — accept staleness for duration of TTL (simplest, best for most cases), (2) Event-driven — CDC events from database trigger cache invalidation (lower staleness, more complex), (3) Write-through — cache is always updated on write (consistent but slower writes). In practice: use TTL as safety net + event-driven invalidation for critical data. Broadcast invalidation to all app instances via Redis Pub/Sub or Kafka.

**Q2: Explain the cache stampede problem and your solution.**
> When a popular key expires, hundreds of concurrent requests all miss cache simultaneously and hit the database. Solutions: (1) Distributed lock — first requester acquires lock, loads from DB, others wait 50ms then retry from cache. (2) Stale-while-revalidate — return expired cached value immediately while refreshing in background. (3) Background refresh — refresh keys BEFORE they expire (probabilistic early expiration). I prefer option 1 for correctness + option 3 for hot keys.

**Q3: When would you use local cache vs distributed cache?**
> Local cache (Caffeine/Guava): ultra-fast (nanoseconds), per-JVM, inconsistent across instances, limited by heap size. Best for: hot keys, immutable data, computed values. Distributed cache (Redis): fast (sub-ms), shared across instances, consistent, larger capacity. Best for: session data, user profiles, shared state. Best practice: BOTH! L1 (local, 5-30s TTL) + L2 (Redis, 5-30min TTL). L1 absorbs 80%+ of reads, L2 handles the rest.

**Q4: How do you choose between Redis and Memcached?**
> Redis: data structures (hashes, lists, sorted sets), persistence, pub/sub, Lua scripting, replication, cluster mode. Choose for: sessions, leaderboards, rate limiting, pub/sub. Memcached: simpler, multi-threaded (better for raw throughput), pure key-value only, no persistence. Choose for: simple caching workloads, where you need maximum throughput and don't need data structures. At Facebook scale: Memcached for simple caching (5B ops/sec), Redis for complex data structures.

**Q5: How do you handle cache in a multi-region deployment?**
> Options: (1) Independent caches per region (simple, but inconsistent — user sees different data depending on region!), (2) Write to primary region's cache, async-replicate to others (consistent writes, eventual reads), (3) Cache-aside with regional DBs (each region has own DB replica + own cache, fed by replication). I prefer option 2 for most cases: write to origin cache, 100-200ms later other regions are updated. For writes: always route to primary region for consistency.

---

## 🔗 Related Topics
- [Caching Strategies](../../BuildingBlocks/CachingStrategies.md) — Read/write-through patterns
- [Consistent Hashing](../../KeyConcepts/ConsistentHashing.md) — How keys map to cache nodes
- [Redis Deep Dive](../../Database/Redis_Deep_Dive.md) — Redis internals & data structures
- [Bloom Filters](../../Database/BloomFilters.md) — Preventing cache penetration

---

*"There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton. After building a distributed cache, you realize he was being generous. It's just cache invalidation. Naming things is easy by comparison.* 💾
