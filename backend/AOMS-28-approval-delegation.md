# AOMS-28 — Approval Delegation (Backend)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-28 |
| **Epic** | Remote Day Scheduling & OOO — AOMS-22 |
| **Type** | Story |
| **Priority** | 🔴 Highest |
| **Story Points** | 3 |
| **Sprint** | Sprint 2 |
| **Labels** | remote-ooo, api, delegation |
| **Perspective** | Backend |

---

## Story

As a manager, I want to nominate a delegate to approve requests on my behalf when I am OOO so that my team's requests are not left pending during my absence.

---

## Background (Backend Context)

`ApprovalDelegate` ties a delegate's approval authority to a specific OOO period. Delegation is created as part of (or just after) the manager's own OOO submission. If a manager submits an OOO request without nominating a delegate, the system automatically resolves HR at the same location as the fallback approver at query time — no automatic `ApprovalDelegate` record is created for the HR fallback (it is resolved dynamically).

Delegation expiry is handled by the `effective_to` date matching the OOO `end_date`. A scheduled deactivation job (or a query-time active check) ensures expired delegations no longer route requests to the delegate.

AOMS-23 and AOMS-25 both query `ApprovalDelegateRepository.findActive()` — this entity must be queryable before those tickets are fully complete.

**Messaging:** SQS — `oms-notifications-queue`. **Pipeline:** Jenkins CI.

---

## Acceptance Criteria (Backend)

- [ ] `ApprovalDelegate` entity: `id` (UUID), `manager_id` (FK → User — the manager delegating), `delegate_id` (FK → User — the person receiving delegation), `location_id`, `ooo_request_id` (FK → OOORequest, nullable — null if created independently), `effective_from` (Date), `effective_to` (Date), `is_active` (Boolean, default true), `created_at`, `updated_at`, `created_by`.
- [ ] Flyway migration creates `approval_delegates` table. Unique constraint on `(manager_id, location_id)` WHERE `is_active = true` — a manager can only have one active delegation at a time.
- [ ] **`POST /api/v1/approval-delegates`** — creates a delegation.
  - Request: `{ "delegateId": "uuid", "effectiveFrom": "2026-04-14", "effectiveTo": "2026-04-18", "oooRequestId": "uuid" (optional) }`.
  - Accessible to Manager (own delegation only) and Super Admin.
  - `delegateId` must be a Manager or HR at the same location as the creating manager — return 400 otherwise.
  - `delegateId` cannot be the same as `managerId` — return 400.
  - If an active delegation already exists for this manager, deactivate it first (`is_active = false`, `updated_at = now()`).
  - `effective_from` and `effective_to` must be valid dates; `effective_to >= effective_from`.
  - Publish SQS `{ "type": "DELEGATION_CREATED", "delegateId": "uuid", "managerId": "uuid", "effectiveFrom": "...", "effectiveTo": "...", "locationId": "..." }`.
  - Write to `AuditLog`.
- [ ] **`DELETE /api/v1/approval-delegates/{delegateId}`** — deactivates the manager's current active delegation (`is_active = false`). Manager can only deactivate their own.
- [ ] **`GET /api/v1/approval-delegates/my`** — returns the authenticated manager's current active delegation (or 404 if none).
- [ ] **HR fallback resolution** (no API — internal service method): `ApprovalDelegateRepository.findActiveDelegate(managerId, locationId, date)` — returns the active delegate for the manager on the given date, or null if none. The caller (AOMS-23, AOMS-25) then falls back to HR resolution: `UserRoleRepository.findHRUsersAtLocation(locationId).findFirst()`.
- [ ] **Automatic expiry**: a `DelegationExpiryJob` runs daily (cron configured via `DELEGATION_EXPIRY_CRON` env var, default `0 1 * * *`) and sets `is_active = false` on all `ApprovalDelegate` records where `effective_to < today`. Job is idempotent.
- [ ] Delegation deactivation on OOO return: when `OOORequest.end_date` is reached, the `DelegationExpiryJob` handles deactivation automatically via the `effective_to` date matching.
- [ ] Unit tests: valid delegation created, duplicate active delegation replaced, delegate at wrong location (400), self-delegation (400), expiry job deactivates expired records, HR fallback returns HR user when no delegate exists.
- [ ] Integration test: manager creates delegation; another service queries `findActiveDelegate`; verifies delegate returned; expiry job runs; verifies delegation deactivated.

---

## Implementation Notes

```
ApprovalDelegateRepository:

findActiveDelegate(managerId, locationId, date):
    SELECT * FROM approval_delegates
    WHERE manager_id = :managerId
      AND location_id = :locationId
      AND is_active = true
      AND effective_from <= :date
      AND effective_to >= :date
    LIMIT 1

HR fallback (called when findActiveDelegate returns null):
    SELECT u.id FROM users u
    JOIN user_roles ur ON ur.user_id = u.id
    WHERE ur.role = 'HR'
      AND ur.location_id = :locationId
      AND u.deleted_at IS NULL
    LIMIT 1
```

- The active delegation constraint (`UNIQUE on manager_id, location_id WHERE is_active = true`) prevents accidental double-delegation. The service handles the deactivation of the old record before inserting the new one within a single transaction.
- `DelegationExpiryJob` query: `UPDATE approval_delegates SET is_active = false, updated_at = NOW() WHERE effective_to < CURRENT_DATE AND is_active = true`.

---

## API Contract

```
POST /api/v1/approval-delegates
Request: { "delegateId": "uuid", "effectiveFrom": "2026-04-14", "effectiveTo": "2026-04-18", "oooRequestId": "uuid" }
Response 201:
{
  "success": true,
  "data": {
    "delegationId": "uuid",
    "managerId": "uuid",
    "delegateId": "uuid",
    "delegateName": "Sarah Jones",
    "effectiveFrom": "2026-04-14",
    "effectiveTo": "2026-04-18",
    "isActive": true
  }
}

Response 400 (wrong location): { "success": false, "error": { "code": "INVALID_DELEGATE", "message": "Delegate must be a Manager or HR at the same location." } }

DELETE /api/v1/approval-delegates/my
Response 200: { "success": true, "data": { "deactivated": true } }

GET /api/v1/approval-delegates/my
Response 200: { "success": true, "data": { "delegationId": "...", "delegateName": "...", "effectiveTo": "..." } }
Response 404: { "success": false, "error": { "code": "NO_ACTIVE_DELEGATION" } }
```

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80%
- [ ] `findActiveDelegate` and HR fallback confirmed in unit tests
- [ ] Duplicate active delegation replaced atomically
- [ ] `DelegationExpiryJob` idempotency confirmed
- [ ] SQS notification published to delegate on creation
- [ ] Audit log entries for create and deactivate
- [ ] Swagger annotations complete
- [ ] Role + location interceptor applied (Manager creates own only)
- [ ] Jenkins pipeline green
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **AOMS-24** must exist (OOO request submits delegation flow — the `oooRequestId` FK references `OOORequest`).
- **ATT-P01**, **ATT-P02**, **ATT-P03** must be complete.
- `UserRole` table must have HR role records seeded per location for HR fallback to resolve correctly.
- SQS queue `oms-notifications-queue` provisioned.
- **AOMS-23** and **AOMS-25** depend on `ApprovalDelegateRepository.findActiveDelegate()` from this ticket.
