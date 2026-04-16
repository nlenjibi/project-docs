# AOMS-29 — Team Schedule Calendar View (Backend)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-29 |
| **Epic** | Remote Day Scheduling & OOO — AOMS-22 |
| **Type** | Story |
| **Priority** | 🟠 High |
| **Story Points** | 3 |
| **Sprint** | Sprint 2 |
| **Labels** | remote-ooo, api, manager, calendar |
| **Perspective** | Backend |

---

## Story

As a manager, I want to view a calendar showing my team's remote, OOO, and in-office status so that I can plan team activities and identify scheduling conflicts at a glance.

---

## Background (Backend Context)

The team schedule calendar is a read-only, multi-source aggregation API. For a given date range, it combines data from three tables — `remote_requests`, `ooo_requests`, and `attendance_records` — to produce a per-employee, per-day status view. Future dates (where no `AttendanceRecord` yet exists) rely on approved `RemoteRequest` and `OOORequest` records. Past dates use finalised `AttendanceRecord` statuses.

The overlap threshold highlight (AC-4) requires computing the percentage of the team that is remote on each day — this is a derived metric computed server-side.

HR can view the calendar for any team at their location; they pass a `managerId` query param to scope the view. Managers can only view their own direct reports.

**Pipeline:** Jenkins CI.

---

## Acceptance Criteria (Backend)

- [ ] `GET /api/v1/schedule/team-calendar` returns a per-employee, per-day status grid for the requested date range.
  - Query params: `from` (Date, required), `to` (Date, required — max 35 days span, ~5 weeks), `managerId` (UUID, optional — HR/Super Admin use this to view another manager's team; Manager role ignores it and uses their own `id`).
  - Response structure: `{ dates: ["2026-04-07", ...], members: [{ userId, name, statuses: { "2026-04-07": "REMOTE", "2026-04-08": "IN_OFFICE", ... } }], overlapFlags: { "2026-04-07": true, "2026-04-08": false, ... } }`.
  - Manager role: `managerId` defaults to `authenticatedUserId`; cannot request another manager's team (403 if `managerId` differs from `authenticatedUserId`).
  - HR role: can pass any `managerId` at their location.
  - Super Admin: can pass any `managerId` at any location.

- [ ] **Status resolution per employee per date:**
  - If `AttendanceRecord` exists for that date with a final status → use it (`PRESENT` → `IN_OFFICE`, `LATE` → `IN_OFFICE`, `ABSENT`, `REMOTE`, `ON_LEAVE`, `PUBLIC_HOLIDAY`).
  - If no `AttendanceRecord` exists (future date) → check `RemoteRequest` (APPROVED) for that date → `REMOTE`.
  - If no `RemoteRequest` → check `OOORequest` (APPROVED, date in range) → `ON_LEAVE`.
  - If none of the above → `UNKNOWN` (future date with no data yet — renders as blank/grey in React).

- [ ] **Overlap flag computation per date:** count the number of team members with status `REMOTE` on that date; divide by team size; if > `policy.team_overlap_threshold_percent` → `overlapFlags[date] = true`. If no policy exists, `overlapFlags` values are all `false`.

- [ ] Response must be efficient — a 35-day range for a team of 20 should not require 700 individual DB queries. All data fetched in bulk:
  - One query for `AttendanceRecord` (location + date range + user_ids in team).
  - One query for approved `RemoteRequest` (user_ids + date range).
  - One query for approved `OOORequest` (user_ids + date range).
  - One query for `PublicHoliday` (location + date range).
  - Status resolution happens in Java, not in SQL.

- [ ] `GET /api/v1/schedule/team-calendar/summary?date=2026-04-07` returns a single-day summary suitable for the React day-view header: `{ date, inOffice: 8, remote: 5, onLeave: 2, unknown: 3, overlapExceeded: true }`.

- [ ] Unit tests: future date resolution (REMOTE from approved request), past date uses AttendanceRecord, overlap flag computation, HR viewing another manager's team, Manager cannot view another manager's team (403).
- [ ] Integration test: seed mixed approved requests and attendance records; call calendar API; verify correct statuses per member per date.

---

## Implementation Notes

```
GET /api/v1/schedule/team-calendar?from=2026-04-07&to=2026-04-11
    ↓
TeamCalendarService.build(managerId, locationId, from, to, actor)
    ├── Resolve team: UserRepository.findDirectReports(managerId, locationId) → List<UUID>
    ├── Bulk fetch AttendanceRecords (user_ids, locationId, from-to)
    ├── Bulk fetch approved RemoteRequests (user_ids, locationId, from-to)
    ├── Bulk fetch approved OOORequests (user_ids, locationId, date range intersects from-to)
    ├── Bulk fetch PublicHolidays (locationId, from-to)
    ├── Resolve policy: RemoteDayPolicyResolver.resolve(managerId, locationId)
    ├── Build Map<userId, Map<date, status>>
    │     For each user, for each date in range:
    │       Apply resolution priority (AR > RemoteReq > OOOReq > UNKNOWN)
    ├── Compute overlapFlags: for each date, count REMOTE members / teamSize vs threshold
    └── Map to TeamCalendarResponseDto
```

- `PRESENT` and `LATE` from `AttendanceRecord` both map to `IN_OFFICE` in the calendar response — the calendar shows location state, not punctuality.
- The `INSUFFICIENT_HOURS` status maps to `IN_OFFICE` for calendar purposes too.
- For OOO range overlap: check `oooRequest.startDate <= date AND oooRequest.endDate >= date` for each date when building the map.
- Keep the date range cap at 35 days — return 400 for ranges exceeding this to prevent expensive bulk queries.

---

## API Contract

```
GET /api/v1/schedule/team-calendar?from=2026-04-07&to=2026-04-11

Response 200:
{
  "success": true,
  "data": {
    "dates": ["2026-04-07", "2026-04-08", "2026-04-09", "2026-04-10", "2026-04-11"],
    "members": [
      {
        "userId": "uuid",
        "name": "Jane Doe",
        "statuses": {
          "2026-04-07": "IN_OFFICE",
          "2026-04-08": "REMOTE",
          "2026-04-09": "ON_LEAVE",
          "2026-04-10": "ON_LEAVE",
          "2026-04-11": "UNKNOWN"
        }
      }
    ],
    "overlapFlags": {
      "2026-04-07": false,
      "2026-04-08": true,
      "2026-04-09": false,
      "2026-04-10": false,
      "2026-04-11": false
    }
  },
  "error": null
}

GET /api/v1/schedule/team-calendar/summary?date=2026-04-08
Response 200:
{ "success": true, "data": { "date": "2026-04-08", "inOffice": 8, "remote": 5, "onLeave": 2, "unknown": 3, "overlapExceeded": true } }
```

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80%
- [ ] All bulk queries confirmed — no per-row DB calls in service loop
- [ ] 35-day cap enforced with 400 response
- [ ] Manager cannot view another manager's team (403 confirmed)
- [ ] HR can view any team at their location (confirmed)
- [ ] Overlap flag computation confirmed in test
- [ ] Swagger annotations complete
- [ ] Role + location interceptor applied
- [ ] Jenkins pipeline green
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **AOMS-23** and **AOMS-24** must be complete (RemoteRequest and OOORequest entities).
- **AOMS-25** must be complete (approved requests exist).
- **AOMS-27** must be complete (`RemoteDayPolicyResolver` for overlap threshold).
- **AOMS-4** / **AOMS-5** — `AttendanceRecord` entity.
- **AOMS-11** — `PublicHoliday` entity.
