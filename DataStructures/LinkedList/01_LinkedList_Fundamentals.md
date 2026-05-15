# 🔗 LinkedList: The Flexible Chain of Data! 🚀

---

## 🎯 Chapter 1: LinkedList Fundamentals

> **"Think of a LinkedList as a treasure hunt: each clue (node) points to the next clue, leading you to the treasure (data)!"**

---

## 🤔 What is a LinkedList?

A **LinkedList** is a linear data structure where elements (called nodes) are connected using **pointers/references**. Unlike arrays, elements are **NOT stored in contiguous memory locations**.

### 🏗️ **Visual Representation**

```
Singly LinkedList:
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│ Data: 10 │───▶│ Data: 20 │───▶│ Data: 30 │───▶│ Data: 40 │───▶ null
│ Next: ●  │    │ Next: ●  │    │ Next: ●  │    │ Next: ●  │
└──────────┘    └──────────┘    └──────────┘    └──────────┘
   HEAD                                             TAIL

Doubly LinkedList:
       ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
null ◀─│ Prev: ●      │◀──▶│ Prev: ●      │◀──▶│ Prev: ●      │───▶ null
       │ Data: 10     │    │ Data: 20     │    │ Data: 30     │
       │ Next: ●      │───▶│ Next: ●      │───▶│ Next: ●      │
       └──────────────┘    └──────────────┘    └──────────────┘
           HEAD                                      TAIL
```

---

## 🌍 Real-World Analogies

### 🚂 **Train Coaches Analogy**

```
🚂 [Engine] → [Coach A] → [Coach B] → [Coach C] → [Last Coach]
   HEAD                                             TAIL

✅ Each coach connected to next
✅ Can add/remove coaches easily
✅ Must traverse from engine to reach specific coach
✅ Can't directly jump to "Coach C" (unlike array seats)
```

### 📿 **Chain/Necklace Analogy**

```
[Bead 1] ⟷ [Bead 2] ⟷ [Bead 3] ⟷ [Bead 4]

✅ Each bead connected to next (and previous in doubly linked)
✅ Easy to add/remove beads
✅ Breaking chain breaks the connection
```

---

## 💻 Java Implementation

### 🔧 **Node Structure**

```java
// Singly LinkedList Node
class Node {
    int data;        // Data stored in node
    Node next;       // Reference to next node
    
    public Node(int data) {
        this.data = data;
        this.next = null;
    }
}

// Doubly LinkedList Node
class DoublyNode {
    int data;
    DoublyNode prev;  // Reference to previous node
    DoublyNode next;  // Reference to next node
    
    public DoublyNode(int data) {
        this.data = data;
        this.prev = null;
        this.next = null;
    }
}
```

---

### 📊 **Complete Singly LinkedList Implementation**

```java
public class SinglyLinkedList {
    private Node head;
    private Node tail;
    private int size;
    
    // Node class
    private static class Node {
        int data;
        Node next;
        
        Node(int data) {
            this.data = data;
            this.next = null;
        }
    }
    
    public SinglyLinkedList() {
        this.head = null;
        this.tail = null;
        this.size = 0;
    }
    
    // Insert at beginning - O(1)
    public void insertAtHead(int data) {
        Node newNode = new Node(data);
        
        if (head == null) {
            head = tail = newNode;
        } else {
            newNode.next = head;
            head = newNode;
        }
        size++;
    }
    
    // Insert at end - O(1) with tail pointer, O(n) without
    public void insertAtTail(int data) {
        Node newNode = new Node(data);
        
        if (head == null) {
            head = tail = newNode;
        } else {
            tail.next = newNode;
            tail = newNode;
        }
        size++;
    }
    
    // Insert at specific position - O(n)
    public void insertAtPosition(int data, int position) {
        if (position < 0 || position > size) {
            throw new IndexOutOfBoundsException("Invalid position");
        }
        
        if (position == 0) {
            insertAtHead(data);
            return;
        }
        
        if (position == size) {
            insertAtTail(data);
            return;
        }
        
        Node newNode = new Node(data);
        Node current = head;
        
        // Traverse to position - 1
        for (int i = 0; i < position - 1; i++) {
            current = current.next;
        }
        
        newNode.next = current.next;
        current.next = newNode;
        size++;
    }
    
    // Delete from beginning - O(1)
    public int deleteFromHead() {
        if (head == null) {
            throw new RuntimeException("List is empty");
        }
        
        int data = head.data;
        head = head.next;
        
        if (head == null) {
            tail = null;
        }
        
        size--;
        return data;
    }
    
    // Delete from end - O(n) for singly linked list
    public int deleteFromTail() {
        if (head == null) {
            throw new RuntimeException("List is empty");
        }
        
        if (head == tail) {
            int data = head.data;
            head = tail = null;
            size--;
            return data;
        }
        
        Node current = head;
        while (current.next != tail) {
            current = current.next;
        }
        
        int data = tail.data;
        current.next = null;
        tail = current;
        size--;
        return data;
    }
    
    // Delete at specific position - O(n)
    public int deleteAtPosition(int position) {
        if (position < 0 || position >= size) {
            throw new IndexOutOfBoundsException("Invalid position");
        }
        
        if (position == 0) {
            return deleteFromHead();
        }
        
        Node current = head;
        for (int i = 0; i < position - 1; i++) {
            current = current.next;
        }
        
        int data = current.next.data;
        current.next = current.next.next;
        
        if (current.next == null) {
            tail = current;
        }
        
        size--;
        return data;
    }
    
    // Search - O(n)
    public boolean contains(int data) {
        Node current = head;
        while (current != null) {
            if (current.data == data) {
                return true;
            }
            current = current.next;
        }
        return false;
    }
    
    // Get element at position - O(n)
    public int get(int position) {
        if (position < 0 || position >= size) {
            throw new IndexOutOfBoundsException("Invalid position");
        }
        
        Node current = head;
        for (int i = 0; i < position; i++) {
            current = current.next;
        }
        return current.data;
    }
    
    // Display - O(n)
    public void display() {
        Node current = head;
        System.out.print("HEAD → ");
        while (current != null) {
            System.out.print(current.data + " → ");
            current = current.next;
        }
        System.out.println("null");
    }
    
    // Get size - O(1)
    public int size() {
        return size;
    }
    
    // Check if empty - O(1)
    public boolean isEmpty() {
        return head == null;
    }
    
    // Reverse the list - O(n)
    public void reverse() {
        if (head == null || head.next == null) {
            return;
        }
        
        Node prev = null;
        Node current = head;
        tail = head; // Old head becomes new tail
        
        while (current != null) {
            Node next = current.next;
            current.next = prev;
            prev = current;
            current = next;
        }
        
        head = prev;
    }
}
```

---

## ⚡ Time & Space Complexity

| Operation | Singly LinkedList | Doubly LinkedList | Array | ArrayList |
|-----------|------------------|-------------------|-------|-----------|
| **Access** | O(n) | O(n) | O(1) | O(1) |
| **Search** | O(n) | O(n) | O(n) | O(n) |
| **Insert at Head** | O(1) | O(1) | O(n) | O(n) |
| **Insert at Tail** | O(1)* | O(1) | O(1)** | O(1)** |
| **Insert at Middle** | O(n) | O(n) | O(n) | O(n) |
| **Delete from Head** | O(1) | O(1) | O(n) | O(n) |
| **Delete from Tail** | O(n) | O(1) | O(1) | O(1) |
| **Delete from Middle** | O(n) | O(n) | O(n) | O(n) |

\* With tail pointer  
\** Amortized (may need resizing)

**Space Complexity**: O(n) for n nodes (extra space for pointers)

---

## 🎯 Common LinkedList Patterns

### 🔥 **Pattern 1: Fast & Slow Pointers (Tortoise and Hare)**

**Use Case**: Find middle of linked list, detect cycle

```java
// Find middle element
public Node findMiddle() {
    Node slow = head;
    Node fast = head;
    
    while (fast != null && fast.next != null) {
        slow = slow.next;        // Move 1 step
        fast = fast.next.next;   // Move 2 steps
    }
    
    return slow; // Slow is at middle
}

// Detect cycle
public boolean hasCycle() {
    if (head == null) return false;
    
    Node slow = head;
    Node fast = head;
    
    while (fast != null && fast.next != null) {
        slow = slow.next;
        fast = fast.next.next;
        
        if (slow == fast) {
            return true; // Cycle detected
        }
    }
    
    return false;
}
```

**Visual**:
```
Normal List:
Slow: 1 → 2 → 3 → 4 → 5 → null
Fast: 1 →   → 3 →   → 5 → null
       ↑           ↑
     (slow)     (fast)

List with Cycle:
1 → 2 → 3 → 4 → 5
         ↑         ↓
         8 ← 7 ← 6
         
After some iterations: slow == fast (cycle detected!)
```

---

### 🔥 **Pattern 2: Reversing LinkedList**

```java
// Iterative approach - O(n) time, O(1) space
public Node reverse(Node head) {
    Node prev = null;
    Node current = head;
    
    while (current != null) {
        Node next = current.next;  // Save next
        current.next = prev;        // Reverse link
        prev = current;             // Move prev
        current = next;             // Move current
    }
    
    return prev; // New head
}

// Recursive approach - O(n) time, O(n) space
public Node reverseRecursive(Node head) {
    if (head == null || head.next == null) {
        return head;
    }
    
    Node newHead = reverseRecursive(head.next);
    head.next.next = head;
    head.next = null;
    
    return newHead;
}
```

**Step-by-step Visual**:
```
Original: 1 → 2 → 3 → 4 → null

Step 1: null ← 1   2 → 3 → 4 → null
        prev  curr next

Step 2: null ← 1 ← 2   3 → 4 → null
               prev  curr next

Step 3: null ← 1 ← 2 ← 3   4 → null
                      prev curr next

Step 4: null ← 1 ← 2 ← 3 ← 4
                          prev curr(null)

Result: 4 → 3 → 2 → 1 → null
```

---

### 🔥 **Pattern 3: Merge Two Sorted Lists**

```java
public Node mergeTwoLists(Node l1, Node l2) {
    Node dummy = new Node(0);
    Node current = dummy;
    
    while (l1 != null && l2 != null) {
        if (l1.data <= l2.data) {
            current.next = l1;
            l1 = l1.next;
        } else {
            current.next = l2;
            l2 = l2.next;
        }
        current = current.next;
    }
    
    // Attach remaining nodes
    current.next = (l1 != null) ? l1 : l2;
    
    return dummy.next;
}
```

---

### 🔥 **Pattern 4: Remove Nth Node From End**

```java
public Node removeNthFromEnd(Node head, int n) {
    Node dummy = new Node(0);
    dummy.next = head;
    Node first = dummy;
    Node second = dummy;
    
    // Move first pointer n+1 steps ahead
    for (int i = 0; i <= n; i++) {
        first = first.next;
    }
    
    // Move both until first reaches end
    while (first != null) {
        first = first.next;
        second = second.next;
    }
    
    // Remove nth node
    second.next = second.next.next;
    
    return dummy.next;
}
```

---

## 🏢 Real-World Industry Applications

### 🔷 **Operating Systems: Process Scheduling**

```java
// Round-robin scheduler using circular linked list
public class ProcessScheduler {
    private class Process {
        String name;
        int timeQuantum;
        Process next;
    }
    
    private Process current;
    
    public void schedule() {
        while (current != null) {
            execute(current);
            current = current.next; // Move to next process
        }
    }
}
```

### 🔷 **Music Player: Playlist Management**

```java
public class Playlist {
    private Song head;
    private Song current;
    
    public void addSong(Song song) {
        if (head == null) {
            head = song;
            song.next = head; // Circular for repeat all
        } else {
            Song temp = head;
            while (temp.next != head) {
                temp = temp.next;
            }
            temp.next = song;
            song.next = head;
        }
    }
    
    public Song nextSong() {
        current = (current == null) ? head : current.next;
        return current;
    }
    
    public Song previousSong() {
        // Requires doubly linked list for efficient implementation
    }
}
```

### 🔷 **Browser: Navigation History**

```java
public class BrowserHistory {
    private Page currentPage;
    
    private class Page {
        String url;
        Page prev;
        Page next;
    }
    
    public void visit(String url) {
        Page newPage = new Page(url);
        if (currentPage != null) {
            currentPage.next = newPage;
            newPage.prev = currentPage;
        }
        currentPage = newPage;
    }
    
    public String back() {
        if (currentPage != null && currentPage.prev != null) {
            currentPage = currentPage.prev;
            return currentPage.url;
        }
        return null;
    }
    
    public String forward() {
        if (currentPage != null && currentPage.next != null) {
            currentPage = currentPage.next;
            return currentPage.url;
        }
        return null;
    }
}
```

### 🔷 **Image Viewer: Undo/Redo Operations**

```java
public class ImageEditor {
    private Edit head;
    private Edit current;
    
    private class Edit {
        Image state;
        Edit prev;
        Edit next;
    }
    
    public void applyEdit(Image image) {
        Edit edit = new Edit(image);
        if (current != null) {
            current.next = edit;
            edit.prev = current;
        }
        current = edit;
        if (head == null) head = edit;
    }
    
    public Image undo() {
        if (current != null && current.prev != null) {
            current = current.prev;
            return current.state;
        }
        return null;
    }
    
    public Image redo() {
        if (current != null && current.next != null) {
            current = current.next;
            return current.state;
        }
        return null;
    }
}
```

---

## 🧩 Practice Problems

### 🟢 **Easy**

1. **Reverse Linked List** (LeetCode #206)
2. **Merge Two Sorted Lists** (LeetCode #21)
3. **Delete Node in a Linked List** (LeetCode #237)
4. **Linked List Cycle** (LeetCode #141)
5. **Palindrome Linked List** (LeetCode #234)

### 🟡 **Medium**

6. **Add Two Numbers** (LeetCode #2)
7. **Remove Nth Node From End** (LeetCode #19)
8. **Reorder List** (LeetCode #143)
9. **Linked List Cycle II** (LeetCode #142) - Find cycle start
10. **Copy List with Random Pointer** (LeetCode #138)

### 🔴 **Hard**

11. **Reverse Nodes in k-Group** (LeetCode #25)
12. **Merge k Sorted Lists** (LeetCode #23)

---

## 💡 Interview Tips

### ✅ **Do's**

1. **Always handle null/empty list**
2. **Use dummy node for edge cases**
3. **Draw diagrams during interview**
4. **Test with single node list**

### ❌ **Common Mistakes**

1. **Losing reference to head**
2. **Not handling cycle detection**
3. **Forgetting to update tail pointer**
4. **Memory leaks (in languages like C++)**

---

## 🎯 Key Takeaways

✅ LinkedList allows **O(1) insertion/deletion** at head  
✅ **No random access** (must traverse from head)  
✅ **Extra memory** for storing pointers  
✅ **Fast & Slow pointers** pattern is crucial  
✅ Perfect for **dynamic size** and **frequent insertions**  

---

## 🚀 What's Next?

➡️ **[Stacks: LIFO Magic](../StacksAndQueues/01_Stacks_Fundamentals.md)**  
➡️ **[Queues: FIFO Power](../StacksAndQueues/02_Queues_Fundamentals.md)**  

**🎮 Achievement Unlocked: LinkedList Mastery!** 🏆

