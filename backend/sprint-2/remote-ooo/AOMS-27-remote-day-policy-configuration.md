# AOMS-27 — Remote Day Policy Configuration (Backend)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-27 |
| **Epic** | Remote Day Scheduling & OOO — AOMS-22 |
| **Type** | Story |
| **Priority** | 🔴 Highest |
| **Story Points** | 3 |
| **Sprint** | Sprint 2 |
| **Labels** | remote-ooo, config, policy |
| **Perspective** | Backend |

---

## Story

As a manager, I want to configure the remote day limit for my team so that the system enforces or warns about our agreed remote working policy.

---

## Background (Backend Context)

`RemoteDayPolicy` is the policy entity that drives enforcement in AOMS-23 (request submission) and AOMS-25 (manager approval). It must be implemented **before** those tickets can apply policy checks. Policy resolution follows a two-tier lookup: team-level policy (scoped to a `manager_id`) takes precedence over the location-level default. If neither exists, no limit is enforced.

Policy changes take effect immediately for new requests but do not retroactively affect already-approved requests.

**Pipeline:** Jenkins CI — lint → unit test → integration test → Docker build.

---

## Acceptance Criteria (Backend)

- [ ] `RemoteDayPolicy` entity: `id` (UUID), `location_id`, `manager_id` (FK → User, nullable — null = location-level default), `max_remote_days_per_week` (Integer, nullable — null = no limit), `max_remote_days_per_month` (Integer, nullable), `enforcement_mode` (Enum: `HARD_BLOCK`, `SOFT_WARNING`), `team_overlap_threshold_percent` (Integer, nullable — 0–100), `is_active` (Boolean, default true), `created_at`, `updated_at`, `created_by`.
- [ ] Unique constraint on `(location_id, manager_id)` — one policy per manager per location; one default policy per location (where `manager_id IS NULL`).
- [ ] Flyway migration creates `remote_day_policies` table.
- [ ] **`POST /api/v1/remote-day-policies`** — creates or replaces a policy.
  - Manager role: can only create/update their own team policy (`manager_id = authenticatedUserId`).
  - Super Admin: can create location-level default policy (`manager_id = null`) and any team policy.
  - Returns 400 if `max_remote_days_per_week` < 1 or > 7 (when provided).
  - Returns 400 if `team_overlap_threshold_percent` outside 0–100 (when provided).
  - Uses upsert semantics (update if policy exists for same `location_id + manager_id`, insert otherwise).
  - Writes to `AuditLog`.
- [ ] **`GET /api/v1/remote-day-policies/my-team`** — returns the effective policy for the authenticated manager's team: team-level policy if it exists, otherwise the location-level default. Returns `{ "source": "TEAM" | "LOCATION_DEFAULT" | "NONE", "policy": {...} }`.
- [ ] **`GET /api/v1/remote-day-policies?locationId=`** — returns all policies for a location (Super Admin only).
- [ ] **`DELETE /api/v1/remote-day-policies/{policyId}`** — soft-deletes a team policy (sets `is_active = false`). After deletion, the location-level default resumes for that manager's team. Manager can only delete their own; Super Admin can delete any.
- [ ] `RemoteDayPolicyResolver` service bean that other services inject: `RemoteDayPolicyResolver.resolve(managerId, locationId)` returns the effective `RemoteDayPolicy` for a manager's team (team-level → location-level → null). This is called by AOMS-23 and AOMS-25.
- [ ] Unit tests: policy resolution (team exists, team missing → fallback to location, neither → null), upsert behaviour, manager cannot set another manager's policy (403).
- [ ] Integration test: set team policy; call resolver; verify team policy returned. Delete team policy; call resolver; verify location default returned.

---

## Implementation Notes

```
RemoteDayPolicyResolver.resolve(managerId, locationId)
    ├── SELECT * FROM remote_day_policies
    │     WHERE location_id = :locationId
    │       AND is_active = true
    │       AND (manager_id = :managerId OR manager_id IS NULL)
    │     ORDER BY manager_id NULLS LAST   ← team policy wins
    │     LIMIT 1
    └── Return policy or null (no limit)
```

- The single-query resolution (`ORDER BY manager_id NULLS LAST LIMIT 1`) returns the team-level policy if it exists; falls back to the location default; returns null if neither is defined.
- `enforcement_mode` defaults to `SOFT_WARNING` when not specified — never default to `HARD_BLOCK` silently.
- All fields are nullable (except `enforcement_mode`) — a policy can have only a weekly limit, only a monthly limit, only an overlap threshold, or any combination.

---

## API Contract

```
POST /api/v1/remote-day-policies
Request (manager creating own team policy):
{
  "locationId": "uuid",
  "maxRemoteDaysPerWeek": 3,
  "enforcementMode": "HARD_BLOCK",
  "teamOverlapThresholdPercent": 50
}
Response 201:
{ "success": true, "data": { "policyId": "uuid", "managerId": "uuid", "source": "TEAM", ... } }

GET /api/v1/remote-day-policies/my-team
Response 200:
{
  "success": true,
  "data": {
    "source": "TEAM",
    "policy": {
      "policyId": "uuid",
      "maxRemoteDaysPerWeek": 3,
      "enforcementMode": "HARD_BLOCK",
      "teamOverlapThresholdPercent": 50
    }
  }
}
```

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80%
- [ ] `RemoteDayPolicyResolver` bean available for injection by AOMS-23 and AOMS-25
- [ ] Manager cannot set another manager's policy (403 confirmed in test)
- [ ] Audit log entries for create, update, delete
- [ ] Swagger annotations complete
- [ ] Role + location interceptor applied
- [ ] Jenkins pipeline green
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **ATT-P01**, **ATT-P02**, **ATT-P03** must be complete.
- `User.manager_id` populated from Arms HR sync.
- **Must be complete before AOMS-23 and AOMS-25** (those tickets call `RemoteDayPolicyResolver`).
