# 🔄 Java Queue, Deque & ArrayDeque: The Complete Guide 🚀

---

## 🎯 Chapter 8: Queue Family — FIFO, LIFO, and Everything In Between

> **"ArrayDeque is the Goldilocks of Java collections — not too big (like LinkedList's memory overhead), not too small (like Stack's limited API), but just right for almost every queue and stack need."**

---

## 📋 Table of Contents

1. [The Queue Interface Contract](#queue-interface)
2. [The Deque Interface — Double-Ended Power](#deque-interface)
3. [ArrayDeque — The Implementation Champion](#arraydeque)
4. [ArrayDeque Internal Architecture](#arraydeque-architecture)
5. [Queue as FIFO — All Patterns](#queue-fifo-patterns)
6. [Deque as Stack (LIFO) — All Patterns](#deque-stack-patterns)
7. [ArrayDeque vs LinkedList vs Stack](#comparison)
8. [When to USE ArrayDeque](#when-to-use)
9. [When to AVOID ArrayDeque](#when-to-avoid)
10. [BlockingQueue for Producers/Consumers](#blockingqueue)
11. [Time & Space Complexity](#time--space-complexity)
12. [Common Pitfalls](#common-pitfalls)
13. [Industry Applications](#industry-applications)
14. [Practice Problems](#practice-problems)
15. [Interview Tips](#interview-tips)

---

## 🔌 The Queue Interface Contract

### 📖 **What Queue Guarantees**

`Queue<E>` extends `Collection<E>` and models a **FIFO** (First-In, First-Out) data structure.

```java
// java.util.Queue<E>
public interface Queue<E> extends Collection<E> {
    // TWO FLAVORS for each operation:
    //   Throws exception  |  Returns special value
    
    // ADD to tail:
    boolean add(E e);       // throws IllegalStateException if capacity exceeded
    boolean offer(E e);     // returns false if capacity exceeded (PREFERRED)
    
    // REMOVE from head:
    E remove();             // throws NoSuchElementException if empty
    E poll();               // returns null if empty (PREFERRED)
    
    // PEEK at head (no remove):
    E element();            // throws NoSuchElementException if empty
    E peek();               // returns null if empty (PREFERRED)
}
```

```
QUEUE (FIFO) — "First In, First Out":
                                 
  ENQUEUE →→→→→→→→→→→→→→→→→→→→→→→→ DEQUEUE
  
  ADD                                REMOVE
  offer("E") → [A][B][C][D][E]  →  poll() returns "A"
  
  [tail end]                         [head end]
  
Real world: Supermarket checkout line, print queue, HTTP request queue
```

---

## 🔄 The Deque Interface — Double-Ended Power

### 📖 **Deque = Double-Ended Queue**

`Deque<E>` extends `Queue<E>` and allows adding/removing from BOTH ends.

```java
// java.util.Deque<E> — extends Queue<E>
public interface Deque<E> extends Queue<E> {
    // FRONT operations:
    void addFirst(E e);     E removeFirst();    E peekFirst();
    boolean offerFirst(E e); E pollFirst();     E getFirst();
    
    // BACK operations:
    void addLast(E e);      E removeLast();     E peekLast();
    boolean offerLast(E e); E pollLast();       E getLast();
    
    // STACK operations (maps to front operations):
    void push(E e);         // = addFirst
    E pop();                // = removeFirst
    E peek();               // = peekFirst
    
    // QUEUE operations (add to back, remove from front):
    boolean offer(E e);     // = offerLast
    E poll();               // = pollFirst
    
    // ITERATOR (both directions):
    Iterator<E> iterator();           // front to back
    Iterator<E> descendingIterator(); // back to front
    
    // SIZE/CONTAINS (inherited from Collection):
    int size(); boolean isEmpty(); boolean contains(Object o);
}
```

```
DEQUE (Double-Ended):

  addFirst ←←←←←←←←←←←←←←→→→→→→→→→ addLast
  push/pop   [C][B][A][D][E]   offer/poll
  peekFirst←                       →peekLast
  
  Can function as:
  ✅ Stack (LIFO): push/pop at front
  ✅ Queue (FIFO): offer at back, poll from front
  ✅ Deque: add/remove at either end
```

---

## ⚡ ArrayDeque — The Implementation Champion

### 📖 **What is ArrayDeque?**

`ArrayDeque<E>` implements `Deque<E>` using a **resizable circular array** (ring buffer). It is the **recommended replacement** for both `Stack` and `LinkedList` for queue/stack operations.

```
ArrayDeque characteristics:
✅ O(1) amortized add/remove at BOTH ends
✅ No null elements allowed (null = "empty" signal)
✅ Resizable circular array (no wasted capacity like ArrayList)
✅ No synchronization overhead (unlike Stack/Vector)
✅ NO null elements (enforced — null has special meaning in Queue operations)
❌ NOT thread-safe (use LinkedBlockingDeque for multi-threading)
❌ NOT good for concurrent use cases

Introduced: Java 6 (2006)
Beats: Stack, LinkedList for stack/queue use cases
```

---

## 🏗️ ArrayDeque Internal Architecture

### 🔧 **The Circular Array (Ring Buffer)**

```java
// Simplified ArrayDeque internals:
public class ArrayDeque<E> extends AbstractCollection<E>
        implements Deque<E>, Cloneable, Serializable {
    
    transient Object[] elements;  // THE ARRAY
    transient int head;           // index of the head element (front)
    transient int tail;           // index PAST the tail element (next slot)
    
    // Default initial capacity = 16 (always power of 2!)
}
```

```
CIRCULAR ARRAY VISUALIZATION:
                   head=2  tail=6
                     ↓       ↓
Array: [null][null][ A ][ B ][ C ][ D ][null][null]
Index:   0     1     2    3    4    5     6     7

elements = {null, null, "A", "B", "C", "D", null, null}
head = 2  (index of first element)
tail = 6  (index where NEXT element would be added)
Size = tail - head = 4

addFirst("X") — add at front:
  head = (head - 1) & (length - 1) = (2-1) & 7 = 1
  elements[1] = "X"
  New layout: [null][ X ][ A ][ B ][ C ][ D ][null][null]
              head=1↑

addLast("Z") — add at back:
  elements[tail=6] = "Z"
  tail = (tail + 1) & (length - 1) = (6+1) & 7 = 7
  New layout: [null][ X ][ A ][ B ][ C ][ D ][ Z ][null]
                                                    tail=7↑

WRAP-AROUND (the "circular" part):
  When tail reaches end of array and wraps to beginning:
  [null][ X ][ A ][ B ][ C ][ D ][ Z ][ Y ] ← tail would wrap to index 0
  [W   ][ X ][ A ][ B ][ C ][ D ][ Z ][ Y ] ← addLast("W") wraps tail to 1
  head=2, tail=1 (tail < head = wrapped!)
  
GROWTH: When head == tail (array is full):
  double capacity, copy elements linearly (unwrap the circle)
  (head - 1) & (length - 1) can't be head → if it equals head → FULL
```

---

## 📥 Queue as FIFO — All Patterns

### 🔧 **FIFO Queue with ArrayDeque**

```java
// ArrayDeque as FIFO Queue:
Deque<String> queue = new ArrayDeque<>();

// ENQUEUE (add to back)
queue.offer("Alice");     // [Alice]
queue.offer("Bob");       // [Alice, Bob]
queue.offer("Charlie");   // [Alice, Bob, Charlie]
queue.offerLast("Dave");  // same as offer — [Alice, Bob, Charlie, Dave]

// PEEK (view front without removing)
String front = queue.peek();       // "Alice" — O(1)
String front2 = queue.peekFirst(); // "Alice" — same

// DEQUEUE (remove from front)
String served = queue.poll();       // "Alice" — O(1), null if empty
String served2 = queue.pollFirst(); // "Bob" — same
// Queue now: [Charlie, Dave]

// Size and emptiness
queue.size();     // 2
queue.isEmpty();  // false
queue.contains("Charlie"); // true — O(n)

// NEVER USE WITH NULL:
queue.offer(null); // ❌ NullPointerException! (ArrayDeque rejects null)

// ITERATION (front to back):
for (String s : queue) { System.out.println(s); } // Charlie, Dave
queue.forEach(System.out::println);
```

### 🔧 **BFS Algorithm Pattern**

```java
// Breadth-First Search — canonical Queue usage:
public List<List<Integer>> levelOrder(TreeNode root) {
    List<List<Integer>> result = new ArrayList<>();
    if (root == null) return result;
    
    Deque<TreeNode> queue = new ArrayDeque<>();  // NOT LinkedList!
    queue.offer(root);
    
    while (!queue.isEmpty()) {
        int levelSize = queue.size();
        List<Integer> level = new ArrayList<>();
        
        for (int i = 0; i < levelSize; i++) {
            TreeNode node = queue.poll();     // O(1)
            level.add(node.val);
            if (node.left != null)  queue.offer(node.left);  // O(1)
            if (node.right != null) queue.offer(node.right); // O(1)
        }
        result.add(level);
    }
    return result;
}
```

---

## 📤 Deque as Stack (LIFO) — All Patterns

### 🔧 **LIFO Stack with ArrayDeque**

```java
// ArrayDeque as LIFO Stack:
Deque<String> stack = new ArrayDeque<>();

// PUSH (add to front)
stack.push("floor 1");     // [floor 1]
stack.push("floor 2");     // [floor 2, floor 1]
stack.push("floor 3");     // [floor 3, floor 2, floor 1]
// addFirst("floor 4") is identical to push():
stack.addFirst("floor 4"); // [floor 4, floor 3, floor 2, floor 1]

// PEEK (view top without removing)
String top = stack.peek();       // "floor 4" — O(1)
String top2 = stack.peekFirst(); // "floor 4" — same

// POP (remove from top)
String popped = stack.pop();       // "floor 4" — O(1), throws if empty
String popped2 = stack.pollFirst(); // "floor 3" — same but returns null if empty

// Stack status
stack.size();     // 2
stack.isEmpty();  // false

// ITERATION (top to bottom):
for (String s : stack) { System.out.println(s); } // floor 2, floor 1
// Note: regular iterator goes front-to-back = top-to-bottom for stacks

// REVERSE ITERATION (bottom to top):
stack.descendingIterator().forEachRemaining(System.out::println);
```

### 🔧 **DFS Algorithm Pattern**

```java
// Iterative DFS — canonical Stack usage:
public List<Integer> inorderTraversal(TreeNode root) {
    List<Integer> result = new ArrayList<>();
    Deque<TreeNode> stack = new ArrayDeque<>();  // NOT Stack class!
    TreeNode current = root;
    
    while (current != null || !stack.isEmpty()) {
        while (current != null) {
            stack.push(current);      // O(1)
            current = current.left;
        }
        current = stack.pop();        // O(1)
        result.add(current.val);
        current = current.right;
    }
    return result;
}
```

---

## 📊 ArrayDeque vs LinkedList vs Stack

```
OPERATION      │ ArrayDeque    │ LinkedList    │ Stack (legacy)│ Notes
───────────────┼───────────────┼───────────────┼───────────────┼────────────────────────
push/addFirst  │ O(1)★         │ O(1)          │ O(1) sync     │ ★amortized
pop/pollFirst  │ O(1)          │ O(1)          │ O(1) sync     │
offer/addLast  │ O(1)★         │ O(1)          │ N/A           │
poll/pollLast  │ O(1)          │ O(1)          │ N/A           │
peek           │ O(1)          │ O(1)          │ O(1) sync     │
contains       │ O(n)          │ O(n)          │ O(n) sync     │
iteration      │ O(n) FAST     │ O(n) SLOW     │ O(n) sync     │ Cache locality!
Memory/element │ ~8 bytes ref  │ ~40 bytes Node│ ~8 bytes sync │ LinkedList = 5x more
Null allowed   │ ❌ No          │ ✅ Yes         │ ✅ Yes         │
Thread-safe    │ ❌ No          │ ❌ No          │ ✅ Yes (slow)  │
───────────────┴───────────────┴───────────────┴───────────────┴────────────────────────

Performance benchmark (1M operations, single thread):
  push+pop:  ArrayDeque ~95ms,  LinkedList ~190ms,  Stack ~310ms
  Reason: ArrayDeque = contiguous array = CPU cache efficient
          LinkedList = scattered heap objects = cache misses on every step
```

---

## ✅ When to USE ArrayDeque

```
✅ USE ArrayDeque WHEN:
─────────────────────────────────────────────────────────────────────
✅ IMPLEMENTING A STACK (LIFO):
   Deque<T> stack = new ArrayDeque<>();
   stack.push(item);     stack.pop();     stack.peek();

✅ IMPLEMENTING A QUEUE (FIFO):
   Deque<T> queue = new ArrayDeque<>();
   queue.offer(item);    queue.poll();    queue.peek();

✅ BFS (breadth-first search) — add to back, remove from front

✅ DFS (depth-first search) — iterative version, use as stack

✅ SLIDING WINDOW ALGORITHMS — add/remove from both ends

✅ MONOTONIC DEQUE (sliding window max/min problems)

✅ UNDO/REDO — push actions, pop to undo

✅ BROWSER HISTORY — back/forward navigation

✅ FUNCTION CALL SIMULATION — recursive → iterative conversion

RULE OF THUMB:
  "I need a stack" → Deque<T> stack = new ArrayDeque<>();
  "I need a queue" → Deque<T> queue = new ArrayDeque<>();
  NEVER: new Stack<>() or new LinkedList<>() for these purposes
```

---

## ❌ When to AVOID ArrayDeque

```
❌ AVOID ArrayDeque WHEN:
─────────────────────────────────────────────────────────────────────
❌ YOU NEED NULL ELEMENTS
   ArrayDeque throws NullPointerException on null
   poll() returns null to signal "empty" — ambiguous if nulls allowed
   Use: LinkedList (allows null, but more memory)

❌ MULTI-THREADED CONCURRENT ACCESS
   ArrayDeque is not thread-safe
   Use: LinkedBlockingDeque (bounded/unbounded blocking)
        ArrayBlockingQueue (bounded, circular array, thread-safe)
        ConcurrentLinkedDeque (non-blocking, unbounded)

❌ WHEN YOU NEED A PRIORITY QUEUE
   ArrayDeque is FIFO/LIFO, not priority-based
   Use: PriorityQueue (next chapter!)

❌ WHEN YOU NEED BLOCKING OPERATIONS
   ArrayDeque.poll() on empty → returns null immediately
   For "wait until something is available" → use BlockingQueue
```

---

## 🚧 BlockingQueue for Producers/Consumers

### 📖 **The Producer-Consumer Pattern**

```java
// BlockingQueue — the thread-safe Queue family:
// Implementations: ArrayBlockingQueue, LinkedBlockingQueue, 
//                  LinkedBlockingDeque, SynchronousQueue

import java.util.concurrent.*;

// Producer thread:
BlockingQueue<Task> queue = new ArrayBlockingQueue<>(100); // capacity 100

// Producer:
new Thread(() -> {
    while (true) {
        Task task = generateTask();
        try {
            queue.put(task);    // BLOCKS if queue is full!
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            break;
        }
    }
}).start();

// Consumer:
new Thread(() -> {
    while (true) {
        try {
            Task task = queue.take();  // BLOCKS until something is available!
            process(task);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
            break;
        }
    }
}).start();

// BlockingQueue method summary:
// Blocking:  put(e) ← blocks if full,    take() ← blocks if empty
// Timed:     offer(e, timeout, unit),     poll(timeout, unit)
// Non-blocking: offer(e) ← false if full, poll() ← null if empty
//               add(e) ← throws if full,  remove() ← throws if empty
```

### 🔧 **Common BlockingQueue Types**

```
TYPE                    │ BOUNDED │ THREAD-SAFE │ NOTES
────────────────────────┼─────────┼─────────────┼──────────────────────────────────
ArrayBlockingQueue      │ ✅ Yes  │ ✅ Yes      │ Circular array, fair/unfair
LinkedBlockingQueue     │ Optional│ ✅ Yes      │ Default capacity = Integer.MAX_VALUE
LinkedBlockingDeque     │ Optional│ ✅ Yes      │ Double-ended blocking queue
SynchronousQueue        │ 0       │ ✅ Yes      │ Direct handoff, no buffering
PriorityBlockingQueue   │ ❌ No   │ ✅ Yes      │ Sorted + blocking (see Ch.9)
DelayQueue              │ ❌ No   │ ✅ Yes      │ Elements released after delay
ConcurrentLinkedQueue   │ ❌ No   │ ✅ Yes      │ Non-blocking, unbounded
ConcurrentLinkedDeque   │ ❌ No   │ ✅ Yes      │ Non-blocking, double-ended
```

---

## ⏱️ Time & Space Complexity

```
OPERATION          │ ArrayDeque   │ Notes
───────────────────┼──────────────┼──────────────────────────────────────
addFirst/push      │ O(1)★        │ Circular array, no shifting needed
addLast/offer      │ O(1)★        │ ★ amortized — O(n) on resize
removeFirst/pop    │ O(1)         │ Just update head pointer
removeLast/poll    │ O(1)         │ Just update tail pointer
peekFirst          │ O(1)         │ array[head]
peekLast           │ O(1)         │ array[(tail-1)&mask]
size()             │ O(1)         │ (tail - head) & mask
isEmpty()          │ O(1)         │
contains(Object)   │ O(n)         │ Linear scan
remove(Object)     │ O(n)         │ Scan + shift (mid-array removal)
iterator.next()    │ O(1)         │
clear()            │ O(n)         │ Must null all refs for GC
toArray()          │ O(n)         │
───────────────────┴──────────────┴──────────────────────────────────────
SPACE: O(n). Capacity is always a power of 2.
       May use up to 2x the elements (due to circular array geometry)
       Unlike ArrayList, growth doesn't leave a tail of null slots unused.
```

---

## ⚠️ Common Pitfalls

### 💣 **Pitfall 1: Using Stack Instead of ArrayDeque**

```java
// ❌ WRONG:
Stack<Integer> stack = new Stack<>();
stack.push(1);
stack.push(2);
int top = stack.pop();

// ✅ CORRECT:
Deque<Integer> stack = new ArrayDeque<>();
stack.push(1);
stack.push(2);
int top = stack.pop();

// Why matters: Stack.push/pop is synchronized, ~3x slower.
// Stack inherits bad List methods (get(0) etc.) that break stack semantics.
```

### 💣 **Pitfall 2: Null Elements in ArrayDeque**

```java
Deque<String> queue = new ArrayDeque<>();
queue.offer(null); // ❌ NullPointerException!

// Why? ArrayDeque uses null as a sentinel for empty slots in internal array
// and poll() returns null to signal "empty" — allowing nulls would be ambiguous.

// FIX: Use Optional or a sentinel non-null value:
queue.offer(Optional.ofNullable(possiblyNull)); // wrap in Optional

// OR: Use LinkedList (allows null but slower):
Deque<String> withNulls = new LinkedList<>();
withNulls.offer(null); // OK
```

### 💣 **Pitfall 3: Iterating While Modifying**

```java
Deque<String> deque = new ArrayDeque<>(List.of("a", "b", "c"));

// ❌ ConcurrentModificationException:
for (String s : deque) {
    if (s.equals("b")) deque.remove(s);
}

// ✅ Use iterator.remove():
Iterator<String> it = deque.iterator();
while (it.hasNext()) {
    if (it.next().equals("b")) it.remove();
}

// ✅ Use removeIf (Java 8+):
deque.removeIf(s -> s.equals("b"));
```

### 💣 **Pitfall 4: Wrong Method for Empty Queue**

```java
Deque<String> empty = new ArrayDeque<>();

// THROWS NoSuchElementException:
empty.remove();     // ❌ throws
empty.element();    // ❌ throws
empty.getFirst();   // ❌ throws
empty.removeFirst();// ❌ throws

// RETURNS NULL (safe):
empty.poll();       // ✅ null
empty.peek();       // ✅ null
empty.peekFirst();  // ✅ null
empty.pollFirst();  // ✅ null

// RULE: In production code, prefer poll/peek (return null)
//       over remove/element (throw exceptions).
//       Use add/remove-style only when empty = bug (assertions)
```

---

## 🏢 Industry Applications

```
ALGORITHM/USE CASE         │ COLLECTION           │ OPERATION
───────────────────────────┼──────────────────────┼────────────────────────────────
BFS (level order)          │ ArrayDeque as Queue   │ offer + poll
DFS (iterative)            │ ArrayDeque as Stack   │ push + pop
Undo/Redo system           │ 2 × ArrayDeque        │ push action, pop to undo
Browser history (back/fwd) │ ArrayDeque as Deque   │ push, pop (back), addLast (fwd)
Sliding window max/min     │ ArrayDeque (monotonic) │ addLast + pollFirst/pollLast
Expression evaluator       │ ArrayDeque as Stack   │ push operand, pop for operator
Parentheses matching       │ ArrayDeque as Stack   │ push open, pop to match close
Reverse a string           │ ArrayDeque as Stack   │ push all, pop all
Task scheduler (simple)    │ ArrayDeque as Queue   │ offer + poll (FIFO scheduling)
Print job queue            │ ArrayDeque as Queue   │ offer (add job) + poll (print)
```

### 💻 **Monotonic Deque — Sliding Window Maximum**

```java
// LeetCode 239 — Sliding Window Maximum
// Classic use of ArrayDeque's ability to remove from both ends!
public int[] maxSlidingWindow(int[] nums, int k) {
    int n = nums.length;
    int[] result = new int[n - k + 1];
    Deque<Integer> deque = new ArrayDeque<>(); // stores INDICES
    
    for (int i = 0; i < n; i++) {
        // Remove indices outside current window from front
        while (!deque.isEmpty() && deque.peekFirst() < i - k + 1) {
            deque.pollFirst();
        }
        
        // Remove indices of smaller elements from back (they'll never be max)
        while (!deque.isEmpty() && nums[deque.peekLast()] < nums[i]) {
            deque.pollLast();
        }
        
        deque.offerLast(i);
        
        // Record maximum (front of deque = index of max in window)
        if (i >= k - 1) {
            result[i - k + 1] = nums[deque.peekFirst()];
        }
    }
    return result;
    // Time: O(n), Space: O(k)
}
```

### 💻 **Spring Boot: Request Rate Limiter with Queue**

```java
@Service
public class RateLimiterService {
    private final int maxRequests;
    private final long windowMs;
    
    // Timestamp of each request in the current window
    private final Deque<Long> requestTimestamps = new ArrayDeque<>();
    
    public synchronized boolean allowRequest() {
        long now = System.currentTimeMillis();
        
        // Remove timestamps outside the window (from front — they're oldest)
        while (!requestTimestamps.isEmpty() 
               && requestTimestamps.peekFirst() < now - windowMs) {
            requestTimestamps.pollFirst();
        }
        
        if (requestTimestamps.size() < maxRequests) {
            requestTimestamps.addLast(now); // Add current timestamp to back
            return true;  // request allowed
        }
        return false;  // rate limit exceeded
    }
}
```

---

## 🎯 Practice Problems

| # | Problem | Difficulty | Queue/Deque Pattern |
|---|---------|-----------|---------------------|
| 1 | [Binary Tree Level Order Traversal](https://leetcode.com/problems/binary-tree-level-order-traversal/) | 🟡 Medium | BFS with Queue |
| 2 | [Sliding Window Maximum](https://leetcode.com/problems/sliding-window-maximum/) | 🔴 Hard | Monotonic Deque |
| 3 | [Number of Islands](https://leetcode.com/problems/number-of-islands/) | 🟡 Medium | BFS with Queue |
| 4 | [Implement Queue using Stacks](https://leetcode.com/problems/implement-queue-using-stacks/) | 🟢 Easy | Two stacks = queue |
| 5 | [Implement Stack using Queues](https://leetcode.com/problems/implement-stack-using-queues/) | 🟢 Easy | Two queues = stack |
| 6 | [Valid Parentheses](https://leetcode.com/problems/valid-parentheses/) | 🟢 Easy | Stack for matching |
| 7 | [Largest Rectangle in Histogram](https://leetcode.com/problems/largest-rectangle-in-histogram/) | 🔴 Hard | Monotonic Stack |
| 8 | [Design Circular Queue](https://leetcode.com/problems/design-circular-queue/) | 🟡 Medium | Circular array = ArrayDeque internals |
| 9 | [Shortest Path in Binary Matrix](https://leetcode.com/problems/shortest-path-in-binary-matrix/) | 🟡 Medium | BFS shortest path |

---

## 💡 Interview Tips

**Q1: Why use ArrayDeque instead of LinkedList for queue/stack?**
```
3 reasons:
1. PERFORMANCE: ArrayDeque stores elements in a contiguous array.
   CPU cache prefetch loads multiple elements at once.
   LinkedList has pointer-chased nodes at random heap addresses → cache misses.
   Benchmark: ArrayDeque 2-3x faster for push/pop in tight loops.

2. MEMORY: Each ArrayDeque slot = 1 reference (8 bytes).
   Each LinkedList Node = 40 bytes (header + prev + next + item).
   For 1M elements: ArrayDeque ~8MB, LinkedList ~40MB.

3. DESIGN: ArrayDeque implements Deque perfectly.
   LinkedList implements BOTH List and Deque → inherits confusing index methods
   that break the pure stack/queue abstraction.
```

**Q2: Why does ArrayDeque prohibit null elements?**
```
Queue's poll() and peek() methods return null to signal an EMPTY queue.
If null elements were allowed:
  queue.poll() returns null → is the queue empty, or did we just return null?
  Ambiguous! Forces callers to call size() or isEmpty() as well.

By prohibiting null, ArrayDeque ensures: poll() == null → queue is definitely empty.
This is consistent with ConcurrentLinkedQueue, LinkedBlockingQueue, etc.
(LinkedList allows null but that's legacy behavior before this design was settled.)
```

**Q3: How does ArrayDeque handle wrap-around in the circular array?**
```java
// Using bitwise AND instead of modulo for circular indexing:
// (index + 1) & (elements.length - 1)
// This works ONLY because length is always a power of 2.

// Example with length=8 (binary: 1000):
// length - 1 = 7 (binary: 0111)
// (7 + 1) & 7 = 8 & 7 = 0000 → wraps to index 0!
// (6 + 1) & 7 = 7 & 7 = 0111 → index 7 (no wrap)

// This is faster than: (index + 1) % length (division is expensive!)
```

**Q4: When should you use BlockingQueue instead of ArrayDeque?**
```
Use BlockingQueue when:
  1. Multiple threads produce and consume from the queue
  2. You want to BLOCK the consumer until data is available (take())
  3. You want to BLOCK the producer when the queue is full (put())
  4. You want a clean thread-safe handoff between threads

Use ArrayDeque when:
  - Single-threaded algorithms (BFS, DFS, undo/redo)
  - The queue is used within one thread only
  - You will manage thread safety yourself (synchronized wrapper)

Producer-consumer pattern → ALWAYS use BlockingQueue (ArrayBlockingQueue or LinkedBlockingQueue)
```

---

*Previous: [LinkedHashMap & TreeMap ←](./07_LinkedHashMap_TreeMap.md) | Next: [PriorityQueue Deep Dive →](./09_PriorityQueue.md)*
