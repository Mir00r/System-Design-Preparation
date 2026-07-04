# Enterprise Microservices Variant

Use this variant when the backend operates as a production microservices platform with strict requirements around contracts, messaging, observability, rollout safety, and operational governance.

## Copy Into A Backend Repo

1. Copy `AGENTS.md` from this folder to the backend repo root.
2. Copy `backend-shared.instructions.md` into `.github/instructions/`.
3. Add only the language-specific instruction files that match the service stack.
4. Add one framework-specific instruction file if the service is opinionated about Spring Boot, Ktor, FastAPI, Django, Gin, or Echo.

## When To Prefer This Variant

- Multiple services are deployed independently.
- API or event compatibility matters across teams.
- Rollouts require canaries, migrations, and operational guardrails.
- SLOs, tracing, incident response, and compliance obligations are real delivery constraints.
