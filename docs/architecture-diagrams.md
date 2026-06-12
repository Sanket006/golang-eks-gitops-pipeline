# 📐 Project Architecture Diagrams

This document contains simple, clear, and professionally styled architecture diagrams for the **Golang EKS GitOps Pipeline** project.

---

## ☁️ 1. AWS Architecture Diagram

This diagram shows how external traffic enters AWS and is routed to the Go application and Argo CD dashboard inside the private cluster network.

```mermaid
graph TD
    %% Define Styles
    classDef client fill:#EEF2F6,stroke:#4A5568,stroke-width:2px,color:#2D3748;
    classDef aws fill:#EBF8FF,stroke:#3182CE,stroke-width:2px,color:#2B6CB0;
    classDef public fill:#F0FFF4,stroke:#38A169,stroke-width:1.5px,color:#276749,stroke-dasharray: 5 5;
    classDef private fill:#FFF5F5,stroke:#E53E3E,stroke-width:1.5px,color:#9B2C2C,stroke-dasharray: 5 5;
    classDef node fill:#FFF,stroke:#DD6B20,stroke-width:2px,color:#DD6B20;
    classDef pod fill:#FFF,stroke:#319795,stroke-width:1.5px,color:#234E52;

    %% Nodes
    User([User / Client]):::client
    GitHub[(GitHub Repository)]:::client
    DockerHub[(Docker Hub)]:::client

    subgraph AWS ["AWS Cloud (us-east-1)"]
        subgraph VPC ["Virtual Private Cloud (VPC)"]
            
            subgraph PublicSubnet ["Public Subnets (Multi-AZ)"]
                NLB[AWS Network Load Balancer]:::aws
                ArgoLB[Argo CD Load Balancer]:::aws
            end

            subgraph PrivateSubnet ["Private Subnets (Multi-AZ)"]
                subgraph EKS ["EKS Cluster (demo-cluster)"]
                    ControlPlane[EKS Managed Control Plane]:::aws
                    
                    subgraph MNG ["Managed Node Group (EC2)"]
                        Node1[Worker Node 1]:::node
                        Node2[Worker Node 2]:::node
                    end
                end
            end
            
        end
    end

    %% Workloads inside Nodes
    IngressPod[NGINX Ingress Pod]:::pod
    AppPod[Go Web App Pod]:::pod
    ArgoPod[Argo CD Pod]:::pod

    Node1 --- IngressPod
    Node1 --- AppPod
    Node2 --- ArgoPod

    %% Connections
    User -->|HTTP/HTTPS| NLB
    User -->|Management| ArgoLB
    
    NLB -->|Target Group| IngressPod
    ArgoLB -->|Target Group| ArgoPod
    IngressPod -->|ClusterIP Service| AppPod
    
    ArgoPod -.->|GitOps Sync| AppPod
    ArgoPod -.->|Watch Manifests| GitHub
    ArgoPod -.->|Pull Images| DockerHub

    %% Apply Styles
    class AWS,VPC,ControlPlane aws;
    class PublicSubnet public;
    class PrivateSubnet private;
```

### Traffic Routing Workflow (How Users Access the App)
1. **Request Initiation**: A User enters the address `http://go-web-app.local` in their web browser.
2. **DNS Resolution**: The hostname resolves to the **AWS Network Load Balancer (NLB)** public IP address.
3. **Ingress Entry**: The AWS NLB receives the request on port 80 (HTTP) or 443 (HTTPS) and routes it inside the private network to the **NGINX Ingress Controller Pod** running on EKS.
4. **Path & Host Routing**: The NGINX Ingress Controller inspects the HTTP headers (e.g., Host: `go-web-app.local`) and matches it against the rules in the `go-web-app` Ingress manifest.
5. **Service Forwarding**: Traffic is forwarded to the cluster-internal **Go Web App Service** (Type: `ClusterIP` on port 80).
6. **Pod Target**: The service forwards the traffic to the destination target port `8080` on the **Go Web App Pod**.
7. **Response**: The Go application handles the request (serving pages like `home`, `about`, `courses`, `contact`) and returns the HTML page back to the User.

---

## ⚙️ 2. CI/CD Workflow Diagram (GitOps)

This diagram shows the automated build, test, and release cycle from a code commit to the final deployment.

```mermaid
graph LR
    %% Define Styles
    classDef dev fill:#EEF2F6,stroke:#4A5568,stroke-width:2px,color:#2D3748;
    classDef git fill:#FFF5F5,stroke:#E53E3E,stroke-width:2px,color:#9B2C2C;
    classDef gha fill:#F0FFF4,stroke:#38A169,stroke-width:2px,color:#276749;
    classDef docker fill:#EBF8FF,stroke:#3182CE,stroke-width:2px,color:#2B6CB0;
    classDef argo fill:#FEFCBF,stroke:#D69E2E,stroke-width:2px,color:#744210;
    classDef eks fill:#EDF2F7,stroke:#4A5568,stroke-width:2px,color:#2D3748;

    Developer([Developer]):::dev
    GitHub[(GitHub Repo)]:::git
    DockerHub[(Docker Hub)]:::docker

    subgraph CI ["GitHub Actions (CI/CD Pipeline)"]
        Trigger{Push to main?}:::gha
        Build[Job: Build & Test]:::gha
        Lint[Job: Code Quality]:::gha
        Push[Job: Build & Push Image]:::gha
        Update[Job: Update Helm Tag]:::gha
    end

    subgraph CD ["GitOps Deployment (CD)"]
        ArgoCD[Argo CD Application Controller]:::argo
    end

    subgraph Cluster ["Target Environment (EKS)"]
        AppPod[Go Web App Pod]:::eks
    end

    Developer -->|1. Commit & Push| GitHub
    GitHub -->|2. Trigger Workflow| Trigger
    
    Trigger -->|Parallel Job| Build
    Trigger -->|Parallel Job| Lint
    
    Build -->|3. On Success| Push
    Push -->|4. Push Image with Run ID| DockerHub
    Push -->|5. On Success| Update
    
    Update -->|6. Commit New Tag| GitHub
    
    GitHub -.->|7. Poll for Updates| ArgoCD
    DockerHub -.->|8. Fetch New Image| ArgoCD
    ArgoCD -->|9. Reconcile / Deploy| AppPod
```

### GitOps Continuous Deployment Workflow (How Code is Released)
1. **Code Commit**: A Developer pushes a code commit or code fix to the `main` branch of the GitHub repository.
2. **CI Pipeline Trigger**: GitHub Actions automatically triggers the `CI/CD` workflow.
3. **Build & Test**: GitHub Actions runs the `build` job (compiling the Go binary and running tests) and the `code-quality` job (running `golangci-lint`) in parallel.
4. **Publish Image**: Once tests pass, the `push` job builds a Docker container image using a secure, multi-stage build and uploads it to **Docker Hub** tagged with the unique GitHub Action `run_id`.
5. **Helm Chart Update**: The `update-newtag-in-helm-chart` job edits the `helm/go-web-app-chart/values.yaml` file, updating the image tag to match the new `run_id`, and commits this change back to the repository.
6. **GitOps Detection**: The **Argo CD Application Controller** in the EKS cluster detects that the repository's target state has updated (the Helm tag has changed).
7. **Reconciliation & Deployment**: Argo CD triggers an automatic Sync, pulls the corresponding new Docker image from Docker Hub, and performs a rolling update on EKS with zero downtime.

---

## ☸️ 3. Kubernetes Architecture Diagram

This diagram displays the namespace separation and service-to-pod routing layout inside EKS.

```mermaid
graph TD
    %% Define Styles
    classDef client fill:#EEF2F6,stroke:#4A5568,stroke-width:2px,color:#2D3748;
    classDef nsbfill fill:#EDF2F7,stroke:#4A5568,stroke-width:2px,stroke-dasharray: 5 5;
    classDef svc fill:#FAF5FF,stroke:#805AD5,stroke-width:2px,color:#553C9A;
    classDef pod fill:#E6FFFA,stroke:#319795,stroke-width:2px,color:#234E52;
    classDef ing fill:#FFF5F5,stroke:#E53E3E,stroke-width:2px,color:#9B2C2C;

    Client([Client / User]):::client
    AWS_NLB[AWS Network Load Balancer]:::client
    Argo_LB[AWS Argo CD Load Balancer]:::client

    subgraph EKS ["EKS Cluster Namespace Topology"]
        
        subgraph IngressNS ["Namespace: ingress-nginx"]
            IngressSvc[Service: ingress-nginx-controller <br> Type: LoadBalancer]:::svc
            IngressPod[NGINX Ingress Controller Pod]:::pod
        end

        subgraph ArgoNS ["Namespace: argocd"]
            ArgoSvc[Service: argocd-server <br> Type: LoadBalancer]:::svc
            ArgoServer[Argo CD Server Pod]:::pod
            ArgoController[Argo CD Controller Pod]:::pod
        end

        subgraph AppNS ["Namespace: default"]
            AppIngress[Ingress: go-web-app <br> Host: go-web-app.local]:::ing
            AppSvc[Service: go-web-app <br> Type: ClusterIP]:::svc
            AppPod[Pod: go-web-app <br> Port: 8080]:::pod
        end

    end

    Client -->|http://go-web-app.local| AWS_NLB
    Client -->|http://argocd.local| Argo_LB
    
    AWS_NLB -->|Port 80/443| IngressSvc
    IngressSvc --> IngressPod
    
    Argo_LB --> ArgoSvc
    ArgoSvc --> ArgoServer
    
    IngressPod -->|Inspect Host Header| AppIngress
    AppIngress -->|Forward Traffic| AppSvc
    AppSvc -->|Load Balances| AppPod
    
    ArgoController -.->|GitOps Reconcile| AppPod
    ArgoController -.->|Deploy & Watch| AppSvc
    ArgoController -.->|Deploy & Watch| AppIngress

    class IngressNS,ArgoNS,AppNS nsbfill;
```

### Component Details
- **NGINX Ingress**: Directs incoming requests based on routing rules.
- **Go Web App Service**: Acts as a stable internal load balancer for the application pods.
- **Argo CD Controller**: Continuously syncs the application pods with the Git repository's Helm chart.
