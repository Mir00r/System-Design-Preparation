---
description: "Use when writing or modifying Ktor backend services, routes, plugins, coroutine-based handlers, service classes, or Ktor integration code."
name: "Ktor Backend Rules"
applyTo: "**/*.kt"
---
# Ktor Backend Rules

- Keep routing modules thin. Put business logic in services or use-case classes, not directly in route lambdas.
- Preserve structured concurrency. Do not launch unmanaged coroutines from request handlers.
- Do not call blocking I/O directly from suspend request paths unless the repo already isolates it correctly.
- Keep request and response DTOs explicit. Validate input at the boundary and return predictable error payloads.
- Use typed configuration and centralized dependency wiring instead of hidden globals.
- Be explicit about authentication, authorization, and principal extraction.
- When integrating databases or external services, propagate timeouts, cancellation, and trace context.
- Prefer plugin-based cross-cutting concerns for logging, metrics, serialization, auth, and error handling rather than ad hoc duplication.
- Add focused tests for routing, serialization, validation, and coroutine-sensitive behavior.
