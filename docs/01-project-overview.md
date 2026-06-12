# 01 — Project Overview

> **Navigation:** [← Documentation Hub](index.md) | **Next →** [Local Development](02-local-development.md)

---

## What Is This Project?

The **Golang EKS GitOps Pipeline** is a complete, real-world DevOps reference implementation. It takes a simple Go web application and wraps it in a fully automated pipeline that handles everything from code quality checks to production deployment on Kubernetes — with zero manual steps after setup.

The application itself is deliberately simple: a multi-page static website served using Go's built-in `net/http` package. This keeps the focus where it belongs — on the **DevOps infrastructure** that surrounds it.

---

## Why Was It Built?

Shipping software manually is slow, error-prone, and doesn't scale. This project was built to demonstrate practical solutions to the most common real-world DevOps problems:

| Problem | Solution Used |
|---|---|
| "Works on my machine" inconsistency | Docker multi-stage builds with distroless runtime |
| Slow, manual deployments | 4-job GitHub Actions pipeline triggered on every push |
| No audit trail for deployments | Git as the single source of truth (GitOps) |
| Configuration drift in the cluster | Argo CD continuously reconciles cluster state with Git |
| Large, insecure container images | `gcr.io/distroless/base` — no shell, no package manager |
| Code quality regressions | `golangci-lint` runs on every PR and push |

---

## The Application

The Go web application serves four static HTML pages over HTTP:

| Route | Page |
|---|---|
| `/home` | Landing page |
| `/courses` | Courses listing |
| `/about` | About page |
| `/contact` | Contact page |

The server listens on **port 8080** and uses only Go's standard library — no external web frameworks. This makes it easy to understand the code while focusing on the DevOps layer.

**Source files:**
- [`main.go`](../main.go) — HTTP server with 4 route handlers
- [`main_test.go`](../main_test.go) — Unit tests for handlers
- [`static/`](../static/) — HTML pages and images

---

## How It All Fits Together

Here is a high-level view of the three interconnected layers of this project:

### Layer 1 — The Go Application
A minimal web server running on port 8080, containerized into a tiny, secure Docker image (~20 MB) using a multi-stage build.

### Layer 2 — The CI/CD Pipeline (GitHub Actions)
Every push to `main` triggers an automated 4-job pipeline:

```
[build + test] ──► [push Docker image] ──► [update Helm chart tag]
[code quality]     (runs in parallel with build)
```

The final step automatically commits the new image tag back to the Git repository, which signals Argo CD to deploy.

### Layer 3 — The GitOps Deployment (Argo CD on EKS)
Argo CD runs inside the Amazon EKS cluster. It continuously watches the Git repository. When it detects the new image tag in the Helm chart's `values.yaml`, it automatically syncs the cluster — pulling the new image and rolling out the updated deployment.

**The key principle:** The Git repository is the **single source of truth**. The cluster always matches what Git says it should be.

---

## Repository Structure at a Glance

```
go-web-app-devops/
├── main.go                    # Go application
├── main_test.go               # Unit tests
├── Dockerfile                 # Multi-stage Docker build
├── helm/go-web-app-chart/     # Helm chart (used by Argo CD)
├── k8s/manifests/             # Raw Kubernetes manifests (alternative)
├── .github/workflows/cicd.yaml # GitHub Actions pipeline
├── eks/                       # EKS setup guides
├── gitops/argocd/             # Argo CD install guide
├── ingress-controller/nginx/  # NGINX Ingress guide
└── docs/                      # ← You are here
```

---

## Where to Go Next

| Goal | Document |
|---|---|
| Run the app on your machine | [02 — Local Development](02-local-development.md) |
| Build the Docker image | [03 — Docker Guide](03-docker-guide.md) |
| Set up AWS tools | [04 — AWS Prerequisites](04-aws-prerequisites.md) |
| View architecture diagrams | [Architecture Diagrams](architecture-diagrams.md) |

---

*[← Documentation Hub](index.md) | [Next: Local Development →](02-local-development.md)*
