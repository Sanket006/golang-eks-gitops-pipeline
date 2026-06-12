# 06 — NGINX Ingress Controller

> **Navigation:** [← EKS Cluster Setup](05-eks-cluster-setup.md) | [Documentation Hub](index.md) | **Next →** [Argo CD Setup](07-argocd-setup.md)

---

## Overview

This guide explains what the NGINX Ingress Controller is, why it is needed, and how to deploy it on your Amazon EKS cluster.

> **Prerequisites:** A running EKS cluster from [05 — EKS Cluster Setup](05-eks-cluster-setup.md)

---

## What Is an Ingress Controller?

Without an Ingress Controller, the only way to expose a Kubernetes service to the internet is by creating a `LoadBalancer` service — which provisions one AWS load balancer **per service**. This is expensive and unmanageable at scale.

An **Ingress Controller** is a single reverse-proxy that:
1. Provisions **one** AWS Network Load Balancer (NLB) for the entire cluster
2. Reads `Ingress` resource rules defined in your manifests
3. Routes traffic to the correct backend service based on hostname and path

**In this project:**
- The NGINX Ingress Controller receives all external HTTP traffic
- It reads the `Ingress` resource defined in `helm/go-web-app-chart/templates/ingress.yaml`
- It routes traffic for `go-web-app.local` → `go-web-app` Service → Go app Pods

---

## Step 1 — Deploy the NGINX Ingress Controller

Apply the official AWS-specific manifest for the NGINX Ingress Controller:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.11.1/deploy/static/provider/aws/deploy.yaml
```

**What this command creates:**
- Namespace: `ingress-nginx`
- `Deployment`: The NGINX Ingress Controller pod
- `Service` (type `LoadBalancer`): Triggers AWS to provision a Network Load Balancer
- `IngressClass`: Registers `nginx` as a valid ingress class in the cluster
- `ClusterRole` / `ClusterRoleBinding`: RBAC permissions for the controller

---

## Step 2 — Wait for the Pods to Start

Check that the Ingress Controller pod is running:

```bash
kubectl get pods -n ingress-nginx
```

**Expected output:**

```
NAME                                        READY   STATUS    RESTARTS   AGE
ingress-nginx-controller-xxxxxxxxxx-xxxxx   1/1     Running   0          2m
```

Wait until `STATUS` is `Running` and `READY` is `1/1`. This typically takes 1–2 minutes.

---

## Step 3 — Get the Load Balancer Hostname

Retrieve the external hostname of the AWS Network Load Balancer:

```bash
kubectl get svc -n ingress-nginx
```

**Expected output:**

```
NAME                                 TYPE           CLUSTER-IP     EXTERNAL-IP                                                          PORT(S)
ingress-nginx-controller             LoadBalancer   10.100.xx.xx   xxxxxx.elb.us-east-1.amazonaws.com   80:31xxx/TCP,443:32xxx/TCP
ingress-nginx-controller-admission   ClusterIP      10.100.xx.xx   <none>                                                               443/TCP
```

> ⏳ The `EXTERNAL-IP` field may show `<pending>` for a few minutes while AWS provisions the NLB. Run the command again until a hostname appears.

Copy the `EXTERNAL-IP` hostname — you will need it for DNS configuration.

---

## Step 4 — Configure Local DNS (for Testing)

Since this project uses `go-web-app.local` as the hostname (not a real domain), you need to map it in your local `/etc/hosts` file.

**On Linux / macOS:**

```bash
sudo nano /etc/hosts
```

**On Windows** (open Notepad as Administrator, then open):
```
C:\Windows\System32\drivers\etc\hosts
```

Add the following line at the bottom, replacing `<NLB-IP>` with the IP address that the NLB hostname resolves to:

```
<NLB-IP>    go-web-app.local
```

**To find the NLB IP address:**

```bash
# Linux/macOS
nslookup xxxxxx.elb.us-east-1.amazonaws.com

# Or use dig
dig xxxxxx.elb.us-east-1.amazonaws.com
```

> **Note for production:** For a real domain, you would create a DNS CNAME or ALIAS record pointing your domain to the NLB hostname instead of editing `/etc/hosts`.

---

## Understanding the Ingress Resource

The Ingress resource for the application is defined in `helm/go-web-app-chart/templates/ingress.yaml`:

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

**What this says:**
- Use the `nginx` ingress class (the controller we just deployed)
- For the hostname `go-web-app.local`...
- Route all requests matching path `/` (prefix)...
- To the `go-web-app` Service on port `80`

This resource is **automatically applied by Argo CD** when it syncs the Helm chart. You do not need to apply it manually.

---

## Traffic Flow Diagram

```
Internet
    │
    ▼
AWS Network Load Balancer (NLB)       ← Provisioned by the NGINX Ingress Service
    │
    ▼
NGINX Ingress Controller Pod          ← Reads Ingress rules
(namespace: ingress-nginx)
    │  matches: host=go-web-app.local, path=/
    ▼
go-web-app Service (ClusterIP)        ← Internal service routing
(namespace: default)
    │
    ▼
go-web-app Pod(s)                     ← Your running Go application
(port 8080)
```

---

## Common Issues

| Issue | Likely Cause | Fix |
|---|---|---|
| Pod stuck in `Pending` | Not enough node capacity | Check `kubectl describe pod <pod-name> -n ingress-nginx` |
| `EXTERNAL-IP` stuck as `<pending>` | AWS taking time or permission issue | Wait up to 5 minutes; check IAM permissions for EC2 load balancers |
| `404 Not Found` from NLB | No Ingress resource applied yet | Deploy the app via Helm/Argo CD first |
| Can't resolve `go-web-app.local` | Hosts file not updated | Check your `/etc/hosts` or Windows hosts file |

---

## What's Next?

The NGINX Ingress Controller is deployed and the NLB is provisioned. Next, install **Argo CD** to enable GitOps-based continuous deployment.

- **Next:** [07 — Argo CD Setup](07-argocd-setup.md)
- **Related:** [09 — Helm Chart Guide](09-helm-chart-guide.md) — how the Ingress resource is defined

---

*[← EKS Cluster Setup](05-eks-cluster-setup.md) | [Documentation Hub](index.md) | [Next: Argo CD Setup →](07-argocd-setup.md)*
