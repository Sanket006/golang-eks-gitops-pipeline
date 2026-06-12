# 04 — AWS Prerequisites

> **Navigation:** [← Docker Guide](03-docker-guide.md) | [Documentation Hub](index.md) | **Next →** [EKS Cluster Setup](05-eks-cluster-setup.md)

---

## Overview

Before creating an Amazon EKS cluster, you need three CLI tools installed and configured on your local machine. This guide walks you through installing and verifying each one.

> **Estimated time:** 15–30 minutes

---

## Tools You Need

| Tool | Purpose |
|---|---|
| `kubectl` | Interact with any Kubernetes cluster — deploy apps, inspect pods, view logs |
| `eksctl` | Create, manage, and delete Amazon EKS clusters with a single command |
| `aws` (AWS CLI) | Authenticate with AWS services and manage resources |

---

## Step 1 — Install `kubectl`

`kubectl` is the standard Kubernetes command-line tool. You use it to talk to the cluster after it is created.

### Installation

Follow the official AWS guide for your operating system:
📖 [Installing or updating kubectl](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html)

**Quick install on Linux/macOS:**

```bash
# Download the latest version
curl -O https://s3.us-west-2.amazonaws.com/amazon-eks/1.29.0/2024-01-04/bin/linux/amd64/kubectl

# Make it executable
chmod +x ./kubectl

# Move it to your PATH
sudo mv ./kubectl /usr/local/bin/kubectl
```

**Verify the installation:**

```bash
kubectl version --client
# Expected: Client Version: v1.29.x
```

---

## Step 2 — Install `eksctl`

`eksctl` is a purpose-built CLI for Amazon EKS. It automates the complex AWS tasks involved in cluster creation (VPC setup, IAM roles, node groups, etc.) into a single command.

### Installation

📖 [Installing or updating eksctl](https://docs.aws.amazon.com/eks/latest/userguide/eksctl.html)

**Quick install on Linux:**

```bash
# Download and extract
curl --silent --location "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp

# Move to PATH
sudo mv /tmp/eksctl /usr/local/bin
```

**Quick install on macOS (Homebrew):**

```bash
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl
```

**Verify the installation:**

```bash
eksctl version
# Expected: 0.175.x or higher
```

---

## Step 3 — Install the AWS CLI

The AWS CLI lets you authenticate with AWS and interact with all AWS services from your terminal.

### Installation

📖 [Installing, updating, and uninstalling the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html)

**Quick install on Linux:**

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

**Quick install on macOS:**

```bash
brew install awscli
```

**Quick install on Windows:**
Download the MSI installer from: https://awscli.amazonaws.com/AWSCLIV2.msi

**Verify the installation:**

```bash
aws --version
# Expected: aws-cli/2.x.x
```

---

## Step 4 — Configure AWS Credentials

Run the AWS configuration wizard:

```bash
aws configure
```

You will be prompted for four values:

```
AWS Access Key ID [None]:      <paste your Access Key ID>
AWS Secret Access Key [None]:  <paste your Secret Access Key>
Default region name [None]:    us-east-1
Default output format [None]:  json
```

### How to Get Your AWS Credentials

1. Log in to the [AWS Management Console](https://console.aws.amazon.com/)
2. Click your account name (top right) → **Security credentials**
3. Scroll to **Access keys** → **Create access key**
4. Choose **CLI** as the use case
5. Download or copy the **Access Key ID** and **Secret Access Key**

> ⚠️ **Security Warning:** Never commit your AWS credentials to Git. They are stored in `~/.aws/credentials` on your machine and should stay there.

📖 [Quick configuration with `aws configure`](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-config)

---

## Step 5 — Verify Everything Works

Run this final verification to confirm all three tools are installed and AWS is configured correctly:

```bash
kubectl version --client
eksctl version
aws --version
aws sts get-caller-identity
```

The `aws sts get-caller-identity` command should return your AWS account details:

```json
{
    "UserId": "AIDAXXXXXXXXXXXXXXXXX",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/your-username"
}
```

If this returns your account information, you are fully configured and ready to create an EKS cluster.

---

## Required AWS Permissions

Your AWS user or IAM role needs the following permissions to create and manage an EKS cluster:

- `eks:*` — Create, describe, and manage EKS clusters
- `ec2:*` — Create VPCs, subnets, security groups, and EC2 instances (for node groups)
- `iam:*` — Create IAM roles for the cluster and node group
- `cloudformation:*` — `eksctl` uses CloudFormation stacks internally

> For a production setup, use a dedicated IAM role with only the minimum required permissions. For a personal learning project, an admin user is acceptable.

---

## Common Issues

| Issue | Likely Cause | Fix |
|---|---|---|
| `aws: command not found` | AWS CLI not in PATH | Re-run the install and check `/usr/local/bin` |
| `Unable to locate credentials` | `aws configure` not run | Run `aws configure` and enter your keys |
| `InvalidClientTokenId` | Wrong or expired access key | Generate a new access key in the AWS console |
| `eksctl: command not found` | eksctl not in PATH | Move the binary to `/usr/local/bin` |

---

## What's Next?

All tools are installed and configured. You are ready to create your EKS cluster.

- **Next:** [05 — EKS Cluster Setup](05-eks-cluster-setup.md)

---

*[← Docker Guide](03-docker-guide.md) | [Documentation Hub](index.md) | [Next: EKS Cluster Setup →](05-eks-cluster-setup.md)*
