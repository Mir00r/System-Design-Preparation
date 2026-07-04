# Copy-Ready Backend Starters

This folder contains self-contained starter packs you can copy into a backend repository.

Each starter includes:

- a root `AGENTS.md`
- a `.github/instructions/` directory
- one shared backend instruction file
- one language-specific instruction file
- one framework-specific instruction file when relevant

## Available Starters

- `spring-boot-java/`
- `spring-boot-kotlin/`
- `ktor/`
- `fastapi/`
- `django/`
- `gin/`
- `echo/`

## Composed Starters

- `spring-boot-java-enterprise/`
- `spring-boot-kotlin-enterprise/`
- `ktor-enterprise/`
- `fastapi-regulated/`
- `django-regulated/`
- `gin-enterprise/`
- `echo-enterprise/`

These starters already combine a stack with either the enterprise microservices rules or the regulated-environment rules.

## How To Use

1. Copy one starter folder into the target backend repository root.
2. Move its `.github/` folder and `AGENTS.md` into the real repo root.
3. Adjust validation commands, package names, and framework-specific conventions if the backend already has local standards.
4. If the backend is highly regulated, start from the regulated variant first and then add the matching stack files.
5. If the backend is a complex microservices platform, start from the enterprise microservices variant first and then add the matching stack files.

## Validation Placeholders

- Java Spring Boot starters include Maven and Gradle placeholders.
- Kotlin Spring Boot and Ktor starters include Gradle placeholders plus detekt or ktlint notes.
- FastAPI and Django starters include `pytest`, Ruff, and typecheck placeholders.
- Gin and Echo starters include `go test`, `go vet`, and GolangCI-Lint placeholders.
