---
description: "Use when writing or modifying Echo backend services, HTTP handlers, middleware, repositories, workers, or Echo integration code."
name: "Echo Backend Rules"
applyTo: "**/*.go"
---
# Echo Backend Rules

- Keep Echo handlers thin. Convert HTTP concerns at the edge and move business logic into services or domain packages.
- Extract and propagate request context, deadlines, and cancellation into downstream operations.
- Keep request binding and validation explicit. Do not silently accept partially invalid payloads.
- Use middleware for auth, correlation IDs, recovery, structured logging, tracing, and rate limiting when the project supports them.
- Avoid mixing transport models with persistence models unless the repo intentionally keeps the service very simple.
- Be explicit about background work, retries, and lifecycle if handlers trigger async processing.
- Add focused tests for handlers, middleware, and validation-sensitive behavior.
