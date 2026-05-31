# 🟣 Intercom | Problem 11 — Design Leaderboard

> **Source:** LeetCode 1244 · Real-time ranking system
> **Difficulty:** 🟡 Medium
> **Topics:** HashMap · TreeMap · Sorting · Class Design · Top-K

---

## 🔥 Ultimate Coding Interview Strategy

---

### **1️⃣ Understanding the Problem Clearly**

**Paraphrasing the Problem:**
> Design a Leaderboard class with three operations:
> 1. `addScore(playerId, score)` — Add score to player's total (create if new)
> 2. `top(K)` — Return sum of the top K players' scores
> 3. `reset(playerId)` — Reset player's score to 0 (remove from board)

**Why Intercom Would Ask This:**
- Tests class design with multiple data structures
- Intercom has team performance dashboards, agent metrics, SLA tracking
- Maps directly to "top performing agents" or "busiest conversations" features
- Similar to their metrics/histogram patterns

**Clarifying Questions:**
- "Does addScore accumulate? (i.e., calling addScore(1, 50) twice gives player 1 a score of 100?)"
- "What if K > number of players?"
- "Can score be negative?"

**Confirmed:**
- `addScore` accumulates (adds to existing score)
- K ≤ number of active players (guaranteed)
- Scores are always positive

---

### **2️⃣ Breaking Down the Solution**

**Approach 1 — HashMap + Sort for top() (Simple)**
- HashMap: playerId → total score
- `top(K)`: get all scores, sort descending, sum first K
- Trade-off: O(1) add/reset, O(n log n) top

**Approach 2 — HashMap + TreeMap (Balanced)**
- HashMap: playerId → score
- TreeMap: score → count (sorted frequency map)
- `top(K)`: iterate from highest score, accumulate
- Trade-off: O(log n) add/reset, O(K) top

**Approach 3 — HashMap + Min-Heap for top (Optimal top)**
- Only useful if top(K) is called very frequently with same K

**For interview: Start with Approach 1 (simple), discuss Approach 2 as optimization.**

---

### **3️⃣ Writing Clean & Efficient Code**

#### ✅ Java — HashMap + TreeMap (Balanced)

```java
import java.util.*;

/**
 * Leaderboard with efficient score tracking and top-K queries.
 * 
 * HashMap: playerId → score (O(1) lookup by player)
 * TreeMap: score → count (scores sorted for efficient top-K)
 *
 * addScore: O(log n)
 * top(K):   O(K + log n) — iterate K highest from TreeMap
 * reset:    O(log n)
 */
public class Leaderboard {

    private final Map<Integer, Integer> playerScores;  // playerId → total score
    private final TreeMap<Integer, Integer> scoreCounts; // score → how many players have this score

    public Leaderboard() {
        playerScores = new HashMap<>();
        scoreCounts = new TreeMap<>(Collections.reverseOrder()); // Descending order
    }

    /**
     * Add score to player's total. Creates player if new.
     */
    public void addScore(int playerId, int score) {
        int oldScore = playerScores.getOrDefault(playerId, 0);
        int newScore = oldScore + score;

        // Remove old score from TreeMap
        if (oldScore > 0) {
            decrementScoreCount(oldScore);
        }

        // Update player's score
        playerScores.put(playerId, newScore);

        // Add new score to TreeMap
        scoreCounts.merge(newScore, 1, Integer::sum);
    }

    /**
     * Return sum of top K players' scores.
     */
    public int top(int K) {
        int sum = 0;
        int remaining = K;

        for (Map.Entry<Integer, Integer> entry : scoreCounts.entrySet()) {
            int score = entry.getKey();
            int count = entry.getValue();

            int take = Math.min(remaining, count);
            sum += score * take;
            remaining -= take;

            if (remaining == 0) break;
        }

        return sum;
    }

    /**
     * Reset player's score to 0 (effectively remove from leaderboard).
     */
    public void reset(int playerId) {
        int oldScore = playerScores.getOrDefault(playerId, 0);
        if (oldScore > 0) {
            decrementScoreCount(oldScore);
        }
        playerScores.remove(playerId);
    }

    private void decrementScoreCount(int score) {
        int count = scoreCounts.getOrDefault(score, 0);
        if (count <= 1) {
            scoreCounts.remove(score);
        } else {
            scoreCounts.put(score, count - 1);
        }
    }
}
```

#### ✅ Java — Simple HashMap + Sort (Start with this in interview)

```java
import java.util.*;

/**
 * Simple Leaderboard: HashMap + sort on demand.
 * 
 * addScore: O(1)
 * top(K):   O(n log n) — sort all scores
 * reset:    O(1)
 * 
 * Good enough for small n or infrequent top() calls.
 */
public class LeaderboardSimple {

    private final Map<Integer, Integer> scores;

    public LeaderboardSimple() {
        scores = new HashMap<>();
    }

    public void addScore(int playerId, int score) {
        scores.merge(playerId, score, Integer::sum);
    }

    public int top(int K) {
        List<Integer> allScores = new ArrayList<>(scores.values());
        allScores.sort(Collections.reverseOrder());

        int sum = 0;
        for (int i = 0; i < K; i++) {
            sum += allScores.get(i);
        }
        return sum;
    }

    public void reset(int playerId) {
        scores.remove(playerId);
    }
}
```

#### ✅ Python — TreeMap Equivalent (SortedList)

```python
from collections import defaultdict
from sortedcontainers import SortedList


class Leaderboard:
    """
    Leaderboard using HashMap + SortedList for efficient top-K.
    
    addScore: O(log n)
    top(K):   O(K)
    reset:    O(log n)
    """

    def __init__(self):
        self._player_scores: dict[int, int] = {}
        self._sorted_scores = SortedList()  # Ascending by default

    def addScore(self, playerId: int, score: int) -> None:
        old_score = self._player_scores.get(playerId, 0)
        new_score = old_score + score

        if old_score > 0:
            self._sorted_scores.remove(old_score)

        self._player_scores[playerId] = new_score
        self._sorted_scores.add(new_score)

    def top(self, K: int) -> int:
        # SortedList is ascending, so take last K elements
        return sum(self._sorted_scores[-K:])

    def reset(self, playerId: int) -> None:
        old_score = self._player_scores.get(playerId, 0)
        if old_score > 0:
            self._sorted_scores.remove(old_score)
        del self._player_scores[playerId]
```

#### ✅ Python — Simple (No External Library)

```python
class LeaderboardSimple:
    """Simple HashMap + sort. Best for interviews without sortedcontainers."""

    def __init__(self):
        self._scores: dict[int, int] = {}

    def addScore(self, playerId: int, score: int) -> None:
        self._scores[playerId] = self._scores.get(playerId, 0) + score

    def top(self, K: int) -> int:
        return sum(sorted(self._scores.values(), reverse=True)[:K])

    def reset(self, playerId: int) -> None:
        del self._scores[playerId]
```

---

### **4️⃣ Explaining the Code**

**Walkthrough:**
```
lb = Leaderboard()
lb.addScore(1, 73)   → {1:73},   TreeMap: {73:1}
lb.addScore(2, 56)   → {1:73, 2:56},   TreeMap: {73:1, 56:1}
lb.addScore(3, 39)   → {1:73, 2:56, 3:39},   TreeMap: {73:1, 56:1, 39:1}
lb.addScore(4, 51)   → {1:73, 2:56, 3:39, 4:51},   TreeMap: {73:1, 56:1, 51:1, 39:1}
lb.addScore(5, 4)    → {1:73, ..., 5:4},   TreeMap: {73:1, 56:1, 51:1, 39:1, 4:1}

lb.top(1)            → 73 (just the highest)
lb.top(3)            → 73 + 56 + 51 = 180

lb.reset(1)          → remove player 1 (score 73)
                        TreeMap: {56:1, 51:1, 39:1, 4:1}

lb.addScore(2, 51)   → player 2 now has 56+51=107
                        TreeMap remove 56, add 107: {107:1, 51:1, 39:1, 4:1}

lb.top(3)            → 107 + 51 + 39 = 197
```

**Complexity Comparison:**

| Operation | Simple (Sort) | TreeMap | 
|-----------|---------------|---------|
| `addScore` | O(1) | O(log n) |
| `top(K)` | O(n log n) | O(K + log n) |
| `reset` | O(1) | O(log n) |

**When to use which:**
- **Simple:** n is small (<1000) or top() is called rarely
- **TreeMap:** n is large AND top() is called frequently

---

### **5️⃣ Edge Cases & Testing**

| Scenario | Input | Expected | Notes |
|----------|-------|----------|-------|
| Single player | addScore(1,50), top(1) | 50 | Basic |
| Accumulate | addScore(1,50), addScore(1,50), top(1) | 100 | Score adds up |
| Reset + add | addScore(1,50), reset(1), addScore(1,20), top(1) | 20 | Fresh start |
| Top all | 3 players, top(3) | Sum of all | K = n |
| Tie scores | Two players with 50, top(1) | 50 | Just one |
| Large K | 5 players, top(5) | Total sum | K = total players |
| Reset then top | 3 players, reset one, top(2) | Sum of remaining 2 | Evicted player excluded |

---

### **6️⃣ Connection to Intercom's Domain**

| Leaderboard Concept | Intercom Equivalent |
|-------------------|---------------------|
| Player | Support Agent |
| Score | Resolution count / CSAT score |
| top(K) | "Show me the top 5 agents this week" |
| reset | Start of new reporting period |
| addScore | Agent closes a conversation |

---

## 📎 Related Problems

| Problem | Connection |
|---------|-----------|
| [LC 703 - Kth Largest Element in a Stream](https://leetcode.com/problems/kth-largest-element-in-a-stream/) | Min-heap for top-K tracking |
| [LC 347 - Top K Frequent Elements](https://leetcode.com/problems/top-k-frequent-elements/) | Frequency + top-K |
| [LC 355 - Design Twitter](https://leetcode.com/problems/design-twitter/) | Multi-method class design |
| [Intercom P05](./05_TransactionHistogram.md) | Frequency counting & ranking |

---

*Previous: [← 10 MeetingRoomsII](./10_MeetingRoomsII.md) | Next: [12 SkillBasedRouting →](./12_SkillBasedRouting.md)*
