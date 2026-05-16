# 🏛️ How to Crack System Design Interviews

> *"System design interviews test your ability to think at scale, make trade-offs, and communicate complex ideas clearly. It's not about getting the 'right' answer — it's about demonstrating structured thinking."*

**⏱️ Estimated Time**: 30 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [InterviewPrep README](./README.md)

---

## 📋 Table of Contents
1. [The Framework (4 Steps)](#-the-framework)
2. [Step 1: Requirements & Scope](#-step-1-requirements)
3. [Step 2: High-Level Design](#-step-2-high-level-design)
4. [Step 3: Deep Dive](#-step-3-deep-dive)
5. [Step 4: Wrap-Up](#-step-4-wrap-up)
6. [Common Mistakes](#-common-mistakes)
7. [Practice Strategy](#-practice-strategy)

---

## 🎯 The Framework

```
45 MINUTE SYSTEM DESIGN INTERVIEW:

  ┌────────────────────────────────────────────────────────────────────┐
  │ 0-5 min   │ STEP 1: Requirements & Constraints                    │
  │           │ Ask questions, clarify scope, define what to build     │
  ├───────────┼────────────────────────────────────────────────────────┤
  │ 5-20 min  │ STEP 2: High-Level Design                             │
  │           │ API design, data model, architecture diagram           │
  ├───────────┼────────────────────────────────────────────────────────┤
  │ 20-40 min │ STEP 3: Deep Dive                                     │
  │           │ Scale bottlenecks, trade-offs, specific components     │
  ├───────────┼────────────────────────────────────────────────────────┤
  │ 40-45 min │ STEP 4: Wrap-Up                                       │
  │           │ Summary, improvements, monitoring, future scale        │
  └───────────┴────────────────────────────────────────────────────────┘
```

---

## 📝 Step 1: Requirements

```
ASK THESE QUESTIONS (don't skip!):

FUNCTIONAL REQUIREMENTS (what does it do?):
  "What are the core features?"
  "Who are the users? What actions do they perform?"
  "Do we need real-time or is near-real-time acceptable?"
  "What's the read/write ratio?"
  
NON-FUNCTIONAL REQUIREMENTS (how well?):
  "How many users? DAU/MAU?"
  "What's the expected QPS (queries per second)?"
  "What latency is acceptable? (p99 < 200ms?)"
  "Availability target? (99.9%? 99.99%?)"
  "How much data? Growth rate?"
  
CONSTRAINTS:
  "Any specific tech stack requirements?"
  "Budget constraints?"
  "Geography? (single region vs global?)"

BACK-OF-ENVELOPE CALCULATIONS:
  Users: 100M DAU
  Each user: 10 requests/day = 1B requests/day
  QPS: 1B / 86400 ≈ 12,000 QPS (peak: 2-3x = 30K QPS)
  Storage: 1B records × 1KB = 1TB/day
  
  Write these on the whiteboard — shows structured thinking!
```

---

## 🏗️ Step 2: High-Level Design

```
DRAW THE ARCHITECTURE:

  [Clients] → [Load Balancer] → [API Gateway]
                                      │
                    ┌─────────────────┼─────────────────┐
                    ▼                 ▼                  ▼
             [Service A]      [Service B]        [Service C]
                    │                │                  │
                    ▼                ▼                  ▼
              [Database]      [Cache]            [Message Queue]

DEFINE APIs:
  POST /api/v1/tweets
    Body: { "content": "...", "media_ids": [...] }
    Response: { "tweet_id": "...", "created_at": "..." }
  
  GET /api/v1/feed?user_id=123&cursor=abc&limit=20
    Response: { "tweets": [...], "next_cursor": "..." }

DATA MODEL:
  tweets: id, user_id, content, media_urls, created_at, like_count
  users: id, username, email, follower_count, following_count
  follows: follower_id, followee_id, created_at

DATABASE CHOICE (and WHY):
  "I'd use PostgreSQL for user data because it's relational with 
   strong consistency. For the tweet timeline/feed, I'd use Redis 
   for caching the pre-computed feed since it's read-heavy."
```

---

## 🔍 Step 3: Deep Dive

```
THE INTERVIEWER WILL ASK YOU TO GO DEEPER ON 1-2 AREAS:
  "How does the news feed work at scale?"
  "How do you handle a celebrity with 100M followers?"
  "What happens when a service goes down?"
  "How do you handle data consistency?"

STRUCTURE YOUR DEEP DIVE:
  1. Identify the challenge/bottleneck
  2. Present 2-3 options with trade-offs
  3. Choose one and explain WHY
  4. Discuss edge cases

EXAMPLE: "How does the feed work?"
  Option A: Fan-out on write (push)
    - When user posts, push to all followers' feeds
    - Pro: Read is fast (pre-computed)
    - Con: Celebrity posts → push to 100M feeds (slow write)
    
  Option B: Fan-out on read (pull)
    - When user opens feed, fetch from all followed users
    - Pro: Write is fast (just store tweet)
    - Con: Read requires merging N timelines (slow read)
    
  Option C: Hybrid (Twitter's approach)
    - Regular users: fan-out on write (push to followers)
    - Celebrities (>1M followers): fan-out on read
    - "I'd choose this because it optimizes for the common case 
       while handling the celebrity edge case separately."
```

---

## 🏁 Step 4: Wrap-Up

```
IN THE LAST 5 MINUTES:

  1. SUMMARIZE: "So our system handles X users with Y QPS using..."
  2. TRADE-OFFS: "The main trade-off is consistency vs latency..."
  3. IMPROVEMENTS:
     - "For monitoring, I'd add distributed tracing and dashboards"
     - "For resilience, circuit breakers between services"
     - "For scale beyond 10x, we could shard the database"
  4. BOTTLENECKS: "The main bottleneck at 100x scale would be..."
```

---

## ❌ Common Mistakes

```
1. DIVING INTO DETAILS TOO EARLY
   ❌ "Let me start with the database schema..."
   ✅ "Let me first clarify requirements, then outline the high-level design..."

2. NOT ASKING QUESTIONS
   ❌ Assuming scope (designing everything for "design Twitter")
   ✅ "Should we focus on the timeline, search, or both?"

3. SINGLE-SOLUTION THINKING
   ❌ "We should use Redis because it's fast"
   ✅ "We could use Redis (fast, in-memory) or DynamoDB (durable, scalable). 
       Given our latency requirements, I'd choose Redis with persistence."

4. IGNORING SCALE NUMBERS
   ❌ Designing without knowing traffic volume
   ✅ "With 30K QPS, a single PostgreSQL handles reads, but we need 
       read replicas for the feed queries."

5. NOT COMMUNICATING
   ❌ Drawing silently for 10 minutes
   ✅ "I'm thinking about the data flow here... Let me walk you through..."
```

---

## 📚 Practice Strategy

```
WEEK 1-2: Learn the building blocks
  - Caching, Load Balancing, Databases, Message Queues
  - Read: this repo's BuildingBlocks/ and KeyConcepts/ sections

WEEK 3-4: Practice common problems
  - Design URL Shortener (easy — warmup)
  - Design Twitter Feed (medium — fan-out)
  - Design WhatsApp (medium — real-time)
  - Design Netflix (hard — streaming, CDN)
  
WEEK 5+: Practice with timer (45 minutes)
  - Mock interviews with friends
  - Record yourself and review
  - Practice explaining trade-offs out loud
  
RESOURCES:
  - This repo's SystemDesignCaseStudies/ section
  - "System Design Interview" by Alex Xu
  - "Designing Data-Intensive Applications" by Martin Kleppmann
```

---

## 📝 Key Signals Interviewers Look For

| Signal | How to Demonstrate |
|---|---|
| Structured thinking | Follow the 4-step framework consistently |
| Trade-off analysis | "Option A gives us X but costs Y. Option B..." |
| Scale awareness | Back-of-envelope math, identify bottlenecks |
| Communication | Think aloud, check in with interviewer |
| Depth of knowledge | Deep dive into specific components when asked |
| Practical experience | Reference real-world systems (FAANG examples) |

---

## 🔗 What to Read Next

1. **[InterviewPrep/How_To_Crack_Coding_Rounds.md](./How_To_Crack_Coding_Rounds.md)** — Coding interview strategy
2. **[SystemDesignCaseStudies/How_To_Approach.md](../SystemDesignCaseStudies/How_To_Approach.md)** — Detailed approach guide
3. **[InterviewPrep/Mock_Interviews.md](./Mock_Interviews.md)** — Practice framework

---

*[← InterviewPrep README](./README.md) | [Back to Index](../INDEX.md) | [Next: Coding Rounds →](./How_To_Crack_Coding_Rounds.md)*
