---
description: "Use when implementing backend features for a small or early-stage product where delivery speed matters, but code still needs to remain maintainable, secure, and testable."
name: "Startup Lean Shared Rules"
applyTo:
  - "**/*.java"
  - "**/*.kt"
  - "**/*.kts"
  - "**/*.go"
  - "**/*.py"
  - "**/*.sql"
  - "**/*.yaml"
  - "**/*.yml"
---
# Startup Lean Shared Rules

- Prefer the simplest architecture that fits the current product. Default to a monolith or modular monolith.
- Keep controllers, routes, and handlers thin. Put business rules in services or feature modules.
- Avoid speculative abstractions and infrastructure-heavy designs.
- Preserve external behavior when practical, but do not add enterprise-grade process where the product does not need it yet.
- Treat migrations and schema changes as part of the feature, but keep the rollout plan proportionate to the actual risk.
- Add structured logging, basic metrics, and clear error handling for any behavior that would be hard to debug in production.
- Add tests for risky or business-critical behavior and validate the changed slice before broader checks.
- Do not add new major frameworks, brokers, or platform layers without a clear delivery benefit.
