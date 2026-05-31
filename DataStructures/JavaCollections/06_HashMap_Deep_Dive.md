# 🗺️ Java HashMap: The Deep Dive Masterclass 🚀

---

## 🎯 Chapter 6: HashMap — The Most Important Collection

> **"HashMap is the data structure that made Java famous for enterprise applications. Understanding it deeply — from initial capacity to treeification — separates a junior from a senior Java developer."**

---

## 📋 Table of Contents

1. [What is HashMap?](#what-is-hashmap)
2. [Internal Architecture — Every Detail](#internal-architecture)
3. [The Hashing Algorithm](#the-hashing-algorithm)
4. [Collision Handling — From Chaining to Trees](#collision-handling)
5. [Load Factor and Resize](#load-factor-and-resize)
6. [Java 8 Treeification](#java-8-treeification)
7. [All Methods Explained](#all-methods-explained)
8. [Java 8+ Advanced Methods](#java-8-advanced-methods)
9. [The Map Interface Contract](#the-map-interface-contract)
10. [Real-World Analogies](#real-world-analogies)
11. [When to USE HashMap](#when-to-use-hashmap)
12. [When to AVOID HashMap](#when-to-avoid-hashmap)
13. [Time & Space Complexity](#time--space-complexity)
14. [Common Pitfalls](#common-pitfalls)
15. [Industry Applications](#industry-applications)
16. [Practice Problems](#practice-problems)
17. [Interview Tips](#interview-tips)

---

## 🤔 What is HashMap?

### 📖 **Definition**

`HashMap<K,V>` is the most widely used implementation of the `Map<K,V>` interface. It stores **key-value pairs** in a **hash table**, providing O(1) average time for get, put, and remove operations.

```
HashMap characteristics:
✅ O(1) average for get, put, remove, containsKey
✅ ONE null key allowed (at bucket 0)
✅ Multiple null values allowed
✅ NO guaranteed iteration order
❌ NOT thread-safe (use ConcurrentHashMap for multi-threading)
❌ Not sorted (use TreeMap for sorted keys)

Note: Map does NOT extend Collection!
  Map has its own parallel interface hierarchy.
  "A Map IS a Map, not a Collection."

Introduced: Java 1.2 (1998), major improvement in Java 8 (treeification)
```

---

## 🏗️ Internal Architecture — Every Detail

### 🔧 **The Internal Structure**

```java
// Simplified HashMap source (Java 17):
public class HashMap<K,V> extends AbstractMap<K,V> implements Map<K,V> {

    // THE HASH TABLE: An array of Node chains (buckets)
    transient Node<K,V>[] table;
    
    // Number of key-value pairs currently stored
    transient int size;
    
    // threshold = capacity × loadFactor — when to resize
    int threshold;
    
    // Load factor (default 0.75)
    final float loadFactor;
    
    // For fail-fast iterators
    transient int modCount;
    
    // Default values:
    static final int DEFAULT_INITIAL_CAPACITY = 16;  // MUST be power of 2
    static final float DEFAULT_LOAD_FACTOR = 0.75f;
    static final int TREEIFY_THRESHOLD = 8;          // chain → tree at 8 nodes
    static final int UNTREEIFY_THRESHOLD = 6;        // tree → chain at 6 nodes
    static final int MIN_TREEIFY_CAPACITY = 64;      // min table size for treeify
    
    // A single Entry in the hash table (singly-linked list node)
    static class Node<K,V> implements Map.Entry<K,V> {
        final int hash;   // cached hash of key
        final K key;      // the key (immutable reference)
        V value;          // the value (mutable)
        Node<K,V> next;   // next node in chain (for collision handling)
    }
    
    // Java 8+ tree node (for treeified buckets)
    static final class TreeNode<K,V> extends LinkedHashMap.Entry<K,V> {
        TreeNode<K,V> parent;
        TreeNode<K,V> left;
        TreeNode<K,V> right;
        TreeNode<K,V> prev;
        boolean red;
    }
}
```

### 📊 **Memory Layout with 3 buckets occupied**

```
HashMap { "Alice"→100, "Bob"→200, "Eve"→300, "Charlie"→400 }
Capacity = 16 (default)

table[] (length = 16):
┌────────────────────────────────────────────────────────────────────────┐
│ [0]  null                                                              │
│ [1]  null                                                              │
│ [2]  Node{hash=2, key="Bob",     value=200, next=Node{key="Eve"}}     │
│      └─→   Node{hash=2, key="Eve",     value=300, next=null}          │
│      (collision! Bob and Eve hash to same bucket)                     │
│ [3]  null                                                              │
│ [4]  Node{hash=4, key="Alice",   value=100, next=null}                │
│ [5]  null                                                              │
│ ...                                                                    │
│ [7]  Node{hash=7, key="Charlie", value=400, next=null}                │
│ ...                                                                    │
│[15]  null                                                              │
└────────────────────────────────────────────────────────────────────────┘
size = 4, threshold = 16 × 0.75 = 12
```

---

## 🔬 The Hashing Algorithm

### 🔧 **Two-Step Hash Computation**

```java
// Step 1: Get the raw hashCode() of the key
// Step 2: SPREAD the hash to reduce clustering — Java 8's "perturbation":

static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
    //                                                  ^^^^^^^^^^^^^^^
    // XOR high 16 bits with low 16 bits
    // This mixes the high bits into the low bits
    // so even with bad hashCode implementations, entries spread better
}

// Step 3: Compute bucket index (VERY fast — bit AND instead of modulo)
int index = (capacity - 1) & hash;
// Since capacity is ALWAYS a power of 2, this is equivalent to hash % capacity
// but 10x faster (single bitwise AND vs division)
```

```
WHY MUST CAPACITY BE A POWER OF 2?

If capacity = 16:  binary = 0001 0000
capacity - 1 = 15: binary = 0000 1111

hash & 15 extracts the last 4 bits of hash = values 0..15
This is IDENTICAL to hash % 16 but uses a single AND instruction.

If capacity = 15 (NOT power of 2): this trick doesn't work!
hash & 14 (binary 1110) would never produce odd indices 1,3,5... → clustering!
```

### 🔧 **hashCode() for Common Types**

```java
// String.hashCode() — polynomial rolling hash:
// s[0]*31^(n-1) + s[1]*31^(n-2) + ... + s[n-1]
"abc".hashCode() // = 'a'*31^2 + 'b'*31 + 'c' = 96721 + 3162 + 99 = 96354

// Integer.hashCode() — just the int value!
Integer.valueOf(42).hashCode() // = 42

// Long.hashCode() — XOR high and low 32 bits:
Long.valueOf(1000000000000L).hashCode() // = (int)(value ^ (value >>> 32))

// null key — special case
// HashMap.hash(null) = 0 → always goes to bucket 0
map.put(null, "value"); // stored at table[0]

// Custom objects: YOU MUST implement hashCode() for correct behavior
// (see HashSet chapter for details)
```

---

## 💥 Collision Handling — From Chaining to Trees

### 🔧 **Phase 1: Separate Chaining (Linked List)**

```
When two different keys hash to the same bucket:

Before: bucket[3] → Node{key="Alice", value=1}

After putting key="Bob" (also hashes to bucket 3):
bucket[3] → Node{key="Bob", value=2, next=→} → Node{key="Alice", value=1, next=null}
             ↑ New nodes added at HEAD of chain (before Java 8)
             ↑ New nodes added at TAIL of chain (Java 8+, avoids infinite loop risk)

Lookup of "Alice":
  1. hash("Alice") → bucket 3
  2. Check first node: "Bob".equals("Alice")? NO, follow next
  3. Check second node: "Alice".equals("Alice")? YES → return value=1
  Cost: O(chain_length), typically O(1) amortized with good hash distribution
```

### 🔧 **The Worst Case — Hash DoS Attack**

```
ATTACK: An adversary crafts keys with the SAME hashCode:
  All keys → bucket[0]
  Result: O(n) for every get/put (single chain of n elements)
  With 100,000 such keys: HashMap becomes unusable!

EXAMPLE: In Java < 8, String hashCode could be attacked.
Putting 65536 Strings with hashCode=0 → HashMap degrades to O(n) linked list.

SOLUTION (Java 8+): Treeification!
```

---

## 🌳 Java 8 Treeification

### 🔧 **The Upgrade from Chain to Tree**

```
When a SINGLE BUCKET's chain length reaches TREEIFY_THRESHOLD = 8
AND the table size >= MIN_TREEIFY_CAPACITY = 64:
  → The linked list in that bucket is converted to a Red-Black Tree!
  → Lookup goes from O(n) to O(log n) in the worst case

RULE:
  chain length: 0-7  → Regular linked list (Node)
  chain length: 8+   → Red-Black Tree (TreeNode)  [if table.length >= 64]
  chain length: 6    → Tree converted BACK to linked list (UNTREEIFY_THRESHOLD=6)

WHY THRESHOLD = 8?
  With a good hashCode, probability of 8 collisions in one bucket is
  (1/16)^8 ≈ 0.00000006 (extremely unlikely in normal usage)
  So treeification only kicks in for PATHOLOGICAL cases (bad hashCode or attack)
```

```
BEFORE (chain): n=8 nodes in bucket
  bucket[3] → Node1 → Node2 → Node3 → Node4 → Node5 → Node6 → Node7 → Node8
  get() = O(8) worst case

AFTER (tree): treeified to Red-Black Tree
  bucket[3] → TreeNode (root)
              /          \
          TreeNode      TreeNode
          /    \         /    \
       ...     ...     ...    ...
  get() = O(log 8) = O(3) worst case — much better!
```

```java
// You don't call treeify manually — Java does it automatically:
HashMap<String, Integer> map = new HashMap<>(16); // small capacity
// If you add many keys with the same hashCode...
for (int i = 0; i < 10; i++) {
    map.put(new BadHashKey(i), i); // all BadHashKey instances return hashCode=0
    // When 8th key is added to bucket 0 → AUTOMATICALLY treeified!
}
```

---

## 📈 Load Factor and Resize

### 🔧 **The Threshold Calculation**

```java
// threshold = capacity × loadFactor
// default: 16 × 0.75 = 12

// When size exceeds threshold → RESIZE (double capacity, rehash all entries)
int newCapacity = oldCapacity << 1;  // double: 16 → 32 → 64 → 128...

// Each resize:
// 1. Create new array of double capacity
// 2. For each existing Entry:
//    new_index = hash & (newCapacity - 1)
//    (Java 8 optimization: new_index is either old_index OR old_index + oldCapacity)
// 3. Move all entries to new positions (rehash = redistribute)
// Cost: O(n) for one resize, but amortized O(1) per put
```

```
DEFAULT BEHAVIOR:
  Initial capacity: 16
  Initial threshold: 16 × 0.75 = 12
  
  After 12 entries: RESIZE → capacity=32, threshold=24
  After 24 entries: RESIZE → capacity=64, threshold=48
  After 48 entries: RESIZE → capacity=128, threshold=96
  ...

LOAD FACTOR TRADE-OFFS:
  LOW load factor (0.25): 
    + Fewer collisions, faster lookup
    - More memory wasted (sparse table), more resizes
    
  HIGH load factor (0.9):
    + More memory efficient
    - More collisions, slower lookup
    
  0.75 is the optimal balance (per Java documentation + empirical testing)
```

### 🔧 **Performance Optimization: Pre-sizing**

```java
// ❌ Without pre-sizing — 1M entries causes ~20 resizes
HashMap<String, Integer> map = new HashMap<>();
for (int i = 0; i < 1_000_000; i++) map.put("key"+i, i);

// ✅ Pre-sized to avoid resizes:
// Formula: capacity = expectedEntries / loadFactor + 1
int expectedEntries = 1_000_000;
int initialCapacity = (int)(expectedEntries / 0.75) + 1; // = 1,333,334
HashMap<String, Integer> map = new HashMap<>(initialCapacity);

// Guava's helper:
Map<K,V> map = Maps.newHashMapWithExpectedSize(1_000_000);
// equivalent to new HashMap<>(1_000_000 * 4 / 3 + 1)
```

---

## 🛠️ All Methods Explained

### ➕ **Put Operations**

```java
Map<String, Integer> map = new HashMap<>();

// put(K key, V value) — returns previous value (null if new key)
Integer prev = map.put("Alice", 100);  // null (new key)
Integer prev2 = map.put("Alice", 200); // 100 (old value overwritten)

// putAll(Map m) — copy all entries from another map
map.putAll(Map.of("Bob", 300, "Charlie", 400));

// putIfAbsent(K key, V value) — only put if key NOT present, returns current value
map.putIfAbsent("Alice", 999);  // "Alice" exists → no change, returns 200
map.putIfAbsent("Dave", 500);   // "Dave" is new → added, returns null

// Java 9+ immutable factory
Map<String, Integer> immutable = Map.of("a", 1, "b", 2);  // max 10 entries
Map<String, Integer> immutable2 = Map.ofEntries(
    Map.entry("a", 1), Map.entry("b", 2), ...  // any number of entries
);
```

### 🔍 **Get Operations**

```java
// get(Object key) — returns value, or null if key absent
Integer val = map.get("Alice");     // 200
Integer missing = map.get("Zoe");  // null

// getOrDefault(Object key, V defaultValue) — Java 8+
Integer safe = map.getOrDefault("Zoe", 0); // 0 (key absent, use default)
Integer found = map.getOrDefault("Alice", 0); // 200 (key present)

// containsKey(Object key) — O(1)
boolean hasAlice = map.containsKey("Alice"); // true

// containsValue(Object value) — O(n)! linear scan of all values
boolean has200 = map.containsValue(200); // true (but O(n)!)

// size()
int sz = map.size();

// isEmpty()
boolean empty = map.isEmpty();
```

### ➖ **Remove Operations**

```java
// remove(Object key) — returns removed value (null if not found)
Integer removed = map.remove("Alice"); // 200

// remove(Object key, Object value) — conditional remove (Java 8+)
boolean removed2 = map.remove("Bob", 300); // true (key+value match)
boolean removed3 = map.remove("Charlie", 999); // false (value doesn't match)

// clear() — remove all entries
map.clear();
```

### 🔄 **Iteration Patterns**

```java
Map<String, Integer> map = new HashMap<>(Map.of("a", 1, "b", 2, "c", 3));

// 1. entrySet() — iterate key-value pairs (PREFERRED — most efficient)
for (Map.Entry<String, Integer> entry : map.entrySet()) {
    System.out.println(entry.getKey() + " → " + entry.getValue());
}

// 2. keySet() — iterate keys only
for (String key : map.keySet()) {
    System.out.println(key);
}

// 3. values() — iterate values only
for (Integer value : map.values()) {
    System.out.println(value);
}

// 4. forEach (Java 8+) — BiConsumer
map.forEach((key, value) -> System.out.println(key + "=" + value));

// 5. Stream API
map.entrySet().stream()
   .filter(e -> e.getValue() > 1)
   .sorted(Map.Entry.comparingByValue())
   .forEach(e -> System.out.println(e.getKey() + "=" + e.getValue()));

// 6. Collect to another map
Map<String, Integer> filtered = map.entrySet().stream()
    .filter(e -> e.getValue() > 1)
    .collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue));
```

---

## 🚀 Java 8+ Advanced Methods

### 🔧 **compute, merge, and Friends**

```java
Map<String, Integer> wordCount = new HashMap<>();

// ── computeIfAbsent ── initialize if key missing, then return value
List<String> groups = new ArrayList<>();
Map<Integer, List<String>> groupByLength = new HashMap<>();
for (String s : List.of("cat", "dog", "car", "dot")) {
    groupByLength.computeIfAbsent(s.length(), k -> new ArrayList<>()).add(s);
    //                            ↑ key          ↑ function to create value if absent
}
// {3: ["cat","dog","car","dot"]}

// ── computeIfPresent ── update only if key exists
Map<String, Integer> scores = new HashMap<>(Map.of("Alice", 100, "Bob", 200));
scores.computeIfPresent("Alice", (k, v) -> v + 10); // Alice=110
scores.computeIfPresent("Zoe",   (k, v) -> v + 10); // no-op (Zoe doesn't exist)

// ── compute ── compute regardless of whether key exists
scores.compute("Alice", (k, v) -> v == null ? 0 : v + 5); // Alice=115
scores.compute("Zoe", (k, v) -> v == null ? 0 : v + 5);   // Zoe=0 (new key!)

// ── merge ── the MOST USEFUL for aggregation patterns
// merge(key, value, BiFunction<oldValue, newValue, result>)

// Word count (common pattern):
String text = "the quick brown fox the quick fox";
Map<String, Integer> freq = new HashMap<>();
for (String word : text.split(" ")) {
    freq.merge(word, 1, Integer::sum);
    //  ↑ key  ↑ if absent=1  ↑ if present: add existing + 1
}
// {the=2, quick=2, brown=1, fox=2}

// ── replaceAll ── update all values
scores.replaceAll((key, value) -> value * 2); // double all scores

// ── replace ── update specific key
scores.replace("Alice", 1000);                // unconditional replace
scores.replace("Alice", 1000, 2000);          // conditional replace (old,new)

// ── getOrDefault examples ──
int count = freq.getOrDefault("unknown", 0);  // 0

// ── merge vs compute for aggregation ──
// PREFER merge for simple aggregation:
map.merge(key, 1, Integer::sum);  // CLEANER

// use compute when logic is more complex:
map.compute(key, (k, v) -> {
    if (v == null) return initValue(k);
    return updateComplexLogic(k, v);
});
```

---

## 🗺️ The Map Interface Contract

### 🔧 **Full Map API**

```java
// java.util.Map<K,V>
public interface Map<K,V> {
    // MODIFICATION
    V put(K key, V value);
    void putAll(Map<? extends K, ? extends V> m);
    V remove(Object key);
    void clear();
    
    // QUERY
    V get(Object key);
    boolean containsKey(Object key);
    boolean containsValue(Object value);  // O(n)!
    int size();
    boolean isEmpty();
    
    // VIEWS (live — backed by the map!)
    Set<K> keySet();
    Collection<V> values();
    Set<Map.Entry<K,V>> entrySet();
    
    // Java 8+
    V getOrDefault(Object key, V defaultValue);
    void forEach(BiConsumer<? super K, ? super V> action);
    V putIfAbsent(K key, V value);
    boolean remove(Object key, Object value);
    boolean replace(K key, V oldValue, V newValue);
    V replace(K key, V value);
    void replaceAll(BiFunction<? super K, ? super V, ? extends V> function);
    V computeIfAbsent(K key, Function<? super K, ? extends V> mappingFunction);
    V computeIfPresent(K key, BiFunction<? super K, ? super V, ? extends V> rf);
    V compute(K key, BiFunction<? super K, ? super V, ? extends V> rf);
    V merge(K key, V value, BiFunction<? super V, ? super V, ? extends V> rf);
    
    // Java 9+
    static <K,V> Map<K,V> of(K k1, V v1, ...);   // max 10 pairs
    static <K,V> Map<K,V> ofEntries(Map.Entry<K,V>... entries);
    static <K,V> Map<K,V> copyOf(Map<? extends K, ? extends V> map);
    static <K,V> Map.Entry<K,V> entry(K k, V v);
    
    // Java 21 (SequencedMap)
    Map.Entry<K,V> firstEntry();
    Map.Entry<K,V> lastEntry();
    Map.Entry<K,V> pollFirstEntry();
    Map.Entry<K,V> pollLastEntry();
    SequencedSet<K> sequencedKeySet();
    SequencedCollection<V> sequencedValues();
    SequencedSet<Map.Entry<K,V>> sequencedEntrySet();
    Map<K,V> reversed();
}
```

---

## 🌍 Real-World Analogies

### 📖 **The Post Office PO Boxes**

```
HashMap is like a Post Office with numbered PO Boxes:

POST OFFICE (table[]):
Box 0:  [Empty]
Box 1:  [Envelope from Alice]
Box 2:  [Envelope from Bob] → [Envelope from Eve]  ← collision chain!
Box 3:  [Empty]
Box 4:  [Envelope from Charlie]
...

Delivering mail to "Alice":
  1. Compute box number: hash("Alice") → box 1
  2. Open box 1 → Alice's envelope found!

Delivering mail to "Bob":
  1. Compute box number: hash("Bob") → box 2
  2. Open box 2 → see Bob's AND Eve's envelope (collision!)
  3. Check name on each → found Bob's!

Adding a new person:
  1. Compute box number → find box
  2. If box occupied → add to chain in that box
  When too many boxes occupied (75%) → BUILD A BIGGER POST OFFICE (resize)
```

---

## ✅ When to USE HashMap

```
✅ USE HashMap WHEN:
─────────────────────────────────────────────────────────────────────
✅ You need KEY-VALUE STORAGE with fast O(1) lookup
   Example: Cache, configuration, dictionaries, indexes

✅ Frequency counting / aggregation
   Example: word count, inventory count, user visit counts

✅ Memoization (caching expensive computation results)
   Example: Fibonacci with memoization, DB query result cache

✅ Grouping elements by a key
   Example: groupBy in stream, same as SQL GROUP BY

✅ Implementing graphs as adjacency lists
   Example: Map<Node, List<Node>> adjacencyList

✅ DEDUPLICATION with tracking
   Example: Set<K> = HashSet (backed by HashMap)

REAL EXAMPLES:
  - HTTP session storage: Map<String, HttpSession>
  - Config properties: Map<String, String>
  - User permissions: Map<UserId, Set<Permission>>
  - Frequency map: Map<Character, Integer> for anagram check
  - Memoization: Map<Integer, Long> fib = new HashMap<>()
```

---

## ❌ When to AVOID HashMap

```
❌ AVOID HashMap WHEN:
─────────────────────────────────────────────────────────────────────
❌ YOU NEED SORTED KEYS → use TreeMap
❌ YOU NEED INSERTION ORDER → use LinkedHashMap
❌ YOU NEED THREAD SAFETY → use ConcurrentHashMap
❌ KEY IS A MUTABLE OBJECT → avoid entirely or ensure key is never mutated
❌ KEY IS NULL AND YOU NEED ConcurrentHashMap LATER → null keys not allowed there
❌ PRIMITIVE KEYS (performance critical) → autoboxing overhead
   Use: Map<Integer,...> boxes each int to Integer object
   Consider: plain int[] or long[] array instead
```

---

## ⏱️ Time & Space Complexity

```
OPERATION          │ Average   │ Worst Case │ Notes
───────────────────┼───────────┼────────────┼────────────────────────────────
get(key)           │ O(1)      │ O(log n)   │ Worst: all keys hash to same bucket (tree)
put(key, value)    │ O(1)★     │ O(log n)   │ ★ amortized (resize is O(n))
remove(key)        │ O(1)      │ O(log n)   │
containsKey(key)   │ O(1)      │ O(log n)   │
containsValue(val) │ O(n)      │ O(n)       │ Linear scan of all values!
size()             │ O(1)      │ O(1)       │
isEmpty()          │ O(1)      │ O(1)       │
iteration          │ O(n+cap)  │ O(n+cap)   │ Iterates ALL buckets including empty ones!
putAll             │ O(k)      │ O(k log n) │ k = number of entries added
───────────────────┴───────────┴────────────┴────────────────────────────────
SPACE: O(n) — ~48 bytes per entry + table array overhead

WHY iteration is O(n + capacity):
  Iterator must scan ALL table[] slots (including empty nulls)
  For a map with size=10 and capacity=1024: iterates 1024 buckets
  → Keep capacity reasonable or use trimToCapacity via new HashMap<>(map)
```

---

## ⚠️ Common Pitfalls

### 💣 **Pitfall 1: Mutable Keys**

```java
// ❌ NEVER use mutable objects as HashMap keys!
List<String> key = new ArrayList<>(List.of("user", "123"));
Map<List<String>, String> map = new HashMap<>();
map.put(key, "Alice");

System.out.println(map.get(key)); // "Alice" ✅

key.add("extra"); // MUTATE the key!

System.out.println(map.get(key)); // null! 💥 hashCode changed, bucket changed!
// The entry is ORPHANED — still in map but unreachable!

// ✅ Use immutable keys: String, Integer, UUID, List.of(), records
record UserId(String type, String id) {}  // record = immutable
map.put(new UserId("user", "123"), "Alice"); // safe!
```

### 💣 **Pitfall 2: Null-Checking on get()**

```java
Map<String, Integer> scores = new HashMap<>(Map.of("Alice", 0));

// ❌ Ambiguous null check:
Integer score = scores.get("Alice");
if (score == null) {
    // Does this mean: (1) Alice not in map, or (2) Alice's score is 0?
    // Wait, score IS 0 but it's an Integer... but 0 is not null!
    // Still: what if someone did map.put("Bob", null)?
}

// ✅ Use containsKey() when null values are possible:
if (scores.containsKey("Alice")) {
    Integer val = scores.get("Alice"); // could be null if mapped to null
}

// ✅ Or use getOrDefault to avoid null:
int safeScore = scores.getOrDefault("Alice", -1);
```

### 💣 **Pitfall 3: ConcurrentModificationException**

```java
Map<String, Integer> map = new HashMap<>(Map.of("a", 1, "b", 2, "c", 3));

// ❌ Removing from map while iterating its entrySet
for (Map.Entry<String, Integer> e : map.entrySet()) {
    if (e.getValue() < 2) {
        map.remove(e.getKey()); // ConcurrentModificationException!
    }
}

// ✅ Use iterator.remove()
Iterator<Map.Entry<String, Integer>> it = map.entrySet().iterator();
while (it.hasNext()) {
    if (it.next().getValue() < 2) {
        it.remove(); // SAFE
    }
}

// ✅ Or use removeIf on the entrySet (Java 8+)
map.entrySet().removeIf(e -> e.getValue() < 2);
```

### 💣 **Pitfall 4: HashMap in Multi-threaded Code**

```java
// ❌ DANGEROUS — HashMap.put() is not atomic!
// In Java 7: Infinite loop risk during concurrent resize!
// (Two threads resizing simultaneously can create a cycle in the linked list chain)
// In Java 8: This was fixed, but data corruption (lost updates) still possible!

static Map<String, Integer> cache = new HashMap<>();  // SHARED, UNSYNCHRONIZED!

// Thread 1: cache.put("key", 1);
// Thread 2: cache.put("key", 2);
// Result: Data may be lost, inconsistent, or exceptions thrown!

// ✅ FIX: Use ConcurrentHashMap for shared maps
static Map<String, Integer> cache = new ConcurrentHashMap<>();

// ✅ FIX: Use local maps (no sharing between threads)
public void processRequest() {
    Map<String, Integer> localMap = new HashMap<>(); // not shared → safe
    // ...
}
```

---

## 🏢 Industry Applications

```
COMPANY          USE CASE                              IMPLEMENTATION
──────────────────────────────────────────────────────────────────────────────
Google           DNS resolution cache                  Map<String, IpAddress>
Amazon           Product recommendation by user        Map<UserId, List<Product>>
Netflix          User watch history lookup             Map<UserId, List<ContentId>>
Uber             Driver location by driver ID          Map<DriverId, Location>
Twitter/X        Tweet index by hashtag                Map<String, List<TweetId>>
Kafka            Topic partition metadata              Map<TopicPartition, PartitionState>
Spring           Bean factory (all Spring beans!)      Map<String, BeanDefinition>
JVM              String intern pool                    HashMap<String, String>
Hibernate        First-level entity cache              Map<EntityKey, Object>
Redis Java       Jedis connection pool                 Map<String, JedisPool>
```

### 💻 **Real Spring Boot Example — Comprehensive**

```java
@Service
public class AnalyticsService {
    
    // 1. Frequency counting using merge()
    public Map<String, Long> countEventsByType(List<Event> events) {
        Map<String, Long> frequency = new HashMap<>();
        events.forEach(e -> frequency.merge(e.getType(), 1L, Long::sum));
        return frequency;
    }
    
    // 2. Grouping using computeIfAbsent()
    public Map<String, List<Event>> groupByUser(List<Event> events) {
        Map<String, List<Event>> grouped = new HashMap<>();
        events.forEach(e -> 
            grouped.computeIfAbsent(e.getUserId(), k -> new ArrayList<>()).add(e)
        );
        return grouped;
    }
    
    // 3. Memoization using computeIfAbsent()
    private final Map<Long, UserProfile> profileCache = new HashMap<>();
    
    public UserProfile getUserProfile(Long userId) {
        return profileCache.computeIfAbsent(userId, 
            id -> userRepository.findById(id).orElseThrow()
        );
    }
    
    // 4. Bi-directional lookup using two maps
    private final Map<String, Long> usernameToId = new HashMap<>();
    private final Map<Long, String> idToUsername = new HashMap<>();
    
    public void registerUser(String username, Long id) {
        usernameToId.put(username, id);
        idToUsername.put(id, username);
    }
    
    // 5. Counting top-N items
    public List<String> getTopNEventTypes(List<Event> events, int n) {
        Map<String, Integer> freq = new HashMap<>();
        events.forEach(e -> freq.merge(e.getType(), 1, Integer::sum));
        
        return freq.entrySet().stream()
            .sorted(Map.Entry.<String, Integer>comparingByValue().reversed())
            .limit(n)
            .map(Map.Entry::getKey)
            .collect(Collectors.toList());
    }
}
```

---

## 🎯 Practice Problems

| # | Problem | Difficulty | HashMap Pattern |
|---|---------|-----------|-----------------|
| 1 | [Two Sum](https://leetcode.com/problems/two-sum/) | 🟢 Easy | num → index lookup |
| 2 | [Group Anagrams](https://leetcode.com/problems/group-anagrams/) | 🟡 Medium | sorted string → group |
| 3 | [Top K Frequent Elements](https://leetcode.com/problems/top-k-frequent-elements/) | 🟡 Medium | freq map + heap |
| 4 | [Subarray Sum Equals K](https://leetcode.com/problems/subarray-sum-equals-k/) | 🟡 Medium | prefix sum → count |
| 5 | [LRU Cache](https://leetcode.com/problems/lru-cache/) | 🟡 Medium | HashMap + LinkedList |
| 6 | [Random Pick with Blacklist](https://leetcode.com/problems/random-pick-with-blacklist/) | 🔴 Hard | Mapping trick |
| 7 | [Minimum Window Substring](https://leetcode.com/problems/minimum-window-substring/) | 🔴 Hard | char count maps |
| 8 | [Design HashMap](https://leetcode.com/problems/design-hashmap/) | 🟢 Easy | Implement from scratch |
| 9 | [Valid Anagram](https://leetcode.com/problems/valid-anagram/) | 🟢 Easy | Character frequency |
| 10| [Word Frequency](https://leetcode.com/problems/word-frequency/) | 🟢 Easy | merge() pattern |

---

## 💡 Interview Tips

**Q1: Explain HashMap internals from put() to get() step by step**
```
PUT "Alice" → 100:
  1. hash = "Alice".hashCode() ^ ("Alice".hashCode() >>> 16)  // perturbation
  2. index = hash & (capacity-1)  // e.g., hash & 15 for capacity=16
  3. If table[index] is null → create new Node there
  4. If table[index] is not null → traverse chain/tree:
     - If found key that equals "Alice" → UPDATE value
     - If not found → ADD new Node at tail of chain
  5. If size > threshold → resize (double capacity, rehash all)
  6. If chain length >= 8 AND table.length >= 64 → treeify that bucket

GET "Alice":
  1. hash = hash("Alice")
  2. index = hash & (capacity-1)
  3. table[index] is null? → return null
  4. table[index] is a Node (chain)? → linear search for "Alice"
  5. table[index] is a TreeNode (tree)? → BST search for "Alice" O(log n)
```

**Q2: What changed in HashMap from Java 7 to Java 8?**
```
Java 7:
  - Single linked list per bucket (chain only)
  - O(n) worst case for get/put
  - New nodes added at HEAD of chain
  - Vulnerable to Hash DoS attacks

Java 8:
  - Linked list → Red-Black Tree when chain length >= 8 (TREEIFY_THRESHOLD)
  - O(log n) worst case for get/put
  - New nodes added at TAIL of chain (avoids infinite loop in concurrent use)
  - HashMap.hash() improved (XOR with high bits) for better distribution
  - Resize algorithm changed (uses bit trick, avoids re-hashing)
  - forEach(), merge(), compute(), replaceAll() added

Java 9+:
  - Map.of(), Map.ofEntries() — immutable factory methods
```

**Q3: Why does HashMap use power-of-2 capacity?**
```
indexing: (capacity-1) & hash
If capacity = 16 (binary: 10000), then capacity-1 = 15 (binary: 01111)
(15 & hash) extracts the last 4 bits of hash → values 0..15
This is IDENTICAL to hash % 16 but ~10x faster (bitwise AND vs modulo)

If capacity is NOT a power of 2, this trick fails:
  If capacity = 10, capacity-1 = 9 (binary: 1001)
  9 & hash never produces values 2,4,6 → uneven distribution!

The power-of-2 constraint ensures even bucket distribution.
```

**Q4: When should you use HashMap.merge() vs compute()?**
```java
// merge() — best for aggregation (count, sum, combine)
map.merge(key, 1, Integer::sum);   // if absent: put 1; if present: add 1
map.merge(key, newVal, (a,b) -> a + b); // combine old and new value

// compute() — best for complex transformations based on current state
map.compute(key, (k, v) -> {
    if (v == null) return initializeComplexObject(k);
    v.updateState();
    return v.isExpired() ? null : v; // return null to REMOVE the entry
});
// Note: returning null from compute() REMOVES the entry from the map!
```

---

*Previous: [TreeSet Deep Dive ←](./05_TreeSet.md) | Next: [LinkedHashMap & TreeMap →](./07_LinkedHashMap_TreeMap.md)*
