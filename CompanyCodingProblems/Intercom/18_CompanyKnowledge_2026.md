# 🟣 Intercom | Company & Engineering Deep Knowledge (2026)

> **Purpose:** Know the company inside-out. Use this in values round, role chat, and system design discussions.
> **Sources:** Intercom Engineering Blog (May 2026, Jan 2026, 2025), product pages, press
> **Last Updated:** May 2026

---

## 📋 Table of Contents

1. [Company Overview & Product](#company-overview--product)
2. [Tech Stack (Confirmed 2026)](#tech-stack-confirmed-2026)
3. [Scale Numbers (Memorize These)](#scale-numbers)
4. [Engineering Culture & Practices](#engineering-culture--practices)
5. [How They Ship Code](#how-they-ship-code)
6. [AI & Fin Agent](#ai--fin-agent)
7. [Role: Product Engineer](#role-product-engineer)
8. [What to Reference in Interview](#what-to-reference-in-interview)
9. [Questions to Ask Them](#questions-to-ask-them)

---

## Company Overview & Product

### What Intercom Does

> **"The best AI agent built on the best customer service platform"**

Intercom is an AI-first customer service platform. Their product enables:
- **Fin AI Agent** — AI that resolves customer questions automatically
- **Inbox** — Help desk for support agents
- **Help Center** — Knowledge base articles
- **Messenger** — Widget embedded in customer's websites/apps
- **Workflows** — Automated routing, triage, and response flows

### Key Users

| User Type | What They Do | Product Surface |
|-----------|-------------|----------------|
| End customers | Ask questions via Messenger widget | Customer-facing chat |
| Support agents | Answer questions, manage conversations | Inbox / Help Desk |
| Support leaders | Set up workflows, monitor metrics | Admin dashboard |
| AI (Fin) | Auto-resolve conversations | Behind the scenes |

### Business Model

- B2B SaaS, subscription pricing
- Customers: Large B2B and B2C companies
- Revenue driver: Seat-based pricing + AI resolution pricing
- Competitive advantage: AI + integrated platform (not point solutions)

---

## Tech Stack (Confirmed 2026)

### From their engineering blog (verified May 2026):

| Layer | Technology | Notes |
|-------|-----------|-------|
| **Primary App** | Ruby on Rails monolith | "Our core application" — deployed most frequently |
| **Frontend** | React + TypeScript | Mentioned "Intercom UI" as separate service |
| **Database** | MySQL + Vitess (PlanetScale) | 128 shards, 2M requests/sec |
| **Search** | Elasticsearch | 650TB, 1.7 trillion documents, 40k req/sec peak |
| **Caching** | Redis | 10M+ cache reads/sec in front of DB |
| **Infrastructure** | AWS | "We use AWS for infrastructure primitives" |
| **Queues** | AWS SQS | Fair queues for tenant isolation |
| **AI/LLM** | Multi-provider (GPT-4, Claude, etc.) | Cross-vendor failover, latency-based routing |
| **Deployment** | Custom pipeline | Merge → production in 12 min |
| **Feature Flags** | Custom (backend UI) | 560 flags created in 3 months |
| **Monitoring** | Datadog | Synthetics, heartbeat metrics |
| **Experiment** | GitHub Scientist | A/B parallel code execution |
| **CI/CD** | Parallel test suite | Finishes in <5 min |

### Key Architecture Decisions

- **Rails monolith** (not microservices) — "boring tech" philosophy
- **Vitess sharding** — moved from single MySQL to 128 shards in 2025
- **Horizontal scaling** — "thousands of large virtual machines"
- **Multi-region** — deployed to 3 data-hosting regions independently
- **Tenant isolation** — can dedicate a shard to a single customer

---

## Scale Numbers

### Memorize These (Use in System Design Discussions)

| Metric | Number | Context |
|--------|--------|---------|
| Peak requests/sec | **150,000+** | Customer requests into platform |
| Async requests/sec | **70,000+** | Background processing |
| Conversations/day | **5,000,000+** | On busiest days |
| Comments/day | **100,000,000+** | Messages across platform |
| DB requests/sec | **2,000,000** | Source-of-truth database layer |
| Cache reads/sec | **10,000,000+** | Redis in front of DB |
| ES storage | **650 TB** | Elasticsearch cluster |
| ES documents | **1.7 trillion** | Across all indexes |
| ES peak requests | **40,000/sec** | Search queries |
| Ships per day | **250** | Code deployments to production |
| Merge to production | **12 minutes** | Average deploy time |
| DB shards | **128** | Vitess/PlanetScale |
| Fin conversations/week | **2,000,000+** | AI agent resolution |
| Feature flags/quarter | **560+** | Created in 3 months |
| Single customer spike | **100+ new conversations/sec** | What they can handle per workspace |

### How to Use These Numbers

In system design round:
> "I know Intercom handles around 2 million database requests per second with 128 shards. For a messaging schema at that scale, I'd think about..."

In role chat:
> "I read your recent blog post about moving to Vitess with 128 shards — that's a fascinating migration. How does the team handle schema changes across shards?"

---

## Engineering Culture & Practices

### Core Philosophy: Speed = Safety

> "Speed is not the enemy of safety; it is the prerequisite for it."
> "Accumulating code creates risk; shipping small batches minimizes it."
> "Shipping is our company's heartbeat."

### Key Practices

| Practice | What It Means | How to Reference |
|----------|---------------|-----------------|
| **Ship in small steps** | 250 deploys/day, each change is tiny | "I love that you ship 250 times a day — I've always found small batches reduce risk" |
| **Boring technology** | Rails monolith, MySQL, familiar patterns | "I align with the boring tech philosophy — expertise compounds on familiar systems" |
| **Run less software** | Fewer systems → deeper expertise | "Reducing cognitive load by using fewer technologies resonates with me" |
| **Be present when you ship** | Watch your deploy, own the outcome | "I always monitor my changes after deployment" |
| **Heartbeat metrics** | Monitor customer outcomes, not systems | "Monitoring what matters to users, not just server health — that's the right approach" |
| **Feature flags** | Decouple deploy from release | "I'm a big fan of feature flags for controlled rollouts" |
| **Automatic rollback** | System reverts if heartbeat drops | Shows mature deployment infrastructure |
| **No blame culture** | Rollbacks are celebrated, not shamed | "A rollback means someone was watching — that's ownership" |

### The "Run Less Software" Philosophy

> "The point is not to have the smallest possible technology stack for its own sake. The point is to compound expertise. When many teams build on the same small set of technologies, our tooling, observability, and operational practice all improve together."

**What this means for your interview:**
- Don't suggest microservices unless they ask specifically
- Don't propose exotic databases — PostgreSQL/MySQL is fine
- Emphasize reusing patterns, not inventing new ones
- Show that you understand the COST of adding new technology

---

## How They Ship Code

### The Pipeline (12 minutes, fully automated)

```
1. Engineer merges to GitHub (human decision)
         │
2. Build slug (4 min) + Parallel CI (<5 min)
         │
3. Deploy to pre-production (2 min)
         │
4. Automated gates: boot test + CI check + Datadog Synthetics
         │
5. Production rollout: rolling restart across fleet (6 min)
         │
6. Engineer monitors heartbeat metrics
         │
7. Auto-rollback if heartbeat drops
```

### Key Engineering Expectations

- **You own your deploys** — "the engineer who writes the code is accountable for its success in production"
- **Watch the dials** — Notifications + observability links in every PR
- **Rollback fast** — "If you're not prepared to rollback, you're not prepared to ship"
- **Fix root cause** — "Resumption of service is never the end of the process"

### For Your Interview

Show you understand this by saying things like:
- "I'd ship this behind a feature flag first, enable for 5% of traffic..."
- "After deploying, I'd watch the key metrics for 10 minutes..."
- "If something looks off, I'd rollback immediately and investigate separately"
- "I'd break this into 3 smaller PRs, each independently deployable"

---

## AI & Fin Agent

### What Fin Is

- Intercom's AI agent that auto-resolves customer conversations
- Resolves **2 million+ conversations per week**
- Uses multiple LLM providers (GPT-4, Claude, etc.)
- Has cross-vendor failover, cross-model failover
- Latency-based routing between providers
- Maintains 2-3x buffer capacity for traffic spikes

### Fin Architecture (From Blog)

```
Customer Question → Fin Engine → LLM Routing Layer → Model Provider
                                      │
                        ┌──────────────┼──────────────┐
                        │              │              │
                   Provider A     Provider B     Provider C
                   (primary)      (fallback)    (overflow)
```

### Why This Matters for Your Interview

- **System Design:** If asked about AI integration, reference multi-provider routing with failover
- **Product Thinking:** Understand that AI is their primary product differentiator now
- **Scale Thinking:** AI workloads spike just like human support (product incidents → surge in questions)
- **Reliability:** "AI providers are not commodity storage systems" — they design for unreliability of upstream LLMs

---

## Role: Product Engineer

### What "Product Engineer" Means at Intercom

> NOT just a frontend or backend developer. A Product Engineer:

| Dimension | What They Expect |
|-----------|-----------------|
| **Full-stack** | Comfortable with Rails backend + React frontend |
| **Product thinking** | Understand WHY you're building, not just HOW |
| **Shipping** | Bias toward action; ship working features fast |
| **Ownership** | Own the feature from design to production monitoring |
| **Collaboration** | Work closely with PM, Design, and other engineers |
| **Craft** | Write clean code but don't over-engineer |

### What Sets This Role Apart from "Software Engineer"

| Software Engineer | Product Engineer |
|------------------|-----------------|
| Given requirements → implement | Help SHAPE requirements |
| Focuses on technical excellence | Focuses on user value |
| Owns code | Owns product outcomes |
| Builds what's asked | Asks "should we build this?" |
| Technical depth | Technical breadth + product sense |

### How to Signal "Product Engineer" in Interview

- Ask "What problem does this solve for the user?"
- Suggest simpler alternatives that still deliver value
- Mention metrics: "How would we measure success?"
- Talk about iteration: "Ship v1, learn, iterate"
- Reference cross-functional collaboration naturally

---

## What to Reference in Interview

### Drop These Knowledge Bombs (Naturally, Don't Force)

**During Coding Round:**
> "I'm thinking about this similar to how you'd assign conversations to agents in Intercom — same load balancing concept."

**During Values — "Build in Small Steps":**
> "I really resonate with your shipping philosophy. I read your January blog post about shipping 250 times a day with 12-minute deploys — that tight feedback loop is exactly how I think about safety."

**During Values — "Boring Technology":**
> "I noticed your blog mentions 'compound expertise' by building on familiar technologies. In my experience, I've seen teams waste months on exciting new tech when the proven solution would have shipped in days."

**During System Design:**
> "Given that your database layer handles 2 million requests per second across 128 shards, for this schema I'd think about partition key design early..."

**During Minicom:**
> "I see this is a Rails app — I'm familiar with the MVC pattern and ActiveRecord conventions."

**During Role Chat:**
> "I was really interested in your heartbeat metrics approach — monitoring customer outcomes rather than system health. How has that changed incident response for the team?"

---

## Questions to Ask Them

### Technical Questions (Show Depth)

1. "You moved to Vitess with 128 shards — how do teams handle cross-shard queries when they're needed?"
2. "With 250 deploys/day, how do you handle database migrations safely? Do you use something like `gh-ost` or Vitess's online DDL?"
3. "Your blog mentioned Fin uses multi-provider LLM routing — how do you handle response quality consistency across providers?"
4. "How does the team balance working on Fin AI features vs. platform reliability?"
5. "I saw you use GitHub Scientist for refactoring — how often do teams use that in practice?"

### Product/Culture Questions (Show Values Fit)

6. "How do Product Engineers participate in product discovery? Do you talk to customers directly?"
7. "What does 'shape the solution' look like in practice for a new engineer joining the team?"
8. "How do you decide when to invest in simplification vs. new features?"
9. "What's the onboarding experience like? How quickly do new PEs ship to production?"
10. "Your blog emphasizes 'no blame culture' — how does that manifest in incident reviews?"

### Role-Specific Questions

11. "What team would I be joining, and what product area does it own?"
12. "What's the biggest challenge the team is facing right now?"
13. "How much autonomy do engineers have in choosing WHAT to build vs. HOW to build it?"
14. "What does success look like in the first 90 days for a new Product Engineer?"

---

## Key Phrases from Their Own Blog (Use Their Language)

- "Every day counts"
- "Shape the solution"
- "Boring technology"
- "Build in small steps"
- "Run less software"
- "Compound expertise"
- "Shipping is your heartbeat"
- "Be present when you ship"
- "Speed is the prerequisite for safety"
- "Stop monitoring systems; start monitoring outcomes"
- "The safe path has to be built into the system"
- "We don't ship fast despite our need for stability; we ship fast to stay in control of change"

---

## Timeline / Recent News (2025-2026)

| Date | Event | Relevance |
|------|-------|-----------|
| May 2026 | "Ready for your busiest day: How we scale" blog | Shows current architecture & scale |
| Apr 2026 | "AI is approving our pull requests" blog | AI-native engineering practices |
| Jan 2026 | "The safety of speed: 180 ships/day" blog | Deployment culture |
| Oct 2025 | "Building R&D hub in Berlin" | Expansion |
| Jul 2025 | "Betting on the future of frontend" | Frontend modernization |
| 2025 | Completed Vitess/PlanetScale migration | Major DB architecture change |
| 2024 | "Evolving database infrastructure" blogs | The migration journey |
| 2024 | Rails World talk on outage lessons | Transparency about incidents |

---

## Red Flags to Avoid (Don't Say These)

| ❌ Don't Say | ✅ Say Instead |
|-------------|---------------|
| "I'd use microservices for this" | "I'd keep it in the monolith unless there's a compelling reason to extract" |
| "Let's use Kubernetes/Docker/Kafka" | "What does the team currently use? I'd prefer to build on existing infrastructure" |
| "We should rewrite this from scratch" | "I'd iterate on the existing system incrementally" |
| "I work best alone" | "I'm productive independently but love pairing on complex problems" |
| "That's not my responsibility" | "I'd flag it and either fix it myself or find the right person" |
| "I prefer backend only" | "I enjoy full-stack work — shipping end-to-end features is satisfying" |

---

*Previous: [← 17 InterviewDay CheatSheet](./17_InterviewDay_CheatSheet.md) | Next: [README →](./README.md)*
