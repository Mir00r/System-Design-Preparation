# 🟣 Intercom | Interview Day — Quick Reference & Cheat Sheet

> **Print this on interview day. One-page summaries for each round.**

---

## ⏰ Interview Timeline

```
Round 2 (Same Day):
├── Problem Solving / Coding (50 min)
│   └── THE load balancer problem (95% probability)
└── Values Assessment (45 min)
    └── 3 deep stories × 15 min each

Round 3 (Usually 1 week later):
├── Minicom — Full Stack (90 min: 45 solo + 45 pairing)
│   └── Build feature in existing messaging app
├── Technical Design / Data Modelling (45 min)
│   └── DB schema for messaging system
└── Role Chat (30 min)
    └── Casual — ask good questions
```

---

## 🖥️ Coding Round Cheat Sheet

### Before You Code (5 min)

```
1. REPEAT the problem: "So I need to build a class that..."
2. CLARIFY: "Are agent IDs unique?" "Is timestamp monotonic?"
3. STATE approach: "I'll use a HashMap for O(1) lookup + a counter for load"
4. DISCUSS complexity: "This gives us O(1) assign and O(n) for finding min-load"
5. THEN code
```

### The Load Balancer Template (Memorize This Structure)

```java
public class AssignmentSystem {
    private Map<String, AgentInfo> agents;      // agent state
    private Map<String, String> assignments;    // conv → agent mapping

    public AssignmentSystem() { ... }
    
    public void addAgent(String agentId) { ... }
    
    public String assign(String conversationId) {
        // Find agent with min load (tie-break: earliest lastAssigned)
        // Record assignment
        // Return agentId
    }
    
    public void close(String conversationId) {
        // Find agent, decrement their count
    }
}
```

### Key Patterns to Remember

| Pattern | When | Implementation |
|---------|------|----------------|
| Track per-entity state | Agent load, user pings | `HashMap<id, Object>` |
| Find minimum in collection | Assign to least-loaded | Iterate map or use PQ |
| Sliding window time | Rate limit, hit counter | Queue + evict expired |
| Dual data structure | O(1) get + O(1) move | HashMap + LinkedList |

### Complexity You MUST State

```
"My assign() is O(n) where n = number of agents.
 I could improve to O(log n) using a TreeMap or Priority Queue,
 trading space and implementation complexity for speed.
 For Intercom's use case (~100 agents), O(n) is perfectly fine."
```

---

## 💬 Values Round Cheat Sheet

### The 5 Values (Remember These)

1. **Shape the Solution** — Don't blindly execute; understand WHY
2. **Be Technically Conservative** — Boring tech, reuse patterns
3. **Build in Small Steps** — Small PRs, feature flags, incremental
4. **Keep it Simple** — Trade perf for clarity
5. **Work with Positivity** — Own mistakes, help others, no blame

### Answer Format (SBI — 4 Minutes Max)

```
SITUATION (30s): "At [Company], we had [problem]. Team of [N]."
BEHAVIOUR (2.5m): "I specifically did [A], then [B], because [reason]."
IMPACT (1m): "Result: [metric] improved by [X%]. I'd do [Y] differently."
```

### Your 3 Ready Stories

| # | Story | Maps to Value | Key Metric |
|---|-------|---------------|------------|
| 1 | _________________ | __________ | ___% / ___x improvement |
| 2 | _________________ | __________ | ___% / ___x improvement |
| 3 | _________________ | __________ | ___% / ___x improvement |

### Red Flags Checklist

- [ ] Never blame others
- [ ] Always quantify impact
- [ ] Clarify "I" vs "we"
- [ ] Include "what I'd do differently"
- [ ] No vague "it helped" — use numbers

---

## 🔨 Minicom Round Cheat Sheet

### First 10 Minutes Routine

```
1. Read README.md
2. npm install && npm start (or docker-compose up)
3. Open browser → click through app
4. Find: components/ folder, routes/ folder, db schema
5. Identify what already exists
6. PLAN which files to modify
```

### Files You'll Likely Touch

```
Frontend: src/components/[NewFeature].tsx
Backend:  src/routes/[resource].ts  OR  app/controllers/[resource]_controller.rb
Database: Add migration if needed (or just raw SQL)
```

### Quick Patterns

**Add a button that calls API:**
```tsx
const handleClick = async () => {
  await fetch('/api/resource', { method: 'POST', headers: {'Content-Type': 'application/json'}, body: JSON.stringify(data) });
  // Refresh state
};
```

**Add an endpoint:**
```typescript
router.post('/resource', async (req, res) => {
  const { field1, field2 } = req.body;
  const result = await pool.query('INSERT INTO table (col1, col2) VALUES ($1, $2) RETURNING *', [field1, field2]);
  res.status(201).json(result.rows[0]);
});
```

### Pairing Communication Templates

```
"I'm thinking about it this way: [explain mental model]"
"Should I start with backend or frontend?"
"I'm going to [action] — does that make sense to you?"
"I'm stuck on [X] — any hints?"
"What would you like me to focus on next?"
```

---

## 📊 Data Modelling Cheat Sheet

### Schema Drawing Order

```
1. ENTITIES: users, conversations, participants, messages
2. RELATIONSHIPS: 
   - users ←→ conversations (M:N via participants)
   - messages → conversations (M:1)
   - messages → users/sender (M:1)
3. KEY COLUMNS: id, FKs, type, status, timestamps
4. INDEXES: conversation_id+created_at on messages
5. EXTENSIONS: reactions, threads, attachments
```

### The Core Tables (Draw from Memory)

```
users:          id, email, display_name, role, created_at
conversations:  id, type, title, status, created_at, updated_at
participants:   id, conversation_id, user_id, role, last_read_at  [UNIQUE(conv,user)]
messages:       id, conversation_id, sender_id, content, reply_to_id, created_at
```

### Extension Answers (Memorize)

| "How would you add..." | Answer |
|------------------------|--------|
| Emoji reactions | `reactions` table: message_id, user_id, emoji + UNIQUE constraint |
| Threading | `reply_to_id` FK on messages (simple) OR separate `threads` table |
| Read receipts | `last_read_at` on participants; unread = COUNT messages after that |
| Typing indicators | Redis TTL keys (NOT database — too ephemeral) |
| File attachments | `attachments` table: message_id, file_url, mime_type |
| Search | Elasticsearch async index (not PostgreSQL LIKE) |
| Online status | Redis SET with TTL (heartbeat refreshes) |

### Scaling Answers (Memorize)

| "How would you scale..." | Answer |
|--------------------------|--------|
| Read-heavy messages | Read replicas |
| Large messages table | Partition by created_at (monthly) |
| Unread counts | Denormalize counter on participants (async update) |
| Real-time delivery | Redis pub/sub → WebSocket servers |
| Message search | Elasticsearch (async index via event queue) |
| Hot conversations | Cache recent N messages in Redis |

---

## 🎯 Role Chat — Questions to Ask

### Questions That Show Product Thinking

1. "How does Fin (AI agent) decide when to hand off to a human agent?"
2. "What's the biggest technical challenge the team is facing right now?"
3. "How do you balance product velocity with technical debt?"
4. "How do engineers participate in product discovery and user research?"
5. "What does a typical sprint/development cycle look like for this team?"

### Questions That Show Culture Fit

6. "What does 'shipping in small steps' look like day-to-day?"
7. "How does the team handle incidents and post-mortems?"
8. "What's the onboarding experience like for new engineers?"
9. "What's one thing you'd change about how the team works?"
10. "How do product engineers collaborate with designers?"

### Avoid Asking

- Salary/benefits (that's for recruiter)
- "What does the company do?" (you should know)
- Anything that sounds like you haven't researched them

---

## 📅 Day-Before Checklist

```
□ Re-read scheduling email (coding question might be revealed!)
□ Practice Problem 01 from scratch one more time (< 25 min)
□ Review your 3 behavioral stories (say them aloud)
□ Draw the messaging schema from memory
□ Test internet, camera, microphone
□ Prepare water, pen, paper for scratch work
□ Look up your interviewers on LinkedIn (know their background)
□ Review Intercom's latest product updates (AI, Fin, new features)
□ Get 8 hours sleep
```

## 🧠 During-Interview Mantras

```
• "Let me make sure I understand the problem correctly..."
• "I'm going to think through the approach before coding..."
• "The time complexity here is O(n) because..."
• "If I had more time, I would optimize by..."
• "To be transparent, I'm not 100% sure about X, but my instinct is..."
• "Let me walk you through my thinking..."
```

---

## 🏆 Final Tips (From Candidates Who Got Offers)

1. **The coding question is the SAME one every time** — know P01 cold
2. **Code is NOT compiled** — readability > correctness of syntax
3. **Spend 5-10 min discussing approach BEFORE coding**
4. **For Minicom: SHIP something working, don't polish**
5. **For values: have SPECIFIC numbers ready (%, hours, $)**
6. **For data modelling: start entities → relationships → indexes**
7. **For role chat: ask questions that show product thinking**
8. **Be yourself — they're checking cultural fit, not acting skills**
