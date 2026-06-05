# EKS Setup — Prerequisites

Before creating an Amazon EKS cluster, make sure the following CLI tools are installed and properly configured on your local machine.

---

## Required Tools

### 1. `kubectl` — Kubernetes CLI

`kubectl` is the command-line tool for interacting with Kubernetes clusters. You'll use it to deploy applications, inspect resources, and manage cluster workloads.

📖 **Installation Guide:** [Installing or updating kubectl](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html)

---

### 2. `eksctl` — EKS Management CLI

`eksctl` is a purpose-built CLI for Amazon EKS that simplifies cluster creation, configuration, and management by automating many underlying AWS tasks.

📖 **Installation Guide:** [Installing or updating eksctl](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html)

---

### 3. AWS CLI — AWS Command-Line Interface

The AWS CLI lets you interact with all AWS services, including Amazon EKS, directly from your terminal. After installation, it's recommended to configure it with your AWS credentials.

📖 **Installation Guide:** [Installing, updating, and uninstalling the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html)

📖 **Configuration Guide:** [Quick configuration with `aws configure`](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-config)

**Run the following to configure your AWS credentials:**

```bash
aws configure
```

You will be prompted to enter your:
- AWS Access Key ID
- AWS Secret Access Key
- Default region (e.g., `us-east-1`)
- Output format (e.g., `json`)

---

## ✅ Verification Checklist

Once all tools are installed, verify each one is working correctly:

```bash
kubectl version --client
eksctl version
aws --version
```

> After completing these steps, proceed to [install-eks-cluster.md](install-eks-cluster.md) to create your EKS cluster.
