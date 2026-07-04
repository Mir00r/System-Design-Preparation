---
description: "Use when implementing backend features, APIs, background jobs, persistence, messaging, service integrations, infrastructure configuration, or schema changes in Java, Kotlin, Go, or Python services."
name: "Backend Shared Rules"
applyTo:
  - "**/*.java"
  - "**/*.kt"
  - "**/*.kts"
  - "**/*.go"
  - "**/*.py"
  - "**/*.sql"
  - "**/*.proto"
  - "**/*.yaml"
  - "**/*.yml"
  - "**/*.properties"
  - "**/*.toml"
  - "**/*.ini"
  - "**/Dockerfile"
---
# Backend Shared Rules

- Respect the repository's architecture. Do not force a microservice split into a monolith or a monolith into microservices during routine feature work.
- If the architecture choice is unclear and the codebase is greenfield, prefer a modular monolith before introducing service boundaries.
- Keep transport, business, and persistence concerns separated when the service has non-trivial rules.
- Keep controllers, routes, and handlers thin. Put business rules in services, use cases, or domain modules.
- Validate input at the boundary. Return structured, predictable errors.
- Preserve compatibility for public APIs, events, and schemas unless the task explicitly allows breaking changes.
- Make write paths idempotent when retries are possible.
- Treat migrations, schema updates, and data backfills as part of the feature.
- Prefer additive and expand-contract data changes for live systems.
- Keep transaction scope explicit and avoid hidden cross-service or cross-module side effects.
- Never hardcode secrets. Use the repo's configuration and secret-management approach.
- Add structured logging, metrics, and trace propagation where the touched behavior crosses important boundaries.
- Add or update tests for the changed behavior, and run the narrowest relevant validation commands before broad validation.
- Do not add new major frameworks or libraries without approval. Prefer existing project capabilities first.