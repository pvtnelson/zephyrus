# Plan: Zephyrus MVP — Approved 2026-03-09

## Problem Statement
Build the Zephyrus async task processing API as a DevOps capstone portfolio project. Primary audience is hiring managers evaluating infrastructure skills. Must demonstrate: IaC (Terraform), CI/CD (GitHub Actions), container orchestration (k3s), security (non-root, Trivy), and observability (Prometheus, structured logs).

## Proposed Solution
Build in 8 incremental steps. Each step = feature branch → PR → squash merge to main. Application-first: get the API working locally, then layer infrastructure. README updated incrementally as the primary deliverable.

## Implementation Steps

### Step 1: Project Foundation
- `pyproject.toml` with FastAPI, Celery, Redis, SQLAlchemy, Alembic, structlog, Pydantic, ruff
- `src/` package structure: `api/`, `worker/`, `models/`, `config.py`
- `src/config.py` — Pydantic Settings, all config from env vars (12-factor III)
- `src/models/task.py` — SQLAlchemy model: `Task(id, type, status, payload, result, error, created_at, updated_at)`
- Task statuses: `PENDING → PROCESSING → COMPLETED / FAILED`
- Alembic setup with initial migration (no downgrade — portfolio project, recreate-from-scratch is the rollback)
- `.env.example` with all required variables documented
- `.pre-commit-config.yaml` — ruff, gitleaks (secrets detection)
- `pytest` setup with fixtures for test DB (SQLite for unit tests)
- README.md skeleton: project description, architecture placeholder, setup placeholder

### Step 2: FastAPI Application
- `src/api/main.py` — FastAPI app with lifespan (startup/shutdown)
- `src/api/routes/tasks.py`:
  - `POST /tasks` — validate payload, create task in DB (PENDING), dispatch to Celery, return 202 + task ID
  - `GET /tasks/{id}` — return task status/result from PostgreSQL (404 if not found)
  - `GET /tasks` — list with pagination (`limit`, `offset`) and status filter
- `src/api/routes/health.py`:
  - `GET /health` — liveness (always 200)
  - `GET /ready` — readiness (DB + Redis ping)
- `src/api/schemas.py` — Pydantic request/response models with examples
- `src/api/middleware.py` — request-id middleware (generate UUID, bind to structlog context)
- Unit tests for all endpoints (TestClient with SQLite)
- Update README: API endpoints section

### Step 3: Celery Workers
- `src/worker/celery_app.py` — Celery with Redis broker, Redis result backend (Celery-internal only)
- Task result pattern: task function updates PostgreSQL Task row directly (single source of truth). Redis result backend is for Celery internal state only.
- `src/worker/tasks/`:
  - `echo.py` — echo task (testing/demo)
  - `compute.py` — CPU-bound task (fibonacci, prime factorization) — no file storage needed
  - `convert.py` — markdown-to-HTML conversion (string in, string out)
- Task lifecycle: set status=PROCESSING → execute → set status=COMPLETED with result OR status=FAILED with error
- Graceful shutdown (SIGTERM → finish current task → exit)
- Unit tests for task functions (mock DB session)
- Update README: task types section

### Step 4: Docker & Local Dev
- `Dockerfile` — multi-stage build:
  - Builder stage: install deps
  - Runtime stage: non-root `appuser`, minimal image (python:3.12-slim)
- `docker-compose.yml` — full local stack: API + worker + PostgreSQL + Redis
- `docker-compose.override.yml.example` — dev overrides (volume mounts, reload)
- `.dockerignore` — exclude tests, docs, .git, terraform, k8s
- `Makefile`:
  - `make dev` — docker compose up
  - `make test` — pytest with coverage
  - `make lint` — ruff check + ruff format --check
  - `make build` — docker build
  - `make smoke` — integration test: submit task, poll status, verify completion
  - `make clean` — docker compose down -v
- Health checks in docker-compose for all services
- Update README: local dev setup section

### Step 5: CI/CD Pipeline (CI only)
- `.github/workflows/ci.yml`:
  - Trigger: PR to main
  - Jobs: test (pytest + coverage ≥80% on `src/api/`), lint (ruff + pre-commit), build (Docker), scan (Trivy, fail on HIGH/CRITICAL), push to ghcr.io (only on main merge)
- Branch protection: require CI pass for merge to main
- Update README: CI section with status badge
- Note: deploy workflow created in Step 7 when K8s manifests exist

### Step 6: Terraform Infrastructure
- `terraform/modules/k3s-node/` — reusable module:
  - Proxmox VM provisioned via Proxmox provider
  - cloud-init for k3s bootstrap
  - Variables: CPU, memory, disk, k3s version, network
- `terraform/modules/postgres/` — separate LXC for PostgreSQL:
  - Proxmox LXC with PostgreSQL installed via cloud-init
  - Variables: storage size, pg version, allowed networks
  - Output: connection string
- `terraform/main.tf` — orchestrates modules
- `terraform/backend.tf` — MinIO S3 backend on NAS (10.10.10.10)
- `scripts/bootstrap-tfstate.sh` — creates MinIO bucket for state (run once before `terraform init`)
- `terraform/variables.tf`, `outputs.tf` (kubeconfig, DB connection string, node IPs)
- `terraform/versions.tf` — provider version constraints
- `terraform/terraform.tfvars.example` — documented variable defaults
- `terraform/README.md` — diagram of what gets provisioned (1 VM for k3s, 1 LXC for PostgreSQL)
- Add to Makefile: `make infra-plan`, `make infra-apply`, `make infra-destroy` (with confirmation prompt)
- Update README: infrastructure section

### Step 7: Kubernetes Manifests + Deploy Workflow
- `k8s/base/` (Kustomize):
  - `namespace.yaml` — `zephyrus` namespace
  - `api-deployment.yaml` — FastAPI pods, readiness (`/ready`) + liveness (`/health`) probes, init container running `alembic upgrade head`
  - `api-service.yaml` — ClusterIP service
  - `worker-deployment.yaml` — Celery worker pods
  - `redis-deployment.yaml` — Redis with PVC persistence
  - `configmap.yaml` — non-secret config
  - `secret.yaml` — DB connection string, Redis URL (encrypted via SOPS with age key)
  - `ingress.yaml` — Traefik IngressRoute
  - `network-policy.yaml` — restrict traffic: API and workers can reach PostgreSQL (external LXC), workers can reach Redis, API can reach Redis
  - `kustomization.yaml`
- `k8s/overlays/production/`:
  - Replica count patches, resource limits, HPA for API pods
  - `kustomization.yaml`
- `.github/workflows/deploy.yml`:
  - Trigger: push to main, path filter `k8s/**`
  - Job: `kubectl apply -k k8s/overlays/production/` via kubeconfig secret
- `scripts/bootstrap-sops.sh` — generate age key, document usage
- Note in manifests: "PostgreSQL runs external to the cluster (Terraform-provisioned LXC). In cloud deployments, replace with managed database service."
- Add to Makefile: `make deploy`, `make undeploy` (with confirmation prompt)
- Update README: deployment section

### Step 8: Observability & Final Documentation
- Prometheus metrics via `prometheus-fastapi-instrumentator` (add to existing `src/api/middleware.py`)
- `/metrics` endpoint for Prometheus scraping
- Structured logging: `structlog` configured in `src/config.py` (JSON to stdout)
- Custom metrics: `tasks_submitted_total`, `tasks_completed_total`, `task_processing_seconds`
- Request-id correlation already in place from Step 2
- Final README.md:
  - Mermaid architecture diagram
  - Complete local dev instructions (`make dev`)
  - Cloud deployment guide (AKS alternative with Terraform module swap)
  - "Design Trade-offs" section linking to ADRs
  - CI badge
- Update CHANGELOG.md

## Files Affected
- `src/` — all application code (new)
- `tests/` — all test files (new)
- `terraform/` — IaC with modules (new)
- `k8s/` — Kustomize base + overlays (new)
- `.github/workflows/` — CI/CD pipelines (new)
- `Dockerfile`, `docker-compose.yml`, `Makefile` — container/dev setup (new)
- `.pre-commit-config.yaml` — linting and secrets detection (new)
- `.env.example` — documented env vars (new)
- `scripts/` — bootstrap scripts (new)
- `README.md` — updated incrementally each step
- `CHANGELOG.md` — updated in Step 8

## Testing Strategy
- **Unit tests**: API endpoints (TestClient + SQLite), task functions (mocked DB), config, schemas
- **Integration tests**: `make smoke` — docker-compose full stack, submit task, poll status, verify
- **Infrastructure tests**: `terraform validate`, `terraform plan` (dry run in CI)
- **Security tests**: Trivy scan in CI, gitleaks pre-commit hook, non-root container verification
- **Coverage**: ≥80% on `src/api/` routes and schemas; Celery tasks tested with mocked DB

## Rollback Plan
- Each step is a squash-merged PR — `git revert` any merge commit
- Terraform: `make infra-destroy` tears down infrastructure (with confirmation)
- K8s: `make undeploy` removes all app resources
- Docker images: tagged by commit SHA, previous versions in ghcr.io
- Database: recreate from scratch (portfolio project — no production data to preserve)

---

## Staff Engineer Review

### Round 1 — APPROVED WITH CHANGES
Fixed: Celery dual-write, WebSocket scope cut, PR workflow, Terraform state backend, .env.example, Makefile, pre-commit config, PostgreSQL as separate LXC, CPU-bound tasks (no file storage), migration init container, README as primary deliverable.

### Round 2 — APPROVED WITH CHANGES
Fixed: SOPS with age for secrets (no controller dependency), deploy workflow moved to Step 7 (avoids failed runs), network policy allows worker-to-PostgreSQL, honest rollback plan (recreate-from-scratch), "Design Trade-offs" framing instead of "How to evaluate".

Remaining suggestions incorporated: `make smoke` target, `terraform/README.md`, request-id correlation from Step 2.
