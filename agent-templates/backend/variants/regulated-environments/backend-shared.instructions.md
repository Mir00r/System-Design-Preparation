---
description: "Use when implementing backend features in regulated environments that require audit trails, compliance-sensitive data handling, authorization controls, retention awareness, or data-governance safeguards."
name: "Regulated Environment Shared Rules"
applyTo:
  - "**/*.java"
  - "**/*.kt"
  - "**/*.kts"
  - "**/*.go"
  - "**/*.py"
  - "**/*.sql"
  - "**/*.proto"
  - "**/*.yaml"
  - "**/*.yml"
---
# Regulated Environment Shared Rules

- Preserve auditability for compliance-sensitive create, update, delete, approval, export, and access operations when the product requires it.
- Avoid hidden data flows, undocumented copies of sensitive data, and silent mutation of regulated records.
- Validate authorization separately from authentication and keep least-privilege behavior explicit.
- Sanitize logs, traces, metrics, and error payloads so sensitive or regulated data is not exposed.
- Treat retention behavior, deletion behavior, export behavior, and schema changes as governance-sensitive surfaces.
- Prefer staged and reversible changes when touching compliance-sensitive workflows or data.
- Add tests for authorization, validation, auditability, and migration-sensitive behavior where relevant.
