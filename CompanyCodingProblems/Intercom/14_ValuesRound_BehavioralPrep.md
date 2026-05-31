# 🟣 Intercom | Values Assessment — Deep Behavioral Preparation

> **Round:** 2B — Values Assessment (45 min, same day as coding)
> **Format:** 3 deep examples, ~15 min each, using SBI (Situation-Behaviour-Impact) framework
> **Key:** They explain a value, then ask YOU for a matching experience from your career

---

## 📋 Table of Contents

1. [How This Round Works](#how-this-round-works)
2. [Intercom's 5 Engineering Values](#intercoms-5-engineering-values)
3. [Pre-Written Story Bank (15 Stories)](#pre-written-story-bank)
4. [Specific Questions Asked (from Glassdoor)](#specific-questions-asked)
5. [Answer Templates & Frameworks](#answer-templates--frameworks)
6. [Red Flags to Avoid](#red-flags-to-avoid)
7. [Practice Drills](#practice-drills)

---

## How This Round Works

### From Intercom's own blog:

> "We explain a particular value, why we think it's important, give an example of what we mean, then ask the candidate to talk about a similar situation from their own experience."

### The Flow:

```
Interviewer: "At Intercom, we believe in [VALUE]. For example, we recently [STORY].
             Can you tell me about a time when you demonstrated something similar?"

You: [3-4 min structured answer using SBI]

Interviewer: "Interesting. What specifically was YOUR role in that?"
             "How long did that take?"
             "What would you do differently?"
             "What was the measurable impact?"

You: [Specific, data-driven follow-ups]
```

### They Cover 3 Values in 45 Minutes

- ~5 min intro + value explanation
- ~5 min your answer
- ~5 min probing follow-ups
- Repeat × 3

---

## Intercom's 5 Engineering Values

### Value 1: "Shape the Solution"

> "We never blindly execute on requirements defined by others. We deeply understand the value of our work, and help design solutions which efficiently deliver that value."

**What they're looking for:**
- You pushed back on a poorly-defined requirement
- You identified a simpler solution than what was requested
- You understood the "why" behind a task, not just the "what"
- You collaborated with PM/design to redefine scope

**Signal phrases to use:**
- "I noticed the original requirement didn't account for..."
- "Instead of building X as specified, I proposed Y because..."
- "I went back to the PM and asked what problem we're actually solving"

---

### Value 2: "Be Technically Conservative"

> "We like familiar solutions with boring technologies. We reuse the same patterns in different solutions as much as possible."

**What they're looking for:**
- You chose a proven technology over a trendy one
- You reused existing patterns/libraries instead of building from scratch
- You resisted over-engineering
- You evaluated tech choices based on team familiarity, not personal interest

**Signal phrases to use:**
- "Rather than introducing a new framework, I reused our existing..."
- "I proposed PostgreSQL because the team already knew it well"
- "We considered [trendy tech] but went with [boring tech] because..."
- "I followed the same pattern we used in [other service]"

---

### Value 3: "Build in Small Steps"

> "Large changes are hard to understand, and harder to debug. We deliver complex changes in a series of small, controlled, easy to understand steps."

**What they're looking for:**
- You broke a large feature into incremental PRs
- You used feature flags for gradual rollout
- You delivered an MVP first, then iterated
- You identified risks and mitigated them with phased delivery

**Signal phrases to use:**
- "I broke this into 4 smaller PRs, each independently deployable"
- "We used a feature flag so we could roll back instantly"
- "The first version only handled the 80% case; we iterated on edge cases later"
- "I shipped to 5% of traffic first, monitored, then expanded"

---

### Value 4: "Keep it Simple"

> "We trade performance/cost for simplicity wherever we can."

**What they're looking for:**
- You simplified an over-engineered system
- You chose clarity over cleverness
- You made a trade-off that sacrificed some performance for maintainability
- You removed unnecessary complexity

**Signal phrases to use:**
- "The existing code was over-engineered for our scale; I simplified it to..."
- "We accepted O(n) instead of O(log n) because n was always < 100"
- "I removed the caching layer because the database was already fast enough"
- "I replaced the custom framework with a simple array iteration"

---

### Value 5: "Work with Positivity, Pride, and Love"

> "We care about craftsmanship, collaboration, and team culture."

**What they're looking for:**
- You helped a struggling teammate without being asked
- You took ownership of something broken (not your fault)
- You gave or received constructive feedback gracefully
- You improved team processes
- You celebrated others' wins

**Signal phrases to use:**
- "I noticed the on-call engineer was overwhelmed, so I..."
- "Even though it wasn't my code, I took responsibility for..."
- "After the incident, I set up a blameless retro and..."
- "I created a shared runbook so the whole team could..."

---

## Pre-Written Story Bank (15 Stories)

> **Customize these with YOUR actual experiences.** Replace [brackets] with specific details.

---

### Story 1: Shaped a Better Solution (Value 1)

**Situation (30 sec):**
> "In my previous role at [Company], the product team requested a real-time notification system that would send push notifications for every single event in the system — status changes, comments, assignments, etc. The estimate was 6 weeks."

**Behaviour (2 min):**
> "Instead of jumping into implementation, I analyzed our analytics data and found that 78% of notifications were being dismissed without reading. I went back to the PM and proposed a 'smart digest' approach — batch non-urgent notifications into a 15-minute digest, and only send real-time for high-priority events. This reduced the scope to 2 weeks and actually improved user engagement."

**Impact (30 sec):**
> "Notification open rate went from 12% to 41%. We shipped in 2 weeks instead of 6. The PM said it was the best pushback they'd received — it improved the product AND reduced scope."

**Follow-up answers:**
- "My specific role was: I analyzed the data, proposed the alternative, built the batching logic, and presented to stakeholders."
- "What I'd do differently: I'd involve the PM earlier in the data analysis so they felt more ownership of the decision."

---

### Story 2: Chose Boring Technology (Value 2)

**Situation:**
> "We needed to add real-time features to our application. Three team members were excited about using GraphQL subscriptions with a new event-sourcing architecture."

**Behaviour:**
> "I suggested we use server-sent events (SSE) with our existing REST API and PostgreSQL LISTEN/NOTIFY. It wasn't as elegant, but our entire team already knew PostgreSQL, we had existing monitoring for it, and the ops overhead was zero. I created a small proof-of-concept that demonstrated it handled our load (5,000 concurrent connections) with a single Node.js process."

**Impact:**
> "We shipped the feature in 1 week instead of the estimated 4 weeks. Zero new dependencies in production. When we had a bug 3 months later, any engineer on the team could debug it without learning a new system."

---

### Story 3: Built in Small Steps (Value 3)

**Situation:**
> "We needed to migrate our payment system from Stripe's old API to the new one. It touched 40+ files and affected all paying customers."

**Behaviour:**
> "Instead of a big-bang migration, I broke it into 5 phases:
> 1. Add new SDK alongside old (no behavior change)
> 2. Dual-write: new purchases use new API, old subscriptions stay on old
> 3. Background job to migrate existing subscriptions (10% daily)
> 4. Verify all migrated with reconciliation script
> 5. Remove old SDK
> Each phase was a separate PR with its own feature flag."

**Impact:**
> "Zero downtime, zero customer complaints. Found 3 edge cases during the phased rollout that would have been critical bugs in a big-bang migration. Total migration took 3 weeks but each individual change was low-risk."

---

### Story 4: Simplified Over-Engineered System (Value 4)

**Situation:**
> "I inherited a microservice that used RabbitMQ, Redis, and a custom state machine to process CSV imports. It had frequent deadlocks and took 3 engineers 2 days to debug each incident."

**Behaviour:**
> "I analyzed the actual throughput: 50 imports/day, average file size 500 rows. This didn't need a distributed system. I replaced the entire pipeline with a single synchronous endpoint that processed the CSV in-memory and wrote directly to PostgreSQL. I used a simple database-backed job queue (Sidekiq) for the few files over 10k rows."

**Impact:**
> "Reduced code from 2,400 lines to 380. Eliminated all deadlock incidents. New engineers could understand the system in 30 minutes instead of 3 days. Processing time actually improved from 45s to 12s because we removed the message serialization overhead."

---

### Story 5: Took Ownership / Positive Culture (Value 5)

**Situation:**
> "During an on-call shift at 2 AM, I received an alert for a service I'd never worked on. The usual owner was on vacation."

**Behaviour:**
> "Instead of escalating to a manager, I spent 30 minutes reading the service's runbook, identified the issue (a database connection pool exhaustion due to leaked connections), applied the documented fix (restart + config change), and monitored for 2 hours. The next morning, I wrote up the incident, created a permanent fix (connection timeout config), and updated the runbook with the pattern I discovered."

**Impact:**
> "The fix prevented the issue from recurring (it had happened 3 times in the previous month). The runbook update helped 2 other engineers resolve similar issues. The team lead mentioned it in the sprint retro as an example of ownership culture."

---

### Story 6: Biggest Professional Mistake (Value 5 - Self Awareness)

**Situation:**
> "Early in my career, I deployed a database migration to production that added a NOT NULL column without a default value. This locked the table for 12 minutes on a table with 50M rows, causing a partial outage."

**Behaviour:**
> "I immediately rolled back, communicated the issue to the team, and spent the next day investigating what went wrong in our process. I discovered we had no automated migration safety checks. I implemented a CI check using `strong_migrations` (a gem that catches unsafe migrations) and added a deployment checklist for schema changes. I also did a knowledge-sharing session with the team about PostgreSQL locking behavior."

**Impact:**
> "We never had a migration-related incident again in 18 months. The CI check caught 7 unsafe migrations during that time. I turned my mistake into a team improvement."

**What I'd do differently:**
> "I should have tested the migration on a production-sized dataset first. Now I always run migrations against a staging environment with realistic data before production."

---

### Story 7: Handled Conflict (Value 5)

**Situation:**
> "A senior engineer on my team insisted we use MongoDB for a new feature that clearly had relational data (users, orders, payments with foreign key relationships). They had strong opinions and the team was deferring to seniority."

**Behaviour:**
> "Instead of arguing in the meeting, I prepared a one-page comparison. I modeled the same 3 core queries in both MongoDB and PostgreSQL, showed the data consistency risks with denormalization, and quantified the operational complexity (we had zero MongoDB expertise on the team). I presented it as 'here's what I found — let's discuss' rather than 'you're wrong.'"

**Impact:**
> "The team chose PostgreSQL. The senior engineer actually thanked me afterward for doing the analysis — they admitted they were defaulting to MongoDB out of habit from a previous company. We shipped on time, and the relational model saved us weeks of debugging later when we needed to add cross-entity reporting."

---

### Story 8: Delivered Under Pressure (Value 1 + 3)

**Situation:**
> "Two weeks before a major product launch, the PM added a requirement for real-time analytics — tracking how many users interacted with each feature within the first hour. The original plan had no analytics."

**Behaviour:**
> "I assessed: building a proper analytics pipeline would take 3 weeks. Instead, I proposed a 'good enough' solution: log key events to our existing structured logging, then use our Grafana dashboards to visualize in near-real-time. Accuracy wasn't critical for launch day; trend direction was enough. I built it in 2 days."

**Impact:**
> "We had working analytics for launch day. The data showed that 73% of users engaged with the new feature within 5 minutes. 3 months later, we replaced it with a proper analytics pipeline — but by then, we had launch data to guide the design of the permanent solution."

---

### Story 9: Improved Team Process (Value 5)

**Situation:**
> "Our team's PR review process was creating bottlenecks. PRs sat for 2–3 days before review, developers were context-switching back to stale PRs, and morale was dropping."

**Behaviour:**
> "I proposed a 'review buddy' rotation: each week, two engineers are paired as review partners. They commit to reviewing each other's PRs within 4 hours. I set up a Slack bot reminder and tracked the metrics: time-to-first-review and time-to-merge."

**Impact:**
> "Average time-to-first-review dropped from 26 hours to 3.5 hours. Time-to-merge dropped from 3.2 days to 1.1 days. Developer satisfaction (quarterly survey) on 'code review process' went from 3.2/5 to 4.6/5. Other teams adopted the same system."

---

### Story 10: Product Thinking (Value 1)

**Situation:**
> "I was building a search feature for customer support tickets. The PM's spec said 'full-text search across all ticket fields.'"

**Behaviour:**
> "I realized that support agents don't typically search by message body — they search by customer name, ticket status, or assignment. I analyzed 1 month of existing search queries from our old system and found that 89% of searches were structured (name, status, date range) and only 11% were free-text. I proposed a split UI: a structured filter panel (instant results from indexed columns) + a separate full-text search for the rare case."

**Impact:**
> "Search results returned in 50ms (structured) vs. 2-3 seconds (full-text Elasticsearch). Agent productivity improved because 89% of their searches were now instant. The PM incorporated this insight into the product roadmap for other tools."

---

### Stories 11-15 (Templates — Fill with YOUR Experiences)

| # | Value | Situation Template |
|---|-------|-------------------|
| 11 | Shape Solution | Time you reduced scope by understanding user needs |
| 12 | Conservative | Time you migrated FROM a complex solution to a simpler one |
| 13 | Small Steps | Time you used feature flags / canary deployment |
| 14 | Keep Simple | Time you said "no" to a feature that added complexity |
| 15 | Positive | Time you mentored a junior engineer / unblocked someone |

---

## Specific Questions Asked (from Glassdoor)

### Confirmed Questions (2022–2026)

| Question | Which Value | Your Story # |
|----------|-------------|-------------|
| "What's your proudest professional achievement?" | Any (pick strongest) | Story # ___ |
| "What's your biggest professional mistake?" | Positivity (self-awareness) | Story #6 |
| "Tell me about a time you faced conflict with a colleague" | Positivity | Story #7 |
| "Describe a project you built from scratch" | Shape + Small Steps | Story # ___ |
| "Tell me about a time you noticed broken code that wasn't yours" | Positivity (ownership) | Story #5 |
| "How do you handle feedback that you disagree with?" | Positivity | Story #7 (variant) |
| "Describe a product engineering challenge" | Shape Solution | Story #10 |
| "Tell me about a time you shipped something faster than expected" | Small Steps + Conservative | Story #8 |
| "How do you prioritize when everything is urgent?" | Shape Solution | Story # ___ |
| "Tell me about a time you improved a team process" | Positivity | Story #9 |

---

## Answer Templates & Frameworks

### SBI Template (Use for EVERY answer)

```markdown
**SITUATION** (30 seconds — set the scene):
"At [Company], in [Year/Quarter], I was working on [Project].
 The challenge was [specific problem/opportunity].
 The team was [size] and I was the [role]."

**BEHAVIOUR** (2 minutes — what YOU did):
"First, I [analyzed/researched/identified]...
 I considered [Option A] vs [Option B]...
 I chose [approach] because [reason]...
 Specifically, I [concrete action 1]...
 Then I [concrete action 2]..."

**IMPACT** (30 seconds — measurable result):
"As a result, [metric improved by X%].
 [Concrete outcome: shipped faster / fewer bugs / team happier].
 Looking back, I would [one thing to do differently]."
```

### Quantification Cheat Sheet

**Always convert your impact to numbers:**

| Vague | Specific |
|-------|----------|
| "It was faster" | "Response time dropped from 2.3s to 180ms" |
| "Fewer bugs" | "Bug reports decreased by 43% in the following sprint" |
| "Team was happier" | "Developer satisfaction score went from 3.2 to 4.6 out of 5" |
| "Saved time" | "Reduced deployment time from 45 min to 8 min" |
| "Improved process" | "PR review time dropped from 26 hours to 3.5 hours" |
| "Shipped quickly" | "Delivered in 2 weeks instead of the estimated 6" |

---

## Red Flags to Avoid

### Things that WILL hurt you:

| ❌ Red Flag | ✅ What to Do Instead |
|------------|---------------------|
| "The QA team should have caught it" | "I should have added better test coverage" |
| "My manager made a bad decision" | "I could have presented my concerns more clearly" |
| "We did X" (without clarifying your role) | "The team did X. My specific contribution was Y" |
| "I think it helped" (vague impact) | "It reduced incidents from 4/month to 0" |
| "I don't make mistakes" | Own a real mistake, show learning |
| "I prefer to work alone" | "I'm most productive with clear context, and I love pairing on complex problems" |
| Using the word "just" ("I just fixed a bug") | "I identified a production issue affecting 10k users and resolved it within 2 hours" |
| Not having follow-up details | Have dates, team sizes, metrics ready |

---

## Practice Drills

### Drill 1: Timer Practice (Do 3x before interview)

Set a 4-minute timer. Pick a story. Tell it out loud:
- 30 sec: Situation
- 2.5 min: Behaviour  
- 1 min: Impact

Record yourself. Listen back. Fix:
- Did you say "we" without clarifying your role?
- Did you give a number for impact?
- Did you mention what you'd do differently?

### Drill 2: Follow-Up Prep

For each story, prepare answers to:
1. "Who else was involved?"
2. "How long did this take?"
3. "What was the hardest part?"
4. "What would you do differently?"
5. "How did you know it worked?"
6. "Did anyone disagree? How did you handle it?"

### Drill 3: Map Stories to Values

Fill this matrix — you should have at least 2 stories per value:

| Value | Primary Story | Backup Story |
|-------|--------------|--------------|
| Shape the Solution | #___ | #___ |
| Be Technically Conservative | #___ | #___ |
| Build in Small Steps | #___ | #___ |
| Keep it Simple | #___ | #___ |
| Positivity, Pride, Love | #___ | #___ |

### Drill 4: The "Mistake" Question

This is asked 90% of the time. Your answer MUST show:
1. A real mistake (not a humble-brag)
2. Immediate accountability (no blame)
3. Concrete action to prevent recurrence
4. Measurable evidence it worked
5. What you learned personally

---

## Quick Reference Card (Print for Interview Day)

```
┌──────────────────────────────────────────┐
│  VALUES ROUND CHEAT SHEET                │
├──────────────────────────────────────────┤
│                                          │
│  Format: SBI (Situation-Behaviour-Impact)│
│  Time:   30s + 2min + 30s per answer     │
│  Depth:  3 stories × 15 min each         │
│                                          │
│  ALWAYS:                                 │
│  • Give numbers (%, hours, $)            │
│  • Clarify YOUR role ("I specifically...") │
│  • End with "what I'd do differently"    │
│  • Connect to their value explicitly     │
│                                          │
│  NEVER:                                  │
│  • Blame others                          │
│  • Give vague impact ("it helped")       │
│  • Say "we" without clarifying "I"       │
│  • Claim you don't make mistakes         │
│                                          │
│  TOP 3 STORIES READY:                    │
│  1. _________________________________    │
│  2. _________________________________    │
│  3. _________________________________    │
└──────────────────────────────────────────┘
```
