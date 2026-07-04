---
description: "Use when writing or modifying Echo backend services, HTTP handlers, middleware, repositories, workers, or Echo integration code."
name: "Echo Backend Rules"
applyTo: "**/*.go"
---
# Echo Backend Rules

- Keep Echo handlers thin.
- Propagate request context into downstream operations.
- Use middleware for auth, correlation IDs, recovery, structured logging, tracing, and rate limiting when appropriate.
- Be explicit about background work, retries, and lifecycle if handlers trigger async processing.
