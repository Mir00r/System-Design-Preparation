# Startup Lean Variant

Use this variant when the backend is early-stage and the goal is to ship useful features quickly without creating avoidable platform debt.

## Copy Into A Backend Repo

1. Copy `AGENTS.md` from this folder to the backend repo root.
2. Copy `backend-shared.instructions.md` into `.github/instructions/`.
3. Add only the language-specific instruction files that match the stack.
4. Add one framework-specific instruction file if the repo is opinionated about a specific backend framework.

## When To Prefer This Variant

- The team is small.
- The product is early-stage.
- Speed of iteration matters more than enterprise process.
- Operational complexity should stay deliberately low.
