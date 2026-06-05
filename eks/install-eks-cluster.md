# EKS Cluster Setup with `eksctl`

> ⚠️ **Before proceeding**, ensure all prerequisites are installed and configured. See [prerequisites.md](prerequisites.md) for the full setup guide.

---

## Create an EKS Cluster

Run the following command to create a new EKS cluster named `demo-cluster` in the `us-east-1` region:

```bash
eksctl create cluster --name demo-cluster --region us-east-1
```

> ⏳ Cluster creation typically takes **10–15 minutes**. `eksctl` will automatically configure your `kubeconfig` to connect `kubectl` to the new cluster.

**Verify the cluster is running:**

```bash
kubectl get nodes
```

---

## Delete the Cluster

When you are finished and want to remove the cluster and all its associated AWS resources, run:

```bash
eksctl delete cluster --name demo-cluster --region us-east-1
```

> ⚠️ **Warning:** This action is irreversible and will permanently delete the cluster and all workloads running on it. Ensure you have backed up any important data before proceeding.

---

## Next Steps

Once your cluster is up and running, proceed with:

1. [Install NGINX Ingress Controller](../ingress-controller/nginx/install-nginx-ingress-controller.md)
2. [Install Argo CD](../gitops/argocd/install-argocd.md)
