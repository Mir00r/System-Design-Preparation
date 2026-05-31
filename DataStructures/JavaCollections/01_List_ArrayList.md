# 📋 Java ArrayList: The Dynamic Array Masterclass 🚀

---

## 🎯 Chapter 1: List Interface & ArrayList Deep Dive

> **"ArrayList is to Java what a Swiss Army knife is to a programmer — it handles 80% of your needs with simplicity, speed, and elegance."**

---

## 📋 Table of Contents

1. [The List Interface Contract](#the-list-interface-contract)
2. [What is ArrayList?](#what-is-arraylist)
3. [Internal Architecture](#internal-architecture)
4. [Creation & Initialization](#creation--initialization)
5. [All Methods Explained](#all-methods-explained)
6. [Dynamic Resizing — The Magic](#dynamic-resizing)
7. [Iteration Patterns](#iteration-patterns)
8. [Real-World Analogies](#real-world-analogies)
9. [When to USE ArrayList](#when-to-use-arraylist)
10. [When to AVOID ArrayList](#when-to-avoid-arraylist)
11. [Time & Space Complexity](#time--space-complexity)
12. [Java Version Features](#java-version-features)
13. [Common Pitfalls](#common-pitfalls)
14. [Industry Applications](#industry-applications)
15. [Practice Problems](#practice-problems)
16. [Interview Tips](#interview-tips)

---

## 🔌 The List Interface Contract

### 📖 **What List Guarantees**

The `List<E>` interface extends `Collection<E>` and adds **index-based access** to elements. A List is an **ordered collection** (sequence) that:

```
✅ Maintains insertion order (elements stay in the order you add them)
✅ Allows DUPLICATE elements
✅ Allows NULL elements (usually)
✅ Supports positional access: get(index), set(index, element)
✅ Supports positional search: indexOf(), lastIndexOf()
✅ Supports range operations: subList(from, to)
```

### 🗺️ **List Interface Methods (Full API)**

```java
// java.util.List<E> — extends Collection<E>
public interface List<E> extends Collection<E> {
    // POSITIONAL ACCESS
    E get(int index);                   // O(1) for ArrayList, O(n) for LinkedList
    E set(int index, E element);        // replace element at index, return old
    void add(int index, E element);     // insert at index, shift right
    E remove(int index);                // remove at index, shift left, return removed
    
    // SEARCH
    int indexOf(Object o);              // first occurrence, -1 if not found
    int lastIndexOf(Object o);          // last occurrence, -1 if not found
    
    // RANGE
    List<E> subList(int fromIndex, int toIndex); // view (not copy!) of portion
    
    // ITERATION
    ListIterator<E> listIterator();     // bidirectional iterator
    ListIterator<E> listIterator(int index); // start at position
    
    // INHERITED FROM Collection:
    // add(E), remove(Object), contains(Object), size(), isEmpty()
    // iterator(), toArray(), clear(), addAll(), removeAll(), retainAll()
    
    // STATIC FACTORY (Java 9+)
    static <E> List<E> of(E... elements);    // UNMODIFIABLE list
    static <E> List<E> copyOf(Collection<? extends E> coll); // Java 10
}
```

---

## 🤔 What is ArrayList?

### 📖 **Definition**

`ArrayList<E>` is the most commonly used collection class in Java. It implements the `List` interface using a **dynamic array** — an array that automatically grows when full.

```
ArrayList is:
✅ Index-based (O(1) random access)
✅ Ordered (insertion order preserved)
✅ Allows duplicates
✅ Allows null values
✅ NOT thread-safe (use CopyOnWriteArrayList or Collections.synchronizedList() if needed)
✅ Implements: List, RandomAccess, Cloneable, Serializable
```

**Introduced**: Java 1.2 (1998)  
**Package**: `java.util`  
**Signature**: `public class ArrayList<E> extends AbstractList<E> implements List<E>, RandomAccess, Cloneable, Serializable`

---

## 🏗️ Internal Architecture

### 🔧 **What's Inside an ArrayList**

```java
// Simplified ArrayList source code (Java 17)
public class ArrayList<E> extends AbstractList<E>
        implements List<E>, RandomAccess, Cloneable, Serializable {

    private static final int DEFAULT_CAPACITY = 10;
    private static final Object[] EMPTY_ELEMENTDATA = {};
    private static final Object[] DEFAULTCAPACITY_EMPTY_ELEMENTDATA = {};

    Object[] elementData;    // THE ACTUAL ARRAY — package private for nested class access
    private int size;        // NUMBER OF ELEMENTS (NOT array length!)
    protected int modCount;  // for fail-fast iterator (from AbstractList)

    // Default constructor — DOES NOT allocate array yet!
    public ArrayList() {
        this.elementData = DEFAULTCAPACITY_EMPTY_ELEMENTDATA; // lazy init
    }

    // Constructor with initial capacity
    public ArrayList(int initialCapacity) {
        if (initialCapacity > 0) {
            this.elementData = new Object[initialCapacity];
        } else if (initialCapacity == 0) {
            this.elementData = EMPTY_ELEMENTDATA;
        } else {
            throw new IllegalArgumentException("Illegal Capacity: " + initialCapacity);
        }
    }
}
```

### 📊 **Memory Layout Visualization**

```
ArrayList<String> list = new ArrayList<>();
list.add("Alice");
list.add("Bob");
list.add("Charlie");

MEMORY LAYOUT:
┌────────────────────────────────────────────────────────────────┐
│  ArrayList object                                              │
│  ┌─────────────────┐                                          │
│  │ elementData  ───┼───→ Object[] (capacity=10)               │
│  │ size = 3         │    ┌─────────┬─────────┬─────────┬──────┤
│  │ modCount = 3     │    │"Alice"  │ "Bob"   │"Charlie"│null.7│
│  └─────────────────┘    └─────────┴─────────┴─────────┴──────┘
│                           index: 0     1         2        3-9  │
└────────────────────────────────────────────────────────────────┘

Key insight:
  - elementData.length = 10 (CAPACITY)
  - size = 3 (ACTUAL elements)
  - indices 3-9 are null but allocated!
```

---

## 🚀 Creation & Initialization

### 🔧 **All Ways to Create an ArrayList**

```java
import java.util.*;
import java.util.stream.*;

// 1. Default constructor (capacity 10, lazy init)
ArrayList<String> list1 = new ArrayList<>();

// 2. With initial capacity (avoids resizing for known sizes)
ArrayList<String> list2 = new ArrayList<>(1000); // pre-allocate for 1000 elements

// 3. From another collection
List<String> source = List.of("a", "b", "c");
ArrayList<String> list3 = new ArrayList<>(source);

// 4. Array to ArrayList
String[] arr = {"x", "y", "z"};
ArrayList<String> list4 = new ArrayList<>(Arrays.asList(arr));
// NOTE: Arrays.asList() returns FIXED-SIZE list (no add/remove)
// wrapping in ArrayList makes it resizable

// 5. Java 8+ — using streams
ArrayList<String> list5 = Stream.of("a", "b", "c")
    .collect(Collectors.toCollection(ArrayList::new));

// 6. Java 9+ immutable factories (NOT ArrayList!)
List<String> immutable = List.of("a", "b", "c"); // throws UnsupportedOperationException on add

// 7. Java 10+ — copy of
List<String> copy = List.copyOf(source); // immutable copy

// 8. Double-brace initialization — DON'T USE IN PRODUCTION
List<String> bad = new ArrayList<>() {{  // creates anonymous subclass! memory leak risk
    add("a"); add("b");
}};
```

---

## 🛠️ All Methods Explained

### ➕ **Adding Elements**

```java
ArrayList<String> list = new ArrayList<>(List.of("a", "b", "c"));

// add(E e) — append to end, O(1) amortized
list.add("d");               // ["a","b","c","d"]

// add(int index, E e) — insert at index, O(n) shift
list.add(1, "X");            // ["a","X","b","c","d"]

// addAll(Collection c) — append all, O(k)
list.addAll(List.of("e", "f")); // ["a","X","b","c","d","e","f"]

// addAll(int index, Collection c) — insert all at index, O(n+k)
list.addAll(2, List.of("Y", "Z")); // inserts at index 2

// Java 21 — addFirst/addLast (SequencedCollection)
list.addFirst("FIRST");
list.addLast("LAST");
```

### 🔍 **Accessing Elements**

```java
ArrayList<String> list = new ArrayList<>(List.of("a", "b", "c", "b"));

// get(int index) — O(1) direct array access
String elem = list.get(0);         // "a"
String last = list.get(list.size() - 1);  // "b"

// Java 21 — getFirst/getLast
String first = list.getFirst();    // "a"
String lastE = list.getLast();     // "b"

// indexOf(Object o) — first occurrence, O(n)
int idx = list.indexOf("b");       // 1

// lastIndexOf(Object o) — last occurrence, O(n)
int lastIdx = list.lastIndexOf("b"); // 3

// contains(Object o) — O(n) linear scan
boolean has = list.contains("c");  // true

// size()
int sz = list.size();              // 4

// isEmpty()
boolean empty = list.isEmpty();    // false
```

### ✏️ **Updating Elements**

```java
ArrayList<String> list = new ArrayList<>(List.of("a", "b", "c"));

// set(int index, E element) — O(1), returns old value
String old = list.set(1, "B");  // old="b", list=["a","B","c"]

// replaceAll(UnaryOperator) — Java 8+, O(n)
list.replaceAll(String::toUpperCase);  // ["A","B","C"]

// sort in-place — Java 8+
list.sort(Comparator.naturalOrder());   // sorts ascending
list.sort(Comparator.reverseOrder());   // sorts descending
list.sort(Comparator.comparingInt(String::length)); // sort by length
```

### ➖ **Removing Elements**

```java
ArrayList<String> list = new ArrayList<>(List.of("a", "b", "c", "b", "d"));

// remove(int index) — O(n) shift, returns removed element
String removed = list.remove(1); // removed="b", list=["a","c","b","d"]

// remove(Object o) — O(n) scan + shift, returns true/false
// ⚠️ IMPORTANT: for Integer list, remove(int) vs remove(Object) ambiguity!
list.remove("c");                // removes first "c"

// removeAll(Collection c) — O(n*m)
list.removeAll(List.of("b", "d")); // removes all occurrences of "b" and "d"

// retainAll(Collection c) — keep only elements in c
list.retainAll(List.of("a"));    // list=["a"]

// removeIf(Predicate) — Java 8+, O(n)
list.removeIf(s -> s.startsWith("a")); // remove all starting with "a"

// clear() — O(n), sets all refs to null (GC friendly)
list.clear();

// Java 21
list.removeFirst();  // remove first element
list.removeLast();   // remove last element
```

### 🔎 **Querying & Conversion**

```java
ArrayList<String> list = new ArrayList<>(List.of("a", "b", "c"));

// toArray() — returns Object[]
Object[] arr = list.toArray();

// toArray(T[]) — returns typed array
String[] strArr = list.toArray(new String[0]); // pass zero-length array for optimal
String[] strArr2 = list.toArray(String[]::new); // Java 11+ method reference syntax

// subList(fromIndex, toIndex) — LIVE VIEW, NOT a copy!
List<String> sub = list.subList(0, 2); // ["a","b"] — backed by original!
sub.set(0, "X");  // MODIFIES original list too!

// iterator() and listIterator()
Iterator<String> it = list.iterator();
ListIterator<String> lit = list.listIterator(); // bidirectional

// stream() — Java 8+
list.stream()
    .filter(s -> s.compareTo("b") > 0)
    .forEach(System.out::println);
```

### ⚠️ **Integer List Remove Trap**

```java
List<Integer> nums = new ArrayList<>(List.of(1, 2, 3, 4, 5));

// TRAP: remove(int index) vs remove(Integer value)
nums.remove(2);               // removes by INDEX 2 → removes value 3!
nums.remove(Integer.valueOf(2)); // removes by VALUE 2 → removes the number 2

// RULE: pass Integer.valueOf(x) to remove by VALUE
// Java auto-selects remove(int) over remove(Object) for plain int literals
```

---

## 📈 Dynamic Resizing — The Magic

### 🔧 **How Growth Works**

```java
// When add() is called and size == elementData.length:
private void grow(int minCapacity) {
    int oldCapacity = elementData.length;
    // NEW CAPACITY = old + (old >> 1) = old * 1.5
    int newCapacity = oldCapacity + (oldCapacity >> 1);
    
    if (newCapacity - minCapacity < 0)
        newCapacity = minCapacity;
    if (newCapacity - MAX_ARRAY_SIZE > 0)
        newCapacity = hugeCapacity(minCapacity);
        
    elementData = Arrays.copyOf(elementData, newCapacity);
    // Uses System.arraycopy internally — FAST native copy
}
```

```
GROWTH PATTERN:
Initial capacity (first add): 10
After 10 elements:  resize to 15   (10 * 1.5)
After 15 elements:  resize to 22   (15 * 1.5)
After 22 elements:  resize to 33
After 33 elements:  resize to 49
After 49 elements:  resize to 73
...

COST:
  - Each resize: O(n) copy operation
  - BUT amortized over n adds: O(1) per add
  
WHY 1.5x and not 2x?
  - 2x: Java 6 and earlier used this
  - 1.5x: Java 7+ uses this — more memory-efficient
  - Too small (e.g. 1.1x): too many resizes
  - Too large (e.g. 2x): wasted memory
```

### 🎯 **Performance Optimization: Pre-sizing**

```java
// ❌ Without pre-sizing: may resize multiple times
List<String> list = new ArrayList<>();
for (String s : readMillionRecordsFromDB()) {  // resizes ~20 times!
    list.add(s);
}

// ✅ With pre-sizing: zero resizes, saves time + memory
int count = getCountFromDB(); // know the size in advance
List<String> list = new ArrayList<>(count);
for (String s : readRecordsFromDB()) {
    list.add(s);
}

// trimToSize() — shrink internal array to actual size
list.trimToSize(); // releases unused capacity (saves memory)

// ensureCapacity(int minCapacity) — pro-actively grow
list.ensureCapacity(10000); // tell ArrayList "I'll add 10000 elements soon"
```

---

## 🔄 Iteration Patterns

```java
List<String> list = new ArrayList<>(List.of("a", "b", "c", "d"));

// 1. Classic for loop — fastest for ArrayList (cache-friendly, no overhead)
for (int i = 0; i < list.size(); i++) {
    System.out.println(list.get(i));
}

// 2. Enhanced for-each (uses iterator internally)
for (String s : list) {
    System.out.println(s);
}

// 3. forEach with lambda — Java 8+
list.forEach(s -> System.out.println(s));
list.forEach(System.out::println);  // method reference

// 4. Iterator — useful when you need to remove during iteration
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    String s = it.next();
    if (s.equals("b")) {
        it.remove(); // SAFE removal during iteration
    }
}

// 5. ListIterator — bidirectional
ListIterator<String> lit = list.listIterator(list.size()); // start at end
while (lit.hasPrevious()) {
    System.out.println(lit.previous()); // iterate backwards
}

// 6. Stream — Java 8+
list.stream()
    .filter(s -> !s.equals("c"))
    .map(String::toUpperCase)
    .collect(Collectors.toList());

// 7. Parallel Stream (large datasets)
list.parallelStream()
    .filter(s -> s.length() > 1)
    .collect(Collectors.toList());
```

---

## 🌍 Real-World Analogies

### 📦 **The Warehouse Shelf Analogy**

```
ArrayList is like a numbered warehouse shelf system:

SHELF (elementData array):
┌────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐
│ #1 │ #2 │ #3 │ #4 │ #5 │ #6 │ #7 │ #8 │ #9 │#10 │  ← CAPACITY (reserved)
│📦  │📦  │📦  │📦  │    │    │    │    │    │    │
└────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘
 SIZE=4                    ↑ empty slots (reserved but unused)

✅ "Get item from shelf #3" → O(1) — direct access by number
✅ "Add item to next empty slot" → O(1) amortized
❌ "Insert item between #2 and #3" → O(n) — must shift everything right
❌ "Remove item from #2" → O(n) — must shift everything left
✅ "Count items on shelf" → O(1)

When shelf is FULL:
  Manager builds a NEW shelf with 1.5x capacity,
  MOVES all items (System.arraycopy),
  DISCARDS old shelf.
  → This is why insertion is O(1) AMORTIZED
```

### 🚌 **Bus Seating Analogy**

```
Imagine a stretchable bus (adds seats automatically):

Seats: [Alice][Bob][Charlie][___][___]   ← 5 seats total, 3 passengers

- "Who's in seat 2?" → O(1): direct lookup
- "Add Dave at the back" → O(1): seat 4 is empty
- "Insert Eve between Alice and Bob" → O(n): shift Bob, Charlie right
  [Alice][Eve][Bob][Charlie][___]
- "Remove Bob" → O(n): shift Charlie left
  [Alice][Eve][Charlie][___][___]
- "Is Charlie on the bus?" → O(n): check every seat

When bus is full: add 5 more seats (System.arraycopy = move everyone)
```

---

## ✅ When to USE ArrayList

```
✅ USE ArrayList WHEN:
─────────────────────────────────────────────────────────────────────
✅ You need FAST RANDOM ACCESS: list.get(i) — O(1)
   Example: array[i] style access patterns

✅ You mostly ADD TO THE END: list.add(element) — O(1) amortized
   Example: collecting query results, building response lists

✅ You need to ITERATE through the entire list (most common)
   ArrayList is CPU cache-friendly (contiguous memory)
   Iteration is ~2x faster than LinkedList in practice

✅ You need INDEX-BASED operations: set, indexOf, subList

✅ You know the approximate size (use new ArrayList<>(capacity))
   Example: processing a batch of 10,000 records

✅ The data is READ FREQUENTLY but WRITTEN INFREQUENTLY

REAL EXAMPLES:
  - REST API response bodies: new ArrayList<>(queryResults)
  - Collecting stream results: .collect(Collectors.toList())
  - JSON arrays: Jackson/Gson deserializes arrays as ArrayList
  - Spring JPA: findAll() returns ArrayList
  - In-memory search results
  - Building UI list components
```

---

## ❌ When to AVOID ArrayList

```
❌ AVOID ArrayList WHEN:
─────────────────────────────────────────────────────────────────────
❌ FREQUENT INSERTIONS/DELETIONS AT THE BEGINNING OR MIDDLE
   Why: Every insert/delete shifts O(n) elements
   Cost: For a list of 1,000,000 elements, inserting at index 0
         requires copying 1,000,000 references!
   Use: LinkedList (if you have node references), ArrayDeque (for stack/queue)

❌ YOU NEED A QUEUE (FIFO) OR STACK
   Why: Removing from front (index 0) is O(n) — not suitable for queue
   Use: ArrayDeque — O(1) for both ends

❌ MULTI-THREADED CONCURRENT ACCESS WITHOUT SYNCHRONIZATION
   Why: ArrayList is not thread-safe. Two threads adding simultaneously
        can corrupt internal state (lost updates, ArrayIndexOutOfBoundsException)
   Use: CopyOnWriteArrayList (read-heavy), Collections.synchronizedList()
        (write-heavy), or ConcurrentLinkedQueue

❌ STORING PRIMITIVES IN PERFORMANCE-CRITICAL CODE
   Why: int/long/double get autoboxed to Integer/Long/Double
        Each element becomes a heap-allocated object with 16-byte overhead
   Use: int[] or long[] arrays directly, or IntStream

❌ VERY LARGE DATASETS WHERE MEMORY MATTERS
   Why: ArrayList may hold up to 49% unused capacity (after grow)
   Use: list.trimToSize() to reclaim, or use a more compact structure
```

---

## ⏱️ Time & Space Complexity

```
OPERATION                │ TIME COMPLEXITY  │ NOTES
─────────────────────────┼──────────────────┼──────────────────────────────────
get(int index)           │ O(1)             │ Direct array access
set(int index, E e)      │ O(1)             │ Direct array write
add(E e) — append        │ O(1) amortized   │ O(n) when resize happens
add(int i, E e) — insert │ O(n)             │ Shifts elements right
remove(int index)        │ O(n)             │ Shifts elements left
remove(Object o)         │ O(n)             │ Linear scan + shift
contains(Object o)       │ O(n)             │ Linear scan
indexOf(Object o)        │ O(n)             │ Linear scan (forward)
lastIndexOf(Object o)    │ O(n)             │ Linear scan (backward)
size()                   │ O(1)             │
isEmpty()                │ O(1)             │
iterator.next()          │ O(1)             │
sort()                   │ O(n log n)       │ TimSort (merge+insertion hybrid)
toArray()                │ O(n)             │ System.arraycopy
clear()                  │ O(n)             │ Nulls all refs for GC
subList()                │ O(1)             │ Returns a view, not a copy
─────────────────────────┼──────────────────┼──────────────────────────────────
SPACE: O(n) for n elements. Internal array may hold up to 50% extra capacity.
RandomAccess marker: JVM can use binary search and optimize accordingly.
```

---

## 📅 Java Version Features

```
JAVA VERSION  │ FEATURE
──────────────┼─────────────────────────────────────────────────────────────
Java 1.2      │ ArrayList introduced, generics via Object (unsafe)
Java 5        │ Generics: ArrayList<String> — type-safe, no casts
              │ Autoboxing: list.add(5) works (int → Integer)
              │ Enhanced for-each loop
Java 6        │ Arrays.copyOf() used internally
Java 7        │ Diamond operator: new ArrayList<>()
              │ Growth factor changed from 2x to 1.5x
Java 8        │ forEach(Consumer), removeIf(Predicate), replaceAll(UnaryOperator)
              │ sort(Comparator), stream(), spliterator()
              │ Collectors.toList() (returns ArrayList)
Java 9        │ List.of("a","b","c") — immutable, NOT ArrayList
              │ List.of() is backed by different internal classes
Java 10       │ List.copyOf() — immutable copy
              │ var list = new ArrayList<String>() — local type inference
Java 11       │ toArray(IntFunction<T[]>) — new overload
Java 14       │ Collectors.toUnmodifiableList()
Java 16       │ Stream.toList() — returns unmodifiable List (more efficient)
Java 21       │ SequencedCollection: getFirst(), getLast(), addFirst(), addLast()
              │ reversed() — reversed view
```

---

## ⚠️ Common Pitfalls

### 💣 **Pitfall 1: ConcurrentModificationException**

```java
List<String> list = new ArrayList<>(List.of("a", "b", "c"));

// ❌ BAD — modifying while iterating
for (String s : list) {           // uses iterator internally
    if (s.equals("b")) {
        list.remove(s);           // ConcurrentModificationException!
    }
}

// ✅ GOOD — use removeIf
list.removeIf(s -> s.equals("b"));

// ✅ GOOD — use iterator's remove
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if (it.next().equals("b")) it.remove();
}
```

### 💣 **Pitfall 2: subList is a VIEW**

```java
List<String> list = new ArrayList<>(List.of("a", "b", "c", "d"));
List<String> sub = list.subList(1, 3); // ["b","c"]

sub.set(0, "X");     // MODIFIES list[1] → list is now ["a","X","c","d"]
sub.clear();         // MODIFIES original list → ["a","d"]

list.add("e");       // ConcurrentModificationException when accessing sub!
// subList becomes invalid after structural modification to backing list
```

### 💣 **Pitfall 3: Arrays.asList() is Fixed-Size**

```java
String[] arr = {"a", "b", "c"};
List<String> fixed = Arrays.asList(arr);  // Fixed-size backed list!
fixed.add("d");   // UnsupportedOperationException!
fixed.set(0, "X"); // OK — set is fine
arr[0] = "Y";      // MODIFIES fixed list! They share the same array.

// CORRECT: wrap in ArrayList
List<String> resizable = new ArrayList<>(Arrays.asList(arr));
```

### 💣 **Pitfall 4: Autoboxing Performance Trap**

```java
// ❌ BAD — 1M autoboxing operations
List<Integer> list = new ArrayList<>();
for (int i = 0; i < 1_000_000; i++) {
    list.add(i);  // Each 'i' creates new Integer object!
}
// Memory: 1M Integer objects × ~16 bytes = ~16 MB

// ✅ BETTER for primitives
int[] arr = new int[1_000_000];
// or use IntStream:
List<Integer> list2 = IntStream.range(0, 1_000_000)
    .boxed().collect(Collectors.toList()); // still autoboxes, but cleaner
```

---

## 🏢 Industry Applications

```
COMPANY       USE CASE                                    HOW
──────────────────────────────────────────────────────────────────────────
Netflix       Video search results                        new ArrayList<>(jdbcResults)
Amazon        Shopping cart items                         ArrayList<CartItem>
Google Search Search autocomplete suggestions             ArrayList<String> sorted
Spring Boot   @GetMapping return type                     service returns List<Dto>
Hibernate     One-to-many relationships                   @OneToMany → ArrayList
Jackson       JSON array deserialization                  JSON [] → ArrayList
Android       RecyclerView / ListView adapter             ArrayList<Model>
Elasticsearch Hit results per shard                       ArrayList<SearchHit>
```

### 💻 **Real Spring Boot Example**

```java
@RestController
@RequestMapping("/api/products")
public class ProductController {
    
    @Autowired
    private ProductRepository productRepository;
    
    @GetMapping
    public List<ProductDto> getAllProducts(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size) {
        
        // JPA returns List<Product> (backed by ArrayList)
        List<Product> products = productRepository.findAll(
            PageRequest.of(page, size)
        ).getContent();
        
        // Pre-sized ArrayList for O(0) resizing
        List<ProductDto> result = new ArrayList<>(products.size());
        
        for (Product p : products) {
            result.add(new ProductDto(p.getId(), p.getName(), p.getPrice()));
        }
        
        return result;
    }
    
    @GetMapping("/search")
    public List<ProductDto> searchProducts(@RequestParam String query) {
        // Filter in-memory using removeIf (avoids ConcurrentModificationException)
        List<Product> all = new ArrayList<>(productRepository.findAll());
        all.removeIf(p -> !p.getName().toLowerCase().contains(query.toLowerCase()));
        return all.stream()
            .map(p -> new ProductDto(p))
            .collect(Collectors.toList());
    }
}
```

---

## 🎯 Practice Problems

| # | Problem | Difficulty | Key Pattern |
|---|---------|-----------|-------------|
| 1 | [Two Sum](https://leetcode.com/problems/two-sum/) | 🟢 Easy | HashMap + ArrayList |
| 2 | [Remove Duplicates from Sorted Array](https://leetcode.com/problems/remove-duplicates-from-sorted-array/) | 🟢 Easy | Two pointers on array |
| 3 | [Merge Sorted Array](https://leetcode.com/problems/merge-sorted-array/) | 🟢 Easy | Two pointers, merge |
| 4 | [Find All Anagrams in a String](https://leetcode.com/problems/find-all-anagrams-in-a-string/) | 🟡 Medium | Sliding window, ArrayList result |
| 5 | [Group Anagrams](https://leetcode.com/problems/group-anagrams/) | 🟡 Medium | HashMap<String, List<String>> |
| 6 | [Rotate Array](https://leetcode.com/problems/rotate-array/) | 🟡 Medium | In-place manipulation |
| 7 | [Subsets](https://leetcode.com/problems/subsets/) | 🟡 Medium | List<List<Integer>> |
| 8 | [Spiral Matrix](https://leetcode.com/problems/spiral-matrix/) | 🟡 Medium | List of Lists |
| 9 | [Merge Intervals](https://leetcode.com/problems/merge-intervals/) | 🟡 Medium | Sort + merge overlapping |

---

## 💡 Interview Tips & Tricks

> **⚡ Top 10 ArrayList Interview Answers:**

**Q1: How does ArrayList grow internally?**
```
Answer: new capacity = old + (old >> 1) = 1.5x growth.
Uses Arrays.copyOf() → System.arraycopy() (native copy).
Amortized O(1) per add because resizing cost is spread over n elements.
Proof: Total copies for n insertions ≤ n + n/1.5 + n/1.5² + ... = O(n)
```

**Q2: ArrayList vs LinkedList — when to use which?**
```
ArrayList:  Random access O(1), add at end O(1), ITERATION IS FASTER
LinkedList: insert/delete at known position O(1), but traversal to find position is O(n)
            ONLY use LinkedList as Deque (addFirst, removeLast) — never as List!
RULE: 99% of the time use ArrayList. LinkedList wins only as queue/deque.
```

**Q3: How to make ArrayList thread-safe?**
```java
// Option 1: synchronizedList — wraps all methods in synchronized block
List<String> sync = Collections.synchronizedList(new ArrayList<>());
// MUST manually synchronize iteration:
synchronized (sync) {
    for (String s : sync) { ... } // otherwise ConcurrentModificationException
}

// Option 2: CopyOnWriteArrayList — snapshot semantics
List<String> cowList = new CopyOnWriteArrayList<>();
// All writes create a NEW copy of the array → safe iteration always
// Best when reads >> writes
```

**Q4: What is the RandomAccess marker interface?**
```
RandomAccess is a marker interface (no methods).
ArrayList implements it to signal that get(index) is O(1).
Collections.binarySearch() checks: if (list instanceof RandomAccess)
  → use index-based binary search (ArrayList)
  → else use iterator-based scan (LinkedList)
JVM and library algorithms can optimize behavior based on this marker.
```

**Q5: Why is iteration faster on ArrayList than LinkedList?**
```
ArrayList: elementData is a contiguous array in memory.
CPU prefetches entire cache line (64 bytes = ~8 Java references).
Iterator steps through adjacent memory → CPU cache HOT.

LinkedList: each node is a separate heap object at a random address.
Every node.next is a pointer chase to a random memory location.
CPU cache MISS on almost every step → 5-10x slower iteration.
```

---

*Previous: [Overview & Architecture ←](./00_Overview_and_Architecture.md) | Next: [LinkedList Deep Dive →](./02_LinkedList.md)*
