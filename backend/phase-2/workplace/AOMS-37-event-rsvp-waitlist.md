# AOMS-37 — Event RSVP and Waitlist Management (Backend)

| Field | Value |
|-------|-------|
| **Ticket** | AOMS-37 |
| **Epic** | Workplace Service — AOMS-31 |
| **Type** | Story |
| **Priority** | 🟡 High |
| **Story Points** | 3 |
| **Phase** | Phase 2 |
| **Labels** | workplace, events, rsvp, api |
| **Service** | `workplace-service` (Spring Boot / Java) |

---

## Story

As an employee, I want to RSVP to office events and be placed on a waitlist when the event is full, so I get a spot automatically if someone cancels.

---

## Acceptance Criteria

- [ ] `POST /api/v1/events/{eventId}/rsvp` — employee RSVPs to an event.
  - Accessible to all authenticated roles.
  - If `event.attending_count < event.capacity`: creates `EventInvite` with `status = ATTENDING`; increments attending count. Publishes `oms.workplace.event.rsvp.confirmed`.
  - If `event.attending_count >= event.capacity`: creates `EventInvite` with `status = WAITLISTED`; assigns `waitlist_position`. Publishes `oms.workplace.event.rsvp.waitlisted`.
  - Returns 409 if employee already has an active RSVP for this event.
  - Returns 400 if event is `CANCELLED` or `eventDate` is in the past.
- [ ] `DELETE /api/v1/events/{eventId}/rsvp` — employee cancels their RSVP.
  - Accessible to the RSVP owner only.
  - If cancelling an `ATTENDING` invite AND waitlist is non-empty:
    - Atomically promotes the next `WAITLISTED` invite to `ATTENDING` (lowest `waitlist_position`).
    - Decrements `waitlist_position` of all remaining waitlisted invites.
    - Publishes `oms.workplace.event.rsvp.promoted` for the promoted user.
  - Sets own `EventInvite.status = CANCELLED`.
- [ ] `GET /api/v1/events/{eventId}/attendees` — returns attendee list.
  - Accessible to event organizer, `HR`, `FACILITIES_ADMIN`.
  - Returns separate lists: `attending []` and `waitlist []` (ordered by `waitlist_position`).
- [ ] **Atomicity constraint:** Waitlist promotion must be a single DB transaction. If two cancellations happen concurrently, only one waitlisted user is promoted per freed slot — use optimistic locking on `Event.attending_count`.
- [ ] Unit tests: RSVP to full event → `WAITLISTED`; cancel `ATTENDING` with waitlist → first waitlisted promoted; cancel `WAITLISTED` invite → attending count unchanged, waitlist positions decremented; duplicate RSVP → 409.
- [ ] Integration test: fill event to capacity; RSVP fifth user → `WAITLISTED` position 1; cancel second attendee → fifth user becomes `ATTENDING`; verify `oms.workplace.event.rsvp.promoted` published.

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80%
- [ ] Atomic waitlist promotion verified — concurrent cancellations tested
- [ ] All three event RabbitMQ events (`confirmed`, `waitlisted`, `promoted`) published and verified
- [ ] Swagger annotations complete
- [ ] Jenkins pipeline green

---

## Dependencies

- **AOMS-36** — `Event` records must exist.
- **ATT-P10** (In-App Notifications) — employees notified on RSVP confirmation, waitlist, and promotion.
