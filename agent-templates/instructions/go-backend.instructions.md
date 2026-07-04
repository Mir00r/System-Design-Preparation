---
description: "Use when writing or modifying Go backend services, HTTP handlers, gRPC services, repositories, workers, concurrency code, or Go tests."
name: "Go Backend Rules"
applyTo: "**/*.go"
---
# Go Backend Rules

- Follow idiomatic Go. Prefer standard library patterns and existing repo conventions over Java-style or Python-style abstractions.
- Keep packages focused and cohesive. Do not create generic `util`, `helper`, or `common` packages without a real boundary.
- Accept `context.Context` as the first parameter for request-scoped, I/O-bound, or cancelable operations.
- Honor context cancellation, deadlines, and timeouts across database calls, HTTP clients, gRPC calls, and background work.
- Return errors instead of panicking for normal failure modes.
- Wrap errors with context using `%w` and preserve the original cause for callers.
- Keep interfaces small and define them near the consuming code when possible.
- Prefer concrete types until an interface is actually needed.
- Be careful with goroutines, channels, and shared state. Do not introduce goroutine leaks, blocked sends, blocked receives, or data races.
- Use concurrency only when it improves latency, throughput, or isolation in a measurable way.
- Keep HTTP handlers thin. Move business logic into services or domain packages.
- Keep database and transport models explicit where translation is needed. Do not let persistence concerns leak everywhere.
- Prefer table-driven tests for behavior with multiple cases.
- Use `gofmt` and the repo's linting setup. If available, validate with `go test ./...`, `go vet ./...`, and `golangci-lint run` or narrower package scopes.