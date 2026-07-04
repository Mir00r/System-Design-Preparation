---
description: "Use when implementing features in enterprise microservices that require strict API or event contracts, messaging semantics, observability, SLO-aware behavior, or controlled rollout and migration planning."
name: "Enterprise Microservices Shared Rules"
applyTo:
  - "**/*.java"
  - "**/*.kt"
  - "**/*.kts"
  - "**/*.go"
  - "**/*.py"
  - "**/*.sql"
  - "**/*.proto"
  - "**/*.yaml"
  - "**/*.yml"
---
# Enterprise Microservices Shared Rules

- Preserve service boundaries and ownership. Do not shortcut contracts with direct database coupling or hidden side channels.
- Preserve backward compatibility for public APIs, events, and schemas unless the task explicitly authorizes a breaking change.
- Prefer additive contract changes and staged migration plans.
- Treat producer rollout, consumer rollout, schema changes, and cleanup as one coordinated change set.
- Assume retries and duplicate message delivery unless the platform guarantees otherwise. Design idempotent consumers and safe writes.
- Define timeout, retry, fallback, and observability behavior for external calls and asynchronous processing.
- Preserve trace context, correlation IDs, and structured telemetry across service boundaries.
- Consider SLO impact for latency-sensitive paths and avoid adding synchronous fan-out without justification.
- Add tests for compatibility-sensitive serialization, contracts, migrations, and message handling.
