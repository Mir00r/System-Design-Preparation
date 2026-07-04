---
description: "Use when writing or modifying Spring Boot Java services, REST controllers, service classes, repositories, schedulers, messaging consumers, or Spring integration code in Java."
name: "Spring Boot Java Rules"
applyTo: "**/*.java"
---
# Spring Boot Java Rules

- Keep `@RestController` classes thin.
- Put `@Transactional` on service-level operations.
- Avoid lazy-loading surprises in serialization and logging.
- Keep Spring Data repositories focused on persistence.
- Use Spring configuration and properties for environment-specific behavior.
