# 🔍 Binary Search Mastery: The Power of Divide and Conquer! ⚡

> **"Binary Search: Where O(n) becomes O(log n) - the magic of halving!"**

Master the **most powerful searching technique** used everywhere from databases to AI systems. This tutorial will make you a **binary search expert** ready for any interview! 🎯

---

## 📋 Table of Contents

1. [What is Binary Search?](#-what-is-binary-search)
2. [The Standard Template](#-the-standard-template)
3. [Binary Search Variations](#-binary-search-variations)
4. [Advanced Applications](#-advanced-applications)
5. [Common Pitfalls](#-common-pitfalls)
6. [Practice Problems](#-practice-problems)

---

## 🎯 What is Binary Search?

**Binary Search** = Find target in sorted array by repeatedly halving search space

```
Array: [1, 3, 5, 7, 9, 11, 13, 15, 17, 19]
Target: 13

Step 1: Check middle (9)  → 13 > 9, search right
Step 2: Check middle (15) → 13 < 15, search left
Step 3: Check middle (13) → Found! ✅

Comparisons: 3 (vs 7 in linear search)
```

### **Why Binary Search?**

```
Linear Search:   O(n)       - Check every element
Binary Search:   O(log n)   - Halve search space each time

Example with 1,000,000 elements:
Linear:    ~1,000,000 comparisons
Binary:    ~20 comparisons (2^20 ≈ 1,000,000)
```

---

## 📝 The Standard Template

### **Template 1: Find Exact Match**

```java
public class BinarySearch {
    public static int binarySearch(int[] arr, int target) {
        int left = 0;
        int right = arr.length - 1;
        
        while (left <= right) {
            int mid = left + (right - left) / 2; // Avoid overflow
            
            if (arr[mid] == target) {
                return mid; // Found!
            } else if (arr[mid] < target) {
                left = mid + 1; // Search right half
            } else {
                right = mid - 1; // Search left half
            }
        }
        
        return -1; // Not found
    }
    
    public static void main(String[] args) {
        int[] arr = {1, 3, 5, 7, 9, 11, 13, 15};
        System.out.println(binarySearch(arr, 13)); // Output: 6
        System.out.println(binarySearch(arr, 6));  // Output: -1
    }
}
```

**Visual Trace**:
```
Array: [1, 3, 5, 7, 9, 11, 13, 15]
Target: 13

Iteration 1:
left = 0, right = 7, mid = 3
arr[3] = 7 < 13 → left = 4

Iteration 2:
left = 4, right = 7, mid = 5
arr[5] = 11 < 13 → left = 6

Iteration 3:
left = 6, right = 7, mid = 6
arr[6] = 13 == 13 → FOUND! ✅
```

---

## 🔄 Binary Search Variations

### **1. Find First Occurrence** (Leftmost)

```java
public static int findFirst(int[] arr, int target) {
    int left = 0, right = arr.length - 1;
    int result = -1;
    
    while (left <= right) {
        int mid = left + (right - left) / 2;
        
        if (arr[mid] == target) {
            result = mid;
            right = mid - 1; // Continue searching left
        } else if (arr[mid] < target) {
            left = mid + 1;
        } else {
            right = mid - 1;
        }
    }
    
    return result;
}

// Example: [1, 2, 2, 2, 3, 4, 5]
// findFirst(arr, 2) → 1 (index of first 2)
```

**Visual**:
```
Array: [1, 2, 2, 2, 3, 4, 5]
Target: 2

Found at mid=3, but continue left:
       [1, 2, 2, 2, 3, 4, 5]
            ↑
         First 2 at index 1
```

---

### **2. Find Last Occurrence** (Rightmost)

```java
public static int findLast(int[] arr, int target) {
    int left = 0, right = arr.length - 1;
    int result = -1;
    
    while (left <= right) {
        int mid = left + (right - left) / 2;
        
        if (arr[mid] == target) {
            result = mid;
            left = mid + 1; // Continue searching right
        } else if (arr[mid] < target) {
            left = mid + 1;
        } else {
            right = mid - 1;
        }
    }
    
    return result;
}

// Example: [1, 2, 2, 2, 3, 4, 5]
// findLast(arr, 2) → 3 (index of last 2)
```

---

### **3. Search Insert Position**

```java
public static int searchInsert(int[] arr, int target) {
    int left = 0, right = arr.length - 1;
    
    while (left <= right) {
        int mid = left + (right - left) / 2;
        
        if (arr[mid] == target) {
            return mid;
        } else if (arr[mid] < target) {
            left = mid + 1;
        } else {
            right = mid - 1;
        }
    }
    
    return left; // Position where target should be inserted
}

// Example: [1, 3, 5, 6]
// searchInsert(arr, 5) → 2
// searchInsert(arr, 2) → 1 (insert between 1 and 3)
// searchInsert(arr, 7) → 4 (insert at end)
```

---

### **4. Search in Rotated Sorted Array**

```java
public static int searchRotated(int[] arr, int target) {
    int left = 0, right = arr.length - 1;
    
    while (left <= right) {
        int mid = left + (right - left) / 2;
        
        if (arr[mid] == target) {
            return mid;
        }
        
        // Determine which half is sorted
        if (arr[left] <= arr[mid]) {
            // Left half is sorted
            if (target >= arr[left] && target < arr[mid]) {
                right = mid - 1;
            } else {
                left = mid + 1;
            }
        } else {
            // Right half is sorted
            if (target > arr[mid] && target <= arr[right]) {
                left = mid + 1;
            } else {
                right = mid - 1;
            }
        }
    }
    
    return -1;
}

// Example: [4, 5, 6, 7, 0, 1, 2]
// searchRotated(arr, 0) → 4
```

**Visual**:
```
Rotated Array: [4, 5, 6, 7, 0, 1, 2]
Target: 0

Original: [0, 1, 2, 4, 5, 6, 7] (rotated at index 4)

Step 1: mid=3 (value 7)
        Left sorted: [4,5,6,7]
        0 not in [4,7] → search right

Step 2: mid=5 (value 1)
        Right sorted: [1,2]
        0 not in [1,2] → search left

Step 3: mid=4 (value 0) → FOUND! ✅
```

---

### **5. Find Peak Element**

```java
public static int findPeakElement(int[] arr) {
    int left = 0, right = arr.length - 1;
    
    while (left < right) {
        int mid = left + (right - left) / 2;
        
        if (arr[mid] < arr[mid + 1]) {
            left = mid + 1; // Peak is on right
        } else {
            right = mid; // Peak is on left or mid
        }
    }
    
    return left; // Peak index
}

// Example: [1, 2, 3, 1]
// findPeakElement(arr) → 2 (value 3 is peak)
```

**Visual**:
```
Array: [1, 2, 3, 1]
         ↗   ↗   ↘
       Peak at index 2 (value 3)
```

---

## 🎯 Advanced Applications

### **1. Find Minimum in Rotated Sorted Array**

```java
public static int findMin(int[] arr) {
    int left = 0, right = arr.length - 1;
    
    while (left < right) {
        int mid = left + (right - left) / 2;
        
        if (arr[mid] > arr[right]) {
            // Minimum is in right half
            left = mid + 1;
        } else {
            // Minimum is in left half or mid
            right = mid;
        }
    }
    
    return arr[left];
}

// Example: [4, 5, 6, 7, 0, 1, 2]
// findMin(arr) → 0
```

---

### **2. Search in 2D Matrix** (Treat as 1D sorted)

```java
public static boolean searchMatrix(int[][] matrix, int target) {
    if (matrix.length == 0) return false;
    
    int m = matrix.length;
    int n = matrix[0].length;
    int left = 0, right = m * n - 1;
    
    while (left <= right) {
        int mid = left + (right - left) / 2;
        int row = mid / n;
        int col = mid % n;
        int midValue = matrix[row][col];
        
        if (midValue == target) {
            return true;
        } else if (midValue < target) {
            left = mid + 1;
        } else {
            right = mid - 1;
        }
    }
    
    return false;
}

// Matrix: [[1,3,5,7],[10,11,16,20],[23,30,34,60]]
// searchMatrix(matrix, 3) → true
```

---

### **3. Find Square Root** (Binary Search on Answer)

```java
public static int mySqrt(int x) {
    if (x < 2) return x;
    
    int left = 1, right = x / 2;
    
    while (left <= right) {
        int mid = left + (right - left) / 2;
        long square = (long) mid * mid;
        
        if (square == x) {
            return mid;
        } else if (square < x) {
            left = mid + 1;
        } else {
            right = mid - 1;
        }
    }
    
    return right; // Floor value
}

// mySqrt(8) → 2 (since sqrt(8) ≈ 2.82)
// mySqrt(16) → 4
```

---

### **4. Koko Eating Bananas** (Binary Search on Speed)

```java
public static int minEatingSpeed(int[] piles, int h) {
    int left = 1;
    int right = Arrays.stream(piles).max().getAsInt();
    
    while (left < right) {
        int mid = left + (right - left) / 2;
        
        if (canFinish(piles, mid, h)) {
            right = mid; // Try slower
        } else {
            left = mid + 1; // Must eat faster
        }
    }
    
    return left;
}

private static boolean canFinish(int[] piles, int speed, int h) {
    int hours = 0;
    for (int pile : piles) {
        hours += (pile + speed - 1) / speed; // Ceiling division
    }
    return hours <= h;
}

// piles = [3,6,7,11], h = 8
// minEatingSpeed(piles, h) → 4
```

**Explanation**:
```
Piles: [3, 6, 7, 11], Hours: 8

Try speed = 4:
Pile 3:  1 hour
Pile 6:  2 hours
Pile 7:  2 hours
Pile 11: 3 hours
Total:   8 hours ✅ (Just enough!)

Lower speeds won't work, higher speeds waste time.
Answer: 4 bananas/hour
```

---

## ⚠️ Common Pitfalls

### **1. Integer Overflow**

❌ **Wrong**:
```java
int mid = (left + right) / 2; // Can overflow!
```

✅ **Correct**:
```java
int mid = left + (right - left) / 2;
```

---

### **2. Infinite Loop**

❌ **Wrong**:
```java
while (left < right) {
    int mid = left + (right - left) / 2;
    if (condition) {
        left = mid; // Infinite loop if left==mid!
    } else {
        right = mid - 1;
    }
}
```

✅ **Correct**:
```java
while (left < right) {
    int mid = left + (right - left) / 2;
    if (condition) {
        left = mid + 1; // Always move
    } else {
        right = mid;
    }
}
```

---

### **3. Off-by-One Errors**

```java
// Finding first occurrence
while (left <= right) { // Use <=, not <
    int mid = left + (right - left) / 2;
    if (arr[mid] == target) {
        result = mid;
        right = mid - 1; // Not mid!
    }
    // ...
}
```

---

## 📊 Binary Search Decision Tree

```
Is array SORTED?
├─ YES → Binary Search applicable
│  │
│  ├─ Find exact match? → Standard template
│  ├─ Find first/last? → Modified template
│  ├─ Find insertion point? → Return left pointer
│  └─ Array rotated? → Check which half sorted
│
└─ NO → Can you binary search on answer space?
   ├─ Monotonic function? → Binary search on value
   └─ Otherwise → Use linear search
```

---

## 🏢 Real-World Applications

### **Google** 🔍
- Search indexing: Binary search on sorted index
- Auto-complete: Prefix search with binary search
- Chrome version control: `git bisect` (binary search on commits)

### **Databases** 💾
- B-tree indexes: Multi-level binary search
- Query optimization: Binary search on sorted columns
- Range queries: Find first/last occurrences

### **Operating Systems** 💻
- Memory allocation: Binary search for free blocks
- Process scheduling: Find next available time slot

---

## 🧩 Practice Problems

### 🟢 **Easy** (Build Foundation)
1. [Binary Search](https://leetcode.com/problems/binary-search/) - Standard template
2. [Search Insert Position](https://leetcode.com/problems/search-insert-position/) - Find insertion point
3. [First Bad Version](https://leetcode.com/problems/first-bad-version/) - Find first occurrence
4. [Sqrt(x)](https://leetcode.com/problems/sqrtx/) - Binary search on answer

### 🟡 **Medium** (Interview Common)
5. [Search in Rotated Array](https://leetcode.com/problems/search-in-rotated-sorted-array/) - Modified search
6. [Find Peak Element](https://leetcode.com/problems/find-peak-element/) - Peak finding
7. [Search 2D Matrix](https://leetcode.com/problems/search-a-2d-matrix/) - 2D binary search
8. [Find Minimum in Rotated](https://leetcode.com/problems/find-minimum-in-rotated-sorted-array/) - Find min
9. [Koko Eating Bananas](https://leetcode.com/problems/koko-eating-bananas/) - Binary search on speed
10. [Capacity To Ship Packages](https://leetcode.com/problems/capacity-to-ship-packages-within-d-days/) - Binary search on capacity

### 🔴 **Hard** (Advanced Mastery)
11. [Median of Two Sorted Arrays](https://leetcode.com/problems/median-of-two-sorted-arrays/) - Advanced binary search
12. [Split Array Largest Sum](https://leetcode.com/problems/split-array-largest-sum/) - Binary search + greedy

---

## 💡 Pro Tips

✅ **Always check if array is sorted** before using binary search  
✅ **Use `left + (right - left) / 2`** to avoid overflow  
✅ **Be careful with `<` vs `<=`** in while condition  
✅ **For first/last occurrence**, continue searching after finding  
✅ **Binary search on answer space** for optimization problems  
✅ **Draw diagrams** to visualize search space reduction  

---

## 🎯 Binary Search Template Cheatsheet

```java
// Template 1: Standard Binary Search
while (left <= right) {
    int mid = left + (right - left) / 2;
    if (arr[mid] == target) return mid;
    if (arr[mid] < target) left = mid + 1;
    else right = mid - 1;
}
return -1;

// Template 2: Find Boundary (first/last)
while (left <= right) {
    int mid = left + (right - left) / 2;
    if (arr[mid] == target) {
        result = mid;
        // For first: right = mid - 1
        // For last: left = mid + 1
    }
    // ... continue search
}
return result;

// Template 3: Minimize/Maximize
while (left < right) {
    int mid = left + (right - left) / 2;
    if (canAchieve(mid)) {
        right = mid; // Try smaller (minimize)
        // left = mid + 1; // Try larger (maximize)
    } else {
        left = mid + 1;
    }
}
return left;
```

---

## 🎯 Key Takeaways

1. **Binary search reduces O(n) → O(log n)**
2. **Requires sorted data** (or searchable answer space)
3. **Many variations** for different problems
4. **Watch for overflow and off-by-one errors**
5. **Can binary search on answer space** for optimization

---

## 🚀 What's Next?

➡️ **[Recursion Fundamentals](../Recursion/01_Recursion_Fundamentals.md)**  
➡️ **[Dynamic Programming](../DynamicProgramming/01_DP_Fundamentals.md)**  
➡️ **[Practice Problems](../ProblemSets/README.md)**

---

**🎮 Achievement Unlocked: Binary Search Master!** 🏆

*Remember: "When in doubt, divide and conquer!"* 🔍⚡

