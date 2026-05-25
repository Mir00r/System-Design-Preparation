# 🗂️ Java LinkedHashMap & TreeMap: Ordered Maps Masterclass 🚀

---

## 🎯 Chapter 7: LinkedHashMap & TreeMap

> **"If HashMap is the sports car (fastest, no order), LinkedHashMap is the GPS-equipped sedan (fast AND knows where it's been), and TreeMap is the meticulous librarian (always sorted, range queries in a blink)."**

---

## 📋 Table of Contents

1. [LinkedHashMap — The Ordered HashMap](#linkedhashmap)
2. [LinkedHashMap Internal Architecture](#linkedhashmap-architecture)
3. [LinkedHashMap — Access Order vs Insertion Order](#access-vs-insertion-order)
4. [LRU Cache with LinkedHashMap](#lru-cache)
5. [TreeMap — The Sorted Map](#treemap)
6. [TreeMap Internal Architecture](#treemap-architecture)
7. [TreeMap All Methods](#treemap-all-methods)
8. [WeakHashMap & IdentityHashMap (Special Maps)](#special-maps)
9. [Map Comparison Table](#map-comparison)
10. [Time & Space Complexity](#time--space-complexity)
11. [Common Pitfalls](#common-pitfalls)
12. [Industry Applications](#industry-applications)
13. [Practice Problems](#practice-problems)
14. [Interview Tips](#interview-tips)

---

## 🔵 LinkedHashMap — The Ordered HashMap

### 📖 **What is LinkedHashMap?**

`LinkedHashMap<K,V>` extends `HashMap<K,V>` and maintains a **doubly-linked list** running through all its entries. This linked list defines either **insertion order** (default) or **access order** for iteration.

```
LinkedHashMap characteristics:
✅ All HashMap operations (O(1) get, put, remove)
✅ Maintains PREDICTABLE ITERATION ORDER:
   - Insertion order (default): keys iterate in order they were inserted
   - Access order (optional): keys iterate in order of last access (LRU!)
✅ ONE null key, multiple null values
❌ Slightly more memory than HashMap (extra prev/next pointers)
❌ Slightly slower than HashMap (must maintain linked list on every op)
❌ NOT thread-safe

Introduced: Java 1.4 (2002)
Extends: HashMap — all HashMap methods inherited
```

---

## 🏗️ LinkedHashMap Architecture

### 🔧 **The Linked Entry**

```java
// LinkedHashMap.Entry extends HashMap.Node and adds prev/next:
static class Entry<K,V> extends HashMap.Node<K,V> {
    Entry<K,V> before;  // previous entry in the linked list
    Entry<K,V> after;   // next entry in the linked list
    
    Entry(int hash, K key, V value, Node<K,V> next) {
        super(hash, key, value, next);
    }
}

// LinkedHashMap also tracks head/tail:
transient LinkedHashMap.Entry<K,V> head; // earliest entry (insertion or least recently used)
transient LinkedHashMap.Entry<K,V> tail; // latest entry (last inserted or most recently used)

final boolean accessOrder; // false=insertion order, true=access order
```

```
MEMORY LAYOUT: LinkedHashMap { "a"→1, "b"→2, "c"→3 } (insertion order)

Hash Table (same as HashMap):             Insertion-Order Doubly-Linked List:
┌────────────────────────────────┐         head                        tail
│ [2] → Entry{key="a", value=1} │    ┌──────────┐   ┌──────────┐   ┌──────────┐
│ [5] → Entry{key="b", value=2} │    │ "a"→1    │⟷ │ "b"→2    │⟷ │ "c"→3    │
│ [8] → Entry{key="c", value=3} │    └──────────┘   └──────────┘   └──────────┘
└────────────────────────────────┘

Each Entry node is in BOTH structures simultaneously!
  - In the hash table: via HashMap's Node chain (for O(1) access)
  - In the linked list: via before/after pointers (for ordered iteration)
```

---

## 🔄 Access Order vs Insertion Order

### 🔧 **Insertion Order (Default)**

```java
// Default constructor — insertion order:
LinkedHashMap<String, Integer> map = new LinkedHashMap<>();
map.put("banana", 3);
map.put("apple", 1);
map.put("cherry", 2);

// Iterates in insertion order regardless of access:
map.get("banana"); // accessing banana...
map.get("banana"); // again...

System.out.println(map.keySet()); // [banana, apple, cherry] — insertion order!

// Putting an existing key doesn't change its position:
map.put("apple", 100);
System.out.println(map.keySet()); // [banana, apple, cherry] — apple stays in place!
```

### 🔧 **Access Order (LRU mode)**

```java
// accessOrder=true → recently accessed entries move to TAIL
LinkedHashMap<String, Integer> accessOrdered = new LinkedHashMap<>(16, 0.75f, true);
accessOrdered.put("a", 1);
accessOrdered.put("b", 2);
accessOrdered.put("c", 3);
// List: a ↔ b ↔ c

accessOrdered.get("a");         // ACCESS "a" → moves to tail
// List: b ↔ c ↔ a

accessOrdered.get("b");         // ACCESS "b" → moves to tail
// List: c ↔ a ↔ b

accessOrdered.put("a", 99);     // PUT also counts as access → moves to tail
// List: c ↔ b ↔ a (a moved to tail)

System.out.println(accessOrdered.keySet()); // [c, b, a]
// head = c (least recently accessed), tail = a (most recently accessed)
```

---

## 🏆 LRU Cache with LinkedHashMap

### 🔧 **The Classic Implementation**

```java
// The most elegant LRU Cache in Java:
public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int maxSize;
    
    public LRUCache(int maxSize) {
        super(maxSize, 0.75f, true); // true = access order (LRU mode!)
        this.maxSize = maxSize;
    }
    
    // Called AUTOMATICALLY by LinkedHashMap after every put()
    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > maxSize; // evict when over capacity
        // eldest = the head node = least recently accessed entry
    }
    
    // Optional: thread-safe wrapper
    public synchronized V get(K key) { return super.get(key); }
    public synchronized V put(K key, V value) { return super.put(key, value); }
}

// Usage:
LRUCache<Integer, String> cache = new LRUCache<>(3);
cache.put(1, "one");
cache.put(2, "two");
cache.put(3, "three");
// Cache: {1=one, 2=two, 3=three} (insertion order)

cache.get(1);           // access 1 → moves to most recently used
// Access order: 2, 3, 1 (2 is LRU)

cache.put(4, "four");   // capacity exceeded → evicts 2 (LRU = head)
// Cache: {3=three, 1=one, 4=four}

cache.get(3); // null? NO! 3 is still there: returns "three"
```

### 🔧 **LRU Cache for LeetCode 146**

```java
// Full production-quality LRU Cache:
class LRUCache {
    private final Map<Integer, Integer> cache;
    private final int capacity;
    
    public LRUCache(int capacity) {
        this.capacity = capacity;
        // accessOrder=true, capacity sets initial size to avoid resize
        this.cache = new LinkedHashMap<>(capacity, 0.75f, true) {
            @Override
            protected boolean removeEldestEntry(Map.Entry<Integer, Integer> eldest) {
                return size() > capacity;
            }
        };
    }
    
    public int get(int key) {
        return cache.getOrDefault(key, -1); // getOrDefault triggers access order update
    }
    
    public void put(int key, int value) {
        cache.put(key, value);
    }
}
```

---

## 🌳 TreeMap — The Sorted Map

### 📖 **What is TreeMap?**

`TreeMap<K,V>` is a `NavigableMap` implementation backed by a **Red-Black Tree**. All keys are kept in **sorted order** (natural or custom Comparator), enabling efficient range operations.

```
TreeMap characteristics:
✅ Keys kept SORTED at all times (natural order or Comparator)
✅ O(log n) get, put, remove
✅ Rich navigation: floorKey, ceilingKey, headMap, tailMap, subMap
✅ No null keys (compareTo fails on null)
✅ Null values allowed
❌ Slower than HashMap (O(log n) vs O(1))
❌ NOT thread-safe

Introduced: Java 1.2 (1998), NavigableMap added in Java 6
```

---

## 🏗️ TreeMap Architecture

```java
// TreeMap uses the SAME Red-Black Tree as TreeSet
// (TreeSet is backed by TreeMap — full circle!)

public class TreeMap<K,V> extends AbstractMap<K,V>
        implements NavigableMap<K,V>, Cloneable, Serializable {

    private final Comparator<? super K> comparator; // null if natural ordering
    private transient Entry<K,V> root; // root of the Red-Black Tree
    private transient int size;
    
    // Tree node
    static final class Entry<K,V> implements Map.Entry<K,V> {
        K key;
        V value;
        Entry<K,V> left;
        Entry<K,V> right;
        Entry<K,V> parent;
        boolean color = BLACK; // RED or BLACK for rebalancing
    }
}
```

---

## 🛠️ TreeMap All Methods

### 🔧 **Basic CRUD**

```java
TreeMap<String, Integer> map = new TreeMap<>();
map.put("cherry", 3);
map.put("apple", 1);
map.put("banana", 2);
map.put("date", 4);
// Internal order: [apple, banana, cherry, date]

// get/put/remove — same as HashMap API
map.get("apple");           // 1 — O(log n) BST search
map.put("elderberry", 5);   // O(log n) insert + rebalance
map.remove("banana");       // O(log n) delete + rebalance
map.containsKey("cherry");  // true — O(log n)
map.containsValue(3);       // true — O(n) linear scan
map.size();                 // 4
```

### 🧭 **SortedMap / NavigableMap Methods**

```java
TreeMap<Integer, String> map = new TreeMap<>();
map.put(10, "ten");   map.put(20, "twenty"); map.put(30, "thirty");
map.put(40, "forty"); map.put(50, "fifty");

// ── FIND BOUNDARY KEYS ──
map.firstKey();          // 10 — minimum
map.lastKey();           // 50 — maximum
map.firstEntry();        // Map.Entry{10="ten"}
map.lastEntry();         // Map.Entry{50="fifty"}

// ── NAVIGATION (same semantics as TreeSet) ──
map.floorKey(35);        // 30 — greatest key ≤ 35
map.floorEntry(35);      // Entry{30="thirty"}

map.ceilingKey(35);      // 40 — smallest key ≥ 35
map.ceilingEntry(35);    // Entry{40="forty"}

map.lowerKey(30);        // 20 — greatest key STRICTLY < 30
map.higherKey(30);       // 40 — smallest key STRICTLY > 30

// ── POLLING (destructive) ──
Map.Entry<Integer,String> min = map.pollFirstEntry(); // removes {10="ten"}
Map.Entry<Integer,String> max = map.pollLastEntry();  // removes {50="fifty"}

// ── RANGES (returns LIVE VIEWS backed by TreeMap) ──
// headMap: all keys strictly < toKey
SortedMap<Integer, String> head = map.headMap(30);     // {20="twenty"}
NavigableMap<Integer, String> head2 = map.headMap(30, true); // inclusive: {20,30}

// tailMap: all keys >= fromKey
SortedMap<Integer, String> tail = map.tailMap(30);     // {30,40}
NavigableMap<Integer, String> tail2 = map.tailMap(30, false); // exclusive: {40}

// subMap: fromKey (inclusive) to toKey (exclusive)
SortedMap<Integer, String> sub = map.subMap(20, 40);   // {20,30}
NavigableMap<Integer, String> sub2 = map.subMap(20, true, 40, true); // {20,30,40}

// ── REVERSED VIEW ──
NavigableMap<Integer, String> desc = map.descendingMap(); // reversed key order
NavigableSet<Integer> descKeys = map.descendingKeySet();  // reversed key set

// ── COMPARATOR ──
Comparator<Integer> comp = map.comparator(); // null for natural ordering
```

### 🔧 **Map Iteration (all approaches)**

```java
TreeMap<String, Integer> map = new TreeMap<>(Map.of("b",2, "a",1, "c",3));
// Always iterates in KEY ORDER: a,b,c

// 1. Sorted entry iteration (most complete)
for (Map.Entry<String, Integer> entry : map.entrySet()) {
    System.out.println(entry.getKey() + " → " + entry.getValue());
}
// Output: a→1, b→2, c→3 (sorted!)

// 2. Sorted key iteration
for (String key : map.keySet()) { ... } // a, b, c

// 3. Values in key-sorted order
for (Integer val : map.values()) { ... } // 1, 2, 3

// 4. forEach (Java 8+)
map.forEach((k, v) -> System.out.println(k + "=" + v));

// 5. Stream with sorted entries
map.entrySet().stream()
   .filter(e -> e.getValue() > 1)
   .collect(Collectors.toMap(
       Map.Entry::getKey, 
       Map.Entry::getValue, 
       (a,b) -> a,          // merge function (never called for unique keys)
       TreeMap::new          // preserve sorted order in result!
   ));
```

### 🔧 **Range Queries — Real Use Cases**

```java
// PRICE RANGE QUERY
TreeMap<Double, Product> priceIndex = new TreeMap<>();
priceIndex.put(9.99, new Product("Budget Item"));
priceIndex.put(29.99, new Product("Mid-range"));
priceIndex.put(99.99, new Product("Premium"));
priceIndex.put(199.99, new Product("Luxury"));

// Products between $20 and $100:
SortedMap<Double, Product> affordable = priceIndex.subMap(20.0, true, 100.0, true);
// {29.99=Mid-range, 99.99=Premium}

// Products under $50:
SortedMap<Double, Product> budget = priceIndex.headMap(50.0, false);
// {9.99=Budget Item, 29.99=Mid-range}

// Most expensive product:
Product mostExpensive = priceIndex.lastEntry().getValue(); // Luxury
```

---

## 🔮 Special Maps

### 🔧 **WeakHashMap — Garbage Collection Friendly**

```java
// WeakHashMap uses WEAK REFERENCES for keys
// Entries are automatically removed when key has no strong references!

Map<Object, String> weakMap = new WeakHashMap<>();
Object key = new Object();  // strong reference to key
weakMap.put(key, "metadata");

System.out.println(weakMap.size()); // 1

key = null;  // remove the only strong reference to key!
System.gc(); // suggest garbage collection

Thread.sleep(100);
System.out.println(weakMap.size()); // 0! Entry was garbage collected!

// USE CASES:
// ✅ Metadata/listener attached to objects you don't own
// ✅ Caches where entries should expire when key is no longer used
// ✅ JVM class loaders (Spring uses WeakHashMap for class metadata caching)
// ❌ Don't use if you want entries to persist!
```

### 🔧 **IdentityHashMap — Reference Equality**

```java
// IdentityHashMap uses == instead of .equals() for key comparison!
// Two keys are "equal" only if they are the SAME OBJECT in memory

IdentityHashMap<String, Integer> idMap = new IdentityHashMap<>();

String s1 = new String("hello");
String s2 = new String("hello"); // different object, same value!

idMap.put(s1, 1);
idMap.put(s2, 2); // NOT a duplicate! s1 != s2 (different references)

System.out.println(idMap.size()); // 2! Both stored as DIFFERENT keys

// vs HashMap:
HashMap<String, Integer> hashMap = new HashMap<>();
hashMap.put(s1, 1);
hashMap.put(s2, 2); // s1.equals(s2) is true → treated as SAME key
System.out.println(hashMap.size()); // 1

// USE CASES:
// ✅ Object identity tracking (which specific instances are processed)
// ✅ Serialization frameworks (tracking which objects are serialized)
// ✅ Traversal algorithms where you need to track visited object instances
// ✅ JVM internals: tracking object references
```

---

## 📊 Map Comparison Table

```
FEATURE              │ HashMap      │ LinkedHashMap │ TreeMap      │ Hashtable   │ ConcurrentHashMap
─────────────────────┼──────────────┼───────────────┼──────────────┼─────────────┼──────────────────
Ordering             │ None         │ Insertion/    │ Sorted by    │ None        │ None
                     │              │ Access order  │ key          │             │
─────────────────────┼──────────────┼───────────────┼──────────────┼─────────────┼──────────────────
get/put performance  │ O(1)         │ O(1)          │ O(log n)     │ O(1)        │ O(1)
─────────────────────┼──────────────┼───────────────┼──────────────┼─────────────┼──────────────────
Null keys            │ ONE allowed  │ ONE allowed   │ NOT allowed  │ NOT allowed │ NOT allowed
Null values          │ Allowed      │ Allowed       │ Allowed      │ NOT allowed │ NOT allowed
─────────────────────┼──────────────┼───────────────┼──────────────┼─────────────┼──────────────────
Thread safe          │ No           │ No            │ No           │ Yes (slow!) │ Yes (fast!)
─────────────────────┼──────────────┼───────────────┼──────────────┼─────────────┼──────────────────
Memory overhead      │ Medium       │ High          │ High         │ Medium      │ Medium
─────────────────────┼──────────────┼───────────────┼──────────────┼─────────────┼──────────────────
Range queries        │ No           │ No            │ Yes          │ No          │ No
LRU cache impl       │ No           │ YES           │ No           │ No          │ No
─────────────────────┼──────────────┼───────────────┼──────────────┼─────────────┼──────────────────
Introduced           │ Java 1.2     │ Java 1.4      │ Java 1.2     │ Java 1.0    │ Java 5
```

---

## ⏱️ Time & Space Complexity

```
OPERATION          │ HashMap      │ LinkedHashMap │ TreeMap
───────────────────┼──────────────┼───────────────┼────────────────────
get(key)           │ O(1)         │ O(1)          │ O(log n)
put(key, value)    │ O(1)★        │ O(1)★         │ O(log n)
remove(key)        │ O(1)         │ O(1)          │ O(log n)
containsKey(key)   │ O(1)         │ O(1)          │ O(log n)
firstKey/lastKey   │ N/A          │ N/A           │ O(log n)
floorKey/ceilKey   │ N/A          │ N/A           │ O(log n)
headMap/tailMap    │ N/A          │ N/A           │ O(log n) to find bounds
iteration          │ O(n+cap)     │ O(n)          │ O(n)
───────────────────┴──────────────┴───────────────┴────────────────────
★ amortized

LinkedHashMap iteration is O(n) NOT O(n+capacity) because:
  It uses the doubly-linked list to iterate, NOT the hash table array.
  So it skips empty buckets automatically!
  This is FASTER than HashMap iteration for sparse maps.
```

---

## ⚠️ Common Pitfalls

### 💣 **LinkedHashMap: subMap is Not Supported**

```java
// LinkedHashMap does NOT implement SortedMap or NavigableMap!
LinkedHashMap<String, Integer> lhm = new LinkedHashMap<>();
// lhm.headMap(...)  // ❌ Compile error! No such method.
// lhm.floorKey(...) // ❌ Compile error!

// If you need BOTH insertion order AND range queries:
// → Use an ordering-aware custom solution (e.g., maintain a TreeMap + LinkedHashMap)
// → Or use TreeMap (lose insertion order, gain sorted navigation)
```

### 💣 **TreeMap: Null Key Causes NullPointerException**

```java
TreeMap<String, Integer> map = new TreeMap<>();
map.put(null, 1);  // ❌ NullPointerException!
// compareTo(null) throws NPE

// WORKAROUND: Use Comparator.nullsFirst() or nullsLast():
TreeMap<String, Integer> nullSafe = new TreeMap<>(
    Comparator.nullsFirst(Comparator.naturalOrder())
);
nullSafe.put(null, 1);   // ✅ OK with custom comparator
nullSafe.put("apple", 2);
System.out.println(nullSafe.firstKey()); // null (null sorts first)
```

### 💣 **WeakHashMap: Interned Strings Never GC'd**

```java
// String literals and interned strings are held by the JVM class loader!
// They are NEVER garbage collected → WeakHashMap entries survive indefinitely.

WeakHashMap<String, Integer> weak = new WeakHashMap<>();
weak.put("interned", 1);     // String literal — NEVER GC'd
weak.put(new String("dynamic"), 2);  // new String object — CAN be GC'd

// RULE: WeakHashMap only works as intended with non-interned key objects
//       (i.e., objects created with 'new', not String literals)
```

---

## 🏢 Industry Applications

```
PATTERN              │ USE LinkedHashMap            │ USE TreeMap
─────────────────────┼──────────────────────────────┼──────────────────────────────────
Caching              │ LRU Cache (removeEldestEntry) │ TTL cache (expire oldest keys)
Ordered API response │ JSON with consistent order    │ Sorted JSON response
Time series data     │ Events in insertion order     │ Events in time order with ranges
Product catalog      │ Show in added order           │ Sort by price, range queries
Config management    │ Properties in definition order│ Alphabetically sorted config
Histogram            │ Bars in insertion order       │ Sorted histogram buckets
```

### 💻 **Spring Boot: Response with Consistent Field Order**

```java
@RestController
public class MetricsController {
    
    // LinkedHashMap ensures JSON fields are always in the same order
    @GetMapping("/metrics")
    public Map<String, Object> getMetrics() {
        Map<String, Object> metrics = new LinkedHashMap<>(); // NOT HashMap!
        metrics.put("timestamp", Instant.now());
        metrics.put("requests", requestCounter.get());
        metrics.put("latency_ms", latencyHistogram.mean());
        metrics.put("error_rate", errorRate.get());
        metrics.put("uptime_seconds", uptimeSeconds());
        return metrics;
        // Output JSON: {"timestamp":..., "requests":..., "latency_ms":..., ...}
        // Always in this exact order (reliable for monitoring tools)
    }
    
    // TreeMap for sorted configuration dump
    @GetMapping("/config")
    public SortedMap<String, String> getConfig() {
        TreeMap<String, String> config = new TreeMap<>(
            applicationProperties.entrySet().stream()
                .collect(Collectors.toMap(
                    e -> e.getKey().toString(),
                    e -> e.getValue().toString(),
                    (a, b) -> a,
                    TreeMap::new
                ))
        );
        return config; // Alphabetically sorted config dump
    }
}
```

### 💻 **Order Book Implementation (TreeMap)**

```java
// Stock exchange order book: sorted by price
public class OrderBook {
    // Buy orders: sorted by price DESCENDING (highest bid first)
    private final TreeMap<BigDecimal, Deque<Order>> bids = 
        new TreeMap<>(Comparator.reverseOrder());
    
    // Sell orders: sorted by price ASCENDING (lowest ask first)
    private final TreeMap<BigDecimal, Deque<Order>> asks = new TreeMap<>();
    
    public void addBid(Order order) {
        bids.computeIfAbsent(order.getPrice(), k -> new ArrayDeque<>())
            .offer(order);
    }
    
    public void addAsk(Order order) {
        asks.computeIfAbsent(order.getPrice(), k -> new ArrayDeque<>())
            .offer(order);
    }
    
    // Best bid: highest price buyer
    public Optional<BigDecimal> bestBidPrice() {
        return bids.isEmpty() ? Optional.empty() 
                              : Optional.of(bids.firstKey());
    }
    
    // Best ask: lowest price seller
    public Optional<BigDecimal> bestAskPrice() {
        return asks.isEmpty() ? Optional.empty() 
                              : Optional.of(asks.firstKey());
    }
    
    // Get all asks within a price range
    public SortedMap<BigDecimal, Deque<Order>> asksInRange(
            BigDecimal from, BigDecimal to) {
        return asks.subMap(from, true, to, true);
    }
    
    // Try to match a market order
    public Optional<Order> matchMarketBuy(int quantity) {
        if (asks.isEmpty()) return Optional.empty();
        Map.Entry<BigDecimal, Deque<Order>> bestAsk = asks.firstEntry();
        Order matched = bestAsk.getValue().poll();
        if (bestAsk.getValue().isEmpty()) asks.remove(bestAsk.getKey());
        return Optional.of(matched);
    }
}
```

---

## 🎯 Practice Problems

| # | Problem | Difficulty | Map Pattern |
|---|---------|-----------|-------------|
| 1 | [LRU Cache](https://leetcode.com/problems/lru-cache/) | 🟡 Medium | LinkedHashMap accessOrder |
| 2 | [LFU Cache](https://leetcode.com/problems/lfu-cache/) | 🔴 Hard | Two LinkedHashMaps |
| 3 | [My Calendar I](https://leetcode.com/problems/my-calendar-i/) | 🟡 Medium | TreeMap floorKey/ceilingKey |
| 4 | [Time Based Key-Value Store](https://leetcode.com/problems/time-based-key-value-store/) | 🟡 Medium | TreeMap floorKey for timestamp |
| 5 | [Maximum Profit in Job Scheduling](https://leetcode.com/problems/maximum-profit-in-job-scheduling/) | 🔴 Hard | TreeMap + DP |
| 6 | [Count of Range Sum](https://leetcode.com/problems/count-of-range-sum/) | 🔴 Hard | TreeMap for range counting |
| 7 | [Snapshot Array](https://leetcode.com/problems/snapshot-array/) | 🟡 Medium | TreeMap floorKey for versions |
| 8 | [Design File System](https://leetcode.com/problems/design-file-system/) | 🟡 Medium | HashMap/TreeMap hierarchy |

---

## 💡 Interview Tips

**Q1: When would you use LinkedHashMap over HashMap?**
```
LinkedHashMap when:
  1. Consistent iteration order matters (JSON output, config display, test assertions)
  2. Implementing LRU cache (accessOrder=true + removeEldestEntry)
  3. Tracking insertion/access sequence of events
  4. "Recently visited" features (browser history, recent files)

HashMap when:
  - Pure key-value lookup with no ordering requirements
  - Maximum performance, memory efficiency
  - Micro-service request routing, caches where order doesn't matter
```

**Q2: TreeMap vs HashMap — performance trade-off?**
```
HashMap:  O(1) average for all operations, but NO ordering
TreeMap:  O(log n) for all operations, but SORTED + range queries

RULE: Unless you need sorting or range queries, ALWAYS use HashMap.
      For 1M entries, log₂(1M) ≈ 20 comparisons per TreeMap operation
      vs ~1-2 operations for HashMap.

Use TreeMap when:
  - You need keys sorted (report generation, sorted API response)
  - Range queries (events between 9AM-5PM, prices between $10-$50)
  - Finding nearest key (floor/ceiling for "what happened right before X?")
  - Implementing rolling windows, sliding windows over sorted data
```

**Q3: How is LinkedHashMap's iteration faster than HashMap's for sparse maps?**
```
HashMap.entrySet().iterator():
  Scans the ENTIRE table[] array (default 16, grows to 32, 64, ...)
  Must skip null slots → O(n + capacity)
  For a map with 5 entries and capacity=1024: iterates 1024 slots!

LinkedHashMap.entrySet().iterator():
  Follows the DOUBLY-LINKED LIST from head to tail
  Visits ONLY the actual entries, no null slot scanning → O(n)
  For 5 entries: exactly 5 steps!

This makes LinkedHashMap FASTER than HashMap for iteration when the map
is sparse (few entries relative to capacity).
```

**Q4: What does removeEldestEntry do in LinkedHashMap?**
```java
// removeEldestEntry is called AFTER every put() or putAll()
// If it returns true, LinkedHashMap removes the "eldest" (head) entry.

// For insertion-order LinkedHashMap: eldest = first inserted entry
// For access-order LinkedHashMap: eldest = LEAST RECENTLY ACCESSED entry (LRU!)

protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
    return false; // DEFAULT: never auto-remove
}

// For a cache:
protected boolean removeEldestEntry(Map.Entry<K,V> eldest) {
    return size() > maxSize; // evict when over capacity
}
```

---

*Previous: [HashMap Deep Dive ←](./06_HashMap_Deep_Dive.md) | Next: [Queue, Deque & ArrayDeque →](./08_Queue_Deque_ArrayDeque.md)*
