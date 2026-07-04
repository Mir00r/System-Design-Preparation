---
description: "Use when writing or modifying Kotlin backend services, Spring Boot Kotlin code, Ktor APIs, coroutines, persistence layers, messaging consumers, or Kotlin tests."
name: "Kotlin Backend Rules"
applyTo:
  - "**/*.kt"
  - "**/*.kts"
---
# Kotlin Backend Rules

- Follow the existing backend framework and build setup. Do not switch between Spring Boot, Ktor, Micronaut, or another stack during normal feature work.
- Prefer `val` over `var`. Favor immutable DTOs and clearly scoped mutation.
- Use null-safety intentionally. Avoid `!!` except where an invariant is already guaranteed and well-justified.
- Prefer data classes for request, response, and value objects when they are plain data carriers.
- Use sealed classes or sealed interfaces for closed result states, error families, and workflow branches when the domain benefits from exhaustiveness.
- Keep controllers or routes thin and move business rules into services or use-case classes.
- Use constructor injection and avoid service locators or hidden static access.
- If the code uses coroutines, keep suspend boundaries honest. Do not call blocking I/O from suspend code without the repo's approved strategy.
- Do not use `runBlocking` in production paths.
- Preserve structured concurrency. Do not launch unmanaged coroutines that outlive their request or job scope.
- Declare explicit return types for public APIs and library-facing code when stability matters.
- Keep extension functions close to the domain they support. Do not create large generic extension dumping grounds.
- Prefer expressive Kotlin, but do not collapse important backend logic into dense chains that hurt debuggability.
- Add focused tests for service logic, coroutine behavior, validation, serialization, and persistence boundaries.
- Use the repo's formatter, detekt, ktlint, Gradle, and test tasks when present rather than introducing new tooling.