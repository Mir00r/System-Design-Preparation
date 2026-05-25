# 🏆 Java PriorityQueue: The Heap-Powered Scheduler 🚀

---

## 🎯 Chapter 9: PriorityQueue — When Order Matters by Priority

> **"PriorityQueue is the hospital triage system of Java — it doesn't process patients in the order they arrived, but in order of urgency. Critical patients skip the queue, and the most critical is always at the front."**

---

## 📋 Table of Contents

1. [What is PriorityQueue?](#what-is-priorityqueue)
2. [Internal Architecture — Binary Heap](#internal-architecture)
3. [Min-Heap vs Max-Heap](#min-heap-vs-max-heap)
4. [All Methods Explained](#all-methods-explained)
5. [Custom Ordering with Comparator](#custom-ordering)
6. [Top-K Pattern (Most Common Interview Pattern)](#top-k-pattern)
7. [Merge K Sorted Lists Pattern](#merge-k-sorted)
8. [Real-World Analogies](#real-world-analogies)
9. [When to USE PriorityQueue](#when-to-use)
10. [When to AVOID PriorityQueue](#when-to-avoid)
11. [Time & Space Complexity](#time--space-complexity)
12. [Common Pitfalls](#common-pitfalls)
13. [Industry Applications](#industry-applications)
14. [Practice Problems](#practice-problems)
15. [Interview Tips](#interview-tips)

---

## 🤔 What is PriorityQueue?

### 📖 **Definition**

`PriorityQueue<E>` implements `Queue<E>` using a **binary min-heap** (by default). Elements are retrieved in **priority order** — the element with the **highest priority** (smallest value by natural ordering) is always at the head.

```
PriorityQueue characteristics:
✅ peek() / poll() always returns the MINIMUM element (natural order)
✅ O(log n) add/offer (heap sift-up)
✅ O(log n) poll/remove (heap sift-down)
✅ O(1) peek (constant — just return root)
✅ Unbounded — grows automatically
✅ Allows DUPLICATE elements
❌ NO null elements (compareTo fails)
❌ Iteration order is NOT priority order (heap is not sorted!)
❌ NOT thread-safe (use PriorityBlockingQueue for multi-threading)
❌ No random access by index

Introduced: Java 5 (2004)
```

---

## 🏗️ Internal Architecture — Binary Heap

### 🔧 **The Binary Heap**

A **Binary Heap** is a complete binary tree stored in an array where:
- **Min-Heap**: parent ≤ both children (Java's default PriorityQueue)
- **Max-Heap**: parent ≥ both children (use `Comparator.reverseOrder()`)

```
MIN-HEAP with elements: 1, 5, 3, 8, 7, 4, 6

TREE VIEW:              ARRAY VIEW:
        1               [_, 1, 5, 3, 8, 7, 4, 6]
       / \               index: 0  1  2  3  4  5  6  7
      5   3              (index 0 unused for simpler math)
     / \ / \
    8  7 4  6

PARENT-CHILD RELATIONSHIP:
  Parent of node i = i / 2 (integer division)
  Left child of i  = 2i
  Right child of i = 2i + 1

Example:
  Parent of node at index 5 (value=7) = 5/2 = index 2 (value=3)  ✅ 3 ≤ 7
  Left child of index 2 (value=3) = 2*2=4 (value=7)              ✅ 3 ≤ 7
  Right child of index 2 (value=3) = 2*2+1=5 (value=4)           ✅ 3 ≤ 4
```

### 🔧 **Java's Actual Implementation**

```java
// Simplified PriorityQueue source:
public class PriorityQueue<E> extends AbstractQueue<E>
        implements Serializable {
    
    private static final int DEFAULT_INITIAL_CAPACITY = 11;
    
    transient Object[] queue; // THE HEAP ARRAY (0-indexed!)
    private int size;
    private final Comparator<? super E> comparator;
    
    // SIFT UP after offer(): bubble new element UP until heap property restored
    private void siftUp(int k, E x) {
        if (comparator != null) siftUpUsingComparator(k, x);
        else siftUpComparable(k, x);
    }
    
    private void siftUpComparable(int k, E x) {
        Comparable<? super E> key = (Comparable<? super E>) x;
        while (k > 0) {
            int parent = (k - 1) >>> 1;  // parent = (k-1)/2
            Object e = queue[parent];
            if (key.compareTo((E) e) >= 0) break; // parent ≤ child: done!
            queue[k] = e;                // swap parent down
            k = parent;
        }
        queue[k] = key;
    }
    
    // SIFT DOWN after poll(): replace root with last element, bubble DOWN
    private void siftDown(int k, E x) { ... }
}
```

### 📊 **Heap Operations Visualized**

```
offer(2) into heap [1, 5, 3, 8, 7, 4, 6]:

Step 1: Add 2 at end of array:
        1
       / \
      5   3
     / \ / \
    8  7 4  6  ← 2 added here (index 7)
   /
  2

Step 2: SIFT UP — compare with parent:
  2 at index 7, parent = (7-1)/2 = 3, value = 8
  2 < 8 → SWAP! Move 2 up, 8 down.
  
  After swap:
        1
       / \
      5   3
     / \ / \
    2  7 4  6
   /
  8

  2 now at index 3, parent = (3-1)/2 = 1, value = 5
  2 < 5 → SWAP! Move 2 up, 5 down.
  
  After swap:
        1
       / \
      2   3
     / \ / \
    5  7 4  6
   /
  8
  
  2 now at index 1, parent = (1-1)/2 = 0, value = 1
  2 ≥ 1 → STOP! Heap property satisfied.

Final heap: [1, 2, 3, 5, 7, 4, 6, 8]
            minimum is still 1 ✅
```

---

## 📉 Min-Heap vs Max-Heap

```java
// ── MIN-HEAP (default) — smallest element at top ──
PriorityQueue<Integer> minHeap = new PriorityQueue<>();
minHeap.offer(5); minHeap.offer(1); minHeap.offer(3);
minHeap.peek(); // 1 (smallest)
minHeap.poll(); // 1, then 3, then 5 — ascending order

// ── MAX-HEAP — largest element at top ──
PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Comparator.reverseOrder());
maxHeap.offer(5); maxHeap.offer(1); maxHeap.offer(3);
maxHeap.peek(); // 5 (largest)
maxHeap.poll(); // 5, then 3, then 1 — descending order

// ── CUSTOM PRIORITY — by a specific field ──
PriorityQueue<Task> taskQueue = new PriorityQueue<>(
    Comparator.comparingInt(Task::getPriority) // lowest number = highest priority
);

// ── MULTIPLE CRITERIA ──
PriorityQueue<Job> jobs = new PriorityQueue<>(
    Comparator.comparingInt(Job::getPriority)          // primary: priority
              .thenComparing(Job::getSubmitTime)        // secondary: submit time
              .thenComparing(Comparator.comparing(Job::getName)) // tertiary: name
);
```

---

## 🛠️ All Methods Explained

```java
PriorityQueue<Integer> pq = new PriorityQueue<>();

// ── CONSTRUCTORS ──
PriorityQueue<Integer> pq1 = new PriorityQueue<>();                    // min-heap, cap=11
PriorityQueue<Integer> pq2 = new PriorityQueue<>(100);                 // min-heap, cap=100
PriorityQueue<Integer> pq3 = new PriorityQueue<>(Comparator.reverseOrder()); // max-heap
PriorityQueue<Integer> pq4 = new PriorityQueue<>(List.of(3,1,4,1,5));  // from collection

// ── ADDING ──
pq.offer(5);    // ✅ PREFERRED — returns false if capacity exceeded (always true for unbounded)
pq.add(3);      // same as offer() for unbounded queue
pq.offer(1);
pq.offer(4);
// Internal heap: [1, 3, 4, 5] (heap order, not sorted!)

// ── PEEKING (O(1)) ──
int top = pq.peek();     // 1 — smallest, NO removal, null if empty
int top2 = pq.element(); // 1 — throws NoSuchElementException if empty

// ── POLLING (O(log n)) ──
int removed = pq.poll();    // 1 — smallest, REMOVES, null if empty
int removed2 = pq.remove(); // next smallest, throws if empty

// After: pq = [3, 4, 5]

// ── SIZE ──
pq.size();      // 3
pq.isEmpty();   // false

// ── CONTAINS ── O(n) linear scan!
pq.contains(4); // true — O(n)!

// ── REMOVE BY VALUE ── O(n) to find + O(log n) to fix heap
pq.remove(4);   // removes first occurrence of 4 — O(n)!

// ── CLEAR ──
pq.clear();

// ── ITERATION — NOT in priority order! ──
PriorityQueue<Integer> pq5 = new PriorityQueue<>(List.of(5,1,3,4,2));
for (int x : pq5) {
    System.out.print(x + " "); // Could be: 1 2 3 4 5 OR 1 4 3 5 2 OR any heap order!
}
// NEVER iterate PQ expecting sorted output! Use poll() to get sorted.

// ── GET ALL IN PRIORITY ORDER ──
List<Integer> sorted = new ArrayList<>();
while (!pq5.isEmpty()) {
    sorted.add(pq5.poll()); // [1, 2, 3, 4, 5] — sorted ascending
}
// But this DRAINS the PQ! If you need to keep it, use a copy:
new PriorityQueue<>(pq5).stream().collect(Collectors.toList()); // drains copy

// ── TO ARRAY ──
Object[] arr = pq.toArray(); // array of heap elements, NOT sorted!
Integer[] typedArr = pq.toArray(new Integer[0]);
```

---

## 🎨 Custom Ordering with Comparator

```java
// ── TASK SCHEDULER ──
record Task(String name, int priority, LocalDateTime submitted) {}

PriorityQueue<Task> scheduler = new PriorityQueue<>(
    Comparator.comparingInt(Task::priority)             // lower number = higher priority
              .thenComparing(Task::submitted)           // FIFO within same priority
);

scheduler.offer(new Task("low-priority", 10, now()));
scheduler.offer(new Task("critical", 1, now()));
scheduler.offer(new Task("medium", 5, now()));

Task next = scheduler.poll(); // "critical" (priority=1) first!

// ── STRING BY LENGTH THEN ALPHABETICAL ──
PriorityQueue<String> pq = new PriorityQueue<>(
    Comparator.comparingInt(String::length)
              .thenComparing(Comparator.naturalOrder())
);
pq.offer("banana"); pq.offer("fig"); pq.offer("apple"); pq.offer("kiwi");
// Order of poll(): fig (3), kiwi (4), apple (5), banana (6)

// ── PAIR/TUPLE with custom key ──
// Common in Dijkstra: [distance, node]
PriorityQueue<int[]> dijkstra = new PriorityQueue<>(
    Comparator.comparingInt(a -> a[0]) // compare by distance (index 0)
);
dijkstra.offer(new int[]{5, 3});   // distance=5, node=3
dijkstra.offer(new int[]{1, 7});   // distance=1, node=7
dijkstra.offer(new int[]{3, 2});   // distance=3, node=2
dijkstra.poll(); // [1,7] — shortest distance first

// ── MAP.ENTRY sorted by value ──
PriorityQueue<Map.Entry<String, Integer>> topK = new PriorityQueue<>(
    Map.Entry.comparingByValue() // min-heap by value → for bottom-K
);
// For TOP-K: use reverseOrder or negate the comparator
```

---

## 🏆 Top-K Pattern (Most Common Interview Pattern)

### 🔧 **Find K Largest Elements**

```java
// PROBLEM: Find K largest elements from a stream of n elements
// NAIVE: Sort all → O(n log n), then take last K
// OPTIMAL: Use a MIN-HEAP of size K → O(n log K)

public int[] topKLargest(int[] nums, int k) {
    // Min-heap of size K
    PriorityQueue<Integer> minHeap = new PriorityQueue<>();
    
    for (int num : nums) {
        minHeap.offer(num);
        if (minHeap.size() > k) {
            minHeap.poll(); // remove the SMALLEST (not in top K)
        }
        // Heap always contains the K LARGEST seen so far
        // Smallest of those K is at the top (minHeap.peek())
    }
    
    // Extract K largest in any order
    return minHeap.stream().mapToInt(Integer::intValue).toArray();
}

// WHY MIN-HEAP for top-K largest?
// When new element > current minimum (heap top):
//   Remove the minimum, add the new element → heap stays as K largest
// When new element ≤ current minimum:
//   Skip — it's not in top K
```

### 🔧 **K-th Largest Element**

```java
// LeetCode 215 — Kth Largest Element in an Array
public int findKthLargest(int[] nums, int k) {
    PriorityQueue<Integer> minHeap = new PriorityQueue<>(k);
    for (int num : nums) {
        minHeap.offer(num);
        if (minHeap.size() > k) minHeap.poll();
    }
    return minHeap.peek(); // The k-th largest is the minimum of top-K heap
}

// ALTERNATIVE: Max-heap (simpler to understand, but O(n log n)):
public int findKthLargestMaxHeap(int[] nums, int k) {
    PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Comparator.reverseOrder());
    for (int num : nums) maxHeap.offer(num);
    for (int i = 0; i < k - 1; i++) maxHeap.poll(); // remove k-1 largest
    return maxHeap.poll(); // k-th largest
}
```

### 🔧 **Top K Frequent Elements**

```java
// LeetCode 347
public int[] topKFrequent(int[] nums, int k) {
    // Step 1: Count frequencies
    Map<Integer, Integer> freq = new HashMap<>();
    for (int n : nums) freq.merge(n, 1, Integer::sum);
    
    // Step 2: Min-heap of size K, sorted by FREQUENCY
    PriorityQueue<Map.Entry<Integer, Integer>> minHeap = new PriorityQueue<>(
        Map.Entry.comparingByValue() // min-heap by frequency
    );
    
    for (Map.Entry<Integer, Integer> entry : freq.entrySet()) {
        minHeap.offer(entry);
        if (minHeap.size() > k) minHeap.poll(); // remove least frequent
    }
    
    // Step 3: Extract results
    return minHeap.stream()
        .mapToInt(Map.Entry::getKey)
        .toArray();
}
```

---

## 🔗 Merge K Sorted Lists Pattern

```java
// LeetCode 23 — Merge K Sorted Lists
// Classic PriorityQueue pattern!

public ListNode mergeKLists(ListNode[] lists) {
    PriorityQueue<ListNode> minHeap = new PriorityQueue<>(
        Comparator.comparingInt(node -> node.val)
    );
    
    // Initialize heap with heads of all lists
    for (ListNode node : lists) {
        if (node != null) minHeap.offer(node);
    }
    
    ListNode dummy = new ListNode(0);
    ListNode current = dummy;
    
    while (!minHeap.isEmpty()) {
        ListNode smallest = minHeap.poll(); // Always get smallest
        current.next = smallest;
        current = current.next;
        
        if (smallest.next != null) {
            minHeap.offer(smallest.next); // Add next from same list
        }
    }
    
    return dummy.next;
    // Time: O(n log k) where n = total nodes, k = number of lists
    // Space: O(k) — heap holds at most k elements at a time
}
```

---

## 🌍 Real-World Analogies

### 🏥 **Hospital Triage System**

```
PriorityQueue is like a hospital emergency room triage:

WAITING ROOM (the heap):
  Patient with severity 1 (critical)   ← ALWAYS next to be treated
  Patient with severity 3 (urgent)
  Patient with severity 5 (moderate)
  Patient with severity 8 (minor)

When doctor calls next patient → poll() → always gets severity=1 (critical)
When new patient arrives → offer() → placed in proper position by severity
New severe patient (severity 2) → jumps ahead of severity=3,5,8 patients

✅ Most critical always treated first (not FIFO!)
✅ New patients inserted in O(log n) time
✅ Always O(1) to know who's next (peek the minimum)
```

---

## ✅ When to USE PriorityQueue

```
✅ USE PriorityQueue WHEN:
─────────────────────────────────────────────────────────────────────
✅ TOP-K problems: "Find K largest/smallest elements"
   Pattern: Min-heap of size K for top-K largest
   Time: O(n log K) vs O(n log n) for full sort

✅ SCHEDULING: Task/job by priority, not submission time
   Examples: OS process scheduler, Dijkstra's algorithm, A* search

✅ STREAM PROCESSING: Process the min/max from a dynamic set
   Examples: Median maintenance, real-time leaderboards

✅ GRAPH ALGORITHMS requiring minimum-cost exploration:
   Dijkstra's shortest path, Prim's MST, A* path finding

✅ MERGING K SORTED SEQUENCES:
   Efficiently get next smallest from K sorted sources

✅ EVENT SIMULATION: Process events in time order
   Examples: Network packet scheduling, job simulators

REMEMBER:
  "I always want the min/max and it changes dynamically" → PriorityQueue
  "I want sorted once and done" → Collections.sort() on ArrayList
```

---

## ❌ When to AVOID PriorityQueue

```
❌ AVOID PriorityQueue WHEN:
─────────────────────────────────────────────────────────────────────
❌ YOU NEED SORTED ITERATION without draining
   PriorityQueue iteration is NOT in priority order!
   for(E e : pq) { ... } → heap traversal, not sorted order
   Use: TreeSet or sort an ArrayList

❌ YOU NEED RANDOM ACCESS
   pq.get(i) doesn't exist — no index access
   Use: sorted ArrayList

❌ FREQUENT ARBITRARY REMOVALS
   pq.remove(specificElement) is O(n) — linear scan!
   Use: TreeSet (O(log n) remove by value)

❌ MULTI-THREADED CONCURRENT ACCESS
   PriorityQueue is not thread-safe
   Use: PriorityBlockingQueue

❌ YOU NEED CUSTOM UPDATE (change priority of existing element)
   PriorityQueue has NO decreaseKey() operation!
   Workaround: lazy deletion (mark as deleted, add updated copy)
   Use: TreeMap or indexed heap for true O(log n) priority update
```

---

## ⏱️ Time & Space Complexity

```
OPERATION          │ PriorityQueue │ TreeSet     │ Notes
───────────────────┼───────────────┼─────────────┼──────────────────────────────
offer(E) / add(E)  │ O(log n)      │ O(log n)    │ Sift up (heap) vs RB rotation
poll() / remove()  │ O(log n)      │ O(log n)    │ Sift down (heap) vs RB rotation
peek() / min()     │ O(1)          │ O(log n)    │ PQ root = O(1)! TreeSet.first() = O(log n)
contains(Object)   │ O(n)          │ O(log n)    │ PQ must scan heap array linearly
remove(Object)     │ O(n)          │ O(log n)    │ PQ scan + sift; TreeSet BST search
size() / isEmpty() │ O(1)          │ O(1)        │
heapify (construct)│ O(n)          │ O(n log n)  │ PQ(collection) = O(n) Floyd's algorithm!
───────────────────┴───────────────┴─────────────┴──────────────────────────────

KEY: PriorityQueue.peek() = O(1) is its superpower over TreeSet.first() = O(log n)
     But PriorityQueue.contains/remove(Object) = O(n) is its weakness vs TreeSet's O(log n)

SPACE: O(n). Heap uses a flat array — very cache-friendly compared to TreeSet's tree.

NOTE: new PriorityQueue<>(collection) uses Floyd's heap construction = O(n)!
      Faster than inserting n elements one by one (which would be O(n log n))
```

---

## ⚠️ Common Pitfalls

### 💣 **Pitfall 1: Iteration is NOT in Priority Order**

```java
PriorityQueue<Integer> pq = new PriorityQueue<>(List.of(5,1,3,4,2));

// ❌ WRONG — don't expect sorted iteration:
for (int x : pq) {
    System.out.print(x + " "); // Prints: 1 2 3 4 5 OR 1 4 3 5 2 OR...
    // The iteration order depends on the internal heap structure!
    // NOT guaranteed to be sorted!
}

// ✅ CORRECT — drain to get sorted order:
while (!pq.isEmpty()) {
    System.out.print(pq.poll() + " "); // 1 2 3 4 5 — sorted!
}

// ✅ CORRECT — use sorted list if you need both keep and iterate:
List<Integer> sorted = new ArrayList<>(pq);
Collections.sort(sorted); // O(n log n)
```

### 💣 **Pitfall 2: No decreaseKey/updatePriority**

```java
// ❌ PROBLEM: Want to update an element's priority
PriorityQueue<Task> pq = new PriorityQueue<>(Comparator.comparingInt(Task::priority));
Task myTask = new Task("cleanup", 10);
pq.offer(myTask);

myTask.setPriority(1); // ← MUTATED the task's priority!
// The heap now has a CORRUPTED invariant — myTask is in the wrong position!

// ✅ WORKAROUND: Remove and re-add (O(n) though!)
pq.remove(myTask); // O(n)
myTask.setPriority(1);
pq.offer(myTask);  // O(log n)

// ✅ WORKAROUND 2: Lazy deletion (better performance)
// Mark old task as "deleted", add new task with new priority
Set<Task> deleted = new HashSet<>();
deleted.add(myTask); // mark as deleted
myTask2 = new Task("cleanup", 1); // new task with new priority
pq.offer(myTask2);

// When polling:
Task next;
do {
    next = pq.poll();
} while (deleted.contains(next)); // skip deleted tasks
```

### 💣 **Pitfall 3: Null Elements**

```java
PriorityQueue<String> pq = new PriorityQueue<>();
pq.offer(null); // ❌ NullPointerException!
// PriorityQueue uses compareTo() on every element → fails on null
```

### 💣 **Pitfall 4: Objects Without Comparable/Comparator**

```java
class User { String name; }

PriorityQueue<User> pq = new PriorityQueue<>(); // no Comparator!
pq.offer(new User("Alice")); // OK (only one element, no comparison needed)
pq.offer(new User("Bob"));   // ❌ ClassCastException! User doesn't implement Comparable!

// ✅ Fix: provide a Comparator
PriorityQueue<User> pq2 = new PriorityQueue<>(Comparator.comparing(u -> u.name));
pq2.offer(new User("Alice"));
pq2.offer(new User("Bob")); // OK!
```

---

## 🏢 Industry Applications

```
ALGORITHM/SYSTEM      │ HOW PriorityQueue IS USED
──────────────────────────────────────────────────────────────────────────
Dijkstra's Algorithm  │ Always expand minimum-distance unvisited node
A* Pathfinding        │ Priority = f(n) = g(n) + h(n) cost estimate
Huffman Coding        │ Merge two lowest-frequency trees repeatedly
CPU Scheduling (OS)   │ Process with highest priority runs first
Netflix Encoding      │ Job queue sorted by encoding priority/deadline
Kafka Consumer Groups │ Process highest-offset messages for ordering
Spring Batch          │ Job prioritization in batch processing
Timer/ScheduledExecutor│ Schedule tasks by next-execution time (min = soonest)
Event-Driven Simulation│ Process events in chronological order
Network Router        │ Packet scheduling by QoS (Quality of Service) priority
```

### 💻 **Dijkstra's Shortest Path**

```java
public int[] dijkstra(int[][] graph, int source) {
    int n = graph.length;
    int[] dist = new int[n];
    Arrays.fill(dist, Integer.MAX_VALUE);
    dist[source] = 0;
    
    // [distance, node] — min-heap by distance
    PriorityQueue<int[]> pq = new PriorityQueue<>(Comparator.comparingInt(a -> a[0]));
    pq.offer(new int[]{0, source});
    
    while (!pq.isEmpty()) {
        int[] curr = pq.poll();
        int d = curr[0], u = curr[1];
        
        if (d > dist[u]) continue; // outdated entry (lazy deletion)
        
        for (int v = 0; v < n; v++) {
            if (graph[u][v] > 0) {  // edge exists
                int newDist = dist[u] + graph[u][v];
                if (newDist < dist[v]) {
                    dist[v] = newDist;
                    pq.offer(new int[]{newDist, v}); // add updated distance
                }
            }
        }
    }
    return dist;
}
```

### 💻 **Spring Boot: Task Execution with Priority**

```java
@Service
public class PriorityTaskExecutor {
    
    record PriorityTask(int priority, Runnable task, String name) 
        implements Comparable<PriorityTask> {
        
        @Override
        public int compareTo(PriorityTask other) {
            return Integer.compare(this.priority, other.priority); // lower = higher priority
        }
    }
    
    private final PriorityBlockingQueue<PriorityTask> taskQueue = 
        new PriorityBlockingQueue<>();
    
    @PostConstruct
    public void startWorker() {
        Thread worker = new Thread(() -> {
            while (!Thread.interrupted()) {
                try {
                    PriorityTask task = taskQueue.take(); // blocks until available
                    log.info("Executing {} (priority={})", task.name(), task.priority());
                    task.task().run();
                } catch (InterruptedException e) {
                    Thread.currentThread().interrupt();
                }
            }
        });
        worker.setDaemon(true);
        worker.start();
    }
    
    public void submit(int priority, String name, Runnable task) {
        taskQueue.offer(new PriorityTask(priority, task, name));
    }
}

// Usage:
executor.submit(1, "critical-alert", () -> sendEmergencyEmail());
executor.submit(10, "report-generation", () -> generateWeeklyReport());
executor.submit(3, "cache-refresh", () -> refreshCache());
// Order: critical-alert → cache-refresh → report-generation
```

---

## 🎯 Practice Problems

| # | Problem | Difficulty | PQ Pattern |
|---|---------|-----------|------------|
| 1 | [Kth Largest Element in an Array](https://leetcode.com/problems/kth-largest-element-in-an-array/) | 🟡 Medium | Min-heap of size K |
| 2 | [Top K Frequent Elements](https://leetcode.com/problems/top-k-frequent-elements/) | 🟡 Medium | Min-heap by frequency |
| 3 | [Merge K Sorted Lists](https://leetcode.com/problems/merge-k-sorted-lists/) | 🔴 Hard | Min-heap of list heads |
| 4 | [Find Median from Data Stream](https://leetcode.com/problems/find-median-from-data-stream/) | 🔴 Hard | Two heaps (max+min) |
| 5 | [K Closest Points to Origin](https://leetcode.com/problems/k-closest-points-to-origin/) | 🟡 Medium | Max-heap of size K by distance |
| 6 | [Task Scheduler](https://leetcode.com/problems/task-scheduler/) | 🟡 Medium | Max-heap by frequency |
| 7 | [Reorganize String](https://leetcode.com/problems/reorganize-string/) | 🟡 Medium | Max-heap by char frequency |
| 8 | [Network Delay Time](https://leetcode.com/problems/network-delay-time/) | 🟡 Medium | Dijkstra with PQ |
| 9 | [Ugly Number II](https://leetcode.com/problems/ugly-number-ii/) | 🟡 Medium | Min-heap for next ugly number |
| 10| [Smallest Range Covering Elements from K Lists](https://leetcode.com/problems/smallest-range-covering-elements-from-k-lists/) | 🔴 Hard | PQ for K-pointer merge |

---

## 💡 Interview Tips

**Q1: How does PriorityQueue's peek() achieve O(1)?**
```
A binary heap stores the minimum (for min-heap) at array index 0.
peek() simply returns queue[0] — a single array lookup, O(1).
No traversal needed because the heap invariant GUARANTEES the minimum is at root.

This is PriorityQueue's biggest advantage over TreeSet:
  TreeSet.first() = O(log n) (navigate to leftmost node)
  PriorityQueue.peek() = O(1) (just array[0])
```

**Q2: What is the "Two Heaps" trick for finding the median?**
```java
// LeetCode 295 — Find Median from Data Stream
// KEY INSIGHT: 
//   Lower half → MAX-HEAP (max of lower half = median candidate)
//   Upper half → MIN-HEAP (min of upper half = median candidate)

class MedianFinder {
    PriorityQueue<Integer> lower = new PriorityQueue<>(Comparator.reverseOrder()); // max-heap
    PriorityQueue<Integer> upper = new PriorityQueue<>(); // min-heap
    
    public void addNum(int num) {
        lower.offer(num);
        upper.offer(lower.poll()); // balance: push max of lower to upper
        if (upper.size() > lower.size()) {
            lower.offer(upper.poll()); // keep lower >= upper in size
        }
    }
    
    public double findMedian() {
        if (lower.size() > upper.size()) return lower.peek();
        return (lower.peek() + upper.peek()) / 2.0;
    }
}
```

**Q3: Why use a min-heap of size K to find the K largest elements?**
```
Intuition: The "K largest" elements form a club.
  The min-heap lets you efficiently answer: "Is this new element
  good enough to join the club? Is it larger than the weakest member?"

Process:
  1. Min-heap top = weakest member of top-K club (smallest of K largest)
  2. New element arrives:
     - If it beats the weakest member → kick out the weakest, add new
     - If it doesn't → it's not in top K, skip it
  
Time: O(n log K) vs O(n log n) for sorting everything
  For n=10M and K=10: 10M×log(10) = 10M×3.3 operations vs 10M×23 operations
```

---

*Previous: [Queue, Deque & ArrayDeque ←](./08_Queue_Deque_ArrayDeque.md) | Next: [Concurrent Collections →](./10_Concurrent_Collections.md)*
