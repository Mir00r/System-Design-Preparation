# 📝 Data Structures Quick Reference Cheat Sheet 🚀

---

## ⚡ Time Complexity Comparison

| Data Structure | Access | Search | Insert | Delete | Space |
|----------------|--------|--------|--------|--------|-------|
| **Array** | O(1) | O(n) | O(n) | O(n) | O(n) |
| **ArrayList** | O(1) | O(n) | O(n)* | O(n) | O(n) |
| **LinkedList** | O(n) | O(n) | O(1)** | O(1)** | O(n) |
| **Stack** | O(n) | O(n) | O(1) | O(1) | O(n) |
| **Queue** | O(n) | O(n) | O(1) | O(1) | O(n) |
| **Hash Table** | N/A | O(1)† | O(1)† | O(1)† | O(n) |
| **Binary Tree** | O(n) | O(n) | O(n) | O(n) | O(n) |
| **BST** | O(log n)‡ | O(log n)‡ | O(log n)‡ | O(log n)‡ | O(n) |
| **Heap** | O(n) | O(n) | O(log n) | O(log n) | O(n) |
| **Trie** | O(m) | O(m) | O(m) | O(m) | O(n×m) |
| **Graph (Adj List)** | O(V+E) | O(V+E) | O(1) | O(E) | O(V+E) |
| **Graph (Adj Matrix)** | O(1) | O(V) | O(1) | O(1) | O(V²) |

\* Amortized for append  
\** With pointer to position  
† Average case; O(n) worst  
‡ Balanced tree; O(n) worst  

---

## 🎯 When to Use What?

| Need | Best Choice | Why |
|------|-------------|-----|
| **Fast Random Access** | Array, ArrayList | O(1) index access |
| **Fast Search by Key** | HashMap, HashSet | O(1) average lookup |
| **Sorted Data** | TreeMap, TreeSet, BST | Maintains order |
| **LIFO (Last In First Out)** | Stack | Function calls, undo/redo |
| **FIFO (First In First Out)** | Queue | BFS, task scheduling |
| **Priority Access** | Heap, PriorityQueue | Top K, Dijkstra |
| **Frequent Insert/Delete** | LinkedList | O(1) at known position |
| **Hierarchical Data** | Tree | File systems, DOM |
| **Relationships/Networks** | Graph | Social networks, maps |
| **Prefix Matching** | Trie | Autocomplete, dictionary |
| **Range Queries** | Segment Tree, Fenwick | Sum/min/max in range |

---

## 🔥 Common Problem Patterns

### 📊 **Arrays**
- **Two Pointers**: Reverse, pair sum, remove duplicates
- **Sliding Window**: Max sum subarray, longest substring
- **Prefix Sum**: Range sum queries
- **Binary Search**: Search in sorted array

### 🔗 **LinkedList**
- **Fast & Slow Pointers**: Find middle, detect cycle
- **Reverse**: Iterative vs recursive
- **Merge**: Combine sorted lists
- **Dummy Node**: Simplify edge cases

### 📚 **Stacks**
- **Valid Parentheses**: Matching brackets
- **Monotonic Stack**: Next greater/smaller element
- **Expression Parsing**: Infix/postfix conversion
- **DFS**: Graph/tree traversal

### 📬 **Queues**
- **BFS**: Level-order traversal, shortest path
- **Sliding Window Max**: Monotonic deque
- **Task Scheduling**: Round-robin

### ⚡ **Hash Tables**
- **Two Sum**: Complement lookup
- **Frequency Counter**: Count occurrences
- **Group Anagrams**: Use sorted string as key
- **Caching**: LRU cache implementation

### 🌳 **Trees**
- **DFS Traversals**: Inorder, preorder, postorder
- **BFS**: Level-order traversal
- **Path Problems**: Root to leaf paths
- **Height/Depth**: Recursive calculation

### 🗺️ **Graphs**
- **BFS**: Shortest path (unweighted)
- **DFS**: Cycle detection, topological sort
- **Union-Find**: Connected components
- **Dijkstra**: Shortest path (weighted)

### 🏔️ **Heaps**
- **Top K Elements**: Min/max heap
- **Kth Largest**: Min heap of size k
- **Merge K Lists**: Priority queue
- **Median Finding**: Two heaps

---

## 💻 Java Quick Reference

### **Array**
```java
int[] arr = new int[5];
arr[0] = 10;
int len = arr.length;
```

### **ArrayList**
```java
ArrayList<Integer> list = new ArrayList<>();
list.add(10);
list.get(0);
list.remove(0);
list.size();
```

### **LinkedList**
```java
LinkedList<Integer> list = new LinkedList<>();
list.addFirst(10);
list.addLast(20);
list.removeFirst();
```

### **Stack**
```java
Stack<Integer> stack = new Stack<>();
stack.push(10);
stack.pop();
stack.peek();
stack.isEmpty();
```

### **Queue**
```java
Queue<Integer> queue = new LinkedList<>();
queue.offer(10);
queue.poll();
queue.peek();
```

### **HashMap**
```java
HashMap<String, Integer> map = new HashMap<>();
map.put("key", 10);
map.get("key");
map.containsKey("key");
map.remove("key");
```

### **HashSet**
```java
HashSet<Integer> set = new HashSet<>();
set.add(10);
set.contains(10);
set.remove(10);
```

### **TreeMap**
```java
TreeMap<Integer, String> map = new TreeMap<>();
map.put(1, "one");
map.firstKey();
map.lastKey();
```

### **PriorityQueue (Min Heap)**
```java
PriorityQueue<Integer> minHeap = new PriorityQueue<>();
minHeap.offer(10);
minHeap.poll();
minHeap.peek();
```

### **PriorityQueue (Max Heap)**
```java
PriorityQueue<Integer> maxHeap = 
    new PriorityQueue<>(Collections.reverseOrder());
```

---

## 🎯 Problem-Solving Strategy

### **Step 1: Understand the Problem**
- What's the input/output?
- What are the constraints?
- What's the expected time complexity?

### **Step 2: Identify the Pattern**
- Is it a two-pointer problem?
- Do I need BFS/DFS?
- Is frequency counting needed?
- Can I use a hash map for O(1) lookup?

### **Step 3: Choose Data Structure**
- Does order matter? → Array/List
- Need fast lookup? → HashMap/HashSet
- Priority access? → Heap
- Relationships? → Graph
- Hierarchical? → Tree

### **Step 4: Optimize**
- Can I reduce time complexity?
- Is there a space-time trade-off?
- Are there edge cases?

---

## 🚀 Interview Tips

### ✅ **Do's**
1. Clarify requirements and constraints
2. Start with brute force, then optimize
3. Explain your thought process aloud
4. Test with edge cases
5. Analyze time and space complexity

### ❌ **Don'ts**
1. Jump to coding without planning
2. Ignore edge cases (empty, null, single element)
3. Forget to handle duplicates
4. Assume sorted input
5. Use magic numbers (use constants)

---

## 🎮 Big-O Cheat Sheet

| Notation | Name | Example | For n=1M |
|----------|------|---------|----------|
| O(1) | Constant | Array access | 1 op |
| O(log n) | Logarithmic | Binary search | 20 ops |
| O(n) | Linear | Loop | 1M ops |
| O(n log n) | Linearithmic | Merge sort | 20M ops |
| O(n²) | Quadratic | Nested loops | 1T ops |
| O(2ⁿ) | Exponential | Fibonacci (naive) | ∞ |

---

## 📚 Common Java Collections Methods

### **Arrays**
```java
Arrays.sort(arr);
Arrays.fill(arr, 0);
Arrays.copyOf(arr, newLength);
Arrays.equals(arr1, arr2);
Arrays.asList(1, 2, 3);
```

### **Collections**
```java
Collections.sort(list);
Collections.reverse(list);
Collections.shuffle(list);
Collections.max(list);
Collections.min(list);
Collections.frequency(list, element);
```

### **String**
```java
s.charAt(i);
s.length();
s.substring(start, end);
s.split(" ");
s.toCharArray();
s.equals(other);
s.compareTo(other);
```

---

## 🎯 Top LeetCode Patterns

1. **Sliding Window** (3, 76, 239, 438, 567)
2. **Two Pointers** (1, 15, 16, 18, 167)
3. **Fast & Slow Pointers** (141, 142, 202, 876)
4. **Merge Intervals** (56, 57, 252, 253)
5. **Cyclic Sort** (41, 268, 287, 442)
6. **In-place Reversal** (206, 92, 25)
7. **BFS** (102, 107, 199, 200)
8. **DFS** (94, 98, 100, 104)
9. **Two Heaps** (295, 480)
10. **Subsets** (78, 90)
11. **Modified Binary Search** (33, 34, 153, 162)
12. **Top K Elements** (215, 347, 692, 973)
13. **K-way Merge** (23, 373, 378)
14. **Dynamic Programming** (70, 198, 322, 518)
15. **Backtracking** (17, 22, 39, 40, 46)

---

## 🏆 Interview Preparation Checklist

- [ ] Understand all basic data structures
- [ ] Implement each DS from scratch
- [ ] Master common patterns
- [ ] Solve 100+ LeetCode problems
- [ ] Practice explaining solutions
- [ ] Analyze time/space complexity
- [ ] Handle edge cases
- [ ] Practice on whiteboard
- [ ] Mock interviews
- [ ] Review mistakes

---

**Keep this cheat sheet handy during your interview preparation! 🚀**

