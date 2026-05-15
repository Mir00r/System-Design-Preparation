# 🪟 Sliding Window Pattern: The Efficiency Game-Changer! 🚀

---

## 🎯 What is the Sliding Window Pattern?

The **Sliding Window** technique uses a window (subarray) that slides through the data structure to solve problems involving **contiguous sequences** in **O(n) time** instead of O(n²).

> **"Instead of recalculating everything for each position, slide the window and update incrementally!"**

---

## 🎨 Window Types

### 1️⃣ **Fixed Size Window**
```
Window size = 3 (constant)

Array: [1, 3, 2, 6, -1, 4, 1, 8, 2]
       [1, 3, 2]                    ← Window
          [3, 2, 6]                 ← Slide right
             [2, 6, -1]             ← Continue...
```

### 2️⃣ **Variable Size Window (Expandable/Shrinkable)**
```
Window expands/shrinks based on condition

Array: [1, 3, 2, 6, -1, 4, 1, 8, 2]
       [1]                          ← Start small
       [1, 3, 2]                    ← Expand
          [3, 2]                    ← Shrink
          [3, 2, 6, -1, 4]          ← Expand again
```

---

## 🔥 Fixed Size Window Problems

### **Problem 1: Maximum Sum Subarray of Size K** 🟢

**Naive Approach O(n×k)**:
```java
public int maxSumNaive(int[] arr, int k) {
    int maxSum = Integer.MIN_VALUE;
    
    for (int i = 0; i <= arr.length - k; i++) {
        int sum = 0;
        for (int j = i; j < i + k; j++) {
            sum += arr[j];
        }
        maxSum = Math.max(maxSum, sum);
    }
    
    return maxSum;
}
```

**Sliding Window O(n)**:
```java
public int maxSumOptimized(int[] arr, int k) {
    int windowSum = 0;
    
    // Calculate sum of first window
    for (int i = 0; i < k; i++) {
        windowSum += arr[i];
    }
    
    int maxSum = windowSum;
    
    // Slide the window
    for (int i = k; i < arr.length; i++) {
        windowSum = windowSum - arr[i - k] + arr[i];
        maxSum = Math.max(maxSum, windowSum);
    }
    
    return maxSum;
}

// Example: arr = [2, 1, 5, 1, 3, 2], k = 3
// Window 1: [2, 1, 5] sum = 8
// Window 2: [1, 5, 1] sum = 7 (remove 2, add 1)
// Window 3: [5, 1, 3] sum = 9 (remove 1, add 3) ← MAX
// Window 4: [1, 3, 2] sum = 6 (remove 5, add 2)
// Result: 9
```

**Visual**:
```
Array: [2, 1, 5, 1, 3, 2], k=3

Step 1: Initial window [2, 1, 5]
        ─────────
        sum = 8

Step 2: Slide right, remove 2, add 1
           ─────────
        sum = 8 - 2 + 1 = 7

Step 3: Slide right, remove 1, add 3
              ─────────
        sum = 7 - 1 + 3 = 9 ✅ MAX

Step 4: Slide right, remove 5, add 2
                 ─────────
        sum = 9 - 5 + 2 = 6
```

---

### **Problem 2: Average of Subarrays of Size K** 🟢

```java
public double[] findAverages(int[] arr, int k) {
    double[] result = new double[arr.length - k + 1];
    int windowSum = 0;
    
    // First window
    for (int i = 0; i < k; i++) {
        windowSum += arr[i];
    }
    result[0] = (double) windowSum / k;
    
    // Slide window
    for (int i = k; i < arr.length; i++) {
        windowSum = windowSum - arr[i - k] + arr[i];
        result[i - k + 1] = (double) windowSum / k;
    }
    
    return result;
}

// Example: arr = [1, 3, 2, 6, -1, 4, 1, 8, 2], k = 5
// Result: [2.2, 2.8, 2.4, 3.6, 2.8]
```

---

### **Problem 3: Max of All Subarrays of Size K** 🔴

**Using Deque (Monotonic Queue)**:
```java
public int[] maxSlidingWindow(int[] nums, int k) {
    Deque<Integer> deque = new ArrayDeque<>();
    int[] result = new int[nums.length - k + 1];
    int ri = 0;
    
    for (int i = 0; i < nums.length; i++) {
        // Remove indices outside current window
        while (!deque.isEmpty() && deque.peekFirst() < i - k + 1) {
            deque.pollFirst();
        }
        
        // Remove smaller elements (not useful)
        while (!deque.isEmpty() && nums[deque.peekLast()] < nums[i]) {
            deque.pollLast();
        }
        
        deque.offerLast(i);
        
        // Add to result if window complete
        if (i >= k - 1) {
            result[ri++] = nums[deque.peekFirst()];
        }
    }
    
    return result;
}

// Example: nums = [1,3,-1,-3,5,3,6,7], k = 3
// Result: [3, 3, 5, 5, 6, 7]
```

**Visual**:
```
nums = [1, 3, -1, -3, 5, 3, 6, 7], k=3

Window [1, 3, -1]:
Deque maintains indices in decreasing order of values
Deque: [1] (index of 3)
Max: 3

Window [3, -1, -3]:
Deque: [1] (index of 3)
Max: 3

Window [-1, -3, 5]:
Deque: [4] (index of 5)
Max: 5

Continue...
```

---

## 🔥 Variable Size Window Problems

### **Problem 4: Longest Substring Without Repeating Characters** 🟡

```java
public int lengthOfLongestSubstring(String s) {
    Set<Character> window = new HashSet<>();
    int maxLength = 0;
    int left = 0;
    
    for (int right = 0; right < s.length(); right++) {
        // Shrink window if duplicate found
        while (window.contains(s.charAt(right))) {
            window.remove(s.charAt(left));
            left++;
        }
        
        // Expand window
        window.add(s.charAt(right));
        maxLength = Math.max(maxLength, right - left + 1);
    }
    
    return maxLength;
}

// Example: s = "abcabcbb"
// Result: 3 ("abc")
```

**Visual**:
```
String: "abcabcbb"

Window: "a"      left=0, right=0, len=1
Window: "ab"     left=0, right=1, len=2
Window: "abc"    left=0, right=2, len=3 ✅ MAX so far
Window: "abc|a"  Duplicate! Shrink
        "bc|a"   left=1, right=3
Window: "bca"    len=3
Window: "bca|b"  Duplicate! Shrink
        "ca|b"   left=2, right=4
Window: "cab"    len=3
...continue

Result: 3
```

---

### **Problem 5: Minimum Window Substring** 🔴

```java
public String minWindow(String s, String t) {
    if (s.length() < t.length()) return "";
    
    Map<Character, Integer> required = new HashMap<>();
    for (char c : t.toCharArray()) {
        required.put(c, required.getOrDefault(c, 0) + 1);
    }
    
    Map<Character, Integer> window = new HashMap<>();
    int left = 0, right = 0;
    int formed = 0;
    int required_size = required.size();
    
    int[] result = {-1, 0, 0}; // {length, left, right}
    
    while (right < s.length()) {
        char c = s.charAt(right);
        window.put(c, window.getOrDefault(c, 0) + 1);
        
        if (required.containsKey(c) && 
            window.get(c).intValue() == required.get(c).intValue()) {
            formed++;
        }
        
        // Try to shrink window
        while (left <= right && formed == required_size) {
            c = s.charAt(left);
            
            // Update result if smaller window found
            if (result[0] == -1 || right - left + 1 < result[0]) {
                result[0] = right - left + 1;
                result[1] = left;
                result[2] = right;
            }
            
            window.put(c, window.get(c) - 1);
            if (required.containsKey(c) && 
                window.get(c) < required.get(c)) {
                formed--;
            }
            
            left++;
        }
        
        right++;
    }
    
    return result[0] == -1 ? "" : 
           s.substring(result[1], result[2] + 1);
}

// Example: s = "ADOBECODEBANC", t = "ABC"
// Result: "BANC"
```

**Visual**:
```
s = "ADOBECODEBANC", t = "ABC"

Expand until all chars found:
"ADOBEC" contains A, B, C ✅

Shrink from left:
"DOBEC" - missing A ❌
Stop shrinking, continue expanding...

"ODEBANC" contains A, B, C ✅
Shrink: "DEBANC", "EBANC", "BANC" ✅ (smallest!)

Result: "BANC" (length 4)
```

---

### **Problem 6: Longest Substring with At Most K Distinct Characters** 🟡

```java
public int lengthOfLongestSubstringKDistinct(String s, int k) {
    Map<Character, Integer> window = new HashMap<>();
    int left = 0;
    int maxLength = 0;
    
    for (int right = 0; right < s.length(); right++) {
        char c = s.charAt(right);
        window.put(c, window.getOrDefault(c, 0) + 1);
        
        // Shrink if more than k distinct
        while (window.size() > k) {
            char leftChar = s.charAt(left);
            window.put(leftChar, window.get(leftChar) - 1);
            if (window.get(leftChar) == 0) {
                window.remove(leftChar);
            }
            left++;
        }
        
        maxLength = Math.max(maxLength, right - left + 1);
    }
    
    return maxLength;
}

// Example: s = "eceba", k = 2
// Result: 3 ("ece")
```

---

### **Problem 7: Fruits Into Baskets** 🟡

```java
public int totalFruit(int[] fruits) {
    // Same as longest substring with at most 2 distinct
    Map<Integer, Integer> basket = new HashMap<>();
    int left = 0;
    int maxFruits = 0;
    
    for (int right = 0; right < fruits.length; right++) {
        basket.put(fruits[right], 
                   basket.getOrDefault(fruits[right], 0) + 1);
        
        // Shrink if more than 2 types
        while (basket.size() > 2) {
            int leftFruit = fruits[left];
            basket.put(leftFruit, basket.get(leftFruit) - 1);
            if (basket.get(leftFruit) == 0) {
                basket.remove(leftFruit);
            }
            left++;
        }
        
        maxFruits = Math.max(maxFruits, right - left + 1);
    }
    
    return maxFruits;
}

// Example: fruits = [1,2,1,2,3,1,1]
// Result: 5 ([1,2,1,2] or [2,3,1,1])
```

---

### **Problem 8: Subarrays with K Different Integers** 🔴

```java
public int subarraysWithKDistinct(int[] nums, int k) {
    return atMostK(nums, k) - atMostK(nums, k - 1);
}

private int atMostK(int[] nums, int k) {
    Map<Integer, Integer> window = new HashMap<>();
    int left = 0;
    int count = 0;
    
    for (int right = 0; right < nums.length; right++) {
        window.put(nums[right], 
                   window.getOrDefault(nums[right], 0) + 1);
        
        while (window.size() > k) {
            window.put(nums[left], window.get(nums[left]) - 1);
            if (window.get(nums[left]) == 0) {
                window.remove(nums[left]);
            }
            left++;
        }
        
        // All subarrays ending at right
        count += right - left + 1;
    }
    
    return count;
}

// Example: nums = [1,2,1,2,3], k = 2
// Result: 7
```

---

## 🎯 Sliding Window Decision Tree

```
Choose Sliding Window When:
│
├─ Need CONTIGUOUS subarray/substring?
│  └─ YES ✅
│      │
│      ├─ FIXED size window?
│      │  └─ YES → Fixed Window Template
│      │        (Max sum, Average, etc.)
│      │
│      └─ VARIABLE size window?
│         ├─ Maximize length?
│         │  └─ Expand until invalid, then shrink
│         │
│         ├─ Minimize length?
│         │  └─ Shrink while valid, then expand
│         │
│         └─ Count subarrays?
│            └─ At most K trick
│
└─ Need NON-CONTIGUOUS?
   └─ NO ❌ → Use other patterns (DP, etc.)
```

---

## 📝 Sliding Window Template

### **Fixed Size**:
```java
public ReturnType fixedWindow(DataType[] arr, int k) {
    // Step 1: Calculate first window
    WindowState window = new WindowState();
    for (int i = 0; i < k; i++) {
        window.add(arr[i]);
    }
    
    ReturnType result = window.getResult();
    
    // Step 2: Slide window
    for (int i = k; i < arr.length; i++) {
        window.remove(arr[i - k]); // Remove left
        window.add(arr[i]);         // Add right
        result = updateResult(result, window);
    }
    
    return result;
}
```

### **Variable Size (Maximize)**:
```java
public int variableWindow(DataType[] arr) {
    int left = 0;
    int maxLength = 0;
    WindowState window = new WindowState();
    
    for (int right = 0; right < arr.length; right++) {
        // Expand window
        window.add(arr[right]);
        
        // Shrink if invalid
        while (!window.isValid()) {
            window.remove(arr[left]);
            left++;
        }
        
        // Update result
        maxLength = Math.max(maxLength, right - left + 1);
    }
    
    return maxLength;
}
```

---

## 🏢 Real-World Applications

### 🔷 **Stock Trading: Moving Average**

```java
public class MovingAverage {
    private Queue<Integer> window;
    private int size;
    private double sum;
    
    public MovingAverage(int size) {
        this.window = new LinkedList<>();
        this.size = size;
        this.sum = 0;
    }
    
    public double next(int val) {
        window.offer(val);
        sum += val;
        
        if (window.size() > size) {
            sum -= window.poll();
        }
        
        return sum / window.size();
    }
}
```

### 🔷 **Rate Limiting: Sliding Window Log**

```java
public class RateLimiter {
    private Queue<Long> requestTimestamps;
    private int maxRequests;
    private long windowMs;
    
    public RateLimiter(int maxRequests, long windowMs) {
        this.requestTimestamps = new LinkedList<>();
        this.maxRequests = maxRequests;
        this.windowMs = windowMs;
    }
    
    public boolean allowRequest() {
        long now = System.currentTimeMillis();
        
        // Remove old timestamps
        while (!requestTimestamps.isEmpty() && 
               requestTimestamps.peek() < now - windowMs) {
            requestTimestamps.poll();
        }
        
        if (requestTimestamps.size() < maxRequests) {
            requestTimestamps.offer(now);
            return true;
        }
        
        return false;
    }
}
```

---

## 🧩 Practice Problem Set

### 🟢 **Easy** (Build Foundation)
1. Maximum Sum Subarray of Size K
2. Average of Subarrays of Size K
3. Contains Duplicate II (LeetCode #219)
4. Max Consecutive Ones III (LeetCode #1004)

### 🟡 **Medium** (Interview Common)
5. Longest Substring Without Repeating (LeetCode #3)
6. Longest Substring with At Most K Distinct (LeetCode #340)
7. Fruits Into Baskets (LeetCode #904)
8. Longest Repeating Character Replacement (LeetCode #424)
9. Permutation in String (LeetCode #567)
10. Find All Anagrams in a String (LeetCode #438)

### 🔴 **Hard** (Advanced)
11. Minimum Window Substring (LeetCode #76)
12. Sliding Window Maximum (LeetCode #239)
13. Subarrays with K Different Integers (LeetCode #992)
14. Minimum Window Subsequence (LeetCode #727)

---

## 💡 Pro Tips

### ✅ **When to Use Sliding Window**
1. Problem involves **contiguous** sequence
2. Keywords: "subarray", "substring", "consecutive"
3. Need to find **longest/shortest/optimal** window
4. Can be solved with **nested loops** (optimize to O(n))

### ❌ **When NOT to Use**
1. Need **non-contiguous** elements (use DP)
2. Array is **unsorted** and order matters differently
3. Problem involves **subsequences** (not subarrays)

---

## 🎯 Key Takeaways

✅ Sliding Window reduces **O(n×k) → O(n)**  
✅ **Two types**: Fixed size, Variable size  
✅ Variable: Expand until invalid, shrink until valid  
✅ Perfect for **contiguous sequences**  
✅ **HashMap/Set** often used for tracking window state  

---

## 🚀 What's Next?

➡️ **[Prefix Sum Pattern](./05_PrefixSum_Pattern.md)**  
➡️ **[Practice Problems](../ProblemSets/README.md)**

---

**🎮 Achievement Unlocked: Sliding Window Master!** 🏆

*Remember: "Slide smart, code fast!"* 🪟🚀

