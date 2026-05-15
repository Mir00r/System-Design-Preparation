# 📬 Queues: The FIFO Powerhouse! 🚀

---

## 🎯 Chapter 1: Queue Fundamentals

> **"Think of a queue like a line at a coffee shop: first person in line gets served first!"**

---

## 🤔 What is a Queue?

A **Queue** is a **First-In-First-Out (FIFO)** data structure where elements are added at the rear (enqueue) and removed from the front (dequeue).

### 🏗️ **Visual Representation**

```
Queue Operations:
                    ← ENQUEUE (rear)
┌─────┬─────┬─────┬─────┐
│  10 │  20 │  30 │  40 │
└─────┴─────┴─────┴─────┘
  ↑ DEQUEUE (front)
  
Front → 10, 20, 30, 40 ← Rear
```

---

## 🌍 Real-World Analogies

### 🏪 **Supermarket Checkout Line**
```
First customer in → First customer served
     [Customer 1] → [Customer 2] → [Customer 3] → [Customer 4]
      ↑ Front                                        ↑ Rear
    (Dequeue)                                     (Enqueue)
```

### 📞 **Call Center Queue**
```
Caller 1 (waiting 10 min) → Caller 2 (waiting 5 min) → Caller 3 (just joined)
  ↑ Next to be served                                    ↑ Just added
```

### 🖨️ **Print Queue**
```
Document 1 → Document 2 → Document 3
  ↑ Printing    ↑ Next     ↑ Just added
```

---

## 💻 Java Implementation

### 🔧 **Array-Based Circular Queue**

```java
public class CircularQueue {
    private int[] queue;
    private int front;
    private int rear;
    private int size;
    private int capacity;
    
    public CircularQueue(int capacity) {
        this.capacity = capacity;
        queue = new int[capacity];
        front = 0;
        rear = -1;
        size = 0;
    }
    
    // Enqueue - O(1)
    public boolean enqueue(int value) {
        if (isFull()) {
            return false;
        }
        
        rear = (rear + 1) % capacity; // Circular increment
        queue[rear] = value;
        size++;
        return true;
    }
    
    // Dequeue - O(1)
    public int dequeue() {
        if (isEmpty()) {
            throw new IllegalStateException("Queue is empty");
        }
        
        int value = queue[front];
        front = (front + 1) % capacity; // Circular increment
        size--;
        return value;
    }
    
    // Peek - O(1)
    public int peek() {
        if (isEmpty()) {
            throw new IllegalStateException("Queue is empty");
        }
        return queue[front];
    }
    
    public boolean isEmpty() {
        return size == 0;
    }
    
    public boolean isFull() {
        return size == capacity;
    }
    
    public int size() {
        return size;
    }
}
```

**Why Circular?**
```
Without Circular (wasteful):
Enqueue 1,2,3 → Dequeue 1,2 → Can't enqueue 4,5!
[_, _, 3, _, _]
 ↑ front  ↑ rear (can't use front positions)

With Circular (efficient):
Enqueue 1,2,3 → Dequeue 1,2 → Can enqueue 4,5!
[4, 5, 3, _, _]
       ↑ front
    ↑ rear (wraps around)
```

---

### 🔗 **LinkedList-Based Queue**

```java
public class LinkedQueue {
    private Node front;
    private Node rear;
    private int size;
    
    private class Node {
        int data;
        Node next;
        
        Node(int data) {
            this.data = data;
        }
    }
    
    public LinkedQueue() {
        front = null;
        rear = null;
        size = 0;
    }
    
    // Enqueue - O(1)
    public void enqueue(int value) {
        Node newNode = new Node(value);
        
        if (isEmpty()) {
            front = rear = newNode;
        } else {
            rear.next = newNode;
            rear = newNode;
        }
        size++;
    }
    
    // Dequeue - O(1)
    public int dequeue() {
        if (isEmpty()) {
            throw new IllegalStateException("Queue is empty");
        }
        
        int value = front.data;
        front = front.next;
        
        if (front == null) {
            rear = null;
        }
        
        size--;
        return value;
    }
    
    // Peek - O(1)
    public int peek() {
        if (isEmpty()) {
            throw new IllegalStateException("Queue is empty");
        }
        return front.data;
    }
    
    public boolean isEmpty() {
        return front == null;
    }
    
    public int size() {
        return size;
    }
}
```

---

### 📦 **Using Java's Built-in Queue**

```java
import java.util.*;

public class QueueExamples {
    public static void main(String[] args) {
        // Queue interface (use LinkedList implementation)
        Queue<Integer> queue = new LinkedList<>();
        
        // Enqueue (add to rear)
        queue.offer(10);  // Preferred (returns false if fails)
        queue.add(20);    // Throws exception if fails
        queue.offer(30);
        
        // Peek (view front without removing)
        int front = queue.peek();  // 10 (returns null if empty)
        int front2 = queue.element(); // 10 (throws exception if empty)
        
        // Dequeue (remove from front)
        int removed = queue.poll();   // 10 (returns null if empty)
        int removed2 = queue.remove(); // 20 (throws exception if empty)
        
        // Check if empty
        boolean empty = queue.isEmpty();
        
        // Get size
        int size = queue.size();
        
        // Iterate
        for (int num : queue) {
            System.out.println(num);
        }
    }
}
```

---

## 🎯 Queue Variants

### 🔥 **1. Deque (Double-Ended Queue)**

```java
import java.util.Deque;
import java.util.ArrayDeque;

public class DequeExample {
    public static void main(String[] args) {
        Deque<Integer> deque = new ArrayDeque<>();
        
        // Add to front
        deque.addFirst(10);
        deque.offerFirst(20);
        
        // Add to rear
        deque.addLast(30);
        deque.offerLast(40);
        
        // Remove from front
        int front = deque.removeFirst();
        int front2 = deque.pollFirst();
        
        // Remove from rear
        int rear = deque.removeLast();
        int rear2 = deque.pollLast();
        
        // Peek both ends
        int peekFront = deque.peekFirst();
        int peekRear = deque.peekLast();
    }
}
```

**Use Cases:**
- Sliding window problems
- Palindrome checking
- Browser history (back/forward)

---

### 🔥 **2. Priority Queue (Min Heap)**

```java
import java.util.PriorityQueue;

public class PriorityQueueExample {
    public static void main(String[] args) {
        // Min heap (smallest element has priority)
        PriorityQueue<Integer> pq = new PriorityQueue<>();
        
        pq.offer(30);
        pq.offer(10);
        pq.offer(20);
        
        System.out.println(pq.poll()); // 10 (smallest)
        System.out.println(pq.poll()); // 20
        System.out.println(pq.poll()); // 30
        
        // Max heap (largest element has priority)
        PriorityQueue<Integer> maxPQ = new PriorityQueue<>(Collections.reverseOrder());
        
        maxPQ.offer(30);
        maxPQ.offer(10);
        maxPQ.offer(20);
        
        System.out.println(maxPQ.poll()); // 30 (largest)
    }
}
```

---

### 🔥 **3. Blocking Queue (Thread-Safe)**

```java
import java.util.concurrent.*;

public class BlockingQueueExample {
    public static void main(String[] args) {
        BlockingQueue<Integer> queue = new ArrayBlockingQueue<>(10);
        
        // Producer thread
        new Thread(() -> {
            try {
                for (int i = 0; i < 100; i++) {
                    queue.put(i); // Blocks if queue is full
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }).start();
        
        // Consumer thread
        new Thread(() -> {
            try {
                while (true) {
                    int value = queue.take(); // Blocks if queue is empty
                    System.out.println("Consumed: " + value);
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        }).start();
    }
}
```

---

## 🎯 Common Queue Patterns

### 🔥 **Pattern 1: BFS (Level Order Traversal)**

```java
// Tree level-order traversal
public List<List<Integer>> levelOrder(TreeNode root) {
    List<List<Integer>> result = new ArrayList<>();
    if (root == null) return result;
    
    Queue<TreeNode> queue = new LinkedList<>();
    queue.offer(root);
    
    while (!queue.isEmpty()) {
        int levelSize = queue.size();
        List<Integer> currentLevel = new ArrayList<>();
        
        for (int i = 0; i < levelSize; i++) {
            TreeNode node = queue.poll();
            currentLevel.add(node.val);
            
            if (node.left != null) queue.offer(node.left);
            if (node.right != null) queue.offer(node.right);
        }
        
        result.add(currentLevel);
    }
    
    return result;
}
```

---

### 🔥 **Pattern 2: Sliding Window Maximum**

```java
public int[] maxSlidingWindow(int[] nums, int k) {
    Deque<Integer> deque = new ArrayDeque<>();
    int[] result = new int[nums.length - k + 1];
    int ri = 0;
    
    for (int i = 0; i < nums.length; i++) {
        // Remove indices outside window
        while (!deque.isEmpty() && deque.peek() < i - k + 1) {
            deque.poll();
        }
        
        // Remove smaller elements (they won't be max)
        while (!deque.isEmpty() && nums[deque.peekLast()] < nums[i]) {
            deque.pollLast();
        }
        
        deque.offer(i);
        
        // Add to result if window is complete
        if (i >= k - 1) {
            result[ri++] = nums[deque.peek()];
        }
    }
    
    return result;
}

// Example: nums = [1,3,-1,-3,5,3,6,7], k = 3
// Result: [3, 3, 5, 5, 6, 7]
```

---

### 🔥 **Pattern 3: Rotting Oranges (Multi-Source BFS)**

```java
public int orangesRotting(int[][] grid) {
    Queue<int[]> queue = new LinkedList<>();
    int fresh = 0;
    
    // Add all rotten oranges to queue
    for (int i = 0; i < grid.length; i++) {
        for (int j = 0; j < grid[0].length; j++) {
            if (grid[i][j] == 2) {
                queue.offer(new int[]{i, j});
            } else if (grid[i][j] == 1) {
                fresh++;
            }
        }
    }
    
    if (fresh == 0) return 0;
    
    int minutes = 0;
    int[][] dirs = {{-1,0}, {1,0}, {0,-1}, {0,1}};
    
    while (!queue.isEmpty()) {
        int size = queue.size();
        
        for (int i = 0; i < size; i++) {
            int[] cell = queue.poll();
            
            for (int[] dir : dirs) {
                int r = cell[0] + dir[0];
                int c = cell[1] + dir[1];
                
                if (r >= 0 && r < grid.length && 
                    c >= 0 && c < grid[0].length && 
                    grid[r][c] == 1) {
                    grid[r][c] = 2;
                    queue.offer(new int[]{r, c});
                    fresh--;
                }
            }
        }
        
        minutes++;
    }
    
    return fresh == 0 ? minutes - 1 : -1;
}
```

---

### 🔥 **Pattern 4: Task Scheduler**

```java
public int leastInterval(char[] tasks, int n) {
    // Count task frequencies
    int[] freq = new int[26];
    for (char task : tasks) {
        freq[task - 'A']++;
    }
    
    // Max heap by frequency
    PriorityQueue<Integer> pq = new PriorityQueue<>(Collections.reverseOrder());
    for (int f : freq) {
        if (f > 0) pq.offer(f);
    }
    
    int time = 0;
    
    while (!pq.isEmpty()) {
        List<Integer> temp = new ArrayList<>();
        
        // Execute tasks in current cycle
        for (int i = 0; i <= n; i++) {
            if (!pq.isEmpty()) {
                int f = pq.poll();
                if (f > 1) temp.add(f - 1);
            }
            time++;
            
            if (pq.isEmpty() && temp.isEmpty()) break;
        }
        
        // Add back remaining tasks
        pq.addAll(temp);
    }
    
    return time;
}
```

---

## 🏢 Real-World Applications

### 🔷 **Operating System: Process Scheduling**

```java
public class ProcessScheduler {
    private Queue<Process> readyQueue;
    
    public ProcessScheduler() {
        readyQueue = new LinkedList<>();
    }
    
    public void scheduleProcess(Process process) {
        readyQueue.offer(process);
        System.out.println("Process " + process.id + " added to ready queue");
    }
    
    public void executeProcesses(int timeQuantum) {
        while (!readyQueue.isEmpty()) {
            Process current = readyQueue.poll();
            System.out.println("Executing process: " + current.id);
            
            current.execute(timeQuantum);
            
            if (!current.isComplete()) {
                readyQueue.offer(current); // Round-robin
            }
        }
    }
}
```

---

### 🔷 **Web Server: Request Handling**

```java
public class WebServer {
    private BlockingQueue<Request> requestQueue;
    private ExecutorService workers;
    
    public WebServer(int queueSize, int numWorkers) {
        requestQueue = new ArrayBlockingQueue<>(queueSize);
        workers = Executors.newFixedThreadPool(numWorkers);
        
        // Start worker threads
        for (int i = 0; i < numWorkers; i++) {
            workers.submit(() -> {
                while (true) {
                    try {
                        Request request = requestQueue.take();
                        handleRequest(request);
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                        break;
                    }
                }
            });
        }
    }
    
    public void acceptRequest(Request request) throws InterruptedException {
        requestQueue.put(request);
    }
    
    private void handleRequest(Request request) {
        // Process the HTTP request
        System.out.println("Handling request from: " + request.clientIP);
    }
}
```

---

### 🔷 **Messaging System: Message Queue**

```java
public class MessageQueue {
    private Queue<Message> queue;
    private int maxSize;
    
    public MessageQueue(int maxSize) {
        this.queue = new LinkedList<>();
        this.maxSize = maxSize;
    }
    
    public boolean publish(Message message) {
        if (queue.size() >= maxSize) {
            return false; // Queue full
        }
        
        queue.offer(message);
        System.out.println("Published: " + message.content);
        return true;
    }
    
    public Message consume() {
        Message message = queue.poll();
        if (message != null) {
            System.out.println("Consumed: " + message.content);
        }
        return message;
    }
    
    public int size() {
        return queue.size();
    }
}
```

---

### 🔷 **Gaming: Matchmaking Queue**

```java
public class MatchmakingQueue {
    private Deque<Player> queue;
    
    public MatchmakingQueue() {
        queue = new ArrayDeque<>();
    }
    
    public void addPlayer(Player player) {
        queue.offer(player);
        System.out.println(player.name + " joined matchmaking");
        tryCreateMatch();
    }
    
    public void addPriorityPlayer(Player player) {
        queue.offerFirst(player); // VIP gets priority
        System.out.println(player.name + " joined with priority");
        tryCreateMatch();
    }
    
    private void tryCreateMatch() {
        if (queue.size() >= 10) {
            List<Player> match = new ArrayList<>();
            for (int i = 0; i < 10; i++) {
                match.add(queue.poll());
            }
            createGame(match);
        }
    }
    
    private void createGame(List<Player> players) {
        System.out.println("Game created with " + players.size() + " players!");
    }
}
```

---

## 🧩 Practice Problems

### 🟢 **Easy**
1. [Implement Queue using Stacks](https://leetcode.com/problems/implement-queue-using-stacks/) (LeetCode #232)
2. [Number of Recent Calls](https://leetcode.com/problems/number-of-recent-calls/) (LeetCode #933)
3. [Design Circular Queue](https://leetcode.com/problems/design-circular-queue/) (LeetCode #622)

### 🟡 **Medium**
4. [Binary Tree Level Order Traversal](https://leetcode.com/problems/binary-tree-level-order-traversal/) (LeetCode #102)
5. [Rotting Oranges](https://leetcode.com/problems/rotting-oranges/) (LeetCode #994)
6. [Task Scheduler](https://leetcode.com/problems/task-scheduler/) (LeetCode #621)
7. [Sliding Window Maximum](https://leetcode.com/problems/sliding-window-maximum/) (LeetCode #239)
8. [Perfect Squares](https://leetcode.com/problems/perfect-squares/) (LeetCode #279)

### 🔴 **Hard**
9. [Sliding Window Median](https://leetcode.com/problems/sliding-window-median/) (LeetCode #480)

---

## ⚡ Queue vs Stack Comparison

| Feature | Queue | Stack |
|---------|-------|-------|
| **Order** | FIFO | LIFO |
| **Add** | Rear (enqueue) | Top (push) |
| **Remove** | Front (dequeue) | Top (pop) |
| **Use Case** | BFS, scheduling | DFS, undo/redo |
| **Real-World** | Line at store | Stack of plates |

---

## 💡 Interview Tips

### ✅ **Do's**
1. Clarify if circular queue is needed
2. Ask about capacity constraints
3. Handle empty/full conditions
4. Consider thread safety for concurrent scenarios

### ❌ **Common Mistakes**
1. Forgetting to handle circular wrap-around
2. Not checking isEmpty() before dequeue
3. Using wrong data structure (queue vs stack)
4. Inefficient BFS implementation

---

## 🎯 Key Takeaways

✅ Queue = **FIFO** (First In, First Out)  
✅ All operations: **O(1)** time complexity  
✅ Perfect for **BFS, scheduling, buffering**  
✅ Deque provides **double-ended** flexibility  
✅ BlockingQueue for **concurrent** scenarios  

---

## 🚀 What's Next?

➡️ **[Hash Tables: Speed Demon](../HashTables/01_HashTable_Fundamentals.md)**  
➡️ **[Trees: Hierarchical Power](../Trees/01_BinaryTree_Fundamentals.md)**

---

**🎮 Achievement Unlocked: Queue Mastery!** 🏆

*Remember: "In queues, patience pays off—first come, first served!"* 🚀

