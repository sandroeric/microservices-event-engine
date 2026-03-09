# Engineering Testing Strategy

## 1. Overview
Reliability in the Event Processing Platform is non-negotiable. To ensure code quality, prevent regressions, and guarantee resilience in our distributed system, we employ a comprehensive, multi-layered testing strategy. This document outlines the standards for Unit, Integration, Contract, End-to-End, Load, Chaos, Security, and Observability testing across all microservices.

## 2. Unit Testing
Unit tests validate the smallest testable parts of an application (functions, methods) in isolation.

- **Scope**: Core domain logic, state machine transitions, template rendering, and utility functions.
- **Dependencies**: All external dependencies (PostgreSQL, Redis, Kafka, HTTP clients) MUST be mocked.
- **Tools**: 
  - Go: Native `go test`.
  - Python: `pytest` with `unittest.mock`.
- **Coverage Target**: 80% minimum line coverage for business logic packages.
- **Mutation Testing**: To ensure test quality and prevent "gaming" the coverage metric, we occasionally run mutation testing tools (e.g., `go-mutesting`) to verify our tests catch injected bugs.
- **Execution**: Runs on every commit to any branch. Must execute in under 30 seconds.

## 3. Integration Testing (Testcontainers)
Integration tests verify that our microservice interacts correctly with its real external dependencies (databases, caches, message brokers). We do NOT use in-memory SQLite or embedded Redis; we test against the exact container images used in production.

- **Scope**: Repository layers (SQL queries), Cache adapters, Kafka Producers/Consumers, and REST API handlers.
- **Tooling**: **[Testcontainers](https://testcontainers.com/)** (available in Go, Python).
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

## 5. End-to-End (E2E) Testing
E2E testing validates system-wide user journeys across multiple services simultaneously in a production-like environment.

- **Scope**: Critical business flows (e.g., "User registers, places an order, and receives an email notification").
- **Tooling**: **Playwright**, **Cypress**, or native Go/Python HTTP test suites.
- **Strategy**: Define realistic test data scenarios. Run black-box tests against exposed API boundaries. Use synthetic data generators (e.g., GoFakeIt, Faker) or sanitized production snapshots. Clean up data post-run via specific teardown hooks. 
- **Execution**: Runs against the `staging` environment post-deployment or during nightly builds.

## 6. Load and Performance Testing (k6)
Load testing simulates high concurrent user traffic to identify bottlenecks, memory leaks, and tune Kubernetes Horizontal Pod Autoscaler (HPA) triggers.

- **Scope**: End-to-end user flows (e.g., "Create 10,000 orders in 1 minute").
- **Tooling**: **[Grafana k6](https://k6.io/)**.
- **Strategy**:
  - **Smoke Testing**: Minimal load to verify the system can handle basic concurrency.
  - **Load Testing**: Simulating expected peak production load (e.g., 500 requests per second) to verify response times remain under SLAs.
  - **Spike Testing**: Very rapid, massive traffic spikes to test the API Gateway Rate Limiting and Kubernetes HPA scale-up latency.
- **Metrics Evaluated**: `http_req_duration` (P95 < 200ms), `http_req_failed` < 0.1%, Kafka Consumer Lag recovery times.
- **Execution**: Executed nightly against a dedicated `staging` environment.

## 7. Chaos Engineering
Chaos testing proactively injects failures into the distributed system to validate the resilience mechanisms (Circuit Breakers, Retries, DLQs) defined in our architecture specifications.

- **Scope**: Network partitions, pod terminations, high latency, dependency outages.
- **Tooling**: **Chaos Mesh** or **Gremlin** running in the Kubernetes cluster.
- **Experiments**:
  - **Pod Kill**: Randomly terminate 20% of `Order Service` pods to verify zero-downtime rolling and active Request retry logic.
  - **Network Latency**: Inject 2-second latency into the connection between the `Notification Service` and `Redis` to test timeout configurations and application fail-fast (Circuit Breaker) behavior.
  - **Dependency Outage**: Completely block traffic to Kafka for 5 minutes. Verify the `User Service` Transactional Outbox queues up messages, and successfully drains the backlog once connectivity is restored.
- **Execution**: Run manually during Game Days or scheduled weekly against the `staging` environment.

## 8. Security Testing (DevSecOps)
Integrating security directly into our testing lifecycle is essential to prevent vulnerabilities in code and dependencies.

- **Scope**: Application source code, third-party libraries, and container images.
- **Tooling**:
  - **SAST (Static Application Security Testing)**: `gosec` for Go, `bandit` for Python.
  - **Dependency Scanning**: `govulncheck`, Snyk, or Dependabot to scan for known CVEs.
  - **DAST (Dynamic Application Security Testing)**: Automated dynamic scanning to catch runtime vulnerabilities in the deployed application.
- **Execution**: SAST and Dependency scanning run on every PR. DAST runs post-deployment to `staging`.

## 9. Observability and Trace-Based Testing
Verifying that tracing and logging work properly is just as important as verifying business logic, ensuring production debuggability.

- **Scope**: Distributed traces (OpenTelemetry spans), business metrics, and structured logs.
- **Tooling**: **[Tracetest](https://tracetest.io/)** or custom assertion libraries.
- **Strategy**: 
  - Fire requests across services and specifically assert against the exported trace (e.g., "Did the event reach Kafka? Did the exact span `process-order` execute?").
  - Validate that essential business metrics and Prometheus counters increment correctly.
- **Execution**: Runs in combination with E2E and Integration test suites.

## 10. Developer Experience (DX) & CI/CD Mapping
To ensure adherence to this testing strategy, developer ergonomics and pipeline stages must be seamless.

- **Local DX**: Standardize a `Makefile` across all microservice repositories:
  - `make test` (Runs fast unit tests)
  - `make test-integration` (Spins up Testcontainers and runs integrations)
  - `make test-coverage` (Generates HTML coverage reports)
- **CI/CD Pipeline Sequence**:
  1. **Lint & Security (SAST/Dep Scan) & Unit Tests** (Parallelize) - Fails fast.
  2. **Integration Tests** - Requires Docker footprint.
  3. **Contract Tests** - validates provider/consumer guarantees.
  4. **Build Image & Push**
  5. **Deploy to Staging**
  6. **E2E, Observability & DAST Testing** (Post-deployment)
  7. **Load/Chaos** (Nightly/Scheduled)
