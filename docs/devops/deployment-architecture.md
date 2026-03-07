# DevOps & Deployment Architecture Specification

## 1. Overview
The Event Processing Platform embraces Cloud-Native principles, treating infrastructure as immutable and relying heavily on container orchestration. This specification details the journey of code from a developer's machine to the production Kubernetes cluster, emphasizing consistency, security, and observability.

## 2. Docker Container Strategy

### Service Image Design
All microservices (API Gateway, User, Order, Notification, Analytics) must be packaged into lightweight, secure Docker images adhering to the following standards:

1. **Multi-Stage Builds**:
   - *Builder Stage*: Contains the full compiler suite (e.g., `golang:1.21` or `node:20`), downloads dependencies, runs unit tests, and compiles the binary.
   - *Production Stage*: Uses a bare-minimum root filesystem (e.g., `alpine`, `distroless`, or `scratch`) containing *only* the compiled binary and essential CA certificates.
2. **Non-Root Execution**: Containers must define a `USER nonroot` explicitly in the Dockerfile. They must never run as root.
3. **Immutability**: Image tags use the exact Git SHA (e.g., `my-service:a1b2c3d`) rather than mutable tags like `latest`.
4. **Health Probes**: Every image exposes HTTP endpoints for Kubernetes:
   - `/health/liveness`: Indicates if the pod is deadlocked and needs restarting.
   - `/health/readiness`: Indicates if the pod has connected to its DB/Redis dependencies and is ready to receive traffic.

## 3. Local Development Environment

Developers orchestrate the entire platform locally using **Docker Compose** to guarantee environment parity (Dev/Prod equivalence).

### Docker Compose Architecture
- **Infrastructure Containers**: `postgres:15-alpine`, `redis:7-alpine`, `confluentinc/cp-kafka` (combined Zookeeper/Broker).
- **Service Containers**: Built locally via `docker-compose build`. Code directories are mounted via volumes for hot-reloading during active development.
- **Networking**: All services run on a custom bridge network (`platform_network`). Only the API Gateway exposes port `8080` to the host machine. Service-to-service communication uses Docker's internal DNS (e.g., `http://user-service:8080`).

## 4. Kubernetes Deployment Architecture

Production traffic is served from a managed Kubernetes cluster (e.g., EKS, GKE, AKS).

```mermaid
flowchart TD
    subgraph K8s Cluster [Kubernetes Cluster]
        ALB[Cloud Load Balancer] --> Ingress[Nginx Ingress Controller]
        
        subgraph Namespace: platform-prod
            Ingress --> GatewayPod[API Gateway Pods (HPA)]
            GatewayPod --> UserPod[User Service Pods (HPA)]
            GatewayPod --> OrderPod[Order Service Pods (HPA)]
            
            Kafka[(Managed MSK / Confluent)]
            Postgres[(Managed RDS)]
            Redis[(Managed ElastiCache)]
            
            OrderPod -->|Publishes| Kafka
            UserPod -->|Publishes| Kafka
            Kafka -->|Consumes| NotifPod[Notification Service Pods (HPA)]
            Kafka -->|Consumes| AnalyticsPod[Analytics Service Pods]
            
            OrderPod --> Postgres
            UserPod --> Postgres
            GatewayPod --> Redis
        end
    end
```

### K8s Manifest Standards (Helm)
Services are deployed via **Helm Charts** defining:
- **Deployments**: Describing replica counts, container images, and resource limits (e.g., Memory limits: `512Mi` to prevent OOM cascading failures).
- **Horizontal Pod Autoscalers (HPA)**: Automatically scaling pod replicas based on CPU target utilization (`targetCPUUtilizationPercentage: 70`).
- **Pod Disruption Budgets (PDB)**: Ensuring at least 50% of a service's replicas remain available during cluster node upgrades.

## 5. CI/CD Pipeline

The pipeline is managed via **GitHub Actions** (or Gitlab CI). It enforces strict progression from code commit to production deployment.

### Continuous Integration (CI) - Triggered on Pull Request
1. **Linting & Formatting**: Enforce code style.
2. **Static Application Security Testing (SAST)**: Run tools like `SonarQube` or `tfsec` to catch hardcoded secrets or raw SQL injection vectors.
3. **Unit & Integration Testing**: Spin up Ephemeral PostgreSQL and Redis instances via Testcontainers. Run test suites.
4. **Build Image**: Compile and build the Docker image.
5. **Image Scanning**: Scan the final image layers with `Trivy` for CVEs. Fail the build on `CRITICAL` vulnerability severity.

### Continuous Deployment (CD) - Triggered on Merge to Main
1. **Push Image**: Tag image with the Git SHA and push to the Container Registry (e.g., ECR, GAR).
2. **Update Helm Values**: Automatically update the `image.tag` value in the separate GitOps configuration repository.
3. **ArgoCD (GitOps Pull Pattern)**: ArgoCD detects the change in the config repository and initiates a rolling update in the Kubernetes cluster without exposing cluster credentials to the CI pipeline.

## 6. Secrets Management

- **The Rule**: Secrets (Database passwords, JWT Private Keys, API Keys) are NEVER stored in Git.
- **Implementation**: We integrate **External Secrets Operator (ESO)** with Kubernetes. 
- **Workflow**:
  1. DevOps engineers store plain-text secrets in a secure vault (AWS Secrets Manager, HashiCorp Vault).
  2. The K8s External Secrets Operator authenticates against the vault via an IAM Role (IRSA).
  3. ESO securely syncs the vault secrets into native K8s `Secret` objects.
  4. Pods mount these native K8s secrets as Environment Variables or secure RAM-backed file volumes.

## 7. Infrastructure Monitoring & Observability

Observability is treated as a first-class citizen using the OpenTelemetry specification.

- **Metrics Collection**: All microservices expose a Prometheus `/metrics` endpoint. A cluster-internal Prometheus server scrapes these endpoints, and Grafana visualizes the dashboards (visualizing JVM/Go memory heaps, HTTP error rates, Kafka lag).
- **Log Aggregation**: Containers write JSON logs to standard output (`stdout`). **Fluent Bit** DaemonSets collect these logs from the K8s node runtime and pipeline them to ElasticSearch or Datadog.
- **Distributed Tracing**: The API Gateway initiates an OpenTelemetry trace ID. This ID is passed via HTTP Headers (`traceparent`) to internal APIs, and via Kafka record headers. The traces are exported to a central Jaeger or Tempo backend to visualize multi-service latencies and pinpoint bottlenecks.
