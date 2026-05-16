# 🎯 SLO, SLA, SLI & Alerting
## Error Budgets, On-Call Best Practices, and Fighting Alert Fatigue

> *"The goal of SRE is not to have no failures — it's to have the right number of failures. An SLO that's never violated means you're over-engineering reliability. An SLO that's always violated means your customers are suffering."*

**⏱️ Estimated Time**: 45 minutes | **🎯 Difficulty**: 🟡 Medium | **🔗 Prerequisites**: [Metrics & Monitoring](./Metrics_Monitoring.md)

---

## 📋 Table of Contents
1. [SLI, SLO, and SLA — The Definitions](#-sli-slo-and-sla--the-definitions)
2. [Error Budgets](#-error-budgets)
3. [Choosing the Right SLIs](#-choosing-the-right-slis)
4. [Defining SLOs in Practice](#-defining-slos-in-practice)
5. [Alerting That Works](#-alerting-that-works)
6. [Alert Fatigue — The Silent Killer](#-alert-fatigue--the-silent-killer)
7. [On-Call Best Practices](#-on-call-best-practices)
8. [Spring Boot: Implementing SLI Metrics](#-spring-boot-implementing-sli-metrics)
9. [Common Pitfalls](#-common-pitfalls)
10. [Mini Challenge](#-mini-challenge)
11. [Interview Q&A](#-interview-qa)

---

## 🤔 The Problem

Two teams run the same service:

**Team A**: "We had no outages last quarter! Zero downtime!"
- Actually had 3 hours of 500ms latency spikes (users noticed but no alerts fired)
- No one complained because they didn't know it was a violation — no SLO defined
- The team over-engineered the system, delaying 6 new features

**Team B**: "We violated our SLO in December."
- P99 latency exceeded 1 second for 2 hours during a deployment
- Error budget consumed: 20% (within acceptable range for the quarter)
- Root cause fixed, deployment process improved, customers were informed
- Shipped 4 features that quarter that drove 15% revenue growth

**Team B is doing it right.** Reliability is a feature, not a goal in itself.

---

## 📐 SLI, SLO, and SLA — The Definitions

### SLI — Service Level Indicator

A **measurable** metric that represents an aspect of service quality from the user's perspective.

```
Good SLIs are:
  ✅ Measurable with real data
  ✅ Reflect user experience directly
  ✅ Have a clear numerator and denominator

Examples:
  Availability:  good_requests / total_requests
  Latency:       requests completing in < 500ms / total_requests
  Error rate:    successful_requests / total_requests
  Freshness:     % of responses with data updated within 5 minutes

Bad SLIs:
  ❌ CPU utilization (internal, doesn't directly reflect user experience)
  ❌ "System is running" (binary, not nuanced)
```

### SLO — Service Level Objective

The **target** for an SLI. An internal commitment.

```
SLO Examples:
  "99.9% of requests will succeed over a rolling 28-day window"
  "P99 latency will be < 1 second for 99.5% of 5-minute windows"
  "Search results will be fresh within 5 minutes for 99% of queries"

SLO ≠ 100%:
  An SLO of 100% means:
  - Every deployment carries unacceptable risk (any change could cause a blip)
  - No room for maintenance windows
  - Feature work stops completely to pursue perfect reliability
  
  The right SLO leaves room to move fast while protecting user experience.
```

### SLA — Service Level Agreement

A **contractual commitment** to customers, with penalties for violation.

```
SLA vs SLO:
  SLO: "We aim for 99.9% availability" (internal goal, aspirational)
  SLA: "We guarantee 99.5% availability; if we fail, you get a credit" (legal contract)

Rule: SLA is always less strict than SLO.
  If SLO = 99.9% → SLA = 99.5% (buffer for unexpected violations)
  
  Violating the SLO is a warning sign.
  Violating the SLA costs money and damages trust.
```

### Summary Diagram

```
┌──────────────────────────────────────────────────────────┐
│                                                          │
│  SLI: "P99 latency over past 5 minutes"                  │
│  (What we measure)                                       │
│                                                          │
│  SLO: "P99 latency < 500ms for 99.9% of 5-min windows"  │
│  (What we aim for, internally)                           │
│                                                          │
│  SLA: "P99 latency < 1s for 99.5% of 5-min windows"     │
│  (What we promise customers, with refunds if violated)   │
│                                                          │
└──────────────────────────────────────────────────────────┘
```

---

## 💰 Error Budgets

An error budget is the **allowed downtime/failures** within the SLO target.

```
SLO: 99.9% availability over 28 days

Error budget:
  28 days = 40,320 minutes
  0.1% × 40,320 = 40.3 minutes of allowed downtime per 28-day window

  → Your team has 40 minutes to spend on failures, deployments, experiments

If your error budget is:
  FULL (haven't spent any): "We're being too conservative. Ship more features!"
  50% consumed: "Normal operations. Monitor closely."
  80% consumed: "Slow down deployments. Investigate stability issues."
  100% consumed: "STOP new deployments. Focus only on reliability until budget refills."
```

### Why Error Budgets Are Powerful

```
Before error budgets:
  Engineering: "We want to deploy 3x/day!"
  Operations:  "Too risky, we'll cause outages!"
  Conflict: Engineers vs Ops is constant.

After error budgets:
  Both teams share the same error budget.
  If engineering deploys too fast and consumes the budget → THEY slow down.
  If ops refuses all changes unnecessarily → budget stays full → prove it's safe.
  Data replaces opinion. Conflict disappears.
```

---

## 📊 Choosing the Right SLIs

Not everything deserves an SLI. Focus on what users care about:

| Service Type | Primary SLI | Secondary SLI |
|---|---|---|
| **API / Web Service** | Availability (success rate) | Latency (P99) |
| **Read-heavy service** | Latency (P99 read) | Freshness (data staleness) |
| **Write-heavy service** | Availability (write success) | Durability (writes persisted) |
| **Streaming/Batch** | Throughput (events/sec) | Completeness (no data loss) |
| **Background jobs** | Freshness (job recency) | Correctness (output validity) |

### The Four Golden Signal SLIs

```
1. AVAILABILITY
   Definition: fraction of requests that succeed
   Measurement: rate(http_requests_total{status=~"[^5].."}[5m]) / rate(http_requests_total[5m])
   Typical SLO: 99.9% (three nines) to 99.99% (four nines)

2. LATENCY
   Definition: fraction of requests completing within a threshold
   Measurement: histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m]))
   Typical SLO: 95% of requests < 200ms AND 99% < 1 second

3. THROUGHPUT (for data pipelines)
   Definition: events processed per second meets minimum
   Measurement: rate(events_processed_total[5m])
   Typical SLO: P50 throughput > 10,000 events/sec

4. FRESHNESS (for caches/feeds)
   Definition: fraction of data updated within the freshness threshold
   Measurement: % of items where updated_at > now() - 5 minutes
   Typical SLO: 99% of items fresh within 5 minutes
```

---

## 🔔 Alerting That Works

### The Two Alerting Methods

**Method 1: Target Error Rate** (alert immediately when error rate exceeds threshold)

```promql
# Alert if error rate > 1% for 2 minutes
ALERT HighErrorRate
  IF sum(rate(http_requests_total{status=~"5.."}[2m]))
     / sum(rate(http_requests_total[2m])) > 0.01
  FOR 2m
  LABELS {severity="critical"}
  ANNOTATIONS {summary="Error rate {{ $value | humanizePercentage }}"}
```

Problem: This is noisy — a 30-second spike triggers an alert even though you have plenty of error budget left.

**Method 2: Burn Rate Alerting** (alert based on how fast you're consuming the error budget)

```
Burn rate = current error consumption rate / expected steady rate

SLO: 99.9% availability → 0.1% budget → allowed to fail 0.1% of requests
Budget period: 30 days

If you fail 0.1% of requests: burn rate = 1x (normal)
If you fail 1%  of requests: burn rate = 10x (consuming budget 10x too fast)
If you fail 5%  of requests: burn rate = 50x (budget exhausted in 14 hours!)

Alert rules:
  Burn rate > 14.4x for 1h  → CRITICAL (budget exhausts in 5 days at this rate)
  Burn rate > 6x for 6h     → WARNING (budget exhausts in 5 days)
  Burn rate > 3x for 3 days → TICKET (investigate before next billing period)
```

```promql
# Burn rate alerting in Prometheus
# SLO: 99.9%, period: 30 days

# Fast burn: > 14.4x rate for 1 hour
ALERT BurnRateFast
  IF (
    sum(rate(http_requests_total{status=~"5.."}[1h]))
    / sum(rate(http_requests_total[1h]))
  ) > 0.001 * 14.4  # 0.001 = 0.1% budget, 14.4 = burn rate multiplier
  FOR 2m

# Slow burn: > 6x rate for 6 hours
ALERT BurnRateSlow
  IF (
    sum(rate(http_requests_total{status=~"5.."}[6h]))
    / sum(rate(http_requests_total[6h]))
  ) > 0.001 * 6
  FOR 2m
```

---

## 🚨 Alert Fatigue — The Silent Killer

Alert fatigue is when engineers receive so many alerts that they start ignoring them — including the critical ones.

```
Signs of alert fatigue:
  - On-call engineers silence alerts without investigating
  - Alerts have fired "for months" without being fixed
  - Multiple alerts fire simultaneously for the same root cause
  - Alerts fire during off-hours for non-urgent issues
  - Runbook links in alerts are stale or missing

The result: The alert that predicted the critical 3 AM outage was silenced
            two weeks ago as "too noisy." Nobody noticed.
```

### Fighting Alert Fatigue

```
1. EVERY ALERT MUST BE ACTIONABLE
   Bad: "CPU is above 70%" — so what? What do I do?
   Good: "DB connection pool > 85% — run this query, if stuck connections > 10, restart service"
   Rule: If you can't write a runbook step, the alert should not exist.

2. EVERY ALERT MUST HAVE A RUNBOOK
   Annotations should include: {{ runbook_url }}
   Runbook contains: symptoms, diagnosis steps, remediation steps, escalation path.

3. RESPECT ON-CALL HOURS
   PagerDuty routing:
     Business hours: Slack notification (not PagerDuty)
     Off-hours: Only page if a human MUST intervene NOW
   If an alert fires at 3 AM and the fix is "wait for it to auto-recover" → not a 3 AM alert.

4. NO ALERT SHOULD FIRE MORE THAN TWICE WITHOUT A FIX
   If an alert fires repeatedly, either:
     a) Fix the root cause
     b) Change the threshold if it's too sensitive
     c) Delete the alert if it's not actionable

5. USE ALERT GROUPS
   If 20 microservices all depend on DB and DB goes down: route to 1 alert, not 20.
   AlertManager grouping: group by [alertname, datacenter] → one page per incident.
```

---

## ⚙️ Spring Boot: Implementing SLI Metrics

```java
@Component
public class SliMetricsConfig {

    @Bean
    public WebMvcObservationFilter webMvcObservationFilter(MeterRegistry registry) {
        // Spring Boot auto-instruments http.server.requests with:
        //   tags: method, uri (template), status, exception
        //   measures: count, sum duration, histogram buckets
        // This powers both latency and availability SLIs.
        return new WebMvcObservationFilter(registry);
    }
}
```

```promql
# PromQL for SLI dashboards and SLO tracking

# Availability SLI: fraction of successful requests (non-5xx)
# "What % of requests succeeded in the last 5 minutes?"
sum(rate(http_server_requests_seconds_count{status!~"5.."}[5m]))
  / sum(rate(http_server_requests_seconds_count[5m]))

# Latency SLI: fraction of requests < 500ms
# "What % of requests completed within 500ms?"
sum(rate(http_server_requests_seconds_bucket{le="0.5"}[5m]))
  / sum(rate(http_server_requests_seconds_count[5m]))

# SLO compliance over 28 days (burn-down chart):
# "What is our cumulative availability over the past 28 days?"
sum(increase(http_server_requests_seconds_count{status!~"5.."}[28d]))
  / sum(increase(http_server_requests_seconds_count[28d]))
```

---

## ⚠️ Common Pitfalls

1. **Setting SLOs at 100%** — This creates extreme risk aversion. Every deployment is terrifying, features stall, and engineers burn out. The right SLO depends on the service criticality, but 100% is never the right answer.

2. **SLO misalignment with user experience** — "99.9% of API calls succeed" sounds good. But if your busiest 0.1% (checkout at peak hours) is failing, you're violating user trust while meeting the SLO. Use time-windowed or user-journey SLOs.

3. **Ignoring the error budget** — Teams define SLOs, measure them in Grafana, and then... never look at them. Error budgets require active management: someone must be responsible for the budget and empowered to stop deployments when it's exhausted.

4. **Too many SLOs** — Each SLO requires a metric, a dashboard, an alert, a runbook, and team attention. Start with 2-3 per service (availability + latency, usually). More is not better.

5. **Not including dependencies in SLO thinking** — Your service has 99.9% availability, but the downstream payment provider has 99.5%. Your effective availability is ≤ 99.5%. Factor external dependencies into your SLO.

---

## 🧩 Mini Challenge

**You've defined an SLO for `payment-service`: 99.9% availability over a 30-day window.**

Your monitoring shows: over the past 30 days, 0.08% of requests failed (better than SLO!). But in the past 24 hours, the error rate jumped to 2.5%.

**Questions**:
1. How many minutes of error budget remain for this month?
2. At the current burn rate of 2.5%, when will the error budget exhaust?
3. Should you page someone at 3 AM tonight?

<details>
<summary>💡 Click to reveal answer</summary>

**1. Error budget remaining:**
- 30-day error budget: 0.1% of requests
- Used so far: 0.08% (past 30 days) — but wait, 0.08% includes the bad 24 hours
- Previous 29 days: ~0.08% - (2.5% × 1/30) ≈ 0.08% - 0.083% = effectively almost zero previous error
- Remaining: 0.1% - 0.08% = **0.02% budget remaining**
- In minutes: 0.001 × 30 days × 24h × 60min = 43.2 minutes total; used ~34.6 minutes; **~8.6 minutes remaining**

**2. Budget exhaustion at current burn rate:**
- Burn rate = 2.5% actual / 0.1% budget rate = **25x burn rate**
- At 25x burn rate: remaining budget (0.02%) exhausted in:
  - 0.02% / 2.5% = 0.008 of 30 days = 0.24 days = **~6 hours**
- **The error budget will exhaust in ~6 hours at the current rate.**

**3. Should you page at 3 AM?**
Yes — this warrants a page, but not necessarily at 3 AM if you can act now:
- Burn rate of 25x means budget exhausts in 6 hours (breach of SLA threshold imminent)
- **Action now**: 
  - Is the error rate still at 2.5%? (check current metric)
  - If yes: page the on-call engineer immediately, don't wait for 3 AM
  - Trigger "error budget exhausted" response: halt non-critical deployments
  - Begin incident investigation
- **Alert rule**: A burn rate > 14.4x for 1 hour should have already triggered a CRITICAL alert. If it didn't, your alerting rules need updating.

</details>

---

## 📝 Interview Q&A

**Q: What is an error budget and how does it change engineering culture?**
> A: An error budget is the amount of unreliability (downtime, errors, latency violations) you're allowed before you've violated your SLO. For a 99.9% availability SLO over 30 days, the error budget is 0.1% — about 43 minutes of downtime. The culture shift: instead of ops saying "no deployments = no risk" and engineering saying "deploy constantly," both teams share the error budget as a common resource. If the budget is full, deploy freely. If it's nearly exhausted, stop deployments and focus on stability. Data replaces opinion in the reliability vs. velocity debate.

**Q: What's the difference between SLI, SLO, and SLA?**
> A: An SLI (Service Level Indicator) is a measured metric: "99.2% of requests succeeded this week." An SLO (Service Level Objective) is the internal target: "We aim for 99.9% success rate." An SLA (Service Level Agreement) is the external contract with penalties: "We guarantee 99.5%, or we credit your account." The relationship: SLA < SLO (always leave a buffer), and both are based on SLI measurements. Violating the SLO triggers internal action; violating the SLA costs money and damages customer trust.

**Q: How do you design alerts to avoid alert fatigue?**
> A: (1) Every alert must be actionable — if no human action is required, it's not an alert, it's a metric to watch on a dashboard. (2) Use burn rate alerting instead of raw threshold alerts — this ensures you're paged when it actually matters (budget is burning fast) rather than on every brief spike. (3) Every alert must link to a runbook with diagnosis steps. (4) Use alert grouping (AlertManager) so 20 alerts from the same root cause become one incident. (5) Review alerts quarterly: delete noisy ones, fix persistent ones, update thresholds. The goal is that every alert page represents a situation requiring human judgment.

---

## 🔗 What to Read Next

1. **[Observability/Metrics_Monitoring.md](./Metrics_Monitoring.md)** — Build the metrics that power your SLIs
2. **[Security/Authentication_vs_Authorization.md](../Security/Authentication_vs_Authorization.md)** — SLOs for auth services require special consideration (availability vs. security)
3. **[SystemDesignCaseStudies/DesignNotificationSystem.md](../SystemDesignCaseStudies/DesignNotificationSystem.md)** — A real system where SLOs define alert delivery guarantees

---

*[← Distributed Tracing](./Distributed_Tracing.md) | [Back to Observability](./README.md)*
