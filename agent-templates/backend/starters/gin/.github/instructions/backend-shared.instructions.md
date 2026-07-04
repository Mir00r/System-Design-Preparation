---
description: "Use when implementing backend features, APIs, background jobs, persistence, messaging, or configuration in Go services."
name: "Backend Shared Rules"
applyTo:
  - "**/*.go"
  - "**/*.sql"
  - "**/*.proto"
  - "**/*.yaml"
  - "**/*.yml"
---
# Backend Shared Rules

- Keep handlers thin and move business rules into services or domain packages.
- Validate input at the boundary and return predictable errors.
- Preserve public API behavior unless the task explicitly allows breaking changes.
- Never hardcode secrets.
