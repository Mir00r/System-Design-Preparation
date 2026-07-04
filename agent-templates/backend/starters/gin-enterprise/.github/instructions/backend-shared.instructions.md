---
description: "Use when implementing backend features in enterprise Go microservices that require strict API or event contracts, messaging semantics, observability, SLO-aware behavior, or controlled rollout and migration planning."
name: "Enterprise Microservices Shared Rules"
applyTo:
  - "**/*.go"
  - "**/*.sql"
  - "**/*.proto"
  - "**/*.yaml"
  - "**/*.yml"
---
# Enterprise Microservices Shared Rules

- Preserve backward compatibility for public APIs, events, and schemas unless explicitly authorized otherwise.
- Treat producer rollout, consumer rollout, and schema changes as one coordinated change set.
- Define retry, timeout, fallback, and observability behavior for external calls and asynchronous processing.
- Preserve trace context and structured telemetry across service boundaries.
