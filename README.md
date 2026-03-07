# Distributed Event Processing Platform 🚀

Welcome to the **Distributed Event Processing Platform**, a scalable, microservices-based architecture designed to handle thousands of concurrent events per second with high reliability and low latency.

This repository currently houses the **Engineering specifications** and **Architecture Designs** that will dictate the development of the codebase.

## 📖 Table of Contents
- [Distributed Event Processing Platform 🚀](#distributed-event-processing-platform-)
  - [📖 Table of Contents](#-table-of-contents)
  - [🏗️ System Architecture](#️-system-architecture)
  - [📚 Documentation Overview](#-documentation-overview)
    - [Microservice Specifications](#microservice-specifications)
    - [Architecture \& System Design](#architecture--system-design)
    - [Database \& State Management](#database--state-management)
    - [Engineering Standards](#engineering-standards)
    - [DevOps \& Deployment](#devops--deployment)
  - [🛠️ Technology Stack](#️-technology-stack)
  - [Testing Strategy](#testing-strategy)
  - [💡 Getting Started](#-getting-started)

---

## 🏗️ System Architecture

The platform follows an **Event-Driven Architecture (EDA)** pattern centered around **Kafka** for asynchronous messaging, supported by **PostgreSQL** for persistent ACID storage, and **Redis** for distributed caching and rate-limiting.

External traffic is routed through a high-performance **API Gateway** to decoupled microservices (User, Order, Notification, Analytics).

For a deep dive into the architecture components, refer to our [System Architecture Guide](docs/system-architecture.md).

---

## 📚 Documentation Overview

Before writing any code, we rely on comprehensive, production-grade blueprints. All specifications are stored in the `/docs` directory.

### Microservice Specifications
Detailed API contracts, responsibilities, and internal logic for each independent deployable service:
- [API Gateway Spec](docs/services/api-gateway-spec.md) - Routing, Auth, Rate Limiting.
- [Order Service Spec](docs/services/order-service-spec.md) - Transactional Outbox, Order Fulfillment.
- [User Service Spec](docs/services/user-service-spec.md) - Identity, CRM, Profile Management.
- [Notification Service Spec](docs/services/notification-service-spec.md) - Email/SMS Dispatching, Kafka Consumption.
- [Analytics Service Spec](docs/services/analytics-service-spec.md) - Real-time metrics streaming, materialized views.

### Architecture & System Design
High-level decisions and distributed computing patterns:
- [Event Streaming Architecture](docs/architecture/event-streaming-architecture.md) - Kafka Topics, Consumer Groups, Retries.
- [Idempotency Design](docs/architecture/idempotency-design.md) - Safely handling duplicate requests and network retries.
- [Redis Caching Strategy](docs/architecture/redis-caching-strategy.md) - Cache Aside, Write-Through, Eviction.
- [Observability](docs/architecture/observability.md) - OpenTelemetry Tracing, Prometheus Metrics.

### Database & State Management
- [Database Design](docs/database/database-design.md) - Schema structures, partitioning, indexing.

### Engineering Standards
- [Testing Strategy](docs/engineering/testing-strategy.md) - Unit, Integration (Testcontainers), Chaos, and Load testing specifications.

### DevOps & Deployment
- [Deployment Architecture](docs/devops/deployment-architecture.md) - Kubernetes, CI/CD, Horizontal Pod Autoscaling.

---

## 🛠️ Technology Stack

Based on the finalized design documents, the upcoming codebase will be generating utilizing the following core technologies:

| Category | Technology | Purpose |
| :--- | :--- | :--- |
| **Backend Language** | `Go` | High-throughput, low-latency microservices |
| **API Framework** | `Gin / Fiber` | High-performance REST APIs |
| **Database** | `PostgreSQL` | Primary source of truth, ACID transactions |
| **Message Broker** | `Kafka` | Asynchronous pub/sub, event streaming |
| **Caching/KV** | `Redis` | Edge caching, Idempotency keys, Rate limit tokens |
| **Testing** | `Testcontainers`| Reliable integration tests against real Docker containers |
| **Infrastructure** | `Kubernetes` | Container orchestration, Service discovery |

---

## Testing Strategy
We test against real dependencies instead of in-memory mocks. Please read our [Testing Strategy](docs/engineering/testing-strategy.md) to understand our expectations for isolated unit tests and Testcontainer-backed integration tests.

---

## 💡 Getting Started

*Note: The platform is currently in the **Specification Phase**. Code generation has not yet started.* 

Stay tuned for upcoming commits where we will initialize the Go Go-workspace modules and begin materializing these architectures!
