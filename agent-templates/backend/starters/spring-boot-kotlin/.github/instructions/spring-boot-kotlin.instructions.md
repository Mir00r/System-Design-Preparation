---
description: "Use when writing or modifying Spring Boot Kotlin services, REST controllers, service classes, repositories, schedulers, messaging consumers, or Spring integration code in Kotlin."
name: "Spring Boot Kotlin Rules"
applyTo: "**/*.kt"
---
# Spring Boot Kotlin Rules

- Keep `@RestController` classes thin.
- Put `@Transactional` on service-level operations.
- Avoid lazy-loading surprises in serialization and logging.
- Keep Spring Data repositories focused on persistence.
- Use Spring configuration and properties for environment-specific behavior.
