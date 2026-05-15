# 🏔️ Heaps: The Priority Powerhouse! 🚀

---

## 🎯 Chapter 1: Heap Fundamentals

> **"Heaps: Where the biggest (or smallest) always rises to the top!"**

---

## 🤔 What is a Heap?

A **Heap** is a **complete binary tree** that satisfies the **heap property**:
- **Max Heap**: Parent ≥ Children
- **Min Heap**: Parent ≤ Children

### 🏗️ **Visual Representation**

```
Max Heap:                Min Heap:
      90                      10
     /  \                    /  \
   80    70                20    30
  / \    /                / \    /
 50 60  40              40 50  60

✅ Max Heap: 90 ≥ 80, 90 ≥ 70, 80 ≥ 50, 80 ≥ 60...
✅ Min Heap: 10 ≤ 20, 10 ≤ 30, 20 ≤ 40, 20 ≤ 50...
```

---

## 💻 Array Representation

```
Max Heap:     [90, 80, 70, 50, 60, 40]
Indices:       0   1   2   3   4   5

Tree View:
         90 (0)
        /  \
      80(1) 70(2)
     / \    /
   50(3)60(4)40(5)

✅ Parent of i: (i-1)/2
✅ Left child of i: 2*i + 1
✅ Right child of i: 2*i + 2
```

---

## 💻 Java Implementation

### 🔧 **Min Heap Implementation**

```java
public class MinHeap {
    private int[] heap;
    private int size;
    private int capacity;
    
    public MinHeap(int capacity) {
        this.capacity = capacity;
        this.size = 0;
        heap = new int[capacity];
    }
    
    private int parent(int i) { return (i - 1) / 2; }
    private int leftChild(int i) { return 2 * i + 1; }
    private int rightChild(int i) { return 2 * i + 2; }
    
    private void swap(int i, int j) {
        int temp = heap[i];
        heap[i] = heap[j];
        heap[j] = temp;
    }
    
    // Insert - O(log n)
    public void insert(int value) {
        if (size == capacity) {
            throw new IllegalStateException("Heap is full");
        }
        
        heap[size] = value;
        int current = size;
        size++;
        
        // Bubble up
        while (current > 0 && heap[current] < heap[parent(current)]) {
            swap(current, parent(current));
            current = parent(current);
        }
    }
    
    // Extract Min - O(log n)
    public int extractMin() {
        if (size == 0) {
            throw new IllegalStateException("Heap is empty");
        }
        
        int min = heap[0];
        heap[0] = heap[size - 1];
        size--;
        
        heapifyDown(0);
        
        return min;
    }
    
    // Heapify Down
    private void heapifyDown(int i) {
        int smallest = i;
        int left = leftChild(i);
        int right = rightChild(i);
        
        if (left < size && heap[left] < heap[smallest]) {
            smallest = left;
        }
        
        if (right < size && heap[right] < heap[smallest]) {
            smallest = right;
        }
        
        if (smallest != i) {
            swap(i, smallest);
            heapifyDown(smallest);
        }
    }
    
    // Peek - O(1)
    public int peek() {
        if (size == 0) {
            throw new IllegalStateException("Heap is empty");
        }
        return heap[0];
    }
    
    public int size() {
        return size;
    }
    
    public boolean isEmpty() {
        return size == 0;
    }
}
```

---

## 📦 **Java PriorityQueue (Built-in Heap)**

```java
import java.util.*;

public class HeapExamples {
    public static void main(String[] args) {
        // Min Heap (default)
        PriorityQueue<Integer> minHeap = new PriorityQueue<>();
        
        minHeap.offer(30);
        minHeap.offer(10);
        minHeap.offer(20);
        
        System.out.println(minHeap.poll()); // 10
        System.out.println(minHeap.poll()); // 20
        
        // Max Heap (reverse order)
        PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Collections.reverseOrder());
        
        maxHeap.offer(30);
        maxHeap.offer(10);
        maxHeap.offer(20);
        
        System.out.println(maxHeap.poll()); // 30
        System.out.println(maxHeap.poll()); // 20
        
        // Custom comparator
        PriorityQueue<int[]> pq = new PriorityQueue<>((a, b) -> a[1] - b[1]);
        pq.offer(new int[]{1, 5});
        pq.offer(new int[]{2, 3});
        pq.offer(new int[]{3, 7});
        
        System.out.println(pq.poll()[1]); // 3
    }
}
```

---

## 🎯 Common Heap Patterns

### 🔥 **Pattern 1: Top K Elements**

```java
public int[] topKFrequent(int[] nums, int k) {
    // Count frequencies
    Map<Integer, Integer> freqMap = new HashMap<>();
    for (int num : nums) {
        freqMap.put(num, freqMap.getOrDefault(num, 0) + 1);
    }
    
    // Min heap of size k
    PriorityQueue<Integer> heap = new PriorityQueue<>(
        (a, b) -> freqMap.get(a) - freqMap.get(b)
    );
    
    for (int num : freqMap.keySet()) {
        heap.offer(num);
        if (heap.size() > k) {
            heap.poll();
        }
    }
    
    int[] result = new int[k];
    for (int i = 0; i < k; i++) {
        result[i] = heap.poll();
    }
    
    return result;
}
```

---

### 🔥 **Pattern 2: Kth Largest Element**

```java
public int findKthLargest(int[] nums, int k) {
    PriorityQueue<Integer> minHeap = new PriorityQueue<>();
    
    for (int num : nums) {
        minHeap.offer(num);
        if (minHeap.size() > k) {
            minHeap.poll(); // Remove smallest
        }
    }
    
    return minHeap.peek();
}
```

---

### 🔥 **Pattern 3: Merge K Sorted Lists**

```java
public ListNode mergeKLists(ListNode[] lists) {
    PriorityQueue<ListNode> heap = new PriorityQueue<>(
        (a, b) -> a.val - b.val
    );
    
    // Add first node of each list
    for (ListNode node : lists) {
        if (node != null) {
            heap.offer(node);
        }
    }
    
    ListNode dummy = new ListNode(0);
    ListNode current = dummy;
    
    while (!heap.isEmpty()) {
        ListNode smallest = heap.poll();
        current.next = smallest;
        current = current.next;
        
        if (smallest.next != null) {
            heap.offer(smallest.next);
        }
    }
    
    return dummy.next;
}
```

---

### 🔥 **Pattern 4: Median of Data Stream**

```java
class MedianFinder {
    private PriorityQueue<Integer> maxHeap; // Left half
    private PriorityQueue<Integer> minHeap; // Right half
    
    public MedianFinder() {
        maxHeap = new PriorityQueue<>(Collections.reverseOrder());
        minHeap = new PriorityQueue<>();
    }
    
    public void addNum(int num) {
        if (maxHeap.isEmpty() || num <= maxHeap.peek()) {
            maxHeap.offer(num);
        } else {
            minHeap.offer(num);
        }
        
        // Balance heaps
        if (maxHeap.size() > minHeap.size() + 1) {
            minHeap.offer(maxHeap.poll());
        } else if (minHeap.size() > maxHeap.size()) {
            maxHeap.offer(minHeap.poll());
        }
    }
    
    public double findMedian() {
        if (maxHeap.size() == minHeap.size()) {
            return (maxHeap.peek() + minHeap.peek()) / 2.0;
        }
        return maxHeap.peek();
    }
}
```

---

## 🏢 Real-World Applications

### 🔷 **Task Scheduling**
```java
// Process highest priority tasks first
PriorityQueue<Task> taskQueue = new PriorityQueue<>(
    (a, b) -> b.priority - a.priority
);
```

### 🔷 **Dijkstra's Algorithm**
```java
// Find shortest path using min heap
PriorityQueue<Node> pq = new PriorityQueue<>(
    (a, b) -> a.distance - b.distance
);
```

### 🔷 **Huffman Coding**
```java
// Build optimal compression tree
PriorityQueue<HuffmanNode> minHeap = new PriorityQueue<>(
    (a, b) -> a.frequency - b.frequency
);
```

---

## ⚡ Complexity Analysis

| Operation | Time Complexity |
|-----------|----------------|
| **Insert** | O(log n) |
| **Extract Min/Max** | O(log n) |
| **Peek** | O(1) |
| **Build Heap** | O(n) |
| **Heapify** | O(log n) |

**Space Complexity**: O(n)

---

## 🧩 Practice Problems

### 🟢 **Easy**
1. Kth Largest Element (LeetCode #215)
2. Last Stone Weight (LeetCode #1046)

### 🟡 **Medium**
3. Top K Frequent Elements (LeetCode #347)
4. Find Median from Data Stream (LeetCode #295)
5. Kth Smallest Element in Sorted Matrix (LeetCode #378)

### 🔴 **Hard**
6. Merge k Sorted Lists (LeetCode #23)
7. IPO (LeetCode #502)

---

## 🎯 Key Takeaways

✅ Heaps provide **O(1) access** to min/max  
✅ **Insert/Delete**: O(log n)  
✅ Perfect for **priority queues**  
✅ Use for **Top K problems**  

---

**🎮 Achievement Unlocked: Heap Mastery!** 🏆

