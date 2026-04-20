# AOMS-9 — Attendance Configuration per Location (FastAPI)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-9 |
| **Epic** | Attendance Management — AOMS-1 |
| **Type** | Story |
| **Priority** | 🔴 Highest |
| **Story Points** | 2 |
| **Sprint** | Sprint 2 |
| **Labels** | attendance, config, admin, fastapi |
| **Service** | `attendance-service` (FastAPI / Python 3.12) |

---

## Story

As a facilities admin, I want to configure attendance policy settings for my office so that the system correctly interprets badge data according to our office's rules.

---

## Background (Service Context)

`LocationConfig` is the source of truth for per-office attendance policy. The `attendance-service` maintains its own `location_config` read-model (a local copy), kept in sync via `oms.user` / location event consumers from `auth-service`. Writes to `location_config` happen via the API endpoints in this ticket and are written directly to the `attendance_db`.

Changes take effect from the **next** nightly sync — they do not retroactively re-stamp existing records. The change history is preserved in `location_config_history` (JSONB snapshots) for audit and future retroactive re-stamp support.

All mutations publish an audit event to the `oms-audit` SNS topic.

---

## Acceptance Criteria

### Pydantic Schemas
- [ ] `LocationConfigResponse`: `location_id` (UUID), `work_start_time` (str — "HH:MM"), `lateness_threshold_minutes` (int), `min_presence_duration_minutes` (int), `no_show_release_time` (str), `session_gap_threshold_hours` (int), `hot_desk_booking_window_days` (int), `booking_cancellation_cutoff_hours` (int), `seat_visibility_mode` (str).
- [ ] `LocationConfigPatchRequest`: all fields optional (Pydantic `model_config = ConfigDict(extra="forbid")`); only provided fields are applied.
- [ ] Validators on `LocationConfigPatchRequest` (using Pydantic `@field_validator`):
  - `lateness_threshold_minutes`: 0 ≤ value ≤ 120.
  - `min_presence_duration_minutes`: 1 ≤ value ≤ 480.
  - `session_gap_threshold_hours`: 1 ≤ value ≤ 12.
  - `work_start_time` and `no_show_release_time`: valid `HH:MM` format (use `datetime.strptime(v, "%H:%M")`).
  - `no_show_release_time` must be after `work_start_time` — validated at the service layer after merging with existing config.

### Read Endpoint
- [ ] `GET /api/v1/locations/{location_id}/config` returns `LocationConfigResponse`.
- [ ] Accessible to all authenticated roles (read-only for `EMPLOYEE`, `MANAGER`).
- [ ] Returns 404 if `location_id` not found.

### Update Endpoint
- [ ] `PATCH /api/v1/locations/{location_id}/config` updates one or more config fields.
- [ ] Accessible to `FACILITIES_ADMIN` and `SUPER_ADMIN` only — return 403 otherwise.
- [ ] `FACILITIES_ADMIN` can only update their own location; `SUPER_ADMIN` can update any location.
- [ ] Before applying: write current config state to `location_config_history` (`changed_at`, `changed_by`, `previous_values` JSONB, `new_values` JSONB — only changed fields, not full snapshot of unchanged fields).
- [ ] Apply patch: only fields present in the request body are updated; all other fields retain current values.
- [ ] After applying: validate `no_show_release_time > work_start_time`; return 400 if violated.
- [ ] Publish audit event to SNS topic `oms-audit` (`SNS_TOPIC_ARN_AUDIT=arn:aws:sns:eu-west-1:ACCOUNT:oms-audit`) using `aioboto3` SNS client: `{ actorId, action: "LOCATION_CONFIG_UPDATED", entityType: "LocationConfig", entityId: location_id, previousState: {...}, newState: {...} }`. Publish pattern:
  ```python
  await sns_client.publish(
      TopicArn=settings.SNS_TOPIC_ARN_AUDIT,
      Message=json.dumps(event),
      MessageAttributes={"eventType": {"DataType": "String", "StringValue": event["eventType"]}},
  )
  ```
- [ ] Returns updated `LocationConfigResponse`.

### History Endpoint
- [ ] `GET /api/v1/locations/{location_id}/config/history` returns paginated change history.
- [ ] Accessible to `FACILITIES_ADMIN` and `SUPER_ADMIN`.
- [ ] Response per record: `changed_at`, `changed_by_user_id`, `previous_values` (dict), `new_values` (dict).

### Tests
- [ ] Unit tests: `lateness_threshold_minutes = 121` → 422 Pydantic validation error; `min_presence_duration_minutes = 0` → 422; `no_show_release_time` before `work_start_time` after merge → 400; partial update only changes provided fields; history record created on every update; FACILITIES_ADMIN at wrong location → 403.
- [ ] Integration test: FACILITIES_ADMIN patches config; re-fetch config; verify only patched fields changed; verify history record created; verify audit event published.

---

## Implementation Notes

```python
@router.patch(
    "/api/v1/locations/{location_id}/config",
    response_model=ApiResponse[LocationConfigResponse],
)
async def patch_location_config(
    location_id: UUID,
    patch: LocationConfigPatchRequest,
    current_user: UserContext = Depends(get_current_user),
    _: None = Depends(require_any_role([Role.FACILITIES_ADMIN, Role.SUPER_ADMIN])),
    db: AsyncSession = Depends(get_db),
):
    # Location ownership check
    if Role.SUPER_ADMIN not in current_user.roles:
        if location_id not in current_user.location_ids:
            raise HTTPException(status_code=403)

    config = await config_repo.get_by_location(location_id, db)  # 404 if not found
    previous = config.to_dict()

    # Apply only provided fields
    patch_data = patch.model_dump(exclude_unset=True)
    for field, value in patch_data.items():
        setattr(config, field, value)

    # Cross-field validation
    if time.fromisoformat(config.no_show_release_time) <= time.fromisoformat(config.work_start_time):
        raise HTTPException(status_code=400, detail={"code": "INVALID_TIME_ORDER"})

    # Persist history
    await history_repo.save(location_id, current_user.user_id, previous, patch_data, db)
    await db.flush()

    # Audit event
    await audit_publisher.publish_config_updated(location_id, current_user.user_id, previous, config.to_dict())

    return ApiResponse.ok(LocationConfigResponse.model_validate(config))
```

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80%
- [ ] All validation rules tested (boundary values)
- [ ] History record created on every update (tested)
- [ ] Audit event published for every config change (tested)
- [ ] Partial update confirmed: un-provided fields unchanged (tested)
- [ ] Role access enforced (FACILITIES_ADMIN / SUPER_ADMIN for write; all roles for read)
- [ ] OpenAPI docs complete
- [ ] Jenkins pipeline green
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **ATT-P04** and **ATT-P05** must be running.
- `location_config` table seeded with default values per location before any nightly jobs run.
- SNS topic `oms-audit` (`SNS_TOPIC_ARN_AUDIT`) must be provisioned before audit event publishing.
