---
description: "Use when writing or modifying Django backend services, views, models, serializers, management commands, background jobs, or Django integration code."
name: "Django Backend Rules"
applyTo: "**/*.py"
---
# Django Backend Rules

- Keep views thin.
- Keep serializers responsible for transport validation and representation, not broad domain orchestration.
- Be explicit about queryset loading strategy.
- Use transactions intentionally for multi-write operations.
