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

---

## 🧠 Common Themes at Intercom

- **Agent/Operator Assignment** — Load balancing with capacity limits and tie-breaking rules
- **Messaging System Design** — DB schema for chats, groups, private messages
- **Time-Series Analysis** — Histogram of transactions/pings over time windows
- **Class-based API Design** — Two or more methods on a class using HashMaps and arrays

---

## 💡 Interview Tips (Intercom Specific)

- Intercom expects strong **OOP fundamentals** — think in terms of classes with clear responsibilities
- Be ready to discuss **Low Level Design** for chat systems (tables, relationships, indexes)
- The load balancer problem is the **most recurring** — know it inside out
- They value **clean, readable code** over clever one-liners
- Always discuss the **tie-breaking logic** explicitly — interviewers probe this

---

## 📚 Related Topics to Review

- [Priority Queue / Heap](../../DataStructures/Heaps/)
- [HashMap / HashTable](../../DataStructures/HashTables/)
- [Load Balancing](../../BuildingBlocks/LoadBalancing.md)
- [System Design — Chat System](../../SystemDesignCaseStudies/)
