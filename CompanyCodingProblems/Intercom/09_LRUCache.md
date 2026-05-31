# 🟣 Intercom | Problem 09 — LRU Cache

> **Source:** LeetCode 146 · Classic OOP/class design problem
> **Difficulty:** 🟡 Medium
> **Topics:** HashMap · Doubly Linked List · Class Design · Cache · O(1) Operations

---

## 🔥 Ultimate Coding Interview Strategy

---

### **1️⃣ Understanding the Problem Clearly**

**Paraphrasing the Problem:**
> Design a data structure that follows LRU (Least Recently Used) eviction policy:
> 1. `LRUCache(capacity)` — Initialize with max size
> 2. `int get(key)` — Return value if exists (marks as recently used), else -1
> 3. `void put(key, value)` — Insert/update key-value. If at capacity, evict least recently used

**Why Intercom Would Ask This:**
- Tests class design with dual data structures (HashMap + LinkedList)
- Caching is fundamental to their product (conversation caching, user sessions)
- Same "class with 2-3 methods + internal state management" pattern as their load balancer

**Key Requirements:**
- Both `get` and `put` must be **O(1)** time
- "Recently used" means any access (get or put) makes it most recent
- On eviction, remove the LEAST recently accessed item

---

### **2️⃣ Breaking Down the Solution**

**Data Structure Choice:**
```
HashMap<key, Node>  — O(1) lookup by key
Doubly Linked List  — O(1) removal and insertion at head/tail
  Head = Most Recently Used
  Tail = Least Recently Used

Combined: O(1) get + O(1) put + O(1) eviction
```

**Operations:**
- `get(key)`: Find in map → move node to head → return value
- `put(key, value)`:
  - If exists: update value, move to head
  - If new + under capacity: insert at head, add to map
  - If new + at capacity: remove tail (LRU) from map & list, then insert at head

---

### **3️⃣ Writing Clean & Efficient Code**

#### ✅ Java Solution

```java
import java.util.HashMap;
import java.util.Map;

/**
 * LRU Cache with O(1) get and put operations.
 * 
 * Uses: HashMap (O(1) lookup) + Doubly Linked List (O(1) move/remove/insert).
 * Head = Most Recently Used, Tail = Least Recently Used.
 */
public class LRUCache {

    // Doubly linked list node
    private static class Node {
        int key, value;
        Node prev, next;

        Node(int key, int value) {
            this.key = key;
            this.value = value;
        }
    }

    private final int capacity;
    private final Map<Integer, Node> cache;  // key → node
    private final Node head;  // Dummy head (MRU side)
    private final Node tail;  // Dummy tail (LRU side)

    public LRUCache(int capacity) {
        this.capacity = capacity;
        this.cache = new HashMap<>();

        // Initialize dummy head/tail (sentinel nodes)
        head = new Node(0, 0);
        tail = new Node(0, 0);
        head.next = tail;
        tail.prev = head;
    }

    /**
     * Get value by key. Returns -1 if not found.
     * Marks the key as most recently used.
     */
    public int get(int key) {
        Node node = cache.get(key);
        if (node == null) return -1;

        // Move to head (most recently used)
        removeNode(node);
        addToHead(node);
        return node.value;
    }

    /**
     * Insert or update key-value pair.
     * If at capacity, evicts the least recently used entry.
     */
    public void put(int key, int value) {
        Node existing = cache.get(key);

        if (existing != null) {
            // Update existing: change value, move to head
            existing.value = value;
            removeNode(existing);
            addToHead(existing);
        } else {
            // Evict if at capacity
            if (cache.size() >= capacity) {
                Node lru = tail.prev;  // Least recently used = just before dummy tail
                cache.remove(lru.key);
                removeNode(lru);
            }

            // Insert new node at head
            Node newNode = new Node(key, value);
            cache.put(key, newNode);
            addToHead(newNode);
        }
    }

    // ---- Helper methods: O(1) doubly linked list operations ----

    /** Remove a node from its current position in the list. */
    private void removeNode(Node node) {
        node.prev.next = node.next;
        node.next.prev = node.prev;
    }

    /** Insert a node right after head (marks it as most recently used). */
    private void addToHead(Node node) {
        node.next = head.next;
        node.prev = head;
        head.next.prev = node;
        head.next = node;
    }
}
```

#### ✅ Python Solution (Using OrderedDict — Interview Shortcut)

```python
from collections import OrderedDict


class LRUCache:
    """
    LRU Cache using Python's OrderedDict (maintains insertion order).
    
    get: O(1) — lookup + move to end
    put: O(1) — insert/update + optional eviction from front
    """

    def __init__(self, capacity: int):
        self._capacity = capacity
        self._cache = OrderedDict()  # key → value, ordered by access time

    def get(self, key: int) -> int:
        if key not in self._cache:
            return -1
        # Move to end (most recently used)
        self._cache.move_to_end(key)
        return self._cache[key]

    def put(self, key: int, value: int) -> None:
        if key in self._cache:
            # Update and mark as recently used
            self._cache.move_to_end(key)
            self._cache[key] = value
        else:
            # Evict LRU if at capacity
            if len(self._cache) >= self._capacity:
                self._cache.popitem(last=False)  # Remove first (oldest)
            self._cache[key] = value
```

#### ✅ Python Solution (From Scratch — No OrderedDict)

```python
class Node:
    __slots__ = ('key', 'value', 'prev', 'next')
    def __init__(self, key: int = 0, value: int = 0):
        self.key = key
        self.value = value
        self.prev = None
        self.next = None


class LRUCacheManual:
    """
    LRU Cache from scratch: HashMap + Doubly Linked List.
    Head = MRU, Tail = LRU.
    """

    def __init__(self, capacity: int):
        self._capacity = capacity
        self._cache: dict[int, Node] = {}
        # Sentinel nodes
        self._head = Node()
        self._tail = Node()
        self._head.next = self._tail
        self._tail.prev = self._head

    def get(self, key: int) -> int:
        node = self._cache.get(key)
        if not node:
            return -1
        self._remove(node)
        self._add_to_head(node)
        return node.value

    def put(self, key: int, value: int) -> None:
        node = self._cache.get(key)
        if node:
            node.value = value
            self._remove(node)
            self._add_to_head(node)
        else:
            if len(self._cache) >= self._capacity:
                lru = self._tail.prev
                del self._cache[lru.key]
                self._remove(lru)
            new_node = Node(key, value)
            self._cache[key] = new_node
            self._add_to_head(new_node)

    def _remove(self, node: Node) -> None:
        node.prev.next = node.next
        node.next.prev = node.prev

    def _add_to_head(self, node: Node) -> None:
        node.next = self._head.next
        node.prev = self._head
        self._head.next.prev = node
        self._head.next = node
```

---

### **4️⃣ Explaining the Code**

**Visual:**
```
Dummy Head ↔ [MRU] ↔ [2nd MRU] ↔ ... ↔ [LRU] ↔ Dummy Tail

get(key):   find node in map → unlink from list → link at head → return value
put(key,v): if exists → update value, move to head
            if new + full → remove tail.prev from map & list, then add new at head
            if new + space → add at head, add to map
```

**Why sentinel (dummy) nodes?**
- Eliminates null checks for head/tail edge cases
- `removeNode` and `addToHead` always work the same way (no special cases)

**Complexity:**

| Operation | Time | Space |
|-----------|------|-------|
| `get` | O(1) | — |
| `put` | O(1) | — |
| Overall | — | O(capacity) |

---

### **5️⃣ Edge Cases & Testing**

| Scenario | Operations | Expected | Notes |
|----------|-----------|----------|-------|
| Basic get/put | put(1,1), get(1) | 1 | Normal |
| Key not found | get(999) | -1 | Missing key |
| Update existing | put(1,1), put(1,2), get(1) | 2 | Value updated |
| Eviction | capacity=2, put(1,1), put(2,2), put(3,3), get(1) | -1 | Key 1 was evicted |
| Access refreshes | capacity=2, put(1,1), put(2,2), get(1), put(3,3), get(2) | -1 | Key 2 evicted (1 was refreshed by get) |
| Capacity 1 | put(1,1), put(2,2), get(1) | -1 | Only room for 1 |
| Put refreshes | capacity=2, put(1,1), put(2,2), put(1,5), put(3,3), get(2) | -1 | put(1,5) refreshed key 1; key 2 is LRU |

---

### **6️⃣ Common Mistakes**

❌ **Mistake 1: Forgetting to store key in the Node**
> When evicting from tail, you need the key to remove from HashMap. Without `node.key`, you can't delete from the map.

❌ **Mistake 2: Not updating value on put() for existing key**
> `put(1, 10)` then `put(1, 20)` — must update to 20, not keep 10.

❌ **Mistake 3: Not moving to head on get()**
> `get` is an access — it must refresh the item's position to MRU.

❌ **Mistake 4: Using single linked list**
> Removal from middle of singly linked list is O(n). Doubly linked list gives O(1) removal.

---

## 📎 Related Problems

| Problem | Connection |
|---------|-----------|
| [LC 460 - LFU Cache](https://leetcode.com/problems/lfu-cache/) | Harder version with frequency tracking |
| [LC 380 - Insert Delete GetRandom O(1)](https://leetcode.com/problems/insert-delete-getrandom-o1/) | Similar dual data structure design |
| [LC 706 - Design HashMap](https://leetcode.com/problems/design-hashmap/) | Simpler class design |
| [Intercom P01](./01_FairConversationAssignment.md) | Same class-with-state pattern |
