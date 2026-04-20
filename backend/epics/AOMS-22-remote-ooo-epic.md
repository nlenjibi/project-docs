# AOMS-22 — Remote Day Scheduling & OOO (Epic)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-22 |
| **Type** | Epic |
| **Priority** | 🔴 Highest |
| **PI Objective** | PI-OBJ-03 |
| **Labels** | remote, ooo, scheduling, epic |
| **Service** | `remote-service` (Spring Boot / Java) |

---

## Epic Summary

Give employees a structured way to declare remote work days and out-of-office (OOO) periods, submit them for manager approval, and have those decisions flow back as notifications. Managers approve or reject requests, can set team-wide remote limits, and can delegate approval authority during their own absence. The full picture — who is remote, OOO, or in-office on any day — is surfaced in a team schedule calendar consumed by both employees and managers.

---

## Business Goals

- Replace ad-hoc Slack/email remote-day approvals with a tracked, auditable system.
- Enforce weekly remote day limits per team/office to maintain in-office attendance targets.
- Give managers one place to see team schedule coverage before approving further requests.
- Support recurring remote schedules so employees don't re-submit the same request every week.
- Ensure approval authority is never blocked by a manager's own absence via delegation.
- Give employees a single history view for all their requests and outcomes.

---

## PI Objective — PI-OBJ-03

> *Remote day request, approval, policy enforcement, and recurring schedule operational end-to-end. OOO submission, delegation, team calendar, and history view complete.*

PI-OBJ-03 is marked **complete at end of Sprint 3**.

Sprint 2 delivers the foundational request/approval/policy flow. Sprint 3 completes the advanced features: OOO, recurring schedules, delegation, team calendar, and employee history.

---

## Stories

### Sprint 2

| ID | Title | Points |
|---|---|---|
| AOMS-23 | Remote Day Request Submission | 3 |
| AOMS-25 | Manager Request Approval Flow + Notifications | 3 |
| AOMS-27 | Remote Day Policy Configuration | 2 |

### Sprint 3

| ID | Title | Points |
|---|---|---|
| AOMS-26 | Recurring Remote Schedule | 5 |
| AOMS-24 | OOO Request Submission | 3 |
| AOMS-28 | Approval Delegation | 3 |
| AOMS-29 | Team Schedule Calendar View | 3 |
| AOMS-30 | Employee Request History View | 2 |

**Total: 8 stories · 24 story points**

---

## Data Model Overview

```
RemoteDayRequest
  (user_id, request_date, status: PENDING | APPROVED | REJECTED | CANCELLED,
   parent_request_id — null for single; set for recurring child instances,
   manager_id, location_id, created_at)

OooRequest
  (user_id, start_date, end_date, reason, delegate_user_id,
   status: PENDING | APPROVED | REJECTED | CANCELLED)

ApprovalDelegation
  (delegator_user_id, delegate_user_id, start_date, end_date, is_active)

RemoteDayPolicy (per LocationConfig)
  (weekly_remote_day_limit, soft_limit_warning_enabled,
   hard_limit_enforcement_enabled)
```

---

## Request Lifecycle

```
Employee submits request (AOMS-23 / AOMS-24)
    ↓ policy check (AOMS-27 — weekly limit)
PENDING → manager notified via SES (AOMS-25 + ATT-P07)
    ↓
Manager approves / rejects (AOMS-25)
    — OR —
Delegated approver approves / rejects (AOMS-28)
    ↓
Employee notified via SES (ATT-P07)
    ↓
Recurring instances generated if parent approved (AOMS-26)
    ↓
Team schedule calendar updated (AOMS-29)
Employee history updated (AOMS-30)
```

---

## Key Constraints

- **Weekly limit enforcement** (AOMS-27): hard block prevents submission over limit; soft warning allows submission with employee acknowledgement. Limit is configurable per team/office via `LocationConfig`.
- **Recurring schedule parent/child pattern** (AOMS-26): a single manager approval on the parent generates all child instances for the defined recurrence period. Revoking individual child instances does not cancel the parent. This is the most complex data model in PI 1 — schema must be reviewed before implementation begins.
- **OOO vs Remote Day**: OOO blocks attendance calculations for the covered date range (attendance records set to `OOO` status). Remote day requests do not affect attendance calculation — they are scheduling declarations only.
- **Delegation boundary** (AOMS-28): a delegated approver cannot approve their own requests during the delegation period. Delegation period is validated against the delegate's own schedule — a delegate who is also OOO cannot accept delegation.
- **Notification dependency**: all approval flow email notifications (AOMS-25) depend on `ATT-P07` (SES email integration, Sprint 1). REM-002 / AOMS-25 is blocked without ATT-P07.
- **Attendance integration**: approved OOO requests (AOMS-24) must be readable by `attendance-service` to suppress lateness/INSUFFICIENT_HOURS flags for the covered dates. Communication via SQS event (`ooo.request.approved`).

---

## Acceptance at Epic Level

- [ ] Employee can submit single and recurring remote day requests within weekly policy limits
- [ ] Manager receives email notification on new requests; employee receives email on decision
- [ ] Hard limit blocks submission beyond weekly cap; soft limit shows warning and allows proceed
- [ ] Recurring schedule: approve parent → all child instances generated; revoke one child → others unaffected
- [ ] OOO request submitted, approved, and propagated to attendance-service to suppress adverse flags
- [ ] Manager can delegate approval authority for a defined period; delegate cannot approve their own requests
- [ ] Team schedule calendar shows remote / OOO / in-office per day for all team members
- [ ] Employee history view shows all requests with status, filterable by type and date range
- [ ] PI-OBJ-03 signed off by Product Owner at Sprint 3 demo

---

## Dependencies

- `ATT-P07` (SES Email Integration, Sprint 1) — approval flow notifications are blocked without this. Must be confirmed live before Sprint 2 REM-002 / AOMS-25 is marked done.
- `ATT-P02` (Role + Location Interceptor) applied to all remote-service endpoints.
- `ATT-P03` (Audit Log) wired to all approval and delegation state changes.
- `AOMS-9` (Attendance Configuration per Location) — `weekly_remote_day_limit` is stored in `LocationConfig` alongside attendance thresholds.
- `attendance-service` SQS consumer must handle `ooo.request.approved` event to suppress adverse attendance flags for OOO dates.
