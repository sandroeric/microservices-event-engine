# Observability Specification

> **Version:** 1.1 | **Last Updated:** 2026-03-07 | **Owner:** Platform Engineering

## 1. Overview
In a distributed event-driven architecture, observing the state of a single service is insufficient to diagnose systemic failures. The Event Processing Platform mandates a unified observability strategy encompassing four pillars: **Logging**, **Metrics**, **Distributed Tracing**, and **Continuous Profiling**.

This specification defines the standards for integrating OpenTelemetry, Prometheus, Grafana, Jaeger, and FluentBit to provide full end-to-end visibility of a request's lifecycle.

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
Microservices must write logs to `stdout` exclusively in structured JSON format. Unstructured text logs (e.g., `log.Printf("Order %s created", id)`) are strictly forbidden, as they break efficient querying in Elasticsearch/OpenSearch.

**Required JSON Log Schema:**
```json
{
  "timestamp": "2026-03-07T12:05:00.123Z",
  "level": "INFO",
  "service": "order-service",
  "trace_id": "5b8aa5a2d2c8646c3ce2fa23df6126a1",
  "span_id": "c2fb4e5d9e6f1a3b",
  "message": "Payment captured successfully",
  "order_id": "ord_987654",
  "duration_ms": 145.2
}
```

> **`span_id` is mandatory.** Without it, log lines cannot be linked to a specific span in Jaeger, breaking trace-log correlation. The `trace_id` alone only narrows down to the request; `span_id` identifies the exact operation.

### Log Level Policy

Every log statement must use the correct level. Incorrect levels are treated as a code quality issue during review.

| Level | When to Use | Examples |
|---|---|---|
| `DEBUG` | Detailed internal state for local development and debugging. **Never** enabled in production by default. | Entering function, intermediate variable values, cache key lookups |
| `INFO` | Normal operational events that confirm the system is working as expected. | Order created, user authenticated, event published |
| `WARN` | Unexpected conditions the system recovered from, or approaching resource limits. | Cache miss on a hot key, retry attempt 2/3, Redis memory > 70% |
| `ERROR` | A request or operation failed. Requires human attention but the service continues running. | Database write failed, external API returned 5xx, idempotency constraint violated |
| `FATAL` / `PANIC` | The service cannot continue. Used only by the process root, not business logic. | Cannot bind to port, required config missing at startup |

### Log Aggregation (FluentBit)
A **FluentBit** DaemonSet runs on every K8s node. It reads the raw JSON from the container stdout interface, enriches it with K8s metadata (`pod_name`, `namespace`, `node_name`), and forwards the pipeline to the central OpenSearch/Elasticsearch cluster.

> **FluentBit** (not Fluentd) is the standard log shipper for this platform. FluentBit is a lightweight C-based agent optimized for Kubernetes node-level collection with lower CPU and memory overhead. Fluentd is not used.

## 4. Metrics Collection (Prometheus)

Every microservice must expose a Prometheus HTTP endpoint (typically `/metrics` on an internal-only port like `9091`) for the Prometheus Server to scrape every 15 seconds.

### Minimum Required Metrics (RED Method)
1. **Rate**: 
   - HTTP: `http_requests_total{method="POST", path="/orders", status="201", service="order-service"}`
   - Kafka: `kafka_messages_published_total{topic="domain.orders.events"}`
2. **Errors**: 
   - HTTP: `http_requests_total{status=~"5.*"}`
3. **Duration**: 
   - HTTP: `http_request_duration_seconds_bucket{path="/orders"}` (Histogram)

### System Resource Metrics (USE Method)
- `process_cpu_seconds_total`
- `go_memstats_heap_inuse_bytes`
- `db_connection_pool_active`

### Kafka Metrics Export
Kafka broker and consumer lag metrics are exposed via the **[Kafka Exporter for Prometheus](https://github.com/danielqsj/kafka_exporter)**, deployed as a dedicated pod in the cluster. It connects to the Kafka brokers and exposes consumer group lag metrics:

- `kafka_consumergroup_lag{consumergroup, topic, partition}` — **primary health signal**
- `kafka_topic_partitions{topic}` — for partition count tracking
- `kafka_brokers` — broker availability

The Kafka Exporter is scraped by Prometheus on the same 15-second interval.

## 5. Continuous Profiling

Continuous profiling is the fourth observability pillar. It captures CPU, memory allocation, goroutine, and blocking profiles from the live service without requiring a separate debugging session.

### Go `pprof` Endpoint (Development & Staging)
All Go services expose the standard `net/http/pprof` handler on the internal metrics port (`/debug/pprof/`). This endpoint **must not** be exposed publicly but is accessible within the cluster for on-demand profiling during incidents.

```go
import _ "net/http/pprof" // Register pprof handlers on DefaultServeMux
```

### Continuous Profiling in Production (Pyroscope)
For always-on production profiling, services integrate the **[Pyroscope Go SDK](https://grafana.com/docs/pyroscope/latest/configure-client/language-sdks/go_push/)** to continuously push profiling data to a central Pyroscope server (or Grafana Cloud Profiles):

```go
pyroscope.Start(pyroscope.Config{
    ApplicationName: "order-service",
    ServerAddress:   "http://pyroscope.monitoring:4040",
    ProfileTypes: []pyroscope.ProfileType{
        pyroscope.ProfileCPU,
        pyroscope.ProfileAllocObjects,
        pyroscope.ProfileGoroutines,
    },
})
```

This enables flame graph analysis correlated with Grafana dashboards and Jaeger traces.

## 6. Monitoring Dashboards (Grafana)

Grafana consumes the Prometheus data source to visualize health. Each service requires three tiers of dashboards:

1. **Executive SLA Dashboard**: 
   - High-level SLIs: Global P99 Request Latency, 24h Error Budget burndown, Active Users.
2. **Service Drill-Down Dashboard**:
   - RED metrics for a specific microservice.
   - Database connection pool saturation graphs.
   - Cache hit/miss ratio (Redis).
3. **Event Streaming Dashboard**:
   - `kafka_consumergroup_lag{consumergroup="notification-service-cg"}`: The most critical asynchronous health metric. If this trend is strictly increasing, the workers are overwhelmed.

## 7. Alerting Strategies

The **Prometheus AlertManager** evaluates metrics against predefined thresholds and routes alerts to external channels (e.g., Slack, PagerDuty).

### Alert Priorities

- **P1 (Page On-Call)**: Immediate human intervention required.
  - SLO Burn Rate: Error budget consumed at >14.4x rate over 1 hour (i.e., 1-hour burn rate alert based on multi-window, multi-burn-rate SLOs per Google SRE guidelines — equivalent to consuming 2% of weekly budget in 1 hour).
  - Kafka Consumer Lag > 10,000 messages (and increasing for 5 minutes).
  - Primary PostgreSQL CPU Utilization > 90% for 15 minutes.
- **P2 (Slack Alert)**: Warning state, needs investigation during business hours.
  - Pod Restart Loop (CrashLoopBackOff) detected in any replica.
  - Redis memory exceeds 80% (approaching `volatile-lru` evictions).
- **P3 (Email Report)**: Non-urgent structural notifications.
  - TLS Certificate expiring within 14 days.

> **Why burn rate instead of raw error percentage?** A raw threshold like "5xx > 5% for 5 minutes" fires at the same urgency whether you have 1 req/s (1 error) or 10,000 req/s (500 errors). SLO burn rate alerts are proportional to actual error budget consumption and eliminate low-traffic false positives.

### Avoiding Alert Fatigue
Alert definitions must include `for:` clauses to prevent noisy flapping on brief spikes.
```yaml
- alert: HighConsumerLag
  expr: kafka_consumergroup_lag > 5000
  for: 5m
  labels:
    severity: page
  annotations:
    summary: "High Kafka Lag for {{ $labels.consumergroup }}"
```
