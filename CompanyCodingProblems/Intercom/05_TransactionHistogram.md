# 🟣 Intercom | Problem 05 — Transaction Histogram

> **Source:** Intercom Coding Round (recurring)
> **Difficulty:** 🟡 Medium
> **Topics:** HashMap · Time Bucketing · Sorting · Data Aggregation · Function Design

---

## 🔥 Ultimate Coding Interview Strategy

---

### **1️⃣ Understanding the Problem Clearly**

**Paraphrasing the Problem:**
> Given a list of transaction log entries (each with a timestamp and metadata), write a function that produces a **histogram** — grouping transactions into fixed-size time buckets across a specified period and returning the count (or aggregate value) per bucket.

**Clarifying Questions to Ask:**
- "What does each log entry contain — just a timestamp, or also user ID, amount, type?"
- "Is the `bucket_size` given in seconds? Same unit as timestamps?"
- "Should we count the number of transactions per bucket, or aggregate a value (e.g., sum of amounts)?"
- "Are log timestamps guaranteed to be within the period range, or do we need to filter?"
- "Should the result be a list of counts, a list of `(label, count)` pairs, or a dict?"
- "Is the log list sorted by time, or unsorted?"
- "What if no transactions fall in a bucket — should that bucket still appear with count 0?"
- "How should the bucket labels look — as ranges like `'0-59'` or just the start time?"

**Confirmed Assumptions (interviewer confirms):**
- Each log is a dict/object: `{ "user_id": str, "timestamp": int, "amount": float }`
- We count transactions per bucket (not sum)
- Filter out transactions outside `[start_time, end_time]`
- Empty buckets appear as `0` in the result
- Return a list of `(bucket_label, count)` pairs, where label is `"start-end"`
- Logs may be unsorted

**Walking Through an Example:**
```
logs = [
    {"user_id": "u1", "timestamp": 5,  "amount": 100.0},
    {"user_id": "u2", "timestamp": 72, "amount": 50.0},
    {"user_id": "u1", "timestamp": 30, "amount": 20.0},
    {"user_id": "u3", "timestamp": 150,"amount": 200.0},
    {"user_id": "u2", "timestamp": 500,"amount": 10.0},  # outside range
]

transaction_histogram(logs, start_time=0, end_time=200, bucket_size=60)

Buckets:
  [0..59]   → timestamps: 5, 30            → count = 2
  [60..119] → timestamp: 72                → count = 1
  [120..179]→ timestamp: 150               → count = 1
  [180..200]→ no transactions              → count = 0  (partial last bucket)
  t=500 outside [0,200] → filtered

Result:
  [("0-59", 2), ("60-119", 1), ("120-179", 1), ("180-200", 0)]
```

---

### **2️⃣ Breaking Down the Solution**

**Brute Force Approach:**
> For each bucket, iterate all logs and count how many fall in that bucket's range. O(B × N) where B = number of buckets, N = number of logs.

**Optimal Approach — Single Pass:**
> For each log, compute its bucket index directly using floor division. One pass through logs → O(N). Build bucket labels separately → O(B).

```
bucket_idx   = (timestamp - start_time) // bucket_size
total_buckets = (end_time - start_time) // bucket_size + 1
bucket_start  = start_time + bucket_idx * bucket_size
bucket_end    = min(bucket_start + bucket_size - 1, end_time)
label         = f"{bucket_start}-{bucket_end}"
```

**Approach Comparison:**

| Approach | Time | Space | Notes |
|----------|------|-------|-------|
| Brute force (nested loop) | O(B × N) | O(B) | Poor for large N or many buckets |
| Single pass + bucket array | O(N + B) | O(B) | Optimal — one scan of logs |
| Sort then scan | O(N log N + B) | O(N) | Useful if range queries are repeated |

> B = number of buckets = `(end_time - start_time) // bucket_size + 1`

---

### **3️⃣ Writing Clean & Efficient Code**

#### ✅ Python Solution

```python
from dataclasses import dataclass


@dataclass
class TransactionLog:
    """A single transaction log entry."""
    user_id: str
    timestamp: int
    amount: float


def transaction_histogram(
    logs: list[TransactionLog],
    start_time: int,
    end_time: int,
    bucket_size: int,
) -> list[tuple[str, int]]:
    """
    Build a histogram of transaction counts grouped into fixed-size time buckets.

    Args:
        logs:        List of transaction log entries.
        start_time:  Start of the period (inclusive), in seconds.
        end_time:    End of the period (inclusive), in seconds.
        bucket_size: Duration of each time bucket in seconds.

    Returns:
        List of (bucket_label, count) tuples, one per bucket.
        bucket_label format: "start_sec-end_sec"
        Empty buckets have count = 0.

    Time:  O(N + B)  where N = len(logs), B = number of buckets
    Space: O(B)      for the counts array
    """
    if bucket_size <= 0:
        raise ValueError("bucket_size must be a positive integer")
    if start_time > end_time:
        raise ValueError("start_time must be <= end_time")

    total_buckets = (end_time - start_time) // bucket_size + 1
    counts = [0] * total_buckets

    # Single pass: assign each in-range log to its bucket
    for log in logs:
        t = log.timestamp
        if start_time <= t <= end_time:
            bucket_idx = (t - start_time) // bucket_size
            counts[bucket_idx] += 1

    # Build labeled result
    result = []
    for i in range(total_buckets):
        bucket_start = start_time + i * bucket_size
        bucket_end = min(bucket_start + bucket_size - 1, end_time)
        label = f"{bucket_start}-{bucket_end}"
        result.append((label, counts[i]))

    return result
```

---

#### ✅ Python — Dict Input Variant (No Dataclass)

```python
def transaction_histogram(
    logs: list[dict],
    start_time: int,
    end_time: int,
    bucket_size: int,
) -> list[tuple[str, int]]:
    """
    Same as above but accepts plain dicts: {"user_id": ..., "timestamp": ..., "amount": ...}
    """
    if bucket_size <= 0:
        raise ValueError("bucket_size must be positive")

    total_buckets = (end_time - start_time) // bucket_size + 1
    counts = [0] * total_buckets

    for log in logs:
        t = log["timestamp"]
        if start_time <= t <= end_time:
            counts[(t - start_time) // bucket_size] += 1

    result = []
    for i in range(total_buckets):
        b_start = start_time + i * bucket_size
        b_end = min(b_start + bucket_size - 1, end_time)
        result.append((f"{b_start}-{b_end}", counts[i]))

    return result
```

---

#### ✅ Java Solution

```java
import java.util.*;

public class TransactionHistogram {

    public record TransactionLog(String userId, int timestamp, double amount) {}

    public record BucketResult(String label, int count) {}

    /**
     * Build a histogram of transaction counts in fixed-size time buckets.
     *
     * @param logs       List of transaction log entries.
     * @param startTime  Start of the period (inclusive).
     * @param endTime    End of the period (inclusive).
     * @param bucketSize Duration of each bucket in seconds.
     * @return List of BucketResult(label, count), one per bucket.
     *
     * Time:  O(N + B)   N = logs.size(), B = number of buckets
     * Space: O(B)       counts array
     */
    public static List<BucketResult> buildHistogram(
            List<TransactionLog> logs,
            int startTime,
            int endTime,
            int bucketSize) {

        if (bucketSize <= 0) throw new IllegalArgumentException("bucketSize must be positive");
        if (startTime > endTime) throw new IllegalArgumentException("startTime must be <= endTime");

        int totalBuckets = (endTime - startTime) / bucketSize + 1;
        int[] counts = new int[totalBuckets];

        // Single pass: place each in-range log into its bucket
        for (TransactionLog log : logs) {
            int t = log.timestamp();
            if (t >= startTime && t <= endTime) {
                int bucketIdx = (t - startTime) / bucketSize;
                counts[bucketIdx]++;
            }
        }

        // Build labeled result
        List<BucketResult> result = new ArrayList<>(totalBuckets);
        for (int i = 0; i < totalBuckets; i++) {
            int bStart = startTime + i * bucketSize;
            int bEnd = Math.min(bStart + bucketSize - 1, endTime);
            String label = bStart + "-" + bEnd;
            result.add(new BucketResult(label, counts[i]));
        }

        return result;
    }
}
```

---

#### ✅ Extended Variant — Aggregate Amount Per Bucket

```python
def transaction_histogram_by_amount(
    logs: list[dict],
    start_time: int,
    end_time: int,
    bucket_size: int,
) -> list[tuple[str, float]]:
    """
    Instead of counting transactions, sum the 'amount' field per bucket.
    Useful when interviewer asks to extend to value-based aggregation.
    """
    total_buckets = (end_time - start_time) // bucket_size + 1
    totals = [0.0] * total_buckets

    for log in logs:
        t = log["timestamp"]
        if start_time <= t <= end_time:
            idx = (t - start_time) // bucket_size
            totals[idx] += log["amount"]

    result = []
    for i in range(total_buckets):
        b_start = start_time + i * bucket_size
        b_end = min(b_start + bucket_size - 1, end_time)
        result.append((f"{b_start}-{b_end}", round(totals[i], 2)))

    return result
```

---

#### ✅ Full Usage Demo

```python
logs = [
    {"user_id": "u1", "timestamp": 5,   "amount": 100.0},
    {"user_id": "u2", "timestamp": 72,  "amount": 50.0},
    {"user_id": "u1", "timestamp": 30,  "amount": 20.0},
    {"user_id": "u3", "timestamp": 150, "amount": 200.0},
    {"user_id": "u2", "timestamp": 500, "amount": 10.0},  # outside [0,200]
]

result = transaction_histogram(logs, start_time=0, end_time=200, bucket_size=60)
for label, count in result:
    print(f"  [{label}]: {'#' * count} ({count})")

# Output:
#   [0-59]:    ## (2)
#   [60-119]:  # (1)
#   [120-179]: # (1)
#   [180-200]:   (0)
```

---

### **4️⃣ Explaining the Code to the Interviewer**

**Algorithm walkthrough — `logs` above, `start=0, end=200, bucket=60`:**
```
total_buckets = (200 - 0) // 60 + 1 = 3 + 1 = 4
counts = [0, 0, 0, 0]

t=5   in [0,200] → idx (5-0)//60   = 0 → counts[0]++ → [1,0,0,0]
t=72  in [0,200] → idx (72-0)//60  = 1 → counts[1]++ → [1,1,0,0]
t=30  in [0,200] → idx (30-0)//60  = 0 → counts[0]++ → [2,1,0,0]
t=150 in [0,200] → idx (150-0)//60 = 2 → counts[2]++ → [2,1,1,0]
t=500 NOT in [0,200] → skip

Build labels:
  i=0: start=0,  end=min(59,200)=59   → "0-59",    count=2
  i=1: start=60, end=min(119,200)=119 → "60-119",  count=1
  i=2: start=120,end=min(179,200)=179 → "120-179", count=1
  i=3: start=180,end=min(239,200)=200 → "180-200", count=0

Result: [("0-59",2), ("60-119",1), ("120-179",1), ("180-200",0)] ✓
```

**Why `min(bucket_start + bucket_size - 1, end_time)` for bucket end:**
```
Without min: bucket i=3 would be labeled "180-239" which exceeds end_time=200.
With min:    bucket i=3 is labeled "180-200" — correctly capped at end_time.
The -1 ensures the label shows the inclusive last second of that bucket.
```

**Complexity:**

| | Value | Explanation |
|--|-------|------------|
| Time | O(N + B) | N = log count (single scan), B = bucket count (label build) |
| Space | O(B) | counts array of size B |
| Typical B | ≤ 200 | `(10^4) // 60 + 1 ≈ 168` for minute-level, constraint-bounded |

---

### **5️⃣ Discussing Edge Cases & Testing**

#### Test Table

| Scenario | Inputs | Expected | Notes |
|----------|--------|----------|-------|
| Normal case | logs at 5,72,30,150 | [2,1,1,0] | Standard |
| All in one bucket | logs at 5,10, bucket=3600 | [2] | Large bucket |
| Transaction at exact end_time | t=200, [0,200], b=100 | [0,0,1] | Boundary inclusive |
| Transaction at start_time | t=0, [0,200], b=100 | [1,0,0] | Start boundary inclusive |
| All transactions outside range | logs at 500, [0,200] | [0,0,0] | All filtered |
| Empty log list | [], [0,200], b=60 | all zeros | No logs |
| Bucket=1 | logs at 0,1,2,3, [0,3], b=1 | [1,1,1,1] | Max granularity |
| Bucket larger than range | logs at 5, [0,10], b=100 | [1] | Single bucket |
| Duplicate timestamps | t=50 three times, [0,100], b=60 | [3,0] | Duplicates counted |
| Amount aggregation | same logs, sum instead of count | float totals | Extension |

#### Off-By-One Checks
- `counts` size = `(end - start) // bucket + 1` — never drop last bucket
- Range filter: `start_time <= t <= end_time` — both bounds inclusive
- Label end: `min(b_start + bucket_size - 1, end_time)` — cap at period end
- Bucket index: `(t - start_time) // bucket_size` — offset from start, not absolute time

---

### **6️⃣ Debugging & Fixing Mistakes**

**Common Mistakes:**

❌ **Mistake 1: Using absolute timestamp for bucket index**
```python
# WRONG — if start_time=100, t=105 gives idx=1 instead of 0
bucket_idx = t // bucket_size

# RIGHT — offset from start_time
bucket_idx = (t - start_time) // bucket_size
```

❌ **Mistake 2: No out-of-range filter → IndexError**
```python
# WRONG — t=500 with start=0, end=200, bucket=60:
# idx = (500-0)//60 = 8, but counts has only 4 elements → IndexError
counts[(t - start_time) // bucket_size] += 1

# RIGHT
if start_time <= t <= end_time:
    counts[(t - start_time) // bucket_size] += 1
```

❌ **Mistake 3: Label bucket end without capping at end_time**
```python
# WRONG — last bucket labeled "180-239" when end_time=200
b_end = b_start + bucket_size - 1

# RIGHT
b_end = min(b_start + bucket_size - 1, end_time)
```

❌ **Mistake 4: Missing `+ 1` in total_buckets**
```python
# WRONG
total_buckets = (end_time - start_time) // bucket_size

# RIGHT
total_buckets = (end_time - start_time) // bucket_size + 1
```

**Debug Helper:**
```python
def transaction_histogram_debug(logs, start_time, end_time, bucket_size):
    total_buckets = (end_time - start_time) // bucket_size + 1
    print(f"total_buckets={total_buckets}")
    counts = [0] * total_buckets
    for log in logs:
        t = log["timestamp"]
        if start_time <= t <= end_time:
            idx = (t - start_time) // bucket_size
            print(f"  t={t} → bucket[{idx}]")
            counts[idx] += 1
    return counts
```

---

## 🚀 Best Practices Demonstrated

✅ **Single-pass O(N+B)** — no nested loops; bucket index computed directly  
✅ **Parameterized bucket size** — flexible for minutes, hours, days, or custom intervals  
✅ **Empty bucket included** — histograms always have a bar even for zero counts  
✅ **Extension ready** — easy to switch from count to sum/avg with one-line change  
✅ **Input validation** — bucket_size > 0 checked; clear error messages  
✅ **Label capped at end_time** — last bucket label reflects actual range, not theoretical  

---

## 📎 Related Problems

| # | Problem | Connection |
|---|---------|-----------|
| [Intercom P02](./02_TweetCountsPerFrequency.md) | Tweet Counts Per Frequency | Same bucketing — class API vs one-shot function |
| [Intercom P04](./04_UserPingTracker.md) | User Ping Tracker | Same bucketing — per-user keyed |
| [LeetCode 1348](https://leetcode.com/problems/tweet-counts-per-frequency/) | Tweet Counts | LeetCode version of this pattern |
| [LeetCode 220](https://leetcode.com/problems/contains-duplicate-iii/) | Contains Duplicate III | Bucket sort / time window concept |

---

*Previous: [← 04 UserPingTracker](./04_UserPingTracker.md) | Next: [06 ChatSystemLowLevelDesign →](./06_ChatSystemLowLevelDesign.md)*
