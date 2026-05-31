# 📊 Performance Metrics & SLOs: Speaking the Performance Language

> *"If you can't measure it, you can't improve it. Performance engineering starts with defining WHAT 'fast' means for YOUR system — that's your SLO."*

**⏱️ Estimated Time**: 20 minutes | **🎯 Difficulty**: 🟢 Beginner | **🔗 Prerequisites**: None

---

## 📋 Table of Contents
1. [Key Metrics Explained](#-key-metrics-explained)
2. [Latency Percentiles (The p50/p95/p99 Mystery)](#-latency-percentiles)
3. [SLI, SLO, SLA — What's the Difference?](#-sli-slo-sla)
4. [Setting Performance Budgets](#-performance-budgets)
5. [Monitoring & Alerting Strategy](#-monitoring--alerting)
6. [Interview Q&A](#-interview-qa)
7. [Boss Battle](#-boss-battle)

---

## 📈 Key Metrics Explained

```
THE 4 PILLARS OF PERFORMANCE METRICS:

┌─────────────────────────────────────────────────────────────────┐
│  1. LATENCY (How fast?)                                         │
│     Time from request sent → response received                  │
│     Measured in: ms (milliseconds)                              │
│     Examples: p50=45ms, p95=120ms, p99=800ms                   │
│                                                                 │
│  2. THROUGHPUT (How much?)                                      │
│     Number of requests processed per unit time                  │
│     Measured in: RPS (requests per second), TPS (transactions)  │
│     Example: 10,000 RPS peak                                    │
│                                                                 │
│  3. ERROR RATE (How reliable?)                                  │
│     Percentage of requests that fail                            │
│     Measured in: % of 5xx errors                                │
│     Example: 0.01% error rate = 99.99% success                  │
│                                                                 │
│  4. SATURATION (How full?)                                      │
│     How close to capacity is the system?                        │
│     Measured in: % of CPU, memory, connections, threads         │
│     Example: 75% CPU utilization at peak                        │
└─────────────────────────────────────────────────────────────────┘

These 4 are Google's "Golden Signals" (from the SRE book)!
```

---

## ⏱️ Latency Percentiles

### Why Average is LYING to You 🤥

```
SCENARIO: API response times over 100 requests
  95 requests: 50ms each
  5 requests: 10,000ms each (10 seconds!)

AVERAGE = (95 × 50 + 5 × 10000) / 100 = 547ms
  → "Average is 547ms, not bad!" 🤷
  → BUT 5% of users wait 10 SECONDS! 😡

PERCENTILES tell the REAL story:
  p50 (median): 50ms   ← Half the users see this or better
  p95:          50ms   ← 95% of users see this or better  
  p99:          10000ms ← 1% of users see THIS nightmare!

RULE: Always measure p50, p95, p99 — never just average!
```

### Percentile Quick Reference

| Percentile | Meaning | Why It Matters |
|-----------|---------|---------------|
| **p50** | Median (half see this or better) | Typical user experience |
| **p90** | 10% of users are slower than this | Power users, heavy requests |
| **p95** | 5% are slower | Industry standard for SLOs |
| **p99** | 1% are slower | Tail latency — your worst users |
| **p99.9** | 1 in 1000 are slower | Enterprise/finance SLOs |

### 🎮 The Concert Ticket Analogy 🎫

```
Imagine buying concert tickets online:
  p50 = 2 seconds → Most fans get tickets in 2s (happy! 🎉)
  p99 = 45 seconds → 1% of fans wait 45s (frustrated! 😤)
  p99.9 = timeout → 0.1% get errors (FURIOUS! 🤬 → tweet storm)

At 1 MILLION fans:
  p99 = 10,000 people with terrible experience
  p99.9 = 1,000 people who couldn't buy tickets at all

That's why p99 matters at scale!
```

---

## 🎯 SLI, SLO, SLA

```
SLI (Service Level Indicator) = WHAT you measure
  "p99 latency of the /checkout API"

SLO (Service Level Objective) = TARGET you set internally
  "p99 latency < 500ms for 99.9% of requests"

SLA (Service Level Agreement) = PROMISE to customers (with consequences!)
  "99.95% uptime or we refund pro-rata"

RELATIONSHIP:
  SLI → measurable metric
  SLO → internal target (tighter than SLA!)
  SLA → external promise (always more lenient than SLO)
  
  If SLO = 99.99% → set SLA = 99.95% (buffer for when things go wrong)
```

### Real-World SLO Examples

| Service | SLO | Why? |
|---------|-----|------|
| **Payment API** | p99 < 200ms, 99.99% success | Money! Can't lose transactions |
| **Product Search** | p95 < 300ms, 99.9% success | UX critical, but tolerates rare failures |
| **Email Notifications** | p99 < 5s, 99.5% success | Async, users don't notice delay |
| **Batch Reports** | Complete < 1 hour | Not user-facing, relaxed timing |

---

## 💰 Performance Budgets

```
PERFORMANCE BUDGET = Total time allowed, allocated across components

Example: "Product page must load in < 2 seconds"
  ├── DNS lookup:        50ms
  ├── TCP handshake:     50ms  
  ├── TLS negotiation:   50ms
  ├── Server processing: 200ms  ← YOUR backend budget!
  ├── Network transfer:  100ms
  ├── Client rendering:  500ms
  └── Buffer:            1050ms (for variability)
  
  Your backend gets 200ms. That's split further:
  ├── Controller logic:  10ms
  ├── Service logic:     20ms
  ├── Database query:    100ms  ← Usually the biggest chunk!
  ├── External API call: 50ms
  └── Serialization:     20ms
```

---

## 📊 Monitoring & Alerting

### The RED Method (for request-driven services)

| Signal | What | Alert When |
|--------|------|-----------|
| **R**ate | Requests per second | Sudden spike/drop |
| **E**rrors | Error count/rate | > 1% of requests |
| **D**uration | Latency percentiles | p99 > SLO |

### The USE Method (for resources)

| Signal | What | Alert When |
|--------|------|-----------|
| **U**tilization | % of resource used | > 80% sustained |
| **S**aturation | Work queued | Queue growing unbounded |
| **E**rrors | Error events | Any hardware errors |

---

## 🎓 Interview Q&A

### Q1: "What's the difference between p95 and p99 latency?"
**A**: p95 means 95% of requests are faster than this value; p99 means 99% are faster. At scale, the difference matters enormously — if you have 1M requests/day, p99=1s means 10,000 users per day have a terrible experience.

### Q2: "How do you set an SLO?"
**A**: Measure current performance (baseline), understand user expectations, consider business impact. Set SLO slightly tighter than what users tolerate, but achievable. Use error budgets to balance reliability with development velocity.

### Q3: "Why not just optimize everything to be as fast as possible?"
**A**: Optimization has diminishing returns and costs engineering time. Going from 100ms to 50ms might take a week, but going from 50ms to 25ms might take 3 months. Set SLOs based on business value, optimize only what's below target.

---

## 🎲 Boss Battle: Set the SLOs 🎯

> **Scenario**: You're designing SLOs for a food delivery app with these services:
> - Restaurant search API
> - Order placement API
> - Payment processing API
> - Delivery tracking (real-time GPS)
> - Push notifications
>
> **Challenge**: Set appropriate p50, p99, and availability SLOs for each.
>
> <details>
> <summary>🔓 Click to reveal answer</summary>
>
> | Service | p50 | p99 | Availability | Reasoning |
> |---------|-----|-----|-------------|-----------|
> | Restaurant search | <100ms | <500ms | 99.9% | User-facing, browsing UX |
> | Order placement | <200ms | <1s | 99.99% | Revenue-critical, must not lose orders |
> | Payment processing | <300ms | <2s | 99.99% | Money! Legal/compliance implications |
> | Delivery tracking | <100ms | <500ms | 99.5% | Real-time UX, but brief outage tolerable |
> | Push notifications | <5s | <30s | 99% | Async, users won't notice seconds of delay |
>
> **Key Insight**: SLOs reflect BUSINESS VALUE, not technical difficulty. Payment is simpler than search but has a higher SLO because losing a payment = losing revenue + trust.
> </details>

---

## 🏆 Achievement Unlocked!

```
┌──────────────────────────────────────────────────────────────┐
│  📊 ACHIEVEMENT: Latency Learner Level 1                     │
│                                                              │
│  You now understand:                                         │
│  ✅ The 4 golden signals of performance                      │
│  ✅ Why percentiles > averages                               │
│  ✅ SLI vs SLO vs SLA                                        │
│  ✅ How to set performance budgets                            │
│                                                              │
│  NEXT: → Profiling & Optimization                            │
└──────────────────────────────────────────────────────────────┘
```

👉 **[Next: Profiling & Optimization →](./Profiling_and_Optimization.md)**  
👉 **[Back to Performance Overview →](./README.md)**
