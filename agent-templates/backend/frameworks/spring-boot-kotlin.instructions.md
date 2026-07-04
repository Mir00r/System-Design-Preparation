---
description: "Use when writing or modifying Spring Boot Kotlin services, REST controllers, service classes, repositories, schedulers, messaging consumers, or Spring integration code in Kotlin."
name: "Spring Boot Kotlin Rules"
applyTo: "**/*.kt"
---
# Spring Boot Kotlin Rules

- Keep `@RestController` classes thin. Put business rules in services or use-case classes.
- Use constructor injection and immutable request or response models where practical.
- Prefer `val` over `var` and avoid `!!` unless an invariant is already guaranteed.
- Validate boundary input with Bean Validation or the repo's standard validation layer.
- Put `@Transactional` on service-level operations, not controllers.
- Do not leak JPA entities directly through public APIs unless the repo already standardizes that pattern.
- Avoid lazy-loading surprises in serialization, logging, and controller return paths.
- Keep coroutine use explicit if the service mixes Spring and suspend APIs. Do not call blocking I/O from suspend code without the repo's approved strategy.
- Prefer focused unit tests for services and focused integration tests for repository queries, web contracts, and serialization-sensitive behavior.
