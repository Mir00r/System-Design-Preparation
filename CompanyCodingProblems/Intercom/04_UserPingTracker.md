# 🟣 Intercom | Problem 04 — User Ping Tracker API

> **Source:** Intercom Coding Round (recurring)
> **Difficulty:** 🟡 Medium
> **Topics:** HashMap · Partitioned Array · Class Design · Time Bucketing · Math

---

## 🔥 Ultimate Coding Interview Strategy

---

### **1️⃣ Understanding the Problem Clearly**

**Paraphrasing the Problem:**
> Build a class with two methods:
> 1. `record_ping(user_id, timestamp)` — record that a user was active at a given time
> 2. `get_pings(user_id, start_time, end_time, partition)` — retrieve how many pings a user had within a time range, **split into equal-sized time buckets** of size `partition` seconds

**Clarifying Questions to Ask:**
- "Is `partition` a duration in seconds?"
- "Should the result be a list of counts (one per bucket), or a total count?"
- "What if `partition` doesn't divide `(end_time - start_time)` evenly — does the last bucket end at `end_time`?"
- "Can the same user be pinged multiple times at the same timestamp?"
- "What should we return if a user has no pings? An empty list of zeros or an empty list?"
- "Can `start_time > end_time`? Should we raise an error or return empty?"
- "What is the time unit — milliseconds or seconds?"

**Confirmed Assumptions:**
- `partition` is a duration in the same unit as timestamps (seconds)
- Result is a **list of integers** — count per bucket — similar to histogram bars
- Last bucket may be shorter than `partition` (it always ends at `end_time`)
- Same user can ping at the same timestamp (duplicates count separately)
- Unknown user → return list of zeros (not an error)
- `start_time ≤ end_time` guaranteed

**Walking Through an Example:**
```
user "alice" pings at times: [10, 25, 61, 90, 100]
get_pings("alice", start_time=0, end_time=100, partition=30)

Buckets:
  [0..29]   → times 10, 25   → count = 2
  [30..59]  → no pings       → count = 0
  [60..89]  → time 61        → count = 1
  [90..100] → times 90, 100  → count = 2  (last bucket shorter than 30)

Result: [2, 0, 1, 2]

Bucket formula:
  total_buckets = (100 - 0) // 30 + 1 = 3 + 1 = 4
  bucket_idx    = (t - start_time) // partition
```

**Another Example — Single Bucket:**
```
pings at [5, 10, 20], get_pings("alice", 0, 100, 3600)
  total_buckets = (100-0)//3600 + 1 = 0 + 1 = 1
  All 3 pings fall in bucket 0
  Result: [3]
```

---

### **2️⃣ Breaking Down the Solution**

**Brute Force Approach:**
> Iterate all pings for the user. For each ping in `[start_time, end_time]`, compute its bucket index. Accumulate counts in an array.

> This is already optimal because we must at minimum read every stored ping once.

**Key Insight — Partitioned Array (Bucketing):**
```
Bucket index  = (timestamp - start_time) // partition
Total buckets = (end_time - start_time) // partition + 1

Why +1?
  If end_time = start_time + partition exactly, there are 2 buckets, not 1.
  (partition // partition) = 1, +1 = 2 → correct.
  The formula ensures the last (possibly partial) bucket is always included.
```

**Rounding Off — The Core Math:**
```
Integer floor division ( // ) acts as "round down", placing each timestamp
into the correct bucket without floating-point errors.

t=61, start=0, partition=30:
  (61 - 0) // 30 = 2  → bucket index 2 = [60..89]  ✓
```

**Optional Optimization — Sorted + Binary Search:**
> If `record_ping` is called far more than `get_pings`, sort timestamps on insert (using `bisect.insort`) and binary-search for `start_time`/`end_time` on query.

| Approach | `record_ping` | `get_pings` | Space |
|----------|--------------|-------------|-------|
| List + linear scan | O(1) | O(T) | O(T) |
| Sorted list + bisect | O(log T) insert | O(log T + M) | O(T) |

> T = total pings for user, M = pings in range

---

### **3️⃣ Writing Clean & Efficient Code**

#### ✅ Python Solution

```python
from collections import defaultdict


class PingTracker:
    """
    Tracks user ping timestamps and provides time-partitioned counts.

    record_ping(user_id, timestamp)                       → O(1)
    get_pings(user_id, start_time, end_time, partition)   → O(T)
        T = number of pings stored for user_id
    """

    def __init__(self):
        # user_id → list of recorded timestamps
        self._pings: dict[str, list[int]] = defaultdict(list)

    def record_ping(self, user_id: str, timestamp: int) -> None:
        """
        Record that user_id was active at the given timestamp.
        Duplicate timestamps are allowed and counted separately.
        """
        self._pings[user_id].append(timestamp)

    def get_pings(
        self,
        user_id: str,
        start_time: int,
        end_time: int,
        partition: int,
    ) -> list[int]:
        """
        Return a partitioned histogram of ping counts within [start_time, end_time].

        Each element in the result represents the number of pings
        in a time bucket of size `partition` seconds.
        The last bucket may be shorter (ends at end_time).

        Args:
            user_id:    The user to query.
            start_time: Range start (inclusive), in seconds.
            end_time:   Range end (inclusive), in seconds.
            partition:  Bucket size in seconds.

        Returns:
            List of int, length = (end_time - start_time) // partition + 1
        """
        if partition <= 0:
            raise ValueError("partition must be a positive integer")

        total_buckets = (end_time - start_time) // partition + 1
        counts = [0] * total_buckets

        for t in self._pings.get(user_id, []):
            if start_time <= t <= end_time:
                bucket_idx = (t - start_time) // partition
                counts[bucket_idx] += 1

        return counts
```

---

#### ✅ Java Solution

```java
import java.util.*;

/**
 * PingTracker — Tracks user ping timestamps with time-partitioned queries.
 *
 * record_ping:  O(1) amortized
 * get_pings:    O(T)   where T = total pings stored for that user
 */
public class PingTracker {

    // userId → list of recorded timestamps
    private final Map<String, List<Integer>> pings;

    public PingTracker() {
        pings = new HashMap<>();
    }

    /**
     * Record that userId was active at the given timestamp.
     * Duplicate timestamps count separately.
     */
    public void recordPing(String userId, int timestamp) {
        pings.computeIfAbsent(userId, k -> new ArrayList<>()).add(timestamp);
    }

    /**
     * Return a partitioned histogram of ping counts within [startTime, endTime].
     *
     * Bucket index  = (timestamp - startTime) / partition
     * Total buckets = (endTime - startTime) / partition + 1
     *
     * @param userId    The user to query.
     * @param startTime Range start (inclusive).
     * @param endTime   Range end (inclusive).
     * @param partition Bucket size in seconds.
     * @return List of ping counts per bucket.
     */
    public List<Integer> getPings(String userId, int startTime, int endTime, int partition) {
        if (partition <= 0) {
            throw new IllegalArgumentException("partition must be positive");
        }

        int totalBuckets = (endTime - startTime) / partition + 1;
        int[] counts = new int[totalBuckets];

        List<Integer> userPings = pings.getOrDefault(userId, Collections.emptyList());
        for (int t : userPings) {
            if (t >= startTime && t <= endTime) {
                int bucketIdx = (t - startTime) / partition;
                counts[bucketIdx]++;
            }
        }

        // Convert int[] → List<Integer>
        List<Integer> result = new ArrayList<>(totalBuckets);
        for (int count : counts) result.add(count);
        return result;
    }
}
```

---

#### ✅ Python — Optimized with Binary Search (Follow-Up)

```python
import bisect
from collections import defaultdict


class PingTrackerOptimized:
    """
    Optimized version: sorted insertion enables binary-search range queries.

    record_ping:  O(log T) — sorted insert
    get_pings:    O(log T + M) — binary search to find range, then scan M matches
    """

    def __init__(self):
        self._pings: dict[str, list[int]] = defaultdict(list)

    def record_ping(self, user_id: str, timestamp: int) -> None:
        """Insert into sorted list to allow binary search on queries."""
        bisect.insort(self._pings[user_id], timestamp)

    def get_pings(
        self,
        user_id: str,
        start_time: int,
        end_time: int,
        partition: int,
    ) -> list[int]:
        total_buckets = (end_time - start_time) // partition + 1
        counts = [0] * total_buckets

        times = self._pings.get(user_id, [])

        # Binary search to find start and end indices in sorted list
        lo = bisect.bisect_left(times, start_time)
        hi = bisect.bisect_right(times, end_time)

        # Only iterate pings in [start_time, end_time]
        for i in range(lo, hi):
            bucket_idx = (times[i] - start_time) // partition
            counts[bucket_idx] += 1

        return counts
```

---

#### ✅ Demonstration — Full Usage Example

```python
tracker = PingTracker()

# Alice pings at various times
tracker.record_ping("alice", 10)
tracker.record_ping("alice", 25)
tracker.record_ping("alice", 61)
tracker.record_ping("alice", 90)
tracker.record_ping("alice", 100)

# Bob has no pings

# Query Alice with 30-second partitions over [0, 100]
result = tracker.get_pings("alice", start_time=0, end_time=100, partition=30)
print(result)  # [2, 0, 1, 2]
#   Bucket [0..29]:   pings at 10, 25    → 2
#   Bucket [30..59]:  no pings            → 0
#   Bucket [60..89]:  ping at 61          → 1
#   Bucket [90..100]: pings at 90, 100    → 2

# Query Bob (never pinged) — returns all zeros, no error
result = tracker.get_pings("bob", 0, 100, 30)
print(result)  # [0, 0, 0, 0]

# Query with partition larger than range → single bucket
result = tracker.get_pings("alice", 0, 100, 3600)
print(result)  # [5]

# Query Alice within sub-range only
result = tracker.get_pings("alice", 20, 70, 20)
# Buckets: [20..39]=1(25), [40..59]=0, [60..70]=1(61)
print(result)  # [1, 0, 1]
```

---

### **4️⃣ Explaining the Code to the Interviewer**

**Walkthrough — `get_pings("alice", 0, 100, 30)`:**
```
partition = 30
total_buckets = (100 - 0) // 30 + 1 = 3 + 1 = 4
counts = [0, 0, 0, 0]

t=10:  in [0,100] → idx (10-0)//30 = 0  → counts[0]++ → [1,0,0,0]
t=25:  in [0,100] → idx (25-0)//30 = 0  → counts[0]++ → [2,0,0,0]
t=61:  in [0,100] → idx (61-0)//30 = 2  → counts[2]++ → [2,0,1,0]
t=90:  in [0,100] → idx (90-0)//30 = 3  → counts[3]++ → [2,0,1,1]
t=100: in [0,100] → idx (100-0)//30 = 3 → counts[3]++ → [2,0,1,2]

Result: [2, 0, 1, 2]  ✓
```

**Why `total_buckets = diff // partition + 1`:**
```
diff=90, partition=30: 90//30 = 3, +1 = 4 buckets  (indices 0,1,2,3)
diff=60, partition=30: 60//30 = 2, +1 = 3 buckets  (indices 0,1,2)
diff=29, partition=30: 29//30 = 0, +1 = 1 bucket   (index 0 only)
diff=0,  partition=30:  0//30 = 0, +1 = 1 bucket   (single point range)
```

**Complexity Analysis:**

| Method | Time | Space | Notes |
|--------|------|-------|-------|
| `__init__` | O(1) | O(1) | Empty HashMap |
| `record_ping` | O(1) | O(1) amortized | List append |
| `get_pings` | O(T) | O(B) | T=pings for user, B=bucket count |
| `record_ping` (optimized) | O(log T) | O(1) | Sorted insert |
| `get_pings` (optimized) | O(log T + M) | O(B) | M=pings in range |

---

### **5️⃣ Discussing Edge Cases & Testing**

#### Test Table

| Scenario | Input | Expected | Notes |
|----------|-------|----------|-------|
| Normal case | alice pings=[10,25,61,90,100], [0,100], p=30 | `[2,0,1,2]` | Standard |
| Unknown user | "bob" never recorded, [0,100], p=30 | `[0,0,0,0]` | No KeyError |
| Single bucket | pings=[5,10,20], [0,100], p=3600 | `[3]` | Large partition |
| Single point | ping=[50], [50,50], p=1 | `[1]` | Zero-range |
| Ping at start_time | ping=[0], [0,60], p=60 | `[1,0]` | Wait — only 1 bucket since `(60-0)//60+1=2` but t=0 → bucket 0 |
| Ping at end_time | ping=[100], [0,100], p=50 | `[0,0,1]` | Boundary inclusive |
| Ping outside range | ping=[200], [0,100], p=30 | `[0,0,0,0]` | Filtered out |
| Multiple pings same time | pings=[50,50,50], [0,100], p=100 | `[3]` | Duplicates count |
| Partition=1 | pings=[0,1,2,3], [0,3], p=1 | `[1,1,1,1]` | Max granularity |
| Empty pings | no pings recorded, any query | All zeros | Default empty list |

#### Off-By-One Checks
- Boundary: `t >= start_time` and `t <= end_time` (both inclusive)
- Bucket count: `(end_time - start_time) // partition + 1` — never forget the `+ 1`
- Bucket index max = `total_buckets - 1` — verify last timestamp maps correctly

---

### **6️⃣ Debugging & Fixing Mistakes**

**Common Mistakes:**

❌ **Mistake 1: Missing `+ 1` in total_buckets**
```python
# WRONG — last partial bucket dropped
total_buckets = (end_time - start_time) // partition

# RIGHT
total_buckets = (end_time - start_time) // partition + 1
```

❌ **Mistake 2: Using absolute time instead of offset for bucket index**
```python
# WRONG — if start_time != 0, this creates wrong bucket index
bucket_idx = t // partition

# RIGHT — always subtract start_time for offset-based indexing
bucket_idx = (t - start_time) // partition
```

❌ **Mistake 3: No range filter — out-of-range pings get negative indices or IndexError**
```python
# WRONG — t=200 with start=0, end=100 would still try counts[bucket_idx]++
for t in pings:
    counts[(t - start_time) // partition] += 1

# RIGHT — filter first
for t in pings:
    if start_time <= t <= end_time:
        counts[(t - start_time) // partition] += 1
```

❌ **Mistake 4: Using `defaultdict` and accessing missing user causes creation**
```python
# WRONG — creates empty list for bob, leaks memory
for t in self._pings["bob"]:

# RIGHT — use .get() with default
for t in self._pings.get("bob", []):
```

**Debug Trace:**
```python
def get_pings(self, user_id, start_time, end_time, partition):
    total_buckets = (end_time - start_time) // partition + 1
    print(f"total_buckets={total_buckets}, partition={partition}")

    counts = [0] * total_buckets
    for t in self._pings.get(user_id, []):
        if start_time <= t <= end_time:
            idx = (t - start_time) // partition
            print(f"  t={t} → bucket[{idx}]")
            counts[idx] += 1
    return counts
```

---

## 🚀 Best Practices Demonstrated

✅ **Clear API contract** — docstrings specify what "partition" means and the output shape  
✅ **Validation at boundary** — `partition <= 0` check prevents infinite loops / division by zero  
✅ **Defensive `.get()`** — unknown users return zeros without raising `KeyError`  
✅ **Separation of concerns** — record is pure write; get is pure read (no mutation)  
✅ **Follow-up ready** — binary search variant shown for read-heavy scenarios  
✅ **Formula verified** — bucket count formula traced through multiple edge cases  

---

## 📎 Related Problems

| # | Problem | Connection |
|---|---------|-----------|
| [LeetCode 933](https://leetcode.com/problems/number-of-recent-calls/) | Number of Recent Calls | Ping-style timestamp tracking |
| [LeetCode 981](https://leetcode.com/problems/time-based-key-value-store/) | Time-Based Key-Value Store | userId + timestamp + binary search |
| [Intercom P02](./02_TweetCountsPerFrequency.md) | Tweet Counts Per Frequency | Same bucketing pattern, fixed frequencies |
| [Intercom P05](./05_TransactionHistogram.md) | Transaction Histogram | Same concept as a one-shot function |

---

*Previous: [← 03 LongestSubstringNoRepeating](./03_LongestSubstringNoRepeating.md) | Next: [05 TransactionHistogram →](./05_TransactionHistogram.md)*
