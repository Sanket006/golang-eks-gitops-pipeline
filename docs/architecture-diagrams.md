# 📐 Project Architecture Diagrams

This document contains simple and clear architecture diagrams for the **Golang EKS GitOps Pipeline** project.

---

## ☁️ 1. AWS Architecture Diagram

This diagram shows how external traffic enters AWS and is routed to the Go application and Argo CD dashboard inside the private cluster network.

```mermaid
graph TD
    User([User]) -->|Access App| NLB[AWS NLB]
    User -->|Access Dashboard| ArgoLB[Argo CD Load Balancer]
    
    subgraph Amazon EKS Cluster
        NLB --> Ingress[NGINX Ingress Controller]
        ArgoLB --> ArgoServer[Argo CD Server]
        
        Ingress -->|Route Traffic| AppPod[Go Web App Pod]
    end
```

- **AWS NLB / Argo CD Load Balancer**: External entry points mapping client traffic to internal EKS services.
- **EKS Cluster**: Runs the application pods and GitOps continuous delivery server.

---

## ⚙️ 2. CI/CD Workflow Diagram (GitOps)

This diagram shows the automated build, test, and release cycle from a code commit to the final deployment.

```mermaid
graph LR
    Dev[Developer] -->|Push Code| Git[GitHub Repo]
    
    subgraph GitHub Actions (CI)
        Git --> Build[Build & Test]
        Build --> Push[Build & Push Image]
        Push --> Update[Update Helm Chart Tag]
    end
    
    Push -->|Upload Image| DH[Docker Hub]
    Update -->|Commit Tag| Git
    
    Git -->|Reconcile| Argo[Argo CD]
    DH -->|Pull Image| Argo
    Argo -->|Deploy| EKS[EKS Cluster]
```

- **GitHub Actions**: Runs unit tests, compiles the binary, pushes the Docker image to Docker Hub, and updates the Helm tag in Git.
- **Argo CD**: Automatically detects the new tag in Git, pulls the corresponding image from Docker Hub, and deploys it.

---

## ☸️ 3. Kubernetes Architecture Diagram

This diagram displays the namespace separation and service-to-pod routing layout inside EKS.

```mermaid
graph TD
    User([User]) --> Ingress[NGINX Ingress]
    
    subgraph default Namespace (Application)
        Ingress --> Service[Go Web App Service]
        Service --> Pod[Go Web App Pod]
    end
    
    subgraph argocd Namespace (GitOps)
        Argo[Argo CD Controller] -->|Deploy & Sync| Pod
    end
```

- **NGINX Ingress**: Directs incoming requests based on routing rules.
- **Go Web App Service**: Acts as a stable internal load balancer for the application pods.
- **Argo CD Controller**: Continuously syncs the application pods with the Git repository's Helm chart.
