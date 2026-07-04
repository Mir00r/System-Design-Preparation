---
description: "Use when writing or modifying Spring Boot backend services, REST controllers, service classes, repositories, schedulers, event handlers, or Spring integration code."
name: "Spring Boot Backend Rules"
applyTo:
  - "**/*.java"
  - "**/*.kt"
---
# Spring Boot Backend Rules

- Follow the existing package structure and dependency injection approach. Prefer constructor injection only.
- Keep `@RestController` classes thin. Move business logic into services or use-case classes.
- Keep request, response, and persistence models separate unless the codebase intentionally uses a simpler pattern.
- Validate boundary input with Bean Validation and return consistent error payloads through the repo's exception handling approach.
- Be explicit about transaction boundaries. Put `@Transactional` on service-level operations, not scattered across controllers.
- Do not rely on lazy JPA relations in serialization paths. Avoid accidental N+1 queries.
- Keep Spring Data repositories focused on persistence. Do not move business rules into repository interfaces or query annotations.
- Use configuration properties or the repo's settings pattern for externalized config. Do not hardcode environment-specific values.
- For async or scheduled work, define retry, idempotency, timeout, and observability behavior explicitly.
- Prefer framework-supported security, validation, serialization, and resilience features before adding new libraries.
- Add tests at the right slice: unit tests for business rules, repository tests for query behavior, and focused web tests or integration tests for controller contracts.
