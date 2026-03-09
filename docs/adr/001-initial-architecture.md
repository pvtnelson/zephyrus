# ADR-001: Initial Architecture

## Status
Accepted

## Context
Build a DevOps capstone portfolio project that demonstrates production-grade infrastructure skills to hiring managers and lead engineers. The application must be:

1. **Simple enough** that the app logic doesn't overshadow the infrastructure story
2. **Complex enough** to justify the infrastructure choices (queue, workers, scaling)
3. **Fully automated** — zero manual steps from `git push` to running environment
4. **Secure by default** — non-root containers, no secrets in git, least-privilege
5. **Self-hosted** on existing Proxmox homelab to avoid cloud costs

### Portfolio Goals
- Demonstrate IaC proficiency (Terraform)
- Show CI/CD pipeline design (GitHub Actions)
- Prove container orchestration skills (Kubernetes via k3s)
- Evidence of monitoring/observability thinking
- Clean git history with Conventional Commits and PR-based workflow

### Trade-off: Homelab vs Cloud
Hosting on homelab saves costs but doesn't directly demonstrate cloud provider skills (VPC design, managed services, IAM). To mitigate this:
- Terraform code uses the same patterns as cloud providers (modules, state management, variables)
- Architecture is cloud-portable — all components run in Kubernetes, no homelab-specific coupling
- README documents how to deploy to AKS/EKS/GKE as an alternative
- The infrastructure story (IaC, GitOps, CI/CD, observability) transfers directly to any cloud

## Decision

### Application Layer
**FastAPI** async Python API with:
- `POST /tasks` — submit a task (document conversion, image resize, etc.)
- `GET /tasks/{id}` — check task status and result
- `GET /tasks` — list tasks with filtering
- `GET /health` — readiness/liveness probes
- WebSocket or SSE endpoint for real-time status updates

### Data Layer
- **PostgreSQL 16** — task metadata, status, results (managed via Terraform)
- **Redis 7** — task queue broker + result backend + caching

### Processing Layer
- **Celery** workers consuming from Redis queue
- Pluggable task types (document conversion via `pandoc`, image processing via `Pillow`)
- Configurable concurrency and autoscaling via HPA

### Infrastructure Layer
- **Terraform** with Proxmox provider — provisions LXC/VM for k3s node(s)
- **k3s** — lightweight Kubernetes (single-node for homelab, multi-node ready)
- **Helm charts** or Kustomize for application deployment manifests
- **Terraform state** stored remotely (S3-compatible MinIO on NAS or Terraform Cloud free tier)

### CI/CD Pipeline (GitHub Actions)
1. **Test** — `pytest` with coverage gate
2. **Lint** — `ruff` for Python, `tflint` for Terraform
3. **Build** — Docker multi-stage build (non-root user)
4. **Scan** — Trivy container vulnerability scan
5. **Push** — to GitHub Container Registry (ghcr.io)
6. **Deploy** — `kubectl apply` via kubeconfig secret, or ArgoCD GitOps

### Observability
- **Prometheus** metrics endpoint (`/metrics`) via `prometheus-fastapi-instrumentator`
- **Structured logging** (JSON) via `structlog`
- Kubernetes-native health checks (readiness + liveness probes)

### Security
- Non-root Dockerfile (`USER appuser`)
- GitHub Secrets for all credentials
- Network policies in k3s
- Trivy scan in CI pipeline (fail on HIGH/CRITICAL)
- No secrets in git history — `.gitignore` + pre-commit hook

### Alternatives Considered

| Option | Rejected Because |
|--------|-----------------|
| **Docker Compose only** (no K8s) | Doesn't demonstrate orchestration skills — the primary goal |
| **RabbitMQ** instead of Redis | Redis serves double duty (queue + cache), reducing operational complexity. RabbitMQ adds a component without proportional value for this project |
| **Kafka** for event streaming | Over-engineering for a task queue. Gemini explicitly warned against this |
| **Cloud-hosted (AKS/EKS)** | Monthly costs for a portfolio project; homelab demonstrates self-hosting competence. Terraform patterns remain transferable |
| **ArgoCD** for GitOps | Considered for deploy stage. Good addition later, but not MVP — `kubectl apply` via CI is sufficient to demonstrate the pattern |
| **Istio service mesh** | Over-engineering. k3s built-in Traefik ingress is sufficient |

### 12-Factor Compliance

| Factor | Implementation |
|--------|---------------|
| I. Codebase | Single repo, tracked in git |
| II. Dependencies | `pyproject.toml` with pinned versions |
| III. Config | Environment variables via `.env` / K8s ConfigMaps + Secrets |
| IV. Backing Services | PostgreSQL and Redis as attached resources (connection strings) |
| V. Build, Release, Run | Docker image → tagged release → K8s deployment |
| VI. Processes | Stateless API containers, state in PostgreSQL/Redis |
| VII. Port Binding | FastAPI binds to `$PORT`, exposed via K8s Service |
| VIII. Concurrency | Horizontal scaling via K8s HPA (API pods) + Celery worker count |
| IX. Disposability | Fast startup, graceful shutdown (SIGTERM handling), K8s probes |
| X. Dev/Prod Parity | Docker Compose for local dev, same images in K8s |
| XI. Logs | Structured JSON to stdout, collected by K8s |
| XII. Admin Processes | Database migrations via Alembic, run as K8s Jobs |

## Consequences

### Easier
- Portfolio demonstrates full DevOps lifecycle in one repo
- k3s is lightweight enough for homelab but real Kubernetes
- Redis as dual-purpose (queue + cache) keeps component count low
- FastAPI + Celery is a well-documented, battle-tested pattern

### Harder
- k3s on Proxmox requires Terraform Proxmox provider knowledge (less common than AWS/Azure providers)
- No managed database — must handle PostgreSQL deployment in K8s or as a separate LXC
- Public repo means extra diligence on secrets hygiene

### Risks
- Homelab-only deployment may not fully satisfy recruiters looking for cloud-specific experience
- Mitigation: README includes cloud deployment instructions, Terraform modules are structured for provider swapping
