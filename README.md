# zephyrus

Scalable async task processing API — accepts resource-intensive jobs via a fast REST API, processes them through a queue with background workers, and provides real-time status tracking.

## Architecture

3-tier async task processing system on self-hosted Kubernetes:

```
Client → FastAPI (REST API) → Redis Queue → Celery Workers
                    ↕                                     ↕
               PostgreSQL                            Task Results
```

- **API**: FastAPI — submits tasks, tracks status, serves results
- **Queue**: Redis — task broker + result backend + caching
- **Workers**: Celery — pluggable task types (document conversion, image processing)
- **Database**: PostgreSQL 16 — task metadata and status
- **Orchestration**: k3s on Proxmox (Terraform-provisioned)
- **CI/CD**: GitHub Actions → test → lint → build → Trivy scan → ghcr.io → deploy
- **Observability**: Prometheus metrics, structured JSON logging, K8s health probes

See `docs/adr/001-initial-architecture.md` for full rationale.

## Setup
[Placeholder — filled during implementation]

## Development
[Placeholder — filled during implementation]
