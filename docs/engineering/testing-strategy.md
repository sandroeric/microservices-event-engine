# Engineering Testing Strategy

## 1. Overview
Reliability in the Event Processing Platform is non-negotiable. To ensure code quality, prevent regressions, and guarantee resilience in our distributed system, we employ a multi-layered testing strategy. This document outlines the standards for Unit, Integration, Contract, Load, and Chaos testing across all microservices.

## 2. Unit Testing
Unit tests validate the smallest testable parts of an application (functions, methods) in isolation.

- **Scope**: Core domain logic, state machine transitions, template rendering, and utility functions.
- **Dependencies**: All external dependencies (PostgreSQL, Redis, Kafka, HTTP clients) MUST be mocked.
- **Tools**: 
  - Go: Native `go test`.
- **Coverage Target**: 80% minimum line coverage for business logic packages.
- **Execution**: Runs on every commit to any branch. Must execute in under 30 seconds.

## 3. Integration Testing (Testcontainers)
Integration tests verify that our microservice interacts correctly with its real external dependencies (databases, caches, message brokers). We do NOT use in-memory SQLite or embedded Redis; we test against the exact container images used in production.

- **Scope**: Repository layers (SQL queries), Cache adapters, Kafka Producers/Consumers, and REST API handlers.
- **Tooling**: **[Testcontainers](https://testcontainers.com/)** (available in Go, Java).
- **Workflow**:
  1. The test suite programmatically spins up Docker containers for PostgreSQL, Redis, and Kafka.
  2. The database receives a fresh schema migration.
  3. The test suite runs API calls against the local service instance.
  4. The test suite asserts that the database state changed correctly or a Kafka message was published.
  5. Testcontainers automatically tears down the Docker containers when the suite finishes.
- **Execution**: Runs on pull requests. Expected to execute in under 3 minutes.

## 4. Contract Testing
In a microservices architecture, services must communicate reliably. Contract testing ensures that the API Gateway, User Service, and Order Service agree on the structure of HTTP JSON payloads and Kafka event schemas.

- **Scope**: HTTP REST APIs and Async Kafka Message Schemas.
- **Tooling**: **Pact**, or schema validation against a centralized **OpenAPI 3.0** registry and **AsyncAPI / Confluent Schema Registry**.
- **Strategy (Consumer-Driven Contracts)**:
  - The *Consumer* (e.g., API Gateway) defines the exact JSON structure it expects from the *Provider* (e.g., User Service) and publishes this "Contract".
  - The *Provider* runs isolated tests against this Contract to ensure it does not introduce breaking changes (e.g., renaming `first_name` to `givenName`).
- **Execution**: Runs as a separate pipeline stage before a branch can be merged to `main`.

## 5. Load and Performance Testing (k6)
Load testing simulates high concurrent user traffic to identify bottlenecks, memory leaks, and tune Kubernetes Horizontal Pod Autoscaler (HPA) triggers.

- **Scope**: End-to-end user flows (e.g., "Create 10,000 orders in 1 minute").
- **Tooling**: **[Grafana k6](https://k6.io/)**.
- **Strategy**:
  - **Smoke Testing**: Minimal load to verify the system can handle basic concurrency.
  - **Load Testing**: Simulating expected peak production load (e.g., 500 requests per second) to verify response times remain under SLAs.
  - **Spike Testing**: Very rapid, massive traffic spikes to test the API Gateway Rate Limiting and Kubernetes HPA scale-up latency.
- **Metrics Evaluated**: `http_req_duration` (P95 < 200ms), `http_req_failed` < 0.1%, Kafka Consumer Lag recovery times.
- **Execution**: Executed nightly against a dedicated `staging` environment.

## 6. Chaos Engineering
Chaos testing proactively injects failures into the distributed system to validate the resilience mechanisms (Circuit Breakers, Retries, DLQs) defined in our architecture specifications.

- **Scope**: Network partitions, pod terminations, high latency, dependency outages.
- **Tooling**: **Chaos Mesh** or **Gremlin** running in the Kubernetes cluster.
- **Experiments**:
  - **Pod Kill**: Randomly terminate 20% of `Order Service` pods to verify zero-downtime rolling and active Request retry logic.
  - **Network Latency**: Inject 2-second latency into the connection between the `Notification Service` and `Redis` to test timeout configurations and application fail-fast (Circuit Breaker) behavior.
  - **Dependency Outage**: Completely block traffic to Kafka for 5 minutes. Verify the `User Service` Transactional Outbox queues up messages, and successfully drains the backlog once connectivity is restored.
- **Execution**: Run manually during Game Days or scheduled weekly against the `staging` environment.
