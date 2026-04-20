# AOMS-3 — Work Session Resolution (FastAPI)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-3 |
| **Epic** | Attendance Management — AOMS-1 |
| **Type** | Story |
| **Priority** | 🔴 Highest |
| **Story Points** | 5 |
| **Sprint** | Sprint 1 |
| **Labels** | attendance, batch-job, fastapi |
| **Service** | `attendance-service` (FastAPI / Python 3.12) |

---

## Story

As a system, I want to collapse raw badge events into WorkSession records so that each employee's daily presence is represented as a coherent block of time.

---

## Background (Service Context)

`WorkSession` is the intermediate layer between raw immutable `BadgeEvent` records and the business-level `AttendanceRecord`. This separation keeps badge-event resolution logic testable independently from status-determination logic, and makes the full trace from raw event → session → record auditable.

This job runs as the second step in the nightly pipeline, immediately after AOMS-2 completes. It uses Python's `datetime` and `timedelta` for all time arithmetic — no external library needed. All logic is pure Python; database I/O is isolated in repository methods so the resolution engine is unit-testable with plain data classes.

---

## Acceptance Criteria

### Database Model
- [ ] `WorkSession` SQLAlchemy model with columns: `id` (UUID PK, default `uuid4`), `user_id` (UUID, not null), `location_id` (UUID, not null), `session_date` (Date — date of `first_badge_in`), `first_badge_in` (DateTime tz-aware, not null), `last_badge_out` (DateTime tz-aware, nullable), `total_duration_minutes` (Integer, nullable), `is_late` (Boolean, not null), `minutes_late` (Integer, nullable), `crosses_midnight` (Boolean, not null, default False), `created_at` (DateTime, server default now), `updated_at` (DateTime, onupdate now), `deleted_at` (DateTime, nullable — soft delete).
- [ ] Alembic migration creates `work_sessions` table with unique constraint on `(user_id, session_date, location_id)` where `deleted_at IS NULL`.

### Resolution Service
- [ ] `WorkSessionResolutionService.resolve(user_id, location_id, date, badge_events, location_config)` is a **pure function** — it takes data objects as arguments and returns a list of `WorkSessionData` dataclasses. No database calls inside this function.
- [ ] Resolution logic:
  - Sort `badge_events` chronologically by `occurred_at`.
  - Split into groups: start a new group when the gap between consecutive events exceeds `location_config.session_gap_threshold_hours`.
  - For each group: `first_badge_in` = earliest event timestamp; `last_badge_out` = latest `BADGE_OUT` timestamp (None if no `BADGE_OUT` exists in group).
  - `total_duration_minutes` = `int((last_badge_out - first_badge_in).total_seconds() / 60)` when `last_badge_out` is not None; else None.
  - `crosses_midnight` = `last_badge_out.date() != first_badge_in.date()` when `last_badge_out` is not None.
  - `session_date` = `first_badge_in.date()` (always the date of session start, even for cross-midnight sessions).
  - `is_late` = `first_badge_in.timetz() > (work_start + timedelta(minutes=lateness_threshold))`.
  - `minutes_late` = `int((first_badge_in - work_start_dt).total_seconds() / 60)` when `is_late` is True, else None.
- [ ] Sessions with no `BADGE_OUT` are not discarded — they are persisted with `last_badge_out = None` and `total_duration_minutes = None`.
- [ ] Resolution is idempotent: re-running for the same `(user_id, location_id, date)` soft-deletes the existing `WorkSession` for that date and inserts a fresh one. The old record's `deleted_at` is set to now; a new record is inserted.

### Job Orchestration
- [ ] `WorkSessionResolutionJob` is triggered by `BadgeSyncJob` (AOMS-2) on completion of each location run, or can be triggered manually via API.
- [ ] Job iterates over all `(user_id, location_id)` pairs that have `BadgeEvent` records from the most recent sync run (identified by `ingested_at` timestamp).
- [ ] Job execution is logged to a `work_session_resolution_logs` table: `location_id`, `processed_date`, `users_processed` (int), `sessions_created` (int), `errors` (int).
- [ ] `LocationConfig` (session gap threshold, work start time, lateness threshold) is fetched once per location per job run via a single DB query — not per user.

### Tests
- [ ] Unit tests (no DB): cross-midnight session produces `crosses_midnight=True`; gap exceeding threshold splits into two sessions; no `BADGE_OUT` produces `last_badge_out=None`; `is_late` computed correctly; `minutes_late=None` when not late; idempotent re-run replaces old session.
- [ ] Unit test: single employee with 6 badge events (3 in, 3 out) across a gap → 2 sessions created.
- [ ] Integration test (`testcontainers-python` PostgreSQL): seed `BadgeEvent` records; run job; assert `WorkSession` records created with correct field values.

---

## Implementation Notes

```python
# Pure resolution engine — no DB I/O, fully unit-testable
@dataclass
class WorkSessionData:
    user_id: UUID
    location_id: UUID
    session_date: date
    first_badge_in: datetime
    last_badge_out: datetime | None
    total_duration_minutes: int | None
    is_late: bool
    minutes_late: int | None
    crosses_midnight: bool


class WorkSessionResolutionService:
    def resolve(
        self,
        user_id: UUID,
        location_id: UUID,
        badge_events: list[BadgeEvent],
        location_config: LocationConfig,
    ) -> list[WorkSessionData]:
        if not badge_events:
            return []
        sorted_events = sorted(badge_events, key=lambda e: e.occurred_at)
        groups = self._split_by_gap(sorted_events, location_config.session_gap_threshold_hours)
        return [self._build_session(user_id, location_id, group, location_config) for group in groups]

    def _split_by_gap(self, events, gap_hours):
        groups, current = [], [events[0]]
        for prev, curr in zip(events, events[1:]):
            gap = (curr.occurred_at - prev.occurred_at).total_seconds() / 3600
            if gap > gap_hours:
                groups.append(current)
                current = []
            current.append(curr)
        groups.append(current)
        return groups
```

Repository upsert pattern (idempotent):
```python
async def upsert_session(self, session: AsyncSession, data: WorkSessionData) -> None:
    # Soft-delete existing session for same (user_id, session_date, location_id)
    await session.execute(
        update(WorkSession)
        .where(
            WorkSession.user_id == data.user_id,
            WorkSession.session_date == data.session_date,
            WorkSession.location_id == data.location_id,
            WorkSession.deleted_at.is_(None),
        )
        .values(deleted_at=func.now())
    )
    # Insert fresh session
    session.add(WorkSession(**asdict(data)))
    await session.flush()
```

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 85% on the resolution engine (critical path)
- [ ] Cross-midnight, gap-threshold, and no-`BADGE_OUT` scenarios explicitly covered in unit tests
- [ ] No audit log required (system computation, not a user action)
- [ ] No SQS message published (pipeline-internal step)
- [ ] Jenkins pipeline green
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **AOMS-2** must be complete — `BadgeEvent` records must exist before sessions can be resolved.
- `location_config` table must include `work_start_time`, `lateness_threshold_minutes`, `session_gap_threshold_hours`.
