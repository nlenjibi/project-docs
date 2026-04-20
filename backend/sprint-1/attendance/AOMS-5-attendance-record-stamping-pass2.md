# AOMS-5 — Attendance Record Stamping — Pass 2: Leave, Remote & Holidays (FastAPI)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-5 |
| **Epic** | Attendance Management — AOMS-1 |
| **Type** | Story |
| **Priority** | 🔴 Highest |
| **Story Points** | 3 |
| **Sprint** | Sprint 1 |
| **Labels** | attendance, batch-job, fastapi |
| **Service** | `attendance-service` (FastAPI / Python 3.12) |

---

## Story

As a system, I want to overlay approved leave, remote day, and public holiday records onto attendance data so that employees without badge events are not incorrectly marked as absent.

---

## Background (Service Context)

Pass 2 runs immediately after Pass 1 in the nightly pipeline. It corrects `AttendanceRecord` statuses for employees who have no badge event because they were on approved leave, working remotely, or the office was closed for a public holiday.

**Critical rule:** Pass 2 only touches records where `work_session_id IS NULL`. If a `WorkSession` was resolved in Pass 1 (the employee badged in), the record is authoritative and is never overwritten by Pass 2. An employee who worked a remote day but still came in keeps their badge-based status.

Pass 2 also runs the ABSENT sweep: any active employee at the location with no `AttendanceRecord` for the date (and whose `employment_start_date <= date`) is stamped `ABSENT`.

Data for `ooo_requests`, `remote_requests`, and `public_holidays` is stored in the `attendance_db` database (populated via Amazon SQS consumer events from `remote-service` and `auth-service`). No cross-service REST calls are made during the nightly job.

---

## Acceptance Criteria

### SQS Consumers (prerequisite for Pass 2 to work)
- [ ] Consumer polling `oms-remote-attendance-service` SQS queue (subscribed to `oms-remote` SNS topic), filtering event type `request.approved` → upsert a `remote_request_read_model` row: `(request_id, user_id, dates[], location_id, status=APPROVED)`. Idempotency check via `processed_events` table.
- [ ] Consumer polling `oms-remote-attendance-service` SQS queue, filtering event type `ooo.request.approved` → upsert an `ooo_request_read_model` row: `(request_id, user_id, start_date, end_date, location_id)`.
- [ ] Consumer polling `oms-user-attendance-service` SQS queue (subscribed to `oms-user` SNS topic), filtering event type `user.deactivated` → set `users.is_active = False` for that `user_id`. Stop creating `AttendanceRecord` rows after `employment_end_date`.
- [ ] All consumers are idempotent: check `processed_events(event_id)` before processing; insert into `processed_events` atomically within the same DB transaction as the business operation.
- [ ] Consumers use `aioboto3` with long polling (`WaitTimeSeconds=20`). On unrecoverable error the message is not deleted — it returns to the queue and routes to the DLQ after `maxReceiveCount=3` (configured via SQS `RedrivePolicy` in Terraform/CDK, not in application code).
- [ ] SQS consumer pattern:
  ```python
  async def poll_sqs(queue_url: str, handler, sqs_client):
      while True:
          response = await sqs_client.receive_message(
              QueueUrl=queue_url,
              MaxNumberOfMessages=10,
              WaitTimeSeconds=20,  # long polling
              AttributeNames=["All"],
          )
          for msg in response.get("Messages", []):
              try:
                  body = json.loads(msg["Body"])
                  # SNS wraps the message: actual payload is in body["Message"]
                  payload = json.loads(body.get("Message", msg["Body"]))
                  await handler(payload)
                  await sqs_client.delete_message(
                      QueueUrl=queue_url, ReceiptHandle=msg["ReceiptHandle"]
                  )
              except Exception as e:
                  logger.error("Failed to process SQS message", error=str(e))
                  # Message returns to queue after visibility timeout; routes to DLQ after maxReceiveCount
  ```

### Pass 2 Logic (`AttendancePass2Service`)
- [ ] `AttendancePass2Service.overlay(location_id, date, db_session)` runs after Pass 1 in the same nightly orchestration.
- [ ] **Bulk fetch** (3 queries, not N+1): fetch all `ooo_request_read_model` rows, `remote_request_read_model` rows, and `public_holidays` rows for `(location_id, date)` in three queries before iterating users.
- [ ] For each active user at the location with `AttendanceRecord.work_session_id IS NULL` for the given date, apply overlay in priority order:
  1. `ON_LEAVE` — if an approved OOO covers this user and date; set `leave_request_id`.
  2. `PUBLIC_HOLIDAY` — if a public holiday exists for `(location_id, date)`.
  3. `REMOTE` — if an approved remote request exists for this user and date; set `remote_request_id`.
- [ ] Overlay upsert skips records where `work_session_id IS NOT NULL` or `is_overridden = True`.
- [ ] **ABSENT sweep**: after overlays, any active user at the location with `employment_start_date <= date` and still no `AttendanceRecord` for that date is inserted as `ABSENT` (`work_session_id=None`, `leave_request_id=None`, `remote_request_id=None`).
- [ ] Pass 2 run is logged to `attendance_stamp_logs`: `pass=2`, counts of ON_LEAVE, PUBLIC_HOLIDAY, REMOTE overlays applied, ABSENT records inserted, errors.

### Tests
- [ ] Unit tests (no DB): ON_LEAVE overlay applied; REMOTE overlay applied; PUBLIC_HOLIDAY overlay applied; priority conflict (user has both OOO and public holiday on same date → ON_LEAVE wins); badge-based record (`work_session_id IS NOT NULL`) not touched; overridden record not touched; ABSENT sweep inserts for users with no record; employment start date boundary (date before start → ABSENT not created).
- [ ] Integration test: seed approved OOO, remote request, public holiday, and a badge-based record; run both passes; assert all final statuses correct.
- [ ] Integration test: consumer idempotency — publish same event twice; assert `remote_request_read_model` has one row.

---

## Implementation Notes

```python
class AttendancePass2Service:
    async def overlay(self, location_id: UUID, target_date: date, db: AsyncSession) -> OverlayResult:
        # Bulk fetch — 3 queries total
        ooo_set: dict[UUID, OOOReadModel] = await self.ooo_repo.get_by_location_date(location_id, target_date, db)
        remote_set: dict[UUID, RemoteReadModel] = await self.remote_repo.get_by_location_date(location_id, target_date, db)
        holidays: set[date] = await self.holiday_repo.get_dates_for_location(location_id, db)

        active_users = await self.user_repo.get_active_at_location(location_id, db)
        records_without_session = await self.record_repo.get_without_session(location_id, target_date, db)
        user_ids_with_record: set[UUID] = set()

        for record in records_without_session:
            user_ids_with_record.add(record.user_id)
            if record.is_overridden:
                continue
            new_status, leave_id, remote_id = self._determine_overlay(
                record.user_id, target_date, ooo_set, remote_set, holidays
            )
            if new_status:
                await self.record_repo.update_status(record.id, new_status, leave_id, remote_id, db)

        # ABSENT sweep
        for user in active_users:
            if user.employment_start_date > target_date:
                continue
            if user.id not in user_ids_with_record:
                await self.record_repo.insert_absent(user.id, location_id, target_date, db)

        return result

    def _determine_overlay(self, user_id, date, ooo_set, remote_set, holidays):
        if user_id in ooo_set:
            return AttendanceStatus.ON_LEAVE, ooo_set[user_id].request_id, None
        if date in holidays:
            return AttendanceStatus.PUBLIC_HOLIDAY, None, None
        if user_id in remote_set:
            return AttendanceStatus.REMOTE, None, remote_set[user_id].request_id
        return None, None, None
```

Consumer setup with `aioboto3` (SQS long polling):
```python
async def start_consumers(app: FastAPI) -> None:
    session = aioboto3.Session()
    async with session.client("sqs", region_name=settings.AWS_REGION) as sqs_client:
        app.state.sqs_client = sqs_client

        asyncio.create_task(
            poll_sqs(
                queue_url=settings.SQS_QUEUE_URL_REMOTE,
                handler=handle_remote_approved,
                sqs_client=sqs_client,
            )
        )
        asyncio.create_task(
            poll_sqs(
                queue_url=settings.SQS_QUEUE_URL_USER,
                handler=handle_user_deactivated,
                sqs_client=sqs_client,
            )
        )
```

---

## Environment Variables

```bash
# SNS / SQS (shared with AOMS-2)
SNS_TOPIC_ARN_ATTENDANCE=arn:aws:sns:eu-west-1:ACCOUNT:oms-attendance
SNS_TOPIC_ARN_AUDIT=arn:aws:sns:eu-west-1:ACCOUNT:oms-audit
SQS_QUEUE_URL_REMOTE=https://sqs.eu-west-1.amazonaws.com/ACCOUNT/oms-remote-attendance-service
SQS_QUEUE_URL_USER=https://sqs.eu-west-1.amazonaws.com/ACCOUNT/oms-user-attendance-service
SQS_QUEUE_URL_SEATING=https://sqs.eu-west-1.amazonaws.com/ACCOUNT/oms-seating-attendance-service
SQS_QUEUE_URL_RESTAMP=https://sqs.eu-west-1.amazonaws.com/ACCOUNT/oms-attendance-restamp
AWS_REGION=eu-west-1

# Auth service (AWS Cloud Map DNS — no Eureka client needed; auto-registered via ECS)
AUTH_SERVICE_URL=http://auth-service.oms.local:8081
```

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80%
- [ ] Priority conflict (ON_LEAVE > PUBLIC_HOLIDAY > REMOTE) tested
- [ ] Badge-based record protection confirmed in unit and integration tests
- [ ] ABSENT sweep tested: correct users get ABSENT; employment start date boundary respected
- [ ] Consumer idempotency tested (duplicate event → one DB row)
- [ ] Dead-letter queue confirmed: message with processing error is not deleted, returns to queue, and routes to DLQ after `maxReceiveCount=3` (SQS `RedrivePolicy` configured in Terraform/CDK)
- [ ] Jenkins pipeline green
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **AOMS-4** must be complete (Pass 1 must run first).
- Amazon SQS `oms-remote-attendance-service` queue must exist and be subscribed to the `oms-remote` SNS topic before consumers start.
- `public_holidays` table and CRUD (AOMS-11) must exist.
- `ooo_request_read_model` and `remote_request_read_model` tables created by Alembic migration in this ticket.
