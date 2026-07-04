---
description: "Use when writing or modifying FastAPI services, routers, dependencies, Pydantic schemas, async endpoints, or FastAPI integration code."
name: "FastAPI Backend Rules"
applyTo: "**/*.py"
---
# FastAPI Backend Rules

- Keep path operation functions thin.
- Use explicit request and response schemas.
- Prefer FastAPI dependency injection over hidden module globals.
- Keep async endpoints truly async.
