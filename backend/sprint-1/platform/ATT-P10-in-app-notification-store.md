# ATT-P10 — In-App Notification Store and Read APIs (notification-service)

| Field | Value |
|-------|-------|
| **Ticket** | ATT-P10 |
| **Epic** | Platform / Infrastructure — M1 |
| **Type** | Story |
| **Priority** | 🔴 Highest |
| **Story Points** | 3 |
| **Sprint** | Sprint 1 |
| **Labels** | platform, notification, in-app, fastapi |
| **Service** | `notification-service` (FastAPI / Python 3.12 · Port 8086) |

---

## Story

As the notification-service, I want to persist in-app notifications to a database and expose read/mark-read APIs so the React frontend can display a notification bell with unread count and a notification drawer.

---

## Background

ATT-P07 covers SES email dispatch — the SQS consumer that handles email sending. This ticket covers the complementary in-app notification concern: persisting `Notification` records to `notification_db` and exposing the read/mark-read endpoints the frontend consumes.

The same SQS consumer that dispatches email (ATT-P07) also writes in-app `Notification` records for events that require in-app delivery. Not all events produce both channels — the notification type determines which channels fire (see table below).

**Why a database for in-app notifications:** WebSocket push is a future concern (AD-020 deferred). At launch, the React frontend polls `GET /api/v1/notifications/my?unreadOnly=true` on page focus. This is sufficient at current scale and avoids WebSocket infrastructure complexity.

**Pipeline:** Jenkins CI.

---

## Acceptance Criteria

### Notification Data Model
- [ ] `notifications` table: `id` (UUID PK), `user_id` (UUID), `event_type` (varchar), `title` (varchar), `body` (text), `action_url` (varchar, nullable — deep link to the relevant entity), `is_read` (bool, default false), `created_at` (timestamptz), `read_at` (timestamptz, nullable).
- [ ] Index on `(user_id, is_read, created_at DESC)` — covers the primary read query.
- [ ] `notification_templates` table: `event_type` (PK), `in_app_title_template` (varchar), `in_app_body_template` (varchar), `email_subject_template` (varchar), `email_body_template` (text). Seeded via Alembic migration from a fixtures file.

### SQS Consumer — In-App Write Path
- [ ] For each SQS event listed in the channel table below, the consumer writes a `Notification` row for each relevant recipient before (or alongside) any SES email dispatch.
- [ ] If a `Notification` write fails (DB error), log `ERROR` and continue — do not block email dispatch.
- [ ] `action_url` populated per event type so the frontend can deep-link to the relevant entity (e.g. `/bookings/{bookingId}`, `/remote-requests/{requestId}`).

### Channel Table

| Event | In-App | Email |
|---|---|---|
| `seat.booking.confirmed` | Booking employee | ✗ (no email) |
| `remote.request.submitted` | Manager / Delegate | ✓ |
| `remote.request.approved` | Requesting employee | ✓ |
| `remote.request.rejected` | Requesting employee | ✓ |
| `ooo.request.approved` | Nominated delegate | ✓ |
| `seating.booking.released` | Facilities Admin (daily summary) | ✓ |
| `workplace.visit.checked_in` | Host employee | ✓ |
| `workplace.event.rsvp.confirmed` | RSVPing employee | ✓ |
| `workplace.event.rsvp.promoted` | Waitlisted employee | ✓ |
| `workplace.event.cancelled` | All RSVPed attendees | ✓ |
| `inventory.supply.request.approved` | Requesting employee | ✓ |
| `inventory.supply.request.fulfilled` | Requesting employee | ✓ |
| `inventory.asset.assigned` | Assigned employee | ✓ |
| `inventory.fault.reported` | Facilities Admin | ✗ (in-app only) |
| `user.created` | New employee | ✓ (welcome email only, no in-app) |

### Read APIs
- [ ] `GET /api/v1/notifications/my` — own notifications, newest first, paginated (size ≤ 50).
  - Query params: `unreadOnly` (bool, default false), `page`, `size`.
  - Response includes `unreadCount` (total unread — computed from DB, returned in envelope).
  - `user_id` always sourced from validated JWT — never from query param.
- [ ] `PATCH /api/v1/notifications/{id}/read` — marks one notification as read.
  - Returns 404 if notification doesn't exist or doesn't belong to the authenticated user.
  - Sets `is_read = true` and `read_at = now()`.
- [ ] `PATCH /api/v1/notifications/read-all` — marks all of the user's unread notifications as read in a single UPDATE.

### Tests
- [ ] Unit test: SQS event for `seat.booking.confirmed` → `Notification` row written; no SES call (in-app only event).
- [ ] Unit test: `GET /api/v1/notifications/my?unreadOnly=true` returns only unread for authenticated user; another user's notifications not returned.
- [ ] Unit test: `PATCH /api/v1/notifications/{id}/read` by wrong user → 404.
- [ ] Integration test: seed 5 unread notifications for user A and 3 for user B; GET as user A → 5 returned; mark-all-read → `unreadCount = 0`; GET again → all `is_read = true`.

---

## Environment Variables

```bash
NOTIFICATION_DB_URL=postgresql+asyncpg://...notification_db...
```

---

## Definition of Done

- [ ] Code reviewed and merged to `main`
- [ ] Unit tests passing; coverage ≥ 80%
- [ ] In-app write path tested for all 14 in-app event types
- [ ] `user_id` sourced from JWT only — no query param override (tested)
- [ ] `unreadCount` correctly computed and returned in response envelope
- [ ] `PATCH /read-all` uses single bulk UPDATE (not per-notification loop)
- [ ] `notification_templates` seeded via Alembic migration
- [ ] Swagger annotations complete
- [ ] Jenkins pipeline green
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **ATT-P07** (SES Email Integration) — shares the same SQS consumer. ATT-P07 adds email dispatch; this ticket adds the in-app write step to the same consumer handler.
- **ATT-P08** (Cloud Map) — `notification-service` resolves `auth-service.oms.local` to look up user email for email channel.
- Phase 2 events (`workplace.*`, `inventory.*`) flow through this same consumer once `workplace-service` and `inventory-service` are deployed. No code change needed — the consumer maps event types to templates from `notification_templates` table.
