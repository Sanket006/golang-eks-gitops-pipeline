# DevOps Pipeline for Go Web Application

This document provides a comprehensive overview of the DevOps practices implemented for the Go web application. The application is a simple website built with Golang using the `net/http` package, enhanced with a full CI/CD pipeline, containerization, and cloud-native deployment.

---

## 📐 Architecture Overview

The diagram below illustrates the end-to-end DevOps pipeline — from code commit to production deployment.

![Pipeline Architecture](https://github.com/user-attachments/assets/45f4ef12-c5b5-4247-9d43-356b5dfb671b)

---

## 🐳 Containerization with Docker

### What is Containerization?

Containerization is the process of packaging an application along with all its dependencies into a portable, self-contained unit called a **container**. This ensures the application runs consistently across different environments, regardless of the underlying infrastructure.

We use **Docker** to containerize the Go web application.

### Multi-Stage Docker Build

The `Dockerfile` uses a **multi-stage build** strategy, which is a Docker feature that allows multiple build stages within a single file. This approach:

- ✅ Reduces the final image size significantly
- ✅ Improves security by excluding build tools and unnecessary files from the runtime image
- ✅ Uses a minimal `distroless` base image for the production stage

### Docker Commands

**Build the Docker image:**

```bash
docker build -t <your-docker-username>/go-web-app .
```

**Run the Docker container locally:**

```bash
docker run -p 8080:8080 <your-docker-username>/go-web-app
```

**Push the image to Docker Hub:**

```bash
docker push <your-docker-username>/go-web-app
```

> 💡 Replace `<your-docker-username>` with your actual Docker Hub username.

---

## ⚙️ Continuous Integration (CI)

### What is CI?

**Continuous Integration (CI)** is the practice of automatically integrating code changes from all contributors into a shared repository multiple times a day. Each integration triggers an automated build and test process, helping catch bugs early and keeping the codebase always in a deployable state.

### CI with GitHub Actions

We use **GitHub Actions** to automate the CI process. GitHub Actions provides a powerful, flexible workflow engine that runs directly within GitHub.

**The CI workflow performs the following steps on every push to `main`:**

1. ✅ Checkout the source code from the repository
2. ✅ Build the Go application binary
3. ✅ Run all unit tests (`go test ./...`)
4. ✅ Run static code analysis using `golangci-lint`

---

## 🚀 Continuous Deployment (CD)

### What is CD?

**Continuous Deployment (CD)** is the practice of automatically deploying code changes to a production environment after passing CI checks. This reduces the time between a code change and its delivery to end users, enabling faster release cycles.

### CD with Argo CD (GitOps)

We use **Argo CD** to implement CD using a **GitOps** approach. Argo CD is a declarative, GitOps-based continuous delivery tool built for Kubernetes.

**Key characteristics:**

- Git repository acts as the **single source of truth** for the desired deployment state
- Argo CD **continuously monitors** the Git repo and automatically syncs the Kubernetes cluster to match
- Any change to the Helm chart or manifests in Git triggers an automatic deployment

**CD pipeline flow:**

1. A code change is pushed to the `main` branch
2. GitHub Actions builds and tests the code (CI)
3. A new Docker image is built and pushed to Docker Hub, tagged with the GitHub run ID
4. The Helm chart's `values.yaml` is automatically updated with the new image tag
5. Argo CD detects the change in Git and syncs the new version to the Kubernetes cluster

---

## 🛠️ Tech Stack

| Component | Technology |
|---|---|
| Application | Go (Golang) |
| Containerization | Docker (multi-stage build) |
| Container Registry | Docker Hub |
| CI Pipeline | GitHub Actions |
| CD / GitOps | Argo CD |
| Kubernetes | Amazon EKS (Managed Node Groups) |
| Package Manager | Helm |
| Ingress | NGINX Ingress Controller |

---

## 📚 Additional Guides

Follow these step-by-step guides to set up the full infrastructure:

1. [EKS Prerequisites](../eks/prerequisites.md) — Install required CLI tools
2. [EKS Cluster Setup](../eks/install-eks-cluster.md) — Create and manage the EKS cluster
3. [NGINX Ingress Controller](../ingress-controller/nginx/install-nginx-ingress-controller.md) — Deploy ingress on AWS
4. [Argo CD Installation](../gitops/argocd/install-argocd.md) — Set up GitOps-based CD

---

## ✅ Conclusion

This project demonstrates a complete, production-ready DevOps pipeline for a Go web application. By combining Docker, GitHub Actions, Helm, and Argo CD on Amazon EKS, we achieve:

- **Fast, reliable releases** through automated CI/CD
- **Secure and lightweight containers** through multi-stage Docker builds
- **Consistent deployments** through GitOps principles with Argo CD
- **Scalable infrastructure** using managed Kubernetes on AWS EKS
