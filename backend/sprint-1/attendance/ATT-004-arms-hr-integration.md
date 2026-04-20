# ATT-004 — Arms HR Integration (attendance-service)

| Field | Value |
|-------|-------|
| **Ticket** | ATT-004 |
| **Epic** | Attendance Management — AOMS-1 |
| **Type** | Story |
| **Priority** | 🔴 Highest |
| **Story Points** | 3 |
| **Sprint** | Sprint 1 |
| **Labels** | attendance, hr, integration, arms, fastapi |
| **Service** | `attendance-service` (FastAPI / Python 3.12) |

---

## Story

As the attendance service, I want to sync employee records, employment start dates, and org structure from the Arms HR system so that attendance calculations are anchored to the correct employment period and team hierarchy is available for dashboard filtering.

---

## Background (Service Context)

Arms HR is the source of truth for employee identity, employment start dates, reporting lines (`manager_id`), and active/inactive status. The `attendance-service` maintains a local read-replica of the subset of HR data it needs — it does not call Arms HR on every request.

A nightly sync job pulls the Arms HR employee delta and upserts into the local `employees` table. This table is the authoritative source for `employment_start_date` (used as the lower bound in all attendance history queries), `manager_id` (used for team-level dashboard grouping), and `is_active` (used to exclude leavers from dashboard counts).

**Stub strategy:** If Arms HR API access is not available by Sprint 1 day 3, stub the `ArmsHrClient` interface with a `MockArmsHrClient` that returns a fixed set of employees from a JSON fixture. The stub must be replaceable with zero code changes to callers — swap via FastAPI `Depends` override. Sprint 2 DoD requires the real integration replacing the stub.

**Pipeline:** Jenkins CI.

---

## Acceptance Criteria

### Employee Data Model (local replica)
- [ ] `employees` table created with columns: `id` (UUID, PK), `arms_employee_id` (varchar, unique — the Arms HR primary key), `email` (varchar, unique), `full_name` (varchar), `manager_id` (UUID, nullable, FK → `employees.id`), `location_id` (UUID), `employment_start_date` (date), `is_active` (bool, default true), `created_at` (timestamptz), `updated_at` (timestamptz), `deleted_at` (timestamptz, nullable).
- [ ] Alembic migration: `0002_create_employees_table.py`.
- [ ] Unique index on `arms_employee_id`; index on `location_id`; index on `manager_id`.

### Arms HR Client Interface
- [ ] `ArmsHrClient` abstract base class defined in `app/hr/client.py`:
  ```python
  class ArmsHrClient(ABC):
      @abstractmethod
      async def fetch_employees(self, location_id: str, since: datetime | None) -> list[ArmsEmployeeDto]: ...
  ```
- [ ] `ArmsEmployeeDto`: `arms_employee_id`, `email`, `full_name`, `manager_arms_id` (str | None), `location_id`, `employment_start_date` (date), `is_active` (bool).
- [ ] `HttpArmsHrClient(ArmsHrClient)` implementation:
  - Base URL from `settings.ARMS_HR_API_URL` env var.
  - Auth header from `settings.ARMS_HR_API_KEY` env var (stored in AWS Secrets Manager; injected via ECS task definition secret reference — never committed to code).
  - `since` parameter maps to `?updated_after=<ISO-8601>` query param; omit for full sync.
  - Raises `ArmsHrClientError` on HTTP 4xx/5xx; caller retries up to 3 times with exponential backoff.
- [ ] `MockArmsHrClient(ArmsHrClient)` implementation:
  - Reads from `tests/fixtures/arms_employees.json`.
  - Activated when `settings.ARMS_HR_USE_MOCK = true` (default false in production, true in CI until real credentials land).

### Nightly Sync Job
- [ ] `ArmsHrSyncJob` scheduled via APScheduler (cron: `0 1 * * *` — 01:00 UTC nightly) or triggered by the same S3 event mechanism used in AOMS-2.
- [ ] On each run:
  1. Read `last_sync_at` from `sync_checkpoints` table (key: `arms_hr`).
  2. Call `ArmsHrClient.fetch_employees(location_id=*, since=last_sync_at)` — all locations, delta since last sync. Full sync if `last_sync_at` is null.
  3. For each `ArmsEmployeeDto`: upsert into `employees` on conflict `arms_employee_id`. Set `is_active = false` for employees absent from the delta who were previously active (soft deactivation).
  4. Resolve `manager_id`: after all upserts, update `employees.manager_id` by joining on `employees.arms_employee_id = dto.manager_arms_id`. Null if manager not found in local table.
  5. Write `last_sync_at = now()` to `sync_checkpoints`.
  6. Emit structured log: `arms_hr_sync_complete`, `employees_upserted`, `employees_deactivated`, `duration_ms`.
- [ ] If `ArmsHrClient.fetch_employees` raises `ArmsHrClientError` after retries: log `ERROR arms_hr_sync_failed`; do NOT update `sync_checkpoints`; raise so the job framework marks the run as failed (triggering CloudWatch alert from ATT-008).
- [ ] Sync job is idempotent: running it twice with the same delta produces the same result.

### Downstream Consumer Contract
- [ ] `attendance-service` queries `employees.employment_start_date` by `user_id` in AOMS-6 self-view and AOMS-7 manager-view to enforce the attendance history lower bound. No direct Arms HR API call per request.
- [ ] `employees.manager_id` is used by AOMS-7 and AOMS-8 to build team groupings. Unresolved `manager_id` (null) means the employee is a top-level manager or the manager record has not yet synced — treat as ungrouped, not as an error.
- [ ] `employees.is_active = false` employees are excluded from manager dashboard counts (AOMS-7) but their historical records are preserved and readable by HR (AOMS-8).

### Tests
- [ ] Unit test: `ArmsHrSyncJob` with `MockArmsHrClient` — upsert two employees; run again with one changed name and one new employee; assert three employees in table, name updated, `last_sync_at` updated.
- [ ] Unit test: `ArmsHrClientError` on fetch → `sync_checkpoints` not updated; error logged.
- [ ] Unit test: manager resolution — `manager_arms_id` present in payload → `manager_id` resolved after upsert; `manager_arms_id` absent → `manager_id` null.
- [ ] Unit test: deactivation — employee present in previous sync but absent from delta → `is_active = false`.
- [ ] Integration test: seed fixture employees via `MockArmsHrClient`; run sync job; verify AOMS-6 self-view returns records only from `employment_start_date` onward.

---

## Implementation Notes

```python
# app/hr/sync.py
class ArmsHrSyncJob:
    def __init__(self, client: ArmsHrClient, db_factory: Callable):
        self.client = client
        self.db_factory = db_factory

    async def run(self) -> None:
        async with self.db_factory() as db:
            checkpoint = await sync_checkpoint_repo.get("arms_hr", db)
            dtos = await self.client.fetch_employees(since=checkpoint.last_sync_at)

            incoming_ids = {dto.arms_employee_id for dto in dtos}
            for dto in dtos:
                await employee_repo.upsert(dto, db)

            # Deactivate employees absent from delta (only on full sync or known-complete delta)
            if checkpoint.last_sync_at is None:
                await employee_repo.deactivate_absent(incoming_ids, db)

            # Resolve manager_id FK after all upserts complete
            await employee_repo.resolve_managers(db)

            checkpoint.last_sync_at = datetime.utcnow()
            await sync_checkpoint_repo.save(checkpoint, db)
```

- `sync_checkpoints` table: `key` (varchar PK), `last_sync_at` (timestamptz nullable).
- Deactivation on incremental sync: only deactivate employees returned in a previous full sync if they are absent from a subsequent FULL sync. Incremental deltas may not include all active employees — do not deactivate on incremental runs unless the Arms HR API explicitly marks an employee as inactive in the payload.

---

## API Contract

Arms HR is consumed by the sync job only — no new public API endpoints are added by this ticket. The `employees` table becomes a shared dependency for AOMS-6, AOMS-7, and AOMS-8.

---

## Environment Variables

```bash
ARMS_HR_API_URL=https://arms-hr.yourorg.internal/api/v2
ARMS_HR_API_KEY=<from AWS Secrets Manager — never hardcoded>
ARMS_HR_USE_MOCK=false   # set true in CI until real credentials land
```

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80%
- [ ] `MockArmsHrClient` active in CI (`ARMS_HR_USE_MOCK=true`); real `HttpArmsHrClient` used in staging/prod
- [ ] `ArmsHrClientError` path tested: sync fails, checkpoint not updated, error logged
- [ ] Manager resolution tested: `manager_id` FK correctly resolved after upsert
- [ ] Deactivation logic tested: leavers set `is_active = false`, historical records preserved
- [ ] `ARMS_HR_API_KEY` sourced from Secrets Manager — not in env file, not in code
- [ ] Alembic migration reviewed and applied to staging DB
- [ ] Sync job CloudWatch log group emitting structured entries
- [ ] Jenkins pipeline green
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **AOMS-2** (badge event ingestion) must exist — `sync_checkpoints` table pattern reused.
- **ATT-008** (sync monitoring and failure alerts) configures the CloudWatch alarm that fires when this job fails.
- Arms HR API credentials must be in Secrets Manager before `HttpArmsHrClient` can be used. Stub with `MockArmsHrClient` until then.
- **AOMS-6** and **AOMS-7** depend on `employees.employment_start_date` and `employees.manager_id` being populated by this job.
