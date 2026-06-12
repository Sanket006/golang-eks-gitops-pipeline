# 📐 Project Architecture Diagrams

This document contains detailed, professionally styled architecture diagrams for the **Golang EKS GitOps Pipeline** project.

To make this simple to understand, each section contains a **Visual Path**, a **Real-World Analogy**, and a **Simple Step-by-Step Guide**.

---

## ☁️ 1. AWS Architecture Diagram

This diagram shows how a user's request enters AWS and reaches the Go application inside the private cluster network.

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

### 🔍 Easy-to-Understand Breakdown

#### ➡️ The Simple Traffic Path
`User (Browser)` ──> `AWS Load Balancer` ──> `Ingress (Router) Pod` ──> `Go Web App Pod`

#### 🏢 The Office Complex Analogy
*   **AWS VPC (Private Subnet)** is like a **Highly Secure Corporate Building**. No one can walk straight into the offices without passing security.
*   **AWS NLB (Load Balancer)** is the **Front Gate Security Guard**. They greet visitors and direct them to the lobby.
*   **NGINX Ingress Pod** is the **Lobby Receptionist**. They look at your request (e.g., "I want to visit the web app") and tell you exactly which room to go to.
*   **Go Web App Pod** is the **Office Room** where the workers reside and answer your questions.

#### 🚶 Step-by-Step Traffic Flow
1.  **Request**: You type the app address in your browser.
2.  **Gatekeeper**: AWS Load Balancer receives the request and sends it to the EKS cluster.
3.  **Receptionist**: NGINX Ingress reads the request and forwards it to the Go Web App.
4.  **Response**: The Go App processes the request and sends the website back to your browser.

---

## ⚙️ 2. CI/CD Workflow Diagram (GitOps)

This diagram shows the automated build, test, and release cycle that takes code from a developer's computer to the running cluster.

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

### 🔍 Easy-to-Understand Breakdown

#### ➡️ The Simple Code Path
`Developer` ──> `GitHub (Code)` ──> `GitHub Actions (Build & Package)` ──> `Docker Hub (Store)` ──> `Argo CD` ──> `EKS Cluster (Deploy)`

#### 🏭 The Toy Factory Analogy
*   **GitHub Repository** is the **Blueprint Office**. When a designer changes the blueprint (code), the factory starts.
*   **GitHub Actions (CI)** is the **Assembly Line & Quality Inspector**. It tests the parts, compiles the toy, packages it into a shipping box (Docker Image), and tags it with a serial number.
*   **Docker Hub** is the **Distribution Warehouse** where finished shipping boxes are stored.
*   **Argo CD (GitOps)** is the **Delivery Agent**. It constantly looks at the blueprints, compares them to the store shelves, and delivers the new box from the warehouse to the store (EKS Cluster) automatically.

#### 🚶 Step-by-Step Pipeline Flow
1.  **Code**: Developer pushes a code update to GitHub.
2.  **Verify**: GitHub Actions compiles the code and runs tests to ensure nothing is broken.
3.  **Package**: The code is packaged into a container image and uploaded to Docker Hub.
4.  **Register**: GitHub Actions writes the new version tag back into Git (Helm chart).
5.  **Deploy**: Argo CD notices the updated tag, pulls the new image from Docker Hub, and replaces the old running app with zero downtime.

---

## ☸️ 3. Kubernetes Architecture Diagram

This diagram shows how resources are organized by "namespaces" (virtual folders) inside the EKS cluster, and how they connect.

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

### 🔍 Easy-to-Understand Breakdown

#### 🏢 The Department Store Analogy
*   **Namespaces** are like **Departments** in a department store (e.g., Clothing, Electronics, Management). They keep things organized and separated.
    *   `ingress-nginx` is the **Customer Entrance & Escalators**.
    *   `default` is the **Sales Floor** where the main products (Go Web App) live.
    *   `argocd` is the **Manager's Office** behind the scenes.
*   **Ingress** is the **Signpost** directing customers (e.g., "Go to floor 2 for clothing").
*   **Service** is the **Checkout Counter** routing requests to the cashiers.
*   **Pod** is the **Cashier / Worker** who actually does the scanning and serves the customer.

#### 🚶 Step-by-Step Routing Flow
1.  **Entry**: External request comes through the Load Balancer to the **Ingress Controller Pod** (`ingress-nginx` namespace).
2.  **Direction**: The Ingress Controller reads the domain name and passes it to the **Go Web App Service** (`default` namespace).
3.  **Work**: The Service forwards the request to the running **Go Web App Pod** (listening on port 8080) which serves the web page.
