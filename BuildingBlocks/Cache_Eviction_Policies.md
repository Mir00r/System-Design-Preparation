# 🗑️ Cache Eviction Policies

> *"Your cache is FULL and a new item needs to go in. Which existing item do you kick out? This seemingly simple decision can make or break your system's performance. Choose wrong and your cache becomes useless — constantly evicting items that'll be needed again immediately. Choose right and your cache has a near-magical hit ratio that makes your database barely break a sweat."*

**⏱️ Estimated Time**: 20 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Caching](Caching.md), [Caching Strategies](CachingStrategies.md)

---

## 📋 Table of Contents
1. [Why Eviction Matters](#-why-eviction-matters)
2. [Common Eviction Policies](#-common-eviction-policies)
3. [LRU Deep Dive](#-lru-deep-dive)
4. [LFU Deep Dive](#-lfu-deep-dive)
5. [Advanced Policies](#-advanced-policies)
6. [Java Implementation](#-java-implementation)
7. [Choosing the Right Policy](#-choosing-the-right-policy)
8. [Interview Q&A](#-interview-qa)

---

## 🎯 Why Eviction Matters

```
PROBLEM:
  Cache size: 1 GB (limited by cost/RAM!)
  Dataset size: 100 GB (all data in DB)
  Only 1% of data fits in cache!
  
  When cache is FULL and new data arrives:
  → Must EVICT something to make room!
  → Bad eviction = cache miss = slow DB query = user waits!

GOAL: Maximize HIT RATIO (% of requests served from cache)
  
  Hit ratio: 95% → 1 DB query per 20 requests (great!)
  Hit ratio: 50% → 1 DB query per 2 requests (terrible!)
  
  The eviction policy determines which items stay vs leave!
  
  ANALOGY: Your desk can hold 10 books. You have 100 books on a shelf.
  When you need book #11, which book do you put back?
  • The one you used longest ago? (LRU)
  • The one you rarely use? (LFU)
  • The oldest one? (FIFO)
  • A random one? (Random!)
```

---

## 📊 Common Eviction Policies

```
┌────────────────────────────────────────────────────────────────────────┐
│  Policy       │  Evicts            │  Best For          │  Used By    │
├────────────────────────────────────────────────────────────────────────┤
│  LRU          │  Least Recently    │  General purpose!  │  Redis,     │
│  (Least       │  Used              │  Recency matters   │  Memcached, │
│   Recently    │                    │                    │  CPU cache  │
│   Used)       │                    │                    │             │
├────────────────────────────────────────────────────────────────────────┤
│  LFU          │  Least Frequently  │  Popularity-based  │  Redis 4.0+ │
│  (Least       │  Used              │  workloads         │  CDN edge   │
│   Frequently  │                    │                    │  caches     │
│   Used)       │                    │                    │             │
├────────────────────────────────────────────────────────────────────────┤
│  FIFO         │  First In,         │  Simple, streaming │  Kafka page │
│  (First In    │  First Out         │  Sequential data   │  cache      │
│   First Out)  │                    │                    │             │
├────────────────────────────────────────────────────────────────────────┤
│  TTL          │  Expired items     │  Time-sensitive    │  DNS cache, │
│  (Time To     │  (oldest first)    │  data              │  HTTP cache │
│   Live)       │                    │                    │             │
├────────────────────────────────────────────────────────────────────────┤
│  Random       │  Random item!      │  Uniform access    │  ARM CPU    │
│               │                    │  patterns          │  TLB cache  │
├────────────────────────────────────────────────────────────────────────┤
│  ARC          │  Adaptive          │  Changing          │  ZFS,       │
│  (Adaptive    │  (learns LRU vs    │  workloads         │  IBM DS     │
│   Replacement │  LFU balance!)     │                    │             │
│   Cache)      │                    │                    │             │
└────────────────────────────────────────────────────────────────────────┘

COMPARISON BY HIT RATIO (typical workloads):
  ARC ≈ LRU-K > LFU > LRU > FIFO > Random
  
  But! It depends on workload:
  • LRU wins for recency-biased (web sessions, user profiles)
  • LFU wins for popularity-biased (CDN, trending content)
  • FIFO wins for streaming/sequential (log processing)
  • Random wins when nothing else works (uniform distribution)
```

---

## 🔄 LRU Deep Dive

```
LRU: "If you haven't been used recently, you're probably not needed."

HOW IT WORKS:
  Items ordered by last access time.
  Evict: the item accessed LONGEST AGO.
  
  Access sequence: A, B, C, D, A, E (cache size = 3)
  
  Step 1: [A]          → Miss, add A
  Step 2: [B, A]       → Miss, add B (A shifts back)
  Step 3: [C, B, A]    → Miss, add C (full!)
  Step 4: [D, C, B]    → Miss, evict A (LRU!), add D
  Step 5: [A, D, C]    → Miss, evict B (LRU!), add A
  Step 6: [E, A, D]    → Miss, evict C (LRU!), add E

DATA STRUCTURE: HashMap + Doubly Linked List = O(1) operations!
  
  HashMap: key → node pointer (O(1) lookup!)
  LinkedList: maintains access order (O(1) move to front!)
  
  ┌─────────────────────────────────────────────┐
  │  HashMap                                     │
  │  "user:1" → ●──────►[User1|prev|next]      │
  │  "user:2" → ●──────►[User2|prev|next]      │
  │  "user:3" → ●──────►[User3|prev|next]      │
  └─────────────────────────────────────────────┘
  
  Linked List (most recent → least recent):
  HEAD ↔ [User3] ↔ [User1] ↔ [User2] ↔ TAIL
                                         ↑ evict this!

OPERATIONS:
  GET(key):  O(1) → HashMap lookup + move node to HEAD
  PUT(key):  O(1) → Add to HEAD. If full: remove TAIL + HashMap entry
  
WEAKNESS: 
  "Cache pollution" — a full table scan evicts ALL useful items!
  One-time access to many items pushes out frequently-used items.
  Solution: LRU-K (requires K accesses before promoting) or ARC!
```

---

## 📈 LFU Deep Dive

```
LFU: "If you're not popular, you get evicted."

HOW IT WORKS:
  Each item has an access COUNT.
  Evict: item with LOWEST count (least popular!)
  Tie-breaker: evict the least recently used among lowest-count items.

EXAMPLE (cache size = 3):
  Access: A, A, A, B, B, C, D
  
  After A,A,A: {A:3}
  After B,B:   {A:3, B:2}
  After C:     {A:3, B:2, C:1} (full!)
  After D:     {A:3, B:2, D:1} ← Evict C (count=1, oldest with min count)!

DATA STRUCTURE: O(1) LFU implementation!
  
  Frequency Map:
  ┌──────────────────────────────────────────┐
  │ freq=1: [item_C] → [item_D] (linked list)│
  │ freq=2: [item_B]                          │
  │ freq=3: [item_A]                          │
  └──────────────────────────────────────────┘
  min_freq = 1 → evict from freq=1 list (tail = LRU within freq)!

WEAKNESSES:
  • "Frequency stagnation": old popular item stays forever
    even after it's no longer relevant!
    (Item was popular YESTERDAY but irrelevant TODAY)
  • Solution: decay/aging — periodically halve all counts!
    Or: Window-TinyLFU (Caffeine uses this!)

REDIS LFU (since 4.0):
  Uses logarithmic counter (8 bits! Caps at ~1M accesses)
  + decay factor (counter decreases over time)
  → Combines frequency AND recency!
```

---

## 🚀 Advanced Policies

```
W-TinyLFU (Caffeine cache — best Java cache!):
  ┌──────────────────────────────────────────────────────────┐
  │  Admission Filter                                         │
  │  "Should this NEW item enter main cache?"                │
  │                                                           │
  │  Window Cache (1%) ──► Candidate                         │
  │                           │                               │
  │                    TinyLFU decides:                        │
  │                    Is candidate more popular than          │
  │                    the LRU victim in main cache?           │
  │                    YES → admit candidate, evict victim!    │
  │                    NO → reject candidate!                  │
  │                                                           │
  │  Main Cache (99%) = Segmented LRU                         │
  │    Probation (20%) → Protected (80%)                      │
  │    Access in probation promotes to protected!              │
  └──────────────────────────────────────────────────────────┘
  
  Why it's amazing:
  • Doesn't cache one-hit-wonders (scan resistance!)
  • Frequency sketching uses Count-Min Sketch (tiny memory!)
  • Adapts window size based on workload!
  • Near-optimal hit ratios for most workloads!

ARC (Adaptive Replacement Cache):
  Maintains 4 lists:
  • T1: recent items (accessed once)
  • T2: frequent items (accessed 2+)  
  • B1: ghost list (recently evicted from T1)
  • B2: ghost list (recently evicted from T2)
  
  Adaptation: 
  • Cache miss in B1? → workload is recency-biased! Grow T1!
  • Cache miss in B2? → workload is frequency-biased! Grow T2!
  • Automatically balances between LRU and LFU behavior!
```

---

## 💻 Java Implementation

### LRU Cache (O(1) Operations)

```java
public class LRUCache<K, V> {
    
    private final int capacity;
    private final Map<K, Node<K, V>> map;
    private final Node<K, V> head; // Most recently used
    private final Node<K, V> tail; // Least recently used (evict!)
    
    public LRUCache(int capacity) {
        this.capacity = capacity;
        this.map = new HashMap<>(capacity);
        head = new Node<>(null, null);
        tail = new Node<>(null, null);
        head.next = tail;
        tail.prev = head;
    }
    
    public V get(K key) {
        Node<K, V> node = map.get(key);
        if (node == null) return null; // Cache miss!
        
        // Move to front (most recently used!)
        removeNode(node);
        addToFront(node);
        return node.value;
    }
    
    public void put(K key, V value) {
        Node<K, V> existing = map.get(key);
        
        if (existing != null) {
            existing.value = value;
            removeNode(existing);
            addToFront(existing);
        } else {
            if (map.size() >= capacity) {
                // Evict LRU (tail.prev)!
                Node<K, V> lru = tail.prev;
                removeNode(lru);
                map.remove(lru.key);
            }
            Node<K, V> newNode = new Node<>(key, value);
            addToFront(newNode);
            map.put(key, newNode);
        }
    }
    
    private void addToFront(Node<K, V> node) {
        node.next = head.next;
        node.prev = head;
        head.next.prev = node;
        head.next = node;
    }
    
    private void removeNode(Node<K, V> node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }
    
    static class Node<K, V> {
        K key; V value;
        Node<K, V> prev, next;
        Node(K key, V value) { this.key = key; this.value = value; }
    }
}
```

### Using Java's LinkedHashMap (Simple LRU!)

```java
// Java's LinkedHashMap with accessOrder=true IS an LRU cache!
public class SimpleLRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int maxSize;
    
    public SimpleLRUCache(int maxSize) {
        super(maxSize, 0.75f, true); // accessOrder=true!
        this.maxSize = maxSize;
    }
    
    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > maxSize; // Evict when full!
    }
}
```

### Using Caffeine (W-TinyLFU — Production!)

```java
// Best cache library for Java! Near-optimal hit ratios.
Cache<String, User> cache = Caffeine.newBuilder()
    .maximumSize(10_000)           // Evicts via W-TinyLFU!
    .expireAfterWrite(5, TimeUnit.MINUTES)  // TTL!
    .recordStats()                  // Hit ratio metrics!
    .build();

// Get or load
User user = cache.get("user:123", 
    key -> userRepository.findById(key)); // Load on miss!

// Stats
CacheStats stats = cache.stats();
log.info("Hit ratio: {}", stats.hitRate());  // Aim for >90%!
```

---

## 🎯 Choosing the Right Policy

```
DECISION GUIDE:

  What's your access pattern?
  │
  ├── Recently accessed = likely accessed again?
  │   └── YES → LRU (web sessions, user profiles, API responses)
  │
  ├── Some items MUCH more popular than others?
  │   └── YES → LFU (CDN content, trending posts, hot products)
  │
  ├── Data has natural expiration (freshness matters)?
  │   └── YES → TTL-based (DNS, OAuth tokens, config)
  │
  ├── Sequential/streaming access?
  │   └── YES → FIFO or no cache! (log processing, ETL)
  │
  ├── Unknown or changing patterns?
  │   └── YES → W-TinyLFU (Caffeine) or ARC (adapts!)
  │
  └── Memory extremely constrained?
      └── YES → Random (no metadata overhead! Surprisingly OK!)

REDIS EVICTION POLICIES:
  ┌──────────────────────────────────────────────────────────┐
  │  noeviction     │  Return error when memory full!       │
  │  allkeys-lru    │  LRU across ALL keys (most common!)   │
  │  volatile-lru   │  LRU only among keys with TTL set     │
  │  allkeys-lfu    │  LFU across all keys (Redis 4.0+)     │
  │  volatile-lfu   │  LFU only among keys with TTL         │
  │  allkeys-random │  Random eviction                       │
  │  volatile-random│  Random among keys with TTL           │
  │  volatile-ttl   │  Evict keys with shortest TTL         │
  └──────────────────────────────────────────────────────────┘
  
  Default: noeviction (fails on full! Set explicitly!)
  Recommended: allkeys-lru or allkeys-lfu for most use cases.
```

---

## ❓ Interview Q&A

**Q1: Why is LRU the most popular cache eviction policy?**
> LRU exploits temporal locality — recently accessed items are likely to be accessed again soon. It's simple to understand, O(1) to implement (HashMap + doubly linked list), and works well for most workloads (web apps, user sessions, API caches). It's the default in Redis, Memcached, CPU caches, and most application caches. Its weakness is scan pollution — a one-time full scan evicts all useful items. Solutions: LRU-K (needs K hits) or W-TinyLFU (admission filter).

**Q2: Implement an LRU cache with O(1) get and put.**
> HashMap<Key, Node> for O(1) lookup + Doubly Linked List for O(1) reordering. On GET: lookup in map, move node to head (MRU position). On PUT: if key exists, update and move to head. If new: add to head, if over capacity evict tail (LRU) and remove from map. Sentinel head/tail nodes simplify edge cases. In Java, LinkedHashMap with accessOrder=true gives this for free!

**Q3: When would you use LFU instead of LRU?**
> When access frequency matters more than recency. Examples: (1) CDN caching — popular content (viral video) should stay cached even if not accessed in last 5 minutes, (2) Database query cache — frequently-run queries should be prioritized, (3) Product catalog — bestsellers accessed constantly should never be evicted. Caveat: pure LFU suffers from stale popular items (were popular yesterday, not today). Solution: use decay/aging to reduce stale counts, or Redis's LFU with configurable decay.

**Q4: What is cache pollution and how do you prevent it?**
> Cache pollution: one-time accesses fill the cache, evicting useful items. Example: batch job scans ALL users → pushes out hot user profiles → hit ratio drops to 0%! Prevention: (1) W-TinyLFU admission filter — new items must prove popularity before entering main cache, (2) LRU-K — only cache after K accesses (not on first access), (3) Segmented LRU — new items go to probation, only promoted to protected on re-access, (4) Separate caches for batch vs interactive workloads!

---

## 🔗 Related Topics
- [Caching](Caching.md) — Caching fundamentals
- [Caching Strategies](CachingStrategies.md) — Write-through, write-back, etc.
- [Redis Deep Dive](../Database/Redis_Deep_Dive.md) — Redis eviction config
- [Distributed Caching](Distributed_Caching.md) — Multi-node cache eviction

---

*"The best cache eviction policy is the one that keeps the items you'll need NEXT. Since we can't predict the future (Bélády's optimal algorithm requires a crystal ball!), LRU is our best practical approximation. But for those who want near-optimal: Caffeine's W-TinyLFU is as close to the crystal ball as we've gotten." — Cache Engineer* 🗑️
