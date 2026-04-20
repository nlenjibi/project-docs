# AOMS-38 — Recurring Events with Edit Scope Control (Backend)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-38 |
| **Epic** | Workplace Service — AOMS-31 |
| **Type** | Story |
| **Priority** | 🟡 High |
| **Story Points** | 5 |
| **Phase** | Phase 2 |
| **Labels** | workplace, events, recurring, api |
| **Service** | `workplace-service` (Spring Boot / Java) |

---

## Story

As an event organizer, I want to manage recurring event series with precise edit scopes so that changing one instance doesn't accidentally affect an entire series.

---

## Acceptance Criteria

- [ ] On creation of a recurring event (`isRecurring = true` in AOMS-36), a `EventSeries` is created and all future `Event` instances are generated up to `recurrencePattern.endDate`. Maximum 52 instances generated per series creation to prevent runaway generation.
- [ ] Three edit scopes enforced on `PATCH /api/v1/events/{eventId}?editScope=`:
  - **`SINGLE`:** Mutates only the targeted `Event` instance. Detaches it from the series by setting `event_series_id = null`. All other instances remain unchanged.
  - **`THIS_AND_FUTURE`:** Ends the current `EventSeries` at `eventDate - 1 day` (sets `EventSeries.ends_at = eventDate - 1`). Creates a new `EventSeries` from `eventDate` onward with updated recurrence settings. Re-generates future `Event` instances under the new series.
  - **`ALL`:** Updates the `EventSeries` template fields (title, description, startTime, endTime, capacity). Propagates changes to all future `Event` instances with `status = SCHEDULED` and `eventDate >= today`. Past instances are not modified.
- [ ] `DELETE /api/v1/events/{eventId}?editScope=` — delete/cancel with same three scopes:
  - `SINGLE`: cancels one instance (sets `status = CANCELLED`).
  - `THIS_AND_FUTURE`: cancels all instances from this date onward; ends the series.
  - `ALL`: cancels all instances in the series including past scheduled ones.
  - Cancellation always publishes `oms.workplace.event.cancelled` for each affected instance with `attendeeIds[]`.
- [ ] `editScope` parameter required for recurring events. Returns 400 if omitted.
- [ ] Unit tests: `SINGLE` edit doesn't affect sibling instances; `THIS_AND_FUTURE` splits at correct date; `ALL` edit propagates to all future instances; `ALL` cancel marks all instances `CANCELLED`.
- [ ] Integration test: create weekly event for 4 weeks; edit `THIS_AND_FUTURE` from week 3 with new capacity → weeks 1–2 retain old capacity; weeks 3–4 have new capacity under new series ID.

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80%
- [ ] All three edit and cancel scopes tested end-to-end
- [ ] `THIS_AND_FUTURE` split creates correct `EventSeries.ends_at` boundary
- [ ] Cancellation `oms.workplace.event.cancelled` events published per affected instance
- [ ] Swagger annotations document `editScope` param with enum values and descriptions
- [ ] Jenkins pipeline green

---

## Dependencies

- **AOMS-36** — `EventSeries` and `Event` structure established there.
- **AOMS-39** — cancellation notification flow in that ticket.
