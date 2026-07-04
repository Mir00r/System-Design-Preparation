---
description: "Use when writing or modifying Spring Boot Java services, REST controllers, service classes, repositories, schedulers, messaging consumers, or Spring integration code in Java."
name: "Spring Boot Java Rules"
applyTo: "**/*.java"
---
# Spring Boot Java Rules

- Keep `@RestController` classes thin. Put business rules in services or use-case classes.
- Use constructor injection only. Do not add field injection.
- Keep request, response, and persistence models separate unless the codebase intentionally uses a simpler pattern.
- Validate boundary input with Bean Validation and the repo's standard exception mapping.
- Put `@Transactional` on service-level operations, not controllers.
- Do not leak JPA entities directly through public APIs unless the repo already standardizes that pattern.
- Avoid lazy-loading surprises in serialization, logging, and controller return paths.
- Keep Spring Data repositories focused on persistence. Do not bury business rules in repository interfaces.
- Prefer focused unit tests for services and focused integration tests for repository queries, web contracts, and serialization-sensitive behavior.
