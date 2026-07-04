---
description: "Use when writing or modifying Gin backend services, HTTP handlers, middleware, repositories, workers, or Gin integration code."
name: "Gin Backend Rules"
applyTo: "**/*.go"
---
# Gin Backend Rules

- Keep Gin handlers thin.
- Propagate request context into downstream operations.
- Use middleware for auth, request IDs, logging, recovery, and tracing.
- Do not start background work from handlers without lifecycle and observability rules.
