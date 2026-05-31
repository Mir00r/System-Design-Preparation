# 🟣 Intercom | Problem 02 — Tweet Counts Per Frequency

> **Source:** LeetCode 1348 · Intercom Coding Round (recurring — histogram / time-series variant)
> **Difficulty:** 🟡 Medium
> **Topics:** HashMap · Time Bucketing · Math · Simulation · Design

---

## 🔥 Ultimate Coding Interview Strategy

---

### **1️⃣ Understanding the Problem Clearly**

**Paraphrasing the Problem:**
> Build a `TweetCounts` class that records tweet timestamps and can query how many tweets occurred in each time window ("chunk") within a given period, at a given frequency (minute/hour/day).

**Clarifying Questions to Ask:**
- "Is `startTime` always ≤ `endTime`?"
- "Can the same tweet name be recorded at the same timestamp multiple times?"
- "Is the last chunk always shorter than the full chunk size (if it doesn't fit evenly)?"
- "Can `startTime` be 0?"
- "Is the input always valid — no null tweet names, no negative times?"
- "Should the result always have at least one chunk (even if zero tweets match)?"

**Confirmed Assumptions (from problem statement):**
- `0 <= time, startTime, endTime <= 10^9`
- `endTime - startTime <= 10^4` (this bounds the max number of chunks: 10^4 / 60 ≈ 167 chunks max)
- At most 10^4 total calls
- Last chunk may be shorter than full chunk size — it always ends at `endTime`
- Frequency strings: exactly `"minute"`, `"hour"`, or `"day"`

**Walking Through the Example:**
```
Frequency → Chunk Size:
  "minute" → 60 seconds
  "hour"   → 3600 seconds
  "day"    → 86400 seconds

Chunking formula:
  chunk_index = (tweet_time - startTime) // chunk_size
  total_chunks = (endTime - startTime) // chunk_size + 1

Period [0, 59] with "minute":
  chunks = (59 - 0) // 60 + 1 = 0 + 1 = 1 chunk → [0, 59]

Period [0, 60] with "minute":
  chunks = (60 - 0) // 60 + 1 = 1 + 1 = 2 chunks → [0,59], [60,60]

Period [0, 210] with "hour":
  chunks = (210 - 0) // 3600 + 1 = 0 + 1 = 1 chunk → [0, 210]
```

**Full example trace:**
```
recordTweet("tweet3", 0)    → stored
recordTweet("tweet3", 60)   → stored
recordTweet("tweet3", 10)   → stored

getTweetCountsPerFrequency("minute", "tweet3", 0, 59)
  chunks = 1, delta = 60
  t=0  → idx (0-0)//60 = 0 ✓
  t=60 → out of range [0,59]
  t=10 → idx (10-0)//60 = 0 ✓
  result = [2] ✓

getTweetCountsPerFrequency("minute", "tweet3", 0, 60)
  chunks = 2, delta = 60
  t=0  → idx 0 ✓
  t=60 → idx (60-0)//60 = 1 ✓
  t=10 → idx 0 ✓
  result = [2, 1] ✓

recordTweet("tweet3", 120)  → stored

getTweetCountsPerFrequency("hour", "tweet3", 0, 210)
  chunks = 1, delta = 3600
  t=0   → idx 0 ✓
  t=60  → idx 0 ✓
  t=10  → idx 0 ✓
  t=120 → idx 0 ✓
  result = [4] ✓
```

---

### **2️⃣ Breaking Down the Solution**

**Brute Force Approach:**
> For each query, iterate through all recorded tweets for that name, check if each falls in `[startTime, endTime]`, compute its chunk index, and increment a counter array.

- Time per query: O(T) where T = number of tweets recorded for that name
- Space: O(total tweets)
- This is already optimal for the given constraints (10^4 calls, 10^4 endTime - startTime max)

**Can We Optimize?**
> Yes — if we sort the tweet timestamps on insert (or after insert), we can use **binary search** to find tweets in `[startTime, endTime]` quickly.

| Approach | `recordTweet` | `getTweetCountsPerFrequency` | Space |
|----------|--------------|------------------------------|-------|
| HashMap + list (brute) | O(1) | O(T) | O(T) |
| HashMap + sorted list + binary search | O(log T) amortized (via bisect) | O(log T + matches) | O(T) |

**Chosen Approach:** HashMap + unsorted list
> For an interview, the brute-force linear scan is clean, correct, and within constraints. Mention the binary-search optimization as a follow-up.

**Key Insight — Chunk Index Formula:**
```
chunk_index = (tweet_time - startTime) // delta

where delta = 60 (minute) | 3600 (hour) | 86400 (day)

Number of chunks = (endTime - startTime) // delta + 1
```
The `+ 1` accounts for the last (possibly shorter) chunk.

---

### **3️⃣ Writing Clean & Efficient Code**

#### ✅ Python Solution

```python
from collections import defaultdict
from math import floor


class TweetCounts:
    """
    Records tweet timestamps and answers frequency-bucketed count queries.

    recordTweet(tweetName, time)        — O(1) amortized
    getTweetCountsPerFrequency(...)     — O(T) where T = tweets for that name
    """

    # Map frequency string → chunk size in seconds
    FREQ_TO_DELTA = {
        "minute": 60,
        "hour": 3600,
        "day": 86400,
    }

    def __init__(self):
        # tweetName → list of recorded timestamps
        self._records: dict[str, list[int]] = defaultdict(list)

    def recordTweet(self, tweetName: str, time: int) -> None:
        """Store a tweet event at the given timestamp (seconds)."""
        self._records[tweetName].append(time)

    def getTweetCountsPerFrequency(
        self,
        freq: str,
        tweetName: str,
        startTime: int,
        endTime: int,
    ) -> list[int]:
        """
        Return a list where each element is the number of tweets
        in the corresponding time chunk of size `freq` within [startTime, endTime].

        Chunk index = (tweet_time - startTime) // delta
        Total chunks = (endTime - startTime) // delta + 1
        """
        delta = self.FREQ_TO_DELTA[freq]
        total_chunks = (endTime - startTime) // delta + 1
        counts = [0] * total_chunks

        for t in self._records.get(tweetName, []):
            if startTime <= t <= endTime:
                chunk_idx = (t - startTime) // delta
                counts[chunk_idx] += 1

        return counts
```

---

#### ✅ Java Solution

```java
import java.util.*;

/**
 * TweetCounts — Records tweet timestamps and answers bucketed count queries.
 *
 * recordTweet:                  O(1)
 * getTweetCountsPerFrequency:   O(T) where T = tweets stored for that name
 */
public class TweetCounts {

    private static final Map<String, Integer> FREQ_TO_DELTA = Map.of(
        "minute", 60,
        "hour",   3600,
        "day",    86400
    );

    // tweetName → list of recorded timestamps
    private final Map<String, List<Integer>> records;

    public TweetCounts() {
        records = new HashMap<>();
    }

    /**
     * Store a tweet event at the given timestamp (seconds).
     */
    public void recordTweet(String tweetName, int time) {
        records.computeIfAbsent(tweetName, k -> new ArrayList<>()).add(time);
    }

    /**
     * Return a list where each element is the number of tweets
     * in the corresponding time chunk within [startTime, endTime].
     *
     * Chunk index = (tweet_time - startTime) / delta
     * Total chunks = (endTime - startTime) / delta + 1
     */
    public List<Integer> getTweetCountsPerFrequency(
            String freq,
            String tweetName,
            int startTime,
            int endTime) {

        int delta = FREQ_TO_DELTA.get(freq);
        int totalChunks = (endTime - startTime) / delta + 1;
        int[] counts = new int[totalChunks];

        List<Integer> times = records.getOrDefault(tweetName, Collections.emptyList());
        for (int t : times) {
            if (t >= startTime && t <= endTime) {
                int chunkIdx = (t - startTime) / delta;
                counts[chunkIdx]++;
            }
        }

        // Convert int[] to List<Integer>
        List<Integer> result = new ArrayList<>(totalChunks);
        for (int count : counts) {
            result.add(count);
        }
        return result;
    }
}
```

---

#### ✅ Optimized Python — Binary Search (Follow-Up)

```python
import bisect
from collections import defaultdict


class TweetCountsOptimized:
    """
    Optimized version using sorted insertion + binary search.

    recordTweet:                 O(log T) — sorted insert via bisect.insort
    getTweetCountsPerFrequency:  O(log T + matches) — binary search + scan
    """

    FREQ_TO_DELTA = {"minute": 60, "hour": 3600, "day": 86400}

    def __init__(self):
        self._records: dict[str, list[int]] = defaultdict(list)

    def recordTweet(self, tweetName: str, time: int) -> None:
        """Insert into sorted list to enable binary search queries."""
        bisect.insort(self._records[tweetName], time)

    def getTweetCountsPerFrequency(
        self, freq: str, tweetName: str, startTime: int, endTime: int
    ) -> list[int]:
        delta = self.FREQ_TO_DELTA[freq]
        total_chunks = (endTime - startTime) // delta + 1
        counts = [0] * total_chunks

        times = self._records.get(tweetName, [])

        # Binary search to find range [startTime, endTime]
        lo = bisect.bisect_left(times, startTime)
        hi = bisect.bisect_right(times, endTime)

        for i in range(lo, hi):
            chunk_idx = (times[i] - startTime) // delta
            counts[chunk_idx] += 1

        return counts
```

---

### **4️⃣ Explaining the Code to the Interviewer**

**Step-by-step walkthrough:**

1. **`recordTweet(tweetName, time)`:**
   - Store the timestamp in a list keyed by tweet name
   - `defaultdict(list)` auto-initializes the list for new names
   - O(1) amortized (simple `append`)

2. **`getTweetCountsPerFrequency(freq, tweetName, startTime, endTime)`:**
   - Look up `delta` from the frequency string (60 / 3600 / 86400)
   - Compute `total_chunks = (endTime - startTime) // delta + 1`
   - Initialize a `counts` array of zeros with size `total_chunks`
   - Iterate recorded timestamps; for each one in `[startTime, endTime]`:
     - Compute `chunk_idx = (t - startTime) // delta`
     - Increment `counts[chunk_idx]`
   - Return `counts`

**Why `(endTime - startTime) // delta + 1`?**
```
Example: startTime=0, endTime=60, delta=60
  Without +1: 60 // 60 = 1 chunk (WRONG — misses chunk for time=60)
  With +1:    60 // 60 + 1 = 2 chunks ✓ → [0,59] and [60,60]

The +1 ensures the last chunk (which may be partial) is always included.
```

**Complexity Analysis:**

| Method | Time | Space | Notes |
|--------|------|-------|-------|
| `__init__` | O(1) | O(1) | Empty map |
| `recordTweet` | O(1) | O(1) amortized | Append to list |
| `getTweetCountsPerFrequency` | O(T) | O(C) | T=stored tweets, C=chunks |
| **Optimized recordTweet** | O(log T) | O(1) | Sorted insert |
| **Optimized getTweets** | O(log T + M) | O(C) | M=tweets in range |

> **Constraints check:** `endTime - startTime <= 10^4`, `delta >= 60` → max chunks = `10^4 / 60 + 1 ≈ 168`. The result list is always small.

---

### **5️⃣ Discussing Edge Cases & Testing**

#### Test Table

| Input | Expected | Edge Case Type |
|-------|----------|----------------|
| `recordTweet("t",0)`, `getTweets("minute","t",0,59)` | `[1]` | Single tweet, exact window boundary |
| `recordTweet("t",60)`, `getTweets("minute","t",0,60)` | `[0,1]` | Tweet at chunk boundary belongs to NEXT chunk |
| `getTweets("minute","unknown",0,60)` | `[0,0]` | Tweet name never recorded → all zeros |
| `getTweets("minute","t",0,0)` | `[0]` or `[1]` | Zero-length range → exactly 1 chunk |
| `recordTweet("t",0)` × 5, `getTweets("day","t",0,86399)` | `[5]` | All in one day chunk |
| `startTime = endTime = 500`, delta=60 | `[count]` | Single-point range |
| Time = 10^9, startTime near 10^9 | Valid result | Large timestamps (int overflow check in Java) |
| Multiple tweet names interleaved | Each name independent | Name isolation |
| Same time recorded twice | Count = 2 in that chunk | Duplicate timestamps allowed |

#### Boundary Verification

```python
# Test: chunk boundary
tc = TweetCounts()
tc.recordTweet("t", 59)
tc.recordTweet("t", 60)
assert tc.getTweetCountsPerFrequency("minute", "t", 0, 60) == [1, 1]
# t=59 → idx (59-0)//60 = 0  ✓
# t=60 → idx (60-0)//60 = 1  ✓

# Test: tweet outside range is excluded
tc2 = TweetCounts()
tc2.recordTweet("t", 100)
assert tc2.getTweetCountsPerFrequency("minute", "t", 0, 59) == [0]
# t=100 is outside [0,59], not counted ✓

# Test: zero-duration range
tc3 = TweetCounts()
tc3.recordTweet("t", 5)
assert tc3.getTweetCountsPerFrequency("minute", "t", 5, 5) == [1]
# chunks = (5-5)//60 + 1 = 1, t=5 → idx 0 ✓
```

#### Off-By-One Checks
- `total_chunks = (endTime - startTime) // delta + 1` — the `+1` is mandatory
- Range check: `startTime <= t <= endTime` (inclusive on both ends)
- `chunk_idx = (t - startTime) // delta` — always subtract `startTime` first (offset-based)

---

### **6️⃣ Debugging & Fixing Mistakes**

**Common Mistakes:**

❌ **Mistake 1: Forgetting `+ 1` in total chunks**
```python
# WRONG — misses the last partial chunk
total_chunks = (endTime - startTime) // delta

# RIGHT
total_chunks = (endTime - startTime) // delta + 1
```

❌ **Mistake 2: Using absolute time instead of offset for chunk index**
```python
# WRONG — chunk_idx is relative to startTime, not time=0
chunk_idx = t // delta

# RIGHT — subtract startTime to get offset
chunk_idx = (t - startTime) // delta
```

❌ **Mistake 3: Not filtering tweets outside [startTime, endTime]**
```python
# WRONG — includes tweets before startTime or after endTime
for t in self._records[tweetName]:
    counts[(t - startTime) // delta] += 1  # negative index if t < startTime!

# RIGHT — filter first
for t in self._records.get(tweetName, []):
    if startTime <= t <= endTime:
        counts[(t - startTime) // delta] += 1
```

❌ **Mistake 4: Java integer overflow with large timestamps**
```java
// WARN: time can be up to 10^9 — fits in int (max ~2.1×10^9)
// But intermediate (endTime - startTime) with large values → use int carefully
// Safe here since (endTime - startTime) <= 10^4 per constraints
int chunkIdx = (t - startTime) / delta;  // Fine within given constraints
```

**Debug Print Pattern:**
```python
def getTweetCountsPerFrequency(self, freq, tweetName, startTime, endTime):
    delta = self.FREQ_TO_DELTA[freq]
    total_chunks = (endTime - startTime) // delta + 1
    counts = [0] * total_chunks

    print(f"delta={delta}, chunks={total_chunks}")  # DEBUG
    for t in self._records.get(tweetName, []):
        if startTime <= t <= endTime:
            idx = (t - startTime) // delta
            print(f"  t={t} → chunk[{idx}]")  # DEBUG
            counts[idx] += 1

    return counts
```

---

## 🚀 Best Practices Demonstrated

✅ **Separation of concerns** — `recordTweet` only stores; `getTweetCountsPerFrequency` only queries  
✅ **Constant lookup** — Frequency → delta mapping in a dict/map avoids brittle `if-elif` chains  
✅ **Defensive coding** — `.get(tweetName, [])` returns empty list for unknown names (no KeyError)  
✅ **Formula verified** — Chunk count formula tested against all example cases before coding  
✅ **Follow-up ready** — Binary search optimization mentioned and implemented  
✅ **Off-by-one handled** — `+ 1` in chunk count, offset-based index, inclusive range filter  

---

## 📎 Related Problems

| # | Problem | Similarity |
|---|---------|-----------|
| [LeetCode 1347](https://leetcode.com/problems/minimum-number-of-steps-to-make-two-strings-anagram/) | Minimum Steps Anagram | Counting / frequency map |
| [LeetCode 981](https://leetcode.com/problems/time-based-key-value-store/) | Time-Based Key-Value Store | Timestamp storage + binary search |
| [LeetCode 729](https://leetcode.com/problems/my-calendar-i/) | My Calendar I | Interval/time design problem |
| Intercom Problem 01 | Fair Conversation Assignment | Class-based API design pattern |

---

## 🔗 Cross-References

- [HashMap fundamentals](../../DataStructures/HashTables/)
- [Binary Search](../../Algorithms/Searching/)
- [Time-Series Databases](../../Database/TimeSeries_Databases.md)

---

*Previous: [← 01 FairConversationAssignment](./01_FairConversationAssignment.md) | Next: [03 LongestSubstringNoRepeating →](./03_LongestSubstringNoRepeating.md)*
