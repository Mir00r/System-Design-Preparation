# 🗺️ Graphs: The Network Powerhouse! 🚀

---

## 🎯 Chapter 1: Graph Fundamentals

> **"Graphs: Where everything is connected!"**

---

## 🤔 What is a Graph?

A **Graph** is a non-linear data structure consisting of **vertices (nodes)** connected by **edges**.

### 🏗️ **Visual Representation**

```
Undirected Graph:
    A --- B
    |     |
    C --- D

Directed Graph (Digraph):
    A → B
    ↑   ↓
    C ← D

Weighted Graph:
    A --5-- B
    |       |
    2       3
    |       |
    C --1-- D
```

---

## 📚 Graph Terminology

```
Graph G = (V, E)
V = {A, B, C, D} (Vertices)
E = {(A,B), (A,C), (B,D), (C,D)} (Edges)

✅ Vertex (Node): Point in graph (A, B, C, D)
✅ Edge: Connection between vertices
✅ Degree: Number of edges connected to vertex
✅ Path: Sequence of vertices connected by edges
✅ Cycle: Path that starts and ends at same vertex
✅ Connected: Every vertex reachable from any other
✅ Weighted: Edges have values (costs/distances)
```

---

## 🎨 Types of Graphs

### 1️⃣ **Undirected Graph**
```
A --- B
|     |
C --- D

Edge (A, B) = Edge (B, A)
```

### 2️⃣ **Directed Graph (Digraph)**
```
A → B
↑   ↓
C ← D

Edge (A, B) ≠ Edge (B, A)
```

### 3️⃣ **Weighted Graph**
```
A --5-- B
|       |
2       3
|       |
C --1-- D

Each edge has a weight (cost)
```

### 4️⃣ **Cyclic vs Acyclic**
```
Cyclic:       Acyclic (DAG):
A → B         A → B
↑   ↓         ↓   ↓
D ← C         C → D
```

---

## 💻 Graph Representations

### 🔧 **1. Adjacency Matrix**

```java
// For graph: A-B, A-C, B-D, C-D
//     A  B  C  D
// A [ 0  1  1  0 ]
// B [ 1  0  0  1 ]
// C [ 1  0  0  1 ]
// D [ 0  1  1  0 ]

public class GraphMatrix {
    private int[][] adjMatrix;
    private int vertices;
    
    public GraphMatrix(int vertices) {
        this.vertices = vertices;
        adjMatrix = new int[vertices][vertices];
    }
    
    public void addEdge(int source, int dest) {
        adjMatrix[source][dest] = 1;
        adjMatrix[dest][source] = 1; // For undirected
    }
    
    public boolean hasEdge(int source, int dest) {
        return adjMatrix[source][dest] == 1;
    }
}

// Space: O(V²)
// Edge lookup: O(1)
// Iterate neighbors: O(V)
```

---

### 🔧 **2. Adjacency List (Most Common)**

```java
// For graph: A-B, A-C, B-D, C-D
// A → [B, C]
// B → [A, D]
// C → [A, D]
// D → [B, C]

import java.util.*;

public class Graph {
    private Map<Integer, List<Integer>> adjList;
    
    public Graph() {
        adjList = new HashMap<>();
    }
    
    public void addVertex(int vertex) {
        adjList.putIfAbsent(vertex, new ArrayList<>());
    }
    
    public void addEdge(int source, int dest) {
        adjList.putIfAbsent(source, new ArrayList<>());
        adjList.putIfAbsent(dest, new ArrayList<>());
        
        adjList.get(source).add(dest);
        adjList.get(dest).add(source); // For undirected
    }
    
    public List<Integer> getNeighbors(int vertex) {
        return adjList.getOrDefault(vertex, new ArrayList<>());
    }
}

// Space: O(V + E)
// Edge lookup: O(degree(V))
// Iterate neighbors: O(degree(V))
```

---

## 🎯 Graph Traversal Algorithms

### 🔥 **1. Breadth-First Search (BFS)**

```java
public void bfs(int start) {
    Set<Integer> visited = new HashSet<>();
    Queue<Integer> queue = new LinkedList<>();
    
    queue.offer(start);
    visited.add(start);
    
    while (!queue.isEmpty()) {
        int current = queue.poll();
        System.out.print(current + " ");
        
        for (int neighbor : adjList.get(current)) {
            if (!visited.contains(neighbor)) {
                visited.add(neighbor);
                queue.offer(neighbor);
            }
        }
    }
}

// Time: O(V + E)
// Space: O(V)
// Use for: Shortest path (unweighted), level-order
```

**Visual**:
```
Graph:      BFS from A:
    A       Level 0: [A]
   / \      Level 1: [B, C]
  B   C     Level 2: [D, E]
 / \
D   E

Output: A → B → C → D → E
```

---

### 🔥 **2. Depth-First Search (DFS)**

```java
// Recursive DFS
public void dfs(int vertex, Set<Integer> visited) {
    visited.add(vertex);
    System.out.print(vertex + " ");
    
    for (int neighbor : adjList.get(vertex)) {
        if (!visited.contains(neighbor)) {
            dfs(neighbor, visited);
        }
    }
}

// Iterative DFS (using stack)
public void dfsIterative(int start) {
    Set<Integer> visited = new HashSet<>();
    Stack<Integer> stack = new Stack<>();
    
    stack.push(start);
    
    while (!stack.isEmpty()) {
        int current = stack.pop();
        
        if (!visited.contains(current)) {
            visited.add(current);
            System.out.print(current + " ");
            
            for (int neighbor : adjList.get(current)) {
                if (!visited.contains(neighbor)) {
                    stack.push(neighbor);
                }
            }
        }
    }
}

// Time: O(V + E)
// Space: O(V)
// Use for: Cycle detection, topological sort, pathfinding
```

**Visual**:
```
Graph:      DFS from A:
    A       A → B → D (backtrack)
   / \          → E (backtrack to B, then A)
  B   C      → C
 / \
D   E

Output: A → B → D → E → C
```

---

## 🎯 Common Graph Patterns

### 🔥 **Pattern 1: Number of Connected Components**

```java
public int countComponents(int n, int[][] edges) {
    // Build adjacency list
    List<List<Integer>> adj = new ArrayList<>();
    for (int i = 0; i < n; i++) {
        adj.add(new ArrayList<>());
    }
    for (int[] edge : edges) {
        adj.get(edge[0]).add(edge[1]);
        adj.get(edge[1]).add(edge[0]);
    }
    
    boolean[] visited = new boolean[n];
    int count = 0;
    
    for (int i = 0; i < n; i++) {
        if (!visited[i]) {
            dfs(i, adj, visited);
            count++;
        }
    }
    
    return count;
}

private void dfs(int node, List<List<Integer>> adj, boolean[] visited) {
    visited[node] = true;
    for (int neighbor : adj.get(node)) {
        if (!visited[neighbor]) {
            dfs(neighbor, adj, visited);
        }
    }
}
```

---

### 🔥 **Pattern 2: Detect Cycle (Undirected Graph)**

```java
public boolean hasCycle(List<List<Integer>> adj, int n) {
    boolean[] visited = new boolean[n];
    
    for (int i = 0; i < n; i++) {
        if (!visited[i]) {
            if (hasCycleDFS(i, -1, adj, visited)) {
                return true;
            }
        }
    }
    return false;
}

private boolean hasCycleDFS(int node, int parent, 
                           List<List<Integer>> adj, 
                           boolean[] visited) {
    visited[node] = true;
    
    for (int neighbor : adj.get(node)) {
        if (!visited[neighbor]) {
            if (hasCycleDFS(neighbor, node, adj, visited)) {
                return true;
            }
        } else if (neighbor != parent) {
            return true; // Cycle found
        }
    }
    return false;
}
```

---

### 🔥 **Pattern 3: Topological Sort (DAG)**

```java
public List<Integer> topologicalSort(int n, List<List<Integer>> adj) {
    int[] indegree = new int[n];
    
    // Calculate indegrees
    for (List<Integer> neighbors : adj) {
        for (int neighbor : neighbors) {
            indegree[neighbor]++;
        }
    }
    
    Queue<Integer> queue = new LinkedList<>();
    for (int i = 0; i < n; i++) {
        if (indegree[i] == 0) {
            queue.offer(i);
        }
    }
    
    List<Integer> result = new ArrayList<>();
    
    while (!queue.isEmpty()) {
        int current = queue.poll();
        result.add(current);
        
        for (int neighbor : adj.get(current)) {
            indegree[neighbor]--;
            if (indegree[neighbor] == 0) {
                queue.offer(neighbor);
            }
        }
    }
    
    return result.size() == n ? result : new ArrayList<>();
}
```

---

### 🔥 **Pattern 4: Shortest Path (Dijkstra's Algorithm)**

```java
public int[] dijkstra(int n, int[][] edges, int source) {
    // Build adjacency list with weights
    List<List<int[]>> adj = new ArrayList<>();
    for (int i = 0; i < n; i++) {
        adj.add(new ArrayList<>());
    }
    for (int[] edge : edges) {
        adj.get(edge[0]).add(new int[]{edge[1], edge[2]});
    }
    
    int[] dist = new int[n];
    Arrays.fill(dist, Integer.MAX_VALUE);
    dist[source] = 0;
    
    PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[1] - b[1]);
    pq.offer(new int[]{source, 0});
    
    while (!pq.isEmpty()) {
        int[] current = pq.poll();
        int node = current[0];
        int distance = current[1];
        
        if (distance > dist[node]) continue;
        
        for (int[] neighbor : adj.get(node)) {
            int next = neighbor[0];
            int weight = neighbor[1];
            int newDist = distance + weight;
            
            if (newDist < dist[next]) {
                dist[next] = newDist;
                pq.offer(new int[]{next, newDist});
            }
        }
    }
    
    return dist;
}
```

---

## 🏢 Real-World Applications

### 🔷 **Social Networks**
```
Facebook Friends Graph:
Alice --- Bob
  |       |
Carol --- Diana

Find mutual friends, suggest connections
```

### 🔷 **Google Maps**
```
Cities (Vertices), Roads (Edges)
Find shortest route using Dijkstra
```

### 🔷 **Web Crawler**
```
Webpages (Vertices), Links (Edges)
Use BFS to crawl pages
```

### 🔷 **Recommendation Systems**
```
Users/Products connected by interactions
Collaborative filtering using graph algorithms
```

---

## 🧩 Practice Problems

### 🟢 **Easy**
1. Number of Islands (LeetCode #200)
2. Clone Graph (LeetCode #133)

### 🟡 **Medium**
3. Course Schedule (LeetCode #207)
4. Number of Connected Components (LeetCode #323)
5. Network Delay Time (LeetCode #743)

### 🔴 **Hard**
6. Word Ladder (LeetCode #127)
7. Alien Dictionary (LeetCode #269)

---

## 🎯 Key Takeaways

✅ Graphs represent **relationships/networks**  
✅ **BFS** for shortest path (unweighted)  
✅ **DFS** for cycle detection, topological sort  
✅ **Dijkstra** for shortest path (weighted)  
✅ Adjacency list is most common representation  

---

## 🚀 What's Next?

➡️ **[Heaps: Priority Queue Power](../Heaps/01_Heap_Fundamentals.md)**

**🎮 Achievement Unlocked: Graph Mastery!** 🏆

