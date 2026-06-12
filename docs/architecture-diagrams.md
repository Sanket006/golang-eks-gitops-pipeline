# Architecture Diagrams

> **Navigation:** [Documentation Hub](index.md) | [← Project Overview](01-project-overview.md) | [Full Deployment Walkthrough →](10-full-deployment-walkthrough.md)

---

This document contains detailed, simple, and clear architecture diagrams for the Go Web Application DevOps Pipeline. It covers the AWS Cloud infrastructure, the CI/CD workflow (GitHub Actions & Argo CD), and the internal Kubernetes cluster topology and routing.

---

## 1. ☁️ AWS Cloud Infrastructure Architecture

This diagram shows how the infrastructure is provisioned on AWS, demonstrating the networking setup, external access, and resources residing within public and private subnets.

```mermaid
flowchart TD
    classDef default fill:#f9f9f9,stroke:#333,stroke-width:1px,color:#1a1a1a;
    classDef user fill:#e1f5fe,stroke:#03a9f4,stroke-width:2px,color:#01579b;
    classDef aws fill:#ffe0b2,stroke:#ff9800,stroke-width:2px,color:#e65100;
    classDef k8s fill:#e8f5e9,stroke:#4caf50,stroke-width:2px,color:#1b5e20;
    classDef ext fill:#eceff1,stroke:#607d8b,stroke-width:2px,color:#263238;

    User["🖥️ End User"]:::user
    DNS["🌐 Local DNS / Hosts<br>(go-web-app.local)"]:::user
    NLB["🔌 AWS Network Load Balancer (NLB)"]:::aws
    IGW["🛣️ Internet Gateway"]:::aws
    NAT["🛣️ NAT Gateway"]:::aws
    
    DockerHub["🐳 Docker Hub<br>(sanket006/go-web-app)"]:::ext
    GitHub["🐙 GitHub Repository"]:::ext

    subgraph VPC ["🛡️ AWS VPC (us-east-1)"]
        direction TB
        subgraph PubSubnet ["🌐 Public Subnets"]
            NLB
            NAT
        end
        subgraph PrivSubnet ["🔒 Private Subnets"]
            subgraph EKS ["📦 Amazon EKS Cluster (demo-cluster)"]
                subgraph Nodes ["⚙️ Managed Node Group (EC2)"]
                    Worker1["🖥️ Worker Node 1"]:::k8s
                    Worker2["🖥️ Worker Node 2"]:::k8s
                end
            end
        end
    end

    %% Connections
    User -->|1. Requests URL| DNS
    DNS -->|2. Resolves IP| NLB
    NLB -->|3. Routes HTTP Traffic| Worker1
    NLB -->|3. Routes HTTP Traffic| Worker2
    
    %% Outbound
    Nodes -->|4. Outbound Request| NAT
    NAT -->|5. Routes Outbound| IGW
    IGW -->|6. Pulls Docker Image| DockerHub
    Nodes -->|7. Fetches Git Manifests| GitHub
```

### 📋 AWS Infrastructure Workflow Explanation

1. **Local DNS Resolution**: The **🖥️ End User** requests `http://go-web-app.local` (1), which resolves locally (2) to the public IP address of the Network Load Balancer (NLB).
2. **Network Load Balancer (NLB)**: The **🔌 AWS NLB** acts as the entry point in the public subnet and routes the inbound HTTP traffic (3) to the worker nodes inside the private subnet.
3. **AWS VPC Isolation**: The worker nodes run inside the **🔒 Private Subnets** for security, with no direct access from the public internet.
4. **Outbound NAT Translation**: Any outbound request (4) originating from the worker nodes is translated by the **🛣️ NAT Gateway** in the public subnet.
5. **Internet Gateway**: The NAT Gateway forwards the outbound traffic through the **🛣️ Internet Gateway** (5) to access the external web.
6. **Docker Hub Registry**: Using the outbound route, the EKS worker nodes pull container images (6) from the external **🐳 Docker Hub** registry.
7. **Git Repository Sync**: The EKS cluster syncs with the **🐙 GitHub Repository** (7) to fetch Helm charts and deployment manifests.

---

## 2. ⚡ CI/CD GitOps Workflow

This diagram represents the step-by-step CI/CD automation pipeline. It covers continuous integration using GitHub Actions and continuous deployment using Argo CD (GitOps approach).

```mermaid
flowchart TD
    classDef dev fill:#e1f5fe,stroke:#03a9f4,stroke-width:2px,color:#01579b;
    classDef git fill:#eceff1,stroke:#607d8b,stroke-width:2px,color:#263238;
    classDef gh fill:#ffe8d6,stroke:#dd8c5a,stroke-width:2px,color:#5d4037;
    classDef docker fill:#e0f7fa,stroke:#00bcd4,stroke-width:2px,color:#006064;
    classDef argocd fill:#f3e5f5,stroke:#9c27b0,stroke-width:2px,color:#4a148c;
    classDef eks fill:#e8f5e9,stroke:#4caf50,stroke-width:2px,color:#1b5e20;

    Dev["💻 Developer"]:::dev
    Git["🐙 GitHub Repository<br>(main branch)"]:::git
    
    subgraph GHA ["⚡ GitHub Actions CI/CD Pipeline"]
        direction TB
        Job1["🛠️ Job: Build & Test<br>(go build & go test)"]:::gh
        Job2["🔍 Job: Code Quality<br>(golangci-lint)"]:::gh
        Job3["🐳 Job: Build & Push Image<br>(Docker Buildx)"]:::gh
        Job4["✍️ Job: Update Helm Chart<br>(sed image tag in values.yaml)"]:::gh
    end
    
    Registry["🐳 Docker Hub<br>(sanket006/go-web-app)"]:::docker
    Argo["☸️ Argo CD GitOps Controller"]:::argocd
    EKS["📦 Amazon EKS Cluster"]:::eks

    %% Flow lines
    Dev -->|1. Git Push| Git
    Git -->|2. Triggers Pipeline| GHA
    
    Job1 -->|Runs Parallel| Job2
    Job1 -->|3. On Success| Job3
    Job3 -->|4. Push Image| Registry
    Job3 -->|5. On Success| Job4
    Job4 -->|6. Commit & Push Tag Update| Git
    
    Argo -->|7. Polls & Detects Tag Change| Git
    Argo -->|8. Sync / Pulls Helm Chart| Git
    Registry -.->|9. Pulls New Image| EKS
    Argo -->|10. Deploys/Updates Workloads| EKS
```

### 📋 CI/CD Pipeline Workflow Explanation

1. **Code Commit**: A **💻 Developer** pushes a code change to the `main` branch of the **🐙 GitHub Repository** (excluding changes in `helm/`, `k8s/`, or `README.md`).
2. **GitHub Actions CI/CD Trigger**: The push triggers the GitHub Actions workflow.
3. **Continuous Integration (CI)**:
   - **🛠️ Build & Test**: Compiles the Go application and runs the unit tests (`go test ./...`).
   - **🔍 Code Quality**: Simultaneously runs static code analysis via `golangci-lint` to enforce style and detect code smell.
4. **Containerization**: Once tests succeed, the pipeline logs into Docker Hub and builds a lightweight, production-ready multi-stage Docker image tagged with the unique GitHub Action run ID.
5. **Docker Registry**: The image is pushed to **🐳 Docker Hub** under `sanket006/go-web-app:<run_id>`.
6. **GitOps Registry Update**: GitHub Actions updates the Helm chart's `values.yaml` in the GitHub repository, replacing the image tag value with the new `<run_id>` and pushing the change back to the repository.
7. **Argo CD Sync**: The **☸️ Argo CD GitOps Controller** (running in EKS) detects the new commit in Git, pulls the updated Helm chart, compares it with the cluster's current state, and pulls the new Docker image from Docker Hub to update the application workloads inside the **📦 Amazon EKS Cluster**.

---

## 3. ☸️ Kubernetes Namespace Topology & Traffic Routing

This diagram details the architecture inside the EKS cluster, illustrating the namespace topology, individual resources, and internal path-based traffic routing.

```mermaid
flowchart TD
    classDef ext fill:#eceff1,stroke:#607d8b,stroke-width:2px,color:#263238;
    classDef ing fill:#e1f5fe,stroke:#03a9f4,stroke-width:2px,color:#01579b;
    classDef app fill:#e8f5e9,stroke:#4caf50,stroke-width:2px,color:#1b5e20;
    classDef cd fill:#f3e5f5,stroke:#9c27b0,stroke-width:2px,color:#4a148c;

    User["🖥️ End User"]:::ext
    NLB["🔌 AWS Network Load Balancer (NLB)"]:::ext
    Git["🐙 GitHub Repository"]:::ext

    subgraph Cluster ["📦 Amazon EKS Cluster (demo-cluster)"]
        
        subgraph NS_Ingress ["🌐 Namespace: ingress-nginx"]
            direction TB
            IngSvc["🔌 NGINX Ingress Service<br>(LoadBalancer)"]:::ing
            IngCtrl["🔌 NGINX Ingress Controller<br>(Pod)"]:::ing
        end
        
        subgraph NS_App ["📦 Namespace: default"]
            direction TB
            IngRes["🕸️ Go Web App Ingress<br>(Ingress Resource)"]:::app
            AppSvc["🕸️ Go Web App Service<br>(ClusterIP)"]:::app
            AppPods["🐳 Go Web App Pods<br>(Deployment)"]:::app
        end
        
        subgraph NS_ArgoCD ["⚙️ Namespace: argocd"]
            direction TB
            ArgoController["☸️ Argo CD Controller"]:::cd
            ArgoServer["☸️ Argo CD Server"]:::cd
            ArgoRepo["☸️ Argo CD Repo Server"]:::cd
        end
        
    end

    %% External to Cluster Flow
    User -->|1. Requests go-web-app.local| NLB
    NLB -->|2. Routes Traffic| IngSvc
    IngSvc -->|3. Forwards HTTP| IngCtrl
    
    %% Ingress to App Flow
    IngCtrl -->|4. Inspects Rules| IngRes
    IngRes -->|5. Forwards to| AppSvc
    AppSvc -->|6. Load Balances to| AppPods
    
    %% Argo CD Flow
    ArgoServer <--> ArgoController
    ArgoController <--> ArgoRepo
    ArgoRepo -->|7. Fetches Manifests| Git
    ArgoController -->|8. Syncs/Applies Manifests| NS_App
```

### 📋 Kubernetes Routing Workflow Explanation

1. **Ingress Entry Point**: External user requests reach the AWS NLB, which forwards them to the **🔌 NGINX Ingress Service** (LoadBalancer) in the `ingress-nginx` namespace.
2. **NGINX Ingress Controller**: The request is passed to the **🔌 NGINX Ingress Controller** Pod, which acts as the cluster reverse-proxy.
3. **Ingress Rule Validation**: The Ingress Controller inspects the **🕸️ Go Web App Ingress** resource in the `default` namespace to resolve the routing rule for `go-web-app.local` at path `/`.
4. **Service Dispatching**: Once the rule matches, the controller forwards the request to the **🕸️ Go Web App Service** (ClusterIP) on port 80.
5. **App Pod Load Balancing**: The Service load balances traffic internally to the active **🐳 Go Web App Pod** (running the containerized Go application on port 8080).
6. **GitOps CD Sync**: Concurrently, the **☸️ Argo CD Controller** running in the `argocd` namespace monitors the git repositories, ensuring that any drift between the Git branch and the running Kubernetes resources is immediately rectified.

---

*[Documentation Hub](index.md) | [← Project Overview](01-project-overview.md) | [Full Deployment Walkthrough →](10-full-deployment-walkthrough.md)*
