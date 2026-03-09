# User Service Engineering Specification

## 1. Overview
The **User Service** is a core domain microservice in the Event Processing Platform. It is responsible for managing user identities, profiles, authentication credentials, and tracking session states. It acts as the source of truth for all user data and securely issues JWTs for the rest of the platform.

## 2. Responsibilities
- **User Registration**: Create new user accounts securely and idempotently.
- **Authentication**: Validate credentials and issue short-lived Access Tokens (JWT) alongside long-lived Refresh Tokens.
- **Profile Management**: Retrieve and update user profile information.
- **Session Management**: Track active refresh tokens and enable remote session revocation (logout).
- **Event Publishing**: Publish `UserCreated` and `UserUpdated` events to Kafka so downstream services (like Notification and Analytics) can react asynchronously.

## 3. Architecture

The service interacts directly with PostgreSQL for persistent storage, Redis for fast read-through caching and token blacklisting, and Kafka to asynchronously broadcast domain events.

```mermaid
flowchart TD
    API_GW[API Gateway] --> UserService[User Service]
    
    subgraph Data Layer
        UserService -->|Read/Write| Postgres[(PostgreSQL: users_db)]
        UserService -->|Cache & Revocation| Redis[(Redis: user_cache)]
    end
    
    subgraph Messaging
        UserService -->|Publish (Outbox Relay)| Kafka[Kafka: domain.users.events]
    end
```

## 4. Data Models

### PostgreSQL Schema design
**Table: `users`**
The primary table storing core identity and profile data.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | UUID | PRIMARY KEY | Unique user identifier. |
| `email` | VARCHAR(255) | UNIQUE, NOT NULL | Normalized lowercase email address. |
| `password_hash` | VARCHAR(255)| NOT NULL | Argon2id hashed password. |
| `first_name` | VARCHAR(100) | NOT NULL | User's given name. |
| `last_name` | VARCHAR(100) | NOT NULL | User's family name. |
| `status` | VARCHAR(50) | DEFAULT 'ACTIVE' | `ACTIVE`, `SUSPENDED`, `DELETED`. |
| `created_at` | TIMESTAMPTZ | DEFAULT NOW() | Record creation timestamp. |
| `updated_at` | TIMESTAMPTZ | DEFAULT NOW() | Record modification timestamp. |
| `deleted_at` | TIMESTAMPTZ | NULLABLE | Soft delete timestamp. |

**Table: `audit_logs`**
Tracks security-critical actions (login attempts, password resets) to detect brute-forcing and credential stuffing.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | UUID | PRIMARY KEY | Unique log identifier. |
| `user_id` | UUID | INDEX | (Optional) if associated with a user. |
| `action` | VARCHAR(100) | NOT NULL | e.g., `LOGIN_SUCCESS`, `LOGIN_FAILED`. |
| `ip_address` | INET | | Client IP address. |
| `user_agent` | TEXT | | Client User-Agent hash. |
| `created_at` | TIMESTAMPTZ | DEFAULT NOW() | Event timestamp. |

**Table: `refresh_tokens`**
Tracks long-lived sessions to allow rotation and revocation.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `token_id` | UUID | PRIMARY KEY | JTI (JWT ID) of the refresh token. |
| `user_id` | UUID | FOREIGN KEY | References `users.id` with `ON DELETE CASCADE`. |
| `expires_at` | TIMESTAMPTZ | NOT NULL | Absolute expiration time. |
| `is_revoked` | BOOLEAN | DEFAULT FALSE | Flag to manually invalidate a session. |
| `device_info`| VARCHAR(255)| | Optional metadata (e.g., User-Agent hash). |

**Table: `outbox_events`**
Used for the Transactional Outbox pattern to guarantee event publishing.

| Column | Type | Constraints | Description |
|--------|------|-------------|-------------|
| `id` | UUID | PRIMARY KEY | Event ID. |
| `aggregate_type` | VARCHAR(100) | NOT NULL | e.g., `user`. |
| `aggregate_id` | UUID | NOT NULL | The `user_id` mutated. |
| `event_type` | VARCHAR(100) | NOT NULL | e.g., `UserCreated`. |
| `payload` | JSONB | NOT NULL | The event body. |
| `status` | VARCHAR(50) | DEFAULT 'PENDING'| `PENDING` or `PUBLISHED`. |
| `created_at` | TIMESTAMPTZ | DEFAULT NOW() | When the event was written. |

## 5. Caching Strategy
- **Mechanism**: Read-through cache backed by Redis.
- **Keys**: `user:profile:{user_id}`
- **TTL**: 1 hour.
- **Invalidation**: 
  - To prevent read-write data races upon rapid subsequent edits, the service utilizes **Change Data Capture (Debezium)**. Upon updating the PostgreSQL row, Debezium streams the WAL changes and asynchronously updates or invalidates the Redis key (`user:profile:{user_id}`).
- **Performance consideration**: Profiles are heavily read by internal systems (if not passed via headers), making this cache critical for latency.

## 6. APIs and Interfaces

### **1. Idempotent User Registration**
`POST /v1/users/register`
- **Headers:** `Idempotency-Key` (UUID, Required)
- **Request Body:**
```json
{
  "email": "jane.doe@example.com",
  "password": "SecurePassword123!",
  "first_name": "Jane",
  "last_name": "Doe"
}
```
- **Response Models (201 Created):**
```json
{
  "user_id": "usr_9cc9b6",
  "email": "jane.doe@example.com",
  "status": "ACTIVE"
}
```
- **Idempotency Logic**: The API Gateway handles Redis idempotency locks. If the same `Idempotency-Key` is sent, the gateway returns the cached 201 response. The User Service *also* enforces a UNIQUE constraint on the `email` column preventing silent duplicate creation. Attempting to register an existing email returns `409 Conflict`.

### **2. Authentication (Login)**
`POST /v1/users/login`
- **Request Body:**
```json
{
  "email": "jane.doe@example.com",
  "password": "SecurePassword123!"
}
```
- **Response (200 OK):**
```json
{
  "access_token": "eyJhbGciOiJ...",
  "expires_in": 900, 
  "refresh_token": "d7a8b9c0..."
}
```
*(Note: Access token lives for 15 mins; Refresh token lives for 7 days).*

### **3. Get Profile**
`GET /v1/users/me`
- **Headers:** `X-User-Id` (Injected by API Gateway after JWT validation).
- **Response (200 OK):**
```json
{
  "id": "usr_9cc9b6",
  "email": "jane.doe@example.com",
  "first_name": "Jane",
  "last_name": "Doe",
  "created_at": "2026-03-07T12:00:00Z"
}
```

## 7. Workflows & Data Consistency

### **Event Publishing on User Creation (Transactional Outbox)**
To ensure the `UserCreated` event is published to Kafka *if and only if* the Postgres insert succeeds:
1. Start Postgres Transaction.
2. Insert new user into `users` table.
3. Insert event payload into `outbox_events` table (Status: `PENDING`).
4. Commit Transaction.
5. An asynchronous background process (or Debezium) sweeps `outbox_events` where `status = PENDING`, publishes them to Kafka topic `domain.users.events`, and updates the status to `PUBLISHED`.

## 8. Security Considerations

- **Password Hashing**: Use **Argon2id** (the memory-hard winner of the Password Hashing Competition) to mitigate GPU-based cracking attacks. Never log passwords in plain text.
- **JWT Generation**: Access tokens are signed using `RS256` (Asymmetric). The private key is securely stored in a Secret Manager (e.g., AWS Secrets Manager, HashiCorp Vault) and loaded into memory on service startup. Only the public key is exposed (via a JWKS endpoint) for the API Gateway to verify signatures.
- **Token Blacklisting (Revocation)**: When a user logs out, the refresh token is marked `is_revoked = true` in Postgres. To immediately invalidate the short-lived access token, its `jti` (JWT ID) is placed in a Redis blacklist (`blacklist:{jti}`) with a TTL equal to the token's remaining lifespan. The API Gateway checks this blacklist during auth.
- **PII Protection**: Fields like `email`, `first_name`, and `last_name` should be encrypted at rest within the PostgreSQL database using Transparent Data Encryption (TDE) or application-level encryption for high-compliance environments.

## 9. Edge Cases

- **Race Conditions on Registration**: Two rapid concurrent requests for the same email but different `Idempotency-Key` values. The Postgres `UNIQUE(email)` constraint throws a violation on the second transaction, which the service catches and translates to a `409 Conflict`.
- **Compromised Refresh Token**: If a user reports suspicious activity, an admin (or the user via another device) can revoke all active sessions by executing `UPDATE refresh_tokens SET is_revoked = true WHERE user_id = ?`.
- **Cache Stampede**: On a popular profile experiencing a cache miss, multiple goroutines/threads might simultaneously query PostgreSQL. Use a single-flight pattern (like Go's `golang.org/x/sync/singleflight`) to ensure only one database query executes while concurrent requests wait for the result.
