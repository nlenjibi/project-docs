# AOMS-10 — Super Admin Attendance Override (FastAPI)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-10 |
| **Epic** | Attendance Management — AOMS-1 |
| **Type** | Story |
| **Priority** | 🟠 High |
| **Story Points** | 2 |
| **Sprint** | Sprint 2 |
| **Labels** | attendance, admin, audit, fastapi |
| **Service** | `attendance-service` (FastAPI / Python 3.12) |

---

## Story

As a super admin, I want to manually override an employee's attendance record so that genuine errors in the system can be corrected.

---

## Background (Service Context)

Override is the only mechanism that allows a human to change a system-computed status. Auditability is the primary constraint: both the original and new status are stored on the record, the override reason is mandatory, and a full before/after JSONB snapshot is published to the `oms-audit` SNS topic.

Once `is_overridden = True`, the nightly Pass 1 and Pass 2 jobs skip that record unconditionally — this is enforced by the upsert SQL in AOMS-4 and AOMS-5. A revert endpoint restores the `original_status` and clears all override fields, allowing future nightly jobs to re-stamp the record normally.

**Super Admin is global** — no location check. The only role check is `SUPER_ADMIN`.

---

## Acceptance Criteria

### Override Endpoint
- [ ] `PATCH /api/v1/attendance/{record_id}/override` — updates the status of an `AttendanceRecord`.
- [ ] Request body (`OverrideRequest`): `new_status` (AttendanceStatus, required), `reason` (str, required, min_length=1, max_length=500).
- [ ] `reason` is required — Pydantic validation; empty string or missing → 422.
- [ ] Only `SUPER_ADMIN` can call this endpoint — return 403 for any other role.
- [ ] If `record_id` does not exist → 404.
- [ ] If `is_overridden = True` already → return HTTP 409:
  ```json
  { "success": false, "error": { "code": "ALREADY_OVERRIDDEN", "message": "Record is already overridden. Revert before overriding again." } }
  ```
- [ ] On success, set on the `AttendanceRecord`: `status = new_status`, `is_overridden = True`, `override_reason = reason`, `original_status = current status`, `overridden_at = utcnow()`, `overridden_by = current_user.user_id`.
- [ ] Returns the updated `AttendanceRecordDetailResponse`.

### Revert Endpoint
- [ ] `PATCH /api/v1/attendance/{record_id}/revert` — reverts an overridden record.
- [ ] Request body (`RevertRequest`): `reason` (str, required).
- [ ] Only `SUPER_ADMIN` can call this endpoint.
- [ ] If record is not currently overridden (`is_overridden = False`) → 409:
  ```json
  { "success": false, "error": { "code": "NOT_OVERRIDDEN", "message": "Record is not overridden." } }
  ```
- [ ] On success: `status = original_status`, `is_overridden = False`, `override_reason = None`, `original_status = None`, `overridden_by = None`, `overridden_at = None`.
- [ ] Returns the reverted `AttendanceRecordDetailResponse`.

### Audit Events
- [ ] Both override and revert publish an audit event to SNS topic `oms-audit` via `aioboto3` (MessageAttribute `eventType=attendance.record.overridden` or `eventType=attendance.record.reverted`):
  ```json
  {
    "eventId": "uuid",
    "eventType": "attendance.record.overridden",
    "version": "1",
    "correlationId": "from X-Correlation-ID header",
    "occurredAt": "ISO8601",
    "actorId": "super_admin_user_id",
    "actorRole": "SUPER_ADMIN",
    "action": "ATTENDANCE_OVERRIDDEN",
    "entityType": "AttendanceRecord",
    "entityId": "record_id",
    "locationId": "record.location_id",
    "previousState": { ...full record before change... },
    "newState": { ...full record after change... }
  }
  ```
- [ ] Publish via `aioboto3`:
  ```python
  await sns_client.publish(
      TopicArn=settings.SNS_TOPIC_ARN_AUDIT,
      Message=json.dumps(event_envelope),
      MessageAttributes={"eventType": {"DataType": "String", "StringValue": event_envelope["eventType"]}},
  )
  ```
- [ ] Env var: `SNS_TOPIC_ARN_AUDIT=arn:aws:sns:eu-west-1:ACCOUNT:oms-audit`
- [ ] `previousState` and `newState` are full JSONB snapshots of the `AttendanceRecord` — not just the changed fields.

### Tests
- [ ] Unit tests: successful override — all fields set correctly; override on already-overridden record → 409; missing `reason` → 422; non-SUPER_ADMIN role → 403; non-existent `record_id` → 404.
- [ ] Unit tests: successful revert — all fields cleared; revert on non-overridden record → 409.
- [ ] Integration test: SUPER_ADMIN overrides a record; verify `is_overridden=True`; verify `original_status` stored; verify audit event published to SNS topic `oms-audit`; run nightly Pass 1 stub; verify overridden record not touched.
- [ ] Integration test: SUPER_ADMIN reverts; verify `status` restored to `original_status`; verify `is_overridden=False`; verify audit event published.

---

## Implementation Notes

```python
@router.patch(
    "/api/v1/attendance/{record_id}/override",
    response_model=ApiResponse[AttendanceRecordDetailResponse],
)
async def override_attendance_record(
    record_id: UUID,
    body: OverrideRequest,
    current_user: UserContext = Depends(get_current_user),
    _: None = Depends(require_role(Role.SUPER_ADMIN)),
    db: AsyncSession = Depends(get_db),
):
    record = await record_repo.get_or_404(record_id, db)

    if record.is_overridden:
        raise HTTPException(
            status_code=409,
            detail={"code": "ALREADY_OVERRIDDEN", "message": "Record is already overridden. Revert before overriding again."},
        )

    previous_state = record.to_dict()
    record.original_status = record.status
    record.status = body.new_status
    record.is_overridden = True
    record.override_reason = body.reason
    record.overridden_at = datetime.utcnow()
    record.overridden_by = current_user.user_id
    await db.flush()

    event_envelope = {
        "eventId": str(uuid4()),
        "eventType": "attendance.record.overridden",
        "version": "1",
        "correlationId": correlation_id,
        "occurredAt": datetime.utcnow().isoformat(),
        "actorId": str(current_user.user_id),
        "actorRole": "SUPER_ADMIN",
        "action": "ATTENDANCE_OVERRIDDEN",
        "entityType": "AttendanceRecord",
        "entityId": str(record_id),
        "locationId": str(record.location_id),
        "previousState": previous_state,
        "newState": record.to_dict(),
    }
    await sns_client.publish(
        TopicArn=settings.SNS_TOPIC_ARN_AUDIT,
        Message=json.dumps(event_envelope),
        MessageAttributes={"eventType": {"DataType": "String", "StringValue": event_envelope["eventType"]}},
    )

    return ApiResponse.ok(AttendanceRecordDetailResponse.model_validate(record))
```

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80%
- [ ] `SUPER_ADMIN` role enforced — no other role can call override or revert (tested)
- [ ] Audit event published for both override and revert (tested end-to-end with SNS topic `oms-audit`)
- [ ] `is_overridden` field returned on all attendance read API responses (AOMS-6, 7, 8 must include this field — verify)
- [ ] Nightly job override-protection confirmed: integration test seeds overridden record, runs Pass 1 stub, verifies record unchanged
- [ ] OpenAPI docs complete
- [ ] Jenkins pipeline green
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **AOMS-4** and **AOMS-5** must be complete (override protection must be implemented there first — the `WHERE is_overridden = FALSE` clause in the upsert SQL).
- SNS topic `oms-audit` must exist; env var `SNS_TOPIC_ARN_AUDIT=arn:aws:sns:eu-west-1:ACCOUNT:oms-audit` must be set.
- `overridden_at` and `overridden_by` columns must be in the `attendance_records` Alembic migration (add in this ticket's migration if not already present from AOMS-4).
