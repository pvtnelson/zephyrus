# zephyrus — Claude Context

## Project
Async task processing API for DevOps capstone portfolio. FastAPI + PostgreSQL + Redis + background workers. Terraform IaC, GitHub Actions CI/CD, Docker containerized.

## Key Paths
- `src/` — Python application source
- `tests/` — test suite
- `docs/plans/` — approved implementation plans

## Before You Change
- Architecture decisions: read docs/adr/ for context
- Adding dependencies: justify in CHANGELOG.md

## Conventions
- Branch: main (stable), feature/* branches via PRs
- Commits: Conventional Commits (feat, fix, docs, chore)
- Secrets: NEVER committed — use .env (gitignored)
- PRs: never push directly to main
