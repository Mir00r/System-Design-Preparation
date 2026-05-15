# 📋 Algorithm Cheat Sheet: Your Quick Reference Guide! ⚡

> **"When in doubt, reference this out!"**

Your **one-stop quick reference** for all algorithms, complexities, patterns, and when to use what! Keep this handy during interviews and practice sessions. 🎯

---

## 📊 Table of Contents

1. [Sorting Algorithms](#-sorting-algorithms)
2. [Searching Algorithms](#-searching-algorithms)
3. [Graph Algorithms](#-graph-algorithms)
4. [Dynamic Programming Patterns](#-dynamic-programming-patterns)
5. [Algorithm Selection Guide](#-algorithm-selection-guide)
6. [Time Complexity Quick Reference](#-time-complexity-quick-reference)
7. [Java Collections Complexity](#-java-collections-complexity)
8. [Common Patterns](#-common-patterns)

---

## 🔄 Sorting Algorithms

### **Complete Comparison Table**

| Algorithm | Best | Average | Worst | Space | Stable? | When to Use |
|-----------|------|---------|-------|-------|---------|-------------|
| **Bubble** | O(n) | O(n²) | O(n²) | O(1) | ✅ Yes | Teaching, nearly sorted |
| **Selection** | O(n²) | O(n²) | O(n²) | O(1) | ❌ No | Memory constrained |
| **Insertion** | O(n) | O(n²) | O(n²) | O(1) | ✅ Yes | Small/nearly sorted |
| **Merge** | O(n log n) | O(n log n) | O(n log n) | O(n) | ✅ Yes | Guaranteed performance |
| **Quick** | O(n log n) | O(n log n) | O(n²) | O(log n) | ❌ No | General purpose |
| **Heap** | O(n log n) | O(n log n) | O(n log n) | O(1) | ❌ No | In-place O(n log n) |
| **Counting** | O(n+k) | O(n+k) | O(n+k) | O(k) | ✅ Yes | Small integer range |
| **Radix** | O(d×n) | O(d×n) | O(d×n) | O(n+k) | ✅ Yes | Fixed-size integers |
| **Bucket** | O(n+k) | O(n+k) | O(n²) | O(n+k) | ✅ Yes | Uniformly distributed |

**Legend:**
- `n` = number of elements
- `k` = range of input (counting sort)
- `d` = number of digits (radix sort)

---

### **Sorting Algorithm Templates**

#### **Bubble Sort** - O(n²)
```java
public void bubbleSort(int[] arr) {
    for (int i = 0; i < arr.length - 1; i++) {
        boolean swapped = false;
        for (int j = 0; j < arr.length - 1 - i; j++) {
            if (arr[j] > arr[j + 1]) {
                swap(arr, j, j + 1);
                swapped = true;
            }
        }
        if (!swapped) break; // Optimization
    }
}
```

#### **Merge Sort** - O(n log n)
```java
public void mergeSort(int[] arr, int left, int right) {
    if (left < right) {
        int mid = left + (right - left) / 2;
        mergeSort(arr, left, mid);
        mergeSort(arr, mid + 1, right);
        merge(arr, left, mid, right);
    }
}
```

#### **Quick Sort** - O(n log n) average
```java
public void quickSort(int[] arr, int low, int high) {
    if (low < high) {
        int pi = partition(arr, low, high);
        quickSort(arr, low, pi - 1);
        quickSort(arr, pi + 1, high);
    }
}
```

---

## 🔍 Searching Algorithms

### **Search Complexity Table**

| Algorithm | Time (Best) | Time (Avg) | Time (Worst) | Space | Precondition |
|-----------|-------------|------------|--------------|-------|--------------|
| **Linear** | O(1) | O(n) | O(n) | O(1) | None |
| **Binary** | O(1) | O(log n) | O(log n) | O(1) | Sorted |
| **Jump** | O(1) | O(√n) | O(√n) | O(1) | Sorted |
| **Interpolation** | O(1) | O(log log n) | O(n) | O(1) | Sorted + Uniform |
| **Exponential** | O(1) | O(log i) | O(log i) | O(1) | Sorted, unbounded |
| **Ternary** | O(1) | O(log₃ n) | O(log₃ n) | O(1) | Unimodal function |

---

### **Binary Search Templates**

#### **Standard Binary Search**
```java
public int binarySearch(int[] arr, int target) {
    int left = 0, right = arr.length - 1;
    while (left <= right) {
        int mid = left + (right - left) / 2;
        if (arr[mid] == target) return mid;
        else if (arr[mid] < target) left = mid + 1;
        else right = mid - 1;
    }
    return -1;
}
```

#### **Find First Occurrence**
```java
public int findFirst(int[] arr, int target) {
    int left = 0, right = arr.length - 1;
    int result = -1;
    while (left <= right) {
        int mid = left + (right - left) / 2;
        if (arr[mid] == target) {
            result = mid;
            right = mid - 1; // Continue left
        } else if (arr[mid] < target) {
            left = mid + 1;
        } else {
            right = mid - 1;
        }
    }
    return result;
}
```

#### **Search in Rotated Array**
```java
public int searchRotated(int[] arr, int target) {
    int left = 0, right = arr.length - 1;
    while (left <= right) {
        int mid = left + (right - left) / 2;
        if (arr[mid] == target) return mid;
        
        if (arr[left] <= arr[mid]) { // Left sorted
            if (target >= arr[left] && target < arr[mid])
                right = mid - 1;
            else
                left = mid + 1;
        } else { // Right sorted
            if (target > arr[mid] && target <= arr[right])
                left = mid + 1;
            else
                right = mid - 1;
        }
    }
    return -1;
}
```

---

## 🗺️ Graph Algorithms

### **Graph Algorithm Complexity**

| Algorithm | Time | Space | Use Case |
|-----------|------|-------|----------|
| **BFS** | O(V+E) | O(V) | Shortest path (unweighted), Level-order |
| **DFS** | O(V+E) | O(V) | Cycle detection, Topological sort |
| **Dijkstra** | O((V+E) log V) | O(V) | Shortest path (weighted, non-negative) |
| **Bellman-Ford** | O(VE) | O(V) | Shortest path (negative edges allowed) |
| **Floyd-Warshall** | O(V³) | O(V²) | All-pairs shortest path |
| **Kruskal** | O(E log E) | O(V) | Minimum spanning tree (edge-based) |
| **Prim** | O((V+E) log V) | O(V) | Minimum spanning tree (vertex-based) |
| **Topological** | O(V+E) | O(V) | DAG ordering, dependency resolution |
| **Tarjan/Kosaraju** | O(V+E) | O(V) | Strongly connected components |
| **A\*** | O(E) | O(V) | Heuristic shortest path |

**Legend:**
- `V` = number of vertices
- `E` = number of edges

---

### **Graph Templates**

#### **BFS (Shortest Path)**
```java
public int bfs(int[][] graph, int start, int end) {
    Queue<Integer> queue = new LinkedList<>();
    boolean[] visited = new boolean[graph.length];
    int[] distance = new int[graph.length];
    
    queue.offer(start);
    visited[start] = true;
    distance[start] = 0;
    
    while (!queue.isEmpty()) {
        int node = queue.poll();
        if (node == end) return distance[node];
        
        for (int neighbor : graph[node]) {
            if (!visited[neighbor]) {
                visited[neighbor] = true;
                distance[neighbor] = distance[node] + 1;
                queue.offer(neighbor);
            }
        }
    }
    return -1;
}
```

#### **DFS (Recursive)**
```java
public void dfs(int[][] graph, int node, boolean[] visited) {
    visited[node] = true;
    // Process node
    
    for (int neighbor : graph[node]) {
        if (!visited[neighbor]) {
            dfs(graph, neighbor, visited);
        }
    }
}
```

#### **Dijkstra's Algorithm**
```java
public int[] dijkstra(Map<Integer, List<int[]>> graph, int start, int n) {
    int[] dist = new int[n];
    Arrays.fill(dist, Integer.MAX_VALUE);
    dist[start] = 0;
    
    PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[1] - b[1]);
    pq.offer(new int[]{start, 0});
    
    while (!pq.isEmpty()) {
        int[] curr = pq.poll();
        int node = curr[0], d = curr[1];
        
        if (d > dist[node]) continue;
        
        for (int[] edge : graph.getOrDefault(node, new ArrayList<>())) {
            int neighbor = edge[0], weight = edge[1];
            int newDist = dist[node] + weight;
            
            if (newDist < dist[neighbor]) {
                dist[neighbor] = newDist;
                pq.offer(new int[]{neighbor, newDist});
            }
        }
    }
    return dist;
}
```

---

## 💎 Dynamic Programming Patterns

### **DP Pattern Recognition**

| Pattern | Recognition | Example Problems |
|---------|-------------|------------------|
| **0/1 Knapsack** | Choose/not choose items | Subset sum, Partition equal |
| **Unbounded Knapsack** | Unlimited items | Coin change, Rod cutting |
| **LCS (Longest Common)** | Two sequences comparison | Edit distance, Diff tool |
| **LIS (Longest Increasing)** | Find increasing sequence | Russian doll, Max envelope |
| **Palindrome** | Substring is palindrome | Longest palindromic, Partition |
| **Matrix Chain** | Optimal grouping | Burst balloons, Scramble string |
| **DP on Trees** | Tree structure | House robber III, Diameter |
| **Digit DP** | Number constraints | Count numbers, Sum of digits |
| **Bitmask DP** | State as bitmask | Traveling salesman, Hamiltonian |
| **State Machine DP** | Finite states | Best time stock, Regex matching |

---

### **DP Templates**

#### **Top-Down (Memoization)**
```java
public int dpTopDown(int n) {
    int[] memo = new int[n + 1];
    Arrays.fill(memo, -1);
    return dpHelper(n, memo);
}

private int dpHelper(int n, int[] memo) {
    if (n <= 1) return n; // Base case
    if (memo[n] != -1) return memo[n];
    
    memo[n] = dpHelper(n - 1, memo) + dpHelper(n - 2, memo);
    return memo[n];
}
```

#### **Bottom-Up (Tabulation)**
```java
public int dpBottomUp(int n) {
    if (n <= 1) return n;
    
    int[] dp = new int[n + 1];
    dp[0] = 0;
    dp[1] = 1;
    
    for (int i = 2; i <= n; i++) {
        dp[i] = dp[i - 1] + dp[i - 2];
    }
    
    return dp[n];
}
```

#### **Space Optimized**
```java
public int dpOptimized(int n) {
    if (n <= 1) return n;
    
    int prev2 = 0, prev1 = 1;
    
    for (int i = 2; i <= n; i++) {
        int curr = prev1 + prev2;
        prev2 = prev1;
        prev1 = curr;
    }
    
    return prev1;
}
```

---

## 🎯 Algorithm Selection Guide

### **When to Use Which Algorithm?**

```
┌─ Need to SORT? ─────────────────────────────────┐
│                                                  │
│  Small data (n < 50)?           → Insertion     │
│  Need stability?                → Merge         │
│  Memory constrained?            → Quick/Heap    │
│  Integers with small range?     → Counting      │
│  Guaranteed O(n log n)?         → Merge/Heap    │
│  General purpose?               → Quick         │
└──────────────────────────────────────────────────┘

┌─ Need to SEARCH? ───────────────────────────────┐
│                                                  │
│  Data sorted?                   → Binary        │
│  Data unsorted?                 → Linear/Hash   │
│  Unbounded array?               → Exponential   │
│  Uniformly distributed?         → Interpolation │
│  Find peak in unimodal?         → Ternary       │
└──────────────────────────────────────────────────┘

┌─ OPTIMIZATION Problem? ─────────────────────────┐
│                                                  │
│  Overlapping subproblems?       → DP            │
│  Optimal substructure?          → DP/Greedy     │
│  Greedy choice works?           → Greedy        │
│  Need all solutions?            → Backtracking  │
└──────────────────────────────────────────────────┘

┌─ GRAPH Problem? ────────────────────────────────┐
│                                                  │
│  Shortest path (unweighted)?    → BFS           │
│  Shortest path (weighted)?      → Dijkstra      │
│  Negative edge weights?         → Bellman-Ford  │
│  All-pairs shortest path?       → Floyd         │
│  Minimum spanning tree?         → Kruskal/Prim  │
│  Detect cycle?                  → DFS/Union-Find│
│  Topological ordering?          → Kahn's/DFS    │
└──────────────────────────────────────────────────┘
```

---

## ⏱️ Time Complexity Quick Reference

### **Common Growth Rates (Best to Worst)**

```
O(1)         Constant      │ ▫️
O(log n)     Logarithmic   │ ▫️▫️
O(√n)        Square root   │ ▫️▫️▫️
O(n)         Linear        │ ▫️▫️▫️▫️▫️
O(n log n)   Linearithmic  │ ▫️▫️▫️▫️▫️▫️▫️
O(n²)        Quadratic     │ ▫️▫️▫️▫️▫️▫️▫️▫️▫️▫️▫️▫️▫️▫️
O(n³)        Cubic         │ ▫️▫️▫️▫️▫️▫️▫️▫️▫️▫️▫️▫️▫️▫️▫️▫️▫️
O(2ⁿ)        Exponential   │ ▫️▫️▫️▫️...💥 (explodes!)
O(n!)        Factorial     │ ▫️▫️▫️▫️...💥💥 (disaster!)
```

### **Input Size vs Maximum Complexity**

| Input Size (n) | Max Acceptable | Algorithms |
|----------------|----------------|------------|
| n ≤ 10 | O(n!) | Permutations, TSP brute force |
| n ≤ 20 | O(2ⁿ) | Subset generation, DP with bitmask |
| n ≤ 500 | O(n³) | Floyd-Warshall, DP 2D optimization |
| n ≤ 5,000 | O(n²) | Bubble/Selection sort, nested loops |
| n ≤ 100,000 | O(n log n) | Merge/Quick sort, Binary search tree |
| n ≤ 1,000,000 | O(n) | Linear search, Hash table, BFS/DFS |
| n > 1,000,000 | O(log n) or O(1) | Binary search, Hash table lookup |

---

## ☕ Java Collections Complexity

### **List Implementations**

| Operation | ArrayList | LinkedList |
|-----------|-----------|------------|
| get(i) | O(1) | O(n) |
| add(element) | O(1)* | O(1) |
| add(i, element) | O(n) | O(n) |
| remove(i) | O(n) | O(n) |
| contains() | O(n) | O(n) |

*Amortized

### **Set Implementations**

| Operation | HashSet | TreeSet | LinkedHashSet |
|-----------|---------|---------|---------------|
| add() | O(1) | O(log n) | O(1) |
| remove() | O(1) | O(log n) | O(1) |
| contains() | O(1) | O(log n) | O(1) |
| Ordering | ❌ No | ✅ Sorted | ✅ Insertion |

### **Map Implementations**

| Operation | HashMap | TreeMap | LinkedHashMap |
|-----------|---------|---------|---------------|
| put() | O(1) | O(log n) | O(1) |
| get() | O(1) | O(log n) | O(1) |
| remove() | O(1) | O(log n) | O(1) |
| Ordering | ❌ No | ✅ Sorted | ✅ Insertion |

### **Queue Implementations**

| Operation | LinkedList | PriorityQueue | ArrayDeque |
|-----------|------------|---------------|------------|
| offer() | O(1) | O(log n) | O(1) |
| poll() | O(1) | O(log n) | O(1) |
| peek() | O(1) | O(1) | O(1) |

---

## 🎯 Common Patterns

### **Two Pointers**
```java
// Opposite direction
int left = 0, right = arr.length - 1;
while (left < right) {
    // Process
    if (condition) left++;
    else right--;
}

// Same direction (fast & slow)
int slow = 0, fast = 0;
while (fast < arr.length) {
    if (condition) {
        arr[slow++] = arr[fast];
    }
    fast++;
}
```

### **Sliding Window**
```java
// Fixed size
int windowSum = 0;
for (int i = 0; i < k; i++) windowSum += arr[i];
int maxSum = windowSum;

for (int i = k; i < arr.length; i++) {
    windowSum = windowSum - arr[i - k] + arr[i];
    maxSum = Math.max(maxSum, windowSum);
}

// Variable size
int left = 0, maxLen = 0;
for (int right = 0; right < arr.length; right++) {
    // Expand window
    while (/* invalid condition */) {
        // Shrink window
        left++;
    }
    maxLen = Math.max(maxLen, right - left + 1);
}
```

### **Fast & Slow Pointers (Cycle Detection)**
```java
public boolean hasCycle(ListNode head) {
    ListNode slow = head, fast = head;
    
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        if (slow == fast) return true;
    }
    return false;
}
```

### **Backtracking Template**
```java
public void backtrack(State state, List<Result> result) {
    if (isGoal(state)) {
        result.add(new Result(state));
        return;
    }
    
    for (Choice choice : getChoices(state)) {
        makeChoice(state, choice);
        backtrack(state, result);
        undoChoice(state, choice);
    }
}
```

### **Union-Find (Disjoint Set)**
```java
class UnionFind {
    private int[] parent, rank;
    
    public UnionFind(int n) {
        parent = new int[n];
        rank = new int[n];
        for (int i = 0; i < n; i++) parent[i] = i;
    }
    
    public int find(int x) {
        if (parent[x] != x) {
            parent[x] = find(parent[x]); // Path compression
        }
        return parent[x];
    }
    
    public boolean union(int x, int y) {
        int px = find(x), py = find(y);
        if (px == py) return false;
        
        // Union by rank
        if (rank[px] < rank[py]) {
            parent[px] = py;
        } else if (rank[px] > rank[py]) {
            parent[py] = px;
        } else {
            parent[py] = px;
            rank[px]++;
        }
        return true;
    }
}
```

---

## 🧮 Bit Manipulation Tricks

```java
// Check if power of 2
boolean isPowerOf2 = (n & (n - 1)) == 0 && n != 0;

// Get ith bit
int getBit = (n >> i) & 1;

// Set ith bit
int setBit = n | (1 << i);

// Clear ith bit
int clearBit = n & ~(1 << i);

// Toggle ith bit
int toggleBit = n ^ (1 << i);

// Count set bits (Brian Kernighan)
int count = 0;
while (n != 0) {
    n &= (n - 1);
    count++;
}

// Find single number (all others appear twice)
int single = 0;
for (int num : nums) single ^= num;
```

---

## 📝 Interview Checklist

### **Before Coding:**
```
✅ Understand the problem
✅ Ask clarifying questions
✅ Discuss edge cases
✅ Propose brute force first
✅ Optimize approach
✅ Discuss time/space complexity
```

### **While Coding:**
```
✅ Think out loud
✅ Write clean, readable code
✅ Use meaningful variable names
✅ Handle edge cases
✅ Test with examples
```

### **After Coding:**
```
✅ Dry run with example
✅ State complexity analysis
✅ Discuss optimization
✅ Mention tradeoffs
✅ Suggest improvements
```

---

## 🚀 Quick Problem-Solving Framework

```
1. UNDERSTAND
   - What is input/output?
   - Constraints?
   - Edge cases?

2. PLAN
   - Brute force approach?
   - Can we optimize?
   - Which pattern fits?

3. CODE
   - Start with brute force
   - Optimize step by step
   - Handle edge cases

4. TEST
   - Normal case
   - Edge cases
   - Large input

5. ANALYZE
   - Time complexity
   - Space complexity
   - Can we do better?
```

---

## 🎯 Pattern Recognition Shortcuts

| Keywords in Problem | Likely Pattern/Algorithm |
|---------------------|--------------------------|
| "Sorted array" | Binary search, Two pointers |
| "Subarray sum/product" | Sliding window, Prefix sum |
| "Substring" | Sliding window, Hash map |
| "Permutations/Combinations" | Backtracking |
| "Connected components" | DFS, BFS, Union-Find |
| "Shortest path" | BFS, Dijkstra, Bellman-Ford |
| "Optimize/minimize/maximize" | DP, Greedy |
| "K largest/smallest" | Heap, Quick select |
| "Palindrome" | DP, Two pointers |
| "Interval" | Sorting, Greedy |

---

**🎯 Bookmark this page! It's your algorithm survival guide!** 📌

**Happy Coding! May Your Complexity Always Be Optimal! ⚡**

---

*Last Updated: May 15, 2026*  
*Quick Reference for 60+ Algorithms*
