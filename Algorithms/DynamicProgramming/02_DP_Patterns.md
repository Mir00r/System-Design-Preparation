# 💎 Dynamic Programming Patterns: The 10 Essentials! 🎯

> **"Master these 10 patterns, and you'll solve 90% of DP interview problems!"**

This guide covers the **10 most important DP patterns** you'll encounter in interviews. Each pattern has a template, examples, and practice problems. 🚀

---

## 📋 Table of Contents

1. [Pattern 1: 0/1 Knapsack](#pattern-1-01-knapsack-)
2. [Pattern 2: Unbounded Knapsack](#pattern-2-unbounded-knapsack-)
3. [Pattern 3: Longest Common Subsequence (LCS)](#pattern-3-longest-common-subsequence-lcs-)
4. [Pattern 4: Longest Increasing Subsequence (LIS)](#pattern-4-longest-increasing-subsequence-lis-)
5. [Pattern 5: Palindrome](#pattern-5-palindrome-)
6. [Pattern 6: Grid Path](#pattern-6-grid-path-)
7. [Pattern 7: Fibonacci Variants](#pattern-7-fibonacci-variants-)
8. [Pattern 8: State Machine](#pattern-8-state-machine-)
9. [Pattern 9: Partition](#pattern-9-partition-)
10. [Pattern 10: Kadane's Algorithm](#pattern-10-kadanes-algorithm-)

---

## Pattern 1: 0/1 Knapsack 🎒

### **Problem Type**
Choose or skip items to maximize/minimize value with constraints.

### **Recognition**
- Keywords: "choose or not choose", "subset", "partition"
- Each item used AT MOST ONCE

### **Template**
```java
public int knapsack(int[] weights, int[] values, int capacity) {
    int n = weights.length;
    int[][] dp = new int[n + 1][capacity + 1];
    
    for (int i = 1; i <= n; i++) {
        for (int w = 1; w <= capacity; w++) {
            if (weights[i-1] <= w) {
                // Choose or skip
                dp[i][w] = Math.max(
                    values[i-1] + dp[i-1][w - weights[i-1]],  // Take
                    dp[i-1][w]                                 // Skip
                );
            } else {
                dp[i][w] = dp[i-1][w];  // Can't take
            }
        }
    }
    
    return dp[n][capacity];
}
```

### **Example: Subset Sum**
```java
public boolean canPartition(int[] nums) {
    int sum = Arrays.stream(nums).sum();
    if (sum % 2 != 0) return false;
    
    int target = sum / 2;
    boolean[] dp = new boolean[target + 1];
    dp[0] = true;
    
    for (int num : nums) {
        for (int j = target; j >= num; j--) {
            dp[j] = dp[j] || dp[j - num];
        }
    }
    
    return dp[target];
}
```

### **Practice Problems**
- [Partition Equal Subset Sum](https://leetcode.com/problems/partition-equal-subset-sum/)
- [Target Sum](https://leetcode.com/problems/target-sum/)
- [Last Stone Weight II](https://leetcode.com/problems/last-stone-weight-ii/)

---

## Pattern 2: Unbounded Knapsack 🔄

### **Problem Type**
Items can be used UNLIMITED times.

### **Recognition**
- Keywords: "unlimited", "infinite supply", "reuse"
- Each item can be selected multiple times

### **Template**
```java
public int unboundedKnapsack(int[] weights, int[] values, int capacity) {
    int[] dp = new int[capacity + 1];
    
    for (int w = 1; w <= capacity; w++) {
        for (int i = 0; i < weights.length; i++) {
            if (weights[i] <= w) {
                dp[w] = Math.max(dp[w], 
                    values[i] + dp[w - weights[i]]);
            }
        }
    }
    
    return dp[capacity];
}
```

### **Example: Coin Change**
```java
public int coinChange(int[] coins, int amount) {
    int[] dp = new int[amount + 1];
    Arrays.fill(dp, amount + 1);
    dp[0] = 0;
    
    for (int i = 1; i <= amount; i++) {
        for (int coin : coins) {
            if (coin <= i) {
                dp[i] = Math.min(dp[i], dp[i - coin] + 1);
            }
        }
    }
    
    return dp[amount] > amount ? -1 : dp[amount];
}
```

### **Practice Problems**
- [Coin Change](https://leetcode.com/problems/coin-change/)
- [Coin Change II](https://leetcode.com/problems/coin-change-2/)
- [Perfect Squares](https://leetcode.com/problems/perfect-squares/)

---

## Pattern 3: Longest Common Subsequence (LCS) 📝

### **Problem Type**
Find longest/shortest subsequence common to two sequences.

### **Recognition**
- Keywords: "subsequence", "two strings", "common"
- Comparing two sequences

### **Template**
```java
public int lcs(String text1, String text2) {
    int m = text1.length(), n = text2.length();
    int[][] dp = new int[m + 1][n + 1];
    
    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (text1.charAt(i-1) == text2.charAt(j-1)) {
                dp[i][j] = dp[i-1][j-1] + 1;  // Match
            } else {
                dp[i][j] = Math.max(dp[i-1][j], dp[i][j-1]);
            }
        }
    }
    
    return dp[m][n];
}
```

### **Example: Edit Distance**
```java
public int minDistance(String word1, String word2) {
    int m = word1.length(), n = word2.length();
    int[][] dp = new int[m + 1][n + 1];
    
    for (int i = 0; i <= m; i++) dp[i][0] = i;
    for (int j = 0; j <= n; j++) dp[0][j] = j;
    
    for (int i = 1; i <= m; i++) {
        for (int j = 1; j <= n; j++) {
            if (word1.charAt(i-1) == word2.charAt(j-1)) {
                dp[i][j] = dp[i-1][j-1];
            } else {
                dp[i][j] = 1 + Math.min(
                    Math.min(dp[i-1][j], dp[i][j-1]),  // Delete/Insert
                    dp[i-1][j-1]                        // Replace
                );
            }
        }
    }
    
    return dp[m][n];
}
```

### **Practice Problems**
- [Longest Common Subsequence](https://leetcode.com/problems/longest-common-subsequence/)
- [Edit Distance](https://leetcode.com/problems/edit-distance/)
- [Delete Operation for Two Strings](https://leetcode.com/problems/delete-operation-for-two-strings/)

---

## Pattern 4: Longest Increasing Subsequence (LIS) 📈

### **Problem Type**
Find longest subsequence with specific property.

### **Recognition**
- Keywords: "increasing", "decreasing", "longest subsequence"
- One sequence, find optimal subsequence

### **Template (O(n²))**
```java
public int lengthOfLIS(int[] nums) {
    int n = nums.length;
    int[] dp = new int[n];
    Arrays.fill(dp, 1);
    int maxLen = 1;
    
    for (int i = 1; i < n; i++) {
        for (int j = 0; j < i; j++) {
            if (nums[i] > nums[j]) {
                dp[i] = Math.max(dp[i], dp[j] + 1);
            }
        }
        maxLen = Math.max(maxLen, dp[i]);
    }
    
    return maxLen;
}
```

### **Optimized (O(n log n))**
```java
public int lengthOfLIS(int[] nums) {
    List<Integer> tails = new ArrayList<>();
    
    for (int num : nums) {
        int pos = Collections.binarySearch(tails, num);
        if (pos < 0) pos = -(pos + 1);
        
        if (pos == tails.size()) {
            tails.add(num);
        } else {
            tails.set(pos, num);
        }
    }
    
    return tails.size();
}
```

### **Practice Problems**
- [Longest Increasing Subsequence](https://leetcode.com/problems/longest-increasing-subsequence/)
- [Russian Doll Envelopes](https://leetcode.com/problems/russian-doll-envelopes/)
- [Number of LIS](https://leetcode.com/problems/number-of-longest-increasing-subsequence/)

---

## Pattern 5: Palindrome 🔄

### **Problem Type**
Problems involving palindromes (strings reading same forwards/backwards).

### **Recognition**
- Keywords: "palindrome", "palindromic"
- String manipulation with symmetry

### **Template**
```java
public String longestPalindrome(String s) {
    int n = s.length();
    boolean[][] dp = new boolean[n][n];
    int start = 0, maxLen = 1;
    
    // All single characters are palindromes
    for (int i = 0; i < n; i++) {
        dp[i][i] = true;
    }
    
    // Check length 2
    for (int i = 0; i < n - 1; i++) {
        if (s.charAt(i) == s.charAt(i + 1)) {
            dp[i][i + 1] = true;
            start = i;
            maxLen = 2;
        }
    }
    
    // Check length 3 and above
    for (int len = 3; len <= n; len++) {
        for (int i = 0; i <= n - len; i++) {
            int j = i + len - 1;
            
            if (s.charAt(i) == s.charAt(j) && dp[i + 1][j - 1]) {
                dp[i][j] = true;
                start = i;
                maxLen = len;
            }
        }
    }
    
    return s.substring(start, start + maxLen);
}
```

### **Example: Palindromic Substrings Count**
```java
public int countSubstrings(String s) {
    int n = s.length();
    boolean[][] dp = new boolean[n][n];
    int count = 0;
    
    for (int i = 0; i < n; i++) {
        dp[i][i] = true;
        count++;
    }
    
    for (int i = 0; i < n - 1; i++) {
        if (s.charAt(i) == s.charAt(i + 1)) {
            dp[i][i + 1] = true;
            count++;
        }
    }
    
    for (int len = 3; len <= n; len++) {
        for (int i = 0; i <= n - len; i++) {
            int j = i + len - 1;
            if (s.charAt(i) == s.charAt(j) && dp[i + 1][j - 1]) {
                dp[i][j] = true;
                count++;
            }
        }
    }
    
    return count;
}
```

### **Practice Problems**
- [Longest Palindromic Substring](https://leetcode.com/problems/longest-palindromic-substring/)
- [Palindromic Substrings](https://leetcode.com/problems/palindromic-substrings/)
- [Longest Palindromic Subsequence](https://leetcode.com/problems/longest-palindromic-subsequence/)

---

## Pattern 6: Grid Path 🗺️

### **Problem Type**
Count paths or find optimal path in grid.

### **Recognition**
- Keywords: "grid", "matrix", "paths", "m×n"
- 2D navigation problems

### **Template**
```java
public int uniquePaths(int m, int n) {
    int[][] dp = new int[m][n];
    
    // Base cases: first row and column
    for (int i = 0; i < m; i++) dp[i][0] = 1;
    for (int j = 0; j < n; j++) dp[0][j] = 1;
    
    // Fill DP table
    for (int i = 1; i < m; i++) {
        for (int j = 1; j < n; j++) {
            dp[i][j] = dp[i-1][j] + dp[i][j-1];
        }
    }
    
    return dp[m-1][n-1];
}
```

### **Example: Minimum Path Sum**
```java
public int minPathSum(int[][] grid) {
    int m = grid.length, n = grid[0].length;
    int[][] dp = new int[m][n];
    
    dp[0][0] = grid[0][0];
    
    for (int i = 1; i < m; i++) {
        dp[i][0] = dp[i-1][0] + grid[i][0];
    }
    
    for (int j = 1; j < n; j++) {
        dp[0][j] = dp[0][j-1] + grid[0][j];
    }
    
    for (int i = 1; i < m; i++) {
        for (int j = 1; j < n; j++) {
            dp[i][j] = grid[i][j] + 
                Math.min(dp[i-1][j], dp[i][j-1]);
        }
    }
    
    return dp[m-1][n-1];
}
```

### **Practice Problems**
- [Unique Paths](https://leetcode.com/problems/unique-paths/)
- [Minimum Path Sum](https://leetcode.com/problems/minimum-path-sum/)
- [Maximal Square](https://leetcode.com/problems/maximal-square/)

---

## Pattern 7: Fibonacci Variants 🔢

### **Problem Type**
Problems following Fibonacci-like recurrence.

### **Recognition**
- Keywords: "ways to reach", "decode", "climb"
- dp[i] depends on dp[i-1] and dp[i-2]

### **Template**
```java
public int fibVariant(int n) {
    if (n <= 1) return baseCase;
    
    int prev2 = base0;
    int prev1 = base1;
    
    for (int i = 2; i <= n; i++) {
        int curr = someFunction(prev1, prev2);
        prev2 = prev1;
        prev1 = curr;
    }
    
    return prev1;
}
```

### **Example: Decode Ways**
```java
public int numDecodings(String s) {
    if (s.charAt(0) == '0') return 0;
    
    int n = s.length();
    int prev2 = 1;  // dp[i-2]
    int prev1 = 1;  // dp[i-1]
    
    for (int i = 1; i < n; i++) {
        int curr = 0;
        
        // Single digit
        if (s.charAt(i) != '0') {
            curr += prev1;
        }
        
        // Two digits
        int twoDigit = Integer.parseInt(s.substring(i-1, i+1));
        if (twoDigit >= 10 && twoDigit <= 26) {
            curr += prev2;
        }
        
        prev2 = prev1;
        prev1 = curr;
    }
    
    return prev1;
}
```

### **Practice Problems**
- [Climbing Stairs](https://leetcode.com/problems/climbing-stairs/)
- [Decode Ways](https://leetcode.com/problems/decode-ways/)
- [Fibonacci Number](https://leetcode.com/problems/fibonacci-number/)

---

## Pattern 8: State Machine 🎰

### **Problem Type**
Problems with distinct states and transitions.

### **Recognition**
- Keywords: "buy/sell", "states", "transitions"
- Finite number of states at each step

### **Template**
```java
public int stateMachine(int[] arr) {
    int state1 = initial1;
    int state2 = initial2;
    
    for (int val : arr) {
        int newState1 = /* transition from states */;
        int newState2 = /* transition from states */;
        
        state1 = newState1;
        state2 = newState2;
    }
    
    return Math.max(state1, state2);
}
```

### **Example: Best Time to Buy/Sell Stock**
```java
public int maxProfit(int[] prices) {
    int hold = Integer.MIN_VALUE;  // State: holding stock
    int sold = 0;                  // State: not holding
    
    for (int price : prices) {
        int newHold = Math.max(hold, sold - price);  // Buy or keep
        int newSold = Math.max(sold, hold + price);  // Sell or keep
        
        hold = newHold;
        sold = newSold;
    }
    
    return sold;
}
```

### **Practice Problems**
- [Best Time to Buy and Sell Stock](https://leetcode.com/problems/best-time-to-buy-and-sell-stock/)
- [Best Time to Buy and Sell Stock III](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-iii/)
- [Best Time to Buy and Sell Stock with Cooldown](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-with-cooldown/)

---

## Pattern 9: Partition 🔪

### **Problem Type**
Partition array/string into parts with constraints.

### **Recognition**
- Keywords: "partition", "split", "break"
- Divide into subproblems with optimal solutions

### **Template**
```java
public boolean wordBreak(String s, List<String> wordDict) {
    Set<String> dict = new HashSet<>(wordDict);
    boolean[] dp = new boolean[s.length() + 1];
    dp[0] = true;
    
    for (int i = 1; i <= s.length(); i++) {
        for (int j = 0; j < i; j++) {
            if (dp[j] && dict.contains(s.substring(j, i))) {
                dp[i] = true;
                break;
            }
        }
    }
    
    return dp[s.length()];
}
```

### **Example: Palindrome Partitioning**
```java
public int minCut(String s) {
    int n = s.length();
    boolean[][] isPalin = new boolean[n][n];
    int[] dp = new int[n];
    
    // Check palindromes
    for (int i = 0; i < n; i++) {
        for (int j = 0; j <= i; j++) {
            if (s.charAt(i) == s.charAt(j) && 
                (i - j <= 2 || isPalin[j + 1][i - 1])) {
                isPalin[j][i] = true;
            }
        }
    }
    
    // Min cuts
    for (int i = 0; i < n; i++) {
        if (isPalin[0][i]) {
            dp[i] = 0;
        } else {
            dp[i] = i;  // Worst case
            for (int j = 0; j < i; j++) {
                if (isPalin[j + 1][i]) {
                    dp[i] = Math.min(dp[i], dp[j] + 1);
                }
            }
        }
    }
    
    return dp[n - 1];
}
```

### **Practice Problems**
- [Word Break](https://leetcode.com/problems/word-break/)
- [Palindrome Partitioning II](https://leetcode.com/problems/palindrome-partitioning-ii/)
- [Partition Equal Subset Sum](https://leetcode.com/problems/partition-equal-subset-sum/)

---

## Pattern 10: Kadane's Algorithm 📊

### **Problem Type**
Maximum/minimum subarray problems.

### **Recognition**
- Keywords: "maximum subarray", "contiguous"
- Find optimal contiguous subsequence

### **Template**
```java
public int maxSubArray(int[] nums) {
    int maxEndingHere = nums[0];
    int maxSoFar = nums[0];
    
    for (int i = 1; i < nums.length; i++) {
        maxEndingHere = Math.max(nums[i], 
                                 maxEndingHere + nums[i]);
        maxSoFar = Math.max(maxSoFar, maxEndingHere);
    }
    
    return maxSoFar;
}
```

### **Example: Maximum Product Subarray**
```java
public int maxProduct(int[] nums) {
    int maxProd = nums[0];
    int minProd = nums[0];
    int result = nums[0];
    
    for (int i = 1; i < nums.length; i++) {
        int temp = maxProd;
        maxProd = Math.max(nums[i], 
                  Math.max(maxProd * nums[i], minProd * nums[i]));
        minProd = Math.min(nums[i], 
                  Math.min(temp * nums[i], minProd * nums[i]));
        result = Math.max(result, maxProd);
    }
    
    return result;
}
```

### **Practice Problems**
- [Maximum Subarray](https://leetcode.com/problems/maximum-subarray/)
- [Maximum Product Subarray](https://leetcode.com/problems/maximum-product-subarray/)
- [Best Time to Buy and Sell Stock](https://leetcode.com/problems/best-time-to-buy-and-sell-stock/)

---

## 🎯 Pattern Recognition Guide

| Pattern | Keywords | Recurrence Type |
|---------|----------|----------------|
| **0/1 Knapsack** | "choose/skip", "subset" | dp[i][w] = max(take, skip) |
| **Unbounded** | "unlimited", "reuse" | dp[i] += dp[i - item] |
| **LCS** | "two strings", "common" | if match: dp[i-1][j-1]+1 |
| **LIS** | "increasing", "subsequence" | dp[i] = max(dp[j]+1) |
| **Palindrome** | "palindrome", "symmetry" | if s[i]==s[j]: dp[i+1][j-1] |
| **Grid Path** | "grid", "m×n", "paths" | dp[i][j] = dp[i-1][j] + dp[i][j-1] |
| **Fibonacci** | "ways", "decode", "climb" | dp[i] = dp[i-1] + dp[i-2] |
| **State Machine** | "buy/sell", "states" | dp[state] = transition |
| **Partition** | "partition", "split" | try all partitions |
| **Kadane's** | "maximum subarray" | max(curr, curr + prev) |

---

## 💡 Pro Tips

✅ **Recognize the pattern first** - saves 90% of thinking time  
✅ **Draw the DP table** - visualize states and transitions  
✅ **Start with brute force** - then optimize with DP  
✅ **Check for space optimization** - often possible  
✅ **Practice each pattern** - muscle memory for interviews  

---

## 🚀 What's Next?

➡️ **[Advanced DP](./03_Advanced_DP.md)** - Complex problems  
➡️ **[Practice Problems](../ProblemSets/README.md)**  

---

**🎮 Achievement Unlocked: DP Pattern Master!** 🏆

*Remember: "Master the patterns, master the interviews!"* 💎✨

