# AOMS-2 — Nightly Badge Event Ingestion (Backend)

> **DEPRECATED (v4.0):** This ticket documents the original Spring Boot implementation. The canonical FastAPI implementation is in [backend/attendance-service/AOMS-2-nightly-badge-event-ingestion.md](attendance-service/AOMS-2-nightly-badge-event-ingestion.md).

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-2 |
| **Epic** | Attendance Management — AOMS-1 |
| **Type** | Story |
| **Priority** | 🔴 Highest |
| **Story Points** | 5 |
| **Sprint** | Sprint 1 |
| **Labels** | attendance, batch-job, aws-athena |
| **Perspective** | Backend |

---

## Story

As a system, I want to query AWS Athena nightly for badge events from the previous day, so that employee attendance data is available for review each morning.

---

## Background (Backend Context)

This is the entry point for the entire attendance pipeline. A Spring `@Scheduled` job queries AWS Athena, ingests raw badge events, and persists them as immutable `BadgeEvent` records. The job must be idempotent — re-running for the same date must not create duplicates. Personnel IDs from Athena are resolved to OMS `User` records via an identity mapping table populated by the Arms HR sync (ATT-004 / AOMS-3 sprint context).

**Messaging note:** SQS is used (not Kafka). On job completion, the job publishes a summary message to the `oms-attendance-sync-status` SQS queue. The no-show job and notification service listen to this queue.

**Pipeline:** Jenkins CI — lint → unit test → integration test → Docker build.

---

## Acceptance Criteria (Backend)

- [ ] `BadgeEvent` entity created with fields: `id` (UUID), `user_id` (FK → User, nullable if unresolved), `location_id` (FK → Location), `personnel_id` (String — raw from Athena), `event_type` (Enum: `BADGE_IN`, `BADGE_OUT`), `occurred_at` (Timestamp), `raw_payload` (JSONB), `ingested_at` (Timestamp). No `deleted_at` — records are immutable.
- [ ] Flyway migration creates the `badge_events` table; a unique constraint on `(personnel_id, occurred_at, location_id)` enforces deduplication at the DB level.
- [ ] `BadgeSyncJob` is a `@Scheduled` Spring component; cron expression configured via `BADGE_SYNC_CRON` environment variable (default `0 2 * * *`).
- [ ] Job queries Athena for badge events where `event_date = yesterday` in the location's configured timezone (read from `LocationConfig.timezone`).
- [ ] Athena query uses **parameterised statements**; no string concatenation on query inputs.
- [ ] `personnel_id` is resolved to `user_id` via a lookup against the `personnel_id_mapping` table (populated by the Arms HR sync). Unresolved IDs are logged as `WARN` and inserted with `user_id = NULL` — the job does not fail.
- [ ] Duplicate events (same `personnel_id`, `occurred_at`, `location_id`) are skipped silently using `INSERT ... ON CONFLICT DO NOTHING`.
- [ ] Job execution result is persisted to a `badge_sync_job_log` table with: `job_id`, `location_id`, `started_at`, `completed_at`, `records_queried`, `records_inserted`, `records_skipped`, `records_failed`, `status` (SUCCESS / PARTIAL / FAILED).
- [ ] On job failure or partial failure, an SQS message is published to `oms-notifications-queue` with `{ "type": "BADGE_SYNC_FAILED", "locationId": "...", "jobId": "..." }` to trigger a Super Admin alert.
- [ ] Job runs per location — one execution per `Location` record with `is_active = true`.
- [ ] All Athena credentials (`ATHENA_DATABASE`, `ATHENA_OUTPUT_BUCKET`, `AWS_REGION`) sourced from environment variables.
- [ ] Unit tests: Athena response parsing, personnel ID resolution, duplicate skipping logic.
- [ ] Integration test: mock Athena client returns sample events; verify correct `BadgeEvent` records are persisted and job log is written.

---

## Implementation Notes

```
BadgeSyncJob (@Scheduled)
    ├── AthenaQueryService.queryBadgeEvents(date, locationId)
    ├── PersonnelIdResolver.resolve(personnelId) → userId or null
    ├── BadgeEventRepository.saveAll() (ON CONFLICT DO NOTHING)
    ├── BadgeSyncJobLogRepository.save(result)
    └── SqsNotificationPublisher.publish() — on failure only
```

- Use `software.amazon.awssdk:athena` (AWS SDK v2) for Athena queries. The query waits for `QueryExecution` to reach `SUCCEEDED` before fetching results — implement a polling loop with a configurable max-wait timeout (`ATHENA_QUERY_TIMEOUT_SECONDS`).
- The job is idempotent: safe to re-run for the same date. Re-running does not duplicate records.
- Run the job per location sequentially (not in parallel) in the first implementation — parallel per-location execution can be added if performance requires it.
- The `raw_payload` JSONB column stores the full Athena row for traceability; it is never used for business logic.

---

## API Endpoints

```
GET  /api/v1/attendance/sync/logs                         → List sync job logs (Super Admin, Facilities Admin)
GET  /api/v1/attendance/sync/logs/{jobId}                 → Single job log detail
POST /api/v1/attendance/sync/trigger                      → Manually trigger sync for a location (Super Admin only)
```

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80% on new code
- [ ] No hardcoded AWS credentials or Athena query strings
- [ ] Parameterised Athena queries confirmed in code review
- [ ] SQS failure notification wired and tested
- [ ] Swagger annotations on all endpoints
- [ ] Role + location interceptor applied on all endpoints
- [ ] Audit log entry written for manual trigger action
- [ ] Jenkins pipeline green
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **ATT-P01**, **ATT-P02**, **ATT-P03** must be complete.
- AWS Athena (`badge_events` database) accessible from the dev environment; schema and sample data provided.
- Arms HR sync must be operational (or stubbed) to populate `personnel_id_mapping` table.
- `LocationConfig` seeded per office with `timezone` field populated.
- SQS queues `oms-attendance-sync-status` and `oms-notifications-queue` provisioned.
