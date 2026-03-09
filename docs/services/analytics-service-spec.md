# Analytics Service Engineering Specification

## 1. Overview
The **Analytics Service** is a high-throughput, data-intensive microservice responsible for consuming massive volumes of business events across the platform and structuring them for real-time and historical analysis. It processes streams of strongly-typed binary events (e.g., Avro or Protobuf via Confluent Schema Registry) from Kafka, ensuring strict schema compatibility, and incrementally aggregates them into structured, queryable metrics within an OLAP-optimized storage layer.

## 2. Responsibilities
- **High-Volume Event Ingestion**: Consume domain events (`UserCreated`, `OrderCreated`, `PaymentFailed`) from multiple Kafka topics continuously.
- **Data Transformation & Enrichment**: Cleanse, flatten, and enrich raw JSON payloads before storage.
- **Batch Aggregation**: Periodically aggregate raw events into time-series rollups (hourly, daily, monthly).
- **Metric Exposure**: Provide fast, aggregated read-APIs to power internal dashboards and executive reports.

## 3. Architecture

### Event Ingestion Pipeline
To handle high velocity without overwhelming the database with individual inserts, the service uses micro-batching.

```mermaid
flowchart TD
    subgraph Event Broker
        Kafka[(Kafka Topics)]
    end
    
    subgraph Analytics Service
        Consumer[Kafka Consumer Group]
        Buffer[In-Memory Buffer / Channel]
        Flusher[Batch Writer (Cron/Ticker)]
        Aggregator[Continuous Materializer]
    end
    
    subgraph Data Warehouse
        DB[(PostgreSQL Analytics DB)]
    end
    
    Kafka -->|Poll 1k msgs| Consumer
    Consumer -->|Append| Buffer
    Buffer -->|Flush every 5s / 10k msgs| Flusher
    Flusher -->|Bulk Insert| DB
    
    DB -->|Read Raw Events| Aggregator
    Aggregator -->|Write Rollups| DB
```

### Technology Choice: PostgreSQL (TimescaleDB / Citus)
While specialized OLAP columnar databases (ClickHouse, Snowflake) are ideal for analytics, a well-tuned **PostgreSQL** instance is highly capable of handling this workload if configured carefully:
1. **Partitioning**: Native declarative table partitioning by `DATE`.
2. **Materialized Views / Cron**: For aggressive pre-aggregation.
3. If volume exceeds 10,000 writes/sec, we will seamlessly upgrade the Postgres instance to use the **TimescaleDB** extension for automatic hypertable partitioning and continuous aggregates.

## 4. Data Models

### Raw Event Ingestion Schema (Fact Table)
Append-only log of all flattened events. Highly partitioned.

**Table: `raw_events` (Partitioned by `created_at` RANGE: Daily)**
| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `event_id` | UUID | | Original Kafka Event ID. |
| `event_type`| VARCHAR(100) | INDEX | E.g., `OrderCreated`. |
| `created_at`| TIMESTAMPTZ | NOT NULL | Event timestamp (from payload, NOT insertion time). |
| `user_id` | UUID | INDEX | Extracted for fast filtering. |
| `metric_value`| NUMERIC | | E.g., `total_amount_cents` for orders. |
| `metadata` | JSONB | | The full raw event body. |

*(Note: We do NOT enforce a `PRIMARY KEY` or `UNIQUE` constraint on this table during ingestion to maximize insert throughput. Deduplication is handled upstream or during aggregation.)*

### Aggregation Schema (Rollup Tables)
**Table: `daily_sales_metrics`**
| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `date` | DATE | PRIMARY KEY | Truncated `created_at`. |
| `total_orders` | BIGINT | | Count of `OrderCreated` events. |
| `gross_revenue_cents`| BIGINT | | Sum of `total_amount_cents`. |
| `unique_users` | BIGINT | | Count distinct `user_id`. |

## 5. Workflows

### Batch Ingestion Strategy
1. The Kafka Consumer pulls events in large batches (e.g., `max.poll.records=5000`).
2. The payload is unmarshalled, validated against the schema, and mapped to the flat `raw_events` structure.
3. The records are held in an in-memory buffer.
4. When the buffer reaches **10,000 records** OR a **5-second ticker** triggers, the Flusher executes a `COPY FROM STDIN` or a multi-row `INSERT` statement into PostgreSQL.
5. **Only after** the database transaction commits successfully does the consumer explicitly `CommitOffsets` to Kafka.

### Aggregation Jobs
Instead of computing aggregations on-the-fly when a dashboard loads (which is slow for millions of rows), background jobs continuously pre-calculate metrics.
- **Hourly Cron Job**: Runs at minute 15 of every hour.
  ```sql
  INSERT INTO daily_sales_metrics (date, total_orders, gross_revenue_cents, unique_users)
  SELECT 
    DATE_TRUNC('day', created_at),
    COUNT(*),
    SUM(metric_value),
    COUNT(DISTINCT user_id)
  FROM raw_events 
  WHERE event_type = 'OrderCreated' 
    AND created_at >= '2026-03-06'
  ON CONFLICT (date) DO UPDATE SET 
    total_orders = EXCLUDED.total_orders,
    gross_revenue_cents = EXCLUDED.gross_revenue_cents,
    unique_users = EXCLUDED.unique_users;
  ```

## 6. Optimization Strategies

### Query Optimization
- **BRIN Indexes**: The `raw_events` table is clustered chronologically. Instead of a B-Tree index on `created_at` (which degrades insert performance and consumes massive RAM), utilize a **BRIN (Block Range Index)**. It stores the min/max values for groups of pages, taking 1/1000th of the space while drastically accelerating time-based queries.
- **JSONB GIN Indexes**: Only index specific keys within the `metadata` payload that are heavily queried. Avoid full `jsonb_path_ops` indexes unless absolutely necessary.
- **Read Replicas**: The application exposes a REST API (`GET /v1/analytics/sales?interval=daily`) pointing exclusively to a read-replica. The Primary node is dedicated 100% to the heavy `INSERT` micro-batches.

## 7. Failure Handling & Consistency
- **Out of Order / Late Events**: Because aggregations (`daily_sales_metrics`) use UPSERT (`ON CONFLICT DO UPDATE`), if an `OrderCreated` event arrives 3 days late due to Kafka replication lag, the aggregation job for that specific past chronological date simply recalculates and corrects the past metric transparently.
- **Idempotency (Exactly-Once Semantics)**: High-performance databases drop `UNIQUE` constraints to speed up writes. The Analytics Service employs a Redis-based deduplication window (`handled:event:{evt_id}`, 7-day TTL) paired with Kafka Streams / ksqlDB upstream for strict Exactly-Once streaming. This guarantees complete accuracy for critical metrics without sacrificing insert throughput.
- **Dead Letter Queue (DLQ)**: Events that fail schema validation (e.g., missing required fields) or routing logic are automatically forwarded to an `domain.analytics.dlq` Kafka topic. This prevents poison pills from blocking the consumer group.
