# 🟣 Intercom | Problem 12 — Skill-Based Routing System

> **Source:** Extension of Intercom's recurring load balancer question
> **Difficulty:** 🟡 Medium
> **Topics:** HashMap · Priority Queue · Class Design · Greedy · Multi-criteria Matching

---

## 🔥 Ultimate Coding Interview Strategy

---

### **1️⃣ Understanding the Problem Clearly**

**Paraphrasing the Problem:**
> Design a **Skill-Based Routing** system that assigns incoming conversations to agents based on:
> 1. Agent must have the required skill for the conversation topic
> 2. Among eligible agents, assign to the one with the **fewest active conversations** (load balance)
> 3. If tied, assign to the agent who has been **waiting longest** (earliest last assignment time)
> 
> Implement:
> - `RoutingSystem()` — constructor
> - `addAgent(agentId, skills[])` — register agent with skill set
> - `assignConversation(conversationId, requiredSkill, timestamp)` — assign to best agent (return agentId or -1)
> - `closeConversation(conversationId, timestamp)` — agent finishes conversation

**Why Intercom Would Ask This:**
- This IS their product — Intercom's "Smart Assignment" feature
- Extension of Problem 01 (adds skill matching as a constraint)
- Tests multi-criteria decision making + class design
- Candidates report: "They just add more constraints to the same base problem"

**Example:**
```
system = RoutingSystem()
system.addAgent(1, ["billing", "technical"])
system.addAgent(2, ["billing"])
system.addAgent(3, ["technical", "sales"])

system.assignConversation("c1", "billing", 100)
→ Agent 1 or 2 (both have billing, both have 0 load → pick lower id or first added)
→ Let's say Agent 1 (first eligible with 0 load)

system.assignConversation("c2", "billing", 101)
→ Agent 2 (Agent 1 has 1 conv, Agent 2 has 0 → Agent 2 wins)

system.assignConversation("c3", "technical", 102)
→ Agent 3 (Agent 1 has 1, Agent 3 has 0 → Agent 3 wins)

system.closeConversation("c1", 200)
→ Agent 1 now has 0 active conversations

system.assignConversation("c4", "billing", 201)
→ Agent 1 (both have 0, Agent 1's lastAssignment is earlier → Agent 1 wins)
```

---

### **2️⃣ Breaking Down the Solution**

**Data Structures:**
```
agents:         Map<agentId, AgentInfo>
  AgentInfo: { skills: Set<String>, activeCount: int, lastAssignedTime: long }

skillIndex:     Map<skill, Set<agentId>>  — reverse index for quick skill lookup

conversations:  Map<conversationId, agentId>  — track who handles what
```

**Assignment Logic:**
1. Look up agents with required skill: `skillIndex.get(skill)`
2. Filter eligible agents
3. Sort/select by: (activeCount ASC, lastAssignedTime ASC)
4. Assign to winner, increment their activeCount, update lastAssignedTime

**Close Logic:**
1. Find agent from conversations map
2. Decrement their activeCount
3. Remove conversation from map

---

### **3️⃣ Writing Clean & Efficient Code**

#### ✅ Java Solution

```java
import java.util.*;

/**
 * Skill-Based Routing System.
 * 
 * Assigns conversations to the best-fit agent based on:
 * 1. Agent has required skill
 * 2. Lowest active conversation count
 * 3. Longest waiting (earliest last assignment)
 *
 * assignConversation: O(A) where A = agents with that skill
 * closeConversation:  O(1)
 * addAgent:           O(S) where S = number of skills
 */
public class RoutingSystem {

    private static class AgentInfo {
        int agentId;
        Set<String> skills;
        int activeCount;
        long lastAssignedTime;

        AgentInfo(int agentId, Set<String> skills) {
            this.agentId = agentId;
            this.skills = skills;
            this.activeCount = 0;
            this.lastAssignedTime = 0; // Never assigned
        }
    }

    private final Map<Integer, AgentInfo> agents;            // agentId → info
    private final Map<String, Set<Integer>> skillIndex;      // skill → set of agentIds
    private final Map<String, Integer> conversations;        // conversationId → agentId

    public RoutingSystem() {
        agents = new HashMap<>();
        skillIndex = new HashMap<>();
        conversations = new HashMap<>();
    }

    /**
     * Register an agent with their skill set.
     */
    public void addAgent(int agentId, List<String> skills) {
        AgentInfo agent = new AgentInfo(agentId, new HashSet<>(skills));
        agents.put(agentId, agent);

        // Build reverse index: skill → agents who have it
        for (String skill : skills) {
            skillIndex.computeIfAbsent(skill, k -> new HashSet<>()).add(agentId);
        }
    }

    /**
     * Assign conversation to the best available agent with the required skill.
     * 
     * Priority: 1) Has skill  2) Fewest active conversations  3) Longest idle
     * 
     * @return assigned agentId, or -1 if no eligible agent
     */
    public int assignConversation(String conversationId, String requiredSkill, long timestamp) {
        Set<Integer> eligibleIds = skillIndex.get(requiredSkill);
        if (eligibleIds == null || eligibleIds.isEmpty()) {
            return -1; // No agent has this skill
        }

        AgentInfo bestAgent = null;

        for (int agentId : eligibleIds) {
            AgentInfo candidate = agents.get(agentId);

            if (bestAgent == null) {
                bestAgent = candidate;
            } else if (candidate.activeCount < bestAgent.activeCount) {
                // Lower load wins
                bestAgent = candidate;
            } else if (candidate.activeCount == bestAgent.activeCount
                       && candidate.lastAssignedTime < bestAgent.lastAssignedTime) {
                // Same load → longer idle wins (earlier lastAssigned)
                bestAgent = candidate;
            }
        }

        if (bestAgent == null) return -1;

        // Assign
        bestAgent.activeCount++;
        bestAgent.lastAssignedTime = timestamp;
        conversations.put(conversationId, bestAgent.agentId);

        return bestAgent.agentId;
    }

    /**
     * Close a conversation, freeing up the agent's capacity.
     */
    public void closeConversation(String conversationId) {
        Integer agentId = conversations.remove(conversationId);
        if (agentId == null) return; // Unknown conversation

        AgentInfo agent = agents.get(agentId);
        if (agent != null && agent.activeCount > 0) {
            agent.activeCount--;
        }
    }

    /**
     * Get current load for an agent (useful for monitoring).
     */
    public int getAgentLoad(int agentId) {
        AgentInfo agent = agents.get(agentId);
        return agent != null ? agent.activeCount : 0;
    }
}
```

#### ✅ Python Solution

```python
from dataclasses import dataclass, field
from collections import defaultdict


@dataclass
class AgentInfo:
    agent_id: int
    skills: set[str]
    active_count: int = 0
    last_assigned_time: int = 0


class RoutingSystem:
    """
    Skill-Based Routing System.
    
    Assignment priority:
    1. Agent has required skill
    2. Lowest active conversation count
    3. Longest idle (earliest last_assigned_time)
    
    assign: O(A) — A = agents with that skill
    close:  O(1)
    """

    def __init__(self):
        self._agents: dict[int, AgentInfo] = {}
        self._skill_index: dict[str, set[int]] = defaultdict(set)  # skill → agent_ids
        self._conversations: dict[str, int] = {}  # conversation_id → agent_id

    def add_agent(self, agent_id: int, skills: list[str]) -> None:
        """Register an agent with their skill set."""
        self._agents[agent_id] = AgentInfo(
            agent_id=agent_id,
            skills=set(skills)
        )
        for skill in skills:
            self._skill_index[skill].add(agent_id)

    def assign_conversation(
        self, conversation_id: str, required_skill: str, timestamp: int
    ) -> int:
        """
        Assign to best agent with required skill.
        Returns agent_id or -1 if none available.
        """
        eligible_ids = self._skill_index.get(required_skill)
        if not eligible_ids:
            return -1

        # Select best: min by (active_count, last_assigned_time)
        best = min(
            (self._agents[aid] for aid in eligible_ids),
            key=lambda a: (a.active_count, a.last_assigned_time)
        )

        # Assign
        best.active_count += 1
        best.last_assigned_time = timestamp
        self._conversations[conversation_id] = best.agent_id
        return best.agent_id

    def close_conversation(self, conversation_id: str) -> None:
        """Free up agent capacity when conversation closes."""
        agent_id = self._conversations.pop(conversation_id, None)
        if agent_id is None:
            return
        agent = self._agents.get(agent_id)
        if agent and agent.active_count > 0:
            agent.active_count -= 1

    def get_agent_load(self, agent_id: int) -> int:
        """Get current active conversation count for agent."""
        agent = self._agents.get(agent_id)
        return agent.active_count if agent else 0
```

---

### **4️⃣ Explaining the Code**

**Architecture:**
```
                    ┌──────────────────────┐
                    │    RoutingSystem      │
                    ├──────────────────────┤
                    │ agents: {id→AgentInfo}│
                    │ skillIndex: {skill→ids}│
                    │ conversations: {c→id} │
                    └──────────┬───────────┘
                               │
            ┌──────────────────┼─────────────────────┐
            │                  │                     │
    assignConversation   closeConversation    addAgent
    1. skill lookup      1. find agent       1. store info
    2. filter eligible   2. decrement count  2. index skills
    3. pick min load     3. remove conv
    4. assign & record
```

**Selection criteria walkthrough:**
```
Agents: 
  Agent 1: skills=[billing, tech], active=2, lastAssigned=100
  Agent 2: skills=[billing],       active=1, lastAssigned=150
  Agent 3: skills=[billing, sales],active=1, lastAssigned=90

assign("c5", "billing", 200):
  Eligible: [1, 2, 3] (all have billing)
  Compare by (activeCount, lastAssignedTime):
    Agent 1: (2, 100) 
    Agent 2: (1, 150)
    Agent 3: (1, 90)   ← Winner! lowest count=1, and 90 < 150 (longer idle)
  
  → Assign to Agent 3
```

---

### **5️⃣ Edge Cases & Testing**

| Scenario | Expected | Notes |
|----------|----------|-------|
| No agent has required skill | return -1 | Graceful handling |
| All agents fully loaded | Assign to least loaded | No max capacity in basic version |
| Close unknown conversation | No-op | Defensive check |
| Single agent, multiple skills | Gets all matching convos | Correct skill indexing |
| Agent added after conversations started | Available for future assigns | Dynamic registration |
| Same agent, multiple closes | activeCount doesn't go below 0 | Guard against underflow |
| Tie on all criteria | Deterministic (pick any) | Python: min() picks first |

---

### **6️⃣ Follow-up Extensions (Interviewer might ask)**

| Extension | How to handle |
|-----------|---------------|
| **Max capacity per agent** | Add `maxCapacity` field, filter out full agents |
| **Agent goes offline** | Add `isOnline` flag, exclude from eligible set |
| **Priority conversations** | Sort conversations by priority, assign high-priority first |
| **Round-robin within same load** | Track `assignmentIndex` per skill group |
| **Skill proficiency levels** | Change skills from `Set<String>` to `Map<String, Integer>`, prefer higher proficiency |
| **Queue when all busy** | Add a waiting queue per skill, process on `closeConversation` |

**Extension: With Max Capacity + Queue**

```java
// Add to AgentInfo:
int maxCapacity;

// Add to RoutingSystem:
Map<String, Queue<String>> waitingQueues; // skill → queue of conversationIds

// In assignConversation:
// If all eligible agents at max capacity → add to waitingQueue, return -1

// In closeConversation:
// After decrementing, check if any queued conversation matches freed agent's skills
// If yes, auto-assign from queue
```

---

## 📎 Related Problems

| Problem | Connection |
|---------|-----------|
| [Intercom P01](./01_FairConversationAssignment.md) | Same core pattern without skill constraint |
| [LC 621 - Task Scheduler](https://leetcode.com/problems/task-scheduler/) | Resource scheduling with constraints |
| [LC 253 - Meeting Rooms II](https://leetcode.com/problems/meeting-rooms-ii/) | Capacity planning |
| [Intercom P10](./10_MeetingRoomsII.md) | Same "min resources for concurrent load" |

---

*Previous: [← 11 DesignLeaderboard](./11_DesignLeaderboard.md) | Next: [13 MinicomRoundPrep →](./13_MinicomRoundPrep.md)*
