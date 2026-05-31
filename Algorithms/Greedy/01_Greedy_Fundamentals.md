# 🎯 Greedy Algorithms: Make the Best Choice Now! 💰

> **"Choose what looks best at the moment - sometimes greed is good!"**

Master **greedy algorithms** - the intuitive approach that makes locally optimal choices to find global solutions. Essential for optimization problems in scheduling, resource allocation, and more! 🚀

---

## 📋 Table of Contents

1. [What is Greedy?](#-what-is-greedy)
2. [Greedy vs Dynamic Programming](#-greedy-vs-dynamic-programming)
3. [Classic Patterns](#-classic-greedy-patterns)
4. [Activity Selection](#-activity-selection)
5. [Huffman Coding](#-huffman-coding)
6. [Practice Problems](#-practice-problems)

---

## 🎯 What is Greedy?

**Greedy Algorithm** = Make the locally optimal choice at each step, hoping to find global optimum.

### **The Greedy Strategy**:
```
At each step:
1. Make the BEST choice NOW
2. Never reconsider past choices
3. Hope it leads to optimal solution
```

### **When Does Greedy Work?**

✅ **Greedy Choice Property** - Local optimum → Global optimum  
✅ **Optimal Substructure** - Optimal solution contains optimal sub-solutions  

```java
// Example: Coin Change (greedy works for certain denominations)
// Coins: [25, 10, 5, 1] cents
// Target: 41 cents

// Greedy approach:
25 + 10 + 5 + 1 = 41 ✅ (4 coins - optimal!)

// Why it works: US coins have special property
```

---

## ⚖️ Greedy vs Dynamic Programming

| Aspect | Greedy | Dynamic Programming |
|--------|--------|---------------------|
| **Decision** | Make best choice NOW | Consider all choices |
| **Reconsider?** | No | Yes |
| **Complexity** | Usually O(n log n) | Usually O(n²) or more |
| **Guarantee** | Sometimes optimal | Always optimal |
| **Example** | Activity Selection | 0/1 Knapsack |

### **Example: Coin Change**
```java
// Greedy (doesn't always work!)
Coins: [10, 7, 1]
Target: 14

Greedy: 10 + 1 + 1 + 1 + 1 = 14 (5 coins) ❌
Optimal: 7 + 7 = 14 (2 coins) ✅

// For some coin systems, greedy fails!
```

---

## 🎨 Classic Greedy Patterns

### **Pattern 1: Activity Selection**

**Problem**: Select maximum number of non-overlapping activities.

```java
static class Activity {
    int start, end;
    Activity(int start, int end) {
        this.start = start;
        this.end = end;
    }
}

public int maxActivities(Activity[] activities) {
    // Greedy choice: Sort by end time
    Arrays.sort(activities, (a, b) -> a.end - b.end);
    
    int count = 1;
    int lastEnd = activities[0].end;
    
    for (int i = 1; i < activities.length; i++) {
        if (activities[i].start >= lastEnd) {
            count++;
            lastEnd = activities[i].end;
        }
    }
    
    return count;
}
```

**Visual**:
```
Activities (sorted by end time):
A: |-----|
B:   |-----|
C:       |-----|
D:    |----|

Selected: A, C (max non-overlapping)
```

**Why Greedy Works?**  
Selecting activity ending earliest leaves most room for others!

---

### **Pattern 2: Fractional Knapsack**

**Problem**: Maximize value with fractional items allowed.

```java
static class Item {
    int value, weight;
    Item(int value, int weight) {
        this.value = value;
        this.weight = weight;
    }
}

public double fractionalKnapsack(int capacity, Item[] items) {
    // Greedy choice: Sort by value/weight ratio
    Arrays.sort(items, (a, b) -> 
        Double.compare((double) b.value / b.weight, 
                      (double) a.value / a.weight));
    
    double totalValue = 0;
    int remainingCapacity = capacity;
    
    for (Item item : items) {
        if (remainingCapacity >= item.weight) {
            // Take whole item
            totalValue += item.value;
            remainingCapacity -= item.weight;
        } else {
            // Take fraction
            totalValue += item.value * 
                         ((double) remainingCapacity / item.weight);
            break;
        }
    }
    
    return totalValue;
}
```

**Example**:
```
Capacity: 50
Items:
- Value: 60, Weight: 10 → Ratio: 6
- Value: 100, Weight: 20 → Ratio: 5
- Value: 120, Weight: 30 → Ratio: 4

Greedy order: Take all of item 1 (60), all of item 2 (100),
              2/3 of item 3 (80)
Total: 240 ✅
```

---

### **Pattern 3: Interval Scheduling**

**Problem**: Minimum rooms needed for meetings.

```java
public int minMeetingRooms(int[][] intervals) {
    int[] starts = new int[intervals.length];
    int[] ends = new int[intervals.length];
    
    for (int i = 0; i < intervals.length; i++) {
        starts[i] = intervals[i][0];
        ends[i] = intervals[i][1];
    }
    
    Arrays.sort(starts);
    Arrays.sort(ends);
    
    int rooms = 0, endPtr = 0;
    
    for (int start : starts) {
        if (start < ends[endPtr]) {
            rooms++;  // Need new room
        } else {
            endPtr++;  // Reuse room
        }
    }
    
    return rooms;
}
```

**Visual**:
```
Meetings:
[0, 30)  |-------------|
[5, 10)      |---|
[15, 20)           |---|

Starts: [0, 5, 15]
Ends:   [10, 20, 30]

Process:
0 < 10 → Need room (1)
5 < 10 → Need room (2)
15 >= 10 → Reuse room (2)

Result: 2 rooms needed
```

---

### **Pattern 4: Huffman Coding**

**Problem**: Optimal prefix-free encoding.

```java
static class Node {
    char ch;
    int freq;
    Node left, right;
    
    Node(char ch, int freq) {
        this.ch = ch;
        this.freq = freq;
    }
}

public Node buildHuffmanTree(Map<Character, Integer> frequencies) {
    PriorityQueue<Node> pq = new PriorityQueue<>((a, b) -> a.freq - b.freq);
    
    // Create leaf nodes
    for (Map.Entry<Character, Integer> entry : frequencies.entrySet()) {
        pq.offer(new Node(entry.getKey(), entry.getValue()));
    }
    
    // Build tree bottom-up
    while (pq.size() > 1) {
        Node left = pq.poll();
        Node right = pq.poll();
        
        Node parent = new Node('\0', left.freq + right.freq);
        parent.left = left;
        parent.right = right;
        
        pq.offer(parent);
    }
    
    return pq.poll();
}

public void generateCodes(Node root, String code, Map<Character, String> codes) {
    if (root == null) return;
    
    if (root.ch != '\0') {
        codes.put(root.ch, code);
    }
    
    generateCodes(root.left, code + "0", codes);
    generateCodes(root.right, code + "1", codes);
}
```

**Example**:
```
Text: "aabbbcccc"
Frequencies: a:2, b:3, c:4

Huffman Tree:
        (9)
       /   \
     (5)   c:4
     / \
   a:2 b:3

Codes: a=00, b=01, c=1
Encoding: "aabbbcccc" → "00000101011111" (14 bits)
Fixed-length: 9 chars * 2 bits = 18 bits
Savings: 22% ✅
```

---

### **Pattern 5: Job Sequencing**

**Problem**: Schedule jobs with deadlines to maximize profit.

```java
static class Job {
    char id;
    int deadline, profit;
    Job(char id, int deadline, int profit) {
        this.id = id;
        this.deadline = deadline;
        this.profit = profit;
    }
}

public int maxProfit(Job[] jobs) {
    // Greedy: Sort by profit (descending)
    Arrays.sort(jobs, (a, b) -> b.profit - a.profit);
    
    int maxDeadline = 0;
    for (Job job : jobs) {
        maxDeadline = Math.max(maxDeadline, job.deadline);
    }
    
    int[] slots = new int[maxDeadline + 1];
    Arrays.fill(slots, -1);
    
    int totalProfit = 0;
    
    for (Job job : jobs) {
        // Find latest available slot
        for (int i = job.deadline; i > 0; i--) {
            if (slots[i] == -1) {
                slots[i] = job.id;
                totalProfit += job.profit;
                break;
            }
        }
    }
    
    return totalProfit;
}
```

**Example**:
```
Jobs:
ID  Deadline  Profit
A      2       100
B      1       19
C      2       27
D      1       25
E      3       15

Sorted by profit: A(100), B(19), D(25), C(27), E(15)

Schedule:
Slot 1: A → Slot 2 (deadline=2)
Slot 2: D → Slot 1 (deadline=1)
Slot 3: C → Slot 1 full, Slot 2 full, skip
...

Result: A + B + E = 142
```

---

## 🏢 Real-World Applications

### **Network Routing** 🌐
- Dijkstra's algorithm (greedy shortest path)
- Prim's/Kruskal's MST

### **Data Compression** 📦
- Huffman coding (ZIP, JPEG)
- Run-length encoding

### **Scheduling** 📅
- CPU scheduling (Shortest Job First)
- Task scheduling with deadlines

### **Resource Allocation** 💰
- Fractional knapsack (investment)
- Activity selection (room booking)

---

## 🧩 Practice Problems

### 🟢 **Easy**
1. [Assign Cookies](https://leetcode.com/problems/assign-cookies/)
2. [Lemonade Change](https://leetcode.com/problems/lemonade-change/)
3. [Best Time to Buy and Sell Stock II](https://leetcode.com/problems/best-time-to-buy-and-sell-stock-ii/)

### 🟡 **Medium**
4. [Jump Game](https://leetcode.com/problems/jump-game/)
5. [Jump Game II](https://leetcode.com/problems/jump-game-ii/)
6. [Gas Station](https://leetcode.com/problems/gas-station/)
7. [Partition Labels](https://leetcode.com/problems/partition-labels/)
8. [Non-overlapping Intervals](https://leetcode.com/problems/non-overlapping-intervals/)
9. [Minimum Number of Arrows](https://leetcode.com/problems/minimum-number-of-arrows-to-burst-balloons/)

### 🔴 **Hard**
10. [Candy](https://leetcode.com/problems/candy/)
11. [IPO](https://leetcode.com/problems/ipo/)

---

## ⚠️ Common Pitfalls

### **1. Greedy Doesn't Always Work**
```java
// Example: 0/1 Knapsack
Capacity: 10
Items: {Value: 100, Weight: 10}, {Value: 60, Weight: 1}

Greedy (by value): Take first → 100 ✅
Greedy (by ratio): Take second → 60 ❌

Lesson: Need DP for 0/1 Knapsack!
```

### **2. Wrong Greedy Choice**
```java
// Activity Selection
❌ Sort by start time → Suboptimal
✅ Sort by end time → Optimal
```

---

## 💡 Pro Tips

✅ **Prove greedy works** before coding  
✅ **Sorting** often reveals greedy choice  
✅ **Test edge cases** - greedy can fail  
✅ **Consider DP** if greedy doesn't work  
✅ **Draw timeline** for interval problems  

---

## 🎯 Key Takeaways

1. **Greedy = Best choice NOW**, never reconsider
2. **Works when**: Greedy choice + Optimal substructure
3. **Faster than DP** but doesn't always work
4. **Common patterns**: Activity, Knapsack, Huffman, Intervals
5. **Sorting** is often key to greedy strategy

---

## 🚀 What's Next?

➡️ **[String Algorithms](../StringAlgorithms/01_String_Matching.md)**  
➡️ **[Practice Problems](../ProblemSets/README.md)**  

---

**🎮 Achievement Unlocked: Greedy Master!** 🏆

*Remember: "Sometimes the best choice now is the best choice forever!"* 🎯✨

---

*Previous: [← DP Patterns](../DynamicProgramming/02_DP_Patterns.md) | Next: [String Algorithms →](../StringAlgorithms/01_String_Matching.md)*

