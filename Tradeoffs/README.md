# ⚖️ System Design Tradeoffs

> *"There are no solutions in system design, only tradeoffs. Every architectural decision is a bet — the skill is knowing which tradeoffs to make for YOUR specific problem."*

This folder covers the essential tradeoffs every system designer must understand. Each tutorial explores both sides of a decision, when to choose one over the other, and how Big Tech companies make these choices.

---

## 📋 Tutorials in This Folder

| # | Topic | Difficulty | Key Insight |
|---|-------|-----------|-------------|
| 1 | [Top 15 Tradeoffs](./Top_15_Tradeoffs.md) | 🟢 Easy | Overview of all major tradeoffs |
| 2 | [Vertical vs Horizontal Scaling](./Vertical_vs_Horizontal_Scaling.md) | 🟢 Easy | Scale up vs scale out |
| 3 | [Concurrency vs Parallelism](./Concurrency_vs_Parallelism.md) | 🟡 Medium | Multitasking vs multi-execution |
| 4 | [Long Polling vs WebSockets](./Long_Polling_vs_WebSockets.md) | 🟡 Medium | Real-time communication choices |
| 5 | [Batch vs Stream Processing](./Batch_vs_Stream_Processing.md) | 🟡 Medium | When to process data |
| 6 | [Stateful vs Stateless Design](./Stateful_vs_Stateless_Design.md) | 🟡 Medium | Where to keep state |
| 7 | [Strong vs Eventual Consistency](./Strong_vs_Eventual_Consistency.md) | 🟡 Medium | CAP theorem in practice |
| 8 | [Push vs Pull Architecture](./Push_vs_Pull_Architecture.md) | 🟡 Medium | Data delivery models |
| 9 | [REST vs RPC](./REST_vs_RPC.md) | 🟡 Medium | API communication styles |
| 10 | [Synchronous vs Asynchronous](./Synchronous_vs_Asynchronous.md) | 🟡 Medium | Communication patterns |
| 11 | [Latency vs Throughput](./Latency_vs_Throughput.md) | 🟢 Easy | Speed vs volume |

---

## 🎯 How to Use This Folder

1. Start with **Top 15 Tradeoffs** for the big picture
2. Deep dive into each tradeoff when you encounter it in system design problems
3. During interviews, explicitly state which tradeoff you're making and WHY

---

## 🔗 Related Folders
- [KeyConcepts](../KeyConcepts/) — Foundation concepts behind these tradeoffs
- [BuildingBlocks](../BuildingBlocks/) — Components where tradeoffs apply
- [SystemDesignCaseStudies](../SystemDesignCaseStudies/) — See tradeoffs in action
