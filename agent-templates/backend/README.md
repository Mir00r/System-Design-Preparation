# Backend AI Instruction Pack

This folder contains a reusable backend instruction pack for AI-assisted development.

Use it in backend repositories where you want project-root instructions plus language-specific rules.

## Recommended Layout

- Keep exactly one project-wide instruction file at the repo root: `AGENTS.md`
- Add language-specific files under `.github/instructions/`
- Keep only the language files that match the backend stack in that repository

## Why This Structure

- A root `AGENTS.md` is good for rules that apply to every task
- Language-specific `.instructions.md` files are better for Java, Kotlin, Go, and Python because they activate only when matching files are involved
- Multiple root-level `AGENTS.md` files for different languages are not the right mechanism in a single repo

## Files In This Pack

- `AGENTS.md`: shared backend rules for architecture, security, testing, observability, and delivery
- `.github/instructions/backend-shared.instructions.md`: backend rules that should apply to backend code and config files
- `.github/instructions/java-backend.instructions.md`: Java backend rules
- `.github/instructions/kotlin-backend.instructions.md`: Kotlin backend rules
- `.github/instructions/go-backend.instructions.md`: Go backend rules
- `.github/instructions/python-backend.instructions.md`: Python backend rules
- `frameworks/*.instructions.md`: framework-specific templates for Spring Boot, Ktor, FastAPI, Django, Gin, and Echo
- `variants/enterprise-microservices/`: stricter enterprise templates for contract-heavy microservices
- `variants/startup-lean/`: lighter templates for small teams shipping quickly
- `variants/regulated-environments/`: stricter templates for audit, compliance, and data-governance heavy systems
- `starters/`: copy-ready backend starter folders with recommended `.github/instructions/` layouts by stack

## Framework Templates

- `frameworks/spring-boot.instructions.md`
- `frameworks/spring-boot-java.instructions.md`
- `frameworks/spring-boot-kotlin.instructions.md`
- `frameworks/ktor.instructions.md`
- `frameworks/fastapi.instructions.md`
- `frameworks/django.instructions.md`
- `frameworks/gin.instructions.md`
- `frameworks/echo.instructions.md`

Copy only one framework file into a backend repo when that framework is actually in use.
If the stack is Spring Boot, prefer the Java or Kotlin companion file over the generic Spring Boot file.

## Variant Packs

- `variants/enterprise-microservices/`: use when services are independently deployed and contract or rollout safety is a major concern
- `variants/startup-lean/`: use when a small team needs fast iteration with low platform ceremony
- `variants/regulated-environments/`: use when auditability, compliance, retention, authorization, and data governance are hard requirements

## Copy-Ready Starters

- `starters/spring-boot-java/`
- `starters/spring-boot-kotlin/`
- `starters/ktor/`
- `starters/fastapi/`
- `starters/django/`
- `starters/gin/`
- `starters/echo/`

### Composed Starters

- `starters/spring-boot-java-enterprise/`
- `starters/spring-boot-kotlin-enterprise/`
- `starters/ktor-enterprise/`
- `starters/fastapi-regulated/`
- `starters/django-regulated/`
- `starters/gin-enterprise/`
- `starters/echo-enterprise/`

Each starter is self-contained and already includes a root `AGENTS.md` plus a recommended `.github/instructions/` layout.

## Decision Matrix

| Situation | Recommended Starting Point | Why |
| --- | --- | --- |
| Small team, early product, low platform overhead | `variants/startup-lean/` or a plain starter like `starters/fastapi/` | Fast iteration with simple guardrails |
| Standard product backend without special compliance or platform pressure | Default pack or a plain starter like `starters/spring-boot-java/` | Balanced guidance without heavy ceremony |
| Multi-team microservices with contract and rollout risk | `variants/enterprise-microservices/` or composed starters like `starters/gin-enterprise/` | Stronger contract, observability, and rollout rules |
| Audit-heavy, privacy-sensitive, or regulated workloads | `variants/regulated-environments/` or composed starters like `starters/fastapi-regulated/` | Explicit audit, authorization, and data-governance guardrails |
| Spring Boot on Java | `frameworks/spring-boot-java.instructions.md` or `starters/spring-boot-java/` | Language-accurate Spring guidance |
| Spring Boot on Kotlin | `frameworks/spring-boot-kotlin.instructions.md` or `starters/spring-boot-kotlin/` | Kotlin-specific Spring guidance |
| Ktor service | `frameworks/ktor.instructions.md` or `starters/ktor/` | Coroutine and route-specific guidance |
| FastAPI service | `frameworks/fastapi.instructions.md` or `starters/fastapi/` | Async and schema-boundary guidance |
| Django service | `frameworks/django.instructions.md` or `starters/django/` | Queryset, serializer, and Django-boundary guidance |
| Gin or Echo service | `frameworks/gin.instructions.md`, `frameworks/echo.instructions.md`, or the matching Go starter | Thin handler and middleware guidance |

## How To Use In A Backend Repo

1. Copy `AGENTS.md` from this folder to the backend repo root.
2. Copy the relevant `.instructions.md` files into `.github/instructions/` in that backend repo.
3. Remove language files that do not match the repo.
4. Update validation commands to the repo's real build, lint, typecheck, and test commands.
5. Update framework references if the repo is opinionated about Spring Boot, Ktor, FastAPI, Django, Gin, Echo, or another stack.
6. If the repo is highly regulated or strongly microservice-oriented, start from `variants/enterprise-microservices/` instead of the default pack.
7. If the repo is an early-stage product, start from `variants/startup-lean/` instead of the default pack.
8. If the repo has formal audit or compliance obligations, start from `variants/regulated-environments/` instead of the default pack.
9. If you want a fast starting point, copy one folder from `starters/` instead of assembling the files manually.
10. If you want stack-plus-variant combinations already assembled, copy one of the composed starters from `starters/`.

## Important Notes

- Prefer one shared backend `AGENTS.md` per repo, not one per language.
- Keep the rules concise enough that models can follow them reliably.
- Do not duplicate full team docs inside the instruction files; link to repo docs when they exist.
- If the repo is a monolith, do not force microservices.
- If the repo is microservice-based, do not collapse boundaries just to simplify one task.
- Do not activate every framework template at once. Copy only the ones that match the real stack.