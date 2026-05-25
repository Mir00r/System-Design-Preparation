# 🔒 Java Concurrent Collections: Thread-Safe Masterclass 🚀

---

## 🎯 Chapter 10: Concurrent Collections — Safe Shared Data

> **"Writing correct concurrent code is hard. Java's concurrent collections let you focus on your business logic while the JDK handles the terrifying details of memory visibility, atomic operations, and lock contention."**

---

## 📋 Table of Contents

1. [Why Concurrent Collections?](#why-concurrent-collections)
2. [ConcurrentHashMap — The Workhorse](#concurrenthashmap)
3. [CopyOnWriteArrayList — For Read-Heavy Workloads](#copyonwritearraylist)
4. [CopyOnWriteArraySet](#copyonwritearrayset)
5. [ConcurrentLinkedQueue & Deque](#concurrentlinkedqueue)
6. [BlockingQueue Family](#blockingqueue-family)
7. [ConcurrentSkipListMap & Set](#concurrentskiplist)
8. [Collections.synchronizedXxx() vs Concurrent Collections](#synchronized-vs-concurrent)
9. [Choosing the Right Concurrent Collection](#choosing)
10. [Time & Space Complexity](#time--space-complexity)
11. [Common Pitfalls](#common-pitfalls)
12. [Industry Applications](#industry-applications)
13. [Interview Tips](#interview-tips)

---

## ⚡ Why Concurrent Collections?

### 📖 **The Problem with Regular Collections in Multi-Threaded Code**

```java
// ❌ DANGER: HashMap in multi-threaded code
static HashMap<String, Integer> counter = new HashMap<>();

// Thread 1:                    // Thread 2:
counter.put("key", 1);         counter.put("key", 2);
// RACE CONDITION! One update will be lost.
// In Java 7 and earlier: could cause INFINITE LOOP during resize!
// (Two threads resizing simultaneously create a cycle in hash chains)

// ❌ DANGER: ArrayList in multi-threaded code
static List<String> events = new ArrayList<>();

// Thread 1:                    // Thread 2:
events.add("login");           events.add("logout");
// RESULT: Possible ArrayIndexOutOfBoundsException, ConcurrentModificationException
//         or silently losing one update!
```

```
THE 4 PROBLEMS WITH NON-THREAD-SAFE COLLECTIONS:
─────────────────────────────────────────────────────────────────────
1. LOST UPDATES: Two threads write simultaneously, one update discarded
2. CORRUPT STATE: Partial writes visible to other threads (torn reads)
3. VISIBILITY: CPU cache may not flush writes to main memory
4. STRUCTURAL BUGS: Resize operation is not atomic (Java 7 HashMap infinite loop)
```

---

## 🗺️ ConcurrentHashMap — The Workhorse

### 📖 **Definition**

`ConcurrentHashMap<K,V>` is the thread-safe, high-performance replacement for `Hashtable` and `Collections.synchronizedMap(HashMap)`. It allows **concurrent reads** and **granular writes** without blocking the entire map.

```
ConcurrentHashMap characteristics:
✅ Thread-safe for all individual operations
✅ High concurrency: reads are non-blocking, writes use fine-grained locks
✅ NO null keys or values (by design — prevents ambiguity in concurrent checks)
✅ Implements ConcurrentMap<K,V> — atomic compound operations
❌ Bulk operations (size, iteration) are NOT atomic — snapshot at a point in time
❌ Slightly slower than HashMap in single-threaded use (CAS overhead)
```

### 🏗️ **Internal Architecture — Java 7 vs Java 8+**

```
JAVA 7 — Segment-Based Locking:
  Array of 16 SEGMENTS (like mini-HashMaps)
  Each segment has its own ReentrantLock
  Concurrent writes: up to 16 threads can write simultaneously
  
  table[] = [Segment0][Segment1]...[Segment15]
              ↓           ↓
           lock0        lock1       ← independent locks
           [bucket0]    [bucket0]
           [bucket1]    [bucket1]
           ...          ...

JAVA 8+ — CAS + Synchronized on Individual Bins:
  Array of Node[] (same as HashMap)
  READS: fully lock-free using volatile reads
  WRITES (new key, no collision): CAS (compareAndSet) — completely atomic, no lock!
  WRITES (collision in existing bucket): synchronized on the SINGLE bucket head node
  RESIZE: done in parallel by multiple threads!

  WHY JAVA 8 IS BETTER:
  - No fixed 16-segment limit → scales to many more threads
  - CAS for most puts → literally zero contention for non-colliding keys
  - Finer-grained locking (per-bucket) → much less contention
```

### 🔧 **ConcurrentMap Atomic Operations**

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

// ── STANDARD MAP OPERATIONS (all thread-safe) ──
map.put("key", 1);
map.get("key");
map.remove("key");
map.containsKey("key");

// ── ATOMIC COMPOUND OPERATIONS (ConcurrentMap interface) ──

// putIfAbsent — atomic: only put if key is absent
// Returns current value if present, null if successfully added
Integer existing = map.putIfAbsent("counter", 0); // atomic check-then-put

// replace — atomic: update only if current value matches
boolean updated = map.replace("key", 1, 2); // if value==1, set to 2, returns true

// remove — conditional atomic: remove only if value matches
boolean removed = map.remove("key", 2); // remove if value==2

// compute — atomic: compute new value
map.compute("counter", (k, v) -> v == null ? 1 : v + 1); // atomic increment!

// merge — atomic: perfect for counters
map.merge("word", 1, Integer::sum); // word count: atomic read-increment-write

// computeIfAbsent — atomic: initialize only if absent
map.computeIfAbsent("list", k -> new ArrayList<>()); // WARNING: not ideal!

// ── PARALLEL OPERATIONS (Java 8+) ──
map.forEach(2, (k, v) -> process(k, v)); // parallelism threshold=2
long sum = map.reduceValues(1, Integer::sum);    // parallel sum
String maxKey = map.search(1, (k,v) -> v > 100 ? k : null); // parallel search
```

### 🔧 **Thread-Safe Counter Pattern**

```java
// ❌ WRONG — race condition!
map.put("hits", map.getOrDefault("hits", 0) + 1); // NOT atomic!

// ✅ CORRECT — all atomic:
// Option 1: ConcurrentHashMap.merge()
map.merge("hits", 1, Integer::sum);

// Option 2: computeIfAbsent + AtomicInteger
map.computeIfAbsent("hits", k -> new AtomicInteger(0)).incrementAndGet();

// Option 3: LongAdder for high-contention counters (Java 8+)
ConcurrentHashMap<String, LongAdder> counters = new ConcurrentHashMap<>();
counters.computeIfAbsent("hits", k -> new LongAdder()).increment();
long total = counters.get("hits").sum();
```

### 🔧 **ConcurrentHashMap as a Set**

```java
// Get a Set backed by ConcurrentHashMap (thread-safe Set):
Set<String> concurrentSet = ConcurrentHashMap.newKeySet();
concurrentSet.add("element");         // thread-safe
concurrentSet.contains("element");    // thread-safe
concurrentSet.remove("element");      // thread-safe

// This is how you create a thread-safe HashSet equivalent!
```

### ❌ **Why Null Is Not Allowed**

```java
// ❌ ConcurrentHashMap rejects null keys/values:
ConcurrentHashMap<String, String> map = new ConcurrentHashMap<>();
map.put(null, "value"); // NullPointerException
map.put("key", null);   // NullPointerException

// WHY? In concurrent code, null is AMBIGUOUS:
// Thread A: containsKey("key") → false
// Thread B: map.put("key", null)
// Thread A: get("key") → null (but does null mean "not found" or "null value"?)
// Without null, get() returning null ALWAYS means "not present" → no ambiguity

// HashMap allows null because single-threaded code can use containsKey() to disambiguate.
// ConcurrentHashMap: containsKey() + get() together is NOT atomic anyway!
```

---

## 📄 CopyOnWriteArrayList — For Read-Heavy Workloads

### 📖 **Definition**

`CopyOnWriteArrayList<E>` implements `List<E>` with a bold strategy: every write operation creates a **complete copy of the underlying array**. This ensures reads are NEVER blocked.

```
CopyOnWriteArrayList characteristics:
✅ All reads are completely non-blocking (read from snapshot array)
✅ Iteration is safe — iterates a snapshot, never ConcurrentModificationException
✅ Thread-safe for all operations
✅ Allows null elements
❌ Writes are VERY EXPENSIVE: O(n) — copies entire array every time!
❌ Memory-intensive: holds old and new array simultaneously during writes
❌ Iterator snapshot may be STALE (doesn't reflect concurrent writes)

Use when: reads >> writes (event listeners, configuration)
Avoid when: frequent writes (VERY slow!)
```

### 🔧 **How Copy-On-Write Works**

```java
// Simplified CopyOnWriteArrayList.add():
public boolean add(E e) {
    synchronized (lock) {
        Object[] elements = getArray();
        int len = elements.length;
        // COPY the ENTIRE array:
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;  // add element to copy
        setArray(newElements); // atomic reference swap
        return true;
    }
    // Readers still reading the OLD array see the old state — that's fine!
    // After setArray(), new readers see the new array (volatile write)
}

// Reader:
public E get(int index) {
    return elementAt(getArray(), index); // reads current volatile array reference
    // Absolutely NO locking, NO blocking!
}
```

```
WRITE SCENARIO (4 threads):
  Threads A,B,C: actively reading (iterating over old array snapshot)
  Thread D: calls add("new")

  Thread D:
    1. Acquires lock
    2. Copies entire array: [a,b,c] → [a,b,c,new]
    3. Atomically swaps array reference (volatile write)
    4. Releases lock
  
  Threads A,B,C: still iterate the OLD [a,b,c] snapshot — no disruption!
  Next read by anyone: sees [a,b,c,new]
```

### 🔧 **CopyOnWriteArrayList Usage**

```java
CopyOnWriteArrayList<String> listeners = new CopyOnWriteArrayList<>();

// ── READS (concurrent, non-blocking) ──
listeners.get(0);                 // O(1) — no lock
listeners.contains("Alice");      // O(n) — no lock
listeners.size();                 // O(1) — no lock

// ── ITERATION (always safe, snapshot semantics) ──
for (String listener : listeners) {      // SAFE! Iterates a snapshot.
    notifyListener(listener);           // Even if other threads add/remove during loop
    // Iterator will NEVER throw ConcurrentModificationException
    // But may NOT see elements added AFTER iteration started
}

// ── WRITES (expensive! O(n)) ──
listeners.add("Bob");           // copies entire array!
listeners.remove("Alice");      // copies entire array!
listeners.set(0, "Charlie");    // copies entire array!

// ── WHEN TO USE ──
// ✅ Event listener lists (registered once, fired many times)
// ✅ Spring ApplicationListener list
// ✅ Thread-safe list of subscribers/observers
// ✅ Configuration snapshots

// ❌ DON'T USE when writes are frequent!
```

---

## 📋 CopyOnWriteArraySet

```java
// CopyOnWriteArraySet = Set backed by CopyOnWriteArrayList
CopyOnWriteArraySet<String> set = new CopyOnWriteArraySet<>();
set.add("Alice");
set.add("Bob");
set.add("Alice"); // duplicate silently ignored

// ⚠️ WARNING: add() scans the backing list for duplicates → O(n)!
// For large sets, prefer ConcurrentHashMap.newKeySet() (O(1) contains):
Set<String> fastSet = ConcurrentHashMap.newKeySet();

// CopyOnWriteArraySet is only suitable for VERY SMALL sets
// where iteration (no CME) is more important than write performance.
```

---

## 🔗 ConcurrentLinkedQueue & Deque

### 🔧 **Non-Blocking Queue**

```java
// ConcurrentLinkedQueue — unbounded, non-blocking, thread-safe
// Uses Michael-Scott queue algorithm (CAS-based, no locks!)
Queue<String> queue = new ConcurrentLinkedQueue<>();

queue.offer("task1");  // thread-safe, non-blocking
queue.offer("task2");

String next = queue.poll(); // thread-safe, null if empty

// ── PROPERTIES ──
// ✅ Non-blocking (uses CAS)
// ✅ High throughput for producers and consumers
// ✅ Unbounded
// ❌ size() is O(n)! (no atomic counter — count requires full traversal)
// ❌ No blocking "wait until element available"

// Use when: multiple producers and consumers, high throughput, no need for blocking
// Use BlockingQueue when: need to wait for elements

// ConcurrentLinkedDeque — double-ended version:
Deque<String> cld = new ConcurrentLinkedDeque<>();
cld.offerFirst("front");
cld.offerLast("back");
cld.pollFirst();
cld.pollLast();
```

---

## 🚧 BlockingQueue Family

### 📖 **Key Implementations**

```
IMPLEMENTATION            │ BOUNDED │ BACKING       │ NOTES
──────────────────────────┼─────────┼───────────────┼────────────────────────────────
ArrayBlockingQueue        │ ✅ Yes  │ Circular array│ Fair/unfair, single lock pair
LinkedBlockingQueue       │ Optional│ Linked nodes  │ Separate locks for put/take
LinkedBlockingDeque       │ Optional│ Doubly-linked │ Both ends, single lock
SynchronousQueue          │ 0       │ None          │ Direct handoff (rendezvous)
PriorityBlockingQueue     │ ❌ No   │ Binary heap   │ Priority ordering + blocking
DelayQueue                │ ❌ No   │ PriorityQueue │ Release after delay expiry
LinkedTransferQueue       │ ❌ No   │ Linked nodes  │ Better than SynchronousQueue
```

```java
// ── ArrayBlockingQueue — bounded FIFO, classic producer-consumer ──
BlockingQueue<Task> abq = new ArrayBlockingQueue<>(1000); // capacity=1000

// Producer (blocks if queue is FULL):
abq.put(new Task("job1"));    // blocks until space available
abq.offer(new Task("job2"));  // returns false if full (non-blocking)
abq.offer(task, 5, TimeUnit.SECONDS); // wait up to 5 seconds

// Consumer (blocks if queue is EMPTY):
Task task = abq.take();       // blocks until element available
Task t2 = abq.poll();         // null if empty (non-blocking)
Task t3 = abq.poll(5, TimeUnit.SECONDS); // wait up to 5 seconds

// ── LinkedBlockingQueue — common default choice ──
BlockingQueue<String> lbq = new LinkedBlockingQueue<>();     // unbounded
BlockingQueue<String> lbqBounded = new LinkedBlockingQueue<>(500); // bounded

// WHY LinkedBlockingQueue often beats ArrayBlockingQueue:
// ABQ: single ReentrantLock for both put and take → producers/consumers compete
// LBQ: SEPARATE locks for put (tail) and take (head) → producer/consumer don't block each other!

// ── SynchronousQueue — zero-capacity handoff ──
BlockingQueue<String> sq = new SynchronousQueue<>();
// put() BLOCKS until another thread calls take()
// take() BLOCKS until another thread calls put()
// Direct thread-to-thread handoff — like a passing a baton
// Used by: Executors.newCachedThreadPool()

// ── DelayQueue — delayed release ──
BlockingQueue<ScheduledTask> dq = new DelayQueue<>();
dq.put(new ScheduledTask(task, 5000)); // available after 5 seconds
// dq.take() blocks until the delay expires!
// Use for: scheduled retry, TTL cache eviction, timer tasks
```

---

## 🔀 ConcurrentSkipListMap & Set

### 📖 **Sorted Concurrent Collections**

```java
// ConcurrentSkipListMap — concurrent sorted map (like thread-safe TreeMap)
ConcurrentNavigableMap<String, Integer> skipMap = new ConcurrentSkipListMap<>();
skipMap.put("banana", 2);
skipMap.put("apple", 1);
skipMap.put("cherry", 3);

skipMap.firstKey();       // "apple" — O(log n)
skipMap.lastKey();        // "cherry"
skipMap.headMap("cherry");  // {apple:1, banana:2}
skipMap.tailMap("banana");  // {banana:2, cherry:3}
skipMap.floorKey("b");    // "banana"
skipMap.ceilingKey("ca"); // "cherry"

// ── ConcurrentSkipListSet — concurrent sorted set ──
ConcurrentSkipListSet<Integer> skipSet = new ConcurrentSkipListSet<>();
skipSet.addAll(List.of(5, 2, 8, 1, 9, 3));
System.out.println(skipSet); // [1, 2, 3, 5, 8, 9] — sorted!

// ── SKIP LIST INTERNALS ──
// A skip list is a probabilistic data structure with layered linked lists
// Layer 0: all elements    [1]→[2]→[3]→[5]→[8]→[9]
// Layer 1: ~50% of elements [1]→    [3]→[5]→    [9]
// Layer 2: ~25% of elements [1]→         [5]→
// This enables O(log n) operations WITHOUT tree rebalancing locks!
// More concurrent-friendly than Red-Black Trees

// ConcurrentSkipListMap vs TreeMap:
//   TreeMap: O(log n) with single-thread access
//   ConcurrentSkipListMap: O(log n) with full concurrent access (no global lock)
```

---

## ⚖️ Collections.synchronizedXxx() vs Concurrent Collections

```
COLLECTIONS.synchronizedXxx()  │ CONCURRENT COLLECTIONS
────────────────────────────────┼──────────────────────────────────────────────────
Simple wrapper — adds           │ Purpose-built for concurrency
synchronized to every method    │

Global lock (synchronized on    │ Fine-grained locking / lock-free (CAS)
the collection object)          │

Compound operations NOT atomic! │ ConcurrentMap has atomic compound operations
  synchronized(list) {          │   map.putIfAbsent(), compute(), merge()
    if (!list.contains(x))      │
      list.add(x);              │
  }                             │

MUST manually sync iteration:   │ Iteration is always safe (snapshot or CAS)
  synchronized(list) {          │   No manual sync needed
    for (E e : list) { ... }    │
  }                             │

Performance: one global lock    │ Performance: striped/CAS, high concurrency
  → ALL threads block on same   │   → many threads can work simultaneously

Simpler API (just wrap)         │ More methods, richer API
```

```java
// synchronizedList example:
List<String> sync = Collections.synchronizedList(new ArrayList<>());
sync.add("a");  // thread-safe individual add

// DANGER: iteration requires external sync!
synchronized (sync) {              // MUST synchronize externally!
    for (String s : sync) {
        process(s);
    }
}
// Forgetting the synchronized block → ConcurrentModificationException!

// CopyOnWriteArrayList example:
List<String> cowl = new CopyOnWriteArrayList<>();
cowl.add("a");  // thread-safe

for (String s : cowl) { // SAFE! No external sync needed!
    process(s);
}
```

---

## 🎯 Choosing the Right Concurrent Collection

```
NEED                              │ COLLECTION
──────────────────────────────────┼──────────────────────────────────────────
Thread-safe key-value map         │ ConcurrentHashMap (almost always!)
Thread-safe sorted map            │ ConcurrentSkipListMap
Thread-safe sorted set            │ ConcurrentSkipListSet
Thread-safe Set (fast contains)   │ ConcurrentHashMap.newKeySet()
Thread-safe Set (tiny, rare write)│ CopyOnWriteArraySet
Thread-safe List (rare write)     │ CopyOnWriteArrayList
Thread-safe List (frequent write) │ Collections.synchronizedList (but be careful)
Producer-consumer queue (bounded) │ ArrayBlockingQueue or LinkedBlockingQueue
Producer-consumer (high perf)     │ LinkedTransferQueue
Direct thread handoff             │ SynchronousQueue
Scheduled tasks / TTL             │ DelayQueue
Non-blocking concurrent queue     │ ConcurrentLinkedQueue
Non-blocking concurrent deque     │ ConcurrentLinkedDeque
Priority + thread-safe            │ PriorityBlockingQueue
```

---

## ⏱️ Time & Space Complexity

```
COLLECTION              │ GET/CONTAINS │ PUT/ADD    │ REMOVE     │ NOTES
────────────────────────┼──────────────┼────────────┼────────────┼────────────────────────
ConcurrentHashMap       │ O(1)         │ O(1)       │ O(1)       │ CAS for no-collision puts
CopyOnWriteArrayList    │ O(1) get     │ O(n)       │ O(n)       │ Copy entire array on write
CopyOnWriteArraySet     │ O(n)         │ O(n)       │ O(n)       │ Backed by CopyOnWrite list
ConcurrentLinkedQueue   │ O(n)         │ O(1)       │ O(n)       │ size() is O(n)!
ArrayBlockingQueue      │ O(n)         │ O(1)       │ O(1)       │ put/take may block
LinkedBlockingQueue     │ O(n)         │ O(1)       │ O(1)       │ Separate lock for put/take
ConcurrentSkipListMap   │ O(log n)     │ O(log n)   │ O(log n)   │ Expected (probabilistic)
ConcurrentSkipListSet   │ O(log n)     │ O(log n)   │ O(log n)   │ Sorted concurrent set
```

---

## ⚠️ Common Pitfalls

### 💣 **Pitfall 1: Compound Operations on ConcurrentHashMap Are Not Atomic**

```java
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();

// ❌ NOT ATOMIC — race condition!
// Thread A: gets value 5
// Thread B: gets value 5
// Thread A: puts 6 (5+1)
// Thread B: puts 6 (5+1) — lost update!
Integer v = map.get("key");
map.put("key", v == null ? 1 : v + 1);

// ✅ ATOMIC — use merge() or compute()
map.merge("key", 1, Integer::sum);                        // atomic increment
map.compute("key", (k, v) -> v == null ? 1 : v + 1);     // atomic compute
map.computeIfPresent("key", (k, v) -> v + 1);             // only if exists
```

### 💣 **Pitfall 2: ConcurrentHashMap.size() Is Not Consistent**

```java
// In a concurrent context, size() may not reflect the "current" count:
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
// Thread 1: map.put("a", 1)
// Thread 2: map.size() — might see 0 or 1 depending on timing!
// size() returns an estimate, not a guaranteed atomic snapshot!

// ✅ For exact counting, use a separate AtomicLong counter
AtomicLong count = new AtomicLong();
map.merge("key", 1, (a, b) -> { count.incrementAndGet(); return a + b; });
```

### 💣 **Pitfall 3: CopyOnWriteArrayList Not for Frequent Writes**

```java
// ❌ TERRIBLE performance — each add copies entire array!
List<String> logs = new CopyOnWriteArrayList<>();
while (true) {
    logs.add(generateLogEntry()); // adds ~10,000/sec → O(n) copy each time!
}
// After 1M entries: each add copies 1M elements!
// O(1) + O(2) + ... + O(n) = O(n²) total

// ✅ For high-write logging: use ConcurrentLinkedQueue or queue-based approach
Queue<String> logs = new ConcurrentLinkedQueue<>();
```

### 💣 **Pitfall 4: synchronizedList Forgetting External Sync for Iteration**

```java
List<String> syncList = Collections.synchronizedList(new ArrayList<>());

// ❌ Thread 2 can add between hasNext() and next():
Iterator<String> it = syncList.iterator(); // iterator is NOT synchronized!
while (it.hasNext()) {
    it.next(); // ConcurrentModificationException if another thread modifies!
}

// ✅ Synchronize the entire iteration:
synchronized (syncList) {
    Iterator<String> it2 = syncList.iterator();
    while (it2.hasNext()) {
        process(it2.next()); // safe
    }
}
// OR: Use CopyOnWriteArrayList which handles this automatically
```

---

## 🏢 Industry Applications

```
COMPANY/FRAMEWORK    USE CASE                                COLLECTION
────────────────────────────────────────────────────────────────────────────────────────
Netflix              User session store (high concurrency)   ConcurrentHashMap
Spring Boot          Application context bean registry       ConcurrentHashMap
Kafka                Consumer group offset tracking          ConcurrentHashMap
Akka                 Actor mailbox                           Non-blocking queue
JVM JIT Compiler     Class compilation cache                 ConcurrentHashMap
Nginx (Java client)  Connection pool                         ConcurrentLinkedQueue
Spring Events        ApplicationListener registry            CopyOnWriteArrayList
Logback              Logger hierarchy cache                  ConcurrentHashMap
Guava                CacheBuilder (internal)                 ConcurrentHashMap segments
ThreadPoolExecutor   Work queue                              LinkedBlockingQueue (default)
ScheduledExecutor    Scheduled task storage                  DelayQueue
```

### 💻 **Real Spring Boot: Event Bus with ConcurrentHashMap + CopyOnWriteArrayList**

```java
@Service
public class EventBus {
    
    // Topic → List of handlers
    // ConcurrentHashMap: thread-safe registration/lookup
    // CopyOnWriteArrayList: thread-safe iteration during publish
    private final Map<String, CopyOnWriteArrayList<EventHandler>> handlers =
        new ConcurrentHashMap<>();
    
    // Subscribe to a topic (happens at startup, rarely)
    public void subscribe(String topic, EventHandler handler) {
        handlers.computeIfAbsent(topic, k -> new CopyOnWriteArrayList<>())
                .add(handler);
    }
    
    // Publish to a topic (happens very frequently)
    public void publish(String topic, Event event) {
        List<EventHandler> topicHandlers = handlers.getOrDefault(
            topic, new CopyOnWriteArrayList<>()
        );
        // CopyOnWriteArrayList: safe iteration, no ConcurrentModificationException
        // Even if subscribe() is called from another thread during publish!
        for (EventHandler handler : topicHandlers) {
            try {
                handler.handle(event);
            } catch (Exception e) {
                log.error("Handler failed for topic {}", topic, e);
            }
        }
    }
}
```

### 💻 **ConcurrentHashMap Word Count**

```java
@Service
public class TextAnalysisService {
    
    public Map<String, Long> countWordsConcurrently(List<String> texts) {
        ConcurrentHashMap<String, Long> wordCount = new ConcurrentHashMap<>();
        
        // Parallel stream with atomic merge
        texts.parallelStream()
             .flatMap(text -> Arrays.stream(text.split("\\s+")))
             .map(String::toLowerCase)
             .forEach(word -> wordCount.merge(word, 1L, Long::sum));
        // merge() is atomic — safe for parallel execution!
        
        return wordCount;
    }
    
    // For very high contention: use LongAdder per key
    public Map<String, Long> countWithLongAdder(List<String> texts) {
        ConcurrentHashMap<String, LongAdder> adders = new ConcurrentHashMap<>();
        
        texts.parallelStream()
             .flatMap(text -> Arrays.stream(text.split("\\s+")))
             .forEach(word -> 
                 adders.computeIfAbsent(word, k -> new LongAdder()).increment()
             );
        
        // Collect final counts
        return adders.entrySet().stream()
                     .collect(Collectors.toMap(
                         Map.Entry::getKey,
                         e -> e.getValue().sum()
                     ));
    }
}
```

---

## 💡 Interview Tips

**Q1: How does Java 8 ConcurrentHashMap differ from Java 7's?**
```
Java 7: Segment-based locking
  - 16 Segments (like 16 mini-HashMaps), each with ReentrantLock
  - Maximum concurrency: 16 write threads simultaneously
  - concurrencyLevel parameter controlled number of segments

Java 8: CAS + per-bin synchronization
  - No segments — uses the main table directly
  - For empty bins: CAS (compareAndSwap) to insert — zero lock contention!
  - For non-empty bins: synchronized on the FIRST NODE of that bin only
  - Maximum concurrency: as many threads as there are bins (up to n)
  - Reads are completely non-blocking (volatile reads)
  - Resize is parallel (multiple threads cooperate on transfer)
  - Size tracking uses CounterCell array (like LongAdder) for accuracy
```

**Q2: When would you choose CopyOnWriteArrayList over synchronizedList?**
```
CopyOnWriteArrayList when:
  - Reads are MUCH more frequent than writes (>100:1 ratio)
  - Iteration must be safe without external synchronization
  - Occasional adds (e.g., event listeners registered at startup)
  - You can tolerate stale reads during concurrent writes

synchronizedList when:
  - Writes are frequent (CopyOnWrite is O(n) per write!)
  - You need up-to-date reads at all times
  - The list is small (so O(n) copy is acceptable)
  - You're wrapping an existing list

Neither when:
  - High-frequency concurrent reads AND writes → use ConcurrentLinkedQueue
    or restructure with immutable data patterns
```

**Q3: Why is ConcurrentHashMap.size() not reliable?**
```
In Java 8, ConcurrentHashMap tracks its size using a distributed counter:
  - Each stripe/cell has a counter
  - put() increments one cell, remove() decrements one cell
  - size() sums all cells

But size() is just a SNAPSHOT — by the time you read it,
another thread may have added or removed elements!

Contrast: For atomic size you'd need to lock the entire map:
  synchronized(map) { return map.size(); }
But that defeats the purpose of ConcurrentHashMap!

For most use cases, the "approximate" size is fine.
If you need exact concurrent counting, maintain a separate AtomicLong.
```

**Q4: What is the difference between poll() and take() in BlockingQueue?**
```
poll():
  - Returns null immediately if queue is empty
  - Non-blocking
  - Use for: "get if available, continue if not"

poll(timeout, unit):
  - Waits up to timeout for an element
  - Returns null if still empty after timeout
  - Use for: "try for N seconds, give up gracefully"

take():
  - BLOCKS indefinitely until an element is available
  - Throws InterruptedException when thread is interrupted
  - Use for: worker threads that should always wait for work

put():  BLOCKS if queue is full  ↔  take(): BLOCKS if queue is empty
offer(): false if full           ↔  poll():  null if empty
```

---

*Previous: [PriorityQueue Deep Dive ←](./09_PriorityQueue.md) | Next: [Collections Utilities & Streams →](./11_Collections_Utilities_Streams.md)*
