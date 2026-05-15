# 🔄 Sorting Fundamentals: Master the Art of Ordering! 🎯

> **"Sorting is where it all begins. Master sorting, and you'll understand the foundation of all algorithms!"**

Welcome to the **most comprehensive sorting tutorial** designed to make you a **sorting expert**! We'll cover all major sorting algorithms with visual explanations, Java code, and real-world applications. 🚀

---

## 📋 Table of Contents

1. [What is Sorting?](#-what-is-sorting)
2. [Why Learn Sorting?](#-why-learn-sorting)
3. [Simple Sorts (O(n²))](#-simple-sorts-on)
4. [Efficient Sorts (O(n log n))](#-efficient-sorts-on-log-n)
5. [Specialized Sorts](#-specialized-sorts)
6. [Comparison Chart](#-complete-comparison-chart)
7. [Practice Problems](#-practice-problems)

---

## 🎯 What is Sorting?

**Sorting** = Arranging data in a specific order (ascending/descending)

```
Unsorted: [64, 34, 25, 12, 22, 11, 90]
Sorted:   [11, 12, 22, 25, 34, 64, 90]
```

### **Key Concepts**

✅ **Stable Sorting**: Equal elements maintain relative order  
✅ **In-Place Sorting**: Uses O(1) extra space  
✅ **Comparison-Based**: Compares elements to sort  
✅ **Adaptive**: Performs better on partially sorted data  

---

## 💡 Why Learn Sorting?

### **Real-World Applications**

🔷 **E-Commerce**: Product listings by price/rating  
🔷 **Social Media**: Posts by timestamp  
🔷 **Databases**: Indexing and query optimization  
🔷 **File Systems**: Organize files by name/date  
🔷 **Search Engines**: Rank search results  

### **Interview Importance**

📊 **20% of coding interviews** involve sorting  
📊 Foundation for **binary search**, **merge operations**  
📊 Tests understanding of **time/space complexity**  

---

## 🐢 Simple Sorts (O(n²))

### **1. Bubble Sort** 🫧

**Concept**: Repeatedly swap adjacent elements if they're in wrong order

**Visual**:
```
Pass 1: [64, 34, 25, 12, 22, 11, 90]
        [34, 64, 25, 12, 22, 11, 90]  (swap 64,34)
        [34, 25, 64, 12, 22, 11, 90]  (swap 64,25)
        [34, 25, 12, 64, 22, 11, 90]  (swap 64,12)
        [34, 25, 12, 22, 64, 11, 90]  (swap 64,22)
        [34, 25, 12, 22, 11, 64, 90]  (swap 64,11)
        [34, 25, 12, 22, 11, 64, 90]  (90 in position!)

Pass 2: Continue until sorted...
```

**Java Implementation**:
```java
public class BubbleSort {
    public static void bubbleSort(int[] arr) {
        int n = arr.length;
        
        for (int i = 0; i < n - 1; i++) {
            boolean swapped = false;
            
            // Last i elements are already sorted
            for (int j = 0; j < n - 1 - i; j++) {
                if (arr[j] > arr[j + 1]) {
                    // Swap
                    int temp = arr[j];
                    arr[j] = arr[j + 1];
                    arr[j + 1] = temp;
                    swapped = true;
                }
            }
            
            // Optimization: If no swap, array is sorted
            if (!swapped) break;
        }
    }
    
    public static void main(String[] args) {
        int[] arr = {64, 34, 25, 12, 22, 11, 90};
        bubbleSort(arr);
        System.out.println(Arrays.toString(arr));
        // Output: [11, 12, 22, 25, 34, 64, 90]
    }
}
```

**Complexity**:
- ⏰ **Time**: Best O(n), Average O(n²), Worst O(n²)
- 💾 **Space**: O(1)
- ✅ **Stable**: Yes
- 🎯 **When to Use**: Teaching, nearly sorted data

---

### **2. Selection Sort** 🎯

**Concept**: Find minimum element, place it at beginning

**Visual**:
```
Initial: [64, 25, 12, 22, 11]
          ↑
         Find min (11), swap with first

Step 1:  [11, 25, 12, 22, 64]
              ↑
             Find min in rest (12), swap

Step 2:  [11, 12, 25, 22, 64]
                  ↑
                 Find min (22), swap

Step 3:  [11, 12, 22, 25, 64]
                      ↑
                     Already sorted!
```

**Java Implementation**:
```java
public class SelectionSort {
    public static void selectionSort(int[] arr) {
        int n = arr.length;
        
        for (int i = 0; i < n - 1; i++) {
            // Find minimum in unsorted portion
            int minIdx = i;
            for (int j = i + 1; j < n; j++) {
                if (arr[j] < arr[minIdx]) {
                    minIdx = j;
                }
            }
            
            // Swap minimum with first unsorted element
            int temp = arr[i];
            arr[i] = arr[minIdx];
            arr[minIdx] = temp;
        }
    }
}
```

**Complexity**:
- ⏰ **Time**: Best O(n²), Average O(n²), Worst O(n²)
- 💾 **Space**: O(1)
- ❌ **Stable**: No (can be made stable)
- 🎯 **When to Use**: Memory writes expensive

---

### **3. Insertion Sort** 📥

**Concept**: Build sorted array one element at a time (like sorting cards)

**Visual**:
```
Array: [12, 11, 13, 5, 6]

Step 1: [12] | 11, 13, 5, 6    (12 is "sorted")
Step 2: [11, 12] | 13, 5, 6    (Insert 11 before 12)
Step 3: [11, 12, 13] | 5, 6    (13 already in place)
Step 4: [5, 11, 12, 13] | 6    (Insert 5 at beginning)
Step 5: [5, 6, 11, 12, 13]     (Insert 6 after 5)
```

**Java Implementation**:
```java
public class InsertionSort {
    public static void insertionSort(int[] arr) {
        int n = arr.length;
        
        for (int i = 1; i < n; i++) {
            int key = arr[i];
            int j = i - 1;
            
            // Move elements greater than key one position ahead
            while (j >= 0 && arr[j] > key) {
                arr[j + 1] = arr[j];
                j--;
            }
            
            arr[j + 1] = key;
        }
    }
}
```

**Complexity**:
- ⏰ **Time**: Best O(n), Average O(n²), Worst O(n²)
- 💾 **Space**: O(1)
- ✅ **Stable**: Yes
- 🎯 **When to Use**: Small arrays, nearly sorted data

---

## 🚀 Efficient Sorts (O(n log n))

### **4. Merge Sort** 🔀

**Concept**: Divide and Conquer - Split, sort, merge

**Visual**:
```
                    [38, 27, 43, 3, 9, 82, 10]
                    /                        \
            [38, 27, 43, 3]              [9, 82, 10]
            /            \                /         \
       [38, 27]       [43, 3]        [9, 82]      [10]
        /    \         /    \         /    \
      [38]  [27]     [43]  [3]      [9]  [82]     [10]
        \    /         \    /         \    /
       [27, 38]       [3, 43]        [9, 82]      [10]
            \            /                \         /
          [3, 27, 38, 43]              [9, 10, 82]
                    \                    /
                  [3, 9, 10, 27, 38, 43, 82]
```

**Java Implementation**:
```java
public class MergeSort {
    public static void mergeSort(int[] arr, int left, int right) {
        if (left < right) {
            int mid = left + (right - left) / 2;
            
            // Sort first and second halves
            mergeSort(arr, left, mid);
            mergeSort(arr, mid + 1, right);
            
            // Merge the sorted halves
            merge(arr, left, mid, right);
        }
    }
    
    private static void merge(int[] arr, int left, int mid, int right) {
        // Calculate sizes
        int n1 = mid - left + 1;
        int n2 = right - mid;
        
        // Create temp arrays
        int[] L = new int[n1];
        int[] R = new int[n2];
        
        // Copy data
        for (int i = 0; i < n1; i++)
            L[i] = arr[left + i];
        for (int j = 0; j < n2; j++)
            R[j] = arr[mid + 1 + j];
        
        // Merge
        int i = 0, j = 0, k = left;
        while (i < n1 && j < n2) {
            if (L[i] <= R[j]) {
                arr[k++] = L[i++];
            } else {
                arr[k++] = R[j++];
            }
        }
        
        // Copy remaining
        while (i < n1) arr[k++] = L[i++];
        while (j < n2) arr[k++] = R[j++];
    }
    
    public static void main(String[] args) {
        int[] arr = {38, 27, 43, 3, 9, 82, 10};
        mergeSort(arr, 0, arr.length - 1);
        System.out.println(Arrays.toString(arr));
        // Output: [3, 9, 10, 27, 38, 43, 82]
    }
}
```

**Complexity**:
- ⏰ **Time**: O(n log n) always
- 💾 **Space**: O(n)
- ✅ **Stable**: Yes
- 🎯 **When to Use**: Need guaranteed O(n log n), stability required

---

### **5. Quick Sort** ⚡

**Concept**: Pick pivot, partition around it, recursively sort

**Visual**:
```
Array: [10, 80, 30, 90, 40, 50, 70]
Pivot: 70 (last element)

Partition:
[10, 30, 40, 50] | 70 | [80, 90]
       ↓                    ↓
   Recursively           Recursively
      sort                 sort

Final: [10, 30, 40, 50, 70, 80, 90]
```

**Java Implementation**:
```java
public class QuickSort {
    public static void quickSort(int[] arr, int low, int high) {
        if (low < high) {
            // Partition and get pivot index
            int pi = partition(arr, low, high);
            
            // Recursively sort before and after partition
            quickSort(arr, low, pi - 1);
            quickSort(arr, pi + 1, high);
        }
    }
    
    private static int partition(int[] arr, int low, int high) {
        int pivot = arr[high];
        int i = low - 1; // Index of smaller element
        
        for (int j = low; j < high; j++) {
            if (arr[j] < pivot) {
                i++;
                // Swap arr[i] and arr[j]
                int temp = arr[i];
                arr[i] = arr[j];
                arr[j] = temp;
            }
        }
        
        // Swap pivot to correct position
        int temp = arr[i + 1];
        arr[i + 1] = arr[high];
        arr[high] = temp;
        
        return i + 1;
    }
    
    public static void main(String[] args) {
        int[] arr = {10, 80, 30, 90, 40, 50, 70};
        quickSort(arr, 0, arr.length - 1);
        System.out.println(Arrays.toString(arr));
        // Output: [10, 30, 40, 50, 70, 80, 90]
    }
}
```

**Complexity**:
- ⏰ **Time**: Best O(n log n), Average O(n log n), Worst O(n²)
- 💾 **Space**: O(log n)
- ❌ **Stable**: No
- 🎯 **When to Use**: General purpose, fastest in practice

---

### **6. Heap Sort** 🏔️

**Concept**: Build max heap, repeatedly extract maximum

**Visual**:
```
Array: [4, 10, 3, 5, 1]

Build Max Heap:
       10
      /  \
     5    3
    / \
   4   1

Extract Max (10): [1, 5, 3, 4] | 10
Heapify:
        5
       / \
      4   3
     /
    1

Continue until sorted: [1, 3, 4, 5, 10]
```

**Java Implementation**:
```java
public class HeapSort {
    public static void heapSort(int[] arr) {
        int n = arr.length;
        
        // Build max heap
        for (int i = n / 2 - 1; i >= 0; i--) {
            heapify(arr, n, i);
        }
        
        // Extract elements from heap one by one
        for (int i = n - 1; i > 0; i--) {
            // Move current root to end
            int temp = arr[0];
            arr[0] = arr[i];
            arr[i] = temp;
            
            // Heapify reduced heap
            heapify(arr, i, 0);
        }
    }
    
    private static void heapify(int[] arr, int n, int i) {
        int largest = i;
        int left = 2 * i + 1;
        int right = 2 * i + 2;
        
        if (left < n && arr[left] > arr[largest])
            largest = left;
        
        if (right < n && arr[right] > arr[largest])
            largest = right;
        
        if (largest != i) {
            int temp = arr[i];
            arr[i] = arr[largest];
            arr[largest] = temp;
            
            heapify(arr, n, largest);
        }
    }
}
```

**Complexity**:
- ⏰ **Time**: O(n log n) always
- 💾 **Space**: O(1)
- ❌ **Stable**: No
- 🎯 **When to Use**: In-place O(n log n) needed

---

## 🎨 Specialized Sorts

### **7. Counting Sort** 🔢

**Concept**: Count occurrences, calculate positions (for integers)

**Visual**:
```
Input: [1, 4, 1, 2, 7, 5, 2]
Range: 1 to 7

Count array:
Index:  0  1  2  3  4  5  6  7
Count: [0, 2, 2, 0, 1, 1, 0, 1]
        ↓  ↓  ↓     ↓  ↓     ↓
       none,1,1,2,2,4,5,7

Output: [1, 1, 2, 2, 4, 5, 7]
```

**Java Implementation**:
```java
public class CountingSort {
    public static void countingSort(int[] arr) {
        if (arr.length == 0) return;
        
        // Find range
        int max = Arrays.stream(arr).max().getAsInt();
        int min = Arrays.stream(arr).min().getAsInt();
        int range = max - min + 1;
        
        // Count occurrences
        int[] count = new int[range];
        for (int num : arr) {
            count[num - min]++;
        }
        
        // Modify count array for positions
        for (int i = 1; i < range; i++) {
            count[i] += count[i - 1];
        }
        
        // Build output array
        int[] output = new int[arr.length];
        for (int i = arr.length - 1; i >= 0; i--) {
            output[count[arr[i] - min] - 1] = arr[i];
            count[arr[i] - min]--;
        }
        
        // Copy to original array
        System.arraycopy(output, 0, arr, 0, arr.length);
    }
}
```

**Complexity**:
- ⏰ **Time**: O(n + k) where k is range
- 💾 **Space**: O(k)
- ✅ **Stable**: Yes
- 🎯 **When to Use**: Small integer range

---

### **8. Radix Sort** 📊

**Concept**: Sort digit by digit (from least to most significant)

**Visual**:
```
Input: [170, 45, 75, 90, 802, 24, 2, 66]

Sort by ones place:
[170, 90, 802, 2, 24, 45, 75, 66]

Sort by tens place:
[802, 2, 24, 45, 66, 170, 75, 90]

Sort by hundreds place:
[2, 24, 45, 66, 75, 90, 170, 802]
```

**Java Implementation**:
```java
public class RadixSort {
    public static void radixSort(int[] arr) {
        // Find maximum to know number of digits
        int max = Arrays.stream(arr).max().getAsInt();
        
        // Do counting sort for every digit
        for (int exp = 1; max / exp > 0; exp *= 10) {
            countingSortByDigit(arr, exp);
        }
    }
    
    private static void countingSortByDigit(int[] arr, int exp) {
        int n = arr.length;
        int[] output = new int[n];
        int[] count = new int[10];
        
        // Count occurrences
        for (int num : arr) {
            count[(num / exp) % 10]++;
        }
        
        // Change count to actual positions
        for (int i = 1; i < 10; i++) {
            count[i] += count[i - 1];
        }
        
        // Build output array
        for (int i = n - 1; i >= 0; i--) {
            int digit = (arr[i] / exp) % 10;
            output[count[digit] - 1] = arr[i];
            count[digit]--;
        }
        
        System.arraycopy(output, 0, arr, 0, n);
    }
}
```

**Complexity**:
- ⏰ **Time**: O(d × n) where d is digits
- 💾 **Space**: O(n + k)
- ✅ **Stable**: Yes
- 🎯 **When to Use**: Fixed-size integers

---

## 📊 Complete Comparison Chart

| Algorithm | Best | Average | Worst | Space | Stable | In-Place |
|-----------|------|---------|-------|-------|--------|----------|
| Bubble | O(n) | O(n²) | O(n²) | O(1) | ✅ | ✅ |
| Selection | O(n²) | O(n²) | O(n²) | O(1) | ❌ | ✅ |
| Insertion | O(n) | O(n²) | O(n²) | O(1) | ✅ | ✅ |
| Merge | O(n log n) | O(n log n) | O(n log n) | O(n) | ✅ | ❌ |
| Quick | O(n log n) | O(n log n) | O(n²) | O(log n) | ❌ | ✅ |
| Heap | O(n log n) | O(n log n) | O(n log n) | O(1) | ❌ | ✅ |
| Counting | O(n+k) | O(n+k) | O(n+k) | O(k) | ✅ | ❌ |
| Radix | O(d×n) | O(d×n) | O(d×n) | O(n+k) | ✅ | ❌ |

---

## 🏢 Real-World Applications

### **Google** 🔍
- Search result ranking: **Modified Quick Sort**
- PageRank sorting: **Merge Sort** (stability)

### **Amazon** 📦
- Product listings: **Quick Sort** (fast)
- Order processing: **Radix Sort** (order IDs)

### **Netflix** 🎬
- Content recommendations: **Merge Sort** (stable ranking)

### **Databases** 💾
- Index building: **Merge Sort**
- Query results: **Quick Sort**

---

## 🧩 Practice Problems

### 🟢 **Easy** (Build Foundation)
1. [Sort an Array](https://leetcode.com/problems/sort-an-array/) - Implement sorting
2. [Sort Colors](https://leetcode.com/problems/sort-colors/) - Dutch flag problem
3. [Squares of Sorted Array](https://leetcode.com/problems/squares-of-a-sorted-array/) - Two pointers
4. [Merge Sorted Array](https://leetcode.com/problems/merge-sorted-array/) - In-place merge
5. [Contains Duplicate](https://leetcode.com/problems/contains-duplicate/) - Sort first

### 🟡 **Medium** (Interview Common)
6. [Sort List](https://leetcode.com/problems/sort-list/) - Merge sort on linked list
7. [Merge Intervals](https://leetcode.com/problems/merge-intervals/) - Sort + merge
8. [Insert Interval](https://leetcode.com/problems/insert-interval/) - Sorted insertion
9. [Meeting Rooms II](https://leetcode.com/problems/meeting-rooms-ii/) - Sort by start time
10. [Wiggle Sort II](https://leetcode.com/problems/wiggle-sort-ii/) - Advanced sorting

### 🔴 **Hard** (Advanced Mastery)
11. [Count of Smaller After Self](https://leetcode.com/problems/count-of-smaller-numbers-after-self/) - Merge sort variant
12. [Reverse Pairs](https://leetcode.com/problems/reverse-pairs/) - Modified merge sort
13. [Maximum Gap](https://leetcode.com/problems/maximum-gap/) - Bucket sort

---

## 💡 Pro Tips

✅ **Choose Quick Sort** for general purpose (fastest in practice)  
✅ **Choose Merge Sort** when stability is required  
✅ **Choose Insertion Sort** for small/nearly sorted arrays  
✅ **Choose Counting/Radix Sort** for integers with small range  
✅ **Avoid Bubble/Selection** in production (too slow)  

---

## 🎯 Key Takeaways

1. **No single best algorithm** - depends on context
2. **O(n log n)** is optimal for comparison-based sorting
3. **Stable sorts** preserve relative order
4. **In-place sorts** save memory
5. **Know when to use each** algorithm

---

## 🚀 What's Next?

➡️ **[Searching Algorithms](../Searching/01_BinarySearch_Mastery.md)**  
➡️ **[Recursion Fundamentals](../Recursion/01_Recursion_Fundamentals.md)**  
➡️ **[Practice Problems](../ProblemSets/README.md)**

---

**🎮 Achievement Unlocked: Sorting Master!** 🏆

*Remember: "A sorted array is a happy array!"* 🔄

