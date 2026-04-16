# ATT-P03 — Audit Log Infrastructure (Backend)

| Field | Value |
|-------|-------|
| **Ticket** | ATT-P03 |
| **Epic** | Platform / Auth — PI-OBJ-04 |
| **Type** | Story |
| **Priority** | 🔴 Highest |
| **Story Points** | 3 |
| **Sprint** | Sprint 1 |
| **Labels** | platform, audit, compliance |
| **Perspective** | Backend |

---

## Story

As the platform, I want an immutable audit log infrastructure so that every state-changing operation in the system is permanently recorded with actor, timestamp, and state delta for compliance and traceability.

---

## Background (Backend Context)

The audit log is the compliance backbone of the OMS. Every create, update, delete, and status change must produce an `AuditLog` record. This ticket builds the infrastructure (entity, write service, and SQS-based async write) — individual feature services will call it as they are built.

**Messaging note:** SQS is used instead of Kafka for async audit event dispatch. The audit write service publishes to an SQS queue; a listener persists the record. This decouples the audit write from the request thread and ensures that an audit persistence failure never blocks the business operation.

**Pipeline:** Jenkins CI validates lint → unit test → integration test → Docker build on every PR.

---

## Acceptance Criteria (Backend)

- [ ] `AuditLog` entity created with fields: `id` (UUID), `actor_id`, `actor_role`, `action` (String), `entity_type`, `entity_id` (UUID), `location_id`, `previous_state` (JSONB), `new_state` (JSONB), `occurred_at`.
- [ ] `AuditLog` table has no `deleted_at` column — records are never soft-deleted.
- [ ] Database migration creates the table with the application DB user granted INSERT privilege only — no UPDATE, no DELETE at the DB level.
- [ ] `AuditLogService.log(AuditLogEntry entry)` is the single write entry point for all features.
- [ ] `AuditLogService` publishes an SQS message to the `oms-audit-events` queue rather than writing synchronously — the business transaction is not blocked by audit persistence.
- [ ] An `SqsAuditEventListener` (Spring Cloud AWS `@SqsListener`) consumes from the queue and persists the `AuditLog` record.
- [ ] The listener is idempotent: duplicate SQS delivery (same `eventId`) results in exactly one persisted record.
- [ ] SQS queue URL is configured via `AUDIT_SQS_QUEUE_URL` environment variable — not hardcoded.
- [ ] `GET /api/v1/audit-logs` — paginated query; filterable by `actorId`, `entityType`, `entityId`, `locationId`, `action`, date range; accessible to HR, Facilities Admin, Super Admin roles only.
- [ ] Audit log records are never returned for locations outside the requesting user's role scope (Super Admin sees all).
- [ ] `AuditLogService` is exposed as a Spring bean so all feature services can inject and call it.
- [ ] Unit tests: verify SQS message is published with correct payload on `log()` call.
- [ ] Integration test: full end-to-end from `log()` call → SQS publish → listener consumes → record persisted in DB.

---

## Implementation Notes

```
Feature Service
    ↓ AuditLogService.log(entry)
    ↓ SQS publish (oms-audit-events queue)
    ↓ SqsAuditEventListener (@SqsListener)
    ↓ AuditLogRepository.save()
    ↓ audit_logs table (INSERT only)
```

```java
// SQS message structure
{
  "eventId": "uuid-v4",           // For deduplication
  "actorId": "uuid",
  "actorRole": "MANAGER",
  "action": "SEAT_BOOKED",
  "entityType": "SeatBooking",
  "entityId": "uuid",
  "locationId": "uuid",
  "previousState": null,
  "newState": { ... },
  "occurredAt": "2026-03-01T09:00:00Z"
}
```

- Use `spring-cloud-aws-messaging` for SQS integration. Configure `AmazonSQSAsync` bean from `AWS_REGION` and `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` environment variables (or IAM role in prod).
- Store the SQS `eventId` in a `processed_audit_events` table to enforce idempotency in the listener.
- JSONB columns (`previous_state`, `new_state`) mapped with `@JdbcTypeCode(SqlTypes.JSON)`.
- Retention archival job (move records > 24 months to S3) is **out of scope for this ticket** but the schema must support it.

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80% on new code
- [ ] SQS queue URL configured via env var; no hardcoded AWS config
- [ ] Swagger annotations on audit query endpoint
- [ ] Role + location interceptor (ATT-P02) applied to audit query endpoint
- [ ] Jenkins pipeline green
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **ATT-P01** and **ATT-P02** must be complete.
- AWS SQS queue (`oms-audit-events`) provisioned in the dev account.
- `AuditLogService` bean must be available before any feature ticket that requires audit logging can be marked done.
