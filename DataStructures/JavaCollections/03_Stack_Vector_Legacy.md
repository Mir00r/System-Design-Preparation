# ⚠️ Java Stack, Vector & Legacy Classes: What NOT to Use 🚫

---

## 🎯 Chapter 3: Legacy Collections — The Anti-Patterns

> **"Those who cannot remember the past are condemned to repeat it. In Java, those who don't understand Vector and Stack will keep using them in new code — and pay the performance price."**

---

## 📋 Table of Contents

1. [Why Study Legacy Classes?](#why-study-legacy-classes)
2. [Vector — The Over-Synchronized ArrayList](#vector)
3. [Stack — The Broken Stack Implementation](#stack)
4. [Hashtable — The Over-Synchronized HashMap](#hashtable)
5. [Properties — The Surviving Legacy Class](#properties)
6. [The Modern Replacements](#modern-replacements)
7. [Performance Comparison](#performance-comparison)
8. [When Legacy Code Appears (How to Handle It)](#handling-legacy-code)
9. [Interview Tips](#interview-tips)

---

## 🤔 Why Study Legacy Classes?

```
3 REASONS:
─────────────────────────────────────────────────────────────────────
1. INTERVIEWS: "What's wrong with Stack/Vector?" is a common question
2. LEGACY CODEBASES: You WILL encounter them in 10+ year old Java code
3. HISTORY: Understanding WHY they failed teaches you API design lessons
```

### 📅 **Timeline of the Legacy Problem**

```
1996 — Java 1.0 released with Vector and Hashtable (both synchronized)
1997 — Java 1.1: Stack added (extending Vector — a design mistake)
1998 — Java 1.2: Collections Framework released (ArrayList, HashMap, etc.)
       Vector and Hashtable "retrofitted" to implement List and Map interfaces
       BUT their thread-safety design was already flawed
2004 — Java 5: java.util.concurrent released (proper thread-safe collections)
TODAY — Vector/Stack/Hashtable still work but should NEVER be used in new code
```

---

## 📦 Vector — The Over-Synchronized ArrayList

### 📖 **What is Vector?**

`Vector<E>` is the synchronized predecessor of `ArrayList`. It was created in Java 1.0 when multi-threading was a primary concern, before Java's memory model was fully understood.

```
Vector characteristics:
✅ Dynamic array (same as ArrayList)
✅ Index-based access
✅ Allows duplicates and null
❌ ALL methods are synchronized → massive overhead even in single-threaded code
❌ Grows by 2x (vs ArrayList's 1.5x) → more memory waste
❌ "Synchronized" ≠ "thread-safe" for compound operations
❌ Cannot configure growth factor
```

### 🏗️ **Internal Difference from ArrayList**

```java
// ArrayList.add() — no synchronization
public boolean add(E e) {
    modCount++;
    add(e, elementData, size);
    return true;
}

// Vector.add() — synchronized on the ENTIRE method
public synchronized boolean add(E e) {
    modCount++;
    add(e, elementData, elementCount);
    return true;
}
```

```
PERFORMANCE IMPACT:
  Single-threaded ArrayList.add() × 1,000,000 = ~85ms
  Single-threaded Vector.add()    × 1,000,000 = ~210ms (2.5x SLOWER!)

WHY? synchronized acquires and releases a mutex lock on EVERY call,
     even when there's only ONE thread. Lock acquisition is expensive
     (CPU memory barrier, potential OS syscall).

AND IT'S STILL NOT PROPERLY THREAD-SAFE!
```

### 💣 **Why Synchronized ≠ Thread-Safe**

```java
Vector<Integer> vector = new Vector<>();

// Thread 1:
if (!vector.isEmpty()) {        // ← synchronized check
    vector.get(0);              // ← CAN THROW IndexOutOfBoundsException!
}
// Thread 2 can remove the element BETWEEN the isEmpty() and get(0) calls!
// Each method is synchronized, but the COMPOUND OPERATION is not atomic.

// PROOF: This race condition exists despite every method being synchronized!
// Proper fix: synchronize on the vector EXTERNALLY for compound ops:
synchronized (vector) {
    if (!vector.isEmpty()) {
        vector.get(0); // safe now
    }
}
// But if you're doing this anyway, just use ArrayList with explicit sync!
```

### 🔧 **Vector-Specific Methods (Now Redundant)**

```java
Vector<String> v = new Vector<>();
v.addElement("a");     // = add() — old name
v.removeElement("a");  // = remove(Object)
v.elementAt(0);        // = get(0)
v.setElementAt("b",0); // = set(0,"b")
v.insertElementAt("c",1); // = add(1,"c")
v.firstElement();      // = get(0)
v.lastElement();       // = get(size()-1)
v.capacity();          // current array capacity (not size)
v.ensureCapacity(100); // same as ArrayList
v.trimToSize();        // same as ArrayList

Enumeration<String> e = v.elements(); // OLD iteration — replaced by iterator()
while (e.hasMoreElements()) {
    System.out.println(e.nextElement());
}
```

### ✅ **Modern Replacement for Vector**

```java
// ❌ DON'T:
Vector<String> v = new Vector<>();

// ✅ DO (choose based on your threading needs):

// Single-threaded:
List<String> list = new ArrayList<>();

// Read-heavy multi-threaded:
List<String> cowList = new CopyOnWriteArrayList<>();

// Write-heavy multi-threaded (need compound atomicity):
List<String> syncList = Collections.synchronizedList(new ArrayList<>());
synchronized (syncList) {
    for (String s : syncList) { ... } // must externally sync for iteration
}
```

---

## 📚 Stack — The Broken Stack Implementation

### 📖 **What is Stack?**

`Stack<E>` extends `Vector<E>` to implement LIFO (Last-In-First-Out) operations. It was added in Java 1.1 as a quick add-on.

```
The DESIGN MISTAKE:
┌────────────────────────────────────────────────────────────────────┐
│  Stack<E> extends Vector<E>                                        │
│                                                                    │
│  Stack inherits ALL Vector methods including:                      │
│    add(index, element) — WRONG for a stack!                        │
│    get(index)          — WRONG for a stack!                        │
│    remove(index)       — WRONG for a stack!                        │
│    set(index, element) — WRONG for a stack!                        │
│                                                                    │
│  A REAL stack should ONLY allow: push, pop, peek, isEmpty, size    │
│  Stack breaks the Liskov Substitution Principle!                   │
└────────────────────────────────────────────────────────────────────┘
```

### 🔧 **Stack Methods**

```java
Stack<Integer> stack = new Stack<>();

// Stack-specific methods:
stack.push(1);         // = addElement (add to top)
stack.push(2);
stack.push(3);

int top = stack.peek();    // = lastElement() — view without removing → 3
int popped = stack.pop();  // = removeElement at top → returns 3, stack=[1,2]
boolean empty = stack.empty(); // = isEmpty()
int pos = stack.search(1); // distance from TOP (1-based) → 2 (1 is 2 from top)
                            // returns -1 if not found

// ⚠️ Problem: ALL Vector methods are also accessible!
stack.add(0, 99);       // Inserts at bottom! Breaks stack semantics!
stack.remove(0);        // Removes from bottom! Breaks stack semantics!
stack.get(0);           // Accesses by index! Breaks stack semantics!

System.out.println(stack); // [99, 1, 2] — 99 was inserted at bottom!
```

### ✅ **Modern Replacement for Stack**

```java
// ❌ DON'T:
Stack<Integer> stack = new Stack<>();
stack.push(1);
int top = stack.pop();

// ✅ DO — Use Deque interface with ArrayDeque:
Deque<Integer> stack = new ArrayDeque<>();  // Interface type, not implementation!
stack.push(1);      // addFirst — adds to front (top of stack)
stack.push(2);
int top = stack.pop();  // removeFirst — returns 2 (LIFO)
int peek = stack.peek(); // peekFirst — view 1 without removing

// WHY ArrayDeque beats Stack:
// 1. No synchronization overhead (faster)
// 2. Pure stack operations only (no broken index access)
// 3. Correct "IS-A" relationship (is a Deque, not a List)
// 4. Can also be used as a Queue (versatile)
```

### 📊 **Stack vs ArrayDeque Performance**

```
OPERATION     │ Stack  │ ArrayDeque │ Ratio
──────────────┼────────┼────────────┼────────
push (1M ops) │ 310ms  │ 95ms       │ 3.3x faster
pop  (1M ops) │ 290ms  │ 88ms       │ 3.3x faster
peek (1M ops) │ 255ms  │ 72ms       │ 3.5x faster

ArrayDeque wins because: no synchronized overhead + cache-friendly array
```

---

## 🗝️ Hashtable — The Over-Synchronized HashMap

### 📖 **What is Hashtable?**

`Hashtable<K,V>` is the synchronized predecessor of `HashMap`. It predates the Collections Framework and has several significant limitations.

```
Hashtable vs HashMap:
────────────────────────────────────────────────────────────────────
FEATURE              │ Hashtable (Java 1.0)  │ HashMap (Java 1.2)
─────────────────────┼───────────────────────┼──────────────────────
Thread safety        │ Synchronized (slow)   │ NOT thread-safe
Null keys            │ ❌ NOT allowed         │ ✅ ONE null key
Null values          │ ❌ NOT allowed         │ ✅ Multiple null values
Performance          │ Slow (synchronized)   │ Fast (no locking)
Iteration order      │ No guaranteed order   │ No guaranteed order
Default capacity     │ 11                    │ 16
Load factor          │ 0.75                  │ 0.75
Collision resolution │ Linked list only      │ LL → Tree (Java 8)
Modern equivalent    │ ConcurrentHashMap     │ HashMap
```

```java
// WHY null is not allowed in Hashtable:
// When you call get(null), the method does: null.hashCode() → NullPointerException!
// HashMap has explicit null handling: hash(null) = 0

Hashtable<String, Integer> ht = new Hashtable<>();
ht.put(null, 1);     // NullPointerException!
ht.put("key", null); // NullPointerException!

// vs HashMap:
HashMap<String, Integer> hm = new HashMap<>();
hm.put(null, 1);     // OK! Stored at bucket 0
hm.put("key", null); // OK! Value can be null
```

### ✅ **Modern Replacement for Hashtable**

```java
// ❌ DON'T:
Hashtable<String, Integer> ht = new Hashtable<>();

// ✅ Single-threaded:
Map<String, Integer> map = new HashMap<>();

// ✅ Multi-threaded:
Map<String, Integer> concurrentMap = new ConcurrentHashMap<>();

// WHY ConcurrentHashMap beats Hashtable:
// Java 7: Segment-based locking (16 segments, 16 threads can write simultaneously)
// Java 8: CAS (Compare-And-Swap) + per-bin synchronized — near lock-free!
//         No need to lock the ENTIRE map for every operation
```

---

## 📝 Properties — The One Legacy Class Still in Use

### 📖 **What is Properties?**

`Properties` extends `Hashtable<Object, Object>` and is specifically designed for key=value configuration files. Unlike Vector/Stack/Hashtable, Properties is **still widely used today** for loading configuration.

```java
import java.util.Properties;
import java.io.*;

// ── CREATING AND WRITING ──
Properties props = new Properties();
props.setProperty("db.url", "jdbc:postgresql://localhost:5432/mydb");
props.setProperty("db.username", "admin");
props.setProperty("db.password", "secret");

// Write to file
try (OutputStream os = new FileOutputStream("config.properties")) {
    props.store(os, "Database Configuration");
}

// ── READING FROM FILE ──
Properties config = new Properties();
try (InputStream is = new FileInputStream("config.properties")) {
    config.load(is);
}
String url = config.getProperty("db.url");
String user = config.getProperty("db.username");
String missing = config.getProperty("missing.key", "default-value"); // with default

// ── READING FROM CLASSPATH (common in Spring) ──
Properties appProps = new Properties();
try (InputStream is = getClass().getResourceAsStream("/application.properties")) {
    appProps.load(is);
}

// ── LISTING ALL PROPERTIES ──
props.forEach((key, value) -> System.out.println(key + "=" + value));
props.propertyNames(); // Enumeration of all keys

// ── XML FORMAT ──
try (InputStream is = new FileInputStream("config.xml")) {
    props.loadFromXML(is);
}
```

### ⚠️ **Properties Gotcha — Inherited Hashtable Methods**

```java
Properties props = new Properties();

// ✅ USE THESE (type-safe):
props.setProperty("key", "value");     // String key, String value
String val = props.getProperty("key"); // returns String (null if missing)

// ❌ AVOID THESE (inherited from Hashtable, not type-safe!):
props.put("key", 42);        // puts Integer! Breaks String contract
props.get("key");            // returns Object, not String
props.getProperty("key");    // returns null! (because value is Integer, not String)

// RULE: Always use getProperty/setProperty, NEVER the inherited Hashtable methods
```

---

## 🔄 The Modern Replacements

```
LEGACY CLASS  │ REPLACEMENT          │ WHEN TO USE REPLACEMENT
──────────────┼──────────────────────┼─────────────────────────────────────────────
Vector        │ ArrayList            │ Single-threaded (99% of cases)
              │ CopyOnWriteArrayList │ Multi-threaded, read-heavy
              │ synchronizedList     │ Multi-threaded, write-heavy (manual sync)
──────────────┼──────────────────────┼─────────────────────────────────────────────
Stack         │ ArrayDeque           │ Always — faster, correct design
              │ Deque (interface)    │ Code to the interface
──────────────┼──────────────────────┼─────────────────────────────────────────────
Hashtable     │ HashMap              │ Single-threaded
              │ ConcurrentHashMap    │ Multi-threaded (always prefer this)
──────────────┼──────────────────────┼─────────────────────────────────────────────
Properties    │ Properties           │ Configuration files — STILL USE IT
              │ Spring @Value/@Bean  │ Spring Boot apps (prefer this)
              │ Map<String,String>   │ For non-file config
```

---

## 📊 Performance Comparison

```java
// Benchmark: 1,000,000 add() operations (single thread)

ArrayList:       ~85ms    ← WINNER
Vector:          ~210ms   (2.5x slower — synchronized overhead)
CopyOnWriteArrayList: ~4,500ms (creates new array copy every write!)

// Benchmark: 1,000,000 put() operations (single thread)

HashMap:         ~120ms   ← WINNER
ConcurrentHashMap: ~190ms (minor overhead for CAS operations)
Hashtable:       ~480ms   (4x slower — synchronized everything)

// Benchmark: 8 threads, 1,000,000 concurrent put() operations

ConcurrentHashMap: ~290ms ← WINNER (striped locking, parallel writes)
Hashtable:         ~1,800ms (global lock, all threads contend)
Collections.synchronizedMap(HashMap): ~1,750ms (same as Hashtable)
HashMap (unsafe!): CORRUPTED DATA / exceptions
```

---

## 🔧 Handling Legacy Code

### **Strategy 1: Encapsulate and Replace**

```java
// You find this in production code:
public class OrderService {
    private Vector<Order> orders = new Vector<>(); // legacy
    
    public synchronized void addOrder(Order o) {
        orders.add(o);
    }
    
    public Order getOrder(int index) {
        return orders.get(index);
    }
}

// Refactor — replace Vector with ArrayList (thread-safety was redundant here):
public class OrderService {
    private final List<Order> orders = new ArrayList<>();
    
    public synchronized void addOrder(Order o) { // keep sync if needed
        orders.add(o);
    }
    
    public Order getOrder(int index) {
        return orders.get(index);
    }
}
```

### **Strategy 2: Adapt for Modern APIs**

```java
// Legacy method returns Vector:
Vector<String> getLegacyData();

// Wrap it when consuming:
List<String> data = new ArrayList<>(getLegacyData()); // copy to ArrayList
// or just:
List<String> data = getLegacyData(); // Vector implements List — just assign!

// Legacy method returns Hashtable:
Hashtable<String, Integer> getLegacyMap();

// Wrap it:
Map<String, Integer> modern = new HashMap<>(getLegacyMap());
```

### **Strategy 3: Deal with Enumeration (Pre-Iterator)**

```java
// Old APIs return Enumeration (Java 1.0 iteration):
Enumeration<String> e = vector.elements();

// Modern way to iterate:
while (e.hasMoreElements()) {
    String s = e.nextElement();
}

// Or convert to Iterator:
Iterator<String> it = vector.iterator(); // Vector implements Iterable

// Or use Collections.list() to get ArrayList:
List<String> list = Collections.list(e); // converts Enumeration to ArrayList
```

---

## 💡 Interview Tips

**Q1: Why is Stack considered a poor design?**
```
Stack extends Vector, which means:
1. Stack inherits ALL Vector methods (add(index), get(index), set, etc.)
   A stack should ONLY support push/pop/peek — index access breaks encapsulation!
2. Violates the Liskov Substitution Principle
   You can't use Stack everywhere Vector is expected with correct semantics
3. All methods synchronized → performance cost even when not needed

Correct approach: Deque<T> stack = new ArrayDeque<>();
ArrayDeque: NO inheritance from List, pure Deque operations, no synchronization.
```

**Q2: Why doesn't ConcurrentHashMap allow null keys/values?**
```
In a concurrent context, null is ambiguous:
  map.get("key") returns null — does it mean:
    (a) key is mapped to null value?
    (b) key doesn't exist?

In single-threaded HashMap, you can call containsKey() to distinguish.
But in a concurrent environment:
  Thread A: containsKey("key") → true
  Thread B: removes "key"
  Thread A: get("key") → null (key no longer exists!)

Doug Lea (author of ConcurrentHashMap) decided to ban null entirely
to force developers to be explicit, preventing subtle concurrency bugs.
```

**Q3: If Vector is thread-safe, why can this code fail?**
```java
Vector<Integer> v = new Vector<>();
v.add(1); v.add(2); v.add(3);

// Thread 1:                 // Thread 2:
if (!v.isEmpty()) {         v.clear();   // removes all!
    int x = v.get(0);  // ← IndexOutOfBoundsException!
}
```
```
Answer: Each individual method call (isEmpty, clear, get) is synchronized,
but the COMPOUND OPERATION "check-then-act" is not atomic.
Between isEmpty() and get(0), another thread can remove all elements.

Solution: synchronize externally on the vector for compound operations,
or use a ConcurrentSkipListList / CopyOnWriteArrayList depending on needs.
```

**Q4: Is there ANY reason to use Vector/Stack/Hashtable in new code?**
```
Vector:    NO. ArrayList or CopyOnWriteArrayList.
Stack:     NO. ArrayDeque as Deque.
Hashtable: NO. HashMap or ConcurrentHashMap.
Properties: YES — it's still the standard for .properties config files.
            (Though Spring's @Value and @ConfigurationProperties are preferred
             in modern Spring Boot apps.)
```

---

*Previous: [LinkedList Deep Dive ←](./02_LinkedList.md) | Next: [HashSet & LinkedHashSet →](./04_HashSet_LinkedHashSet.md)*
