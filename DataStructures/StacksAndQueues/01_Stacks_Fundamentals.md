# 📚 Stacks: The LIFO Powerhouse! 🚀

---

## 🎯 Chapter 1: Stack Fundamentals

> **"Think of a stack like a stack of plates: you can only add or remove from the top!"**

---

## 🤔 What is a Stack?

A **Stack** is a **Last-In-First-Out (LIFO)** data structure where elements are added and removed from the same end (top).

### 🏗️ **Visual Representation**

```
Stack Operations:
                    ↓ PUSH(40)
┌─────┐            ┌─────┐
│  40 │ ← TOP      │  40 │ ← TOP
├─────┤            ├─────┤
│  30 │            │  30 │
├─────┤            ├─────┤
│  20 │            │  20 │
├─────┤            ├─────┤
│  10 │            │  10 │
└─────┘            └─────┘
                    ↑ POP() returns 40
```

---

## 🌍 Real-World Analogies

### 📚 **Stack of Books**
```
     [Book 4] ← Can only take top book
     [Book 3]
     [Book 2]
     [Book 1]
  ──────────────
```

### ⏪ **Undo/Redo in Editors**
```
Actions Stack:
┌───────────────┐
│ Type "world"  │ ← UNDO removes this
├───────────────┤
│ Type "Hello " │
├───────────────┤
│ Delete line   │
└───────────────┘
```

---

## 💻 Java Implementation

### 🔧 **Array-Based Stack**

```java
public class ArrayStack {
    private int[] arr;
    private int top;
    private int capacity;
    
    public ArrayStack(int size) {
        arr = new int[size];
        capacity = size;
        top = -1;
    }
    
    // Push - O(1)
    public void push(int value) {
        if (isFull()) {
            throw new StackOverflowError("Stack is full");
        }
        arr[++top] = value;
    }
    
    // Pop - O(1)
    public int pop() {
        if (isEmpty()) {
            throw new EmptyStackException();
        }
        return arr[top--];
    }
    
    // Peek - O(1)
    public int peek() {
        if (isEmpty()) {
            throw new EmptyStackException();
        }
        return arr[top];
    }
    
    public boolean isEmpty() {
        return top == -1;
    }
    
    public boolean isFull() {
        return top == capacity - 1;
    }
    
    public int size() {
        return top + 1;
    }
}
```

### 🔗 **LinkedList-Based Stack**

```java
public class LinkedListStack {
    private Node top;
    private int size;
    
    private class Node {
        int data;
        Node next;
        
        Node(int data) {
            this.data = data;
        }
    }
    
    // Push - O(1)
    public void push(int value) {
        Node newNode = new Node(value);
        newNode.next = top;
        top = newNode;
        size++;
    }
    
    // Pop - O(1)
    public int pop() {
        if (isEmpty()) {
            throw new EmptyStackException();
        }
        int value = top.data;
        top = top.next;
        size--;
        return value;
    }
    
    // Peek - O(1)
    public int peek() {
        if (isEmpty()) {
            throw new EmptyStackException();
        }
        return top.data;
    }
    
    public boolean isEmpty() {
        return top == null;
    }
    
    public int size() {
        return size;
    }
}
```

### 📦 **Using Java's Built-in Stack**

```java
import java.util.Stack;

public class StackExample {
    public static void main(String[] args) {
        Stack<Integer> stack = new Stack<>();
        
        // Push elements
        stack.push(10);
        stack.push(20);
        stack.push(30);
        
        // Peek top element
        int top = stack.peek(); // 30 (doesn't remove)
        
        // Pop elements
        int popped = stack.pop(); // 30 (removes and returns)
        
        // Check if empty
        boolean empty = stack.isEmpty(); // false
        
        // Search (returns 1-based position from top)
        int position = stack.search(10); // 2
        
        // Size
        int size = stack.size(); // 2
    }
}
```

---

## 🎯 Common Stack Patterns

### 🔥 **Pattern 1: Valid Parentheses**

**Problem**: Check if brackets are balanced

```java
public boolean isValid(String s) {
    Stack<Character> stack = new Stack<>();
    
    for (char c : s.toCharArray()) {
        if (c == '(' || c == '{' || c == '[') {
            stack.push(c);
        } else {
            if (stack.isEmpty()) return false;
            
            char top = stack.pop();
            if ((c == ')' && top != '(') ||
                (c == '}' && top != '{') ||
                (c == ']' && top != '[')) {
                return false;
            }
        }
    }
    
    return stack.isEmpty();
}

// Examples:
// "()" → true
// "()[]{}" → true
// "(]" → false
// "([)]" → false
```

---

### 🔥 **Pattern 2: Evaluate Postfix Expression**

```java
public int evaluatePostfix(String expression) {
    Stack<Integer> stack = new Stack<>();
    
    for (char c : expression.toCharArray()) {
        if (Character.isDigit(c)) {
            stack.push(c - '0');
        } else {
            int b = stack.pop();
            int a = stack.pop();
            
            switch (c) {
                case '+': stack.push(a + b); break;
                case '-': stack.push(a - b); break;
                case '*': stack.push(a * b); break;
                case '/': stack.push(a / b); break;
            }
        }
    }
    
    return stack.pop();
}

// Example: "231*+9-" → 2 + (3 * 1) - 9 = -4
```

---

### 🔥 **Pattern 3: Next Greater Element**

```java
public int[] nextGreaterElement(int[] nums) {
    int n = nums.length;
    int[] result = new int[n];
    Stack<Integer> stack = new Stack<>();
    
    // Traverse from right to left
    for (int i = n - 1; i >= 0; i--) {
        // Pop smaller elements
        while (!stack.isEmpty() && stack.peek() <= nums[i]) {
            stack.pop();
        }
        
        result[i] = stack.isEmpty() ? -1 : stack.peek();
        stack.push(nums[i]);
    }
    
    return result;
}

// Example: [4, 5, 2, 10] → [5, 10, 10, -1]
```

---

### 🔥 **Pattern 4: Min Stack (O(1) getMin)**

```java
class MinStack {
    private Stack<Integer> stack;
    private Stack<Integer> minStack;
    
    public MinStack() {
        stack = new Stack<>();
        minStack = new Stack<>();
    }
    
    public void push(int val) {
        stack.push(val);
        
        if (minStack.isEmpty() || val <= minStack.peek()) {
            minStack.push(val);
        }
    }
    
    public void pop() {
        if (stack.pop().equals(minStack.peek())) {
            minStack.pop();
        }
    }
    
    public int top() {
        return stack.peek();
    }
    
    public int getMin() {
        return minStack.peek();
    }
}
```

---

## 🏢 Real-World Applications

### 🔷 **Browser Back Button**

```java
public class Browser {
    private Stack<String> history = new Stack<>();
    private String currentPage;
    
    public void visit(String url) {
        if (currentPage != null) {
            history.push(currentPage);
        }
        currentPage = url;
    }
    
    public String back() {
        if (!history.isEmpty()) {
            String previousPage = currentPage;
            currentPage = history.pop();
            return currentPage;
        }
        return currentPage;
    }
}
```

### 🔷 **Function Call Stack**

```java
void functionA() {
    functionB();
}

void functionB() {
    functionC();
}

void functionC() {
    // Base case
}

// Call Stack:
// ┌───────────┐
// │ functionC │
// ├───────────┤
// │ functionB │
// ├───────────┤
// │ functionA │
// └───────────┘
```

### 🔷 **Expression Parsing (Compilers)**

```java
// Infix to Postfix conversion
public String infixToPostfix(String infix) {
    Stack<Character> stack = new Stack<>();
    StringBuilder result = new StringBuilder();
    
    for (char c : infix.toCharArray()) {
        if (Character.isLetterOrDigit(c)) {
            result.append(c);
        } else if (c == '(') {
            stack.push(c);
        } else if (c == ')') {
            while (!stack.isEmpty() && stack.peek() != '(') {
                result.append(stack.pop());
            }
            stack.pop(); // Remove '('
        } else {
            while (!stack.isEmpty() && precedence(c) <= precedence(stack.peek())) {
                result.append(stack.pop());
            }
            stack.push(c);
        }
    }
    
    while (!stack.isEmpty()) {
        result.append(stack.pop());
    }
    
    return result.toString();
}

private int precedence(char op) {
    if (op == '+' || op == '-') return 1;
    if (op == '*' || op == '/') return 2;
    return 0;
}
```

---

## 🧩 Practice Problems

### 🟢 **Easy**
1. Valid Parentheses (LeetCode #20)
2. Min Stack (LeetCode #155)
3. Implement Queue using Stacks (LeetCode #232)

### 🟡 **Medium**
4. Evaluate Reverse Polish Notation (LeetCode #150)
5. Daily Temperatures (LeetCode #739)
6. Decode String (LeetCode #394)

### 🔴 **Hard**
7. Largest Rectangle in Histogram (LeetCode #84)
8. Basic Calculator (LeetCode #224)

---

## 🎯 Key Takeaways

✅ Stack = **LIFO** (Last In, First Out)  
✅ All operations: **O(1)** time complexity  
✅ Perfect for **parsing, undo/redo, DFS**  
✅ Function calls use **call stack**  

---

## 🚀 What's Next?

➡️ **[Queues: FIFO Power](./02_Queues_Fundamentals.md)**

**🎮 Achievement Unlocked: Stack Mastery!** 🏆

