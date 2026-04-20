# AOMS-4 — Attendance Record Stamping — Pass 1: Badge Events (FastAPI)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-4 |
| **Epic** | Attendance Management — AOMS-1 |
| **Type** | Story |
| **Priority** | 🔴 Highest |
| **Story Points** | 3 |
| **Sprint** | Sprint 1 |
| **Labels** | attendance, batch-job, fastapi |
| **Service** | `attendance-service` (FastAPI / Python 3.12) |

---

## Story

As a system, I want to create AttendanceRecords from resolved WorkSessions so that each employee has an official attendance status for each working day.

---

## Background (Service Context)

Pass 1 is the first of two nightly stamping steps. It runs immediately after `WorkSessionResolutionJob` (AOMS-3) and translates `WorkSession` records into `AttendanceRecord` statuses based on configurable thresholds. Only badge-based statuses (`PRESENT`, `LATE`, `INSUFFICIENT_HOURS`) are assigned here. Pass 2 (AOMS-5) overlays leave, remote, and holiday corrections on top.

Pipeline order: **AOMS-2 (ingest) → AOMS-3 (resolve sessions) → AOMS-4 (Pass 1) → AOMS-5 (Pass 2)**.

The upsert uses PostgreSQL `ON CONFLICT ... DO UPDATE ... WHERE is_overridden = FALSE` so overridden records can never be silently re-stamped by the pipeline.

---

## Acceptance Criteria

### Database Model
- [ ] `AttendanceRecord` SQLAlchemy model with columns: `id` (UUID PK), `user_id` (UUID), `location_id` (UUID), `record_date` (Date), `status` (Enum: `PRESENT`, `LATE`, `ABSENT`, `INSUFFICIENT_HOURS`, `REMOTE`, `ON_LEAVE`, `PUBLIC_HOLIDAY`), `work_session_id` (UUID FK → work_sessions, nullable), `leave_request_id` (UUID, nullable), `remote_request_id` (UUID, nullable), `is_overridden` (Boolean, default False), `override_reason` (String, nullable), `original_status` (Enum, nullable), `overridden_at` (DateTime, nullable), `overridden_by` (UUID, nullable), `created_at`, `updated_at`, `deleted_at`.
- [ ] Alembic migration creates `attendance_records` with unique constraint on `(user_id, record_date, location_id)` where `deleted_at IS NULL`.

### Pass 1 Logic (`AttendancePass1Service`)
- [ ] `AttendancePass1Service.stamp(location_id, date, db_session)` processes all `WorkSession` records for the given location and date.
- [ ] Status determination rules (from `LocationConfig.min_presence_duration_minutes`):
  - `total_duration_minutes >= min_presence_duration_minutes` AND `is_late = False` → `PRESENT`
  - `total_duration_minutes >= min_presence_duration_minutes` AND `is_late = True` → `LATE`
  - `total_duration_minutes < min_presence_duration_minutes` OR `total_duration_minutes is None` → `INSUFFICIENT_HOURS`
- [ ] Records are only created for dates **on or after** `User.employment_start_date`. Earlier dates are silently skipped.
- [ ] Upsert query: `INSERT INTO attendance_records ... ON CONFLICT (user_id, record_date, location_id) DO UPDATE SET status = EXCLUDED.status, work_session_id = EXCLUDED.work_session_id, updated_at = now() WHERE attendance_records.is_overridden = FALSE`.
- [ ] Re-running Pass 1 for the same date is idempotent — no duplicates; no changes to overridden records.
- [ ] `work_session_id` is set on the `AttendanceRecord` when created from a `WorkSession`.
- [ ] Pass 1 run is logged to `attendance_stamp_logs`: `location_id`, `processed_date`, `pass` (1), `records_processed`, `records_created`, `records_updated`, `records_skipped_overridden`, `errors`.

### Tests
- [ ] Unit tests (no DB): `PRESENT` path; `LATE` path; `INSUFFICIENT_HOURS` path (duration below threshold); `INSUFFICIENT_HOURS` path (no `BADGE_OUT` → duration is None); employment start date boundary (date before start → skipped); override protection (is_overridden=True record not touched).
- [ ] Integration test (`testcontainers-python`): seed `WorkSession` records with known values; run Pass 1; assert `AttendanceRecord` statuses match expected; re-run Pass 1; assert no duplicates; assert overridden record is unchanged.

---

## Implementation Notes

```python
class AttendancePass1Service:
    async def stamp(
        self,
        location_id: UUID,
        date: date,
        db: AsyncSession,
    ) -> StampResult:
        config = await self.config_repo.get_by_location(location_id, db)
        sessions = await self.session_repo.find_by_location_and_date(location_id, date, db)
        results = StampResult()

        for session in sessions:
            user = await self.user_repo.get(session.user_id, db)
            if user.employment_start_date > date:
                results.skipped += 1
                continue

            status = self._determine_status(session, config)
            await self.record_repo.upsert_if_not_overridden(
                AttendanceRecordUpsert(
                    user_id=session.user_id,
                    location_id=location_id,
                    record_date=date,
                    status=status,
                    work_session_id=session.id,
                )
            )
            results.processed += 1

        await self.stamp_log_repo.save(location_id, date, pass_number=1, result=results, db=db)
        return results

    def _determine_status(self, session: WorkSession, config: LocationConfig) -> AttendanceStatus:
        if session.total_duration_minutes is None:
            return AttendanceStatus.INSUFFICIENT_HOURS
        if session.total_duration_minutes < config.min_presence_duration_minutes:
            return AttendanceStatus.INSUFFICIENT_HOURS
        return AttendanceStatus.LATE if session.is_late else AttendanceStatus.PRESENT
```

Raw upsert using SQLAlchemy Core for the `WHERE is_overridden = FALSE` condition:
```python
from sqlalchemy.dialects.postgresql import insert

stmt = insert(AttendanceRecord).values(**record_data)
stmt = stmt.on_conflict_do_update(
    index_elements=["user_id", "record_date", "location_id"],
    set_={"status": stmt.excluded.status, "work_session_id": stmt.excluded.work_session_id, "updated_at": func.now()},
    where=(AttendanceRecord.is_overridden == False),
)
await db.execute(stmt)
```

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80%
- [ ] Override protection confirmed via unit test
- [ ] Employment start date boundary confirmed via unit test
- [ ] Upsert SQL reviewed for correctness (no full-table scans; index on `(user_id, record_date, location_id)`)
- [ ] Jenkins pipeline green
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **AOMS-2** and **AOMS-3** must be complete.
- `User.employment_start_date` populated (requires Arms HR sync stub for Sprint 1 tests).
- `location_config` must include `min_presence_duration_minutes`.
