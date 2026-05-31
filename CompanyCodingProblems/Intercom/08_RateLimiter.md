# 🟣 Intercom | Problem 08 — Sliding Window Rate Limiter

> **Source:** Common System Design Coding Problem · Directly relevant to Intercom's product
> **Difficulty:** 🟡 Medium
> **Topics:** Sliding Window · HashMap · Queue · Class Design · Time-Based

---

## 🔥 Ultimate Coding Interview Strategy

---

### **1️⃣ Understanding the Problem Clearly**

**Paraphrasing the Problem:**
> Design a rate limiter class that allows at most `maxRequests` within any sliding window of `windowSize` seconds for each customer. Implement:
> 1. `RateLimiter(maxRequests, windowSize)` — constructor
> 2. `boolean allowRequest(customerId, timestamp)` — returns true if request is allowed, false if rate-limited

**Why Intercom Would Ask This:**
- Intercom's API serves millions of requests — rate limiting is core infrastructure
- Tests class design + time-windowed counting (their favorite pattern)
- Same bucketing logic as their histogram/ping problems

**Clarifying Questions:**
- "Is the window fixed (e.g., per-minute) or sliding (last N seconds from now)?"
- "Should timestamps be monotonically increasing?"
- "Can multiple requests have the same timestamp?"
- "Should we track per-customer limits?"

**Confirmed:**
- **Sliding window** (not fixed) — any consecutive `windowSize` seconds
- Timestamps are in seconds, monotonically non-decreasing
- Per-customer tracking
- Multiple requests at same timestamp are separate

**Example:**
```
limiter = RateLimiter(maxRequests=3, windowSize=10)

limiter.allowRequest("alice", 1)   → true  (1 request in [0,10])
limiter.allowRequest("alice", 2)   → true  (2 requests in [0,10])
limiter.allowRequest("alice", 5)   → true  (3 requests in [0,10])
limiter.allowRequest("alice", 8)   → false (already 3 in window [0,10])
limiter.allowRequest("alice", 12)  → true  (window [3,12]: only t=5 in window → 1 request)
limiter.allowRequest("bob", 8)     → true  (bob has separate counter)
```

---

### **2️⃣ Breaking Down the Solution**

**Approach 1 — Queue per Customer (Sliding Window Log):**
- For each customer, maintain a queue of timestamps
- On each request: remove expired timestamps (older than `timestamp - windowSize`)
- If remaining count < maxRequests → allow and add timestamp
- Else → deny

**Approach 2 — Sliding Window Counter (Fixed Buckets):**
- Divide time into fixed buckets; use weighted overlap
- More memory-efficient but approximate

**Chosen: Queue per Customer (exact, clean for interview)**

| Approach | `allowRequest` | Space | Exactness |
|----------|---------------|-------|-----------|
| Queue (sliding log) | O(expired) amortized O(1) | O(maxRequests per customer) | Exact |
| Fixed window | O(1) | O(customers) | Approximate (burst at boundary) |
| Sliding window counter | O(1) | O(customers × buckets) | Approximate but better |

---

### **3️⃣ Writing Clean & Efficient Code**

#### ✅ Java Solution

```java
import java.util.*;

/**
 * Sliding Window Rate Limiter.
 * 
 * Allows at most maxRequests within any sliding window of windowSize seconds
 * per customer.
 *
 * Time:  O(1) amortized per call (expired entries cleaned lazily)
 * Space: O(C × maxRequests) where C = number of active customers
 */
public class RateLimiter {

    private final int maxRequests;
    private final int windowSize;
    // customerId → queue of request timestamps within the current window
    private final Map<String, Deque<Integer>> requestLog;

    public RateLimiter(int maxRequests, int windowSize) {
        if (maxRequests <= 0 || windowSize <= 0) {
            throw new IllegalArgumentException("maxRequests and windowSize must be positive");
        }
        this.maxRequests = maxRequests;
        this.windowSize = windowSize;
        this.requestLog = new HashMap<>();
    }

    /**
     * Check if a request is allowed for the given customer at the given time.
     * If allowed, records the request. If denied, does not record.
     *
     * @param customerId Unique identifier for the customer.
     * @param timestamp  Current time in seconds (monotonically non-decreasing).
     * @return true if the request is allowed, false if rate-limited.
     */
    public boolean allowRequest(String customerId, int timestamp) {
        Deque<Integer> queue = requestLog.computeIfAbsent(customerId, k -> new ArrayDeque<>());

        // Remove expired timestamps outside the sliding window
        int windowStart = timestamp - windowSize;
        while (!queue.isEmpty() && queue.peekFirst() <= windowStart) {
            queue.pollFirst();
        }

        // Check if under limit
        if (queue.size() < maxRequests) {
            queue.addLast(timestamp);
            return true;
        }

        return false; // Rate limited
    }
}
```

#### ✅ Python Solution

```python
from collections import defaultdict, deque


class RateLimiter:
    """
    Sliding Window Rate Limiter.
    
    Allows at most max_requests within any sliding window of window_size seconds
    per customer.
    
    Time:  O(1) amortized per call
    Space: O(C × max_requests) — C active customers
    """

    def __init__(self, max_requests: int, window_size: int):
        if max_requests <= 0 or window_size <= 0:
            raise ValueError("max_requests and window_size must be positive")
        self._max_requests = max_requests
        self._window_size = window_size
        self._request_log: dict[str, deque[int]] = defaultdict(deque)

    def allow_request(self, customer_id: str, timestamp: int) -> bool:
        """
        Returns True if request is allowed (under rate limit).
        Records the request if allowed.
        """
        queue = self._request_log[customer_id]

        # Evict expired timestamps
        window_start = timestamp - self._window_size
        while queue and queue[0] <= window_start:
            queue.popleft()

        # Check rate limit
        if len(queue) < self._max_requests:
            queue.append(timestamp)
            return True

        return False  # Rate limited
```

#### ✅ Extended — With Retry-After Header

```java
/**
 * Returns seconds until the next request will be allowed.
 * Returns 0 if the request is currently allowed.
 */
public int getRetryAfter(String customerId, int timestamp) {
    Deque<Integer> queue = requestLog.getOrDefault(customerId, new ArrayDeque<>());

    // Clean expired
    int windowStart = timestamp - windowSize;
    while (!queue.isEmpty() && queue.peekFirst() <= windowStart) {
        queue.pollFirst();
    }

    if (queue.size() < maxRequests) return 0; // Allowed now

    // Earliest request in window will expire at: earliest + windowSize + 1
    int earliest = queue.peekFirst();
    return earliest + windowSize - timestamp + 1;
}
```

---

### **4️⃣ Explaining the Code**

**Walkthrough:**
```
RateLimiter(maxRequests=3, windowSize=10)

allowRequest("alice", 1):
  queue = [], window_start = 1-10 = -9 (nothing to evict)
  size(0) < 3 → allow, queue = [1]

allowRequest("alice", 5):
  queue = [1], window_start = 5-10 = -5
  size(1) < 3 → allow, queue = [1, 5]

allowRequest("alice", 8):
  queue = [1, 5], window_start = 8-10 = -2
  size(2) < 3 → allow, queue = [1, 5, 8]

allowRequest("alice", 9):
  queue = [1, 5, 8], window_start = 9-10 = -1
  size(3) >= 3 → DENIED

allowRequest("alice", 12):
  queue = [1, 5, 8], window_start = 12-10 = 2
  evict: 1 <= 2 → remove → queue = [5, 8]
  size(2) < 3 → allow, queue = [5, 8, 12]
```

**Complexity:**

| Operation | Time | Space |
|-----------|------|-------|
| `allowRequest` | O(1) amortized | O(maxRequests) per customer |
| Overall space | — | O(C × maxRequests) |
| Eviction | O(expired) per call, O(1) amortized | — |

---

### **5️⃣ Edge Cases & Testing**

| Scenario | Input | Expected | Notes |
|----------|-------|----------|-------|
| Under limit | 2 requests, max=3, window=10 | Both true | Normal |
| Exactly at limit | 3 requests, max=3 | All true | Boundary |
| Over limit | 4th request in same window | false | Rate limited |
| Window expires | Request after windowSize passes | true | Old entries evicted |
| Different customers | alice at limit, bob requests | bob: true | Independent counters |
| Same timestamp | 3 requests at t=5, max=3 | All true | Duplicates allowed |
| Window=0 edge | Not valid (constructor rejects) | Exception | Input validation |
| Burst after reset | Fill → wait → fill again | All allowed | Window fully resets |

---

### **6️⃣ Connection to Intercom**

- **API Rate Limiting:** Intercom limits API calls per workspace
- **Agent Message Rate:** Prevent agents from sending too many messages too fast
- **Bot Cooldown:** AI agents have rate limits on responses
- **Same pattern as P01:** Class with state, HashMap per entity, time-based rules

---

## 📎 Related Problems

| Problem | Connection |
|---------|-----------|
| [LC 362 - Design Hit Counter](https://leetcode.com/problems/design-hit-counter/) | Same queue-based timestamp counting |
| [LC 359 - Logger Rate Limiter](https://leetcode.com/problems/logger-rate-limiter/) | Simpler version (dedup, not count) |
| [Intercom P01](./01_FairConversationAssignment.md) | Class design with time-based state |
| [Intercom P04](./04_UserPingTracker.md) | Time-bucketed counting per entity |

---

*Previous: [← 07 TaskScheduler](./07_TaskScheduler.md) | Next: [09 LRUCache →](./09_LRUCache.md)*
