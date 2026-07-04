---
description: "Use when implementing Python backend features in regulated environments that require audit trails, compliance-sensitive data handling, authorization controls, retention awareness, or data-governance safeguards."
name: "Regulated Environment Shared Rules"
applyTo:
  - "**/*.py"
  - "**/*.sql"
  - "**/*.yaml"
  - "**/*.yml"
---
# Regulated Environment Shared Rules

- Preserve auditability for compliance-sensitive create, update, delete, approval, export, and access operations when the product requires it.
- Avoid hidden data flows, undocumented copies of sensitive data, and silent mutation of regulated records.
- Validate authorization separately from authentication.
- Sanitize logs, traces, metrics, and error payloads so sensitive or regulated data is not exposed.
