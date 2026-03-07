# Observability Specification

## 1. Overview
In a distributed event-driven architecture, observing the state of a single service is insufficient to diagnose systemic failures. The Event Processing Platform mandates a unified observability strategy encompassing three pillars: **Logging**, **Metrics**, and **Distributed Tracing**. 

This specification defines the standards for integrating OpenTelemetry, Prometheus, Grafana, and Jaeger to provide full end-to-end visibility of a request's lifecycle.

## 2. Distributed Tracing (OpenTelemetry & Jaeger)

Tracing maps the journey of a single user request across HTTP boundaries and asynchronous Kafka topics.

### Standard Trace Flow
1. **Initiation**: The **API Gateway** creates a Root Span and generates a W3C trace ID (e.g., `trace_id=5b8aa...`) upon receiving an external `POST /orders` request.
2. **Synchronous Propagation**: The Gateway injects the trace ID into the HTTP header `traceparent` before proxying to the `Order Service`.
3. **Asynchronous Propagation**: When the `Order Service` publishes the `OrderCreated` event to Kafka, the producer injects the W3C trace ID into the native **Kafka Record Headers**.
4. **Consumption**: The `Notification Service` extracts the trace ID from the Kafka header, continuing the trace seamlessly through the asynchronous boundary when dialing SendGrid.
5. **Exporting**: All microservices use the standard OTLP (OpenTelemetry Protocol) exporter to push trace spans asynchronously to a Jaeger Collector.

## 3. Logging Standards

### Structured JSON Format
Microservices must write logs to `stdout` exclusively in structured JSON format. Unstructured text logs (e.g., `log.Printf("Order %s created", id)`) are strictly forbidden, as they break efficient querying in Elasticsearch/Datadog.

**Required JSON Log Schema:**
```json
{
  "timestamp": "2026-03-07T12:05:00.123Z",
  "level": "INFO",
  "service": "order-service",
  "trace_id": "5b8aa5a2d2c8646c3ce2fa23df6126a1",
  "message": "Payment captured successfully",
  "order_id": "ord_987654",
  "duration_ms": 145.2
}
```

### Log Aggregation Collection (FluentBit)
A FluentBit DaemonSet runs on every K8s node. It reads the raw JSON from the container standard output interfaces, enriches the JSON with K8s metadata (`pod_name`, `namespace`), and forwards the pipeline to the central log storage.

## 4. Metrics Collection (Prometheus)

Every microservice must expose a Prometheus HTTP endpoint (typically `/metrics` inside a protected administrative port like `9090`) for the Prometheus Server to scrape every 15 seconds.

### Minimum Required Metrics (RED Method)
1. **Rate**: 
   - HTTP: `http_requests_total{method="POST", path="/orders", status="201", service="order-service"}`
   - Kafka: `kafka_messages_published_total{topic="orders_topic"}`
2. **Errors**: 
   - HTTP: `http_requests_total{status=~"5.*"}`
3. **Duration**: 
   - HTTP: `http_request_duration_seconds_bucket{path="/orders"}` (Histogram)

### System Resource Metrics (USE Method)
- `process_cpu_seconds_total`
- `go_memstats_heap_inuse_bytes` (or JVM equivalent)
- `db_connection_pool_active`

## 5. Monitoring Dashboards (Grafana)

Grafana consumes the Prometheus data source to visualize health. Each service requires three tiers of dashboards:

1. **Executive SLA Dashboard**: 
   - High-level SLIs: Global P99 Request Latency, 24h Error Budget burndown, Active Users.
2. **Service Drill-Down Dashboard**:
   - RED metrics for a specific microservice.
   - Database connection pool saturation graphs.
   - Cache hit/miss ratio (Redis).
3. **Event Streaming Dashboard**:
   - `kafka_consumer_lag_records{group="notification-service-cg"}`: The most critical asynchronous health metric. If this trend is strictly increasing, the workers are overwhelmed.

## 6. Alerting Strategies

The **Prometheus AlertManager** evaluates metrics against predefined thresholds and routes alerts to external channels (e.g., Slack, PagerDuty).

### Alert Priorities
- **P1 (Page On-Call)**: Immediate human intervention required.
  - API Gateway 5xx Error Rate > 5% over 5 minutes.
  - Kafka Consumer Lag > 10,000 messages (and increasing).
  - Primary PostgreSQL CPU Utilization > 90% for 15 minutes.
- **P2 (Slack Alert)**: Warning state, needs investigation during business hours.
  - Pod Restart Loop (CrashLoopBackOff) detected in any replica.
  - Redis memory exceeds 80% (approaching `volatile-lru` evictions).
- **P3 (Email Report)**: Non-urgent structural notifications.
  - TLS Certificate expiring within 14 days.

### Avoiding Alert Fatigue
Alert definitions must include `for: 5m` clauses to prevent noisy flapping on brief spikes.
```yaml
- alert: HighConsumerLag
  expr: kafka_consumergroup_lag > 5000
  for: 5m
  labels:
    severity: page
  annotations:
    summary: "High Kafka Lag for {{ $labels.consumergroup }}"
```
