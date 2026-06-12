# Architecture Diagrams & Workflows

This document provides a detailed visual representation and workflow explanation for the three main sub-systems of the Go Web Application DevOps pipeline:
1. **AWS Infrastructure Architecture** (Traffic routing from the Client to the EKS Cluster)
2. **CI/CD Workflow** (Continuous Integration & GitOps Continuous Deployment Pipeline)
3. **Kubernetes Cluster Topology** (Namespace separation, Services, and Pods)

---

## ☁️ 1. AWS Architecture Diagram

The following diagram illustrates how external client traffic enters the AWS Cloud environment and reaches the web application running in Amazon EKS.

```mermaid
graph TD
    %% Styling definitions
    classDef client fill:#e3f2fd,stroke:#1e88e5,stroke-width:2px
    classDef aws fill:#ffe0b2,stroke:#fb8c00,stroke-width:2px
    classDef k8s fill:#e8f5e9,stroke:#43a047,stroke-width:2px
    classDef component fill:#ffffff,stroke:#757575,stroke-width:1px

    Client["Client (Browser)"]:::client
    
    subgraph AWS["AWS Cloud (us-east-1)"]
        NLB["AWS Network Load Balancer (NLB)"]:::aws
        
        subgraph EKS["Amazon EKS Cluster (demo-cluster)"]
            IngressCtrl["NGINX Ingress Controller"]:::k8s
            AppService["Go Web App Service"]:::k8s
            AppPods["Go Web App Pods"]:::k8s
        end
    end

    style AWS fill:#fff9c4,stroke:#fbc02d,stroke-width:2px
    style EKS fill:#e8f5e9,stroke:#43a047,stroke-width:2px

    Client -- "Request: go-web-app.local" --> NLB
    NLB -- "Forward Traffic" --> IngressCtrl
    IngressCtrl -- "Route based on Host rule" --> AppService
    AppService -- "Load balance requests" --> AppPods
```

### Traffic Flow Workflow
1. **Domain Resolution**: The Client resolves the domain name `go-web-app.local` directly to the IP address of the AWS Network Load Balancer (NLB). This resolution bypasses Route 53 and is handled via local hosts configuration or local DNS resolution.
2. **Edge Entry**: Traffic reaches the **AWS Network Load Balancer (NLB)** on port 80/443.
3. **Cluster Entrance**: The NLB routes external TCP traffic to the active **NGINX Ingress Controller** pods inside the EKS cluster.
4. **Ingress Rule Matching**: The Ingress Controller inspects the HTTP `Host` header (`go-web-app.local`) and matches it with the rules defined in the Ingress manifest.
5. **App Redirection**: Traffic is forwarded internally to the **Go Web App ClusterIP Service** (port 80).
6. **Backend Execution**: The Kubernetes Service distributes the traffic across the running **Go Web App Pods** (listening on container port 8080).

---

## ⚙️ 2. CI/CD Workflow Diagram (GitOps)

The following diagram shows the end-to-end continuous integration and deployment pipeline, from the developer's push to the final rolling update in EKS.

```mermaid
graph LR
    %% Styling definitions
    classDef actor fill:#e3f2fd,stroke:#1e88e5,stroke-width:2px
    classDef git fill:#ffebee,stroke:#e53935,stroke-width:2px
    classDef ci fill:#e8f5e9,stroke:#43a047,stroke-width:2px
    classDef registry fill:#e0f7fa,stroke:#00acc1,stroke-width:2px
    classDef cd fill:#fff3e0,stroke:#fb8c00,stroke-width:2px
    classDef cluster fill:#f3e5f5,stroke:#8e24aa,stroke-width:2px

    Dev["Developer"]:::actor
    GitHub["GitHub Code Repo"]:::git
    
    subgraph GHA["GitHub Actions Runner"]
        CI_Build["1. Build & Test"]:::ci
        CI_Lint["2. Lint Code"]:::ci
        CI_Push["3. Build & Push Image"]:::ci
        CI_UpdateTag["4. Update Helm Tag"]:::ci
    end

    DockerHub["Docker Hub"]:::registry
    HelmConfig["GitHub Helm Repo"]:::git
    ArgoCD["Argo CD (GitOps)"]:::cd
    EKS["Amazon EKS Cluster"]:::cluster

    style GHA fill:#f1f8e9,stroke:#7cb342,stroke-width:2px

    Dev -- "Push code to main" --> GitHub
    GitHub -- "Trigger Workflow" --> GHA
    
    CI_Build --> CI_Push
    CI_Lint --> CI_Push
    CI_Push -- "Push Image with Run ID tag" --> DockerHub
    CI_Push --> CI_UpdateTag
    CI_UpdateTag -- "Commit updated values.yaml" --> HelmConfig
    
    ArgoCD -- "Poll / Detect drift" --> HelmConfig
    ArgoCD -- "Sync State (kubectl apply)" --> EKS
    EKS -- "Pull new image" --> DockerHub
```

### Pipeline Workflow
1. **Developer Action**: A developer pushes code changes (excluding helm/k8s/docs paths) to the `main` branch on GitHub.
2. **Trigger**: GitHub Actions starts the CI/CD workflow runner.
3. **Continuous Integration (CI)**:
   - **Build & Test**: Compiles the Go binary and runs unit tests (`go test ./...`).
   - **Lint**: Runs `golangci-lint` to check code style and detect bugs.
4. **Build & Push Image**: After checks succeed, a multi-stage Docker image is built and pushed to **Docker Hub** using the unique GitHub Run ID (`${{github.run_id}}`) as the image tag.
5. **Declarative Tag Update**: The workflow automatically updates the `tag` field in `helm/go-web-app-chart/values.yaml` using a `sed` command and pushes the changes back to the repository.
6. **Continuous Delivery (CD)**:
   - **Drift Detection**: **Argo CD** detects that the active cluster state differs from the desired Git state (new Helm tag).
   - **Sync & Deploy**: Argo CD applies the updated Helm configuration to the **Amazon EKS Cluster**.
   - **Rolling Update**: The EKS nodes pull the newly tagged image from **Docker Hub** and perform a zero-downtime rolling update.

---

## ☸️ 3. Kubernetes Namespace & Architecture Diagram

The following diagram illustrates the deployment topology inside the Amazon EKS cluster, showing the namespace segregation and resource configurations.

```mermaid
graph TD
    %% Styling definitions
    classDef svc fill:#e8f5e9,stroke:#388e3c,stroke-width:2px
    classDef pod fill:#fff3e0,stroke:#f57c00,stroke-width:2px
    classDef ing fill:#f3e5f5,stroke:#7b1fa2,stroke-width:2px

    subgraph Cluster["Amazon EKS Cluster (demo-cluster)"]
        
        subgraph NS_Ingress["Namespace: ingress-nginx"]
            IngressSvc["NGINX Ingress Service (NLB)"]:::svc
            IngressPod["NGINX Ingress Pods"]:::pod
        end

        subgraph NS_App["Namespace: default"]
            AppIngress["Go Web App Ingress"]:::ing
            AppSvc["Go Web App Service"]:::svc
            AppPod["Go Web App Pod"]:::pod
        end

        subgraph NS_ArgoCD["Namespace: argocd"]
            ArgoSvc["Argo CD Server Service (LB)"]:::svc
            ArgoServer["Argo CD Server Pod"]:::pod
        end

    end

    %% Apply compatible styling to subgraphs
    style Cluster fill:#fafafa,stroke:#9e9e9e,stroke-width:2px
    style NS_Ingress fill:#e1f5fe,stroke:#0288d1,stroke-width:2px
    style NS_App fill:#e1f5fe,stroke:#0288d1,stroke-width:2px
    style NS_ArgoCD fill:#e1f5fe,stroke:#0288d1,stroke-width:2px

    IngressSvc --> IngressPod
    IngressPod -- "Inspect Host Header" --> AppIngress
    AppIngress -- "Route to Service" --> AppSvc
    AppSvc -- "Port 80 -> 8080" --> AppPod
    ArgoSvc --> ArgoServer
```

### Namespace Topology Workflow
1. **Namespace Isolation**: Resources are separated into logical zones for security, governance, and traffic control:
   - **`ingress-nginx` namespace**: Houses the ingress proxy engine.
   - **`argocd` namespace**: Houses the GitOps automation controllers.
   - **`default` namespace**: Houses the Go web application resources.
2. **Ingress Controller Service**: Exposes NGINX externally as a Network Load Balancer. It handles external requests and sends them to the **NGINX Ingress Pods**.
3. **Application Ingress Rules**: An Ingress resource configured for host `go-web-app.local` watches the HTTP host header and maps traffic to the application's service.
4. **App Service (ClusterIP)**: The internal **Go Web App Service** routes traffic from port `80` to target port `8080` of the application pods.
5. **App Pod**: The running application container receives the request and returns the static pages (`home.html`, `about.html`, etc.).
6. **Argo CD UI Access**: The **Argo CD Server Service** is patched to type `LoadBalancer` to expose the Argo CD administrative control panel dashboard externally.
