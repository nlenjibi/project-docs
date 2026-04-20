# AOMS-36 — Event Creation and Management (Backend)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-36 |
| **Epic** | Workplace Service — AOMS-31 |
| **Type** | Story |
| **Priority** | 🟡 High |
| **Story Points** | 3 |
| **Phase** | Phase 2 |
| **Labels** | workplace, events, api |
| **Service** | `workplace-service` (Spring Boot / Java) |

---

## Story

As a Manager, HR, or Facilities Admin, I want to create and manage office events so that employees can discover, plan for, and attend relevant gatherings.

---

## Acceptance Criteria

- [ ] `POST /api/v1/events` — creates a new event.
  - Accessible to `MANAGER`, `HR`, `FACILITIES_ADMIN`.
  - Request: `{ title, description, locationId, eventDate, startTime, endTime, capacity, isRecurring (bool) }`.
  - If `isRecurring = true`: also requires `recurrencePattern` (`{ frequency: WEEKLY|MONTHLY, dayOfWeek, endDate }`). Creates an `EventSeries` and the first `Event` child instance.
  - If `isRecurring = false`: creates a standalone `Event` with `event_series_id = null`.
  - Validates `capacity > 0`; `eventDate` must not be in the past.
  - Publishes `oms.workplace.event.created` → audit-service.
- [ ] `PATCH /api/v1/events/{eventId}` — updates an event.
  - Accessible to original organizer, `HR`, `FACILITIES_ADMIN`.
  - For recurring events: requires `?editScope=SINGLE|THIS_AND_FUTURE|ALL`.
    - `SINGLE`: updates only this `Event` instance; detaches from series (sets `event_series_id = null`).
    - `THIS_AND_FUTURE`: ends the current `EventSeries` at `eventDate - 1`; creates a new `EventSeries` from `eventDate` onward with updated settings.
    - `ALL`: updates the `EventSeries` template and all future instances with `status = SCHEDULED`.
  - Cannot update an event with `status = CANCELLED`.
- [ ] `GET /api/v1/events` — browse upcoming events at a location.
  - Query params: `locationId` (required), `from` (date), `to` (date), `page`, `size`.
  - Returns events with `status = SCHEDULED` and `eventDate >= today`.
  - Accessible to all authenticated roles.
- [ ] `GET /api/v1/events/{eventId}` — event detail including current RSVP count and capacity remaining.
- [ ] Unit tests: create standalone event with past date → 400; update CANCELLED event → 409; `THIS_AND_FUTURE` split creates new series at correct date; `ALL` edit updates all future instances.
- [ ] Integration test: create recurring weekly event (5 instances); edit `THIS_AND_FUTURE` from instance 3 → verify instances 1–2 unchanged, instances 3–5 belong to new series.

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80%
- [ ] All three edit scopes tested with recurring event fixture
- [ ] `oms.workplace.event.created` event published
- [ ] Audit log entries for `EVENT_CREATED`, `EVENT_UPDATED`
- [ ] Swagger annotations complete
- [ ] Jenkins pipeline green

---

## Dependencies

- **AOMS-37** (RSVP) depends on `Event` records created here.
- **AOMS-38** (recurring) depends on `EventSeries` created here.
