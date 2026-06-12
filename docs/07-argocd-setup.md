# 07 — Argo CD Setup

> **Navigation:** [← NGINX Ingress Controller](06-nginx-ingress-controller.md) | [Documentation Hub](index.md) | **Next →** [GitHub Actions CI/CD](08-github-actions-cicd.md)

---

## Overview

This guide walks you through installing **Argo CD** on your EKS cluster, exposing its web UI, and connecting it to your Git repository to enable **GitOps-based continuous deployment**.

> **Prerequisites:** A running EKS cluster from [05 — EKS Cluster Setup](05-eks-cluster-setup.md)
>
> **Estimated time:** 10–15 minutes

---

## What Is Argo CD?

**Argo CD** is a declarative, GitOps-based continuous delivery tool for Kubernetes.

### The GitOps Principle

In a traditional CD pipeline, a script or person runs `kubectl apply` to deploy. In GitOps:

1. **Git is the single source of truth** — the desired state of the cluster is defined in Git (Helm charts, YAML manifests)
2. **Argo CD continuously watches Git** — it polls the repository for changes
3. **When Git changes, the cluster changes** — Argo CD automatically syncs the cluster to match Git
4. **The cluster is always in sync** — any manual change to the cluster is automatically reverted

This means deployments are:
- ✅ **Auditable** — every change is a Git commit with author, timestamp, and message
- ✅ **Reversible** — rolling back is `git revert`
- ✅ **Consistent** — the cluster always matches what Git says
- ✅ **Automated** — no human needs to run deployment commands

---

## Step 1 — Install Argo CD

Create a dedicated namespace and apply the official installation manifest:

```bash
kubectl create namespace argocd

kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

**Verify all Argo CD pods are starting:**

```bash
kubectl get pods -n argocd
```

You should see these pods:

```
NAME                                                READY   STATUS    AGE
argocd-application-controller-0                     1/1     Running   2m
argocd-applicationset-controller-xxxxxxxxx-xxxxx    1/1     Running   2m
argocd-dex-server-xxxxxxxxx-xxxxx                   1/1     Running   2m
argocd-notifications-controller-xxxxxxxxx-xxxxx     1/1     Running   2m
argocd-redis-xxxxxxxxx-xxxxx                        1/1     Running   2m
argocd-repo-server-xxxxxxxxx-xxxxx                  1/1     Running   2m
argocd-server-xxxxxxxxx-xxxxx                       1/1     Running   2m
```

> ⏳ All pods may take 2–3 minutes to reach `Running` status. Re-run the command until all are ready.

---

## Step 2 — Expose the Argo CD Web UI

By default, the Argo CD server is only accessible inside the cluster. Patch the service to expose it via an AWS Load Balancer:

**On Linux / macOS:**

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

**On Windows (Command Prompt or PowerShell):**

```bash
kubectl patch svc argocd-server -n argocd -p '{\"spec\": {\"type\": \"LoadBalancer\"}}'
```

---

## Step 3 — Get the Argo CD External IP

Retrieve the external IP or hostname for the Argo CD server:

```bash
kubectl get svc argocd-server -n argocd
```

Look for the `EXTERNAL-IP` column:

```
NAME            TYPE           CLUSTER-IP     EXTERNAL-IP                                       PORT(S)
argocd-server   LoadBalancer   10.100.xx.xx   xxxxxx.elb.us-east-1.amazonaws.com   80:xxxx/TCP,443:xxxx/TCP
```

> ⏳ `EXTERNAL-IP` may show `<pending>` for a few minutes. Re-run until a hostname appears.

Open the Argo CD UI in your browser:

```
https://<EXTERNAL-IP>
```

> You may see a browser security warning about an untrusted certificate. Click **Advanced → Proceed** — this is because Argo CD uses a self-signed certificate by default.

---

## Step 4 — Log In to Argo CD

**Username:** `admin`

**Get the initial password:**

```bash
# Linux / macOS
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo

# Windows PowerShell
$encoded = kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}"
[System.Text.Encoding]::UTF8.GetString([System.Convert]::FromBase64String($encoded))
```

Log in with `admin` and the decoded password.

> 🔐 **Security:** After your first login, immediately change the admin password:
> Argo CD UI → **User Info** (top left) → **Update Password**

---

## Step 5 — Create an Argo CD Application

An **Application** in Argo CD connects a Git repository (source of truth) to a Kubernetes namespace (deployment target).

### Option A — Via the Argo CD UI

1. Click **+ New App** in the Argo CD UI
2. Fill in the following:

| Field | Value |
|---|---|
| **Application Name** | `go-web-app` |
| **Project** | `default` |
| **Sync Policy** | `Automatic` |
| **Repository URL** | `https://github.com/Sanket006/golang-eks-gitops-pipeline` |
| **Revision** | `main` |
| **Path** | `helm/go-web-app-chart` |
| **Cluster URL** | `https://kubernetes.default.svc` |
| **Namespace** | `default` |

3. Click **Create**

### Option B — Via YAML Manifest

Create and apply the following manifest:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: go-web-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/Sanket006/golang-eks-gitops-pipeline
    targetRevision: main
    path: helm/go-web-app-chart
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true       # Remove resources that are deleted from Git
      selfHeal: true    # Revert manual changes made directly to the cluster
```

```bash
kubectl apply -f argocd-application.yaml
```

---

## Step 6 — Verify the Sync

Back in the Argo CD UI, you should see the `go-web-app` application syncing and turning green (`Synced` / `Healthy`).

Click on the application to see the full resource tree:

```
go-web-app (Application)
├── Deployment/go-web-app        ✅ Healthy
├── Service/go-web-app           ✅ Healthy
└── Ingress/go-web-app           ✅ Healthy
```

---

## How Argo CD Fits Into the CI/CD Pipeline

In the automated pipeline (defined in [`.github/workflows/cicd.yaml`](../.github/workflows/cicd.yaml)):

1. GitHub Actions builds and pushes a new Docker image (tagged with the run ID)
2. GitHub Actions updates `helm/go-web-app-chart/values.yaml` with the new tag and commits it
3. **Argo CD polls Git**, detects the new commit, and syncs the cluster
4. The cluster pulls the new image and rolls out the updated Pods

See [08 — GitHub Actions CI/CD](08-github-actions-cicd.md) for the full pipeline breakdown.

---

## Common Issues

| Issue | Likely Cause | Fix |
|---|---|---|
| Pods stuck in `Pending` | Insufficient cluster capacity | Verify node group has enough resources with `kubectl describe node` |
| UI shows `OutOfSync` | Helm values mismatch | Check `values.yaml` matches what's running. Click **Sync** to force apply |
| Application shows `Degraded` | Pod crash or image pull error | Check `kubectl describe pod <name>` and pod logs |
| Can't reach UI | Load Balancer pending | Wait a few minutes and retry `kubectl get svc argocd-server -n argocd` |
| `ImagePullBackOff` | Docker Hub image not found | Verify the image tag in `values.yaml` exists on Docker Hub |

---

## What's Next?

Argo CD is installed and connected to your repository. Now configure the **GitHub Actions CI/CD pipeline** to automate the build and deployment trigger.

- **Next:** [08 — GitHub Actions CI/CD](08-github-actions-cicd.md)
- **Related:** [09 — Helm Chart Guide](09-helm-chart-guide.md)

---

*[← NGINX Ingress Controller](06-nginx-ingress-controller.md) | [Documentation Hub](index.md) | [Next: GitHub Actions CI/CD →](08-github-actions-cicd.md)*
