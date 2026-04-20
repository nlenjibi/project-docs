# OMS Infrastructure — Service Catalogue

| Author | Version | Status | Last Updated |
|--------|---------|--------|--------------|
| DevOps Team | 1.0 | Active | April 2026 |

---

This document describes the configuration, responsibilities, and infrastructure requirements of each microservice in the OMS platform.

All services:
- Run on **ECS Fargate** in the private subnet
- Register with **AWS Cloud Map** at `<service>.oms.local` on startup
- Expose **`/metrics`** for Prometheus scraping
- Expose **`/internal/export`** for the data engineering pipeline
- Use **Java / Spring Boot** as the application runtime
- Are deployed via **Jenkins → ECR → AWS CodeDeploy** (Blue/Green)

---

## 🪑 Attendance Service

### Responsibility

Manages all attendance-related data. Processes badge events (check-in / check-out), maintains `WorkSession` records (a continuous period of presence), and produces `AttendanceRecord` summaries (daily aggregate per employee). Runs a two-pass nightly resolution job to reconcile any missed badge events.

### Database

| Variant | Value |
|---------|-------|
| v2 schema | `attendance` |
| v3 RDS instance | `attendance_db` (db.t3.micro) |

### Endpoints Exposed

**Inbound (via API Gateway → VPC Link):**

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/attendance/events` | Ingest a badge event (check-in/check-out) |
| `GET` | `/api/attendance/sessions` | List work sessions for a date range |
| `GET` | `/api/attendance/records` | Query attendance records |
| `GET` | `/api/attendance/records/{employeeId}` | Get records for a specific employee |
| `GET` | `/metrics` | Prometheus metrics endpoint |

**Internal (Cloud Map, not exposed via API Gateway):**

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/internal/export` | Data engineering export endpoint |
| `GET` | `/internal/health` | Health check for Cloud Map |

### Outbound Dependencies (Cloud Map)

| Service | DNS Name | Reason |
|---------|----------|--------|
| Notification | `notification.oms.local` | Notify employee/manager on attendance anomalies |

### Redis Usage

**No** — Attendance does not use Redis. Attendance records are write-heavy (badge events) and reads are typically historical queries not suited to caching.

### SQS Usage

**No** — Attendance does not enqueue to SQS directly. It calls the Notification Service via Cloud Map when notification is needed.

### ECS Fargate Task Sizing

| Environment | vCPU | Memory | Desired Count |
|-------------|------|--------|---------------|
| dev | 0.25 | 512 MB | 1 |
| staging | 0.5 | 1024 MB | 1–2 |

### Environment Variables

```
SPRING_DATASOURCE_URL
SPRING_DATASOURCE_USERNAME
SPRING_DATASOURCE_PASSWORD
NOTIFICATION_SERVICE_URL      # attendance.oms.local resolved via Cloud Map
AWS_REGION
ENVIRONMENT                   # dev | staging | prod
JWT_ISSUER_URI               # SSO OIDC issuer for token validation
NIGHTLY_RESOLUTION_CRON      # Cron expression for the two-pass job
LOG_LEVEL
```

---

## 📋 Seating Service

### Responsibility

Manages hot-desk booking. Maintains real-time desk availability, floor plans, and booking records. Employees book desks for specific dates/times. Provides availability data to the frontend in real time.

### Database

| Variant | Value |
|---------|-------|
| v2 schema | `seating` |
| v3 RDS instance | `seating_db` (db.t3.micro) |

### Endpoints Exposed

**Inbound (via API Gateway → VPC Link):**

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/seating/availability` | Get real-time desk availability |
| `GET` | `/api/seating/floor-plans` | Get floor plan layout |
| `POST` | `/api/seating/bookings` | Create a desk booking |
| `DELETE` | `/api/seating/bookings/{id}` | Cancel a booking |
| `GET` | `/api/seating/bookings/my` | Get current user's bookings |
| `GET` | `/metrics` | Prometheus metrics endpoint |

**Internal:**

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/internal/export` | Data engineering export |
| `GET` | `/internal/availability` | Availability query for inter-service use |
| `GET` | `/internal/health` | Health check |

### Outbound Dependencies (Cloud Map)

| Service | DNS Name | Reason |
|---------|----------|--------|
| Notification | `notification.oms.local` | Booking confirmation emails |

### Redis Usage

**Yes** — Redis is the primary store for real-time desk availability state.

| Key Pattern | Purpose | TTL |
|-------------|---------|-----|
| `seating:availability:{date}:{deskId}` | Desk booking status | End of day |
| `seating:floor:{floorId}` | Cached floor plan | 1 hour |

### SQS Usage

**No** — Calls Notification Service directly via Cloud Map.

### ECS Fargate Task Sizing

| Environment | vCPU | Memory | Desired Count |
|-------------|------|--------|---------------|
| dev | 0.25 | 512 MB | 1 |
| staging | 0.5 | 1024 MB | 2 |

### Environment Variables

```
SPRING_DATASOURCE_URL
SPRING_DATASOURCE_USERNAME
SPRING_DATASOURCE_PASSWORD
REDIS_HOST                    # ElastiCache Redis endpoint
REDIS_PORT                    # Default 6379
REDIS_KEY_PREFIX              # seating:
NOTIFICATION_SERVICE_URL      # notification.oms.local
AWS_REGION
ENVIRONMENT
JWT_ISSUER_URI
LOG_LEVEL
```

---

## 🏠 Remote/OOO Service

### Responsibility

Manages remote work and out-of-office requests and approvals. Employees submit requests; managers receive approval tasks. Supports manager delegation (designating an alternate approver while manager is away).

### Database

| Variant | Value |
|---------|-------|
| v2 schema | `remote` |
| v3 RDS instance | `remote_ooo_db` (db.t3.micro) |

### Endpoints Exposed

**Inbound (via API Gateway → VPC Link):**

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/remote/requests` | Submit a remote/OOO request |
| `GET` | `/api/remote/requests` | List requests (filtered by status, date) |
| `GET` | `/api/remote/requests/{id}` | Get a specific request |
| `PATCH` | `/api/remote/requests/{id}/approve` | Approve a request |
| `PATCH` | `/api/remote/requests/{id}/reject` | Reject a request |
| `POST` | `/api/remote/delegation` | Set manager delegation |
| `GET` | `/metrics` | Prometheus metrics endpoint |

**Internal:**

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/internal/export` | Data engineering export |
| `GET` | `/internal/health` | Health check |

### Outbound Dependencies (Cloud Map)

| Service | DNS Name | Reason |
|---------|----------|--------|
| Notification | `notification.oms.local` | Request submitted / approved / rejected notifications |

### Redis Usage

**No** — Request state is persisted to RDS. No caching layer required.

### SQS Usage

**No** — Calls Notification Service directly.

### ECS Fargate Task Sizing

| Environment | vCPU | Memory | Desired Count |
|-------------|------|--------|---------------|
| dev | 0.25 | 512 MB | 1 |
| staging | 0.25 | 512 MB | 1 |

### Environment Variables

```
SPRING_DATASOURCE_URL
SPRING_DATASOURCE_USERNAME
SPRING_DATASOURCE_PASSWORD
NOTIFICATION_SERVICE_URL
AWS_REGION
ENVIRONMENT
JWT_ISSUER_URI
LOG_LEVEL
```

---

## 🚶 Visitor Service

### Responsibility

Manages visitor access. Supports pre-registration (host creates a visit in advance), walk-in registration (receptionist registers visitor on arrival), and NDA capture per visit. Generates visitor badges.

### Database

| Variant | Value |
|---------|-------|
| v2 schema | `visitor` |
| v3 RDS instance | `visitor_db` (db.t3.micro) |

### Endpoints Exposed

**Inbound (via API Gateway → VPC Link):**

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/visitor/pre-register` | Pre-register a visitor |
| `POST` | `/api/visitor/walk-in` | Register a walk-in visitor |
| `GET` | `/api/visitor/visits` | List visits (date range) |
| `GET` | `/api/visitor/visits/{id}` | Get visit details |
| `POST` | `/api/visitor/visits/{id}/check-in` | Mark visitor as checked in |
| `POST` | `/api/visitor/visits/{id}/check-out` | Mark visitor as checked out |
| `POST` | `/api/visitor/visits/{id}/nda` | Capture NDA acceptance |
| `GET` | `/metrics` | Prometheus metrics endpoint |

**Internal:**

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/internal/export` | Data engineering export |
| `GET` | `/internal/health` | Health check |

### Outbound Dependencies (Cloud Map)

| Service | DNS Name | Reason |
|---------|----------|--------|
| Notification | `notification.oms.local` | Visitor arrival alert to host; pre-registration confirmation |

### Redis Usage

**No** — Visitor records are low-frequency writes; no caching needed.

### SQS Usage

**No** — Calls Notification Service directly.

### ECS Fargate Task Sizing

| Environment | vCPU | Memory | Desired Count |
|-------------|------|--------|---------------|
| dev | 0.25 | 512 MB | 1 |
| staging | 0.25 | 512 MB | 1 |

### Environment Variables

```
SPRING_DATASOURCE_URL
SPRING_DATASOURCE_USERNAME
SPRING_DATASOURCE_PASSWORD
NOTIFICATION_SERVICE_URL
AWS_REGION
ENVIRONMENT
JWT_ISSUER_URI
LOG_LEVEL
```

---

## 📅 Events Service

### Responsibility

Manages company events. Handles RSVP, capacity limits, and recurring event series using a parent/child record pattern (parent event defines the series; child events are individual occurrences). Prevents overbooking via capacity checks.

### Database

| Variant | Value |
|---------|-------|
| v2 schema | `events` |
| v3 RDS instance | `events_db` (db.t3.micro) |

### Endpoints Exposed

**Inbound (via API Gateway → VPC Link):**

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/events` | Create an event (single or recurring series) |
| `GET` | `/api/events` | List events |
| `GET` | `/api/events/{id}` | Get event details |
| `POST` | `/api/events/{id}/rsvp` | RSVP to an event |
| `DELETE` | `/api/events/{id}/rsvp` | Cancel RSVP |
| `GET` | `/api/events/{id}/attendees` | List confirmed attendees |
| `PATCH` | `/api/events/{id}` | Update event details |
| `DELETE` | `/api/events/{id}` | Cancel an event |
| `GET` | `/metrics` | Prometheus metrics endpoint |

**Internal:**

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/internal/export` | Data engineering export |
| `GET` | `/internal/health` | Health check |

### Outbound Dependencies (Cloud Map)

| Service | DNS Name | Reason |
|---------|----------|--------|
| Notification | `notification.oms.local` | RSVP confirmations, event reminders, cancellation alerts |

### Redis Usage

**No** — Event data is query-heavy but not high-frequency enough to require caching in the initial design.

### SQS Usage

**No** — Calls Notification Service directly.

### ECS Fargate Task Sizing

| Environment | vCPU | Memory | Desired Count |
|-------------|------|--------|---------------|
| dev | 0.25 | 512 MB | 1 |
| staging | 0.5 | 512 MB | 1 |

### Environment Variables

```
SPRING_DATASOURCE_URL
SPRING_DATASOURCE_USERNAME
SPRING_DATASOURCE_PASSWORD
NOTIFICATION_SERVICE_URL
AWS_REGION
ENVIRONMENT
JWT_ISSUER_URI
LOG_LEVEL
```

---

## 📦 Supplies & Assets Service

### Responsibility

Manages office supplies and physical assets. Employees submit supply/asset requests. Requests go through a two-stage approval process (line manager → finance/ops). Tracks batch stock levels and generates low-stock alerts.

### Database

| Variant | Value |
|---------|-------|
| v2 schema | `supplies` |
| v3 RDS instance | `supplies_assets_db` (db.t3.micro) |

### Endpoints Exposed

**Inbound (via API Gateway → VPC Link):**

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/api/supplies/requests` | Submit a supply/asset request |
| `GET` | `/api/supplies/requests` | List requests |
| `GET` | `/api/supplies/requests/{id}` | Get request details |
| `PATCH` | `/api/supplies/requests/{id}/approve` | First-stage approval |
| `PATCH` | `/api/supplies/requests/{id}/final-approve` | Second-stage approval |
| `PATCH` | `/api/supplies/requests/{id}/reject` | Reject a request |
| `GET` | `/api/supplies/inventory` | Query stock levels |
| `PATCH` | `/api/supplies/inventory/{id}` | Update stock level (batch) |
| `GET` | `/metrics` | Prometheus metrics endpoint |

**Internal:**

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/internal/export` | Data engineering export |
| `GET` | `/internal/health` | Health check |

### Outbound Dependencies (Cloud Map)

| Service | DNS Name | Reason |
|---------|----------|--------|
| Notification | `notification.oms.local` | Request submitted / approved / rejected / low-stock alert |

### Redis Usage

**No** — Supply and asset data does not require caching in the initial design.

### SQS Usage

**No** — Calls Notification Service directly.

### ECS Fargate Task Sizing

| Environment | vCPU | Memory | Desired Count |
|-------------|------|--------|---------------|
| dev | 0.25 | 512 MB | 1 |
| staging | 0.25 | 512 MB | 1 |

### Environment Variables

```
SPRING_DATASOURCE_URL
SPRING_DATASOURCE_USERNAME
SPRING_DATASOURCE_PASSWORD
NOTIFICATION_SERVICE_URL
AWS_REGION
ENVIRONMENT
JWT_ISSUER_URI
LOW_STOCK_THRESHOLD           # Minimum stock level before alert is triggered
LOG_LEVEL
```

---

## 🔔 Notification Service

### Responsibility

The async email gateway for all OMS services. Other services call Notification via Cloud Map to request an email. Notification Service enqueues the request to SQS. A consumer within the same service dequeues and calls Amazon SES for delivery. Handles retry logic via SQS visibility timeout and moves permanently failed messages to a Dead Letter Queue (DLQ).

Also manages notification preferences (opt-in/opt-out per category) and rate-limits outbound emails per user.

### Database

| Variant | Value |
|---------|-------|
| v2 schema | `notification` |
| v3 RDS instance | `notification_db` (db.t3.micro) |

### Endpoints Exposed

**Inbound (via API Gateway → VPC Link):**

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/api/notifications/preferences` | Get user notification preferences |
| `PATCH` | `/api/notifications/preferences` | Update preferences (opt-in/opt-out) |
| `GET` | `/metrics` | Prometheus metrics endpoint |

**Internal (Cloud Map — called by other services):**

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/internal/notify` | Enqueue a notification request |
| `GET` | `/internal/export` | Data engineering export |
| `GET` | `/internal/health` | Health check |

### Outbound Dependencies (Cloud Map)

Notification Service is a **leaf service** — it does not call other OMS services. All other services call it.

### Redis Usage

**Yes** — Redis is used for rate limiting outbound email per user.

| Key Pattern | Purpose | TTL |
|-------------|---------|-----|
| `notification:ratelimit:{userId}:{channel}` | Count of emails sent in window | 1 hour (rolling window) |

### SQS Usage

**Yes** — Core to the notification flow.

| Queue | Purpose |
|-------|---------|
| `oms-email-queue` | Async email dispatch — all notification requests |
| `oms-email-dlq` | Dead letter queue — messages that fail after max retries |

**Flow:**
1. Service calls `POST /internal/notify` on Notification Service
2. Notification Service enqueues message to `oms-email-queue`
3. SQS consumer (same service, polling loop) dequeues the message
4. Checks rate limit in Redis
5. Calls Amazon SES to deliver the email
6. On SES failure: message becomes visible again after visibility timeout
7. After max receive count: message moved to `oms-email-dlq`

### ECS Fargate Task Sizing

| Environment | vCPU | Memory | Desired Count |
|-------------|------|--------|---------------|
| dev | 0.25 | 512 MB | 1 |
| staging | 0.5 | 1024 MB | 1–2 |

### Environment Variables

```
SPRING_DATASOURCE_URL
SPRING_DATASOURCE_USERNAME
SPRING_DATASOURCE_PASSWORD
REDIS_HOST
REDIS_PORT
REDIS_KEY_PREFIX              # notification:
AWS_REGION
AWS_SQS_QUEUE_URL             # oms-email-queue URL
AWS_SQS_DLQ_URL              # oms-email-dlq URL
AWS_SES_FROM_ADDRESS         # Verified sender email (SES)
AWS_SES_REGION
EMAIL_RATE_LIMIT_PER_HOUR    # Max emails per user per hour
SQS_VISIBILITY_TIMEOUT       # Seconds before failed message reappears
SQS_MAX_RECEIVE_COUNT        # Retries before DLQ
ENVIRONMENT
JWT_ISSUER_URI
LOG_LEVEL
```

---

## Cross-Service Summary Table

| Service | Redis | SQS | Calls (Cloud Map) | Called By |
|---------|-------|-----|-------------------|-----------|
| Attendance | No | No | Notification | — |
| Seating | Yes | No | Notification | — |
| Remote/OOO | No | No | Notification | — |
| Visitor | No | No | Notification | — |
| Events | No | No | Notification | — |
| Supplies & Assets | No | No | Notification | — |
| Notification | Yes | Yes | — (leaf) | All services |

---

## Data Engineering Export Endpoints

All services expose `GET /internal/export` for the data engineering pipeline. These endpoints are resolved via Cloud Map and are not accessible through API Gateway.

| Service | Export DNS | Typical Data |
|---------|-----------|--------------|
| Attendance | `attendance.oms.local/internal/export` | WorkSessions, AttendanceRecords |
| Seating | `seating.oms.local/internal/export` | Bookings, utilisation rates |
| Remote/OOO | `remote.oms.local/internal/export` | Requests, approvals, delegation events |
| Visitor | `visitor.oms.local/internal/export` | Visits, NDA records |
| Events | `events.oms.local/internal/export` | Events, RSVPs, attendance counts |
| Supplies & Assets | `supplies.oms.local/internal/export` | Requests, stock levels, approvals |
| Notification | `notification.oms.local/internal/export` | Delivery logs, DLQ stats |

Data flows: Service APIs → (optional S3 staging → AWS Glue) → `data_warehouse_db` (RDS PostgreSQL, analytics-only, never writes back to service DBs).
