# 05 — EKS Cluster Setup

> **Navigation:** [← AWS Prerequisites](04-aws-prerequisites.md) | [Documentation Hub](index.md) | **Next →** [NGINX Ingress Controller](06-nginx-ingress-controller.md)

---

## Overview

This guide walks you through creating an **Amazon EKS (Elastic Kubernetes Service)** cluster using `eksctl`. EKS is a fully managed Kubernetes service — AWS handles the Kubernetes control plane (API server, etcd, scheduler) while you manage your worker nodes.

> **Estimated time:** 15–20 minutes (most of it waiting for AWS to provision resources)
>
> **Prerequisites:** Complete [04 — AWS Prerequisites](04-aws-prerequisites.md) first.

---

## What `eksctl` Creates

When you run the create command below, `eksctl` automatically provisions:

- A **VPC** with public and private subnets across two availability zones
- An **Internet Gateway** for outbound traffic
- A **NAT Gateway** for private subnet outbound access
- The **EKS control plane** (API server, etcd — managed by AWS)
- A **Managed Node Group** — EC2 instances that become your Kubernetes worker nodes
- All required **IAM roles** for the cluster and node group
- Updated **kubeconfig** on your local machine so `kubectl` connects to the new cluster

---

## Step 1 — Create the EKS Cluster

Run the following command to create a cluster named `demo-cluster` in the `us-east-1` region:

```bash
eksctl create cluster --name demo-cluster --region us-east-1
```

> ⏳ **This takes 10–15 minutes.** `eksctl` uses AWS CloudFormation stacks internally. You can watch the progress in your terminal or in the [CloudFormation console](https://console.aws.amazon.com/cloudformation).

**What you will see in the terminal:**

```
2024-xx-xx xx:xx:xx [ℹ]  eksctl version 0.175.0
2024-xx-xx xx:xx:xx [ℹ]  using region us-east-1
2024-xx-xx xx:xx:xx [ℹ]  setting availability zones to [us-east-1a us-east-1b]
2024-xx-xx xx:xx:xx [ℹ]  creating VPC stack for cluster "demo-cluster"
...
2024-xx-xx xx:xx:xx [✔]  EKS cluster "demo-cluster" in "us-east-1" region is ready
```

---

## Step 2 — Verify the Cluster

Once `eksctl` completes, verify that your nodes are running and `kubectl` is connected:

```bash
kubectl get nodes
```

**Expected output:**

```
NAME                          STATUS   ROLES    AGE   VERSION
ip-192-168-xx-xx.ec2.internal Ready    <none>   2m    v1.29.x
ip-192-168-xx-xx.ec2.internal Ready    <none>   2m    v1.29.x
```

Both nodes should show `Ready`. If they show `NotReady`, wait another minute and check again.

**Verify cluster info:**

```bash
kubectl cluster-info
```

---

## Step 3 — Explore the Cluster

Check the system namespaces and pods that AWS has pre-installed:

```bash
# List all namespaces
kubectl get namespaces

# List system pods (DNS, networking, etc.)
kubectl get pods -n kube-system
```

You should see pods like `coredns`, `aws-node` (VPC CNI), and `kube-proxy` running.

---

## Understanding the Cluster Architecture

```
AWS us-east-1
└── VPC (10.0.0.0/16)
    ├── Public Subnets  (us-east-1a, us-east-1b)
    │   ├── NAT Gateway
    │   └── (NLB is added later by NGINX Ingress)
    └── Private Subnets (us-east-1a, us-east-1b)
        └── EKS Managed Node Group
            ├── Worker Node 1 (EC2 t3.medium)
            └── Worker Node 2 (EC2 t3.medium)
```

- **Worker nodes live in private subnets** — they are not directly accessible from the internet.
- **Outbound traffic** from nodes flows: Node → NAT Gateway → Internet Gateway → Internet.
- **Inbound traffic** will flow through a Network Load Balancer (NLB) provisioned by the NGINX Ingress Controller in the next step.

---

## Customizing the Cluster (Optional)

If you need a different configuration, `eksctl` supports many options:

```bash
# Specify instance type and node count
eksctl create cluster \
  --name demo-cluster \
  --region us-east-1 \
  --nodegroup-name demo-ng \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3

# Use a config file for full control (recommended for production)
eksctl create cluster -f cluster-config.yaml
```

---

## Managing Your Cluster

### Update kubeconfig (if you switch machines)

If you need to reconnect `kubectl` to an existing cluster:

```bash
aws eks update-kubeconfig --name demo-cluster --region us-east-1
```

### List existing clusters

```bash
eksctl get cluster --region us-east-1
```

---

## Step 4 — Delete the Cluster (When Finished)

> ⚠️ **Warning:** This action permanently destroys the cluster and all workloads running on it. **This cannot be undone.** Ensure all important data is backed up before proceeding.

```bash
eksctl delete cluster --name demo-cluster --region us-east-1
```

`eksctl` will delete all CloudFormation stacks, EC2 instances, the VPC, and all associated resources. This takes approximately 5–10 minutes.

> 💡 **Cost Tip:** EKS charges for the control plane (~$0.10/hour) plus the EC2 instances in your node group. Always delete the cluster when not in use to avoid unnecessary charges.

---

## Common Issues

| Issue | Likely Cause | Fix |
|---|---|---|
| `Error: unable to determine AMI` | Unsupported region | Use `us-east-1`, `us-west-2`, or another supported region |
| Nodes stuck in `NotReady` | VPC networking issue | Check `kubectl describe node <name>` and VPC subnet routing |
| `error: You must be logged in to the server` | kubeconfig not updated | Run `aws eks update-kubeconfig --name demo-cluster --region us-east-1` |
| Cluster creation fails halfway | IAM permission missing | Ensure your AWS user has `eks:*`, `ec2:*`, `iam:*`, `cloudformation:*` |

---

## What's Next?

Your EKS cluster is running with worker nodes ready. The next step is to deploy the **NGINX Ingress Controller**, which provisions an AWS Network Load Balancer and handles routing traffic into the cluster.

- **Next:** [06 — NGINX Ingress Controller](06-nginx-ingress-controller.md)

---

*[← AWS Prerequisites](04-aws-prerequisites.md) | [Documentation Hub](index.md) | [Next: NGINX Ingress Controller →](06-nginx-ingress-controller.md)*
