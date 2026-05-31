# 🟣 Intercom | Problem 07 — Task Scheduler

> **Source:** LeetCode 621 · Intercom Coding Round ("Priority Queue based leetcode")
> **Difficulty:** 🟡 Medium
> **Topics:** Priority Queue · Greedy · HashMap · Scheduling · Cooldown

---

## 🔥 Ultimate Coding Interview Strategy

---

### **1️⃣ Understanding the Problem Clearly**

**Paraphrasing the Problem:**
> Given a list of tasks (characters A-Z) and a cooldown integer `n`, find the minimum number of intervals needed to execute all tasks. The same task must wait at least `n` intervals before running again. CPU can sit idle if needed.

**Clarifying Questions:**
- "Can tasks execute in any order, as long as the cooldown is respected?"
- "Do idle slots count as intervals?"
- "Is `n=0` valid? (consecutive same tasks allowed)"
- "Can the same task appear many times?"

**Example:**
```
tasks = ["A","A","A","B","B","B"], n = 2

One valid schedule: A B _ A B _ A B
Intervals = 8

Explanation: After each A, we must wait 2 slots before next A.
```

---

### **2️⃣ Breaking Down the Solution**

**Greedy Insight:**
> The most frequent task dictates the minimum schedule length. The "frame" consists of `(maxFreq - 1)` groups of size `(n + 1)`, plus the tasks that share the max frequency.

**Formula Approach (O(n) — no simulation needed):**
```
maxFreq = frequency of the most common task
maxCount = number of tasks that have this max frequency

result = max(len(tasks), (maxFreq - 1) * (n + 1) + maxCount)
```

**Why this works:**
```
Example: tasks=[A,A,A,B,B,B], n=2
maxFreq = 3 (A appears 3 times)
maxCount = 2 (A and B both appear 3 times)

Frame: (3-1) * (2+1) + 2 = 2 * 3 + 2 = 8

Visualization:
  [A B _] [A B _] [A B]
   ^frame  ^frame  ^tail(maxCount)
```

**Alternate: Heap/Simulation Approach (for interview discussion):**
- Use a max-heap of frequencies
- Each step: pop top `n+1` elements, decrement, push back non-zero
- Count total steps including idle

---

### **3️⃣ Writing Clean & Efficient Code**

#### ✅ Java — Formula Approach (Optimal)

```java
import java.util.*;

public class TaskScheduler {

    /**
     * Find minimum intervals to complete all tasks with cooldown n.
     *
     * Greedy insight: Most frequent task creates the "frame".
     * result = max(tasks.length, (maxFreq - 1) * (n + 1) + maxCount)
     *
     * Time:  O(T)  — T = tasks.length (one pass for counting)
     * Space: O(1)  — 26 slots for task frequencies
     */
    public int leastInterval(char[] tasks, int n) {
        // Count frequency of each task
        int[] freq = new int[26];
        for (char task : tasks) {
            freq[task - 'A']++;
        }

        // Find max frequency
        int maxFreq = 0;
        for (int f : freq) {
            maxFreq = Math.max(maxFreq, f);
        }

        // Count how many tasks share this max frequency
        int maxCount = 0;
        for (int f : freq) {
            if (f == maxFreq) maxCount++;
        }

        // Formula: frame slots + tail
        int formulaResult = (maxFreq - 1) * (n + 1) + maxCount;

        // Answer is max of formula vs. total tasks (no idle needed if tasks fill gaps)
        return Math.max(tasks.length, formulaResult);
    }
}
```

#### ✅ Java — Heap Simulation (For Interview Discussion)

```java
import java.util.*;

public class TaskSchedulerHeap {

    /**
     * Heap-based simulation approach.
     * 
     * Time:  O(T * n)  — T = total tasks
     * Space: O(26) = O(1)
     */
    public int leastInterval(char[] tasks, int n) {
        int[] freq = new int[26];
        for (char task : tasks) freq[task - 'A']++;

        // Max-heap of frequencies (most frequent first)
        PriorityQueue<Integer> maxHeap = new PriorityQueue<>(Collections.reverseOrder());
        for (int f : freq) {
            if (f > 0) maxHeap.offer(f);
        }

        int intervals = 0;
        while (!maxHeap.isEmpty()) {
            List<Integer> temp = new ArrayList<>();

            // Process up to (n+1) different tasks in this cycle
            for (int i = 0; i <= n; i++) {
                if (!maxHeap.isEmpty()) {
                    int count = maxHeap.poll() - 1; // execute once
                    if (count > 0) temp.add(count);
                }

                intervals++;

                // If all tasks done, break (no more idle needed)
                if (maxHeap.isEmpty() && temp.isEmpty()) break;
            }

            // Push remaining tasks back
            for (int count : temp) maxHeap.offer(count);
        }

        return intervals;
    }
}
```

#### ✅ Python — Formula (Clean)

```python
from collections import Counter

def leastInterval(tasks: list[str], n: int) -> int:
    """
    Minimum intervals to complete all tasks with cooldown n.
    
    Time:  O(T)
    Space: O(1) — at most 26 entries
    """
    freq = Counter(tasks)
    max_freq = max(freq.values())
    max_count = sum(1 for f in freq.values() if f == max_freq)

    return max(len(tasks), (max_freq - 1) * (n + 1) + max_count)
```

---

### **4️⃣ Explaining the Code**

**Formula derivation:**
```
tasks = [A,A,A,A,B,B,B,C,C], n = 2

maxFreq = 4 (A)
maxCount = 1 (only A has freq 4)

Frame:
  [A _ _] [A _ _] [A _ _] [A]
  (maxFreq-1) frames of size (n+1), plus tail of maxCount

Fill in other tasks:
  [A B C] [A B C] [A B _] [A]
  
Total = (4-1) * 3 + 1 = 10

But if tasks fill all gaps: result = max(len(tasks), formula)
When n=0: formula = (maxFreq-1)*1 + maxCount ≤ len(tasks) → always len(tasks)
```

**Complexity:**

| Approach | Time | Space |
|----------|------|-------|
| Formula | O(T) | O(1) |
| Heap Simulation | O(T × 26) | O(26) |

---

### **5️⃣ Edge Cases & Testing**

| Input | n | Expected | Explanation |
|-------|---|----------|-------------|
| `["A","A","A","B","B","B"]` | 2 | 8 | Standard case |
| `["A","A","A","B","B","B"]` | 0 | 6 | No cooldown needed |
| `["A"]` | 100 | 1 | Single task |
| `["A","A","A"]` | 2 | 7 | Only one type: A_A_A |
| `["A","B","C","D"]` | 3 | 4 | All unique, no idle |
| `["A","A","B","B","C","C"]` | 2 | 6 | All same freq, fills perfectly |

---

### **6️⃣ Why Intercom Asks This**

This tests:
- **Priority Queue knowledge** (heap approach)
- **Greedy/mathematical reasoning** (formula approach)
- **Ability to explain trade-offs** between simulation and formula
- **Connection to their product**: scheduling agent assignments with cooldowns is the same pattern

---

## 📎 Related Problems

| Problem | Connection |
|---------|-----------|
| [LC 358 - Rearrange String k Distance Apart](https://leetcode.com/problems/rearrange-string-k-distance-apart/) | Same cooldown concept with characters |
| [LC 767 - Reorganize String](https://leetcode.com/problems/reorganize-string/) | Greedy + heap, no adjacent duplicates |
| [Intercom P01](./01_FairConversationAssignment.md) | Same scheduling concept applied to agents |
