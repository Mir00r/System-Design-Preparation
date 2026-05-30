# 🤖 Agent Pattern — Autonomous AI That Plans & Acts 🎯

> **"An Agent is an LLM with a plan, tools, and the autonomy to execute."** — The simplest definition

```
┌─────────────────────────────────────────────────────────────────┐
│  🎮 LEVEL 5.3: AI AGENT PATTERN                                 │
│                                                                 │
│  XP Reward: +300 🌟                                             │
│  Badge: 🤖 Agent Architect                                      │
│  Time: ~55 minutes                                              │
│  Industry Relevance: ⭐⭐⭐⭐⭐ (THE hottest AI pattern in 2025!)  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 📑 Table of Contents

1. [What is an AI Agent?](#-what-is-an-ai-agent)
2. [Agent vs Chatbot vs Pipeline](#-agent-vs-chatbot-vs-pipeline)
3. [The Agent Loop (Plan → Act → Observe → Repeat)](#-the-agent-loop)
4. [Tool Use — Giving Agents Superpowers](#-tool-use--giving-agents-superpowers)
5. [Planning Strategies](#-planning-strategies)
6. [Memory Systems for Agents](#-memory-systems)
7. [Agent Frameworks & Architectures](#-agent-frameworks)
8. [Java Implementation](#-java-implementation)
9. [Real-World Applications](#-real-world-applications)
10. [Interview Questions & Puzzles](#-interview-questions--puzzles)

---

## 🤔 What is an AI Agent?

### The Core Idea

```
CHATBOT: You ask → It answers. Done. (One turn)

AGENT: You give a GOAL → It PLANS → Uses TOOLS → OBSERVES results 
       → Adjusts plan → Continues → Eventually achieves the goal!
       (Multi-step, autonomous)
```

### 🎮 The Chef Analogy 👨‍🍳

```
Chatbot = A cookbook
  "What's the recipe for chocolate cake?" → Gives recipe. Done.

Agent = An actual chef
  "Make me a chocolate cake!" → Chef:
  1. Plans: "I need flour, eggs, chocolate, oven time..."
  2. Acts: Opens fridge → checks ingredients
  3. Observes: "No eggs! Need to substitute or go buy."
  4. Replans: "I'll use applesauce instead"
  5. Acts: Mixes ingredients, preheats oven
  6. Observes: "Batter looks right"
  7. Acts: Bakes for 30 min
  8. Final: "Here's your cake!" 🎂

The chef AUTONOMOUSLY handles unexpected situations! 🎯
```

### The Agent Components

```
┌─────────────────────────────────────────────────────────────┐
│                        AI AGENT                              │
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐ │
│  │    BRAIN    │  │    TOOLS    │  │      MEMORY         │ │
│  │   (LLM)    │  │  (Actions)  │  │  (Context/History)  │ │
│  ├─────────────┤  ├─────────────┤  ├─────────────────────┤ │
│  │ • Reasoning │  │ • Search    │  │ • Short-term (chat) │ │
│  │ • Planning  │  │ • Code exec │  │ • Long-term (facts) │ │
│  │ • Deciding  │  │ • API calls │  │ • Episodic (past)   │ │
│  │ • Reflecting│  │ • Database  │  │ • Working (scratch) │ │
│  └─────────────┘  │ • File I/O  │  └─────────────────────┘ │
│                    │ • Browser   │                           │
│                    └─────────────┘                           │
│                                                             │
│  + PLANNING STRATEGY (how to approach the task)             │
│  + STOPPING CRITERIA (when to say "done!")                  │
└─────────────────────────────────────────────────────────────┘
```

---

## ⚔️ Agent vs Chatbot vs Pipeline

```
┌─────────────────────────────────────────────────────────────────────┐
│  CHATBOT (Simple)                                                    │
│  User → LLM → Response                                              │
│  • One turn, no tools, no memory                                    │
│  • Example: FAQ bot                                                  │
├─────────────────────────────────────────────────────────────────────┤
│  PIPELINE (Fixed flow)                                               │
│  User → Step 1 → Step 2 → Step 3 → Response                        │
│  • Predetermined steps, no deviation                                │
│  • Example: RAG (search → retrieve → generate)                      │
├─────────────────────────────────────────────────────────────────────┤
│  AGENT (Autonomous)                                                  │
│  User → Plan → [Act → Observe → Replan]* → Response                 │
│  • Dynamic steps, self-correcting, tool-using                       │
│  • Example: "Research and write a report on X"                      │
└─────────────────────────────────────────────────────────────────────┘
```

| Aspect | Chatbot | Pipeline | Agent |
|--------|---------|----------|-------|
| Steps | 1 | Fixed N | Dynamic |
| Tool Use | ❌ | Predetermined | Dynamic choice |
| Error Handling | None | Predefined | Self-correcting |
| Autonomy | None | Low | High |
| Cost | $ | $$ | $$$ (many LLM calls!) |
| Risk | Low | Low | Medium-High ⚠️ |

---

## 🔄 The Agent Loop

### The ReAct Pattern (Reasoning + Acting)

```
TASK: "What's the current stock price of Apple, and is it higher 
      than its 52-week average?"

THOUGHT: I need to find Apple's current stock price and its 
         52-week average. Let me search for this information.

ACTION: search("Apple AAPL current stock price")

OBSERVATION: Apple (AAPL) is currently trading at $198.50.

THOUGHT: Good, I have the current price. Now I need the 52-week 
         average. Let me search for that.

ACTION: search("AAPL 52-week average price")

OBSERVATION: AAPL 52-week range: $165.67 - $199.62. 
             Average: ~$182.64.

THOUGHT: Current price ($198.50) is above the 52-week average 
         ($182.64). I have all the information to answer.

ACTION: respond("Apple (AAPL) is currently at $198.50, which is 
        above its 52-week average of approximately $182.64, 
        trading about 8.7% above average.")

✅ DONE!
```

### The Loop Diagram

```
                    ┌──────────────┐
                    │    START     │
                    │ (User Goal)  │
                    └──────┬───────┘
                           │
                    ┌──────▼───────┐
             ┌─────│   THINK      │ ← LLM reasons about what to do
             │     │  (Plan/Reason)│
             │     └──────┬───────┘
             │            │
             │     ┌──────▼───────┐
             │     │    ACT       │ ← Execute a tool/action
             │     │ (Use Tool)   │
             │     └──────┬───────┘
             │            │
             │     ┌──────▼───────┐
             │     │   OBSERVE    │ ← Get result from tool
             │     │ (Get Result) │
             │     └──────┬───────┘
             │            │
             │     ┌──────▼───────┐
             └─────│  DONE?       │──── YES ──→ 🎯 FINAL ANSWER!
                   │  (Evaluate)  │
                   └──────────────┘
                   
Repeat until: Goal achieved OR max iterations OR error! ⚠️
```

---

## 🔧 Tool Use — Giving Agents Superpowers

### What Are "Tools"?

```
A tool = A function the agent can call to interact with the world

Without tools: LLM can only generate text (limited!)
With tools: LLM can SEARCH, CALCULATE, CODE, API-CALL, READ FILES...

The LLM doesn't execute tools itself!
It generates a STRUCTURED request → The system executes it → Returns result!
```

### Common Agent Tools

| Tool | What It Does | Example Use |
|------|-------------|-------------|
| 🔍 Web Search | Search the internet | "Find latest news about..." |
| 💻 Code Interpreter | Run Python/JS code | "Calculate compound interest" |
| 📊 Database Query | SQL queries | "How many orders last month?" |
| 📧 Email | Send/read email | "Send meeting invite to team" |
| 📁 File System | Read/write files | "Create a report document" |
| 🌐 API Call | Hit external APIs | "Get weather for London" |
| 🖥️ Browser | Navigate web pages | "Fill out this form" |
| 📝 Note Taking | Store information | "Remember this for later" |

### Function Calling (How It Actually Works)

```json
// The agent's tool definitions (schema)
{
  "tools": [
    {
      "name": "search_web",
      "description": "Search the internet for information",
      "parameters": {
        "query": {"type": "string", "description": "Search query"}
      }
    },
    {
      "name": "calculate",
      "description": "Perform mathematical calculations",
      "parameters": {
        "expression": {"type": "string", "description": "Math expression"}
      }
    }
  ]
}

// LLM decides to use a tool:
{
  "tool_call": {
    "name": "search_web",
    "arguments": {"query": "AAPL stock price today"}
  }
}

// System executes and returns:
{
  "tool_result": "AAPL is trading at $198.50 as of market close."
}

// LLM incorporates result into its reasoning...
```

---

## 🧠 Planning Strategies

### 1️⃣ ReAct (Reason + Act) — Most Common!

```
Interleaves thinking and acting:
Think → Act → Observe → Think → Act → Observe → ...

Pros: Simple, works well for most tasks
Cons: Can get stuck in loops, doesn't plan far ahead
```

### 2️⃣ Plan-and-Execute — Better for Complex Tasks

```
Step 1: Create a FULL plan first
  "To answer this, I need to: 1) search X, 2) calculate Y, 3) compare"

Step 2: Execute each step
  Execute step 1 → result
  Execute step 2 → result
  Execute step 3 → result

Step 3: Synthesize final answer

Pros: Better for multi-step tasks, can parallelize!
Cons: Plan may need adjustment (rigid)
```

### 3️⃣ Tree of Thoughts — For Difficult Reasoning

```
             ┌─── Approach A ─── Good? ✅
             │
Problem ─────┼─── Approach B ─── Dead end ❌
             │
             └─── Approach C ─── Best! ⭐
             
Explore multiple solution paths, evaluate each,
pick the best. Like chess: think multiple moves ahead!

Pros: Better for complex reasoning
Cons: Expensive (many LLM calls per branch)
```

---

## 🧠 Memory Systems

### Why Agents Need Memory

```
Without Memory:
Turn 1: "My name is Alice" 
Turn 5: "What's my name?" → "I don't know" 😱

With Memory:
Turn 1: "My name is Alice" → [Stored: user_name = Alice]
Turn 5: "What's my name?" → [Retrieved: Alice] → "Your name is Alice!" ✅
```

### Types of Agent Memory

```
1️⃣ SHORT-TERM (Conversation Buffer)
   What: Last N messages in current conversation
   How: Just keep them in context window
   Limit: Context window size!

2️⃣ LONG-TERM (Persistent Knowledge)
   What: Facts learned across conversations
   How: Store in vector DB, retrieve when relevant
   Example: User preferences, past decisions

3️⃣ EPISODIC (Past Experiences)
   What: Records of past task completions
   How: "Last time you asked about X, I found Y useful"
   Use: Learn from past successes/failures

4️⃣ WORKING MEMORY (Scratchpad)
   What: Notes during current task execution
   How: Agent writes notes to itself
   Use: Multi-step tasks, intermediate results
```

---

## ☕ Java Implementation

```java
/**
 * AI Agent implementation in Java
 * Demonstrates the ReAct (Reason + Act) loop pattern
 */
public class AIAgent {
    
    private final ChatModel llm;
    private final Map<String, Tool> tools;
    private final List<Message> conversationHistory;
    private final int maxIterations;
    
    public AIAgent(ChatModel llm, List<Tool> toolList, int maxIterations) {
        this.llm = llm;
        this.tools = toolList.stream()
            .collect(Collectors.toMap(Tool::getName, t -> t));
        this.conversationHistory = new ArrayList<>();
        this.maxIterations = maxIterations;
    }
    
    /**
     * Main agent loop: Think → Act → Observe → Repeat
     */
    public String execute(String userGoal) {
        conversationHistory.add(new UserMessage(userGoal));
        
        for (int i = 0; i < maxIterations; i++) {
            // THINK: Ask LLM what to do next
            AgentDecision decision = think();
            
            if (decision.isFinished()) {
                // Agent decided it's done!
                return decision.getFinalAnswer();
            }
            
            // ACT: Execute the chosen tool
            ToolCall toolCall = decision.getToolCall();
            String observation = act(toolCall);
            
            // OBSERVE: Add result to conversation
            conversationHistory.add(new ToolMessage(
                toolCall.getName(), observation
            ));
        }
        
        return "I couldn't complete the task within " + maxIterations + " steps.";
    }
    
    private AgentDecision think() {
        String systemPrompt = """
            You are an AI agent that can use tools to accomplish tasks.
            
            Available tools:
            %s
            
            For each step, either:
            1. Call a tool: {"tool": "tool_name", "args": {...}}
            2. Give final answer: {"answer": "your final response"}
            
            Think step by step. Explain your reasoning before acting.
            """.formatted(getToolDescriptions());
        
        String response = llm.call(systemPrompt, conversationHistory);
        return parseDecision(response);
    }
    
    private String act(ToolCall toolCall) {
        Tool tool = tools.get(toolCall.getName());
        if (tool == null) {
            return "Error: Tool '" + toolCall.getName() + "' not found!";
        }
        
        try {
            return tool.execute(toolCall.getArguments());
        } catch (Exception e) {
            return "Error executing tool: " + e.getMessage();
        }
    }
    
    // ─── TOOL INTERFACE ────────────────────────────────────────
    
    public interface Tool {
        String getName();
        String getDescription();
        Map<String, String> getParameterSchema();
        String execute(Map<String, Object> args) throws Exception;
    }
    
    // ─── EXAMPLE TOOL: Web Search ──────────────────────────────
    
    public static class WebSearchTool implements Tool {
        @Override
        public String getName() { return "web_search"; }
        
        @Override
        public String getDescription() { 
            return "Search the internet for information"; 
        }
        
        @Override
        public String execute(Map<String, Object> args) {
            String query = (String) args.get("query");
            // Call actual search API here
            return searchAPI.search(query);
        }
    }
}
```

---

## 🏢 Real-World Applications

### 🔷 GitHub Copilot Agent Mode

```
Task: "Fix the failing tests in the auth module"

Agent Flow:
1. THINK: "I need to find failing tests first"
2. ACT: Run test suite → get failures
3. OBSERVE: "test_login_with_expired_token fails"
4. THINK: "Let me look at that test and the related code"
5. ACT: Read test file + auth module
6. OBSERVE: Found the bug (token expiry check is wrong)
7. THINK: "I know how to fix this"
8. ACT: Edit the code, add proper expiry check
9. ACT: Run tests again
10. OBSERVE: All tests pass! ✅
11. RESPOND: "Fixed! The token expiry was checking > instead of >="
```

### 🔷 Devin (Software Engineering Agent)

```
Task: "Implement dark mode for the settings page"

Agent Flow:
1. Plans: Analyze existing UI, find CSS variables, plan changes
2. Reads: Examines current codebase structure
3. Codes: Creates theme toggle, adds CSS variables
4. Tests: Opens browser, verifies visual changes
5. Debugs: Finds contrast issue, fixes it
6. Commits: Creates PR with description

All AUTONOMOUSLY! (But with human approval at the end!)
```

### 🔷 Customer Service Agents (Many Companies)

```
Customer: "I was charged twice for my order #12345"

Agent:
1. THINK: "I need to verify the order and check payment history"
2. ACT: query_database("SELECT * FROM orders WHERE id = 12345")
3. OBSERVE: Order exists, one charge of $49.99
4. ACT: query_payments("SELECT * FROM payments WHERE order_id = 12345")
5. OBSERVE: TWO charges found! $49.99 each!
6. THINK: "Confirmed double charge. I should refund the duplicate."
7. ACT: initiate_refund(payment_id=67890, amount=49.99)
8. OBSERVE: Refund initiated successfully
9. RESPOND: "I've confirmed the double charge and initiated a refund 
   of $49.99 to your original payment method. It should appear in 
   3-5 business days."

Human agent would have taken 10 minutes. AI agent: 5 seconds! ⚡
```

---

## 🎯 Interview Questions & Puzzles

### Question 1: "What's the difference between a chatbot and an AI agent?"

**Great Answer**: "A chatbot is a single LLM call — query in, response out, no tools, no memory between turns. An AI agent is an LLM in a loop that can: (1) reason about complex goals, (2) use tools (search, code, APIs) to interact with the world, (3) observe results and adapt its plan, (4) maintain memory across steps. The key differences are autonomy (agent decides what to do), tool use (agent can act), and iterative execution (agent loops until done). Agents are more powerful but also more expensive, slower, and harder to control — so I'd use a simple pipeline unless the task truly requires dynamic decision-making."

### Question 2: "How do you prevent an AI agent from going off the rails?"

**Great Answer**: "Several safety mechanisms: (1) Max iterations — hard stop after N steps to prevent infinite loops, (2) Allowed tool list — restrict which actions the agent can take, (3) Human-in-the-loop — require approval for destructive actions (delete, send email, make purchases), (4) Guardrails — validate tool inputs/outputs against schemas, (5) Budget limits — cap API calls and compute costs, (6) Sandboxing — run code execution in isolated environments, (7) Monitoring — log every thought and action for review. In production, I'd default to a conservative agent that asks for confirmation on anything irreversible."

---

### 🧩 Puzzle: Agent vs Pipeline Decision

```
Which approach for each scenario? 🤔

A) "Summarize this PDF" 
   → PIPELINE ✅ (fixed: parse → chunk → summarize)

B) "Research competitors and write a report"
   → AGENT ✅ (needs dynamic search, planning, iteration)

C) "Answer questions about our docs"
   → PIPELINE (RAG) ✅ (search → retrieve → answer)

D) "Debug this failing deployment"
   → AGENT ✅ (needs to check logs, try fixes, verify)

E) "Translate this paragraph"
   → CHATBOT ✅ (single LLM call, no tools needed!)

F) "Book me the cheapest flight to Tokyo next week"
   → AGENT ✅ (search, compare, check dates, maybe book!)
```

---

## 🎯 Key Takeaways

| Concept | One-Liner | Remember |
|---------|-----------|----------|
| Agent | LLM + tools + planning + autonomy | "AI that ACTS, not just talks" |
| ReAct | Think → Act → Observe loop | Most common agent pattern |
| Tools | Functions the agent can call | "Superpowers" for the LLM |
| Memory | Short/long-term information storage | Enables multi-step tasks |
| Planning | How agent approaches complex tasks | Plan-first vs improvise |
| Guardrails | Safety boundaries | CRITICAL for production! |

---

## ➡️ Next Up

👉 [Module 5.4: Chain of Thought Pattern →](./04_Chain_of_Thought.md)

---

*"An AI agent is like giving an intern access to Google, a calculator, and your company database, and saying 'Figure it out.' Sometimes brilliant. Sometimes... chaotic."* 🤖😅
