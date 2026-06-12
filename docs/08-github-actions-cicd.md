# 08 — GitHub Actions CI/CD Pipeline

> **Navigation:** [← Argo CD Setup](07-argocd-setup.md) | [Documentation Hub](index.md) | **Next →** [Helm Chart Guide](09-helm-chart-guide.md)

---

## Overview

This guide explains the **GitHub Actions CI/CD pipeline** — the automation layer that takes a code commit and turns it into a deployed application. You will learn what each job does, how to configure the required secrets, and how the pipeline connects to Argo CD.

**Pipeline file:** [`.github/workflows/cicd.yaml`](../.github/workflows/cicd.yaml)

---

## When Does the Pipeline Run?

The pipeline triggers on every **push to the `main` branch**, with the following **exclusions** (changes to these paths do NOT trigger a run):

```yaml
on:
  push:
    branches:
      - main
    paths-ignore:
      - 'helm/**'      # Helm chart updates (done by the pipeline itself)
      - 'k8s/**'       # Raw Kubernetes manifests
      - 'README.md'    # Documentation
```

> **Why exclude `helm/**`?** The pipeline's final job commits an updated image tag back to `helm/values.yaml`. Without the exclusion, that commit would trigger another pipeline run — causing an infinite loop.

---

## Pipeline Overview

The pipeline has **4 jobs**. Two run in parallel, then two run in sequence:

```
Push to main
    │
    ├──► Job 1: build         (compile + test)     ──► triggers Job 3
    │
    └──► Job 2: code-quality  (lint)                (independent, parallel)
              │
              ▼
         Job 3: push          (Docker build + push) ──► triggers Job 4
              │
              ▼
         Job 4: update-newtag-in-helm-chart         (update values.yaml → git push)
              │
              ▼
         Argo CD detects the commit → syncs the cluster → deployment complete ✅
```

---

## Job Breakdown

### Job 1 — `build`

**Purpose:** Compile the Go binary and run all unit tests.

```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: 1.22
      - run: go build -o go-web-app    # Compile
      - run: go test ./...             # Run tests
```

**What it validates:**
- The Go code compiles without errors
- All unit tests in `main_test.go` pass
- If either step fails, Jobs 3 and 4 are **blocked** — nothing is deployed

---

### Job 2 — `code-quality`

**Purpose:** Run static analysis and linting to catch code style issues and potential bugs.

```yaml
  code-quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-go@v5
        with:
          go-version: 1.22
      - uses: golangci/golangci-lint-action@v6
        with:
          version: v1.56.2
```

**What it validates:**
- Code style and formatting (equivalent to `gofmt`)
- Common Go anti-patterns and bugs
- Unused variables, unreachable code, etc.

> This job runs **in parallel with Job 1**. It does not block the build/push flow — but failures are visible in the GitHub UI and serve as a quality gate for code reviews.

---

### Job 3 — `push`

**Purpose:** Build the Docker image and push it to Docker Hub.

```yaml
  push:
    runs-on: ubuntu-latest
    needs: build    # Only runs after Job 1 succeeds
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v1
      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      - uses: docker/build-push-action@v6
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: ${{ secrets.DOCKERHUB_USERNAME }}/go-web-app:${{ github.run_id }}
```

**Key details:**
- Uses **Docker Buildx** for efficient, cross-platform image building
- The image tag is `${{ github.run_id }}` — a unique integer for each pipeline run (e.g., `27403656107`)
- Using the run ID instead of `latest` means every deployment is **traceable and reversible**

---

### Job 4 — `update-newtag-in-helm-chart`

**Purpose:** Automatically update the Helm chart's `values.yaml` with the new image tag, then push the change to Git — which triggers Argo CD.

```yaml
  update-newtag-in-helm-chart:
    runs-on: ubuntu-latest
    needs: push    # Only runs after Job 3 succeeds
    steps:
      - uses: actions/checkout@v4
        with:
          token: ${{ secrets.TOKEN }}    # PAT with push permission
      - name: Update tag in Helm chart
        run: |
          sed -i 's/tag: .*/tag: "${{ github.run_id }}"/' \
            helm/go-web-app-chart/values.yaml
      - name: Commit and push changes
        run: |
          git config --global user.email "sanketchopade6@gmail.com"
          git config --global user.name "Sanket Chopade"
          git add helm/go-web-app-chart/values.yaml
          git commit -m "Update tag in Helm chart"
          git push
```

**What happens:**
1. Uses `sed` to find the `tag:` line in `values.yaml` and replace the value with the new run ID
2. Commits the change with a descriptive message
3. Pushes the commit to the `main` branch (using the GitHub PAT — the `TOKEN` secret)
4. Argo CD detects this new commit (within ~3 minutes) and syncs the cluster

---

## Required GitHub Secrets

You must configure three secrets in your GitHub repository before the pipeline can run:

**Navigate to:** GitHub Repository → Settings → Secrets and variables → Actions → New repository secret

| Secret Name | Value | How to Get It |
|---|---|---|
| `DOCKERHUB_USERNAME` | Your Docker Hub username | Your Docker Hub account username |
| `DOCKERHUB_TOKEN` | Docker Hub access token | Docker Hub → Account Settings → Security → New Access Token |
| `TOKEN` | GitHub Personal Access Token | GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic) → Generate new token — select `repo` scope |

> ⚠️ **Security:** Never put these values directly in `cicd.yaml`. Always use GitHub Secrets. Secrets are encrypted and only injected at runtime.

---

## Setting Up GitHub Secrets (Step by Step)

### Creating a Docker Hub Access Token

1. Log in to [hub.docker.com](https://hub.docker.com)
2. Click your profile → **Account Settings**
3. Go to **Security** → **New Access Token**
4. Name it `github-actions-go-web-app`
5. Set permissions to **Read & Write**
6. Copy the token and add it as the `DOCKERHUB_TOKEN` secret

### Creating a GitHub Personal Access Token (PAT)

1. Go to [github.com](https://github.com) → Click your profile → **Settings**
2. Scroll to **Developer settings** → **Personal access tokens** → **Tokens (classic)**
3. Click **Generate new token (classic)**
4. Name it `go-web-app-cicd`
5. Set expiration to 90 days (or No expiration for testing)
6. Check the **`repo`** scope (all checkboxes under it)
7. Click **Generate token**
8. Copy the token and add it as the `TOKEN` secret

---

## Monitoring Pipeline Runs

1. Go to your GitHub repository
2. Click the **Actions** tab
3. Click on any workflow run to see the job graph and logs

**Pipeline status indicators:**

| Symbol | Meaning |
|---|---|
| 🟡 Yellow circle | Job is running |
| ✅ Green check | Job passed |
| ❌ Red X | Job failed — click to view logs |
| ⏭️ Skipped | Job was skipped (dependency failed) |

---

## Common Issues

| Issue | Likely Cause | Fix |
|---|---|---|
| `go test` fails | Code has a bug or test failure | Fix the failing test — check logs in the `build` job |
| `denied: requested access to the resource is denied` | Wrong Docker Hub credentials | Re-create the `DOCKERHUB_TOKEN` secret |
| `fatal: could not read Password` | `TOKEN` secret missing or expired | Re-create the `TOKEN` secret with `repo` scope |
| Job 4 commit fails | PAT lacks push permission | Ensure the PAT has the full `repo` scope |
| Argo CD not syncing after Job 4 | Argo CD poll interval | Wait up to 3 minutes, or manually click **Sync** in the Argo CD UI |

---

## What's Next?

The CI/CD pipeline is fully understood. Next, explore the Helm chart that packages the Kubernetes manifests.

- **Next:** [09 — Helm Chart Guide](09-helm-chart-guide.md)
- **Related:** [07 — Argo CD Setup](07-argocd-setup.md) — how Argo CD reacts to the pipeline

---

*[← Argo CD Setup](07-argocd-setup.md) | [Documentation Hub](index.md) | [Next: Helm Chart Guide →](09-helm-chart-guide.md)*
