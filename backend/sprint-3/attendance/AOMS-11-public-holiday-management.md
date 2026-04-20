# AOMS-11 — Public Holiday Management (FastAPI)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-11 |
| **Epic** | Attendance Management — AOMS-1 |
| **Type** | Story |
| **Priority** | 🔴 Highest |
| **Story Points** | 2 |
| **Sprint** | Sprint 3 |
| **Labels** | attendance, config, admin, fastapi |
| **Service** | `attendance-service` (FastAPI / Python 3.12) |

---

## Story

As a facilities admin, I want to configure public holidays for my office location so that employees are not marked absent on days the office is closed.

---

## Background (Service Context)

`PublicHoliday` records are consumed by the nightly attendance Pass 2 job (AOMS-5) to overlay `PUBLIC_HOLIDAY` status. Holidays must be configured **before** the nightly sync runs for those dates.

Hard deletes are allowed (no soft delete on this entity). Deleting a **past** holiday creates incorrect historical records — it requires re-running Pass 2 for the affected date. This re-stamp is triggered by publishing a message to the SNS topic `oms-attendance` (MessageAttribute `eventType=restamp.requested`) which a consumer in the same service polls from the SQS queue `oms-attendance-restamp` to re-run `AttendancePass2Service.overlay(location_id, date)`.

All mutations publish an audit event to SNS topic `oms-audit`.

---

## Acceptance Criteria

### Database Model
- [ ] `PublicHoliday` SQLAlchemy model: `id` (UUID PK), `location_id` (UUID, not null), `holiday_date` (Date, not null), `name` (str, not null, max_length=100), `created_at`, `created_by` (UUID). **No `deleted_at`** — hard delete only.
- [ ] Unique constraint on `(location_id, holiday_date)`.
- [ ] Alembic migration creates `public_holidays` table.

### Pydantic Schemas
- [ ] `PublicHolidayCreate`: `holiday_date` (date), `name` (str, min_length=1, max_length=100). Validator: `holiday_date` must not be in the past (unless role is `SUPER_ADMIN` — validated at service layer).
- [ ] `PublicHolidayUpdate`: `name` (str, optional), `holiday_date` (date, optional). Past holiday cannot be edited.
- [ ] `PublicHolidayResponse`: `id`, `location_id`, `holiday_date`, `name`, `created_at`.

### Create Endpoint
- [ ] `POST /api/v1/locations/{location_id}/public-holidays` — creates a holiday.
- [ ] Accessible to `FACILITIES_ADMIN` and `SUPER_ADMIN`. Location ownership check for `FACILITIES_ADMIN`.
- [ ] `holiday_date` in the past → 400 (unless `SUPER_ADMIN`).
- [ ] Duplicate `(location_id, holiday_date)` → 409: `{ "code": "HOLIDAY_ALREADY_EXISTS" }`.
- [ ] Returns 201 with `PublicHolidayResponse`.
- [ ] Publishes audit event.

### List Endpoint
- [ ] `GET /api/v1/locations/{location_id}/public-holidays` — returns all holidays for the location sorted by `holiday_date ASC`.
- [ ] Accessible to all authenticated roles (read-only).
- [ ] Optional `year` query param: `?year=2026` — filters by year.

### Update Endpoint
- [ ] `PUT /api/v1/locations/{location_id}/public-holidays/{holiday_id}` — updates `name` or `holiday_date` of a future holiday.
- [ ] Accessible to `FACILITIES_ADMIN` and `SUPER_ADMIN`.
- [ ] Past holidays cannot be edited — return 400: `{ "code": "PAST_HOLIDAY_IMMUTABLE" }`.
- [ ] If updating `holiday_date`, check for duplicate on new date → 409.
- [ ] Publishes audit event.

### Delete Endpoint
- [ ] `DELETE /api/v1/locations/{location_id}/public-holidays/{holiday_id}`.
- [ ] Accessible to `FACILITIES_ADMIN` and `SUPER_ADMIN`.
- [ ] **Future holiday**: delete immediately; return `{ "restamp_queued": false }`.
- [ ] **Past holiday**: only `SUPER_ADMIN` may delete; return 403 if `FACILITIES_ADMIN` attempts. On success: hard delete + publish message to SNS topic `oms-attendance` (MessageAttribute `eventType=restamp.requested`, payload `{ locationId, date }`); return `{ "restamp_queued": true }`.
- [ ] Publishes audit event for both cases.

### Re-stamp Consumer
- [ ] `Restamp` consumer polls SQS queue `oms-attendance-restamp` (subscribed to SNS topic `oms-attendance`, filter policy `eventType=restamp.requested`). Uses `aioboto3` long polling (WaitTimeSeconds=20). Calls `AttendancePass2Service.overlay(location_id, date)` for the affected date. Deletes message on success. Routes to DLQ after maxReceiveCount=3 (configured via Terraform RedrivePolicy). Idempotent via `processed_events`.
- [ ] SQS consumer pattern:
  ```python
  async def poll_sqs(queue_url: str, handler, sqs_client):
      while True:
          response = await sqs_client.receive_message(
              QueueUrl=queue_url,
              MaxNumberOfMessages=10,
              WaitTimeSeconds=20,
              AttributeNames=["All"],
          )
          for msg in response.get("Messages", []):
              try:
                  body = json.loads(msg["Body"])
                  payload = json.loads(body.get("Message", msg["Body"]))
                  await handler(payload)
                  await sqs_client.delete_message(
                      QueueUrl=queue_url, ReceiptHandle=msg["ReceiptHandle"]
                  )
              except Exception as e:
                  logger.error("Failed to process SQS message", error=str(e))
                  # Message returns to queue; routes to DLQ after maxReceiveCount=3
  ```
- [ ] Env vars: `SNS_TOPIC_ARN_ATTENDANCE=arn:aws:sns:eu-west-1:ACCOUNT:oms-attendance`, `SQS_QUEUE_URL_RESTAMP=https://sqs.eu-west-1.amazonaws.com/ACCOUNT/oms-attendance-restamp`

### Tests
- [ ] Unit tests: create holiday (past date → 400 for FACILITIES_ADMIN, allowed for SUPER_ADMIN); duplicate → 409; delete future → `restamp_queued=False`; delete past as FACILITIES_ADMIN → 403; delete past as SUPER_ADMIN → hard delete + message published to SNS; update past holiday → 400.
- [ ] Integration test: create holiday; run Pass 2 for that date; assert employees get `PUBLIC_HOLIDAY` status.

---

## Implementation Notes

```python
@router.delete(
    "/api/v1/locations/{location_id}/public-holidays/{holiday_id}",
    response_model=ApiResponse[HolidayDeleteResponse],
)
async def delete_public_holiday(
    location_id: UUID,
    holiday_id: UUID,
    current_user: UserContext = Depends(get_current_user),
    _: None = Depends(require_any_role([Role.FACILITIES_ADMIN, Role.SUPER_ADMIN])),
    db: AsyncSession = Depends(get_db),
):
    holiday = await holiday_repo.get_or_404(holiday_id, location_id, db)
    is_past = holiday.holiday_date < date.today()

    if is_past and Role.SUPER_ADMIN not in current_user.roles:
        raise HTTPException(status_code=403, detail={"code": "PAST_HOLIDAY_SUPER_ADMIN_ONLY"})

    await holiday_repo.delete(holiday_id, db)

    restamp_queued = False
    if is_past:
        restamp_envelope = {
            "eventType": "restamp.requested",
            "locationId": str(location_id),
            "date": holiday.holiday_date.isoformat(),
        }
        await sns_client.publish(
            TopicArn=settings.SNS_TOPIC_ARN_ATTENDANCE,
            Message=json.dumps(restamp_envelope),
            MessageAttributes={"eventType": {"DataType": "String", "StringValue": "restamp.requested"}},
        )
        restamp_queued = True

    await audit_publisher.publish_holiday_deleted(holiday, current_user.user_id, restamp_queued)
    return ApiResponse.ok(HolidayDeleteResponse(restamp_queued=restamp_queued))
```

Audit events use the same SNS publish pattern targeting `SNS_TOPIC_ARN_AUDIT`:
```python
await sns_client.publish(
    TopicArn=settings.SNS_TOPIC_ARN_AUDIT,
    Message=json.dumps(event_envelope),
    MessageAttributes={"eventType": {"DataType": "String", "StringValue": event_envelope["eventType"]}},
)
```

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80%
- [ ] Past-date delete guard: `FACILITIES_ADMIN` blocked → 403; `SUPER_ADMIN` proceeds (tested)
- [ ] Re-stamp message published to SNS topic `oms-attendance` and SQS consumer implemented and tested
- [ ] Audit events for all mutations published to SNS topic `oms-audit`
- [ ] Duplicate date → 409 (tested)
- [ ] OpenAPI docs complete
- [ ] Jenkins pipeline green
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **AOMS-5** must be complete (`AttendancePass2Service.overlay` must exist for the re-stamp consumer to call).
- SNS topic `oms-attendance` and SQS queue `oms-attendance-restamp` must exist (provisioned via Terraform); DLQ configured via RedrivePolicy (maxReceiveCount=3).
- SNS topic `oms-audit` must exist; env var `SNS_TOPIC_ARN_AUDIT=arn:aws:sns:eu-west-1:ACCOUNT:oms-audit` must be set.
- Env vars: `SNS_TOPIC_ARN_ATTENDANCE=arn:aws:sns:eu-west-1:ACCOUNT:oms-attendance`, `SQS_QUEUE_URL_RESTAMP=https://sqs.eu-west-1.amazonaws.com/ACCOUNT/oms-attendance-restamp`.
