# Event Streaming Architecture (Kafka)

> **Version:** 1.1 | **Last Updated:** 2026-03-07 | **Owner:** Platform Engineering

## 1. Overview
The Event Processing Platform utilizes **Apache Kafka** as its central nervous system. This architecture defines the standards for producing, routing, and consuming domain events to ensure high throughput, durability, and loose coupling between microservices. We have chosen Kafka over RabbitMQ specifically for its persistent log structure, which enables Event Sourcing, consumer replays, and massive horizontal scalability via partitioned topics.

## 2. Topic & Partition Strategy

### Topic Structure
Topics are strictly bounded by Domain Entities (Aggregates). We avoid grouping unrelated events into a single "firehose" topic.

- `domain.users.events`: All mutations related to the User Aggregate (e.g., `UserCreated`, `UserUpdated`).
- `domain.orders.events`: All mutations related to the Order Aggregate (e.g., `OrderCreated`, `PaymentFailed`).
- `sys.notifications.dlq`: Dead Letter Queue for failed notification processing.

### Partitioning & Ordering
- **Partition Key**: All events related to a specific aggregate MUST use the Aggregate's ID (e.g., `user_id` or `order_id`) as the Kafka message key.
- **Why?**: Kafka guarantees strict ordering of messages *within a partition*. By hashing the aggregate ID to a partition, we guarantee that all state changes for a single Order (e.g., `PENDING` -> `SHIPPED`) are consumed in the exact sequence they occurred, preventing race conditions like delivering a shipment before the payment completes.

## 3. Event Schema Design

### JSON vs. Avro — When to Use Each

| Scenario | Format | Rationale |
|---|---|---|
| Internal domain events, low-to-medium throughput | **JSON (CloudEvents)** | Human-readable, easy debugging, no schema registry dependency |
| High-throughput pipelines (>50k msg/s), cross-team contracts | **Avro + Schema Registry** | ~5x smaller payload, compile-time schema enforcement, enforced compatibility |

All messages must adhere to the **CloudEvents** specification (v1.0). For Avro, a **Confluent Schema Registry** instance (or compatible) is required.

```json
{
  "specversion": "1.0",
  "id": "evt_abc123789",
  "source": "/services/order-service",
  "type": "domain.orders.OrderCreated",
  "datacontenttype": "application/json",
  "time": "2026-03-07T12:00:00Z",
  "subject": "ord_987654",
  "data": {
    "user_id": "usr_9cc9b6",
    "totalAmountCents": 5000,
    "items": [
      {
        "productId": "prod_a1",
        "quantity": 1
      }
    ]
  }
}
```
*Note: The `id` functions as the Global Idempotency Key for downstream consumers.*

### Schema Evolution Policy (Avro)
When using Avro with Schema Registry, all schemas must be registered under **`BACKWARD` compatibility** mode (the default). This ensures new consumers can read events produced by older producers.

- **Allowed**: Adding fields with a default value.
- **Allowed**: Removing fields that have a default value.
- **Forbidden**: Renaming or removing mandatory fields without a major version bump.
- **Version bumping**: When a breaking change is required, create a new topic (`domain.orders.events.v2`) and run both versions in parallel during the migration window.

## 4. Topic Governance & Retention

### Topic Creation Policy
Topics must **not** be created ad-hoc in production. All topic definitions are source-controlled via Terraform (or Confluent's `kafka-topics` operator for K8s). A CI schema check must pass before topic creation is merged.

**Naming convention**: `{domain}.{aggregate}.{purpose}` (e.g., `domain.orders.events`, `sys.notifications.dlq`).

### Retention Policy
| Topic Type | Retention | Rationale |
|---|---|---|
| Domain event topics (`domain.*`) | **7 days** (default) | Allows full consumer group replay for new services |
| Dead Letter Queues (`*.dlq`) | **30 days** | Extended window for engineering investigation |
| Audit/Compliance events | **365 days** | Regulatory requirement |

> **Note:** Retention is a time-based guarantee, not an infinite log. For event sourcing scenarios requiring indefinite replay, enable **log compaction** on the relevant topic in addition to time retention.

## 5. Producer & Consumer Design

### Event Flow Architecture
The platform strictly separates the act of persisting state from broadcasting events to prevent the Dual-Write Problem.

```mermaid
flowchart TD
    subgraph Publisher Service [e.g. Order Service]
        API[API Handler]
        DB[(PostgreSQL)]
        Outbox[Outbox Relay / Debezium]
        
        API -->|1. Write State & Event| DB
        Outbox -->|2. Poll / Tail Log| DB
        Outbox -->|3. Publish| KafkaTopic
    end
    
    subgraph Kafka Cluster
        KafkaTopic[Kafka: domain.orders.events]
    end
    
    subgraph Consumer Service A [e.g. Notification Service]
        CG_A[Consumer Group A]
        Redis_A[(Redis: Idempotency)]
        Logic_A[Process Logic]
        
        KafkaTopic -->|4. Poll| CG_A
        CG_A <-->|5. SETNX id| Redis_A
        CG_A --> Logic_A
    end
    
     subgraph Consumer Service B [e.g. Analytics Service]
        CG_B[Consumer Group B]
        DB_B[(OLAP DB)]
        
        KafkaTopic -->|4. Poll| CG_B
        CG_B -->|5. UPSERT | DB_B
    end
```

### Producer Design (Transactional Outbox)
- Microservices MUST NOT explicitly call `kafka_producer.send()` during an HTTP request.
- **Implementation**: The service inserts the event into a local PostgreSQL `outbox_events` table within the same transaction that updates the business entity. A background CDC (Change Data Capture) tool like **Debezium** or a custom polling worker reads the outbox table and guarantees *At-Least-Once* delivery to Kafka.
- **Prerequisite**: PostgreSQL must have **logical replication** enabled (`wal_level = logical`) for Debezium to tail the WAL. This must be provisioned at cluster creation time.

### Consumer Groups
- **Logical Grouping**: Each reacting microservice defines its own unique `group.id` (e.g., `notification-service-cg`, `analytics-service-cg`). 
- **Broadcast Semantics**: Because they use different group IDs, both the Notification Service and the Analytics Service receive a copy of every `OrderCreated` event to process independently.
- **Horizontal Scaling**: If the Notification Service is lagging, K8s scales the pods. Kafka automatically rebalances the topic partitions across the new pods within the same consumer group. (Max pods = Number of topic partitions).

## 6. Idempotency & Failure Handling

### Idempotent Event Consumption
Kafka's *At-Least-Once* delivery guarantee means consumers will occasionally receive the same event twice (e.g., during consumer rebalancing).
- **Rule**: Every consumer MUST be idempotent.
- **Strategy A (Strict)**: The consumer checks the event `id` against a Redis cache (`SETNX event:processed:{id} 1 EX 86400`). If the key exists, acknowledge the offset and skip processing.
- **Strategy B (Database UPSERT)**: The Analytics Service simply uses SQL `ON CONFLICT (id) DO UPDATE`. Replaying the event yields the same final state.

### Consumer Rebalancing & Long Retries
Since Kafka is a sequential log, a consumer cannot easily "skip" a failing message and process the next one without committing the offset of the failure.

- **Transient Failures (e.g., 3rd party API down)**: The consumer uses Exponential Backoff **without blocking the poll loop**. Use `consumer.pause(partitions)` while the retry is in-flight, then call `consumer.resume(partitions)` once the retry succeeds or is abandoned.
- **`max.poll.interval.ms` Constraint**: Blocking the poll loop during retries will cause Kafka to assume the consumer is dead, triggering a rebalance storm. The retry window MUST complete within `max.poll.interval.ms` (default 5 minutes). Increase this setting if your processing SLA genuinely requires longer retry windows.

| Config | Recommended Value | Notes |
|---|---|---|
| `max.poll.interval.ms` | `600000` (10 min) | Adjust per service SLA |
| `max.poll.records` | `50` | Limit batch size to reduce rebalance impact |
| `session.timeout.ms` | `45000` | Must be < `max.poll.interval.ms` |

### Dead Letter Queues (DLQ)
For semantic or permanent failures (e.g., JSON schema validation fails, or 3rd party API returns 403 Forbidden).

**DLQ Naming Convention**: Every consumer group has its own DLQ: `{domain}.{service}.dlq` (e.g., `domain.notifications.dlq`, `domain.analytics.dlq`).

1. The consumer catches the unrecoverable exception.
2. The consumer publishes the exact payload, wrapped in diagnostic metadata (Exception Stack Trace, Original Topic, Original Partition, Original Offset, Timestamp), to its sidecar DLQ topic.
3. The consumer explicitly commits the offset for the failed message on the primary topic, allowing it to move on to the next valid message.
4. Engineers monitor the DLQ, deploy a code fix, and execute a tool to replay the DLQ back into the primary topic.
