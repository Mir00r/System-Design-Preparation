# 🟣 Intercom | Problem 01 — Fair Conversation Assignment API

> **Source:** Intercom Coding Round (recurring, last 2+ years)
> **Difficulty:** 🟠 Medium-Hard
> **Topics:** OOP · Load Balancing · Priority Queue · Simulation · HashMap

---

## 🔥 Ultimate Coding Interview Strategy

---

### **1️⃣ Understanding the Problem Clearly**

**Paraphrasing the Problem:**
> We need to build an `AssignmentSystem` that manages how conversations are distributed among customer support operators. The system must assign conversations fairly — always picking the least-loaded operator, with a tie-breaking rule — and must respect per-operator capacity limits.

**Clarifying Questions to Ask:**
- "Is the default limit 2 for all operators unless `set_limit` is called?"
- "Does `get_assignment_queue(n)` simulate future assignments WITHOUT actually modifying state?"
- "What happens if all operators are at full capacity — return empty or raise error?"
- "For tie-breaking: by 'most recent assignment the earliest', do you mean the one who waited the longest since their last assignment?"
- "Are operator names guaranteed unique?"
- "Can `set_limit` be called after assignments have already been made?"

**Confirmed Assumptions:**
- Default limit = 2 per operator
- `get_assignment_queue(n)` → **read-only simulation**, does NOT change actual state
- Tie-breaking: same count → operator whose last assignment happened **earliest** (i.e., waited longest)
- If an operator has never been assigned, their "last assigned time" is effectively 0 (treated as earliest)
- If all operators are full, stop and return what we have

**Walking Through the Example:**
```
operators = ["Alice", "Bob", "Charlie"]
Bob.limit = 4, Charlie.limit = 3, Alice.limit = 2 (default)

Initial state:
  Alice:   count=0, limit=2, last_assigned=t0
  Bob:     count=0, limit=4, last_assigned=t0
  Charlie: count=0, limit=3, last_assigned=t0

get_assignment_queue(4):
  Round 1 → all tied at count=0, same last_assigned → insertion order → Alice
  Round 2 → Alice:1, Bob:0, Charlie:0 → Bob (least load)
  Round 3 → Alice:1, Bob:1, Charlie:0 → Charlie
  Round 4 → Alice:1, Bob:1, Charlie:1 → tied! Alice's last was earliest → Alice
  Result: ["Alice", "Bob", "Charlie", "Alice"] ✓
```

---

### **2️⃣ Breaking Down the Solution**

**Brute Force Approach — O(n) per operation:**
- Store operators in a list/map with their current counts and last-assigned timestamps
- On each assignment: iterate all operators, find the one with min count (tie-break by last assigned time)
- `get_assignment_queue(n)`: deep copy state, simulate n rounds

**Why Brute Force is Acceptable Here:**
> With typical operator counts (10–1000 agents in a support team), O(n) per assignment is completely fine. We do NOT need a heap for correctness, though a heap gives O(log n) per operation.

**Optimized Approach — Min-Heap O(log n) per operation:**
- Use a priority queue ordered by `(count, last_assigned_time, insertion_index)`
- On assignment: pop the minimum, increment count, re-push
- For simulation: clone the heap and simulate

**Choosing the Approach:**
> Brute force (O(n) scan) is clean and readable — ideal for an interview. The heap optimization is worth mentioning as a follow-up.

**Key Data to Track Per Operator:**
```
operator_name  → current assignment count
operator_name  → assignment limit
operator_name  → last assigned timestamp (global monotonic counter)
insertion_order → for stable tie-breaking when timestamps are equal (initial state)
```

**Tie-Breaking Summary:**
```
Priority = (count ASC, last_assigned_time ASC, insertion_index ASC)
Lower is better — operator with fewest assignments, who waited longest, listed first
```

---

### **3️⃣ Writing Clean & Efficient Code**

#### ✅ Python Solution

```python
import copy


class AssignmentSystem:
    """
    Load-balanced conversation assignment system.
    
    Assigns conversations to operators with:
    - Least current assignments (primary sort)
    - Earliest last-assignment time (tie-breaker)
    - Insertion order (final stable tie-breaker)
    """

    DEFAULT_LIMIT = 2

    def __init__(self, operators: list[str]):
        """
        Initialize with a list of operator names.
        Insertion order is preserved for stable tie-breaking.
        """
        self._operators = operators
        self._counts = {op: 0 for op in operators}
        self._limits = {op: self.DEFAULT_LIMIT for op in operators}
        self._last_assigned = {op: 0 for op in operators}
        self._insertion_index = {op: i for i, op in enumerate(operators)}
        self._time = 0  # Global monotonic counter

    def set_limit(self, operator_name: str, n: int) -> None:
        """Set maximum conversations an operator can handle."""
        if operator_name not in self._limits:
            raise ValueError(f"Unknown operator: {operator_name}")
        self._limits[operator_name] = n

    def assign(self, conversation_id: int) -> str | None:
        """
        Assign a conversation to the next available operator.
        Returns operator name, or None if all operators are full.
        """
        self._time += 1
        operator = self._pick_next(self._counts, self._last_assigned)

        if operator is None:
            return None  # All operators at capacity

        self._counts[operator] += 1
        self._last_assigned[operator] = self._time
        return operator

    def get_assignment_queue(self, n: int) -> list[str]:
        """
        Simulate the next n assignments WITHOUT modifying actual state.
        Returns a list of operator names in assignment order.
        """
        # Deep copy mutable state for simulation
        sim_counts = dict(self._counts)
        sim_last = dict(self._last_assigned)
        sim_time = self._time

        result = []
        for _ in range(n):
            sim_time += 1
            operator = self._pick_next(sim_counts, sim_last)

            if operator is None:
                break  # All operators at capacity

            sim_counts[operator] += 1
            sim_last[operator] = sim_time
            result.append(operator)

        return result

    # ------------------------------------------------------------------
    # Private helpers
    # ------------------------------------------------------------------

    def _pick_next(
        self,
        counts: dict[str, int],
        last_assigned: dict[str, int],
    ) -> str | None:
        """
        Select the best available operator using:
            1. Minimum current assignment count
            2. Earliest last-assigned timestamp (longest wait)
            3. Earliest insertion index (stable order)
        Returns None if no operator is available.
        """
        best = None
        best_key = None

        for op in self._operators:
            if counts[op] >= self._limits[op]:
                continue  # Skip operators at capacity

            # Sort key: (count, last_assigned_time, insertion_index)
            key = (counts[op], last_assigned[op], self._insertion_index[op])

            if best_key is None or key < best_key:
                best = op
                best_key = key

        return best
```

---

#### ✅ Java Solution

```java
import java.util.*;

/**
 * Load-balanced conversation assignment system.
 *
 * Assigns conversations to operators with:
 *   1. Least current assignments (primary)
 *   2. Earliest last-assignment timestamp (tie-breaker)
 *   3. Insertion order (stable final tie-breaker)
 */
public class AssignmentSystem {

    private static final int DEFAULT_LIMIT = 2;

    private final List<String> operators;         // Preserves insertion order
    private final Map<String, Integer> counts;     // Current assignment count
    private final Map<String, Integer> limits;     // Max assignments allowed
    private final Map<String, Integer> lastAssigned; // Timestamp of last assignment
    private final Map<String, Integer> insertionIndex; // For stable tie-breaking
    private int timeCounter;                       // Global monotonic clock

    public AssignmentSystem(List<String> operators) {
        this.operators = new ArrayList<>(operators);
        this.counts = new HashMap<>();
        this.limits = new HashMap<>();
        this.lastAssigned = new HashMap<>();
        this.insertionIndex = new HashMap<>();
        this.timeCounter = 0;

        for (int i = 0; i < operators.size(); i++) {
            String op = operators.get(i);
            counts.put(op, 0);
            limits.put(op, DEFAULT_LIMIT);
            lastAssigned.put(op, 0);
            insertionIndex.put(op, i);
        }
    }

    /**
     * Set maximum conversations an operator can handle.
     */
    public void setLimit(String operatorName, int n) {
        if (!limits.containsKey(operatorName)) {
            throw new IllegalArgumentException("Unknown operator: " + operatorName);
        }
        limits.put(operatorName, n);
    }

    /**
     * Assign a conversation to the next available operator.
     * Returns operator name, or null if all operators are full.
     */
    public String assign(int conversationId) {
        timeCounter++;
        String operator = pickNext(counts, lastAssigned);

        if (operator == null) return null;

        counts.put(operator, counts.get(operator) + 1);
        lastAssigned.put(operator, timeCounter);
        return operator;
    }

    /**
     * Simulate the next n assignments WITHOUT modifying actual state.
     * Returns list of operator names in assignment order.
     */
    public List<String> getAssignmentQueue(int n) {
        // Copy mutable state for simulation
        Map<String, Integer> simCounts = new HashMap<>(counts);
        Map<String, Integer> simLast = new HashMap<>(lastAssigned);
        int simTime = timeCounter;

        List<String> result = new ArrayList<>();

        for (int i = 0; i < n; i++) {
            simTime++;
            String operator = pickNext(simCounts, simLast);

            if (operator == null) break; // All at capacity

            simCounts.put(operator, simCounts.get(operator) + 1);
            simLast.put(operator, simTime);
            result.add(operator);
        }

        return result;
    }

    // ----------------------------------------------------------------
    // Private helpers
    // ----------------------------------------------------------------

    /**
     * Select best available operator by:
     *   (count ASC, last_assigned ASC, insertion_index ASC)
     */
    private String pickNext(
            Map<String, Integer> countState,
            Map<String, Integer> lastState) {

        String best = null;
        int[] bestKey = null; // [count, lastAssigned, insertionIndex]

        for (String op : operators) {
            if (countState.get(op) >= limits.get(op)) continue;

            int[] key = {
                countState.get(op),
                lastState.get(op),
                insertionIndex.get(op)
            };

            if (bestKey == null || compareKeys(key, bestKey) < 0) {
                best = op;
                bestKey = key;
            }
        }

        return best;
    }

    private int compareKeys(int[] a, int[] b) {
        if (a[0] != b[0]) return Integer.compare(a[0], b[0]);
        if (a[1] != b[1]) return Integer.compare(a[1], b[1]);
        return Integer.compare(a[2], b[2]);
    }
}
```

---

### **4️⃣ Explaining the Code to the Interviewer**

**Step-by-step walkthrough:**

1. **`__init__` / constructor:** We store each operator's `count`, `limit`, `last_assigned` time, and their `insertion_index` (to break ties in initial state where all timestamps are equal). A global `time` counter tracks ordering.

2. **`set_limit(operator, n)`:** Simple map update. Can be called any time.

3. **`assign(conversation_id)`:**
   - Increment global time counter (this is the "clock tick")
   - Call `_pick_next` with the live state maps
   - If an operator is found, update their count and record the current time
   - Return the operator name

4. **`_pick_next(counts, last_assigned)`:**
   - Iterate all operators; skip any at capacity
   - Build sort key `(count, last_assigned_time, insertion_index)`
   - Track the minimum — this is the correct operator per the rules

5. **`get_assignment_queue(n)`:**
   - **Crucially**: shallow copy `counts` and `last_assigned` dicts into simulation copies
   - Run n rounds of `_pick_next` on the simulation copies
   - The real state is never touched

**Complexity Analysis:**

| Method | Time Complexity | Space Complexity | Notes |
|--------|----------------|------------------|-------|
| `__init__` | O(k) | O(k) | k = number of operators |
| `set_limit` | O(1) | O(1) | Hash map update |
| `assign` | O(k) | O(1) | Linear scan of operators |
| `get_assignment_queue(n)` | O(n × k) | O(k) | n simulation rounds × k scan |

> **Alternative with Heap:** `assign` → O(log k), `get_assignment_queue(n)` → O(n log k + k log k). Worth mentioning, but linear scan is cleaner for an interview.

---

### **5️⃣ Discussing Edge Cases & Testing**

#### Full Verification of Given Example

```python
# Setup
operators = ["Alice", "Bob", "Charlie"]
system = AssignmentSystem(operators)
system.set_limit("Bob", 4)
system.set_limit("Charlie", 3)
# Alice.limit = 2 (default)

# get_assignment_queue(4) — no assignments yet
print(system.get_assignment_queue(4))
# Expected: ["Alice", "Bob", "Charlie", "Alice"]

# Make actual assignments
system.assign(101)  # → Alice  (count: Alice=1, Bob=0, Charlie=0)
system.assign(102)  # → Bob    (count: Alice=1, Bob=1, Charlie=0)
system.assign(103)  # → Charlie(count: Alice=1, Bob=1, Charlie=1)
system.assign(104)  # → Alice  (count: Alice=2, Bob=1, Charlie=1)
# Alice is now FULL (2/2)

# get_assignment_queue(5)
print(system.get_assignment_queue(5))
# Expected: ['Bob', 'Charlie', 'Bob', 'Charlie', 'Bob']
```

#### Trace for `get_assignment_queue(5)` after 4 assigns:
```
State: Alice(2/2 FULL), Bob(1/4, last=t2), Charlie(1/3, last=t3)

Sim t5: Bob(1,t2) vs Charlie(1,t3) → Bob (t2 < t3)   → Bob:2
Sim t6: Bob(2,t5) vs Charlie(1,t3) → Charlie (count 1 < 2)  → Charlie:2
Sim t7: Bob(2,t5) vs Charlie(2,t6) → Bob (t5 < t6)   → Bob:3
Sim t8: Bob(3,t7) vs Charlie(2,t6) → Charlie (count 2 < 3)  → Charlie:3
Sim t9: Bob(3,t7) vs Charlie(3,t8) → Bob (t7 < t8)   → Bob:4

Result: ['Bob','Charlie','Bob','Charlie','Bob'] ✓
```

#### Edge Case Test Table

| Scenario | Input | Expected Output | Notes |
|----------|-------|-----------------|-------|
| Normal round-robin | 3 ops, all limit=2, assign 6 | Each gets 2 in balanced order | Happy path |
| All operators full | All at limit, call `assign` | Returns `None` | Capacity exceeded |
| All full in queue | All at limit, `get_queue(5)` | `[]` | Returns empty list |
| Single operator | `["Alice"]`, limit=3, `assign` 5 times | Alice,Alice,Alice,None,None | 3 then None |
| `set_limit` after assigns | Assign 2, `set_limit("Alice",1)` | Alice can't get more | Limit retroactive? |
| `get_queue` doesn't mutate | Call `get_queue(4)`, then `assign` | Same result as fresh state | Read-only contract |
| All tied in initial state | 3 ops, all 0 assignments | Insertion order breaks tie | Initial ordering |
| One operator limits 0 | `set_limit("Bob", 0)` | Bob is never assigned | Zero capacity |

#### Off-By-One Checks
- `count >= limit` not `count > limit` — operator is full AT the limit, not after
- Simulation time counter starts AFTER real time counter (`sim_time = self._time`)
- Insertion index is 0-based; comparison is consistent

---

### **6️⃣ Debugging & Fixing Mistakes**

**Common Mistakes:**

❌ **Mistake 1: `get_assignment_queue` modifies real state**
```python
# WRONG — modifying self._counts directly
self._counts[operator] += 1
```
```python
# RIGHT — work on a copy
sim_counts = dict(self._counts)  # shallow copy is fine for int values
```

❌ **Mistake 2: Tie-breaking ignores insertion order**
```python
# WRONG — same last_assigned time (initial state) gives arbitrary result
key = (counts[op], last_assigned[op])
```
```python
# RIGHT — insertion index as final tie-breaker
key = (counts[op], last_assigned[op], insertion_index[op])
```

❌ **Mistake 3: Simulation clock doesn't continue from real clock**
```python
# WRONG — sim starts from 0, incorrectly treats old assignments as newer
sim_time = 0
```
```python
# RIGHT — simulation continues from where real clock stopped
sim_time = self._time
```

❌ **Mistake 4: `set_limit` not checked against operator existence**
```python
# WRONG — silently creates invalid operator
self._limits[operator_name] = n
```
```python
# RIGHT — validate operator exists
if operator_name not in self._limits:
    raise ValueError(f"Unknown operator: {operator_name}")
```

**Debug Print Statements for Tracing:**
```python
def _pick_next(self, counts, last_assigned):
    candidates = []
    for op in self._operators:
        if counts[op] < self._limits[op]:
            key = (counts[op], last_assigned[op], self._insertion_index[op])
            candidates.append((key, op))
            print(f"  Candidate {op}: {key}")  # DEBUG
    candidates.sort()
    return candidates[0][1] if candidates else None
```

---

## 🚀 Best Practices Demonstrated

✅ **Think before you code** — Verified tie-breaking logic with example trace before implementing  
✅ **Descriptive variable names** — `insertion_index`, `last_assigned`, `sim_counts` over `x`, `d`, `c`  
✅ **Modular functions** — `_pick_next` extracted as reusable helper shared by `assign` and `get_assignment_queue`  
✅ **Immutable contract honored** — `get_assignment_queue` never touches `self._counts`  
✅ **Edge cases handled** — Capacity exceeded returns `None`/early stop; unknown operator raises `ValueError`  
✅ **Complexity stated** — O(k) per assign, O(n×k) for queue simulation  

---

## 📎 Related Problems

- [LeetCode 1405 - Longest Happy String](https://leetcode.com/problems/longest-happy-string/) — Greedy with heap
- [LeetCode 1834 - Single-Threaded CPU](https://leetcode.com/problems/single-threaded-cpu/) — Scheduling with priority queue
- [LeetCode 630 - Course Schedule III](https://leetcode.com/problems/course-schedule-iii/) — Greedy scheduling
