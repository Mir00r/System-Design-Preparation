# 🔗 Java LinkedList: The Doubly-Linked Powerhouse 🚀

---

## 🎯 Chapter 2: LinkedList & Deque Deep Dive

> **"LinkedList is the Swiss Army knife of the collection hierarchy — it implements both List AND Deque, making it a list, a queue, a stack, and a double-ended queue all in one. But like a Swiss Army knife, it's often the wrong tool for a specific job."**

---

## 📋 Table of Contents

1. [What is LinkedList?](#what-is-linkedlist)
2. [Internal Architecture](#internal-architecture)
3. [LinkedList as List](#linkedlist-as-list)
4. [LinkedList as Deque](#linkedlist-as-deque)
5. [All Methods Explained](#all-methods-explained)
6. [Real-World Analogies](#real-world-analogies)
7. [ArrayList vs LinkedList Benchmark](#arraylist-vs-linkedlist-benchmark)
8. [When to USE LinkedList](#when-to-use-linkedlist)
9. [When to AVOID LinkedList](#when-to-avoid-linkedlist)
10. [Time & Space Complexity](#time--space-complexity)
11. [Common Pitfalls](#common-pitfalls)
12. [Industry Applications](#industry-applications)
13. [Practice Problems](#practice-problems)
14. [Interview Tips](#interview-tips)

---

## 🤔 What is LinkedList?

### 📖 **Definition**

`LinkedList<E>` is a **doubly-linked list** implementation that implements both `List<E>` and `Deque<E>`. Each element is stored in a `Node` that contains a reference to the **previous** and **next** node.

```
Characteristics:
✅ Doubly-linked (each node points to prev AND next)
✅ Implements BOTH List AND Deque (rare in collections!)
✅ O(1) insert/delete at both ENDS
✅ O(1) insert/delete at any known node position
❌ O(n) random access by index (no direct jump)
❌ High memory overhead (~40 bytes per element vs ~4-8 bytes for ArrayList)
❌ Poor cache locality (nodes scattered in memory)
❌ NOT thread-safe

Introduced: Java 1.2 (1998)
Package: java.util
Signature: public class LinkedList<E> extends AbstractSequentialList<E>
           implements List<E>, Deque<E>, Cloneable, Serializable
```

---

## 🏗️ Internal Architecture

### 🔧 **The Node Structure**

```java
// Exact source from OpenJDK:
private static class Node<E> {
    E item;          // The actual data
    Node<E> next;    // Reference to NEXT node
    Node<E> prev;    // Reference to PREVIOUS node

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

### 📊 **Memory Layout**

```
LinkedList<String> list = new LinkedList<>();
list.add("Alice");
list.add("Bob");
list.add("Charlie");

MEMORY LAYOUT (heap — nodes are at RANDOM addresses):
┌─────────────────────┐
│  LinkedList object  │
│  first  ────────────┼──→ ┌──────────────────────────┐
│  last   ──────────┐ │    │  Node @0x1A2B             │
│  size = 3         │ │    │  prev = null (HEAD)        │
└─────────────────────┘    │  item = "Alice"            │
                       │    │  next ─────────────────→  │ ┌──────────────────────┐
                       │    └──────────────────────────┘  │  Node @0x8F4C        │
                       │                                   │  prev ←──────────── │
                       │                                   │  item = "Bob"        │
                       │                                   │  next ─────────────→ │ ┌────────────────────┐
                       │                                   └──────────────────────┘ │  Node @0x3D7E       │
                       └──────────────────────────────────────────────────────────→ │  prev ←─────────── │
                                                                                     │  item = "Charlie"  │
                                                                                     │  next = null (TAIL)│
                                                                                     └────────────────────┘
LinkedList fields:
  first → Node("Alice")   ← HEAD
  last  → Node("Charlie") ← TAIL
  size  = 3

KEY: Each node is a SEPARATE heap object at a RANDOM memory address!
     This is why iteration is cache-unfriendly (vs ArrayList's contiguous array)
```

### 🔧 **LinkedList Class Structure (simplified)**

```java
public class LinkedList<E> extends AbstractSequentialList<E>
        implements List<E>, Deque<E>, Cloneable, Serializable {

    transient int size = 0;
    transient Node<E> first;  // POINTER TO HEAD
    transient Node<E> last;   // POINTER TO TAIL

    // Core insertion helper
    private void linkFirst(E e) {
        final Node<E> f = first;
        final Node<E> newNode = new Node<>(null, e, f); // prev=null, item=e, next=current_first
        first = newNode;
        if (f == null) last = newNode;   // was empty
        else f.prev = newNode;
        size++;
        modCount++;
    }

    private void linkLast(E e) {
        final Node<E> l = last;
        final Node<E> newNode = new Node<>(l, e, null); // prev=current_last, item=e, next=null
        last = newNode;
        if (l == null) first = newNode;  // was empty
        else l.next = newNode;
        size++;
        modCount++;
    }

    // Finding node by index (the O(n) bottleneck)
    Node<E> node(int index) {
        // OPTIMIZATION: search from nearest end
        if (index < (size >> 1)) {  // index in first half → start from front
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {                    // index in second half → start from back
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }
}
```

---

## 📋 LinkedList as List

### 🔧 **List Operations**

```java
LinkedList<String> list = new LinkedList<>();

// ── ADDING ──
list.add("c");           // append to end: ["c"]
list.add(0, "a");        // insert at index 0: ["a","c"]
list.add(1, "b");        // insert at index 1: ["a","b","c"]
list.addAll(List.of("d","e")); // append collection

// ── ACCESSING ──
String s = list.get(1);  // "b" — O(n)! traverses from nearest end
list.set(0, "A");        // replace at index 0: ["A","b","c"]
int idx = list.indexOf("b");    // 1
int last = list.lastIndexOf("b"); // -1 (only one "b")

// ── REMOVING ──
list.remove(1);           // remove at index 1: O(n) to find + O(1) to unlink
list.remove("A");         // remove by value: O(n) scan

// ── RANGE ──
List<String> sub = list.subList(0, 2); // view (backed by LinkedList)
```

### ⚠️ **Why list.get(i) is Slow for LinkedList**

```
list.get(500_000) in a list of 1,000,000 elements:

Step 1: index 500_000 >= size/2 (500_000)? YES → start from back
Step 2: traverse backwards from last node 500_000 times
Step 3: node.prev, node.prev, node.prev... (500_000 pointer jumps)
Step 4: each jump → random memory location → CPU cache MISS every time

TIME: ~milliseconds for ONE get() on large LinkedList
vs ArrayList: nanoseconds (direct array[i] lookup)

MORAL: NEVER use LinkedList as a random-access list!
```

---

## 🔄 LinkedList as Deque

### 🔧 **Deque (Double-Ended Queue) Operations**

This is where LinkedList SHINES — O(1) operations at both ends!

```java
LinkedList<String> deque = new LinkedList<>();

// ── ADDING TO FRONT (STACK PUSH / DEQUE addFirst) ──
deque.addFirst("c");        // ["c"]
deque.addFirst("b");        // ["b","c"]
deque.addFirst("a");        // ["a","b","c"]
deque.offerFirst("_");      // ["_","a","b","c"] (Deque API)
deque.push("X");            // ["X","_","a","b","c"] (Stack API — same as addFirst)

// ── ADDING TO BACK ──
deque.addLast("z");         // append (same as add())
deque.offerLast("zz");      // same as addLast (Queue API)
deque.offer("zzz");         // same as offerLast

// ── PEEKING (non-destructive) ──
String front = deque.peekFirst();  // "X" — no removal
String back  = deque.peekLast();   // "zzz"
String frontOrNull = deque.peek(); // same as peekFirst

// ── POLLING (destructive remove) ──
String polled = deque.pollFirst(); // removes and returns "X"
String polledLast = deque.pollLast(); // removes and returns "zzz"
String popped = deque.pop();       // removes from front (Stack API = pollFirst)

// ── THROWS EXCEPTIONS if empty (vs poll which returns null) ──
String removed = deque.removeFirst(); // throws NoSuchElementException if empty
String removedLast = deque.removeLast();
String element = deque.element();    // = getFirst(), throws if empty
```

### 🔧 **Deque API Summary**

```
OPERATION  │ Throws Exception │ Returns null/false │ Stack API
───────────┼──────────────────┼────────────────────┼──────────────
ADD FRONT  │ addFirst(e)      │ offerFirst(e)      │ push(e)
ADD BACK   │ addLast(e)       │ offerLast(e)       │ ──
REMOVE FRONT│ removeFirst()   │ pollFirst()        │ pop()
REMOVE BACK │ removeLast()    │ pollLast()         │ ──
PEEK FRONT │ getFirst()       │ peekFirst()        │ peek()
PEEK BACK  │ getLast()        │ peekLast()         │ ──

Queue API (from front to back):
ADD BACK   │ add(e)           │ offer(e)           │
REMOVE FRONT│ remove()        │ poll()             │
PEEK FRONT │ element()        │ peek()             │
```

---

## 🛠️ All Methods Explained

```java
LinkedList<Integer> list = new LinkedList<>();

// ───── List Methods ─────
list.add(1);                      // append to end
list.add(0, 0);                   // insert at index
list.addAll(List.of(2, 3));       // append all
list.get(1);                      // get at index — O(n)
list.set(0, 99);                  // set at index — O(n)
list.remove(0);                   // remove at index — O(n) to find
list.remove(Integer.valueOf(99)); // remove by value — O(n)
list.indexOf(2);                  // first occurrence — O(n)
list.lastIndexOf(2);              // last occurrence — O(n)
list.contains(3);                 // O(n)
list.size();                      // O(1)
list.isEmpty();                   // O(1)
list.clear();                     // O(n) — must null all node references
list.subList(0, 2);               // view

// ───── Deque Methods ─────
list.addFirst(0);                 // prepend — O(1)
list.addLast(100);                // append — O(1)
list.removeFirst();               // remove head — O(1), throws if empty
list.removeLast();                // remove tail — O(1), throws if empty
list.peekFirst();                 // view head — O(1), null if empty
list.peekLast();                  // view tail — O(1), null if empty
list.pollFirst();                 // remove+return head — O(1), null if empty
list.pollLast();                  // remove+return tail — O(1), null if empty
list.push(0);                     // = addFirst (Stack API)
list.pop();                       // = removeFirst (Stack API)
list.peek();                      // = peekFirst (Queue API)
list.offer(100);                  // = addLast (Queue API)
list.poll();                      // = pollFirst (Queue API)

// ───── Iterator Methods ─────
Iterator<Integer> it = list.iterator();
ListIterator<Integer> lit = list.listIterator();
Iterator<Integer> descIt = list.descendingIterator(); // reverse order

// ───── Conversion ─────
Object[] arr = list.toArray();
Integer[] typedArr = list.toArray(new Integer[0]);

// ───── Java 21 ─────
list.getFirst();    // = peekFirst but throws if empty
list.getLast();     // = peekLast but throws if empty
list.reversed();    // returns reversed view (new in Java 21)
```

---

## 🌍 Real-World Analogies

### 🚂 **Train Coaches (Doubly-Linked)**

```
HEAD                                          TAIL
 ↓                                             ↓
[Engine] ⟷ [Coach 1] ⟷ [Coach 2] ⟷ [Buffet] ⟷ [Last Coach]

✅ Add/remove coach at front or back: O(1)
   (just update the coupling at that end)
✅ Add/remove coach anywhere: O(1) — IF you're standing there already
   (just uncouple A-B, couple A-NEW and NEW-B)
❌ "What's in coach 47?" → must count from Engine or Last Coach
❌ Coaches scattered across different yards (poor cache locality)
```

### 📿 **Doubly-Linked Necklace**

```
[Clasp] ⟷ [Ruby] ⟷ [Emerald] ⟷ [Diamond] ⟷ [Sapphire] ⟷ [Clasp]
   HEAD                                               TAIL

✅ Add bead at either end: unclip clasp, attach, reclip → O(1)
✅ Remove any bead (if you're holding it): just update prev/next ← O(1)
❌ "3rd bead from the left?" → count from start → O(n)
❌ No "jump to position 3" — must traverse the chain
```

### 🎭 **Theater Queue Example**

```java
// VIP line management using LinkedList as Deque:
LinkedList<String> vipLine = new LinkedList<>();

// New VIPs arrive and get added to back
vipLine.offerLast("Alice");    // ["Alice"]
vipLine.offerLast("Bob");      // ["Alice","Bob"]
vipLine.offerLast("Charlie");  // ["Alice","Bob","Charlie"]

// Celebrity jumps to front
vipLine.offerFirst("Taylor Swift"); // ["Taylor Swift","Alice","Bob","Charlie"]

// First person enters venue
String enters = vipLine.pollFirst(); // "Taylor Swift"

// Bouncer checks who's next without removing
String next = vipLine.peekFirst();   // "Alice"
```

---

## 📊 ArrayList vs LinkedList Benchmark

```
TEST: 10,000,000 operations on each

OPERATION              │ ArrayList  │ LinkedList │ Winner
───────────────────────┼────────────┼────────────┼──────────────
Iterate all elements   │   45ms     │   240ms    │ ArrayList 5x
get(randomIndex)       │    2ms     │  5,400ms   │ ArrayList 2700x
add(e) at end          │   85ms     │   180ms    │ ArrayList (GC pressure)
add(0, e) at front     │ 3,200ms    │    95ms    │ LinkedList 34x
remove(0) at front     │ 3,100ms    │    50ms    │ LinkedList 62x
add at middle          │ 1,600ms    │ 2,700ms    │ ArrayList! (finding node is slow)

KEY INSIGHT: LinkedList "insert at middle O(1)" is MISLEADING.
You still need O(n/2) to FIND the position. The O(1) only applies
if you already have the iterator/node reference at that position!
```

---

## ✅ When to USE LinkedList

```
✅ USE LinkedList WHEN:
─────────────────────────────────────────────────────────────────────
✅ AS A DEQUE — frequent O(1) operations at BOTH ends
   Example: BFS queue, browser history, undo-redo stack
   (But prefer ArrayDeque — faster and more memory-efficient!)

✅ AS A QUEUE (FIFO) — if using List interface is required
   offerLast (enqueue) + pollFirst (dequeue) = O(1) each

✅ WHEN ITERATING WITH FREQUENT REMOVALS AT CURRENT POSITION
   Using iterator.remove() on LinkedList is O(1)
   (iterator holds the Node reference directly)

✅ IMPLEMENTING CUSTOM DATA STRUCTURES
   LRU Cache with HashMap + LinkedList (or use LinkedHashMap!)
   Music playlist (prev/next song in O(1))

HONEST ADVICE:
  In 95% of cases where you think you need LinkedList,
  ArrayDeque is the correct choice for queue/stack operations.
  LinkedList as a List is almost never the right call.
```

---

## ❌ When to AVOID LinkedList

```
❌ AVOID LinkedList WHEN:
─────────────────────────────────────────────────────────────────────
❌ YOU NEED RANDOM ACCESS: list.get(i)
   LinkedList.get(500) on 1M elements → traverses 500 nodes!
   Use ArrayList: O(1) array indexing

❌ SORTING THE LIST
   sort() internally copies to Object[], sorts, copies back
   (defeats the purpose of LinkedList)
   Use ArrayList for sortable data

❌ MEMORY IS CONSTRAINED
   Each node: 24 bytes overhead (object header + prev ref + next ref)
   1,000,000 integers in LinkedList: ~40MB (node overhead + Integer boxing)
   vs ArrayList: ~8MB (int[]) or ~20MB (Integer[])

❌ ITERATION-HEAVY CODE
   LinkedList iteration is 3-5x slower than ArrayList
   CPU cache misses on every node (random heap addresses)

❌ AS A GENERAL-PURPOSE STACK/QUEUE
   ArrayDeque is ALWAYS faster than LinkedList for queue/stack
   ArrayDeque: circular array, cache-friendly
   LinkedList: node-per-element, cache-unfriendly
   Use: Deque<String> stack = new ArrayDeque<>() — NOT LinkedList
```

---

## ⏱️ Time & Space Complexity

```
OPERATION              │ LinkedList  │ ArrayList   │ Notes
───────────────────────┼─────────────┼─────────────┼──────────────────────────────
get(int index)         │ O(n/2)      │ O(1)        │ LL: traverse from nearest end
set(int index, E e)    │ O(n/2)      │ O(1)        │ LL: must find node first
add(E e) — at end      │ O(1)        │ O(1)★       │ LL: update last pointer
add(0, E) — at front   │ O(1)        │ O(n)        │ LL: update first pointer
add(int i, E e)        │ O(n/2)+O(1) │ O(n)        │ LL: find + update pointers
remove(int i)          │ O(n/2)+O(1) │ O(n)        │ LL: find + unlink
removeFirst()          │ O(1)        │ O(n)        │ LL wins here!
removeLast()           │ O(1)        │ O(1)        │ Both fast
contains(Object o)     │ O(n)        │ O(n)        │ Linear scan both
indexOf(Object o)      │ O(n)        │ O(n)        │ Linear scan both
size()                 │ O(1)        │ O(1)        │
iterator.next()        │ O(1)        │ O(1)        │
iterator.remove()      │ O(1)        │ O(n)        │ LL wins — already at node!
sort()                 │ O(n log n)  │ O(n log n)  │ Both use TimSort
Iteration (loop)       │ O(n) slow   │ O(n) fast   │ ArrayList 3-5x faster (cache)
───────────────────────┴─────────────┴─────────────┴──────────────────────────────
SPACE: O(n) for both
  ArrayList:  n × (4 or 8 bytes per reference) + array overhead = ~n×8 bytes
  LinkedList: n × (24 bytes Node overhead + 8 bytes for data ref) = ~n×32 bytes
★ amortized
```

---

## ⚠️ Common Pitfalls

### 💣 **Pitfall 1: Using LinkedList where ArrayDeque is Better**

```java
// ❌ Common mistake
LinkedList<String> stack = new LinkedList<>();
stack.push("a");
stack.push("b");
String top = stack.pop();

// ✅ Correct — ArrayDeque is always faster
Deque<String> stack = new ArrayDeque<>();
stack.push("a");
stack.push("b");
String top = stack.pop();
```

### 💣 **Pitfall 2: Random Access Loop is O(n²)**

```java
LinkedList<String> list = new LinkedList<>(/* 100,000 elements */);

// ❌ CATASTROPHIC: O(n²) — for 100,000 elements: 10 BILLION operations!
for (int i = 0; i < list.size(); i++) {
    String s = list.get(i);  // Each get(i) traverses up to n/2 nodes!
    process(s);
}

// ✅ Use iterator for O(n) iteration
for (String s : list) {    // uses iterator internally, O(n) total
    process(s);
}
// or
list.forEach(s -> process(s));
```

### 💣 **Pitfall 3: Memory Leak via Node References**

```java
// LinkedList nodes strongly reference their data
// If you hold a reference to a node (via iterator), the element can't be GC'd

// Example: LinkedList used as LRU cache can be tricky
// Prefer LinkedHashMap.removeEldestEntry() for LRU instead
```

### 💣 **Pitfall 4: Forgetting Deque vs Queue Method Differences**

```java
LinkedList<String> q = new LinkedList<>();
q.add("a");

// THROWS NoSuchElementException if empty:
q.remove();    // = removeFirst()
q.element();   // = getFirst()

// Returns null if empty (SAFER):
q.poll();      // = pollFirst()
q.peek();      // = peekFirst()

// RULE: In production code, use poll/peek (null-safe) not remove/element
```

---

## 🏢 Industry Applications

```
USE CASE                              IMPLEMENTATION
──────────────────────────────────────────────────────────────────────────
BFS (Breadth-First Search)            Queue<Node> queue = new ArrayDeque<>()
                                      (ArrayDeque preferred over LinkedList)

Undo/Redo functionality               Deque<Action> history = new ArrayDeque<>()
                                      push for do, pop for undo

Browser back/forward                  history.push(currentUrl)
                                      back = history.pop()

Music playlist (prev/next)            LinkedList<Song> playlist
                                      (one of few legitimate LinkedList uses!)

Message queue processing              BlockingDeque<Message> in Java EE
                                      (prefer LinkedBlockingDeque for threads)

LRU Cache implementation              LinkedHashMap is BETTER than LinkedList+HashMap
                                      but understanding LinkedList helps explain LHM

Task scheduler                        Deque<Task> tasks — add urgent tasks to front
```

### 💻 **LRU Cache with LinkedHashMap (Better than LinkedList + HashMap)**

```java
// The "right way" — LinkedHashMap handles this internally:
class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;

    public LRUCache(int capacity) {
        super(capacity, 0.75f, true); // accessOrder=true → move to end on access
        this.capacity = capacity;
    }

    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity; // evict when over capacity
    }
}

// Usage:
LRUCache<Integer, String> cache = new LRUCache<>(3);
cache.put(1, "a");
cache.put(2, "b");
cache.put(3, "c");
cache.get(1);       // marks 1 as recently used
cache.put(4, "d");  // evicts entry 2 (least recently used)
```

### 💻 **Music Playlist (Legitimate LinkedList Use)**

```java
// Bidirectional navigation makes LinkedList natural here
class MusicPlayer {
    private LinkedList<Song> playlist = new LinkedList<>();
    private ListIterator<Song> position;

    public void addSong(Song song) {
        playlist.addLast(song);
        if (position == null) {
            position = playlist.listIterator();
        }
    }

    public Song next() {
        if (position.hasNext()) return position.next();
        return null; // end of playlist
    }

    public Song previous() {
        if (position.hasPrevious()) return position.previous();
        return null; // at beginning
    }

    public void insertNextInQueue(Song song) {
        position.add(song); // O(1) — insert at current iterator position!
    }
}
```

---

## 🎯 Practice Problems

| # | Problem | Difficulty | LinkedList Usage |
|---|---------|-----------|-----------------|
| 1 | [Reverse Linked List](https://leetcode.com/problems/reverse-linked-list/) | 🟢 Easy | Pointer manipulation |
| 2 | [Merge Two Sorted Lists](https://leetcode.com/problems/merge-two-sorted-lists/) | 🟢 Easy | Merge with pointers |
| 3 | [Linked List Cycle](https://leetcode.com/problems/linked-list-cycle/) | 🟢 Easy | Floyd's algorithm |
| 4 | [LRU Cache](https://leetcode.com/problems/lru-cache/) | 🟡 Medium | DoublyLinkedList + HashMap |
| 5 | [Add Two Numbers](https://leetcode.com/problems/add-two-numbers/) | 🟡 Medium | List traversal |
| 6 | [Flatten a Multilevel Doubly Linked List](https://leetcode.com/problems/flatten-a-multilevel-doubly-linked-list/) | 🟡 Medium | DLL manipulation |
| 7 | [Design Browser History](https://leetcode.com/problems/design-browser-history/) | 🟡 Medium | LinkedList as deque |
| 8 | [Design Circular Deque](https://leetcode.com/problems/design-circular-deque/) | 🟡 Medium | Full Deque impl |

---

## 💡 Interview Tips & Tricks

**Q1: LinkedList implements both List and Deque — what does that mean?**
```
It means LinkedList can be used as:
  - A List (indexed access, allows null/duplicates)
  - A Queue (FIFO via offer/poll)
  - A Stack (LIFO via push/pop)
  - A Deque (add/remove from both ends)

The Node structure naturally supports O(1) operations at both ends
because we maintain first and last pointers.
However, ArrayDeque is preferred for pure Queue/Stack use.
```

**Q2: Why is ArrayList.iterator().remove() O(n) but LinkedList.iterator().remove() O(1)?**
```
ArrayList: after finding the element (O(n)), removing it requires
  System.arraycopy to shift all subsequent elements left → O(n)

LinkedList: the iterator holds a reference to the current Node.
  To remove: prev.next = current.next, current.next.prev = prev → O(1)
  (just pointer updates, no shifting!)

This is the ONE case where LinkedList genuinely beats ArrayList.
```

**Q3: When would you actually choose LinkedList over ArrayDeque?**
```
When you need BOTH:
  1. The List interface (indexed access, subList, listIterator)
  2. Deque operations at both ends

Example: A playlist where you need songs by position AND fast prev/next navigation.
In most other cases: use ArrayDeque for queue/stack, ArrayList for list.
```

**Q4: How much memory does each element take in LinkedList vs ArrayList?**
```
ArrayList (storing String):
  - Reference in Object[]: 8 bytes (64-bit JVM)
  - + String object itself: 24+ bytes header
  Total overhead per element: ~8 bytes (just the reference in array)

LinkedList (storing String):
  - Node object: 16 bytes (object header)
  - Node.prev reference: 8 bytes
  - Node.next reference: 8 bytes
  - Node.item reference: 8 bytes
  = 40 bytes overhead PER NODE (before the actual element!)
  
For 1 million elements: LinkedList uses ~32MB MORE than ArrayList
```

---

*Previous: [ArrayList Deep Dive ←](./01_List_ArrayList.md) | Next: [Stack, Vector & Legacy Classes →](./03_Stack_Vector_Legacy.md)*
