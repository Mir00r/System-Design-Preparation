---
description: "Use when writing or modifying Kotlin backend services, coroutines, persistence layers, controllers, messaging consumers, or Kotlin tests."
name: "Kotlin Backend Rules"
applyTo:
  - "**/*.kt"
  - "**/*.kts"
---
# Kotlin Backend Rules

- Prefer `val` over `var`.
- Avoid `!!` unless an invariant is already guaranteed.
- Keep suspend boundaries honest.
- Keep extension functions close to the domain they support.
