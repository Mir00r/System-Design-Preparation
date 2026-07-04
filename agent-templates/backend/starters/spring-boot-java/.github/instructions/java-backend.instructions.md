---
description: "Use when writing or modifying Java backend services, persistence layers, controllers, schedulers, messaging consumers, or Java tests."
name: "Java Backend Rules"
applyTo: "**/*.java"
---
# Java Backend Rules

- Use constructor injection only.
- Keep transaction boundaries on service methods, not controllers.
- Do not expose JPA entities directly from public APIs unless the codebase intentionally does that.
- Use Bean Validation or the repo's existing validation approach at API boundaries.
- Prefer focused unit tests for service logic and focused integration tests for queries, web contracts, and serialization.
