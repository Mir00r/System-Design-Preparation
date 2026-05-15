# 👉👈 Two Pointers Pattern: The Interview Hacker's Secret! 🚀

---

## 🎯 What is the Two Pointers Pattern?

The **Two Pointers** technique uses two references (indices) that move through the data structure (usually an array or linked list) in a coordinated way to solve problems **efficiently** without nested loops.

> **"Instead of checking every pair (O(n²)), move two pointers smartly (O(n))!"**

---

## 🎨 Pattern Variations

### 1️⃣ **Opposite Direction (Collision)**
```
Left starts at beginning, Right starts at end
[1, 2, 3, 4, 5, 6, 7, 8]
 ↑                     ↑
left                 right

Move towards each other until they meet
```

### 2️⃣ **Same Direction (Fast & Slow)**
```
Both start at beginning, move at different speeds
[1, 2, 3, 4, 5, 6, 7, 8]
 ↑ ↑
slow fast

Fast moves 2x speed of slow
```

### 3️⃣ **Same Direction (Fixed Distance)**
```
Maintain fixed distance k between pointers
[1, 2, 3, 4, 5, 6, 7, 8]
 ↑     ↑
left  right (distance = 2)

Move both together maintaining distance
```

---

## 🔥 Common Two Pointers Problems

### **Problem 1: Valid Palindrome** 🟢

**Pattern**: Opposite Direction

```java
public boolean isPalindrome(String s) {
    s = s.toLowerCase().replaceAll("[^a-z0-9]", "");
    
    int left = 0;
    int right = s.length() - 1;
    
    while (left < right) {
        if (s.charAt(left) != s.charAt(right)) {
            return false;
        }
        left++;
        right--;
    }
    
    return true;
}

// Example: "A man, a plan, a canal: Panama"
// After cleanup: "amanaplanacanalpanama"
// Check from both ends: a==a, m==m, a==a... ✅
```

**Visual**:
```
"racecar"
 ↑     ↑  r == r ✅
  ↑   ↑   a == a ✅
   ↑ ↑    c == c ✅
    ↑     (middle, done)
Result: true
```

---

### **Problem 2: Two Sum II (Sorted Array)** 🟢

**Pattern**: Opposite Direction

```java
public int[] twoSum(int[] numbers, int target) {
    int left = 0;
    int right = numbers.length - 1;
    
    while (left < right) {
        int sum = numbers[left] + numbers[right];
        
        if (sum == target) {
            return new int[]{left + 1, right + 1}; // 1-indexed
        } else if (sum < target) {
            left++; // Need larger sum
        } else {
            right--; // Need smaller sum
        }
    }
    
    return new int[]{-1, -1};
}

// Example: numbers = [2, 7, 11, 15], target = 9
// Step 1: 2 + 15 = 17 > 9, right--
// Step 2: 2 + 11 = 13 > 9, right--
// Step 3: 2 + 7 = 9 ✅ return [1, 2]
```

**Why O(n) instead of O(n²)?**
```
❌ Brute Force (nested loops): O(n²)
for i in range(n):
    for j in range(i+1, n):
        check i + j

✅ Two Pointers: O(n)
Use sorted property + two pointers
```

---

### **Problem 3: Container With Most Water** 🟡

**Pattern**: Opposite Direction + Greedy

```java
public int maxArea(int[] height) {
    int left = 0;
    int right = height.length - 1;
    int maxArea = 0;
    
    while (left < right) {
        int width = right - left;
        int minHeight = Math.min(height[left], height[right]);
        int area = width * minHeight;
        
        maxArea = Math.max(maxArea, area);
        
        // Move pointer with smaller height (greedy)
        if (height[left] < height[right]) {
            left++;
        } else {
            right--;
        }
    }
    
    return maxArea;
}

// Example: [1,8,6,2,5,4,8,3,7]
// Max area: min(8,7) * 8 = 56
```

**Visual**:
```
Height: [1, 8, 6, 2, 5, 4, 8, 3, 7]
Index:   0  1  2  3  4  5  6  7  8

         |       |           |
         |       |           |   |
         | |     |     | |   | | |
       | | | | | | | | | | | | | |
       0 1 2 3 4 5 6 7 8

Start: left=0, right=8
Area = min(1, 7) * 8 = 8
Move left (smaller height)

left=1, right=8
Area = min(8, 7) * 7 = 49
Move right (smaller height)

left=1, right=7
Area = min(8, 3) * 6 = 18
...continue until left >= right
```

---

### **Problem 4: Remove Duplicates from Sorted Array** 🟢

**Pattern**: Same Direction (Slow & Fast)

```java
public int removeDuplicates(int[] nums) {
    if (nums.length == 0) return 0;
    
    int slow = 0; // Position for unique elements
    
    for (int fast = 1; fast < nums.length; fast++) {
        if (nums[fast] != nums[slow]) {
            slow++;
            nums[slow] = nums[fast];
        }
    }
    
    return slow + 1; // Length of unique elements
}

// Example: [1, 1, 2, 2, 3, 4, 4]
// Result:  [1, 2, 3, 4, _, _, _]
//                      ↑ slow+1 = 4 unique elements
```

**Visual**:
```
[1, 1, 2, 2, 3, 4, 4]
 ↑  ↑                 slow=0, fast=1: 1==1, skip
 
[1, 1, 2, 2, 3, 4, 4]
 ↑     ↑              slow=0, fast=2: 1!=2, slow++, copy
 
[1, 2, 2, 2, 3, 4, 4]
    ↑     ↑           slow=1, fast=3: 2==2, skip
    
[1, 2, 2, 2, 3, 4, 4]
    ↑        ↑        slow=1, fast=4: 2!=3, slow++, copy
    
[1, 2, 3, 2, 3, 4, 4]
       ↑        ↑     slow=2, fast=5: 3!=4, slow++, copy
       
[1, 2, 3, 4, 3, 4, 4]
          ↑        ↑  slow=3, fast=6: 4==4, skip

Result: [1, 2, 3, 4, ...]
```

---

### **Problem 5: 3Sum** 🟡

**Pattern**: Sort + Two Pointers (Combination)

```java
public List<List<Integer>> threeSum(int[] nums) {
    List<List<Integer>> result = new ArrayList<>();
    Arrays.sort(nums); // MUST sort first
    
    for (int i = 0; i < nums.length - 2; i++) {
        // Skip duplicates for first number
        if (i > 0 && nums[i] == nums[i - 1]) continue;
        
        int left = i + 1;
        int right = nums.length - 1;
        int target = -nums[i];
        
        while (left < right) {
            int sum = nums[left] + nums[right];
            
            if (sum == target) {
                result.add(Arrays.asList(nums[i], nums[left], nums[right]));
                
                // Skip duplicates
                while (left < right && nums[left] == nums[left + 1]) left++;
                while (left < right && nums[right] == nums[right - 1]) right--;
                
                left++;
                right--;
            } else if (sum < target) {
                left++;
            } else {
                right--;
            }
        }
    }
    
    return result;
}

// Example: [-1, 0, 1, 2, -1, -4]
// After sort: [-4, -1, -1, 0, 1, 2]
// Result: [[-1, -1, 2], [-1, 0, 1]]
```

**Visual**:
```
Sorted: [-4, -1, -1, 0, 1, 2]

i=0 (nums[i]=-4), target=4:
         -1  -1   0   1   2
          ↑               ↑  sum=-1+2=1 < 4, left++
          
i=1 (nums[i]=-1), target=1:
              -1   0   1   2
               ↑           ↑  sum=-1+2=1 ✅ Found!
               
         Skip duplicate -1
              
              0   1   2
              ↑       ↑  sum=0+2=2 > 1, right--
              ↑   ↑      sum=0+1=1 ✅ Found!
```

---

### **Problem 6: Linked List Cycle (Floyd's Algorithm)** 🟢

**Pattern**: Fast & Slow Pointers

```java
public boolean hasCycle(ListNode head) {
    if (head == null) return false;
    
    ListNode slow = head;
    ListNode fast = head;
    
    while (fast != null && fast.next != null) {
        slow = slow.next;      // Move 1 step
        fast = fast.next.next; // Move 2 steps
        
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
1 → 2 → 3 → 4 → 5 → null
slow: 1, 2, 3, 4, 5, null
fast: 1, 3, 5, null
Never meet → No cycle

Cycle List:
1 → 2 → 3 → 4 → 5
         ↑         ↓
         8 ← 7 ← 6

slow: 1, 2, 3, 4, 5, 6, 7, 8, 3...
fast: 1, 3, 5, 7, 3, 5, 7, 3...
Eventually meet → Cycle exists!
```

---

### **Problem 7: Remove Nth Node From End** 🟡

**Pattern**: Fixed Distance

```java
public ListNode removeNthFromEnd(ListNode head, int n) {
    ListNode dummy = new ListNode(0);
    dummy.next = head;
    
    ListNode first = dummy;
    ListNode second = dummy;
    
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

**Visual**:
```
Remove 2nd from end in: 1 → 2 → 3 → 4 → 5

Step 1: Create gap of n+1 (3)
dummy → 1 → 2 → 3 → 4 → 5 → null
  ↑              ↑
second         first

Step 2: Move both together
dummy → 1 → 2 → 3 → 4 → 5 → null
              ↑              ↑
           second         first

Step 3: second.next is node to remove
Remove 4: second.next = second.next.next
Result: 1 → 2 → 3 → 5
```

---

### **Problem 8: Trapping Rain Water** 🔴

**Pattern**: Opposite Direction + Max Tracking

```java
public int trap(int[] height) {
    if (height.length == 0) return 0;
    
    int left = 0;
    int right = height.length - 1;
    int leftMax = 0;
    int rightMax = 0;
    int water = 0;
    
    while (left < right) {
        if (height[left] < height[right]) {
            if (height[left] >= leftMax) {
                leftMax = height[left];
            } else {
                water += leftMax - height[left];
            }
            left++;
        } else {
            if (height[right] >= rightMax) {
                rightMax = height[right];
            } else {
                water += rightMax - height[right];
            }
            right--;
        }
    }
    
    return water;
}
```

**Visual**:
```
Height: [0,1,0,2,1,0,1,3,2,1,2,1]

         ■
     ■~~~■■~~~■
 ■~~~■■■■■■■■■
─■■■■■■■■■■■■─

Water trapped (~ areas):
At index 2: min(1, 3) - 0 = 1
At index 4: min(2, 3) - 1 = 1
At index 5: min(2, 3) - 0 = 2
At index 6: min(2, 3) - 1 = 1
At index 9: min(2, 2) - 1 = 1

Total: 6 units
```

---

## 🎯 Two Pointers Decision Tree

```
Choose Two Pointers When:
│
├─ Array is SORTED?
│  ├─ YES → Two Sum, 3Sum (Opposite Direction)
│  └─ NO → Can you SORT it?
│         ├─ YES → Sort first, then Two Pointers
│         └─ NO → Consider other patterns
│
├─ Need to REMOVE/REPLACE in-place?
│  └─ YES → Slow & Fast Pointers
│
├─ Working with LINKED LIST?
│  ├─ Detect cycle → Fast & Slow (Floyd's)
│  └─ Remove nth from end → Fixed Distance
│
└─ Find PAIRS with condition?
   └─ YES → Opposite Direction
```

---

## 🏢 Real-World Applications

### 🔷 **Merge Sort (External Merge)**

```java
public int[] mergeSortedArrays(int[] arr1, int[] arr2) {
    int[] result = new int[arr1.length + arr2.length];
    int i = 0, j = 0, k = 0;
    
    while (i < arr1.length && j < arr2.length) {
        if (arr1[i] <= arr2[j]) {
            result[k++] = arr1[i++];
        } else {
            result[k++] = arr2[j++];
        }
    }
    
    while (i < arr1.length) result[k++] = arr1[i++];
    while (j < arr2.length) result[k++] = arr2[j++];
    
    return result;
}
```

### 🔷 **Partitioning (Quick Sort)**

```java
public int partition(int[] arr, int low, int high) {
    int pivot = arr[high];
    int i = low - 1;
    
    for (int j = low; j < high; j++) {
        if (arr[j] < pivot) {
            i++;
            swap(arr, i, j);
        }
    }
    
    swap(arr, i + 1, high);
    return i + 1;
}
```

---

## 🧩 Practice Problem Set

### 🟢 **Easy** (Master These First)
1. Valid Palindrome (LeetCode #125)
2. Remove Duplicates from Sorted Array (LeetCode #26)
3. Merge Sorted Array (LeetCode #88)
4. Move Zeroes (LeetCode #283)
5. Reverse String (LeetCode #344)
6. Two Sum II (LeetCode #167)
7. Linked List Cycle (LeetCode #141)

### 🟡 **Medium** (Interview Common)
8. 3Sum (LeetCode #15)
9. Container With Most Water (LeetCode #11)
10. Remove Nth Node From End (LeetCode #19)
11. Sort Colors (Dutch Flag) (LeetCode #75)
12. Find Duplicate Number (LeetCode #287)
13. Partition List (LeetCode #86)
14. Rotate Array (LeetCode #189)

### 🔴 **Hard** (Advanced)
15. Trapping Rain Water (LeetCode #42)
16. Minimum Window Substring (LeetCode #76)
17. Longest Substring with K Distinct (LeetCode #340)

---

## 💡 Pro Tips

### ✅ **When to Use Two Pointers**
1. Array/List is **sorted** (or can be sorted)
2. Need to find **pairs/triplets** with target sum
3. **In-place** modification required
4. **Palindrome** checking
5. **Linked list** cycle detection
6. **Merging** sorted arrays/lists

### ❌ **When NOT to Use**
1. Need to track **all elements** (use hash map)
2. Array is **unsorted** and can't be sorted
3. Need **frequency counting**
4. Problem requires **backtracking**

---

## 🎯 Key Takeaways

✅ Two Pointers reduces **O(n²) → O(n)**  
✅ **Three patterns**: Opposite, Fast-Slow, Fixed Distance  
✅ Often requires **sorted** input  
✅ Perfect for **pairs, palindromes, cycles**  
✅ **Space efficient**: O(1) extra space  

---

## 🚀 What's Next?

➡️ **[Sliding Window Pattern](./04_SlidingWindow_Pattern.md)**  
➡️ **[Practice Problems](../ProblemSets/README.md)**

---

**🎮 Achievement Unlocked: Two Pointers Master!** 🏆

*Remember: "Two pointers, twice the efficiency!"* 👉👈

