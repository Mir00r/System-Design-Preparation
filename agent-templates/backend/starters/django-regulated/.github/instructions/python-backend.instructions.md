---
description: "Use when writing or modifying Python backend services, background jobs, repositories, async code, schemas, or Python tests."
name: "Python Backend Rules"
applyTo: "**/*.py"
---
# Python Backend Rules

- Use type hints on public and important boundary functions.
- Keep exception handling narrow.
- Do not rely on global mutable state for request-specific behavior.
- Keep database access explicit and avoid hidden N+1 patterns.
