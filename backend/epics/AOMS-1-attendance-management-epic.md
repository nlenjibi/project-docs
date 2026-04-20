# AOMS-1 — Attendance Management (Epic)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-1 |
| **Type** | Epic |
| **Priority** | 🔴 Highest |
| **PI Objective** | PI-OBJ-01 |
| **Labels** | attendance, epic |
| **Service** | `attendance-service` (FastAPI / Python 3.12) |

---

## Epic Summary

Enable the organisation to track employee attendance automatically and accurately across all office locations. Attendance data is derived from badge events — not manual entry. The system computes work sessions, applies configurable lateness and duration thresholds, and surfaces attendance status to employees, managers, and HR.

This epic covers the full lifecycle: badge event ingestion → work session resolution → attendance record generation → self-service views → manager dashboard → HR reporting → admin configuration → exception management.

---

## Business Goals

- Replace manual attendance tracking with automated badge-event-driven records.
- Give employees visibility into their own attendance history from their employment start date.
- Give managers a real-time dashboard flagging lateness, insufficient hours, and no-shows for their teams.
- Give HR a cross-organisation view for compliance and reporting.
- Allow super admins to correct exceptional cases without modifying underlying badge data.
- Support configurable thresholds per location so offices with different working norms are handled correctly.

---

## PI Objective — PI-OBJ-01

> *Attendance tracking operational end-to-end: badge events ingested nightly, attendance records generated with configurable thresholds, employee self-view and manager dashboard live.*

PI-OBJ-01 is marked **complete at end of Sprint 2**. Sprint 3 stories deliver admin-tier extensions (HR org view, super-admin override, public holidays) that extend the feature set without blocking the PI objective.

---

## Stories

### Sprint 1

| ID | Title | Points |
|---|---|---|
| ATT-004 | Arms HR Integration | 3 |
| AOMS-2 | Nightly Badge Event Ingestion | 3 |
| AOMS-3 | Work Session Resolution | 3 |
| AOMS-4 | Attendance Record Stamping — Pass 1 | 3 |
| AOMS-5 | Attendance Record Stamping — Pass 2 | 2 |
| AOMS-6 | Employee Attendance Self-View | 2 |

### Sprint 2

| ID | Title | Points |
|---|---|---|
| AOMS-7 | Manager Team Attendance View | 3 |
| AOMS-9 | Attendance Configuration per Location | 2 |
| AOMS-12 | No-Show Reporting for Admin | 2 |

### Sprint 3

| ID | Title | Points |
|---|---|---|
| AOMS-8 | HR Org-Wide Attendance View | 3 |
| AOMS-10 | Super Admin Attendance Override | 2 |
| AOMS-11 | Public Holiday Management | 2 |

**Total: 12 stories · 30 story points**

---

## Data Flow

```
Arms HR System
    ↓ (nightly sync — ATT-004)
employees table (employment_start_date, manager_id, is_active)

AWS S3 / Athena (badge events)
    ↓ (nightly ingestion — AOMS-2)
badge_events table
    ↓ (session grouping — AOMS-3)
work_sessions table
    ↓ (threshold application — AOMS-4, AOMS-5)
attendance_records table
    ↓
Employee self-view (AOMS-6)
Manager dashboard (AOMS-7)
HR org view (AOMS-8)
Admin config (AOMS-9)
No-show report (AOMS-12)
Super-admin override (AOMS-10)
Public holiday exclusion (AOMS-11)
```

---

## Key Constraints

- `employment_start_date` from Arms HR is the hard lower bound for all attendance history queries. No records are shown before this date.
- All thresholds (lateness, minimum duration, session gap) are configurable per location via `LocationConfig` (AOMS-9). Defaults must be set at location creation.
- Badge events are ingested from S3 — not a real-time stream. The attendance record for a given day is not final until the nightly job completes.
- Super-admin overrides (AOMS-10) write to `AuditLog` and do not modify underlying `BadgeEvent` or `WorkSession` data.
- Public holidays (AOMS-11) exclude the covered date from lateness and INSUFFICIENT_HOURS calculations — the attendance record for that day is set to `PUBLIC_HOLIDAY` status.

---

## Acceptance at Epic Level

- [ ] Nightly badge event pipeline running in production with CloudWatch alerting on failure
- [ ] Attendance records generated for all active employees at all active locations
- [ ] Employee self-view available from day 1 of employment
- [ ] Manager dashboard showing real-time team attendance flags (lateness, no-shows, INSUFFICIENT_HOURS)
- [ ] HR can view attendance cross-organisation with export capability
- [ ] Configurable thresholds working at boundary values across multiple locations
- [ ] Super-admin can override a record with audit trail
- [ ] Public holidays excluded from adverse attendance flags
- [ ] PI-OBJ-01 signed off by Product Owner at Sprint 2 demo

---

## Dependencies

- Arms HR API access and credentials (ART dependency — must be escalated at PI Planning if unresolved)
- AWS S3 bucket and Athena query access for badge event ingestion
- `ATT-P08` (Cloud Map) must be deployed before `attendance-service` can resolve `auth-service.oms.local`
- `ATT-P07` (SES email) required for sync failure alerts (AOMS-12 / ATT-008)
