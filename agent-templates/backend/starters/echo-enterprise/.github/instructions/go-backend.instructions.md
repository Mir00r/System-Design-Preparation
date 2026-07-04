---
description: "Use when writing or modifying Go backend services, HTTP handlers, repositories, workers, concurrency code, or Go tests."
name: "Go Backend Rules"
applyTo: "**/*.go"
---
# Go Backend Rules

- Accept `context.Context` for request-scoped or I/O-bound work.
- Return errors instead of panicking for normal failure modes.
- Keep interfaces small and define them near consuming code when possible.
- Be careful with goroutines, channels, and shared state.
