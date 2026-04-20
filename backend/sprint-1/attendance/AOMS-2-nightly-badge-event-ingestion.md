# AOMS-2 — Nightly Badge Event Ingestion (FastAPI)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-2 |
| **Epic** | Attendance Management — AOMS-1 |
| **Type** | Story |
| **Priority** | 🔴 Highest |
| **Story Points** | 5 |
| **Sprint** | Sprint 1 |
| **Labels** | attendance, batch-job, aws-athena, fastapi |
| **Service** | `attendance-service` (FastAPI / Python 3.12) |

---

## Story

As a system, I want to query AWS Athena nightly for badge events from the previous day, so that employee attendance data is available for review each morning.

---

## Background (Service Context)

This is the entry point for the entire attendance pipeline. A scheduled job (`APScheduler`) queries AWS Athena, ingests raw badge events, and persists them as immutable `BadgeEvent` rows. The job must be idempotent — re-running for the same date must not create duplicates.

Personnel IDs from Athena are resolved to OMS `user_id` values via a `personnel_id_mapping` table, populated by the Arms HR sync in `auth-service`. Unresolved IDs are logged as warnings; the job continues.

On job failure, a message is published to the SNS topic `oms-attendance` (event type `attendance.sync.failed`) so the notification-service can alert Super Admin.

**API Gateway:** Spring Cloud Gateway routes `/api/v1/attendance/**` to this service. The internal JWT injected by the gateway is validated on every request via a FastAPI `Depends()` security dependency.

---

## Acceptance Criteria

### Database Model
- [ ] `BadgeEvent` SQLAlchemy model with columns: `id` (UUID PK), `user_id` (UUID, nullable FK → users), `location_id` (UUID, not null), `personnel_id` (String, not null — raw from Athena), `event_type` (Enum: `BADGE_IN`, `BADGE_OUT`), `occurred_at` (DateTime with timezone), `raw_payload` (JSONB), `ingested_at` (DateTime, server default = now). **No `deleted_at`** — records are immutable once inserted.
- [ ] Alembic migration creates `badge_events` table with a unique constraint on `(personnel_id, occurred_at, location_id)`.
- [ ] `BadgeSyncJobLog` model: `id` (UUID), `location_id` (UUID), `started_at`, `completed_at`, `records_queried` (int), `records_inserted` (int), `records_skipped` (int), `records_failed` (int), `status` (Enum: `SUCCESS`, `PARTIAL`, `FAILED`).

### Scheduled Job
- [ ] `BadgeSyncJob` is registered with `APScheduler` (`AsyncIOScheduler`) on app startup. Cron expression is configured via `BADGE_SYNC_CRON` environment variable (default `0 2 * * *` — 2am daily).
- [ ] The scheduler is started in the FastAPI `lifespan` context manager — not in a `@app.on_event` handler (deprecated).
- [ ] Job iterates over all active locations (fetched from `attendance_db.location_config`) sequentially. Parallel per-location execution is deferred to a future ticket.
- [ ] For each location, the job queries Athena for `event_date = yesterday` in the location's configured timezone (`LocationConfig.timezone`).

### Athena Integration
- [ ] Athena queries use `aioboto3` (async AWS SDK). Query uses a parameterised approach — date is passed as a query parameter, not string-concatenated.
- [ ] A polling loop waits for `QueryExecution.Status` to reach `SUCCEEDED`. Max wait is `ATHENA_QUERY_TIMEOUT_SECONDS` (default 120). If exceeded, the job marks that location's run as `FAILED` and continues to the next location.
- [ ] `ATHENA_DATABASE`, `ATHENA_OUTPUT_BUCKET`, `AWS_REGION` are all sourced from environment variables. Fail fast at startup if any are missing.

### Ingestion Logic
- [ ] `personnel_id` is resolved to `user_id` via the `personnel_id_mapping` table. Unresolved IDs are logged as `WARNING` and inserted with `user_id = NULL` — the job does not fail.
- [ ] Duplicate events (same `personnel_id`, `occurred_at`, `location_id`) are skipped using `INSERT ... ON CONFLICT (personnel_id, occurred_at, location_id) DO NOTHING`.
- [ ] `raw_payload` JSONB column stores the full Athena row for traceability. It is never used in business logic.
- [ ] Job result is persisted to `badge_sync_job_logs` after each location run.

### Failure Notification
- [ ] On `FAILED` or `PARTIAL` status, the job publishes to SNS topic `oms-attendance` (event type `attendance.sync.failed`), payload:
  ```json
  { "eventId": "uuid", "eventType": "attendance.sync.failed", "version": "1",
    "locationId": "uuid", "jobId": "uuid", "occurredAt": "ISO8601" }
  ```
- [ ] SNS publishing uses `aioboto3`. The SNS topic ARN (`SNS_TOPIC_ARN_ATTENDANCE`) and AWS region (`AWS_REGION`) are sourced from environment variables.
- [ ] Publish uses the standard SNS publish pattern:
  ```python
  async def publish_to_sns(sns_client, topic_arn: str, message: dict):
      await sns_client.publish(
          TopicArn=topic_arn,
          Message=json.dumps(message),
          MessageAttributes={
              "eventType": {"DataType": "String", "StringValue": message.get("eventType", "")}
          },
      )
  ```

### Manual Trigger API
- [ ] `POST /api/v1/attendance/sync/trigger` — manually triggers the sync for a given `locationId`. Body: `{ "locationId": "uuid" }`. Accessible to `SUPER_ADMIN` only. Returns `202 Accepted` immediately; job runs asynchronously via `BackgroundTasks`.
- [ ] `GET /api/v1/attendance/sync/logs` — returns paginated sync job logs. Accessible to `SUPER_ADMIN` and `FACILITIES_ADMIN`.
- [ ] `GET /api/v1/attendance/sync/logs/{job_id}` — returns a single job log detail.
- [ ] All responses use the standard envelope: `{ "success": true/false, "data": ..., "error": null }`.

### Tests
- [ ] Unit tests: Athena response parsing, `personnel_id` resolution (found and not-found paths), duplicate skipping, failure notification publishing.
- [ ] Integration test: mock `aioboto3` Athena client returns sample rows; assert correct `BadgeEvent` rows inserted; assert `BadgeSyncJobLog` written with correct counts.
- [ ] Integration test uses `testcontainers-python` (`PostgreSQLContainer`) for a real DB — no SQLite.

---

## Implementation Notes

```
lifespan (startup)
    → AsyncIOScheduler.start()
    → aioboto3_session = aioboto3.Session()   # SNS/SQS clients created per-request

BadgeSyncJob (APScheduler cron)
    → for each active location:
        AthenaService.query_badge_events(date, location_id)  # aioboto3
        PersonnelIdResolver.resolve(personnel_id)            # local DB lookup
        BadgeEventRepository.bulk_insert_on_conflict(events)
        BadgeSyncJobLogRepository.save(result)
        if FAILED: SNSPublisher.publish(SNS_TOPIC_ARN_ATTENDANCE, attendance.sync.failed, payload)
```

Key modules:
```
attendance_service/
├── main.py                    # FastAPI app, lifespan, router registration
├── scheduler.py               # APScheduler setup, job registration
├── models/
│   ├── badge_event.py         # SQLAlchemy ORM model
│   └── badge_sync_job_log.py
├── repositories/
│   ├── badge_event_repo.py
│   └── badge_sync_log_repo.py
├── services/
│   ├── athena_service.py      # aioboto3 Athena query + polling
│   ├── personnel_resolver.py  # personnel_id → user_id lookup
│   └── sync_job_service.py    # orchestrates the full sync
├── messaging/
│   └── publisher.py           # aioboto3 SNS publisher
├── routers/
│   └── sync.py                # /api/v1/attendance/sync/...
├── schemas/
│   └── sync.py                # Pydantic request/response models
├── security/
│   └── jwt.py                 # Internal JWT validation Depends()
└── db/
    └── session.py             # SQLAlchemy async engine + session factory
```

Internal JWT validation (reused by all routers):
```python
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from jose import jwt, JWTError

bearer_scheme = HTTPBearer()

async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(bearer_scheme),
) -> UserContext:
    try:
        payload = jwt.decode(
            credentials.credentials,
            settings.INTERNAL_JWT_SECRET,
            algorithms=["HS256"],
        )
        return UserContext(**payload)
    except JWTError:
        raise HTTPException(status_code=status.HTTP_401_UNAUTHORIZED)
```

---

## Environment Variables

```bash
# Scheduling
BADGE_SYNC_CRON="0 2 * * *"

# Athena
ATHENA_DATABASE=badge_events
ATHENA_OUTPUT_BUCKET=s3://oms-athena-results/
AWS_REGION=eu-west-1
ATHENA_QUERY_TIMEOUT_SECONDS=120

# Database
DATABASE_URL=postgresql+asyncpg://attendance_app:${DB_PASSWORD}@attendance-db:5432/attendance_db

# SNS / SQS
SNS_TOPIC_ARN_ATTENDANCE=arn:aws:sns:eu-west-1:ACCOUNT:oms-attendance
SNS_TOPIC_ARN_AUDIT=arn:aws:sns:eu-west-1:ACCOUNT:oms-audit
SQS_QUEUE_URL_REMOTE=https://sqs.eu-west-1.amazonaws.com/ACCOUNT/oms-remote-attendance-service
SQS_QUEUE_URL_USER=https://sqs.eu-west-1.amazonaws.com/ACCOUNT/oms-user-attendance-service
SQS_QUEUE_URL_SEATING=https://sqs.eu-west-1.amazonaws.com/ACCOUNT/oms-seating-attendance-service
SQS_QUEUE_URL_RESTAMP=https://sqs.eu-west-1.amazonaws.com/ACCOUNT/oms-attendance-restamp

# Auth service (AWS Cloud Map DNS — no Eureka client needed; auto-registered via ECS)
AUTH_SERVICE_URL=http://auth-service.oms.local:8081

# Security
INTERNAL_JWT_SECRET=${INTERNAL_JWT_SECRET}   # from Secrets Manager
```

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80% on new code
- [ ] No hardcoded AWS credentials — fail fast at startup if missing
- [ ] Parameterised Athena queries confirmed (no string concatenation on user-controlled input)
- [ ] SNS failure notification tested end-to-end
- [ ] Swagger docs auto-generated by FastAPI; manual trigger endpoint has `summary`, `description`, and response examples
- [ ] Role check enforced on all endpoints (`SUPER_ADMIN` for trigger, `SUPER_ADMIN`/`FACILITIES_ADMIN` for logs)
- [ ] Audit event published to SNS topic `oms-audit` on manual trigger
- [ ] Jenkins pipeline green (lint → test → Docker build → ECR push)
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **ATT-P04** (API Gateway) must be running to route requests to this service.
- **ATT-P05** (AWS Cloud Map) must be running; this service is auto-registered via ECS — no client library required.
- AWS Athena `badge_events` database accessible from the dev environment; schema and sample data must be provided.
- Arms HR sync must populate `personnel_id_mapping` table (or stub with test data for Sprint 1).
- `location_config` table seeded with `timezone` per location.
- Amazon SQS provisioned; `oms-attendance` SNS topic and associated SQS queues created.
