# Argo CD — Installation Guide

**Argo CD** is a declarative, GitOps-based continuous delivery tool for Kubernetes. It monitors a Git repository as the source of truth and automatically syncs the desired application state to your Kubernetes cluster.

---

## Step 1: Install Argo CD Using Official Manifests

Create a dedicated namespace for Argo CD and apply the official installation manifest:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

**Verify that all Argo CD pods are running:**

```bash
kubectl get pods -n argocd
```

> ⏳ It may take a couple of minutes for all pods to reach the `Running` state.

---

## Step 2: Expose the Argo CD UI

By default, the Argo CD server is not exposed externally. To access the UI via a browser, patch the `argocd-server` service to use a **LoadBalancer**:

### Linux / macOS

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

### Windows (Command Prompt / PowerShell)

On Windows, the JSON payload requires escaped quotes:

```bash
kubectl patch svc argocd-server -n argocd -p '{\"spec\": {\"type\": \"LoadBalancer\"}}'
```

---

## Step 3: Get the Load Balancer IP / Hostname

Retrieve the external IP or hostname assigned to the Argo CD server:

```bash
kubectl get svc argocd-server -n argocd
```

Look for the value under the **`EXTERNAL-IP`** column. Once available, open it in your browser:

```
https://<EXTERNAL-IP>
```

> ⏳ It may take a few minutes for the Load Balancer to be provisioned by AWS.

---

## Step 4: Log In to the Argo CD UI

The default **admin username** is `admin`. To get the initial password, run:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

> 🔐 It is strongly recommended to change the default password after your first login via **User Info → Update Password** in the Argo CD UI.

---

## Next Steps

After logging in, you can:

- Connect your Git repository as an Argo CD application source
- Create an **Application** resource pointing to your Helm chart or Kubernetes manifests
- Enable **auto-sync** to automatically deploy changes when Git is updated
