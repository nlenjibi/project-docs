# AOMS-12 — No-Show Reporting for Admin (FastAPI)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-12 |
| **Epic** | Attendance Management — AOMS-1 |
| **Type** | Story |
| **Priority** | 🟠 High |
| **Story Points** | 2 |
| **Sprint** | Sprint 2 |
| **Labels** | attendance, reporting, admin, fastapi |
| **Service** | `attendance-service` (FastAPI / Python 3.12) |

---

## Story

As a facilities admin, I want to view a report of seat booking no-shows so that I can identify patterns and take action where necessary.

---

## Background (Service Context)

`NoShowRecord` entities are created by the `seating-service` no-show auto-release job (AOMS-18) and published via the SNS topic `oms-seating` (MessageAttribute `eventType=booking.released`). The `attendance-service` consumes these events by polling SQS queue `oms-seating-attendance-service` (subscribed to the `oms-seating` SNS topic with a filter policy on `eventType=booking.released`) and stores a local `no_show_record_read_model` table — this avoids cross-service DB queries at report time.

The report API is read-only; no mutations. Only `FACILITIES_ADMIN` and `SUPER_ADMIN` may access it. `MANAGER` and `EMPLOYEE` roles are explicitly blocked.

`no_show_count_in_period` is computed via a SQL subquery in the main report query — not in Python — to avoid N+1.

---

## Acceptance Criteria

### SQS Consumer (prerequisite)
- [ ] Consumer polls SQS queue `oms-seating-attendance-service` (subscribed to SNS topic `oms-seating`, filter policy `eventType=booking.released`). Uses `aioboto3` long polling (WaitTimeSeconds=20). On receipt: upsert `no_show_record_read_model` row:
  `(no_show_record_id, user_id, location_id, booking_date, seat_reference, auto_released_at)`. `seat_reference` is included in the SNS event payload from `seating-service`.
- [ ] Deletes message from SQS on successful processing.
- [ ] Routes to DLQ via Terraform RedrivePolicy after maxReceiveCount=3 on unrecoverable error.
- [ ] Idempotent via `processed_events(event_id)`.
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
- [ ] Env var: `SQS_QUEUE_URL_SEATING=https://sqs.eu-west-1.amazonaws.com/ACCOUNT/oms-seating-attendance-service`

### Pydantic Schemas
- [ ] `NoShowReportRecord`: `no_show_record_id` (UUID), `employee_id` (UUID), `employee_name` (str), `department` (str | None), `booking_date` (date), `seat_reference` (str), `auto_released_at` (datetime), `no_show_count_in_period` (int).
- [ ] `NoShowReportResponse`: `PaginatedResponse[NoShowReportRecord]`.

### List Endpoint
- [ ] `GET /api/v1/attendance/no-shows` returns paginated `NoShowReportRecord` list for the requesting user's location.
- [ ] Role check: `FACILITIES_ADMIN` or `SUPER_ADMIN` only. Return 403 for `MANAGER` and `EMPLOYEE`.
- [ ] `FACILITIES_ADMIN` is scoped to their location; `SUPER_ADMIN` may filter by `location_id` query param.
- [ ] Query params: `from_date` (date), `to_date` (date), `employee_id` (UUID, optional), `department_id` (str, optional), `page`, `size`.
- [ ] `no_show_count_in_period` = count of no-show records for that `user_id` within `[from_date, to_date]` — computed as a SQL correlated subquery or a separate window function within the main query. No per-row Python computation.

### CSV Export Endpoint
- [ ] `GET /api/v1/attendance/no-shows/export` streams a CSV.
- [ ] Same filter params as list endpoint. Max 90-day window.
- [ ] CSV columns: `Employee Name`, `Employee ID`, `Department`, `Booking Date`, `Seat Reference`, `Auto Released At`, `No-Show Count (Period)`.
- [ ] `StreamingResponse` — no in-memory buffer.
- [ ] Response headers: `Content-Type: text/csv`, `Content-Disposition: attachment; filename="no-show-report-{from_date}-{to_date}.csv"`.

### Tests
- [ ] Unit tests: role guard — `MANAGER` → 403; `EMPLOYEE` → 403; `FACILITIES_ADMIN` → 200; `no_show_count_in_period` computed correctly for multiple records per user.
- [ ] Integration test: seed 5 `no_show_record_read_model` rows (2 for same employee); fetch report; verify `no_show_count_in_period = 2` for that employee; verify CSV row count matches.
- [ ] Integration test: consumer idempotency — publish same `booking.released` event twice (via SQS); verify one row in `no_show_record_read_model`.

---

## Implementation Notes

```python
@router.get(
    "/api/v1/attendance/no-shows",
    response_model=ApiResponse[PaginatedResponse[NoShowReportRecord]],
)
async def get_no_show_report(
    from_date: date = Query(...),
    to_date: date = Query(...),
    employee_id: UUID | None = Query(None),
    department_id: str | None = Query(None),
    page: int = Query(0, ge=0),
    size: int = Query(20, ge=1, le=100),
    current_user: UserContext = Depends(get_current_user),
    _: None = Depends(require_any_role([Role.FACILITIES_ADMIN, Role.SUPER_ADMIN])),
    db: AsyncSession = Depends(get_db),
):
    location_id = current_user.primary_location_id  # FACILITIES_ADMIN scope
    records, total = await no_show_repo.get_report(
        location_id=location_id,
        from_date=from_date,
        to_date=to_date,
        employee_id=employee_id,
        department_id=department_id,
        page=page,
        size=size,
        db=db,
    )
    return ApiResponse.ok(PaginatedResponse.of(records, total, page, size))
```

`no_show_count_in_period` via window function — avoids N+1:
```python
count_subq = (
    select(func.count(NoShowReadModel.id))
    .where(
        NoShowReadModel.user_id == NoShowReadModel.user_id,  # correlated
        NoShowReadModel.booking_date.between(from_date, to_date),
        NoShowReadModel.location_id == location_id,
    )
    .correlate(NoShowReadModel)
    .scalar_subquery()
)

stmt = (
    select(NoShowReadModel, UserReadModel, count_subq.label("no_show_count"))
    .join(UserReadModel, NoShowReadModel.user_id == UserReadModel.id)
    .where(NoShowReadModel.location_id == location_id, ...)
    .order_by(desc(NoShowReadModel.booking_date))
    .offset(page * size).limit(size)
)
```

---

## API Contract

```
GET /api/v1/attendance/no-shows?from_date=2026-03-01&to_date=2026-03-31&page=0&size=20

Response 200:
{
  "success": true,
  "data": {
    "content": [
      {
        "noShowRecordId": "uuid",
        "employeeId": "uuid",
        "employeeName": "Jane Doe",
        "department": "Engineering",
        "bookingDate": "2026-03-12",
        "seatReference": "Floor 2 / Zone A / Seat 14",
        "autoReleasedAt": "2026-03-12T10:30:00Z",
        "noShowCountInPeriod": 3
      }
    ],
    "page": 0, "size": 20, "totalElements": 12, "totalPages": 1
  },
  "error": null
}

GET /api/v1/attendance/no-shows/export?from_date=2026-03-01&to_date=2026-03-31
→ 200 CSV stream
```

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80%
- [ ] Role guard confirmed: `MANAGER` and `EMPLOYEE` return 403 (tested)
- [ ] `no_show_count_in_period` computed via SQL (no per-row Python loop) — verified via query logging
- [ ] CSV export streams; no in-memory buffer
- [ ] 90-day export window cap enforced
- [ ] Consumer idempotency tested
- [ ] OpenAPI docs complete
- [ ] Jenkins pipeline green
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **AOMS-18** (No-Show Auto-Release, in `seating-service`) must publish `booking.released` events to SNS topic `oms-seating` before this service has any data to report.
- SQS queue `oms-seating-attendance-service` must be subscribed to SNS topic `oms-seating` with filter policy `eventType=booking.released`; DLQ configured via Terraform RedrivePolicy (maxReceiveCount=3).
- Env var: `SQS_QUEUE_URL_SEATING=https://sqs.eu-west-1.amazonaws.com/ACCOUNT/oms-seating-attendance-service`.
- `users` read-model must include `department` (populated via Cloud Map service discovery, no client library required).
