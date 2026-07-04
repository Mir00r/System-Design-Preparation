---
description: "Use when writing or modifying Python backend services, FastAPI or Django endpoints, background jobs, repositories, async code, schemas, or Python tests."
name: "Python Backend Rules"
applyTo: "**/*.py"
---
# Python Backend Rules

- Follow the existing framework and runtime model. Do not switch between FastAPI, Django, Flask, aiohttp, or another stack during routine feature work.
- Keep route handlers, views, and controllers thin. Put business rules in services, use cases, or domain modules.
- Use type hints for public functions, public methods, and important internal boundaries.
- Keep request or response schemas separate from ORM models unless the project intentionally uses the same objects.
- Do not rely on global mutable state for request-specific or job-specific behavior.
- Use dependency injection or the framework's composition patterns instead of hidden module-level state.
- In async code, do not call blocking I/O directly unless the project already wraps it appropriately.
- Keep exception handling narrow. Avoid bare `except:` and overly broad `except Exception:` blocks unless the boundary truly requires them.
- Log failures with enough context to diagnose them, but do not leak secrets or sensitive payloads.
- Use settings and environment-driven configuration for secrets, URLs, feature flags, and environment-specific behavior.
- Keep database access explicit. Avoid hidden N+1 queries, side effects inside serializers, and heavy work inside model property access.
- Prefer small, composable modules over huge files that mix routing, validation, business logic, and persistence.
- Add focused tests with the repo's existing stack, typically `pytest`, and run `ruff`, `mypy`, or the repo's actual lint and typecheck commands when present.