# Regulated Environments Variant

Use this variant when the backend operates in a regulated environment where auditability, compliance controls, data governance, and change traceability are mandatory delivery constraints.

## Copy Into A Backend Repo

1. Copy `AGENTS.md` from this folder to the backend repo root.
2. Copy `backend-shared.instructions.md` into `.github/instructions/`.
3. Add only the language-specific instruction files that match the service stack.
4. Add one framework-specific instruction file if the service is opinionated about Spring Boot, Ktor, FastAPI, Django, Gin, or Echo.

## When To Prefer This Variant

- The system handles regulated, financial, healthcare, government, or privacy-sensitive data.
- Auditability, approvals, retention, and traceability matter.
- Data lineage, access control, and change management are real requirements.
- Production changes must be explainable to auditors, security reviewers, or compliance teams.
