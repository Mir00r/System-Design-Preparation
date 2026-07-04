---
description: "Use when writing or modifying Django backend services, views, models, serializers, management commands, background jobs, or Django integration code."
name: "Django Backend Rules"
applyTo: "**/*.py"
---
# Django Backend Rules

- Follow the repo's architecture. Do not put all business logic into models or views by default.
- Keep views thin. Move domain rules into services, domain modules, or clearly named application-layer functions when the project uses them.
- Be explicit about queryset loading strategy. Avoid hidden N+1 problems in serializers, templates, and admin actions.
- Keep serializers responsible for transport validation and representation, not broad domain orchestration.
- Use transactions intentionally for multi-write operations.
- Keep management commands, Celery tasks, and background jobs idempotent and observable.
- Use Django settings and environment-based configuration for secrets and environment-specific behavior.
- Add tests for model behavior, views or API contracts, permission rules, and query-sensitive code.
