# 🗺️ Graph Algorithms: Master the Network! 🚀

> **"Graphs are everywhere - from social networks to GPS navigation. Master graphs, master the real world!"**

Welcome to the **most comprehensive graph algorithms guide**! Learn BFS, DFS, Dijkstra's, and more - the essential algorithms powering Google Maps, Facebook, LinkedIn, and every network system! 🎯

---

## 📋 Table of Contents

1. [Graph Fundamentals](#-graph-fundamentals)
2. [Graph Representations](#-graph-representations)
3. [BFS (Breadth-First Search)](#-bfs-breadth-first-search)
4. [DFS (Depth-First Search)](#-dfs-depth-first-search)
5. [Dijkstra's Algorithm](#-dijkstras-algorithm)
6. [Topological Sort](#-topological-sort)
7. [Practice Problems](#-practice-problems)

---

## 🎯 Graph Fundamentals

### **What is a Graph?**

**Graph** = Collection of vertices (nodes) connected by edges

```
Vertices: {A, B, C, D}
Edges: {(A,B), (A,C), (B,D), (C,D)}

Visual:
    A --- B
    |     |
    C --- D
```

### **Graph Types**

```java
// 1. DIRECTED (one-way edges)
A → B → C
↓       ↑
D ------+

// 2. UNDIRECTED (two-way edges)
A --- B
|     |
C --- D

// 3. WEIGHTED (edges have values)
    5
A ----- B
|3    2|
C ----- D
    4

// 4. CYCLIC vs ACYCLIC
Cyclic: A → B → C → A (has cycle)
Acyclic (DAG): A → B → C (no cycles)
```

---

## 💾 Graph Representations

### **1. Adjacency Matrix** (2D Array)

```java
public class GraphMatrix {
    private int[][] adj;
    private int V;  // Number of vertices
    
    public GraphMatrix(int vertices) {
        V = vertices;
        adj = new int[V][V];
    }
    
    public void addEdge(int u, int v, int weight) {
        adj[u][v] = weight;  // Directed
        adj[v][u] = weight;  // Undirected (add this line)
    }
    
    public boolean hasEdge(int u, int v) {
        return adj[u][v] != 0;
    }
}
```

**Visual**:
```
Graph:      Matrix:
  0 - 1        0  1  2  3
  |   |       +-----------
  2 - 3     0 | 0  1  1  0
           1 | 1  0  0  1
           2 | 1  0  0  1
           3 | 0  1  1  0
```

**Pros**: ✅ O(1) edge lookup  
**Cons**: ❌ O(V²) space  

---

### **2. Adjacency List** (HashMap of Lists)

```java
public class GraphList {
    private Map<Integer, List<Edge>> adj;
    
    static class Edge {
        int dest;
        int weight;
        
        Edge(int dest, int weight) {
            this.dest = dest;
            this.weight = weight;
        }
    }
    
    public GraphList() {
        adj = new HashMap<>();
    }
    
    public void addVertex(int v) {
        adj.putIfAbsent(v, new ArrayList<>());
    }
    
    public void addEdge(int u, int v, int weight) {
        adj.get(u).add(new Edge(v, weight));
        adj.get(v).add(new Edge(u, weight));  // Undirected
    }
    
    public List<Edge> getNeighbors(int v) {
        return adj.getOrDefault(v, new ArrayList<>());
    }
}
```

**Visual**:
```
Graph:      List:
  0 - 1     0 → [1, 2]
  |   |     1 → [0, 3]
  2 - 3     2 → [0, 3]
            3 → [1, 2]
```

**Pros**: ✅ O(V + E) space (efficient for sparse graphs)  
**Cons**: ❌ O(V) edge lookup  

---

## 🌊 BFS (Breadth-First Search)

### **Concept**
Explore level by level (like ripples in water).

### **Use Cases**
✅ **Shortest path** in unweighted graph  
✅ **Level-order traversal**  
✅ **Minimum steps** to reach target  

### **Template**
```java
public void bfs(int start) {
    Queue<Integer> queue = new LinkedList<>();
    Set<Integer> visited = new HashSet<>();
    
    queue.offer(start);
    visited.add(start);
    
    while (!queue.isEmpty()) {
        int node = queue.poll();
        System.out.print(node + " ");
        
        for (int neighbor : graph.get(node)) {
            if (!visited.contains(neighbor)) {
                visited.add(neighbor);
                queue.offer(neighbor);
            }
        }
    }
}
```

### **Example: Shortest Path**
```java
public int shortestPath(int[][] graph, int start, int end) {
    Queue<Integer> queue = new LinkedList<>();
    Map<Integer, Integer> distance = new HashMap<>();
    
    queue.offer(start);
    distance.put(start, 0);
    
    while (!queue.isEmpty()) {
        int node = queue.poll();
        
        if (node == end) {
            return distance.get(node);
        }
        
        for (int neighbor : graph[node]) {
            if (!distance.containsKey(neighbor)) {
                distance.put(neighbor, distance.get(node) + 1);
                queue.offer(neighbor);
            }
        }
    }
    
    return -1;  // Not reachable
}
```

**Visual**:
```
Graph:
    1
   / \
  2   3
 / \   \
4   5   6

BFS from 1: 1 → 2,3 → 4,5,6
Levels:     0   1     2
```

**Complexity**: O(V + E) time, O(V) space

---

## 🌳 DFS (Depth-First Search)

### **Concept**
Explore as deep as possible before backtracking.

### **Use Cases**
✅ **Detect cycles**  
✅ **Topological sort**  
✅ **Connected components**  
✅ **Path finding** (all paths)  

### **Template (Recursive)**
```java
public void dfs(int node, Set<Integer> visited) {
    visited.add(node);
    System.out.print(node + " ");
    
    for (int neighbor : graph.get(node)) {
        if (!visited.contains(neighbor)) {
            dfs(neighbor, visited);
        }
    }
}
```

### **Template (Iterative)**
```java
public void dfsIterative(int start) {
    Stack<Integer> stack = new Stack<>();
    Set<Integer> visited = new HashSet<>();
    
    stack.push(start);
    
    while (!stack.isEmpty()) {
        int node = stack.pop();
        
        if (visited.contains(node)) continue;
        visited.add(node);
        System.out.print(node + " ");
        
        for (int neighbor : graph.get(node)) {
            if (!visited.contains(neighbor)) {
                stack.push(neighbor);
            }
        }
    }
}
```

### **Example: Detect Cycle**
```java
public boolean hasCycle(int node, Set<Integer> visited, 
                        Set<Integer> recStack) {
    visited.add(node);
    recStack.add(node);
    
    for (int neighbor : graph.get(node)) {
        if (!visited.contains(neighbor)) {
            if (hasCycle(neighbor, visited, recStack)) {
                return true;
            }
        } else if (recStack.contains(neighbor)) {
            return true;  // Cycle detected!
        }
    }
    
    recStack.remove(node);  // Backtrack
    return false;
}
```

**Visual**:
```
Graph:
    1
   / \
  2   3
 / \   \
4   5   6

DFS from 1: 1 → 2 → 4 (back) → 5 (back to 1) → 3 → 6
```

**Complexity**: O(V + E) time, O(V) space

---

## ⚡ Dijkstra's Algorithm

### **Concept**
Find shortest path in **weighted** graph (non-negative weights).

### **Use Cases**
✅ **GPS navigation** (shortest route)  
✅ **Network routing** (fastest path)  
✅ **Game AI** (pathfinding)  

### **Template**
```java
public int[] dijkstra(Map<Integer, List<Edge>> graph, int start, int n) {
    int[] dist = new int[n];
    Arrays.fill(dist, Integer.MAX_VALUE);
    dist[start] = 0;
    
    PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[1] - b[1]);
    pq.offer(new int[]{start, 0});  // {node, distance}
    
    while (!pq.isEmpty()) {
        int[] curr = pq.poll();
        int node = curr[0];
        int d = curr[1];
        
        if (d > dist[node]) continue;  // Already processed
        
        for (Edge edge : graph.getOrDefault(node, new ArrayList<>())) {
            int newDist = dist[node] + edge.weight;
            
            if (newDist < dist[edge.dest]) {
                dist[edge.dest] = newDist;
                pq.offer(new int[]{edge.dest, newDist});
            }
        }
    }
    
    return dist;
}
```

**Visual**:
```
Graph:
     1
   /5|2\
  A  B  C
   \3|4/
     D

Dijkstra from A:
Step 1: A(0)
Step 2: A(0) → B(5), C(2)
Step 3: A(0) → B(5), C(2) → D(6)
Step 4: A(0) → B(5), C(2) → D(6) ← B(7)

Shortest paths: A:0, B:5, C:2, D:6
```

**Complexity**: O((V + E) log V) with min-heap

---

## 📊 Topological Sort

### **Concept**
Linear ordering of vertices in **DAG** (Directed Acyclic Graph).

### **Use Cases**
✅ **Task scheduling** with dependencies  
✅ **Course prerequisites**  
✅ **Build systems** (compile order)  

### **Template (Kahn's Algorithm - BFS)**
```java
public List<Integer> topologicalSort(int n, int[][] edges) {
    // Build graph and calculate in-degrees
    Map<Integer, List<Integer>> graph = new HashMap<>();
    int[] inDegree = new int[n];
    
    for (int i = 0; i < n; i++) {
        graph.put(i, new ArrayList<>());
    }
    
    for (int[] edge : edges) {
        int from = edge[0], to = edge[1];
        graph.get(from).add(to);
        inDegree[to]++;
    }
    
    // Find nodes with in-degree 0
    Queue<Integer> queue = new LinkedList<>();
    for (int i = 0; i < n; i++) {
        if (inDegree[i] == 0) {
            queue.offer(i);
        }
    }
    
    // Process nodes
    List<Integer> result = new ArrayList<>();
    while (!queue.isEmpty()) {
        int node = queue.poll();
        result.add(node);
        
        for (int neighbor : graph.get(node)) {
            inDegree[neighbor]--;
            if (inDegree[neighbor] == 0) {
                queue.offer(neighbor);
            }
        }
    }
    
    // Check for cycle
    return result.size() == n ? result : new ArrayList<>();
}
```

**Example: Course Schedule**
```java
public boolean canFinish(int numCourses, int[][] prerequisites) {
    List<Integer> order = topologicalSort(numCourses, prerequisites);
    return order.size() == numCourses;
}
```

**Visual**:
```
Courses with prerequisites:
0 → 1 → 3
↓   ↓
2 → 4

Topological order: [0, 2, 1, 4, 3]
(One valid order among multiple)
```

**Complexity**: O(V + E)

---

## 🏢 Real-World Applications

### **Google Maps** 🗺️
- Dijkstra's for shortest route
- A* for optimized pathfinding

### **Facebook/LinkedIn** 👥
- BFS for friend suggestions
- DFS for mutual connections

### **Compiler** 💻
- Topological sort for dependencies
- DFS for syntax tree traversal

### **Network Routing** 🌐
- Dijkstra's/Bellman-Ford for packet routing

---

## 🧩 Practice Problems

### 🟢 **Easy**
1. [Find Center of Star Graph](https://leetcode.com/problems/find-center-of-star-graph/)
2. [Find if Path Exists](https://leetcode.com/problems/find-if-path-exists-in-graph/)
3. [Flood Fill](https://leetcode.com/problems/flood-fill/)

### 🟡 **Medium**
4. [Number of Islands](https://leetcode.com/problems/number-of-islands/)
5. [Clone Graph](https://leetcode.com/problems/clone-graph/)
6. [Course Schedule](https://leetcode.com/problems/course-schedule/)
7. [Course Schedule II](https://leetcode.com/problems/course-schedule-ii/)
8. [Rotting Oranges](https://leetcode.com/problems/rotting-oranges/)
9. [Network Delay Time](https://leetcode.com/problems/network-delay-time/)
10. [All Paths Source to Target](https://leetcode.com/problems/all-paths-from-source-to-target/)

### 🔴 **Hard**
11. [Word Ladder II](https://leetcode.com/problems/word-ladder-ii/)
12. [Alien Dictionary](https://leetcode.com/problems/alien-dictionary/)

---

## 💡 Pro Tips

✅ **Choose BFS** for shortest path (unweighted)  
✅ **Choose DFS** for cycle detection, topological sort  
✅ **Use Dijkstra's** for weighted shortest path  
✅ **Adjacency list** for sparse graphs  
✅ **Draw the graph** before coding!  

---

## 🎯 Key Takeaways

1. **BFS** = Queue, level-by-level, shortest path
2. **DFS** = Stack/Recursion, depth-first, cycles
3. **Dijkstra's** = PriorityQueue, weighted shortest path
4. **Topological Sort** = DAG ordering, dependencies
5. **Choose representation** based on graph density

---

## 🚀 What's Next?

➡️ **[Greedy Algorithms](../Greedy/01_Greedy_Fundamentals.md)**  
➡️ **[Practice Problems](../ProblemSets/README.md)**  

---

**🎮 Achievement Unlocked: Graph Master!** 🏆

*Remember: "Graphs connect the world!"* 🗺️✨

---

*Previous: [← Backtracking Patterns](../Recursion/02_Backtracking_Patterns.md) | Next: [DP Fundamentals →](../DynamicProgramming/01_DP_Fundamentals.md)*

