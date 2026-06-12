# 10 — Full Deployment Walkthrough

> **Navigation:** [← Helm Chart Guide](09-helm-chart-guide.md) | [Documentation Hub](index.md)

---

## Overview

This is the complete, step-by-step guide to deploying the Golang EKS GitOps Pipeline from a fresh clone to a live application running on Amazon EKS. Follow every step in order.

**Total estimated time:** 45–60 minutes (including AWS provisioning)

---

## Prerequisites Checklist

Before starting, confirm you have:

- [ ] An **AWS account** with admin or sufficient permissions
- [ ] A **Docker Hub account**
- [ ] A **GitHub account** and a fork of this repository
- [ ] All three CLI tools installed (see [04 — AWS Prerequisites](04-aws-prerequisites.md)):
  - [ ] `go` — `go version`
  - [ ] `docker` — `docker --version`
  - [ ] `kubectl` — `kubectl version --client`
  - [ ] `eksctl` — `eksctl version`
  - [ ] `aws` — `aws --version` and `aws sts get-caller-identity` returns your account

---

## Phase 1 — Prepare the Repository

### 1.1 Fork the Repository

Fork the project to your own GitHub account:

1. Visit: `https://github.com/Sanket006/golang-eks-gitops-pipeline`
2. Click **Fork** → **Create fork**
3. All remaining steps will use **your fork's URL**

### 1.2 Clone Your Fork

```bash
git clone https://github.com/<your-username>/golang-eks-gitops-pipeline.git
cd golang-eks-gitops-pipeline
```

### 1.3 Update the Image Repository Reference

Open `helm/go-web-app-chart/values.yaml` and update the `repository` field to your Docker Hub username:

```yaml
image:
  repository: <your-dockerhub-username>/go-web-app  # ← Change this
  pullPolicy: IfNotPresent
  tag: "latest"
```

Also update the `sed` command in `.github/workflows/cicd.yaml` if your repository name differs:

```yaml
# The DOCKERHUB_USERNAME secret is used automatically — no hardcoded changes needed
tags: ${{ secrets.DOCKERHUB_USERNAME }}/go-web-app:${{github.run_id}}
```

Commit and push these changes:

```bash
git add helm/go-web-app-chart/values.yaml
git commit -m "chore: set Docker Hub repository to my account"
git push origin main
```

---

## Phase 2 — Verify Locally

### 2.1 Run the Application

```bash
go run main.go
```

Visit `http://localhost:8080/home` in your browser. You should see the home page.

### 2.2 Run Tests

```bash
go test ./...
# Expected: ok  github.com/...  0.012s
```

### 2.3 Build and Test the Docker Image

```bash
docker build -t <your-dockerhub-username>/go-web-app .
docker run -p 8080:8080 <your-dockerhub-username>/go-web-app
```

Visit `http://localhost:8080/home` again to confirm the container works.

Stop the container with `Ctrl + C`.

---

## Phase 3 — Configure GitHub Secrets

Navigate to your GitHub fork → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**

Add these three secrets:

| Secret | Value |
|---|---|
| `DOCKERHUB_USERNAME` | Your Docker Hub username |
| `DOCKERHUB_TOKEN` | A Docker Hub access token (not your password) |
| `TOKEN` | A GitHub PAT with `repo` scope |

> See [08 — GitHub Actions CI/CD](08-github-actions-cicd.md) for step-by-step instructions on creating Docker Hub tokens and GitHub PATs.

---

## Phase 4 — Create the EKS Cluster

### 4.1 Create the Cluster

```bash
eksctl create cluster --name demo-cluster --region us-east-1
```

> ⏳ Wait 10–15 minutes. Watch for the final line: `EKS cluster "demo-cluster" in "us-east-1" region is ready`

### 4.2 Verify the Cluster

```bash
kubectl get nodes
```

Both nodes should show `Ready`. If they are `NotReady`, wait 1–2 more minutes.

---

## Phase 5 — Deploy the NGINX Ingress Controller

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.11.1/deploy/static/provider/aws/deploy.yaml
```

Wait for the pod to be ready:

```bash
kubectl get pods -n ingress-nginx --watch
```

Press `Ctrl + C` once the pod shows `Running`. Then get the NLB hostname:

```bash
kubectl get svc -n ingress-nginx
```

Copy the `EXTERNAL-IP` hostname — you will need it shortly.

---

## Phase 6 — Install Argo CD

### 6.1 Install Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait for all pods to start:

```bash
kubectl get pods -n argocd --watch
```

Press `Ctrl + C` when all pods are `Running`.

### 6.2 Expose the Argo CD UI

```bash
# Linux / macOS
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

# Windows
kubectl patch svc argocd-server -n argocd -p '{\"spec\": {\"type\": \"LoadBalancer\"}}'
```

Get the Argo CD external IP:

```bash
kubectl get svc argocd-server -n argocd
```

Wait until the `EXTERNAL-IP` column has a hostname (not `<pending>`).

### 6.3 Get the Initial Admin Password

```bash
# Linux / macOS
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo

# Windows PowerShell
$encoded = kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}"
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($encoded))
```

Open `https://<argocd-EXTERNAL-IP>` in your browser and log in with `admin` and the decoded password.

---

## Phase 7 — Create the Argo CD Application

In the Argo CD UI, click **+ New App** and fill in:

| Field | Value |
|---|---|
| Application Name | `go-web-app` |
| Project | `default` |
| Sync Policy | `Automatic` |
| Repository URL | `https://github.com/<your-username>/golang-eks-gitops-pipeline` |
| Revision | `main` |
| Path | `helm/go-web-app-chart` |
| Cluster URL | `https://kubernetes.default.svc` |
| Namespace | `default` |

Click **Create**. Argo CD will perform an initial sync, deploying the application.

**Verify the sync:**

```bash
kubectl get pods -n default
kubectl get ingress -n default
```

You should see the `go-web-app` pod running and the Ingress resource created.

---

## Phase 8 — Configure Local DNS

Find the NLB IP address (from the NGINX Ingress Controller's NLB hostname):

```bash
nslookup <NLB-hostname-from-phase-5>
```

Add the IP to your hosts file:

**Linux / macOS:**
```bash
echo "<NLB-IP>  go-web-app.local" | sudo tee -a /etc/hosts
```

**Windows** (run Notepad as Administrator):
Add to `C:\Windows\System32\drivers\etc\hosts`:
```
<NLB-IP>    go-web-app.local
```

**Test access:**

```
http://go-web-app.local/home
```

🎉 **You should see the Go web application running on EKS!**

---

## Phase 9 — Trigger the Full CI/CD Pipeline

Make a small change to the application to trigger the full automated pipeline:

```bash
# Edit any Go file — for example, add a comment to main.go
echo "// Pipeline test $(date)" >> main.go

git add main.go
git commit -m "test: trigger CI/CD pipeline"
git push origin main
```

**Watch the pipeline run:**

1. Go to your GitHub fork → **Actions** tab
2. Click the running workflow — watch all 4 jobs complete

**After ~5 minutes:**

- Job 1 (`build`) and Job 2 (`code-quality`) run in parallel
- Job 3 (`push`) builds and pushes a new Docker image
- Job 4 (`update-newtag-in-helm-chart`) commits the new tag to `values.yaml`
- Argo CD detects the commit and syncs the cluster

**Verify the new deployment in Argo CD:**
- The application card should show a new commit hash
- The Pod should be restarted with the new image

---

## Verifying the Complete Setup

Run this checklist to confirm everything is working:

```bash
# 1. EKS nodes are healthy
kubectl get nodes

# 2. Go web app is running
kubectl get pods -l app=go-web-app

# 3. Service is created
kubectl get svc go-web-app

# 4. Ingress rule is applied
kubectl get ingress go-web-app

# 5. NGINX Ingress Controller is running
kubectl get pods -n ingress-nginx

# 6. Argo CD is running and synced
kubectl get pods -n argocd
```

All resources should show `Running` or `Synced`.

---

## Cleanup — Deleting All Resources

When you are finished, clean up to avoid AWS charges:

```bash
# 1. Delete the Argo CD application (optional — eksctl delete will remove everything)
kubectl delete application go-web-app -n argocd

# 2. Delete the EKS cluster (also removes all resources: NLB, VPC, nodes, etc.)
eksctl delete cluster --name demo-cluster --region us-east-1
```

> ⏳ Cluster deletion takes ~5–10 minutes. Confirm in the AWS CloudFormation console that all stacks are deleted.

**Also clean up locally:**

```bash
# Remove the context from kubeconfig
kubectl config delete-context <context-name>
```

---

## Summary

You have successfully:

- ✅ Run the Go application locally and passed all tests
- ✅ Built and pushed a Docker image to Docker Hub
- ✅ Created an Amazon EKS cluster with managed worker nodes
- ✅ Deployed the NGINX Ingress Controller and provisioned an AWS NLB
- ✅ Installed Argo CD and connected it to your Git repository
- ✅ Triggered the full CI/CD pipeline with a code commit
- ✅ Accessed the live application via `go-web-app.local`

---

## Further Reading

| Topic | Document |
|---|---|
| Project architecture | [01 — Project Overview](01-project-overview.md) |
| Architecture diagrams | [Architecture Diagrams](architecture-diagrams.md) |
| Docker multi-stage builds | [03 — Docker Guide](03-docker-guide.md) |
| GitHub Actions pipeline | [08 — GitHub Actions CI/CD](08-github-actions-cicd.md) |
| Helm chart details | [09 — Helm Chart Guide](09-helm-chart-guide.md) |

---

*[← Helm Chart Guide](09-helm-chart-guide.md) | [Documentation Hub](index.md)*
