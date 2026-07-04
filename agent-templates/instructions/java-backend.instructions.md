---
description: "Use when writing or modifying Java backend services, Spring Boot code, Java APIs, persistence layers, controllers, schedulers, messaging consumers, or Java tests."
name: "Java Backend Rules"
applyTo: "**/*.java"
---
# Java Backend Rules

- Follow the existing stack first. If the repo uses Spring Boot, Spring Data, Micronaut, Quarkus, or plain Java, stay inside those patterns.
- Prefer package-by-feature or bounded context over package-by-technical-layer when the repo allows it.
- Use constructor injection. Do not add field injection.
- Keep controllers thin. Map requests to services or use cases, not directly to repositories.
- Keep transaction boundaries on service methods or explicit application-layer units of work, not scattered across controllers.
- Do not expose JPA entities or persistence objects directly from public API layers unless the codebase already does this intentionally.
- Use immutable DTOs where practical. If the project uses modern Java features, prefer records for simple request and response models.
- Use Bean Validation annotations or the repo's validation mechanism at API boundaries.
- Avoid lazy-loading surprises in serialization paths, controller return values, and logging.
- Keep repository methods explicit. Do not hide expensive joins, full scans, or N+1 access patterns behind innocent-looking calls.
- Use checked versus unchecked exceptions consistently with the existing codebase. Never swallow exceptions silently.
- Prefer `Optional` only for return types where absence is legitimate. Do not use it for fields, ORM entities, or method parameters unless the existing codebase already standardizes that pattern.
- Add focused tests for service logic and integration tests for repository queries, controller validation, and serialization-sensitive code.
- If the repo uses Maven, prefer `./mvnw test`, `./mvnw verify`, or narrower module commands. If it uses Gradle, prefer `./gradlew test`, `./gradlew check`, or narrower module tasks.