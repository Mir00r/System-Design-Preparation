# 💻 How to Crack Coding Rounds: Patterns, Strategy & Execution

> *"Coding interviews aren't about memorizing solutions — they're about recognizing patterns. Once you know the 15 core patterns, you can solve 90% of interview problems by mapping them to a pattern you've practiced."*

**⏱️ Estimated Time**: 25 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [InterviewPrep README](./README.md)

---

## 📋 Table of Contents
1. [The 15 Core Patterns](#-core-patterns)
2. [Problem-Solving Framework](#-framework)
3. [Time Management](#-time-management)
4. [Communication Strategy](#-communication)
5. [Common Mistakes](#-common-mistakes)
6. [Practice Plan](#-practice-plan)

---

## 🧩 Core Patterns

```
MASTER THESE 15 PATTERNS TO SOLVE 90% OF PROBLEMS:

┌────────────────────────┬─────────────────────────────────────────────┐
│ Pattern                │ When to Use                                 │
├────────────────────────┼─────────────────────────────────────────────┤
│ 1. Two Pointers        │ Sorted array, find pairs, palindrome       │
│ 2. Sliding Window      │ Subarray/substring with condition          │
│ 3. Fast & Slow Pointer │ Cycle detection, middle of linked list     │
│ 4. Binary Search       │ Sorted data, find boundary/condition       │
│ 5. BFS                 │ Shortest path, level-order, graph layer    │
│ 6. DFS                 │ All paths, permutations, tree traversal    │
│ 7. Backtracking        │ Generate combinations, sudoku, N-queens    │
│ 8. Dynamic Programming │ Overlapping subproblems, optimization      │
│ 9. Greedy              │ Local optimal → global optimal             │
│10. Stack               │ Matching brackets, monotonic, next greater │
│11. Heap/Priority Queue │ Top K, median, merge K sorted              │
│12. HashMap/Set         │ Frequency count, lookup, deduplication     │
│13. Union-Find          │ Connected components, cycle in undirected  │
│14. Trie                │ Prefix search, autocomplete               │
│15. Topological Sort    │ Dependencies, course schedule, build order │
└────────────────────────┴─────────────────────────────────────────────┘

PATTERN RECOGNITION TRIGGERS:
  "Find pair that sums to X"           → Two Pointers (sorted) or HashMap
  "Longest substring with condition"   → Sliding Window
  "Minimum/maximum subarray"           → Sliding Window or Kadane's
  "All permutations/combinations"      → Backtracking
  "Shortest path unweighted"           → BFS
  "Number of ways to reach..."         → DP
  "Top K elements"                     → Heap
  "Connected components in graph"      → Union-Find or DFS
```

---

## 🎯 Framework

```
THE UMPIRE METHOD (for every problem):

  U — UNDERSTAND the problem (2-3 min)
      - Read carefully, identify inputs/outputs
      - Ask clarifying questions
      - "What if input is empty? Negative? Duplicates?"
      - Confirm with an example
      
  M — MATCH to a pattern (1-2 min)
      - "This looks like a sliding window problem because..."
      - "The sorted input suggests two pointers or binary search"
      
  P — PLAN your approach (3-5 min)
      - Describe algorithm in plain English
      - Walk through with example
      - State time/space complexity
      - Get interviewer buy-in: "Does this approach sound good?"
      
  I — IMPLEMENT (15-20 min)
      - Write clean, readable code
      - Use meaningful variable names
      - Handle edge cases
      
  R — REVIEW (3-5 min)
      - Trace through with example
      - Check edge cases (empty, single element, duplicates)
      - Verify time/space complexity
      
  E — EVALUATE trade-offs
      - "This is O(n log n). We could do O(n) with a hash map 
         but that uses O(n) extra space."
```

---

## ⏱️ Time Management

```
45-MINUTE CODING INTERVIEW:

  0-5 min:   Understand problem + ask questions
  5-10 min:  Plan approach + discuss with interviewer
  10-30 min: Implement solution
  30-40 min: Test + debug
  40-45 min: Optimize + discuss alternatives

COMMON TIME TRAPS:
  ❌ Spending 15 min on a "clever" approach that's too complex
  ❌ Not starting to code until minute 20
  ❌ Debugging silently for 10 minutes
  
  ✅ If stuck for 3 minutes, say it: "I'm thinking about X vs Y..."
  ✅ Start with brute force if needed: "Let me start with O(n²) 
     and then optimize"
  ✅ Code incrementally: get something working, then improve
```

---

## 🗣️ Communication

```
THINK OUT LOUD (most important skill):

  ❌ Silent coding for 20 minutes → interviewer has no idea if you're stuck
  
  ✅ "I notice the array is sorted, which makes me think of binary search..."
  ✅ "The brute force would check all pairs in O(n²). Can we do better?"
  ✅ "I'm going to use a hash map to track frequencies as I iterate..."
  ✅ "Let me handle the edge case where the list is empty..."

WHEN STUCK:
  1. State what you know: "I need to find X that satisfies Y"
  2. State what's blocking: "I'm not sure how to handle the case where..."
  3. Think of simpler version: "If the array were sorted, I could..."
  4. Ask for a hint (it's OK!): "Could you point me in the right direction?"
  
  Getting a hint and solving > struggling silently for 15 minutes
```

---

## ❌ Common Mistakes

```
1. NOT CLARIFYING CONSTRAINTS
   "Can the array contain negative numbers?"
   "Is the string ASCII or Unicode?"
   "Can there be duplicate elements?"
   → Changes your approach significantly!

2. JUMPING TO CODE WITHOUT A PLAN
   Plan first, code second. 5 minutes planning saves 15 minutes debugging.
   
3. IGNORING EDGE CASES
   Always consider: empty input, single element, all same, already sorted,
   negative numbers, integer overflow, null
   
4. OVER-ENGINEERING THE FIRST SOLUTION
   Start simple (even brute force), then optimize.
   A working O(n²) solution beats an incomplete O(n log n).
   
5. NOT TESTING YOUR CODE
   After writing, trace through with a small example.
   Check the off-by-one errors (boundaries of loops, array indices).
```

---

## 📅 Practice Plan

```
GRIND 75 (curated LeetCode list — best time-to-value):

WEEK 1: Arrays & Strings (warm up)
  - Two Sum, Best Time to Buy Stock, Valid Palindrome
  - Contains Duplicate, Maximum Subarray, Group Anagrams
  
WEEK 2: Linked Lists & Stacks
  - Reverse Linked List, Merge Two Lists, Detect Cycle
  - Valid Parentheses, Min Stack, Daily Temperatures
  
WEEK 3: Trees & Graphs
  - Max Depth of Tree, Invert Tree, Validate BST
  - Number of Islands, Clone Graph, Course Schedule
  
WEEK 4: Dynamic Programming & Advanced
  - Climbing Stairs, Coin Change, Longest Subsequence
  - Word Break, House Robber, Longest Palindromic Substring
  
DAILY PRACTICE:
  - 2-3 problems per day (1 easy, 1-2 medium)
  - Time yourself (25 min per problem)
  - Review solutions you couldn't solve (learn the pattern)
  - Revisit solved problems after 1 week (spaced repetition)
```

---

## 🔗 What to Read Next

1. **[InterviewPrep/Mock_Interviews.md](./Mock_Interviews.md)** — Practice with real simulation
2. **[DataStructures/README.md](../DataStructures/README.md)** — Data structures review
3. **[Algorithms/README.md](../Algorithms/README.md)** — Algorithm patterns

---

*[← System Design](./How_To_Crack_System_Design.md) | [Back to Index](../INDEX.md) | [Next: Mock Interviews →](./Mock_Interviews.md)*
