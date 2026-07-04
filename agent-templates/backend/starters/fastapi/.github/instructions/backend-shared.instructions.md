---
description: "Use when implementing backend features, APIs, background jobs, persistence, messaging, or configuration in Python services."
name: "Backend Shared Rules"
applyTo:
  - "**/*.py"
  - "**/*.sql"
  - "**/*.yaml"
  - "**/*.yml"
---
# Backend Shared Rules

- Keep route handlers thin and move business rules into services or use cases.
- Validate input at the boundary and return predictable errors.
- Preserve public API behavior unless the task explicitly allows breaking changes.
- Never hardcode secrets.
