# System-Wide Idempotency Design Specification

## 1. Overview
In a distributed system where network timeouts and retries are guaranteed, **Idempotency** ensures that an operation can be applied multiple times without changing the result beyond the initial application. This document defines the engineering standards for implementing idempotent APIs and event consumers across the Event Processing Platform.

## 2. API Design: Idempotency Keys

For all state-mutating API requests (e.g., `POST`, `PUT`, `PATCH`), clients are required to provide a unique token that identifies the specific intent of the request.

- **Header Specification**: `Idempotency-Key: <UUID-v4>`
- **Client Responsibility**: The client generates a UUID for a specific action (e.g., clicking "Checkout"). If the client's network drops before receiving the `201 Created` response, the client resends the exact same HTTP request with the *same* `Idempotency-Key`.
- **Server Responsibility**: The server recognizes the duplicate key and returns the identical, cached response from the original successful execution, without re-processing the financial transaction.

### Example Contract
```http
POST /v1/orders HTTP/1.1
Content-Type: application/json
Idempotency-Key: a4b9c1d2-e3f4...

{ "product_id": "123", "quantity": 1 }
```

## 3. Storage Strategy: Redis vs. Database

The platform employs a two-tiered approach to idempotency storage to balance extreme scale (Redis) with absolute financial consistency (PostgreSQL).

### Tier 1: Redis (The API Gateway / Fast Path)
- **Role**: Blocks rapid, immediate duplicate requests (e.g., a user double-clicking a submit button).
- **Mechanism**:
  1. Gateway attempts to `SETNX idemp:api:{Idempotency-Key} "PROCESSING" EX 86400`.
  2. If it succeeds, the request routes to the downstream service.
  3. When the downstream service returns the response body (e.g., `{"order_id": "123"}`), the Gateway updates the Redis key with the serialized HTTP response: `SET idemp:api:{Idempotency-Key} "{...response...}"`.
  4. If a duplicate request arrives and the key equals `"PROCESSING"`, the Gateway returns `409 Conflict` (indicating a concurrent request is already handling this intent).
  5. If a duplicate request arrives and the key holds a JSON payload, the Gateway instantly returns `200 OK` with the cached payload, saving the downstream service from execution.

### Tier 2: PostgreSQL (The Absolute Source of Truth)
- **Role**: Guarantees absolute data consistency preventing dual-writes, even if Redis data is evicted early or fails.
- **Mechanism**:
  - Critical domain entities (`orders`, `users`) contain a unique constraint on the idempotency key: `ALTER TABLE orders ADD CONSTRAINT unique_idempotency UNIQUE (idempotency_key);`
  - When the microservice attempts an `INSERT`, if the key already exists, PostgreSQL throws a unique constraint violation.
  - The microservice catches this exact SQL driver error, looks up the existing record by that key, and returns the result safely.

## 4. Event Consumer Idempotency

Kafka provides At-Least-Once delivery semantics by default. A consumer (e.g., Notification Service) may receive the exact same `OrderCreated` event twice if a consumer group rebalances or a batch fails to commit offsets entirely.

- **Idempotent Identifiers**: All Kafka events adhere to CloudEvents and include a globally unique `id` (e.g., `evt_abc123`).
- **Consumer Processing Logic**:
  1. Consumer reads message `evt_abc123` from Kafka.
  2. Consumer executes a Redis Atomic Check: `SETNX handled:event:{evt_abc123} 1 EX 604800` (7-day TTL).
  3. **If 0 (False)**: The event was already processed. The consumer acknowledges/commits the Kafka offset immediately and skips processing.
  4. **If 1 (True)**: First time seeing this event. Process the notification payload.
- **OLAP Exceptions**: For databases built for UPSERT operations (like the Analytics Service PostgreSQL data warehouse), explicit Redis checks are bypassed. The service relies on `ON CONFLICT (event_id) DO UPDATE` to achieve idempotency at the storage layer, maximizing throughput.

## 5. Retry-Safe Operations

To fully leverage idempotency, the orchestration of actions within a single microservice must be designed carefully.

### The "Check-Then-Act" Anti-Pattern
**DO NOT DO THIS**:
```go
// Risky: Two racing requests both pass the check before the insert.
if !db.OrderExists(idempotencyKey) {
    db.InsertOrder(...)
    stripe.Charge(...)
}
```

### The "Rely on Constraints" Pattern
**DO THIS**:
```go
// Safe: Pushes the concurrency control to the ACID database constraint.
err := db.InsertOrder(..., idempotencyKey)
if isUniqueConstraintViolation(err) {
    // Already processed. Return existing order.
    return db.GetOrderByKey(idempotencyKey)
}

// Guaranteed to be the only thread executing this block.
stripe.Charge(...)
```

### External API Calls
When interacting with 3rd-party services (like Stripe or SendGrid), always pass the `Idempotency-Key` or `event_id` along to the provider. High-quality APIs will respect this token and drop duplicate requests on their end, acting as the final safety net.
