---
description: "Use when writing or modifying Ktor backend services, routes, plugins, coroutine-based handlers, service classes, or Ktor integration code."
name: "Ktor Backend Rules"
applyTo: "**/*.kt"
---
# Ktor Backend Rules

- Keep routing modules thin.
- Preserve structured concurrency.
- Do not call blocking I/O directly from suspend request paths without the repo's approved strategy.
- Prefer plugin-based cross-cutting concerns for auth, logging, metrics, and error handling.
