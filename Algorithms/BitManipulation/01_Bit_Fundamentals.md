# 🔢 Bit Manipulation: The Power of Binary! ⚡

> **"Think in bits, solve in milliseconds!"**

Master **bit manipulation** - the low-level technique that makes code blazingly fast! Essential for optimization, cryptography, graphics, and competitive programming. Unlock the magic of XOR, bit masks, and bitwise tricks! 🚀

---

## 📋 Table of Contents

1. [Bit Fundamentals](#-bit-fundamentals)
2. [Basic Operations](#-basic-operations)
3. [Common Patterns](#-common-patterns)
4. [XOR Tricks](#-xor-tricks)
5. [Bit Masking](#-bit-masking)
6. [Practice Problems](#-practice-problems)

---

## 🎯 Bit Fundamentals

### **Binary Representation**

```java
// Decimal to Binary
int n = 13;  // Binary: 1101

// Powers of 2:
// 2³ 2² 2¹ 2⁰
//  1  1  0  1
// 8 + 4 + 0 + 1 = 13

// Java methods
Integer.toBinaryString(13);  // "1101"
Integer.parseInt("1101", 2); // 13
```

### **Bit Positions** (0-indexed from right)

```
Number: 13 = 1101
Position:    3210
             ↑  ↑
         Bit 3  Bit 0
```

---

## ⚙️ Basic Operations

### **1. AND (&)** - Both bits 1 → 1

```java
int a = 5;  // 0101
int b = 3;  // 0011
int c = a & b;  // 0001 = 1

// Use case: Check if bit is set
boolean isBitSet(int num, int pos) {
    return (num & (1 << pos)) != 0;
}
```

### **2. OR (|)** - Any bit 1 → 1

```java
int a = 5;  // 0101
int b = 3;  // 0011
int c = a | b;  // 0111 = 7

// Use case: Set bit
int setBit(int num, int pos) {
    return num | (1 << pos);
}
```

### **3. XOR (^)** - Different bits → 1

```java
int a = 5;  // 0101
int b = 3;  // 0011
int c = a ^ b;  // 0110 = 6

// Magic properties:
a ^ a = 0     // Same numbers cancel
a ^ 0 = a     // Zero is identity
a ^ b ^ a = b // Commutative & associative
```

### **4. NOT (~)** - Flip all bits

```java
int a = 5;   // 00000101
int b = ~a;  // 11111010 = -6 (two's complement)
```

### **5. Left Shift (<<)** - Multiply by 2

```java
int a = 5;      // 0101
int b = a << 1; // 1010 = 10

// Pattern: n << k = n * 2^k
5 << 2 = 5 * 4 = 20
```

### **6. Right Shift (>>)** - Divide by 2

```java
int a = 20;     // 10100
int b = a >> 2; // 00101 = 5

// Pattern: n >> k = n / 2^k
20 >> 2 = 20 / 4 = 5
```

---

## 🎨 Common Patterns

### **1. Check if Power of 2**

```java
boolean isPowerOfTwo(int n) {
    return n > 0 && (n & (n - 1)) == 0;
}

// Why?
// 8:  1000
// 7:  0111
// &:  0000 ✅

// 6:  0110
// 5:  0101
// &:  0100 ❌ (not 0)
```

### **2. Count Set Bits (Brian Kernighan's)**

```java
int countBits(int n) {
    int count = 0;
    while (n > 0) {
        n &= (n - 1);  // Remove rightmost 1
        count++;
    }
    return count;
}

// Example: n = 13 (1101)
// Step 1: 1101 & 1100 = 1100 (count=1)
// Step 2: 1100 & 1011 = 1000 (count=2)
// Step 3: 1000 & 0111 = 0000 (count=3)
// Result: 3 ✅
```

### **3. Get/Set/Clear Bit**

```java
// Get bit at position i
int getBit(int num, int i) {
    return (num >> i) & 1;
}

// Set bit at position i
int setBit(int num, int i) {
    return num | (1 << i);
}

// Clear bit at position i
int clearBit(int num, int i) {
    return num & ~(1 << i);
}

// Toggle bit at position i
int toggleBit(int num, int i) {
    return num ^ (1 << i);
}
```

### **4. Isolate Rightmost 1**

```java
int rightmost1(int n) {
    return n & -n;
}

// Example: n = 12 (1100)
// -n = -12 (two's complement)
// In binary (8-bit): 11110100
// n & -n = 00000100 = 4 ✅
```

---

## ⚡ XOR Tricks

### **1. Swap Without Temp**

```java
void swap(int a, int b) {
    a = a ^ b;
    b = a ^ b;  // b = a ^ b ^ b = a
    a = a ^ b;  // a = a ^ b ^ a = b
}
```

### **2. Find Single Number**

**Problem**: Array has duplicates except one. Find it.

```java
public int singleNumber(int[] nums) {
    int result = 0;
    for (int num : nums) {
        result ^= num;  // Duplicates cancel out!
    }
    return result;
}

// Example: [2, 3, 2, 4, 4]
// 2 ^ 3 ^ 2 ^ 4 ^ 4
// = (2 ^ 2) ^ (4 ^ 4) ^ 3
// = 0 ^ 0 ^ 3
// = 3 ✅
```

### **3. Find Two Missing Numbers**

```java
public int[] findTwoMissing(int[] nums, int n) {
    int xor = 0;
    
    // XOR all numbers
    for (int num : nums) xor ^= num;
    for (int i = 1; i <= n; i++) xor ^= i;
    
    // Now xor = missing1 ^ missing2
    
    // Find rightmost set bit
    int rightBit = xor & -xor;
    
    int num1 = 0, num2 = 0;
    
    // Divide into two groups
    for (int num : nums) {
        if ((num & rightBit) == 0) {
            num1 ^= num;
        } else {
            num2 ^= num;
        }
    }
    
    for (int i = 1; i <= n; i++) {
        if ((i & rightBit) == 0) {
            num1 ^= i;
        } else {
            num2 ^= i;
        }
    }
    
    return new int[]{num1, num2};
}
```

---

## 🎭 Bit Masking

### **Concept**: Use bits to represent states/subsets

### **1. Generate All Subsets**

```java
public List<List<Integer>> subsets(int[] nums) {
    List<List<Integer>> result = new ArrayList<>();
    int n = nums.length;
    int totalSubsets = 1 << n;  // 2^n
    
    for (int mask = 0; mask < totalSubsets; mask++) {
        List<Integer> subset = new ArrayList<>();
        
        for (int i = 0; i < n; i++) {
            if ((mask & (1 << i)) != 0) {
                subset.add(nums[i]);
            }
        }
        
        result.add(subset);
    }
    
    return result;
}

// Example: [1,2,3]
// Masks: 000, 001, 010, 011, 100, 101, 110, 111
// Subsets: [], [1], [2], [1,2], [3], [1,3], [2,3], [1,2,3]
```

### **2. Check All Bits Set in Range**

```java
boolean allBitsSet(int num, int l, int r) {
    int mask = ((1 << (r - l + 1)) - 1) << l;
    return (num & mask) == mask;
}

// Example: Check bits 2-4 in 11111110
// mask = ((1<<3)-1)<<2 = 111<<2 = 11100
// num & mask = 11100 = mask ✅
```

### **3. Traveling Salesman (DP with Bitmask)**

```java
public int tsp(int[][] dist, int mask, int pos, int[][] dp) {
    int n = dist.length;
    
    if (mask == (1 << n) - 1) {
        return dist[pos][0];  // Return to start
    }
    
    if (dp[mask][pos] != -1) {
        return dp[mask][pos];
    }
    
    int minCost = Integer.MAX_VALUE;
    
    for (int city = 0; city < n; city++) {
        if ((mask & (1 << city)) == 0) {  // Not visited
            int newMask = mask | (1 << city);
            int cost = dist[pos][city] + tsp(dist, newMask, city, dp);
            minCost = Math.min(minCost, cost);
        }
    }
    
    return dp[mask][pos] = minCost;
}
```

---

## 🏢 Real-World Applications

### **Cryptography** 🔐
- XOR encryption
- Hash functions
- Secure random numbers

### **Graphics** 🎨
- Color manipulation (RGB)
- Image compression
- Pixel operations

### **Networking** 🌐
- IP address manipulation
- Subnet masks
- Packet filtering

### **Databases** 💾
- Bloom filters
- Bitmap indexes
- Permission flags

---

## 🧩 Practice Problems

### 🟢 **Easy**
1. [Number of 1 Bits](https://leetcode.com/problems/number-of-1-bits/)
2. [Power of Two](https://leetcode.com/problems/power-of-two/)
3. [Reverse Bits](https://leetcode.com/problems/reverse-bits/)
4. [Missing Number](https://leetcode.com/problems/missing-number/)

### 🟡 **Medium**
5. [Single Number](https://leetcode.com/problems/single-number/)
6. [Single Number II](https://leetcode.com/problems/single-number-ii/)
7. [Bitwise AND of Numbers Range](https://leetcode.com/problems/bitwise-and-of-numbers-range/)
8. [Counting Bits](https://leetcode.com/problems/counting-bits/)
9. [Sum of Two Integers](https://leetcode.com/problems/sum-of-two-integers/)

### 🔴 **Hard**
10. [Maximum XOR of Two Numbers](https://leetcode.com/problems/maximum-xor-of-two-numbers-in-an-array/)
11. [Smallest Sufficient Team](https://leetcode.com/problems/smallest-sufficient-team/)

---

## 💡 Quick Reference

### **Useful Bit Tricks**

```java
// Check even/odd
boolean isEven(int n) { return (n & 1) == 0; }

// Multiply by 2^k
int multiplyBy2Power(int n, int k) { return n << k; }

// Divide by 2^k
int divideBy2Power(int n, int k) { return n >> k; }

// Toggle case (uppercase ↔ lowercase)
char toggleCase(char c) { return (char)(c ^ 32); }

// Get absolute value
int abs(int n) {
    int mask = n >> 31;  // All 1s if negative, 0 if positive
    return (n + mask) ^ mask;
}

// Min/Max without branching
int min(int a, int b) {
    return b ^ ((a ^ b) & -(a < b ? 1 : 0));
}

// Check if opposite signs
boolean oppositeSigns(int a, int b) {
    return (a ^ b) < 0;
}
```

---

## ⚠️ Common Pitfalls

### **1. Signed vs Unsigned**
```java
// >> is arithmetic shift (preserves sign)
-8 >> 1 = -4  // 11111000 → 11111100

// >>> is logical shift (fills with 0)
-8 >>> 1 = 2147483644  // 11111000 → 01111100
```

### **2. Operator Precedence**
```java
❌ if (n & 1 == 0)  // Wrong! == has higher precedence
✅ if ((n & 1) == 0)  // Correct
```

---

## 🎯 Key Takeaways

1. **Bitwise operations** are O(1) and super fast
2. **XOR magic**: a ^ a = 0, a ^ 0 = a
3. **n & (n-1)** removes rightmost 1 bit
4. **Bit masking** represents subsets efficiently
5. **Think binary** for optimization problems

---

## 🚀 What's Next?

➡️ **[Practice Problems](../ProblemSets/README.md)**  
➡️ **[Main Algorithm Hub](../README.md)**  

---

**🎮 Achievement Unlocked: Bit Master!** 🏆

*Remember: "10 types of people: those who understand binary, and those who don't!"* 🔢✨

