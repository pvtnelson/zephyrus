# 🚀 Project Zephyrus: Event-Driven Cloud Platform

[![CI/CD Pipeline](https://github.com/pvtnelson/zephyrus/actions/workflows/main.yml/badge.svg)](https://github.com/pvtnelson/zephyrus/actions)
[![Infrastructure: Terraform](https://img.shields.io/badge/IaC-Terraform-623CE4.svg?logo=terraform)](https://www.terraform.io/)
[![Orchestration: Kubernetes](https://img.shields.io/badge/Orchestration-Kubernetes-326CE5.svg?logo=kubernetes)](https://kubernetes.io/)
[![Autoscaling: KEDA](https://img.shields.io/badge/Autoscaling-KEDA-FF6600.svg)](https://keda.sh/)

## 🎯 Project Overview
**Project Zephyrus** is an end-to-end, event-driven processing platform built to demonstrate advanced Cloud-Native and Platform Engineering principles.

Instead of a traditional synchronous API that bottlenecks under heavy load, Zephyrus implements a decoupled, asynchronous architecture. It is designed to ingest high volumes of resource-intensive tasks, queue them efficiently, and scale background workers dynamically based on queue length, rather than relying on sluggish CPU metrics.

This repository serves as a blueprint for **GitOps workflows, Infrastructure as Code (IaC), and resilient distributed systems.**

---

## 🏗️ The Architecture: Decoupled & Scalable

The platform is split into four distinct tiers to ensure high availability and clear separation of concerns:

1. **The Gateway (FastAPI):** A lightweight, stateless REST API. It receives requests, instantly generates a `Job ID`, pushes the payload to the message queue, and returns a `202 Accepted`. The API never hangs.
2. **The Message Broker (RabbitMQ / Redis):** The stateful event hub that holds pending tasks, ensuring zero data loss during traffic spikes.
3. **The Workers (Python Background Processes):** Stateless consumers that listen to the broker, execute the heavy lifting, and can be scaled horizontally from 0 to 100+ instantly.
4. **The Database (PostgreSQL):** The persistent storage layer that tracks the real-time status (`Pending`, `Processing`, `Completed`) of every job.

---

## ✨ Core Engineering Principles Applied

This project wasn't built just to write code; it was built to operate reliably in production.

* **Event-Driven Autoscaling (KEDA):** Standard Kubernetes Horizontal Pod Autoscaler (HPA) reacts too slowly to CPU spikes. Zephyrus uses KEDA to scale worker pods directly based on the *depth of the message queue*, allowing for instant burst-scaling and scaling down to zero to save costs.
* **100% Infrastructure as Code:** The entire environment (VPC, Managed PostgreSQL, Kubernetes Cluster) is provisioned via Terraform. No manual "ClickOps" are required.
* **GitOps CI/CD:** Powered by GitHub Actions. Every push triggers automated linting, testing, multi-stage Docker builds (optimized for size and security), container vulnerability scanning (Trivy), and automated deployment.
* **Security by Design:** Containers run as strictly `non-root` users. Secrets are never hardcoded; they are injected securely via CI/CD pipelines and Kubernetes Secrets.

---

## 🚀 Quick Start: Run it Locally

Developer Experience (DevEx) matters. You can spin up the entire application stack locally without needing a Kubernetes cluster, using the provided Docker Compose configuration.

### Prerequisites
* [Docker Desktop](https://docs.docker.com/get-docker/) or Docker Engine installed.

### Spin up the environment
```bash
# 1. Clone the repository
git clone https://github.com/pvtnelson/zephyrus.git
cd zephyrus

# 2. Start the decoupled stack
docker-compose up --build -d

# 3. Submit a test job to the API
curl -X POST http://localhost:8000/api/v1/jobs -H "Content-Type: application/json" -d '{"task": "process_data", "payload": "sample"}'
```

You can monitor the workers processing the queue via `docker-compose logs -f worker`.

---

## 🧠 Architecture Decision Records (ADRs)

In enterprise environments, *why* a tool was chosen is more important than *what* tool was chosen. I document major technical decisions in the `docs/adrs/` directory.

* **ADR-001:** Why KEDA over Standard HPA for Autoscaling?
* **ADR-002:** Separating Stateful and Stateless workloads in Kubernetes
* **ADR-003:** Choice of RabbitMQ vs. Kafka for Task Queuing

---

Architected and maintained by **Niels van de Steeg** - [Connect with me on LinkedIn](https://www.linkedin.com/in/nielsvandesteeg/)
