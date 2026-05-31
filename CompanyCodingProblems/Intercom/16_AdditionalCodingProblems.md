# 🟣 Intercom | Additional Coding Problems — Backup Set

> **Purpose:** Extra problems that match Intercom's patterns, for deeper preparation
> **Format:** Quick solutions + key insights (complements the detailed files 01-12)
> **Topics:** HashMap grouping, Stack-based design, Time-based stores, Merge Intervals

---

## Problem A: Group Anagrams (LC 49)

> **Why relevant:** HashMap grouping — same "class with API + internal state" pattern

### Solution (Java)

```java
import java.util.*;

public class GroupAnagrams {
    public List<List<String>> groupAnagrams(String[] strs) {
        Map<String, List<String>> map = new HashMap<>();
        
        for (String s : strs) {
            // Create sorted key
            char[] chars = s.toCharArray();
            Arrays.sort(chars);
            String key = new String(chars);
            
            map.computeIfAbsent(key, k -> new ArrayList<>()).add(s);
        }
        
        return new ArrayList<>(map.values());
    }
}
```

### Solution (Python)

```python
from collections import defaultdict

def groupAnagrams(strs: list[str]) -> list[list[str]]:
    groups = defaultdict(list)
    for s in strs:
        key = ''.join(sorted(s))  # or tuple(sorted(s))
        groups[key].append(s)
    return list(groups.values())
```

**Time:** O(n × k log k) where k = max string length | **Space:** O(n × k)

**Alternative key:** Count array `[0]*26` → tuple → faster for long strings

---

## Problem B: Min Stack (LC 155)

> **Why relevant:** Class design with O(1) operations + auxiliary data structure

### Solution (Java)

```java
import java.util.ArrayDeque;
import java.util.Deque;

public class MinStack {
    private final Deque<Integer> stack;
    private final Deque<Integer> minStack; // Tracks min at each level

    public MinStack() {
        stack = new ArrayDeque<>();
        minStack = new ArrayDeque<>();
    }

    public void push(int val) {
        stack.push(val);
        int currentMin = minStack.isEmpty() ? val : Math.min(val, minStack.peek());
        minStack.push(currentMin);
    }

    public void pop() {
        stack.pop();
        minStack.pop();
    }

    public int top() {
        return stack.peek();
    }

    public int getMin() {
        return minStack.peek();
    }
}
```

### Solution (Python)

```python
class MinStack:
    def __init__(self):
        self._stack = []      # (value, current_min) pairs
    
    def push(self, val: int) -> None:
        current_min = min(val, self._stack[-1][1]) if self._stack else val
        self._stack.append((val, current_min))
    
    def pop(self) -> None:
        self._stack.pop()
    
    def top(self) -> int:
        return self._stack[-1][0]
    
    def getMin(self) -> int:
        return self._stack[-1][1]
```

**All operations: O(1)** | **Space: O(n)**

---

## Problem C: Time-Based Key-Value Store (LC 981)

> **Why relevant:** HashMap + Binary Search + timestamps — directly maps to versioned message storage

### Solution (Java)

```java
import java.util.*;

public class TimeMap {
    // key → list of (timestamp, value) pairs sorted by timestamp
    private final Map<String, List<int[]>> store; // Using Object[] for value
    private final Map<String, List<long[]>> timeStore;
    
    // Simpler: store as separate lists
    private final Map<String, TreeMap<Integer, String>> map;

    public TimeMap() {
        map = new HashMap<>();
    }

    public void set(String key, String value, int timestamp) {
        map.computeIfAbsent(key, k -> new TreeMap<>()).put(timestamp, value);
    }

    public String get(String key, int timestamp) {
        TreeMap<Integer, String> treeMap = map.get(key);
        if (treeMap == null) return "";
        
        // Find largest key ≤ timestamp
        Map.Entry<Integer, String> entry = treeMap.floorEntry(timestamp);
        return entry != null ? entry.getValue() : "";
    }
}
```

### Solution (Python)

```python
from collections import defaultdict
import bisect

class TimeMap:
    def __init__(self):
        self._store = defaultdict(list)  # key → [(timestamp, value), ...]

    def set(self, key: str, value: str, timestamp: int) -> None:
        self._store[key].append((timestamp, value))

    def get(self, key: str, timestamp: int) -> str:
        entries = self._store[key]
        if not entries:
            return ""
        
        # Binary search for rightmost entry with ts <= timestamp
        idx = bisect.bisect_right(entries, (timestamp, chr(127))) - 1
        return entries[idx][1] if idx >= 0 else ""
```

**set: O(1)** | **get: O(log n)** | **Space: O(n)**

---

## Problem D: Merge Intervals (LC 56)

> **Why relevant:** Time-range manipulation — relevant to conversation windows, agent schedules

### Solution (Java)

```java
import java.util.*;

public class MergeIntervals {
    public int[][] merge(int[][] intervals) {
        if (intervals.length <= 1) return intervals;
        
        // Sort by start time
        Arrays.sort(intervals, (a, b) -> a[0] - b[0]);
        
        List<int[]> result = new ArrayList<>();
        int[] current = intervals[0];
        result.add(current);
        
        for (int i = 1; i < intervals.length; i++) {
            if (intervals[i][0] <= current[1]) {
                // Overlapping — extend current
                current[1] = Math.max(current[1], intervals[i][1]);
            } else {
                // No overlap — start new interval
                current = intervals[i];
                result.add(current);
            }
        }
        
        return result.toArray(new int[result.size()][]);
    }
}
```

### Solution (Python)

```python
def merge(intervals: list[list[int]]) -> list[list[int]]:
    intervals.sort(key=lambda x: x[0])
    result = [intervals[0]]
    
    for start, end in intervals[1:]:
        if start <= result[-1][1]:
            result[-1][1] = max(result[-1][1], end)
        else:
            result.append([start, end])
    
    return result
```

**Time: O(n log n)** | **Space: O(n)**

---

## Problem E: Top K Frequent Elements (LC 347)

> **Why relevant:** HashMap + Heap — Intercom's favorite combo (frequency counting + top-K)

### Solution (Java)

```java
import java.util.*;

public class TopKFrequent {
    public int[] topKFrequent(int[] nums, int k) {
        // Count frequencies
        Map<Integer, Integer> freq = new HashMap<>();
        for (int n : nums) freq.merge(n, 1, Integer::sum);
        
        // Min-heap of size k (keeps top k frequent)
        PriorityQueue<Integer> heap = new PriorityQueue<>(
            (a, b) -> freq.get(a) - freq.get(b)
        );
        
        for (int num : freq.keySet()) {
            heap.offer(num);
            if (heap.size() > k) heap.poll(); // Remove least frequent
        }
        
        int[] result = new int[k];
        for (int i = 0; i < k; i++) result[i] = heap.poll();
        return result;
    }
}
```

### Solution (Python)

```python
from collections import Counter
import heapq

def topKFrequent(nums: list[int], k: int) -> list[int]:
    freq = Counter(nums)
    return heapq.nlargest(k, freq.keys(), key=freq.get)

# Alternative: Bucket sort O(n)
def topKFrequentBucket(nums: list[int], k: int) -> list[int]:
    freq = Counter(nums)
    buckets = [[] for _ in range(len(nums) + 1)]
    for num, count in freq.items():
        buckets[count].append(num)
    
    result = []
    for i in range(len(buckets) - 1, -1, -1):
        result.extend(buckets[i])
        if len(result) >= k:
            return result[:k]
    return result
```

**Heap: O(n log k)** | **Bucket sort: O(n)**

---

## Problem F: Design Hit Counter (LC 362)

> **Why relevant:** Identical to Intercom's ping tracker/histogram pattern

### Solution (Java)

```java
public class HitCounter {
    private final int[] timestamps;
    private final int[] hits;
    
    public HitCounter() {
        timestamps = new int[300]; // 5 min window
        hits = new int[300];
    }
    
    public void hit(int timestamp) {
        int idx = timestamp % 300;
        if (timestamps[idx] != timestamp) {
            timestamps[idx] = timestamp;
            hits[idx] = 1;
        } else {
            hits[idx]++;
        }
    }
    
    public int getHits(int timestamp) {
        int total = 0;
        for (int i = 0; i < 300; i++) {
            if (timestamp - timestamps[i] < 300) {
                total += hits[i];
            }
        }
        return total;
    }
}
```

### Solution (Python)

```python
class HitCounter:
    def __init__(self):
        self._timestamps = [0] * 300
        self._hits = [0] * 300
    
    def hit(self, timestamp: int) -> None:
        idx = timestamp % 300
        if self._timestamps[idx] != timestamp:
            self._timestamps[idx] = timestamp
            self._hits[idx] = 1
        else:
            self._hits[idx] += 1
    
    def getHits(self, timestamp: int) -> int:
        return sum(
            self._hits[i] for i in range(300)
            if timestamp - self._timestamps[i] < 300
        )
```

**hit: O(1)** | **getHits: O(300) = O(1)** | **Space: O(300) = O(1)**

---

## Problem G: Insert Delete GetRandom O(1) (LC 380)

> **Why relevant:** Multi-method class design with O(1) operations — same pattern as their assignments

### Solution (Java)

```java
import java.util.*;

public class RandomizedSet {
    private final List<Integer> list;
    private final Map<Integer, Integer> indexMap; // value → index in list
    private final Random rand;

    public RandomizedSet() {
        list = new ArrayList<>();
        indexMap = new HashMap<>();
        rand = new Random();
    }

    public boolean insert(int val) {
        if (indexMap.containsKey(val)) return false;
        indexMap.put(val, list.size());
        list.add(val);
        return true;
    }

    public boolean remove(int val) {
        if (!indexMap.containsKey(val)) return false;
        int idx = indexMap.get(val);
        int last = list.get(list.size() - 1);
        
        // Swap with last element
        list.set(idx, last);
        indexMap.put(last, idx);
        
        // Remove last
        list.remove(list.size() - 1);
        indexMap.remove(val);
        return true;
    }

    public int getRandom() {
        return list.get(rand.nextInt(list.size()));
    }
}
```

**All operations: O(1)** | Pattern: ArrayList + HashMap (index tracking)

---

## Problem H: Longest Consecutive Sequence (LC 128)

> **Why relevant:** HashSet usage for O(n) — tests set-based thinking

### Solution (Java)

```java
import java.util.HashSet;
import java.util.Set;

public class LongestConsecutive {
    public int longestConsecutive(int[] nums) {
        Set<Integer> set = new HashSet<>();
        for (int n : nums) set.add(n);
        
        int longest = 0;
        for (int n : set) {
            // Only start counting from sequence beginning
            if (!set.contains(n - 1)) {
                int length = 1;
                while (set.contains(n + length)) {
                    length++;
                }
                longest = Math.max(longest, length);
            }
        }
        return longest;
    }
}
```

**Time: O(n)** | **Space: O(n)** | Key insight: only start from sequence start (n-1 not in set)

---

## 🧠 Pattern Summary for Intercom

| Pattern | Problems | Core Data Structure | Frequency |
|---------|----------|-------------------|-----------|
| **Class + HashMap + State** | P01, P04, P05, P08, P12 | HashMap + Queue/Array | 🔴 Very High |
| **Priority Queue** | P07, P10, Top K | Min/Max Heap | 🟠 High |
| **Sliding Window** | P03, P08, Hit Counter | HashMap + Two Pointers | 🟠 High |
| **Dual Data Structure** | P09, Min Stack, RandomizedSet | HashMap + List/DLL | 🟡 Medium |
| **Interval/Scheduling** | P10, Merge Intervals | Sort + Sweep/Heap | 🟡 Medium |
| **Frequency Counting** | P05, P11, Group Anagrams, Top K | HashMap + Sort/Heap | 🟡 Medium |

---

## Quick Practice Plan

### Week 1: Core Patterns (30 min/day)

| Day | Problem | Focus |
|-----|---------|-------|
| Mon | P01 — Agent Assignment | Write from scratch in 25 min |
| Tue | P08 — Rate Limiter | Class + sliding window |
| Wed | P09 — LRU Cache | Dual data structure |
| Thu | P07 — Task Scheduler | Priority Queue + Greedy |
| Fri | P10 — Meeting Rooms II | Heap + intervals |
| Sat | P12 — Skill-Based Routing | Complex class design |
| Sun | Review all — focus on explaining aloud | Communication practice |

### Week 2: Backup Problems (30 min/day)

| Day | Problem | Focus |
|-----|---------|-------|
| Mon | Top K Frequent | HashMap + Heap combo |
| Tue | Merge Intervals | Sort + sweep |
| Wed | Group Anagrams | HashMap grouping |
| Thu | Hit Counter (LC 362) | Time-windowed counting |
| Fri | Min Stack | Auxiliary structure |
| Sat | Longest Consecutive | Set-based O(n) thinking |
| Sun | Mock interview — pick random problem | Timed, explain aloud |
