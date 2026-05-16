# 🔭 Observability
## Logs, Metrics, and Traces — The Three Pillars

> *"You can't fix what you can't see. Observability is the ability to understand the internal state of a system from its external outputs. Without it, you're flying blind in production."*

---

## 🎯 What Is Observability?

Observability is the practice of instrumenting your systems so you can answer questions about their behavior in production — especially questions you didn't know you'd need to ask when you built the system.

**Monitoring** (traditional): Did this specific thing I expected to go wrong, go wrong?
**Observability** (modern): Why is this happening? What is the system doing right now?

---

## 🏛️ The Three Pillars

```
┌──────────────────────────────────────────────────────────────────┐
│                    OBSERVABILITY                                 │
│                                                                  │
│  ┌──────────────┐   ┌──────────────┐   ┌────────────────────┐   │
│  │     LOGS     │   │   METRICS    │   │      TRACES        │   │
│  │              │   │              │   │                    │   │
│  │ What happened│   │ How much /   │   │ How did request X  │   │
│  │ (events,     │   │ how often    │   │ flow through all   │   │
│  │  errors,     │   │ (counters,   │   │ services?          │   │
│  │  audit trail)│   │  gauges,     │   │ (distributed call  │   │
│  │              │   │  histograms) │   │  graph with timing)│   │
│  └──────────────┘   └──────────────┘   └────────────────────┘   │
│                                                                  │
│  Tools:  ELK/Loki     Prometheus        Jaeger/Zipkin/Tempo     │
│          Splunk        Grafana           OpenTelemetry           │
└──────────────────────────────────────────────────────────────────┘
```

| Pillar | Answers | Tools | Storage |
|---|---|---|---|
| **Logs** | "What happened?" — detailed event records | ELK Stack, Loki, Splunk | High volume, compressed |
| **Metrics** | "How is the system performing over time?" | Prometheus, Datadog, CloudWatch | Time-series, aggregated |
| **Traces** | "How did this request flow through the system?" | Jaeger, Zipkin, OpenTelemetry | Sampled, graph data |

---

## 📚 Contents

| File | What You'll Learn |
|---|---|
| [Logging Best Practices](./Logging_Best_Practices.md) | Structured logging, log levels, ELK stack, what to log (and what NOT to) |
| [Metrics & Monitoring](./Metrics_Monitoring.md) | Prometheus, Grafana, RED/USE methods, alerting on the right signals |
| [Distributed Tracing](./Distributed_Tracing.md) | OpenTelemetry, Jaeger, trace context propagation across microservices |
| [SLO, SLA, SLI & Alerting](./Alerting_SLO_SLA_SLI.md) | Error budgets, on-call best practices, alert fatigue, PagerDuty |

---

## 🔗 Why All Three Together?

```
Scenario: Users report the checkout page is slow.

Without observability:
  "I'll deploy a fix and hope it works."

With Metrics:
  "Response time P99 spiked from 200ms to 3s at 14:32."
  → Tells you WHEN it happened and HOW BAD it is.

With Logs:
  "ERROR: database connection pool exhausted at 14:32"
  → Tells you WHAT happened.

With Traces:
  "Request ID x7f2b: 2800ms spent waiting in order-service → db connection wait"
  → Tells you WHERE exactly (which service, which operation).

All three together = root cause found in 5 minutes instead of 2 hours.
```

---

## 🔗 Related Domains

- [BuildingBlocks/](../BuildingBlocks/) — Each building block should emit all three signal types
- [Microservices/](../Microservices/) — Observability is harder and more important in microservices
- [DevOps/](../DevOps/) — Observability is the feedback loop for CI/CD

---

*[← Back to Index](../INDEX.md)*
