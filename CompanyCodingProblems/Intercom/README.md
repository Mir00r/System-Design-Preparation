# 🟣 Intercom — Coding Interview Problems

Intercom builds a customer service platform. Their coding rounds heavily focus on:
- **Load balancing / fair assignment systems** (OOP, priority queues, simulation)
- **Data modeling** for chat/messaging systems (DB schema, relational design)
- **HashMap + time-windowed aggregation** (histogram problems)
- **API design** with class-based interfaces

---

## 📁 Problems

| # | Problem | Difficulty | Topics | Source |
|---|---------|------------|--------|--------|
| 01 | [Fair Conversation Assignment API](./01_FairConversationAssignment.md) | 🟠 Medium-Hard | OOP, Heap, Simulation, Load Balancing | Intercom Coding Round (recurring) |
| 02 | [Tweet Counts Per Frequency](./02_TweetCountsPerFrequency.md) | 🟡 Medium | HashMap, Time Bucketing, Math | LeetCode 1348 / Intercom Coding Round |
| 03 | [Longest Substring Without Repeating Characters](./03_LongestSubstringNoRepeating.md) | 🟡 Medium | Sliding Window, HashMap, Two Pointers | LeetCode 3 / Intercom Coding Round |
| 04 | [User Ping Tracker API](./04_UserPingTracker.md) | 🟡 Medium | HashMap, Partitioned Array, Class Design, Math | Intercom Coding Round (recurring) |
| 05 | [Transaction Histogram](./05_TransactionHistogram.md) | 🟡 Medium | HashMap, Time Bucketing, Data Aggregation | Intercom Coding Round (recurring) |
| 06 | [Chat System Low Level Design (DB Schema)](./06_ChatSystemLowLevelDesign.md) | 🔴 Hard | RDBMS, Schema Design, Indexing, Normalization | Intercom System Design Round (recurring) |
| 07 | [Task Scheduler](./07_TaskScheduler.md) | 🟡 Medium | Priority Queue, Greedy, HashMap, Cooldown | LeetCode 621 / Expected (PQ-based) |
| 08 | [Sliding Window Rate Limiter](./08_RateLimiter.md) | 🟡 Medium | Sliding Window, HashMap, Queue, Class Design | Expected (API Rate Limiting) |
| 09 | [LRU Cache](./09_LRUCache.md) | 🟡 Medium | HashMap, Doubly Linked List, Class Design | LeetCode 146 / Expected (Dual DS) |
| 10 | [Meeting Rooms II](./10_MeetingRoomsII.md) | 🟡 Medium | Sorting, Priority Queue, Intervals, Greedy | LeetCode 253 / Expected (Scheduling) |
| 11 | [Design Leaderboard](./11_DesignLeaderboard.md) | 🟡 Medium | HashMap, TreeMap, Sorting, Top-K | LeetCode 1244 / Expected (Ranking) |
| 12 | [Skill-Based Routing](./12_SkillBasedRouting.md) | 🟡 Medium | HashMap, Priority Queue, Multi-criteria | Expected (Extension of P01) |

---

## 🧠 Common Themes at Intercom

- **Agent/Operator Assignment** — Load balancing with capacity limits and tie-breaking rules
- **Messaging System Design** — DB schema for chats, groups, private messages
- **Time-Series Analysis** — Histogram of transactions/pings over time windows
- **Class-based API Design** — Two or more methods on a class using HashMaps and arrays

---

## � Interview Preparation Guides

| # | Guide | Covers |
|---|-------|--------|
| 00 | [Complete Interview Prep Guide](./00_InterviewPrepGuide.md) | Master overview: all rounds, patterns, 90-day plan |
| 13 | [Minicom Round Prep](./13_MinicomRoundPrep.md) | Full-stack coding: React + Node + Rails patterns, time management |
| 14 | [Values/Behavioral Deep Dive](./14_ValuesRound_BehavioralPrep.md) | 15 pre-written stories, SBI framework, red flags |
| 15 | [System Design & Data Modelling](./15_SystemDesign_DataModelling.md) | Schema design, extensions, scaling, architecture |
| 16 | [Additional Coding Problems](./16_AdditionalCodingProblems.md) | 8 backup problems (Group Anagrams, Min Stack, Top K, etc.) |
| 17 | [Interview Day Cheat Sheet](./17_InterviewDay_CheatSheet.md) | One-page quick reference for each round |
| 18 | [Company Knowledge 2026](./18_CompanyKnowledge_2026.md) | Tech stack, scale numbers, culture, what to reference |

---

## 💡 Interview Tips (Intercom Specific)

- Intercom expects strong **OOP fundamentals** — think in terms of classes with clear responsibilities
- Be ready to discuss **Low Level Design** for chat systems (tables, relationships, indexes)
- The load balancer problem is the **most recurring** — know it inside out (P01)
- They value **clean, readable code** over clever one-liners
- Always discuss the **tie-breaking logic** explicitly — interviewers probe this
- **Code is NOT compiled/run** — readability > syntax correctness
- **Spend 5-10 min discussing approach BEFORE coding**
- **The coding question may be revealed in the prep email** — read it carefully!
- Their tech stack: **Rails monolith, MySQL/Vitess, Elasticsearch, Redis, AWS**
- Philosophy: "Boring technology", "Ship in small steps", "Run less software"

---

## 📚 Related Topics to Review

- [Priority Queue / Heap](../../DataStructures/Heaps/)
- [HashMap / HashTable](../../DataStructures/HashTables/)
- [Load Balancing](../../BuildingBlocks/LoadBalancing.md)
- [System Design — Chat System](../../SystemDesignCaseStudies/)
- [Caching Strategies](../../BuildingBlocks/CachingStrategies.md)
- [Message Queues](../../BuildingBlocks/MessageQueues.md)
- [Database Sharding](../../Database/Sharding.md)
