# AOMS-39 — Event Cancellation with Attendee Notification (Backend)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-39 |
| **Epic** | Workplace Service — AOMS-31 |
| **Type** | Story |
| **Priority** | 🟡 High |
| **Story Points** | 2 |
| **Phase** | Phase 2 |
| **Labels** | workplace, events, cancellation, notification |
| **Service** | `workplace-service` (Spring Boot / Java) |

---

## Story

As an event organizer, I want to cancel an event and have all registered attendees notified immediately so they can adjust their plans.

---

## Acceptance Criteria

- [ ] `DELETE /api/v1/events/{eventId}` (standalone or `editScope=SINGLE` for recurring) cancels an event.
  - Accessible to event organizer, `HR`, `FACILITIES_ADMIN`.
  - Sets `Event.status = CANCELLED`.
  - Collects all `EventInvite` records with `status = ATTENDING` or `WAITLISTED` for the event.
  - For each affected invite: sets `EventInvite.status = CANCELLED`.
  - Publishes ONE `oms.workplace.event.cancelled` event with `{ eventId, locationId, attendeeIds[] }` — a single bulk message, not one per attendee. Notification-service fans out to individual attendees.
  - If no attendees exist: still publishes the event (empty `attendeeIds[]`).
- [ ] Returns 409 if event is already `CANCELLED`.
- [ ] A cancelled event is returned in history queries with `status = CANCELLED` but excluded from `GET /api/v1/events` browse results (browse shows only `SCHEDULED` events).
- [ ] Reminder job: a scheduled daily job publishes `oms.workplace.event.reminder` 24 hours before each `SCHEDULED` event for all `ATTENDING` invites. Uses `attendeeIds[]` bulk format.
- [ ] Unit tests: cancel event with 5 attendees → all invites set to `CANCELLED`; event already cancelled → 409; reminder job only fires for SCHEDULED events 24h away.
- [ ] Integration test: create event with 3 RSVPs; cancel → verify `oms.workplace.event.cancelled` published with 3 `attendeeIds`; verify all 3 `EventInvite` records status = `CANCELLED`.

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80%
- [ ] Bulk `attendeeIds[]` in cancellation event (not per-attendee messages)
- [ ] Reminder job running correctly 24h before scheduled events
- [ ] `oms.workplace.event.cancelled` event published and verified
- [ ] Audit log entry for `EVENT_CANCELLED`
- [ ] Swagger annotations complete
- [ ] Jenkins pipeline green

---

## Dependencies

- **AOMS-36** — events created there.
- **AOMS-37** — RSVP invites created there.
- **ATT-P10** (In-App Notifications) — notification-service fans out to attendees.
