# AOMS-31 — Workplace Service (Epic)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-31 |
| **Type** | Epic |
| **Priority** | 🟡 High |
| **Phase** | Phase 2 |
| **Labels** | workplace, visitor, events, epic |
| **Service** | `workplace-service` (Spring Boot / Java · Port 8088) |

---

## Epic Summary

Manage non-employee presence at the office through two capabilities: **visitor lifecycle management** (pre-registration, walk-in, check-in/out, NDA agreement versioning, audit log) and **office event management** (creation, RSVP, waitlist, recurring series, cancellation). Both capabilities are operated by the same roles (Facilities Admin, HR) at a location and feed into a shared audit trail.

`workplace-service` owns `workplace_db` and produces RabbitMQ events consumed by `notification-service` and `audit-service`.

---

## Business Goals

- Give employees a way to pre-register expected visitors so security teams have advance notice.
- Enable Facilities Admins to check in/out visitors with agreement signature enforcement — visitors always sign the latest NDA version.
- Give HR and Facilities Admins a complete, searchable visitor log for compliance and security incident investigations.
- Allow managers, HR, and Facilities Admins to create office events that employees can discover and RSVP to.
- Automate waitlist promotion so event capacity is never wasted due to manual cancellations.
- Support recurring event series with targeted edit scopes so a change to one instance doesn't accidentally affect an entire series.

---

## Stories

### Visitor Management

| ID | Title | Points |
|---|---|---|
| AOMS-32 | Visitor Registration (Pre-register + Walk-in) | 3 |
| AOMS-33 | Visitor Check-in / Check-out + Agreement Enforcement | 3 |
| AOMS-34 | Agreement Template Management | 2 |
| AOMS-35 | Visitor Audit Log View | 2 |

### Event Management

| ID | Title | Points |
|---|---|---|
| AOMS-36 | Event Creation and Management | 3 |
| AOMS-37 | Event RSVP and Waitlist | 3 |
| AOMS-38 | Recurring Events with Edit Scope Control | 5 |
| AOMS-39 | Event Cancellation with Attendee Notification | 2 |

**Total: 8 stories · 23 story points**

---

## Data Model Overview

```
VisitorProfile (email PK — reusable across visits)
  └── ParentVisit (host_user_id, location_id, expected_date, status)
        └── VisitRecord (check_in_at, check_out_at, agreement_template_version_signed)

AgreementTemplate (version, content, is_active, created_at)

EventSeries (recurrence_pattern, location_id, organizer_id)
  └── Event (title, description, date, capacity, location_id, status: SCHEDULED|CANCELLED)
        └── EventInvite (user_id, status: ATTENDING|WAITLISTED|CANCELLED)
```

---

## Key Constraints

- **Agreement versioning:** On every check-in, compare visitor's `last_signed_version` against the active `AgreementTemplate.version`. If different, a new signature is required before check-in completes.
- **Waitlist promotion:** Transactional — when an `ATTENDING` RSVP is cancelled, the next `WAITLISTED` invite is promoted atomically. Cannot leave two attendees promoted for the same slot.
- **Recurring event edit scopes:** `SINGLE` (one instance), `THIS_AND_FUTURE` (splits the series at this date), `ALL` (modifies entire series). The edit scope must be validated before any mutation.
- **Location scoping:** All visitor and event queries are filtered by `location_id`. Cross-location data is never returned to employees or facilities staff.
- **Role access:** Visitor check-in/out and walk-in registration restricted to `FACILITIES_ADMIN`. Event creation restricted to `MANAGER`, `HR`, `FACILITIES_ADMIN`. All employees can RSVP and view their own visitor history.

---

## RabbitMQ Events Produced

```
oms.workplace.visit.checked_in       → notification-service → host employee notified (in-app + email)
oms.workplace.visit.checked_out      → audit-service
oms.workplace.event.created          → audit-service
oms.workplace.event.rsvp.confirmed   → notification-service → employee confirmation (in-app + email)
oms.workplace.event.rsvp.waitlisted  → notification-service → employee informed of waitlist (in-app)
oms.workplace.event.rsvp.promoted    → notification-service → waitlisted employee notified (in-app + email)
oms.workplace.event.cancelled        → notification-service → all RSVPed attendees notified (in-app + email)
oms.workplace.event.reminder         → notification-service → reminder 24h before event (in-app + email)
```

---

## Acceptance at Epic Level

- [ ] Employee can pre-register an expected visitor; Facilities Admin can register walk-in visitors
- [ ] Check-in enforces agreement version; returning visitor with stale signature must re-sign
- [ ] HR and Facilities Admin can view full visitor log filtered by location and date range
- [ ] Employees can browse upcoming events, RSVP, and cancel RSVP
- [ ] Cancelled RSVP automatically promotes next waitlisted attendee (transactional)
- [ ] Recurring event series manageable with SINGLE / THIS_AND_FUTURE / ALL edit scopes
- [ ] Event cancellation triggers notifications to all RSVPed attendees
- [ ] All state-changing operations write an audit entry via RabbitMQ

---

## Dependencies

- `ATT-P02` (Role + Location Interceptor) on all endpoints
- `ATT-P03` (Audit Log) wired to all state changes
- `ATT-P10` (In-App Notifications) for all workplace notification events
- `ATT-P07` (SES Email) for email channel notifications
- `ATT-P08` (Cloud Map) for `notification-service.oms.local` and `audit-service.oms.local` resolution
