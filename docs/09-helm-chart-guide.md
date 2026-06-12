# 09 — Helm Chart Guide

> **Navigation:** [← GitHub Actions CI/CD](08-github-actions-cicd.md) | [Documentation Hub](index.md) | **Next →** [Full Deployment Walkthrough](10-full-deployment-walkthrough.md)

---

## Overview

This guide explains what Helm is, how the Helm chart in this project is structured, and how Argo CD uses it to deploy the Go application to Kubernetes.

**Chart location:** [`helm/go-web-app-chart/`](../helm/go-web-app-chart/)

---

## What Is Helm?

**Helm** is the package manager for Kubernetes. It allows you to define, install, and upgrade Kubernetes applications using **charts** — reusable, versioned bundles of Kubernetes YAML templates.

**Why use Helm instead of raw YAML?**

| Raw Kubernetes YAML | Helm Chart |
|---|---|
| Static — values are hardcoded | Dynamic — values are parameterized via `values.yaml` |
| No versioning | Charts are versioned with semantic versioning |
| Hard to manage across environments | Easy to swap values (e.g., image tag) without changing templates |
| Manual updates required | Automated — CI/CD can update a single value |

In this project, Helm serves one critical purpose in the CI/CD pipeline: the image tag in `values.yaml` is the **trigger mechanism** for Argo CD deployments. When GitHub Actions updates the tag, Argo CD syncs the new version.

---

## Chart Structure

```
helm/go-web-app-chart/
├── Chart.yaml            # Chart metadata
├── values.yaml           # Default configuration values
├── .helmignore           # Files to exclude from the chart package
└── templates/
    ├── _helpers.tpl      # Reusable template helper functions
    ├── deployment.yaml   # Kubernetes Deployment resource
    ├── service.yaml      # Kubernetes Service resource
    └── ingress.yaml      # Kubernetes Ingress resource
```

---

## `Chart.yaml` — Chart Metadata

```yaml
apiVersion: v2
name: go-web-app-chart
description: A Helm chart for Kubernetes
type: application
version: 0.1.0      # Chart version — increment when you change the chart structure
appVersion: "1.16.0" # Application version — informational only
```

- **`version`** — The chart version. Increment this when templates or default values change.
- **`appVersion`** — The version of the application the chart deploys. In this project, the actual running version is tracked via the Docker image tag in `values.yaml`, not this field.

---

## `values.yaml` — Configuration Values

This is the **most important file** from a CI/CD perspective:

```yaml
replicaCount: 1

image:
  repository: sanket006/go-web-app
  pullPolicy: IfNotPresent
  tag: "27403656107"     # ← This value is automatically updated by the CI/CD pipeline

ingress:
  enabled: false
  className: ""
  annotations: {}
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: ImplementationSpecific
```

### Key Values Explained

| Value | Description | Default |
|---|---|---|
| `replicaCount` | Number of Pod replicas to run | `1` |
| `image.repository` | Docker Hub image repository | `sanket006/go-web-app` |
| `image.pullPolicy` | When to pull the image (`Always`, `IfNotPresent`, `Never`) | `IfNotPresent` |
| `image.tag` | The specific image version to deploy | Updated by CI/CD |

### How the CI/CD Pipeline Updates `values.yaml`

The `update-newtag-in-helm-chart` job in GitHub Actions uses `sed` to update the tag:

```bash
sed -i 's/tag: .*/tag: "${{ github.run_id }}"/' helm/go-web-app-chart/values.yaml
```

This replaces the `tag:` line with the new GitHub Actions run ID, then commits and pushes the change. Argo CD detects this commit and triggers a new deployment.

---

## Template Files

### `deployment.yaml` — Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: go-web-app
  labels:
    app: go-web-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: go-web-app
  template:
    metadata:
      labels:
        app: go-web-app
    spec:
      containers:
      - name: go-web-app
        image: sanket006/go-web-app:{{ .Values.image.tag }}
        ports:
        - containerPort: 8080
```

**What this deploys:**
- A `Deployment` named `go-web-app` managing 1 Pod replica
- Each Pod runs the container `sanket006/go-web-app:<tag>` on port `8080`
- The `{{ .Values.image.tag }}` is Helm's Go template syntax — it reads the value from `values.yaml`

### `service.yaml` — Kubernetes Service

The `Service` exposes the Deployment inside the cluster:

```
go-web-app Service (ClusterIP, port 80)
    │
    └──► Routes to Pods matching label app=go-web-app on port 8080
```

- **Type:** `ClusterIP` — internal only, not accessible from outside the cluster
- **Port:** 80 (external) → 8080 (container)
- The Ingress Controller forwards traffic to this Service

### `ingress.yaml` — Kubernetes Ingress Resource

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: go-web-app
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: go-web-app.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: go-web-app
            port:
              number: 80
```

**What this does:**
- Registers a routing rule with the NGINX Ingress Controller
- Any HTTP request for hostname `go-web-app.local` at path `/` is forwarded to the `go-web-app` Service on port 80

---

## Raw Kubernetes Manifests (Alternative)

The `k8s/manifests/` directory contains equivalent raw YAML files without Helm templating. These can be used if you want to deploy manually without Helm:

```bash
kubectl apply -f k8s/manifests/deployment.yaml
kubectl apply -f k8s/manifests/service.yaml
kubectl apply -f k8s/manifests/ingress.yaml
```

> **Note:** The raw manifests use a hardcoded image tag (`v1`) and are not connected to the CI/CD pipeline. Use the Helm chart for automated deployments.

---

## Useful Helm Commands

```bash
# Preview what Helm will apply (dry run — does not apply anything)
helm template go-web-app-chart ./helm/go-web-app-chart

# Install the chart manually
helm install go-web-app ./helm/go-web-app-chart

# Upgrade with a new image tag
helm upgrade go-web-app ./helm/go-web-app-chart --set image.tag=<new-tag>

# Check release status
helm status go-web-app

# Uninstall
helm uninstall go-web-app

# List all installed releases
helm list
```

> In normal operation, you do NOT need to run any `helm` commands manually. Argo CD manages all Helm operations automatically.

---

## Common Issues

| Issue | Likely Cause | Fix |
|---|---|---|
| Argo CD shows `OutOfSync` | `values.yaml` doesn't match cluster state | Click **Sync** in Argo CD UI or wait for auto-sync |
| Pod using old image | `sed` command in CI failed to update tag | Check Job 4 logs in GitHub Actions |
| `ImagePullBackOff` | Tag in `values.yaml` doesn't exist on Docker Hub | Verify the image tag at hub.docker.com |
| Ingress not routing traffic | `ingressClassName: nginx` mismatch | Verify NGINX controller is installed with `kubectl get ingressclass` |

---

## What's Next?

You now understand all the individual components. The final guide walks through the complete end-to-end deployment from a fresh clone to a live app.

- **Next:** [10 — Full Deployment Walkthrough](10-full-deployment-walkthrough.md)
- **Related:** [Architecture Diagrams](architecture-diagrams.md)

---

*[← GitHub Actions CI/CD](08-github-actions-cicd.md) | [Documentation Hub](index.md) | [Next: Full Deployment Walkthrough →](10-full-deployment-walkthrough.md)*
