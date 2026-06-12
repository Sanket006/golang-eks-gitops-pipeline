# 📚 Documentation Hub — Golang EKS GitOps Pipeline

Welcome to the full documentation for the **Golang EKS GitOps Pipeline** project.

This hub is the starting point for understanding and deploying the project from scratch. Whether you are new to DevOps or an experienced engineer looking to explore the architecture, you will find a structured guide here.

---

## 📖 What Is This Project?

A lightweight Go web application served with a fully automated, production-grade DevOps pipeline. The focus of the project is **not** the application itself, but the surrounding automation — containerization, CI/CD, GitOps, and cloud-native Kubernetes deployment.

**The result:** A single `git push` to `main` automatically builds, tests, containerizes, and deploys the latest version of the application to Amazon EKS — with zero manual steps.

---

## 🗺️ Documentation Map

Use this map to navigate to any topic. Documents are ordered to follow the natural implementation sequence.

| # | Document | Description |
|---|---|---|
| 1 | [01-project-overview.md](01-project-overview.md) | What the project does, why it was built, and how it all fits together |
| 2 | [02-local-development.md](02-local-development.md) | Run and test the Go application locally |
| 3 | [03-docker-guide.md](03-docker-guide.md) | Containerize the app with Docker and push to Docker Hub |
| 4 | [04-aws-prerequisites.md](04-aws-prerequisites.md) | Install and configure all CLI tools needed for AWS + EKS |
| 5 | [05-eks-cluster-setup.md](05-eks-cluster-setup.md) | Create and manage the Amazon EKS cluster with `eksctl` |
| 6 | [06-nginx-ingress-controller.md](06-nginx-ingress-controller.md) | Deploy the NGINX Ingress Controller on EKS |
| 7 | [07-argocd-setup.md](07-argocd-setup.md) | Install Argo CD and expose its UI |
| 8 | [08-github-actions-cicd.md](08-github-actions-cicd.md) | Understand and configure the GitHub Actions CI/CD pipeline |
| 9 | [09-helm-chart-guide.md](09-helm-chart-guide.md) | Understand the Helm chart structure and values |
| 10 | [10-full-deployment-walkthrough.md](10-full-deployment-walkthrough.md) | End-to-end walkthrough: from first clone to live deployment |
| 11 | [architecture-diagrams.md](architecture-diagrams.md) | AWS, CI/CD, and Kubernetes architecture diagrams (Mermaid) |

---

## ⚡ Quick Start Paths

### → I want to run the app locally
Go to [02-local-development.md](02-local-development.md)

### → I want to build and run the Docker image
Go to [03-docker-guide.md](03-docker-guide.md)

### → I want to deploy everything to AWS from scratch
Follow this sequence:
1. [04-aws-prerequisites.md](04-aws-prerequisites.md)
2. [05-eks-cluster-setup.md](05-eks-cluster-setup.md)
3. [06-nginx-ingress-controller.md](06-nginx-ingress-controller.md)
4. [07-argocd-setup.md](07-argocd-setup.md)
5. [08-github-actions-cicd.md](08-github-actions-cicd.md)
6. [10-full-deployment-walkthrough.md](10-full-deployment-walkthrough.md)

### → I just want to understand the architecture
Go to [01-project-overview.md](01-project-overview.md) and [architecture-diagrams.md](architecture-diagrams.md)

---

## 🛠️ Technology Overview

| Category | Technology |
|---|---|
| Application | Go (Golang) 1.22 |
| Containerization | Docker (multi-stage build) |
| Container Registry | Docker Hub |
| CI Pipeline | GitHub Actions |
| Code Quality | `golangci-lint` |
| CD / GitOps | Argo CD |
| Kubernetes | Amazon EKS |
| Package Manager | Helm |
| Ingress | NGINX Ingress Controller |
| Cloud | AWS (VPC, EKS, NLB, NAT Gateway) |

---

## 🔗 External References

- [Go Documentation](https://go.dev/doc/)
- [Docker Documentation](https://docs.docker.com/)
- [Amazon EKS User Guide](https://docs.aws.amazon.com/eks/latest/userguide/)
- [Argo CD Documentation](https://argo-cd.readthedocs.io/)
- [Helm Documentation](https://helm.sh/docs/)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/)

---

*Back to [README.md](../README.md)*
