# ♻️ Recursion Fundamentals: Think Recursively! 🌀

> **"To understand recursion, you must first understand recursion!"** 😄

Master the **art of thinking recursively** - the foundation for solving complex problems through elegant, simple solutions. Recursion is the key to backtracking, dynamic programming, and tree/graph algorithms! 🚀

---

## 📋 Table of Contents

1. [What is Recursion?](#-what-is-recursion)
2. [How Recursion Works](#-how-recursion-works)
3. [The 3 Laws of Recursion](#-the-3-laws-of-recursion)
4. [Recursion vs Iteration](#-recursion-vs-iteration)
5. [Common Recursion Patterns](#-common-recursion-patterns)
6. [Tail Recursion](#-tail-recursion)
7. [Practice Problems](#-practice-problems)

---

## 🎯 What is Recursion?

**Recursion** = A function that calls itself to solve smaller instances of the same problem.

```java
public int factorial(int n) {
    if (n <= 1) return 1;           // Base case
    return n * factorial(n - 1);     // Recursive call
}
```

### **Key Idea**:
```
Break big problem → Smaller subproblems → Solve recursively → Combine results
```

---

## 🔄 How Recursion Works

### **Example: Factorial(4)**

```
factorial(4)
    ↓
    4 * factorial(3)
         ↓
         3 * factorial(2)
              ↓
              2 * factorial(1)
                   ↓
                   1 (BASE CASE)
                   
Unwinding:
1 ← factorial(1)
2 ← 2 * 1 = 2
6 ← 3 * 2 = 6
24 ← 4 * 6 = 24 ✅
```

### **Call Stack Visualization**:
```
                factorial(4) = 24
                ↑
                factorial(3) = 6
                ↑
                factorial(2) = 2
                ↑
                factorial(1) = 1  ← BASE CASE
```

---

## 📜 The 3 Laws of Recursion

### **Law 1: Base Case** 🛑

Every recursive function MUST have a base case (stopping condition).

```java
❌ BAD - Infinite recursion:
public int bad(int n) {
    return n + bad(n - 1);  // No base case! 💥
}

✅ GOOD:
public int good(int n) {
    if (n <= 0) return 0;   // Base case ✅
    return n + good(n - 1);
}
```

### **Law 2: Progress Toward Base Case** 🎯

Each recursive call must get closer to the base case.

```java
❌ BAD - Never reaches base:
public int bad(int n) {
    if (n == 0) return 0;
    return n + bad(n + 1);  // Moving away! 💥
}

✅ GOOD:
public int good(int n) {
    if (n == 0) return 0;
    return n + good(n - 1);  // Moving toward 0 ✅
}
```

### **Law 3: Assume Recursion Works** 🤝

Trust that the recursive call will work correctly.

```java
// To calculate factorial(5):
// ASSUME factorial(4) works correctly
// Then: factorial(5) = 5 * factorial(4)

public int factorial(int n) {
    if (n <= 1) return 1;
    
    // Assume factorial(n-1) works! 🤝
    return n * factorial(n - 1);
}
```

---

## ⚖️ Recursion vs Iteration

### **Same Problem, Two Approaches**

```java
// RECURSIVE
public int sumRecursive(int n) {
    if (n <= 0) return 0;
    return n + sumRecursive(n - 1);
}

// ITERATIVE
public int sumIterative(int n) {
    int sum = 0;
    for (int i = 1; i <= n; i++) {
        sum += i;
    }
    return sum;
}
```

### **Comparison**

| Aspect | Recursion | Iteration |
|--------|-----------|-----------|
| **Code** | Often shorter, cleaner | Often longer |
| **Space** | O(n) stack space | O(1) space |
| **Speed** | Slower (function calls) | Faster |
| **Use When** | Tree/graph, divide-conquer | Simple loops |
| **Natural For** | Hierarchical data | Sequential data |

---

## 🎨 Common Recursion Patterns

### **Pattern 1: Linear Recursion** (Single call)

```java
// Example: Sum of array
public int sumArray(int[] arr, int index) {
    if (index >= arr.length) return 0;
    return arr[index] + sumArray(arr, index + 1);
}

// Example: Power
public int power(int base, int exp) {
    if (exp == 0) return 1;
    return base * power(base, exp - 1);
}
```

---

### **Pattern 2: Binary Recursion** (Two calls)

```java
// Example: Fibonacci
public int fibonacci(int n) {
    if (n <= 1) return n;
    return fibonacci(n - 1) + fibonacci(n - 2);
}

// Example: Binary Search
public int binarySearch(int[] arr, int target, int left, int right) {
    if (left > right) return -1;
    
    int mid = left + (right - left) / 2;
    if (arr[mid] == target) return mid;
    
    if (arr[mid] < target) {
        return binarySearch(arr, target, mid + 1, right);
    } else {
        return binarySearch(arr, target, left, mid - 1);
    }
}
```

---

### **Pattern 3: Multiple Recursion** (>2 calls)

```java
// Example: Permutations
public void permutations(int[] nums, List<Integer> current, 
                         List<List<Integer>> result) {
    if (current.size() == nums.length) {
        result.add(new ArrayList<>(current));
        return;
    }
    
    for (int num : nums) {
        if (!current.contains(num)) {
            current.add(num);
            permutations(nums, current, result);  // Recursive call
            current.remove(current.size() - 1);
        }
    }
}
```

---

### **Pattern 4: Divide and Conquer**

```java
// Example: Merge Sort
public void mergeSort(int[] arr, int left, int right) {
    if (left >= right) return;
    
    int mid = left + (right - left) / 2;
    
    // Divide
    mergeSort(arr, left, mid);
    mergeSort(arr, mid + 1, right);
    
    // Conquer
    merge(arr, left, mid, right);
}
```

---

## 🎯 Classic Recursion Problems

### **1. Reverse a String**

```java
public String reverseString(String s) {
    if (s.isEmpty()) return s;
    return reverseString(s.substring(1)) + s.charAt(0);
}

// Example: "abc"
// reverseString("bc") + 'a'
// (reverseString("c") + 'b') + 'a'
// ("" + 'c' + 'b') + 'a'
// "cba" ✅
```

---

### **2. Check Palindrome**

```java
public boolean isPalindrome(String s, int left, int right) {
    if (left >= right) return true;
    
    if (s.charAt(left) != s.charAt(right)) return false;
    
    return isPalindrome(s, left + 1, right - 1);
}

// Usage:
isPalindrome("racecar", 0, 6);  // true
```

---

### **3. Generate Parentheses**

```java
public List<String> generateParenthesis(int n) {
    List<String> result = new ArrayList<>();
    generate(result, "", 0, 0, n);
    return result;
}

private void generate(List<String> result, String current, 
                     int open, int close, int max) {
    if (current.length() == max * 2) {
        result.add(current);
        return;
    }
    
    if (open < max) {
        generate(result, current + "(", open + 1, close, max);
    }
    
    if (close < open) {
        generate(result, current + ")", open, close + 1, max);
    }
}

// generateParenthesis(3) → ["((()))", "(()())", "(())()", "()(())", "()()()"]
```

---

### **4. Tower of Hanoi**

```java
public void towerOfHanoi(int n, char from, char to, char aux) {
    if (n == 1) {
        System.out.println("Move disk 1 from " + from + " to " + to);
        return;
    }
    
    // Move n-1 disks from 'from' to 'aux'
    towerOfHanoi(n - 1, from, aux, to);
    
    // Move largest disk from 'from' to 'to'
    System.out.println("Move disk " + n + " from " + from + " to " + to);
    
    // Move n-1 disks from 'aux' to 'to'
    towerOfHanoi(n - 1, aux, to, from);
}

// towerOfHanoi(3, 'A', 'C', 'B');
```

**Visual (3 disks)**:
```
Initial:          Goal:
  |                 |
 -|-                |
--+--               |
-----             -----
  A                 C

Steps: 2^3 - 1 = 7 moves
```

---

## 🚀 Tail Recursion

### **What is Tail Recursion?**

Recursive call is the LAST operation (no computation after return).

```java
// NOT tail recursive (multiplication after return)
public int factorial(int n) {
    if (n <= 1) return 1;
    return n * factorial(n - 1);  // Multiplication after ❌
}

// Tail recursive (accumulator pattern)
public int factorialTail(int n, int accumulator) {
    if (n <= 1) return accumulator;
    return factorialTail(n - 1, n * accumulator);  // No operation after ✅
}

// Usage:
factorialTail(5, 1);  // 5! = 120
```

### **Why Tail Recursion?**

✅ Can be optimized to O(1) space by compiler (tail call optimization)  
✅ No stack overflow for deep recursion  

---

## 🎯 Recursion Thinking Process

### **Step-by-Step Approach**:

```
1. IDENTIFY BASE CASE(S)
   └─ What's the simplest input? When to stop?

2. DEFINE RECURSIVE CASE
   └─ How to break problem into smaller subproblem?

3. TRUST THE RECURSION
   └─ Assume recursive call works correctly

4. COMBINE RESULTS
   └─ How to use recursive result to solve current problem?

5. VERIFY
   └─ Test with small inputs, trace execution
```

### **Example: Sum of Digits**

```java
// Problem: Sum digits of 1234 → 1+2+3+4 = 10

// Step 1: Base case
// Single digit: return the digit

// Step 2: Recursive case
// Last digit + sum of remaining digits

// Step 3-4: Implementation
public int sumDigits(int n) {
    if (n < 10) return n;  // Base case
    
    return (n % 10) + sumDigits(n / 10);  // Recursive case
}

// sumDigits(1234)
// = 4 + sumDigits(123)
// = 4 + (3 + sumDigits(12))
// = 4 + 3 + (2 + sumDigits(1))
// = 4 + 3 + 2 + 1
// = 10 ✅
```

---

## 🏢 Real-World Applications

### **File System** 📁
```java
// Count files in directory recursively
public int countFiles(File dir) {
    if (dir.isFile()) return 1;
    
    int count = 0;
    for (File file : dir.listFiles()) {
        count += countFiles(file);  // Recursive
    }
    return count;
}
```

### **JSON Parser** 🔧
```java
// Parse nested JSON objects recursively
public void parseJSON(JSONObject obj) {
    for (String key : obj.keySet()) {
        Object value = obj.get(key);
        if (value instanceof JSONObject) {
            parseJSON((JSONObject) value);  // Recursive
        } else {
            processValue(key, value);
        }
    }
}
```

---

## 🧩 Practice Problems

### 🟢 **Easy** (Build Foundation)
1. [Fibonacci Number](https://leetcode.com/problems/fibonacci-number/) - Classic recursion
2. [Power of Two](https://leetcode.com/problems/power-of-two/) - Base case practice
3. [Reverse String](https://leetcode.com/problems/reverse-string/) - String recursion
4. [Reverse Linked List](https://leetcode.com/problems/reverse-linked-list/) - List recursion
5. [Maximum Depth of Binary Tree](https://leetcode.com/problems/maximum-depth-of-binary-tree/) - Tree recursion

### 🟡 **Medium** (Apply Skills)
6. [Generate Parentheses](https://leetcode.com/problems/generate-parentheses/) - Backtracking
7. [Subsets](https://leetcode.com/problems/subsets/) - Power set
8. [Permutations](https://leetcode.com/problems/permutations/) - Backtracking
9. [Combination Sum](https://leetcode.com/problems/combination-sum/) - Backtracking
10. [Letter Combinations of Phone](https://leetcode.com/problems/letter-combinations-of-a-phone-number/) - Multiple recursion

### 🔴 **Hard** (Master Level)
11. [N-Queens](https://leetcode.com/problems/n-queens/) - Classic backtracking
12. [Sudoku Solver](https://leetcode.com/problems/sudoku-solver/) - Constraint satisfaction

---

## ⚠️ Common Pitfalls

### **1. Missing Base Case**
```java
❌ public int infinite(int n) {
    return n + infinite(n - 1);  // Stack overflow! 💥
}

✅ public int correct(int n) {
    if (n <= 0) return 0;  // Base case ✅
    return n + correct(n - 1);
}
```

### **2. Stack Overflow**
```java
// Too deep recursion
factorial(100000);  // 💥 StackOverflowError

// Solution: Use iteration or tail recursion
```

### **3. Inefficient Recursion**
```java
❌ // O(2^n) - Recalculates same values
public int fib(int n) {
    if (n <= 1) return n;
    return fib(n-1) + fib(n-2);
}

✅ // O(n) - Memoization
public int fib(int n, int[] memo) {
    if (n <= 1) return n;
    if (memo[n] != 0) return memo[n];
    memo[n] = fib(n-1, memo) + fib(n-2, memo);
    return memo[n];
}
```

---

## 💡 Pro Tips

✅ **Draw the recursion tree** - visualize calls  
✅ **Trace execution** with small inputs first  
✅ **Identify base case** before recursive case  
✅ **Use memoization** for overlapping subproblems  
✅ **Consider iteration** if recursion is too deep  
✅ **Trust the recursion** - don't overthink!  

---

## 🎯 Key Takeaways

1. **Recursion = Self-calling function** solving smaller subproblems
2. **3 Laws**: Base case, Progress toward base, Assume it works
3. **Patterns**: Linear, Binary, Multiple, Divide & Conquer
4. **Tail recursion** = Optimization opportunity
5. **Recursion ≈ Mathematical induction** in proof

---

## 🚀 What's Next?

➡️ **[Backtracking Patterns](./02_Backtracking_Patterns.md)** - N-Queens, Sudoku  
➡️ **[Practice Problems](../ProblemSets/README.md)**  

---

**🎮 Achievement Unlocked: Recursion Master!** 🏆

*Remember: "To understand recursion, you must first understand recursion!"* ♻️✨

---

*Previous: [← Binary Search Mastery](../Searching/01_BinarySearch_Mastery.md) | Next: [Backtracking Patterns →](./02_Backtracking_Patterns.md)*

