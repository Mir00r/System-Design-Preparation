---
description: "Use when writing or modifying Gin backend services, HTTP handlers, middleware, repositories, workers, or Gin integration code."
name: "Gin Backend Rules"
applyTo: "**/*.go"
---
# Gin Backend Rules

- Keep Gin handlers thin. Parse input, call service logic, map outputs, and return responses without embedding domain rules in the handler.
- Propagate `context.Context` from `gin.Context.Request.Context()` into downstream operations.
- Keep binding, validation, and response contracts explicit. Return consistent error bodies.
- Use middleware for cross-cutting concerns like auth, request IDs, logging, recovery, and tracing instead of duplicating handler logic.
- Avoid package sprawl. Keep routers, handlers, services, and repositories coherent and easy to test.
- Keep goroutine usage deliberate in request paths. Do not start background work from handlers without lifecycle and observability rules.
- Add focused tests for handlers, middleware, validation, and service behavior.
