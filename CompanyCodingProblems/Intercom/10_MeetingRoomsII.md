# 🟣 Intercom | Problem 10 — Meeting Rooms II (Minimum Conference Rooms)

> **Source:** LeetCode 253 · Scheduling with capacity constraints
> **Difficulty:** 🟡 Medium
> **Topics:** Sorting · Priority Queue (Min-Heap) · Intervals · Greedy

---

## 🔥 Ultimate Coding Interview Strategy

---

### **1️⃣ Understanding the Problem Clearly**

**Paraphrasing the Problem:**
> Given an array of meeting time intervals `[[start, end], ...]`, find the minimum number of conference rooms required so that no two overlapping meetings share a room.

**Why Intercom Would Ask This:**
- Models agent capacity: how many agents needed for concurrent conversations?
- Same scheduling + capacity logic as their load balancer problem
- Tests priority queue usage and interval reasoning

**Clarifying Questions:**
- "Are intervals half-open [start, end)? (i.e., if one ends at 10 and another starts at 10, do they conflict?)"
- "Can start == end? (zero-length meetings?)"
- "Is the input sorted?"

**Confirmed:**
- Intervals are `[start, end)` — a meeting ending at time 10 frees the room at time 10
- Input is NOT sorted
- start < end for all intervals

**Example:**
```
intervals = [[0,30], [5,10], [15,20]]

Timeline:
[0--------30]
    [5-10]
         [15-20]

At time 5: meetings [0,30] and [5,10] overlap → need 2 rooms
At time 15: meetings [0,30] and [15,20] overlap → need 2 rooms
Max concurrent = 2 → Answer: 2
```

---

### **2️⃣ Breaking Down the Solution**

**Approach 1: Min-Heap (Track room end times)**
1. Sort meetings by start time
2. Use a min-heap to track the earliest ending time among active meetings
3. For each meeting:
   - If heap's min end ≤ current start → reuse that room (poll & push new end)
   - Else → allocate new room (push new end)
4. Answer = heap size at end

**Approach 2: Sweep Line (Event counting)**
1. Create events: +1 at each start, -1 at each end
2. Sort events (by time, ties: end before start)
3. Track running sum → max running sum = answer

**Both are O(n log n). Min-Heap is more intuitive for interviews.**

---

### **3️⃣ Writing Clean & Efficient Code**

#### ✅ Java — Min-Heap Approach (Primary)

```java
import java.util.*;

/**
 * Meeting Rooms II: Find minimum rooms needed.
 * 
 * Sort by start time, use min-heap to track when rooms free up.
 * 
 * Time:  O(n log n) — sorting + heap operations
 * Space: O(n)       — heap can hold all meetings
 */
public class MeetingRoomsII {

    public int minMeetingRooms(int[][] intervals) {
        if (intervals == null || intervals.length == 0) return 0;

        // Sort by start time
        Arrays.sort(intervals, (a, b) -> a[0] - b[0]);

        // Min-heap of end times (earliest ending room on top)
        PriorityQueue<Integer> roomEndTimes = new PriorityQueue<>();

        for (int[] meeting : intervals) {
            int start = meeting[0];
            int end = meeting[1];

            // If the earliest-ending room is free by this meeting's start, reuse it
            if (!roomEndTimes.isEmpty() && roomEndTimes.peek() <= start) {
                roomEndTimes.poll(); // Free this room
            }

            // Allocate room (or reallocate freed room) with this meeting's end time
            roomEndTimes.offer(end);
        }

        // Heap size = total rooms needed (each entry = one active room)
        return roomEndTimes.size();
    }
}
```

#### ✅ Java — Sweep Line Approach (Alternative)

```java
import java.util.*;

public class MeetingRoomsIISweep {

    /**
     * Sweep line: +1 at start, -1 at end, find max overlap.
     * 
     * Time:  O(n log n) — sorting events
     * Space: O(n)       — event list
     */
    public int minMeetingRooms(int[][] intervals) {
        int n = intervals.length;
        int[] starts = new int[n];
        int[] ends = new int[n];

        for (int i = 0; i < n; i++) {
            starts[i] = intervals[i][0];
            ends[i] = intervals[i][1];
        }

        Arrays.sort(starts);
        Arrays.sort(ends);

        int rooms = 0, maxRooms = 0;
        int s = 0, e = 0;

        while (s < n) {
            if (starts[s] < ends[e]) {
                rooms++;  // New meeting starts before any ends
                s++;
            } else {
                rooms--;  // A meeting ended
                e++;
            }
            maxRooms = Math.max(maxRooms, rooms);
        }

        return maxRooms;
    }
}
```

#### ✅ Python — Min-Heap

```python
import heapq


def minMeetingRooms(intervals: list[list[int]]) -> int:
    """
    Find minimum conference rooms required.
    
    Sort by start, use min-heap of end times to track room availability.
    
    Time:  O(n log n)
    Space: O(n)
    """
    if not intervals:
        return 0

    intervals.sort(key=lambda x: x[0])  # Sort by start time

    # Min-heap: earliest room end time on top
    rooms = []
    heapq.heappush(rooms, intervals[0][1])

    for start, end in intervals[1:]:
        # If earliest-ending room is free, reuse it
        if rooms[0] <= start:
            heapq.heappop(rooms)

        # Assign this meeting to a room
        heapq.heappush(rooms, end)

    return len(rooms)
```

#### ✅ Python — Sweep Line

```python
def minMeetingRoomsSweep(intervals: list[list[int]]) -> int:
    """Sweep line approach: separate starts and ends, count max overlap."""
    starts = sorted(s for s, _ in intervals)
    ends = sorted(e for _, e in intervals)

    rooms = max_rooms = 0
    s = e = 0

    while s < len(intervals):
        if starts[s] < ends[e]:
            rooms += 1
            s += 1
        else:
            rooms -= 1
            e += 1
        max_rooms = max(max_rooms, rooms)

    return max_rooms
```

---

### **4️⃣ Explaining the Code**

**Min-Heap Walkthrough:**
```
intervals = [[0,30], [5,10], [15,20]]
Sorted by start: [[0,30], [5,10], [15,20]]

Step 1: meeting [0,30]
  heap empty → push 30 → heap = [30]
  rooms needed: 1

Step 2: meeting [5,10]
  heap.peek() = 30 > 5 → no room freed
  push 10 → heap = [10, 30]
  rooms needed: 2

Step 3: meeting [15,20]
  heap.peek() = 10 ≤ 15 → room freed! pop 10 → heap = [30]
  push 20 → heap = [20, 30]
  rooms needed: 2 (same)

Answer: heap.size() = 2
```

**Why the heap works:**
> Each entry in the heap represents one active room and when it becomes available. If the earliest available room works, we reuse it. Otherwise, we need a new room.

**Complexity:**

| Approach | Time | Space |
|----------|------|-------|
| Min-Heap | O(n log n) | O(n) |
| Sweep Line | O(n log n) | O(n) |

---

### **5️⃣ Edge Cases & Testing**

| Input | Expected | Explanation |
|-------|----------|-------------|
| `[[1,5]]` | 1 | Single meeting |
| `[[1,5],[6,10]]` | 1 | Sequential, no overlap |
| `[[1,5],[2,6],[3,7]]` | 3 | All overlap, need 3 rooms |
| `[[1,10],[10,20]]` | 1 | Touching boundaries don't overlap ([start, end)) |
| `[[1,3],[2,4],[3,5]]` | 2 | [1,3] and [2,4] overlap; [3,5] reuses room from [1,3] |
| `[]` | 0 | Empty input |
| `[[5,10],[0,5]]` | 1 | Unsorted but non-overlapping |

---

### **6️⃣ Connection to Intercom's Domain**

| Meeting Rooms Concept | Intercom Equivalent |
|----------------------|---------------------|
| Meeting | Conversation / Support ticket |
| Conference Room | Support Agent |
| Min rooms needed | Min agents for peak load |
| Start/End time | Conversation open/close time |
| Room reuse | Agent takes next conversation |

This is literally the **same problem as Intercom's load balancer** but framed differently. If you can solve Meeting Rooms II, you can handle their capacity planning questions.

---

## 📎 Related Problems

| Problem | Connection |
|---------|-----------|
| [LC 252 - Meeting Rooms](https://leetcode.com/problems/meeting-rooms/) | Simpler: just check if any overlap |
| [LC 56 - Merge Intervals](https://leetcode.com/problems/merge-intervals/) | Interval manipulation |
| [LC 1094 - Car Pooling](https://leetcode.com/problems/car-pooling/) | Sweep line with capacity |
| [Intercom P01](./01_FairConversationAssignment.md) | Same capacity/scheduling domain |

---

*Previous: [← 09 LRUCache](./09_LRUCache.md) | Next: [11 DesignLeaderboard →](./11_DesignLeaderboard.md)*
