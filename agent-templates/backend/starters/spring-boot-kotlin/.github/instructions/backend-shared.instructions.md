---
description: "Use when implementing backend features, APIs, background jobs, persistence, messaging, service integrations, infrastructure configuration, or schema changes in Kotlin services."
name: "Backend Shared Rules"
applyTo:
  - "**/*.kt"
  - "**/*.kts"
  - "**/*.sql"
  - "**/*.proto"
  - "**/*.yaml"
  - "**/*.yml"
  - "**/*.properties"
---
# Backend Shared Rules

- Keep controllers thin and move business rules into services or use cases.
- Validate input at the boundary and return predictable errors.
- Preserve public API compatibility unless the task explicitly allows breakage.
- Treat migrations and schema updates as part of the feature.
- Never hardcode secrets.
- Add tests for changed behavior and validate the narrowest useful slice first.
