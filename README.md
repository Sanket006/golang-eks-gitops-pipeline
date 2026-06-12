# 🚀 Golang EKS GitOps Pipeline

> A lightweight, production-ready Go web application — containerized with Docker, deployed on **Amazon EKS**, and delivered via a fully automated **CI/CD pipeline** using GitHub Actions and Argo CD (GitOps).

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)
[![Go Version](https://img.shields.io/badge/Go-1.22-00ADD8?logo=go)](https://go.dev/dl/)
[![Docker](https://img.shields.io/badge/Docker-Ready-2496ED?logo=docker)](Dockerfile)
[![GitHub Actions](https://img.shields.io/badge/CI%2FCD-GitHub%20Actions-2088FF?logo=githubactions)](https://github.com/features/actions)
[![Argo CD](https://img.shields.io/badge/GitOps-Argo%20CD-EF7B4D?logo=argo)](https://argo-cd.readthedocs.io/)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-Amazon%20EKS-326CE5?logo=kubernetes)](https://aws.amazon.com/eks/)

---

## 📑 Table of Contents

- [Project Overview](#-project-overview)
- [Problem Statement](#-problem-statement)
- [Architecture Summary](#-architecture-summary)
- [Tool Stack](#-tool-stack)
- [Implementation Steps](#-implementation-steps)
- [Application Routes](#-application-routes)
- [End-to-End Workflow](#-end-to-end-workflow)
- [Project Structure](#-project-structure)
- [Screenshots](#-screenshots)
- [What I Learned](#-what-i-learned)
- [Related Documentation](#-related-documentation)
- [Conclusion](#-conclusion)
- [License](#-license)

---

## 📖 Project Overview

This project demonstrates a **complete, real-world DevOps pipeline** built around a simple Go web application. The application itself is intentionally minimal — a multi-page static website served with Go's built-in `net/http` package — so the focus remains entirely on the DevOps practices and infrastructure surrounding it.

**The core idea:** Write code → push to Git → everything else happens automatically.

From a single `git push`, the pipeline automatically:
1. Builds and tests the Go application
2. Runs static code analysis
3. Builds a secure, minimal Docker image
4. Pushes the image to Docker Hub
5. Updates the Kubernetes Helm chart
6. Deploys the new version to Amazon EKS via Argo CD

This project is ideal for anyone looking to understand how modern DevOps teams ship software to production using **containerization**, **GitOps**, and **managed Kubernetes**.

---

## 🧩 Problem Statement

Deploying applications manually is slow, error-prone, and doesn't scale. Teams face common pain points:

- **No consistency** — "it works on my machine" is a real problem without containerization.
- **Slow releases** — manual builds, tests, and deployments slow down delivery.
- **No audit trail** — it's hard to know what version is running in production and who deployed it.
- **Deployment drift** — the actual cluster state gradually diverges from what's defined in configuration files.
- **Security risks** — large container images with unnecessary tools increase the attack surface.

**This project addresses all of these** by building an end-to-end automated pipeline with:

| Problem | Solution |
|---|---|
| Environment inconsistency | Docker multi-stage builds with a distroless base image |
| Manual, slow deployments | GitHub Actions CI/CD pipeline triggered on every push |
| No audit trail | Git as the single source of truth (GitOps with Argo CD) |
| Configuration drift | Argo CD continuously reconciles cluster state with Git |
| Large, insecure images | `gcr.io/distroless/base` runtime image — no shell, no package manager |

---

## 🏗️ Architecture Summary

The system is composed of three layers that work together seamlessly.

### Layer 1 — AWS Cloud Infrastructure

The application runs inside a **private subnet** within an AWS VPC on Amazon EKS (managed Kubernetes). An **AWS Network Load Balancer (NLB)** in the public subnet serves as the entry point, routing external traffic into the cluster. A **NAT Gateway** enables outbound access (e.g., pulling images from Docker Hub) from the private nodes.

### Layer 2 — CI/CD Pipeline (GitHub Actions)

Every push to `main` triggers a **4-job GitHub Actions pipeline**:

```
build ──────────────────────────────────────────► push ──► update-newtag-in-helm-chart
code-quality (runs in parallel with build)
```

The final job automatically commits the new Docker image tag back to the Git repository — which triggers Argo CD.

### Layer 3 — GitOps Deployment (Argo CD on EKS)

**Argo CD** runs inside the EKS cluster and continuously watches the Git repository. When it detects a new image tag in `helm/go-web-app-chart/values.yaml`, it automatically syncs the cluster to match — pulling the new image and rolling out an updated deployment.

> 📐 For detailed Mermaid architecture diagrams covering AWS infrastructure, the CI/CD workflow, and Kubernetes namespace topology, see [docs/architecture-diagrams.md](docs/architecture-diagrams.md).

---

## 🛠️ Tool Stack

| Category | Tool / Technology | Purpose |
|---|---|---|
| **Language** | Go (Golang) 1.22 | Application runtime |
| **Containerization** | Docker (multi-stage build) | Package app into a portable image |
| **Base Image** | `gcr.io/distroless/base` | Minimal, secure production runtime |
| **Container Registry** | Docker Hub | Store and version Docker images |
| **CI Pipeline** | GitHub Actions | Automate build, test, lint, and push |
| **Code Quality** | `golangci-lint` v1.56.2 | Static analysis and linting |
| **CD / GitOps** | Argo CD | Declarative, Git-driven deployments |
| **Kubernetes** | Amazon EKS (Managed Nodes) | Managed Kubernetes cluster on AWS |
| **Package Manager** | Helm | Kubernetes manifest templating |
| **Ingress** | NGINX Ingress Controller | Route external HTTP traffic into the cluster |
| **DNS (local)** | `/etc/hosts` mapping | Resolve `go-web-app.local` for testing |
| **Cloud Provider** | AWS (VPC, NLB, NAT, EKS) | Cloud infrastructure |

---

## 📋 Implementation Steps

Follow these steps in order to reproduce this project from scratch.

### Step 1 — Build the Go Application

The application uses Go's standard `net/http` library. No external frameworks are needed.

```bash
# Clone the repository
git clone https://github.com/<your-username>/go-web-app-devops.git
cd go-web-app-devops

# Run locally
go run main.go

# Run unit tests
go test ./...
go test -v ./...
```

The server starts on **port 8080**. Navigate to `http://localhost:8080/home` to verify.

---

### Step 2 — Containerize with Docker

The `Dockerfile` uses a **multi-stage build** for a minimal, secure image:

- **Stage 1 (`base`)**: `golang:1.21` image compiles the Go binary.
- **Stage 2 (final)**: `gcr.io/distroless/base` copies only the compiled binary and static files.

```bash
# Build the image
docker build -t <your-dockerhub-username>/go-web-app .

# Run the container locally
docker run -p 8080:8080 <your-dockerhub-username>/go-web-app

# Push to Docker Hub
docker push <your-dockerhub-username>/go-web-app
```

> The final image contains **no shell, no package manager, and no OS utilities** — only the Go binary and static files.

---

### Step 3 — Set Up Amazon EKS

Install prerequisites and create the EKS cluster. See the full guides:

- [docs/04-aws-prerequisites.md](docs/04-aws-prerequisites.md) — Install `kubectl`, `eksctl`, and AWS CLI
- [docs/05-eks-cluster-setup.md](docs/05-eks-cluster-setup.md) — Create the managed EKS cluster

```bash
# Example: Create a cluster with eksctl
eksctl create cluster --name demo-cluster --region us-east-1 --nodegroup-name demo-ng --node-type t3.medium --nodes 2
```

---

### Step 4 — Deploy the NGINX Ingress Controller

The NGINX Ingress Controller provisions an AWS NLB and handles routing rules inside the cluster.

```bash
# Install via Helm
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm install ingress-nginx ingress-nginx/ingress-nginx --namespace ingress-nginx --create-namespace
```

See the full guide: [docs/06-nginx-ingress-controller.md](docs/06-nginx-ingress-controller.md)

---

### Step 5 — Install Argo CD

Deploy Argo CD into the cluster and expose its UI.

```bash
# Create namespace and install Argo CD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

See the full guide: [docs/07-argocd-setup.md](docs/07-argocd-setup.md)

---

### Step 6 — Configure GitHub Actions Secrets

In your GitHub repository, add the following **Actions Secrets** under `Settings → Secrets and variables → Actions`:

| Secret Name | Description |
|---|---|
| `DOCKERHUB_USERNAME` | Your Docker Hub username |
| `DOCKERHUB_TOKEN` | Docker Hub access token (not your password) |
| `TOKEN` | GitHub Personal Access Token (PAT) with `repo` scope — used to push tag updates back to the repo |

---

### Step 7 — Connect Argo CD to Your Repository

Create an Argo CD `Application` resource pointing to your Git repo and Helm chart:

```yaml
# gitops/argocd/application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: go-web-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<your-username>/go-web-app-devops
    targetRevision: main
    path: helm/go-web-app-chart
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

### Step 8 — Trigger the Full Pipeline

Push any code change to the `main` branch (not in `helm/`, `k8s/`, or `README.md`):

```bash
git add .
git commit -m "feat: update application"
git push origin main
```

Watch the pipeline run in the **Actions** tab on GitHub. Argo CD will sync automatically when the Helm chart tag is updated.

---

## 🌐 Application Routes

The server exposes four HTTP routes on **port 8080**:

| Route | Static File | Description |
|---|---|---|
| `/home` | `static/home.html` | Home / Landing page |
| `/courses` | `static/courses.html` | Courses listing page |
| `/about` | `static/about.html` | About page |
| `/contact` | `static/contact.html` | Contact page |

---

## 🔄 End-to-End Workflow

Here is the complete journey of a code change from a developer's machine to production:

```
Developer
    │
    │  git push → main branch
    ▼
GitHub Repository
    │
    │  Triggers CI/CD workflow
    ▼
GitHub Actions Pipeline
    ├── Job 1: build        →  go build + go test ./...
    ├── Job 2: code-quality →  golangci-lint (runs in parallel with build)
    ├── Job 3: push         →  Docker build + push to Docker Hub (tagged with run ID)
    └── Job 4: update-tag   →  sed image tag in helm/values.yaml + git commit + git push
    │
    │  New commit detected in Git
    ▼
Argo CD (running in EKS cluster)
    │
    │  Polls Git, detects tag change in values.yaml
    │  Pulls updated Helm chart
    │  Applies manifests to EKS cluster
    ▼
Amazon EKS Cluster
    ├── NGINX Ingress Controller  →  Routes go-web-app.local → app Service
    ├── Go Web App Deployment     →  Runs updated Pod(s) with new image
    └── Go Web App Service        →  ClusterIP routing to Pods
    │
    ▼
End User
    Accesses http://go-web-app.local → served by the new deployment ✅
```

**Key GitOps principle:** The Helm chart in Git is the **single source of truth**. Argo CD ensures the cluster always matches what is in Git — any manual changes to the cluster are automatically reverted.

---

## 📁 Project Structure

```
go-web-app-devops/
│
├── main.go                    # Application entry point — HTTP server with 4 routes
├── main_test.go               # Unit tests for all HTTP handlers
├── go.mod                     # Go module definition
├── Dockerfile                 # Multi-stage Docker build (golang:1.21 → distroless)
├── LICENSE                    # Apache License 2.0
├── README.md                  # This file
│
├── static/                    # Frontend static HTML pages and assets
│   ├── home.html
│   ├── courses.html
│   ├── about.html
│   ├── contact.html
│   └── images/
│       └── golang-website.png
│
├── helm/                      # Helm chart for Kubernetes deployment (used by Argo CD)
│   └── go-web-app-chart/
│       ├── Chart.yaml         # Chart metadata
│       ├── values.yaml        # Image tag — auto-updated by CI/CD pipeline
│       └── templates/
│           ├── deployment.yaml
│           ├── service.yaml
│           └── ingress.yaml
│
├── k8s/                       # Raw Kubernetes manifests (manual deploy alternative)
│   └── manifests/
│       ├── deployment.yaml
│       ├── service.yaml
│       └── ingress.yaml
│
├── docs/                      # Full documentation suite (12 files)
│   ├── index.md               # Documentation hub — start here
│   ├── 01-project-overview.md
│   ├── 02-local-development.md
│   ├── 03-docker-guide.md
│   ├── 04-aws-prerequisites.md
│   ├── 05-eks-cluster-setup.md
│   ├── 06-nginx-ingress-controller.md
│   ├── 07-argocd-setup.md
│   ├── 08-github-actions-cicd.md
│   ├── 09-helm-chart-guide.md
│   ├── 10-full-deployment-walkthrough.md
│   └── architecture-diagrams.md
│
└── .github/
    └── workflows/
        └── cicd.yaml          # GitHub Actions CI/CD pipeline (4 jobs)
```

---

## 📸 Screenshots

### Application Home Page

![Go Web App Home Page](static/images/golang-website.png)

### CI/CD Pipeline Architecture

> The diagram below illustrates the complete DevOps pipeline — from a code commit to production deployment on EKS.

![Pipeline Architecture](https://github.com/user-attachments/assets/45f4ef12-c5b5-4247-9d43-356b5dfb671b)

> 📐 For detailed sub-system diagrams (AWS Cloud, GitOps CI/CD Workflow, Kubernetes Namespace Topology), see [docs/architecture-diagrams.md](docs/architecture-diagrams.md).

---

## 💡 What I Learned

Here are the key things I picked up while building this project:

1. **Git can do more than store code** — I learned that Git can also be the source of truth for what is running in production. When Argo CD watches the repo, updating a file in Git is enough to trigger a real deployment. No manual commands needed.

2. **Docker images can be very small and safe** — Instead of shipping a big image with all the build tools inside, I learned to use a two-stage build. The final image only contains the running app — nothing extra that could be a security risk.

3. **CI/CD pipelines save a lot of time** — I set up GitHub Actions to automatically build, test, and deploy the app on every code push. This removed all the repetitive manual steps and made the whole process faster and less error-prone.

4. **Helm makes Kubernetes easier to manage** — Instead of editing multiple YAML files by hand, Helm lets you change one value (like the image version) and it updates everything automatically.

5. **Cloud infrastructure has layers** — Setting up EKS taught me how traffic flows from the internet to a container: through a load balancer, into the cluster, past routing rules, and finally to the app.

6. **One load balancer can handle many services** — The NGINX Ingress Controller acts like a traffic director. It receives all incoming requests and routes them to the right service based on simple rules.

7. **Automation removes human mistakes** — Every step I automated is a step that can no longer go wrong due to a typo or a forgotten command. The less manual work, the more reliable the system.


---

## 🔗 Related Documentation

> 📚 **Full Documentation Hub:** [`docs/index.md`](docs/index.md) — Start here for structured, step-by-step guides covering every aspect of this project.

Follow these guides in order to set up the complete infrastructure from scratch:

| Step | Topic | Guide |
|---|---|---|
| — | **Documentation Hub (Start Here)** | [docs/index.md](docs/index.md) |
| 1 | Project Overview | [docs/01-project-overview.md](docs/01-project-overview.md) |
| 2 | Local Development | [docs/02-local-development.md](docs/02-local-development.md) |
| 3 | Docker Guide | [docs/03-docker-guide.md](docs/03-docker-guide.md) |
| 4 | AWS Prerequisites | [docs/04-aws-prerequisites.md](docs/04-aws-prerequisites.md) |
| 5 | EKS Cluster Setup | [docs/05-eks-cluster-setup.md](docs/05-eks-cluster-setup.md) |
| 6 | NGINX Ingress Controller | [docs/06-nginx-ingress-controller.md](docs/06-nginx-ingress-controller.md) |
| 7 | Argo CD Setup | [docs/07-argocd-setup.md](docs/07-argocd-setup.md) |
| 8 | GitHub Actions CI/CD | [docs/08-github-actions-cicd.md](docs/08-github-actions-cicd.md) |
| 9 | Helm Chart Guide | [docs/09-helm-chart-guide.md](docs/09-helm-chart-guide.md) |
| 10 | Full Deployment Walkthrough | [docs/10-full-deployment-walkthrough.md](docs/10-full-deployment-walkthrough.md) |
| — | Architecture Diagrams | [docs/architecture-diagrams.md](docs/architecture-diagrams.md) |

---

## ✅ Conclusion

This project brings together a modern DevOps toolchain to create a **fully automated, production-grade deployment pipeline** for a Go web application. It demonstrates that even a simple application benefits enormously from proper DevOps practices.

**Key outcomes:**

- ✅ **Zero-touch deployments** — a `git push` to `main` is all it takes to ship code to production
- ✅ **Secure containers** — distroless images with no unnecessary attack surface
- ✅ **Consistent environments** — Docker ensures the app runs identically everywhere
- ✅ **Self-healing infrastructure** — Argo CD continuously reconciles cluster state with Git
- ✅ **Scalable and managed** — Amazon EKS handles Kubernetes control plane operations
- ✅ **Observable pipeline** — every step is logged and visible in GitHub Actions

This architecture is a solid foundation that can be extended with monitoring (Prometheus/Grafana), secret management (AWS Secrets Manager), multi-environment deployments (staging/production), and more.

---

## 📄 License

Copyright © 2024 Sanket Chopade. Licensed under the [Apache License 2.0](LICENSE).

---

<div align="center">

Made with ❤️ by [Sanket Chopade](https://github.com/Sanket006)

</div>
