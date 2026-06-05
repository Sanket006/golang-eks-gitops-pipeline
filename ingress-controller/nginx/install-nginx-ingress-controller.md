# NGINX Ingress Controller — Installation on AWS

This guide walks you through deploying the **NGINX Ingress Controller** on an Amazon EKS cluster. The Ingress Controller manages external HTTP/HTTPS access to services inside your Kubernetes cluster.

---

## Step 1: Deploy the NGINX Ingress Controller

Apply the official NGINX Ingress Controller manifest for AWS using the following command:

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.11.1/deploy/static/provider/aws/deploy.yaml
```

This will:
- Create the `ingress-nginx` namespace
- Deploy the Ingress Controller as a Kubernetes `Deployment`
- Create an AWS **Network Load Balancer (NLB)** to expose the controller externally

---

## Step 2: Verify the Deployment

Check that the Ingress Controller pods are running:

```bash
kubectl get pods -n ingress-nginx
```

Retrieve the external Load Balancer hostname assigned by AWS:

```bash
kubectl get svc -n ingress-nginx
```

> ⏳ It may take a few minutes for the AWS Load Balancer to be provisioned and become available.

---

## Next Steps

Once the Ingress Controller is running, you can:

- Deploy your application using a Kubernetes `Ingress` resource
- Configure DNS to point your domain to the Load Balancer hostname
- Proceed to [Install Argo CD](../../gitops/argocd/install-argocd.md) for GitOps-based CD
