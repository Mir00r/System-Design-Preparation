# 🔙 Backtracking Patterns: The Art of Trying All Possibilities! 🎯

> **"Try, fail, backtrack, try again - the systematic approach to solving constraint problems!"**

Master **backtracking** - the powerful technique for generating all possible solutions and finding optimal ones through systematic trial and error. Essential for permutations, combinations, N-Queens, Sudoku, and more! 🚀

---

## 📋 Table of Contents

1. [What is Backtracking?](#-what-is-backtracking)
2. [Backtracking Template](#-backtracking-template)
3. [Classic Patterns](#-classic-backtracking-patterns)
4. [N-Queens Problem](#-n-queens-problem)
5. [Sudoku Solver](#-sudoku-solver)
6. [Practice Problems](#-practice-problems)

---

## 🎯 What is Backtracking?

**Backtracking** = Systematic way to try all possibilities by building solutions incrementally and abandoning ("backtracking") when constraints are violated.

### **The 3-Step Process**:
```
1. CHOOSE    → Make a choice
2. EXPLORE   → Recursively explore
3. UNCHOOSE  → Undo the choice (backtrack)
```

### **When to Use Backtracking?**

```
✅ Need to find ALL solutions
✅ Problem has CONSTRAINTS
✅ Build solution INCREMENTALLY
✅ Can detect INVALID states early
✅ Keywords: "all combinations", "all permutations", "all valid"
```

---

## 📝 Backtracking Template

### **Universal Template**

```java
public void backtrack(State state, List<Solution> solutions) {
    // 1. BASE CASE - Solution found
    if (isComplete(state)) {
        solutions.add(new Solution(state));
        return;
    }
    
    // 2. TRY all choices
    for (Choice choice : getChoices(state)) {
        // 3. CHECK if choice is valid
        if (isValid(state, choice)) {
            // 4. MAKE choice
            makeChoice(state, choice);
            
            // 5. RECURSE
            backtrack(state, solutions);
            
            // 6. UNDO choice (backtrack)
            undoChoice(state, choice);
        }
    }
}
```

### **Visual Flow**:
```
                Start
                  ↓
            Try choice 1
            /         \
         Valid?      Invalid?
           ↓            ↓
        Recurse      Skip
           ↓
     Complete?
     /        \
   Yes        No
    ↓          ↓
  Save     Backtrack
           ↓
      Try choice 2
           ...
```

---

## 🎨 Classic Backtracking Patterns

### **Pattern 1: Permutations** (Order Matters)

**Problem**: Generate all permutations of [1,2,3]

```java
public List<List<Integer>> permute(int[] nums) {
    List<List<Integer>> result = new ArrayList<>();
    backtrack(nums, new ArrayList<>(), result);
    return result;
}

private void backtrack(int[] nums, List<Integer> current, 
                       List<List<Integer>> result) {
    // Base case: permutation complete
    if (current.size() == nums.length) {
        result.add(new ArrayList<>(current));
        return;
    }
    
    // Try each number
    for (int num : nums) {
        if (current.contains(num)) continue;  // Skip used numbers
        
        current.add(num);                // Choose
        backtrack(nums, current, result); // Explore
        current.remove(current.size() - 1); // Unchoose
    }
}

// Input: [1,2,3]
// Output: [[1,2,3], [1,3,2], [2,1,3], [2,3,1], [3,1,2], [3,2,1]]
```

**Decision Tree**:
```
                    []
          /         |         \
        [1]        [2]        [3]
       /  \       /  \       /  \
    [1,2][1,3] [2,1][2,3] [3,1][3,2]
      |    |     |    |     |    |
   [1,2,3][1,3,2][2,1,3][2,3,1][3,1,2][3,2,1]
```

---

### **Pattern 2: Combinations** (Order Doesn't Matter)

**Problem**: Generate all combinations of k numbers from [1..n]

```java
public List<List<Integer>> combine(int n, int k) {
    List<List<Integer>> result = new ArrayList<>();
    backtrack(1, n, k, new ArrayList<>(), result);
    return result;
}

private void backtrack(int start, int n, int k, 
                       List<Integer> current, List<List<Integer>> result) {
    // Base case: combination complete
    if (current.size() == k) {
        result.add(new ArrayList<>(current));
        return;
    }
    
    // Try numbers from start to n
    for (int i = start; i <= n; i++) {
        current.add(i);                     // Choose
        backtrack(i + 1, n, k, current, result); // Explore (i+1 to avoid duplicates)
        current.remove(current.size() - 1);  // Unchoose
    }
}

// Input: n=4, k=2
// Output: [[1,2], [1,3], [1,4], [2,3], [2,4], [3,4]]
```

---

### **Pattern 3: Subsets** (Power Set)

**Problem**: Generate all subsets of [1,2,3]

```java
public List<List<Integer>> subsets(int[] nums) {
    List<List<Integer>> result = new ArrayList<>();
    backtrack(nums, 0, new ArrayList<>(), result);
    return result;
}

private void backtrack(int[] nums, int start, 
                       List<Integer> current, List<List<Integer>> result) {
    // Add current subset (every state is valid!)
    result.add(new ArrayList<>(current));
    
    // Try adding each remaining element
    for (int i = start; i < nums.length; i++) {
        current.add(nums[i]);              // Choose
        backtrack(nums, i + 1, current, result); // Explore
        current.remove(current.size() - 1); // Unchoose
    }
}

// Input: [1,2,3]
// Output: [[], [1], [1,2], [1,2,3], [1,3], [2], [2,3], [3]]
```

**Decision Tree**:
```
                        []
                /              \
            [1]                 []
          /     \             /    \
      [1,2]   [1]        [2]       []
      /   \   /  \       /  \
  [1,2,3][1,2][1,3][1] [2,3][2]   [3] []
```

---

### **Pattern 4: Combination Sum** (With Repetition)

**Problem**: Find all combinations that sum to target (can reuse elements)

```java
public List<List<Integer>> combinationSum(int[] candidates, int target) {
    List<List<Integer>> result = new ArrayList<>();
    Arrays.sort(candidates);  // Optional: helps with pruning
    backtrack(candidates, target, 0, new ArrayList<>(), result);
    return result;
}

private void backtrack(int[] candidates, int remain, int start,
                       List<Integer> current, List<List<Integer>> result) {
    // Base cases
    if (remain < 0) return;  // Exceeded target
    if (remain == 0) {       // Found solution
        result.add(new ArrayList<>(current));
        return;
    }
    
    // Try each candidate
    for (int i = start; i < candidates.length; i++) {
        current.add(candidates[i]);
        // Can reuse same element: pass i (not i+1)
        backtrack(candidates, remain - candidates[i], i, current, result);
        current.remove(current.size() - 1);
    }
}

// Input: candidates=[2,3,6,7], target=7
// Output: [[2,2,3], [7]]
```

---

### **Pattern 5: Palindrome Partitioning**

**Problem**: Partition string into all possible palindromic substrings

```java
public List<List<String>> partition(String s) {
    List<List<String>> result = new ArrayList<>();
    backtrack(s, 0, new ArrayList<>(), result);
    return result;
}

private void backtrack(String s, int start, 
                       List<String> current, List<List<String>> result) {
    // Base case: reached end of string
    if (start == s.length()) {
        result.add(new ArrayList<>(current));
        return;
    }
    
    // Try all possible partitions
    for (int end = start + 1; end <= s.length(); end++) {
        String substring = s.substring(start, end);
        
        if (isPalindrome(substring)) {
            current.add(substring);           // Choose
            backtrack(s, end, current, result); // Explore
            current.remove(current.size() - 1); // Unchoose
        }
    }
}

private boolean isPalindrome(String s) {
    int left = 0, right = s.length() - 1;
    while (left < right) {
        if (s.charAt(left++) != s.charAt(right--)) return false;
    }
    return true;
}

// Input: "aab"
// Output: [["a","a","b"], ["aa","b"]]
```

---

## 👑 N-Queens Problem

**Problem**: Place N queens on N×N chessboard so no two queens attack each other.

### **Constraints**:
- Queens attack horizontally, vertically, and diagonally
- Only one queen per row/column/diagonal

### **Solution**:
```java
public List<List<String>> solveNQueens(int n) {
    List<List<String>> result = new ArrayList<>();
    char[][] board = new char[n][n];
    
    // Initialize board
    for (int i = 0; i < n; i++) {
        Arrays.fill(board[i], '.');
    }
    
    backtrack(board, 0, result);
    return result;
}

private void backtrack(char[][] board, int row, 
                       List<List<String>> result) {
    // Base case: all queens placed
    if (row == board.length) {
        result.add(construct(board));
        return;
    }
    
    // Try placing queen in each column
    for (int col = 0; col < board.length; col++) {
        if (isValid(board, row, col)) {
            board[row][col] = 'Q';           // Choose
            backtrack(board, row + 1, result); // Explore next row
            board[row][col] = '.';            // Unchoose
        }
    }
}

private boolean isValid(char[][] board, int row, int col) {
    int n = board.length;
    
    // Check column
    for (int i = 0; i < row; i++) {
        if (board[i][col] == 'Q') return false;
    }
    
    // Check diagonal (top-left)
    for (int i = row - 1, j = col - 1; i >= 0 && j >= 0; i--, j--) {
        if (board[i][j] == 'Q') return false;
    }
    
    // Check diagonal (top-right)
    for (int i = row - 1, j = col + 1; i >= 0 && j < n; i--, j++) {
        if (board[i][j] == 'Q') return false;
    }
    
    return true;
}

private List<String> construct(char[][] board) {
    List<String> result = new ArrayList<>();
    for (char[] row : board) {
        result.add(new String(row));
    }
    return result;
}
```

**Visual (4-Queens)**:
```
Solution 1:        Solution 2:
. Q . .            . . Q .
. . . Q            Q . . .
Q . . .            . . . Q
. . Q .            . Q . .
```

---

## 🎲 Sudoku Solver

**Problem**: Fill 9×9 grid following Sudoku rules.

### **Constraints**:
- Each row contains 1-9
- Each column contains 1-9
- Each 3×3 box contains 1-9

### **Solution**:
```java
public void solveSudoku(char[][] board) {
    backtrack(board);
}

private boolean backtrack(char[][] board) {
    // Find empty cell
    for (int row = 0; row < 9; row++) {
        for (int col = 0; col < 9; col++) {
            if (board[row][col] == '.') {
                // Try digits 1-9
                for (char num = '1'; num <= '9'; num++) {
                    if (isValid(board, row, col, num)) {
                        board[row][col] = num;      // Choose
                        
                        if (backtrack(board)) {      // Explore
                            return true;
                        }
                        
                        board[row][col] = '.';       // Unchoose
                    }
                }
                return false;  // No valid digit found
            }
        }
    }
    return true;  // Board complete
}

private boolean isValid(char[][] board, int row, int col, char num) {
    // Check row
    for (int j = 0; j < 9; j++) {
        if (board[row][j] == num) return false;
    }
    
    // Check column
    for (int i = 0; i < 9; i++) {
        if (board[i][col] == num) return false;
    }
    
    // Check 3×3 box
    int boxRow = (row / 3) * 3;
    int boxCol = (col / 3) * 3;
    for (int i = boxRow; i < boxRow + 3; i++) {
        for (int j = boxCol; j < boxCol + 3; j++) {
            if (board[i][j] == num) return false;
        }
    }
    
    return true;
}
```

---

## 🎯 Optimization Techniques

### **1. Pruning** ✂️

Stop early when constraints violated:
```java
private void backtrack(int[] nums, int target, int sum, ...) {
    if (sum > target) return;  // Prune! No need to continue
    // ...
}
```

### **2. Sorting** 📊

Sort candidates for better pruning:
```java
Arrays.sort(candidates);  // Now can break early
for (int i = start; i < candidates.length; i++) {
    if (candidates[i] > remain) break;  // All larger, stop!
    // ...
}
```

### **3. Memoization** 💾

Cache results for overlapping subproblems:
```java
Map<String, List<Solution>> memo = new HashMap<>();

private List<Solution> backtrack(State state) {
    String key = state.toString();
    if (memo.containsKey(key)) return memo.get(key);
    // ... backtracking logic
    memo.put(key, result);
    return result;
}
```

---

## 🏢 Real-World Applications

### **Scheduling** 📅
- Course scheduling with constraints
- Employee shift assignment
- Meeting room allocation

### **Games** 🎮
- Chess move generation
- Puzzle solvers (Sudoku, Crossword)
- Path finding in mazes

### **Configuration** ⚙️
- Network configuration
- Circuit design
- Resource allocation

---

## 🧩 Practice Problems

### 🟢 **Easy**
1. [Permutations](https://leetcode.com/problems/permutations/)
2. [Subsets](https://leetcode.com/problems/subsets/)
3. [Combinations](https://leetcode.com/problems/combinations/)

### 🟡 **Medium**
4. [Generate Parentheses](https://leetcode.com/problems/generate-parentheses/)
5. [Letter Combinations of Phone Number](https://leetcode.com/problems/letter-combinations-of-a-phone-number/)
6. [Combination Sum](https://leetcode.com/problems/combination-sum/)
7. [Palindrome Partitioning](https://leetcode.com/problems/palindrome-partitioning/)
8. [Word Search](https://leetcode.com/problems/word-search/)
9. [Subsets II](https://leetcode.com/problems/subsets-ii/)
10. [Permutations II](https://leetcode.com/problems/permutations-ii/)

### 🔴 **Hard**
11. [N-Queens](https://leetcode.com/problems/n-queens/)
12. [Sudoku Solver](https://leetcode.com/problems/sudoku-solver/)
13. [Word Search II](https://leetcode.com/problems/word-search-ii/)

---

## 💡 Pro Tips

✅ **Visualize the decision tree** - draw it out!  
✅ **Identify constraints early** - validate before recursing  
✅ **Use pruning** - stop invalid paths ASAP  
✅ **Handle duplicates** - sort + skip duplicates  
✅ **Practice the template** - muscle memory!  

---

## 🎯 Key Takeaways

1. **Backtracking = Try + Recurse + Undo**
2. **Three steps**: Choose → Explore → Unchoose
3. **Use when**: Need all solutions with constraints
4. **Optimize**: Prune, sort, memoize
5. **Template works** for 90% of backtracking problems

---

## 🚀 What's Next?

➡️ **[Dynamic Programming](../DynamicProgramming/01_DP_Fundamentals.md)**  
➡️ **[Graph Algorithms](../GraphAlgorithms/01_Graph_Traversal.md)**  
➡️ **[Practice Problems](../ProblemSets/README.md)**  

---

**🎮 Achievement Unlocked: Backtracking Master!** 🏆

*Remember: "Try all paths, backtrack when stuck!"* 🔙✨

