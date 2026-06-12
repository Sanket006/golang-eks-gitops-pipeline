# 03 — Docker Guide

> **Navigation:** [← Local Development](02-local-development.md) | [Documentation Hub](index.md) | **Next →** [AWS Prerequisites](04-aws-prerequisites.md)

---

## Overview

This guide explains how to containerize the Go application using Docker, run it locally as a container, and push it to Docker Hub. Understanding this step is essential — Docker is the foundation on which the entire deployment pipeline is built.

---

## Prerequisites

- **Docker Desktop** installed and running — [Download Docker](https://www.docker.com/products/docker-desktop/)
- A **Docker Hub account** — [Sign up free at hub.docker.com](https://hub.docker.com/)

Verify Docker is running:

```bash
docker --version
# Expected: Docker version 24.x.x or higher
```

---

## Understanding the Dockerfile

The project uses a **multi-stage Docker build**. This is a key DevOps best practice — it separates the build environment from the runtime environment, producing a final image that is tiny and secure.

```dockerfile
# ── Stage 1: Build ─────────────────────────────────────────
FROM golang:1.21 as base

WORKDIR /app
COPY go.mod ./
RUN go mod download
COPY . .
RUN go build -o main .

# ── Stage 2: Runtime ───────────────────────────────────────
FROM gcr.io/distroless/base

COPY --from=base /app/main .
COPY --from=base /app/static ./static

EXPOSE 8080
CMD ["./main"]
```

### Why Two Stages?

| Stage | Base Image | Purpose | What It Contains |
|---|---|---|---|
| **Build** (`base`) | `golang:1.21` (~800 MB) | Compile the Go binary | Full Go toolchain, source code, dependencies |
| **Runtime** (final) | `gcr.io/distroless/base` (~20 MB) | Run the binary | **Only** the compiled binary + static files |

**What is `distroless`?**
`gcr.io/distroless/base` is a Google-maintained minimal base image that contains:
- ✅ The OS libraries needed to run a compiled binary
- ❌ No shell (`sh`, `bash`)
- ❌ No package manager (`apt`, `yum`)
- ❌ No debugging tools

This dramatically reduces the attack surface of the running container. There is nothing for an attacker to execute even if they break in.

---

## Step 1 — Build the Docker Image

From the project root directory, build the image:

```bash
docker build -t <your-dockerhub-username>/go-web-app .
```

Replace `<your-dockerhub-username>` with your actual Docker Hub username (e.g., `sanket006`).

**What happens during the build:**
1. Docker runs Stage 1: pulls `golang:1.21`, copies source, runs `go build`
2. Docker runs Stage 2: pulls `distroless/base`, copies only the binary and `static/` folder
3. The final image is tagged with your username

**Verify the image was created:**

```bash
docker images | grep go-web-app
```

---

## Step 2 — Run the Container Locally

```bash
docker run -p 8080:8080 <your-dockerhub-username>/go-web-app
```

**What `-p 8080:8080` means:**
- Left side (`8080`): Port on your **host machine** (your laptop)
- Right side (`8080`): Port **inside the container**

The app is now accessible at the same URLs as the local run:

| URL | Page |
|---|---|
| `http://localhost:8080/home` | Home page |
| `http://localhost:8080/courses` | Courses page |
| `http://localhost:8080/about` | About page |
| `http://localhost:8080/contact` | Contact page |

**Stop the container** with `Ctrl + C` or:

```bash
docker ps                          # Find the container ID
docker stop <container-id>
```

---

## Step 3 — Log In to Docker Hub

```bash
docker login
```

Enter your Docker Hub username and password (or access token) when prompted.

> **Security tip:** Use a Docker Hub **Access Token** instead of your account password. Generate one at: Docker Hub → Account Settings → Security → New Access Token.

---

## Step 4 — Push the Image to Docker Hub

```bash
docker push <your-dockerhub-username>/go-web-app
```

This uploads the image to your Docker Hub repository, making it accessible from anywhere — including your EKS cluster.

**Verify on Docker Hub:** Visit `https://hub.docker.com/r/<your-dockerhub-username>/go-web-app`

---

## How the CI/CD Pipeline Uses Docker

In the automated pipeline, this manual process is replaced entirely by GitHub Actions:

1. The `push` job uses **Docker Buildx** for efficient multi-platform builds
2. The image is tagged with the **GitHub Actions run ID** (a unique number), not `latest`
3. The tag is automatically written into `helm/go-web-app-chart/values.yaml` by the next job

This means every deployment is tied to a specific, traceable pipeline run. You can always roll back to any previous image tag.

See [08 — GitHub Actions CI/CD](08-github-actions-cicd.md) for the full pipeline breakdown.

---

## Common Issues

| Issue | Likely Cause | Fix |
|---|---|---|
| `Cannot connect to the Docker daemon` | Docker Desktop not running | Start Docker Desktop |
| `denied: requested access to the resource is denied` | Not logged in to Docker Hub | Run `docker login` |
| `port is already allocated` | Something else is on port 8080 | Use `-p 9090:8080` to map to a different host port |
| Image builds but app can't find static files | Running binary from wrong directory | This is handled correctly in the Dockerfile via `COPY --from=base /app/static ./static` |

---

## What's Next?

Your Docker image is ready and published. The next step is to set up the AWS tools needed to create your EKS cluster.

- **Next:** [04 — AWS Prerequisites](04-aws-prerequisites.md)
- **Related:** [08 — GitHub Actions CI/CD](08-github-actions-cicd.md) — how Docker is used in the pipeline

---

*[← Local Development](02-local-development.md) | [Documentation Hub](index.md) | [Next: AWS Prerequisites →](04-aws-prerequisites.md)*
