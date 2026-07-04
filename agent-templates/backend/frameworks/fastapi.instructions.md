---
description: "Use when writing or modifying FastAPI services, routers, dependencies, Pydantic schemas, async endpoints, or FastAPI integration code."
name: "FastAPI Backend Rules"
applyTo: "**/*.py"
---
# FastAPI Backend Rules

- Keep path operation functions thin. Put business rules in services or use-case modules.
- Use Pydantic models or the repo's schema layer for request and response contracts. Do not leak ORM models as public API shapes unless the codebase already standardizes that pattern.
- Prefer dependency injection through FastAPI dependencies over hidden module globals.
- Keep async endpoints truly async. Do not run blocking I/O in async routes without the repo's approved wrapper or executor strategy.
- Return explicit response models and predictable HTTP status codes.
- Centralize exception handling for validation, domain errors, and integration failures.
- Be explicit about auth dependencies, pagination, filtering, and idempotency for write endpoints.
- Add tests for request validation, response serialization, dependency overrides, and async behavior.
