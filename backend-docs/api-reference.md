# API Reference
## Office Management System (OMS)

---

## Global Conventions

| Convention | Value |
|-----------|-------|
| Base path | `/api/v1/` |
| Auth | Session cookie on every request; validated by API Gateway → `identity-service` |
| Response envelope | `{ "success": true/false, "data": {}, "error": null }` |
| Error envelope | `{ "success": false, "data": null, "error": { "code": "...", "message": "..." } }` |
| Content type | `application/json` |
| Pagination | `?page=0&size=20&sort=createdAt,desc` — all list endpoints |
| Versioning | `/api/v1/` from day one; breaking changes → `/api/v2/` |

**Roles:** `EMPLOYEE` · `MANAGER` · `HR` · `FACILITIES_ADMIN` · `SUPER_ADMIN`

**Location scoping:** All role checks are coupled with a location check. A Manager in Accra cannot access Lagos data. `SUPER_ADMIN` has a NULL `location_id` on their `UserRole` — the only cross-location exception.

---

## API Gateway Routing

```
/api/v1/auth/**          → identity-service:8081
/api/v1/users/**         → user-service:8082
/api/v1/locations/**     → user-service:8082
/api/v1/attendance/**    → attendance-service:8083
/api/v1/seat-bookings/** → seating-service:8084
/api/v1/seats/**         → seating-service:8084
/api/v1/remote-**/**     → remote-service:8085
/api/v1/ooo-**/**        → remote-service:8085
/api/v1/teams/**         → remote-service:8085
/api/v1/approval-**/**   → remote-service:8085
/api/v1/notifications/** → notification-service:8086
/api/v1/audit-logs/**    → audit-service:8087
/api/v1/visitors/**      → visitor-service:8088
/api/v1/visits/**        → visitor-service:8088
/api/v1/agreement-**/**  → visitor-service:8088
/api/v1/events/**        → event-service:8089
/api/v1/supplies/**      → supplies-service:8090
/api/v1/supply-**/**     → supplies-service:8090
/api/v1/assets/**        → assets-service:8091
/api/v1/asset-**/**      → assets-service:8091
/api/v1/fault-**/**      → assets-service:8091
/api/v1/maintenance-**/** → assets-service:8091
```

---

## identity-service — Auth

### POST /api/v1/auth/login
Initiates redirect to the SSO provider.

**Auth required:** No  
**Response:** `302 Found` → SSO provider URL

---

### GET /api/v1/auth/callback
Exchanges OIDC authorisation code for a token, verifies against JWKS, and issues an HTTP-only session cookie.

**Auth required:** No  
**Query params:** `code`, `state` (from SSO provider)  
**Success:** `200` + session cookie set  
**Errors:** `401 UNAUTHORIZED` — invalid or expired token

---

### POST /api/v1/auth/logout
Invalidates the server-side session and clears the session cookie.

**Auth required:** Yes (any role)  
**Response:** `200 { "success": true }`

---

### GET /api/v1/auth/validate
Internal endpoint called by the API Gateway on every inbound request.

**Auth required:** Session cookie  
**Response:** `200` with user context headers injected, or `401`

---

### GET /api/v1/auth/me
Returns the authenticated user's identity and roles from the session context.

**Auth required:** Yes (any role)  
**Response:**
```json
{
  "success": true,
  "data": {
    "userId": "uuid",
    "email": "user@org.com",
    "roles": [{ "role": "MANAGER", "locationId": "uuid" }]
  }
}
```

---

## user-service — Users & Locations

### GET /api/v1/users/{id}
**Auth:** Any role (own profile) or HR / Super Admin (any profile)  
**Response:** `UserResponse { id, email, displayName, locationId, active }`

### GET /api/v1/users/{id}/roles
**Auth:** HR, Super Admin  
**Response:** `List<UserRoleResponse> { roleId, role, locationId, locationName }`

### GET /api/v1/locations
**Auth:** Any role  
**Response:** Paginated list of `LocationResponse { id, name, timezone, active }`

### GET /api/v1/locations/{id}/config
**Auth:** Any role (own location) or Super Admin  
**Response:** `LocationConfigResponse { bookingWindowDays, noShowReleaseTime, attendanceCutoffTime, maxRemoteDaysPerWeek, ... }`

### PATCH /api/v1/locations/{id}/config
**Auth:** Super Admin  
**Body:** `LocationConfigUpdateRequest` (partial update)  
**Response:** `200` updated config

### POST /api/v1/users/{id}/roles
**Auth:** Super Admin  
**Body:** `{ "role": "MANAGER", "locationId": "uuid" }`  
**Response:** `201` created UserRole

### DELETE /api/v1/users/{id}/roles/{roleId}
**Auth:** Super Admin  
**Response:** `204`

---

## attendance-service — Attendance

### GET /api/v1/attendance/{userId}
**Auth:** Employee (own), Manager (team), HR / Super Admin (any)  
**Query:** `?from=2026-01-01&to=2026-01-31&page=0&size=30`  
**Response:** Paginated `AttendanceRecordResponse { date, status, workDurationMinutes, isLate, minutesLate }`

### GET /api/v1/attendance/team/{managerId}
**Auth:** Manager (own team), HR, Super Admin  
**Response:** Paginated team attendance records

### GET /api/v1/attendance/location/{locationId}
**Auth:** HR, Super Admin  
**Response:** Paginated location-wide attendance

### POST /api/v1/attendance/{id}/override
**Auth:** Super Admin  
**Body:** `{ "status": "PRESENT", "reason": "Badge reader fault" }`  
**Response:** `200` updated record

### GET /api/v1/attendance/reports/no-shows
**Auth:** HR, Facilities Admin, Super Admin  
**Query:** `?locationId=uuid&from=2026-01-01&to=2026-01-31`  
**Response:** Paginated no-show records

### GET /api/v1/attendance/export
**Auth:** HR, Super Admin  
**Query:** `?locationId=uuid&from=...&to=...&format=CSV`  
**Response:** File download (CSV or Excel)

### GET /api/v1/attendance/sync/logs
**Auth:** Super Admin, Facilities Admin  
**Response:** Paginated `BadgeSyncJobLogResponse { jobId, locationId, startedAt, status, recordsInserted, recordsSkipped }`

### POST /api/v1/attendance/sync/trigger
**Auth:** Super Admin  
**Body:** `{ "locationId": "uuid", "date": "2026-01-15" }`  
**Response:** `202 Accepted`

---

## seating-service — Seats & Bookings

### GET /api/v1/locations/{id}/floor-plan
**Auth:** Any role  
**Response:** `FloorPlanResponse` with floors, zones, seats, and real-time availability counts

### POST /api/v1/seat-bookings
**Auth:** Employee (own), Manager (team)  
**Body:** `SeatBookingRequest { seatId, bookingDate, locationId }`  
**Response:** `201 SeatBookingResponse { bookingId, seatId, date, status: CONFIRMED }`  
**Errors:** `409 SEAT_UNAVAILABLE`, `400 BOOKING_WINDOW_EXCEEDED`

### DELETE /api/v1/seat-bookings/{id}
**Auth:** Employee (own booking), Manager (team booking), Facilities Admin  
**Response:** `204`  
**Errors:** `400 CANCELLATION_CUTOFF_PASSED`

### GET /api/v1/seat-bookings/my
**Auth:** Employee  
**Query:** `?from=2026-01-01&to=2026-01-31`  
**Response:** Paginated own bookings

### POST /api/v1/seat-bookings/block
**Auth:** Manager  
**Body:** `BlockBookingRequest { seatIds[], dateRange, teamUserIds[], locationId }`  
**Response:** `201` created block reservation

### POST /api/v1/seats/{id}/permanent
**Auth:** HR, Facilities Admin  
**Body:** `PermanentSeatRequest { userId, locationId }`  
**Response:** `200`

### DELETE /api/v1/seats/{id}/permanent
**Auth:** HR, Facilities Admin  
**Response:** `204`

### PUT /api/v1/locations/{id}/floor-plan
**Auth:** Facilities Admin  
**Body:** Floor plan configuration (floors, zones, seat layout)  
**Response:** `200`

---

## remote-service — Remote & OOO

### POST /api/v1/remote-requests
**Auth:** Employee  
**Body:** `RemoteRequestRequest { dates[], locationId, reason? }`  
**Response:** `201` with policy check result (APPROVED / PENDING / WARNING)

### POST /api/v1/ooo-requests
**Auth:** Employee  
**Body:** `OOORequestRequest { startDate, endDate, locationId, reason }`  
**Response:** `201 OOORequestResponse`

### PATCH /api/v1/remote-requests/{id}/approve
**Auth:** Manager, Delegate  
**Response:** `200`

### PATCH /api/v1/remote-requests/{id}/reject
**Auth:** Manager, Delegate  
**Body:** `{ "reason": "Team coverage too low" }`  
**Response:** `200`

### GET /api/v1/teams/{managerId}/schedule
**Auth:** Manager (own team), HR, Super Admin  
**Query:** `?weekOf=2026-03-10`  
**Response:** `TeamScheduleResponse` — per-day location breakdown per employee

### POST /api/v1/approval-delegates
**Auth:** Manager  
**Body:** `{ "delegateUserId": "uuid", "startDate": "...", "endDate": "...", "locationId": "uuid" }`  
**Response:** `201`

### GET /api/v1/remote-day-policies/{locationId}
**Auth:** Any role  
**Response:** `RemoteDayPolicyResponse { maxDaysPerWeek, teamOverlapThreshold, policyType: HARD_BLOCK | SOFT_WARNING }`

### PATCH /api/v1/remote-day-policies/{managerId}
**Auth:** HR, Super Admin  
**Body:** Updated policy fields  
**Response:** `200`

### GET /api/v1/remote-requests/history
**Auth:** Employee (own), Manager (team), HR, Super Admin  
**Response:** Paginated request history with status

---

## notification-service — Notifications

### GET /api/v1/notifications/my
**Auth:** Any role (own notifications only)  
**Query:** `?unreadOnly=true&page=0&size=20`  
**Response:** Paginated `NotificationResponse { id, message, type, read, createdAt }`

### PATCH /api/v1/notifications/{id}/read
**Auth:** Notification owner  
**Response:** `200`

### PATCH /api/v1/notifications/read-all
**Auth:** Any role  
**Response:** `200`

---

## audit-service — Audit Logs

### GET /api/v1/audit-logs
**Auth:** HR, Facilities Admin, Super Admin  
**Query (all optional):**
- `actorId=uuid`
- `entityType=SeatBooking`
- `entityId=uuid`
- `locationId=uuid`
- `action=SEAT_BOOKED`
- `from=2026-01-01&to=2026-01-31`
- `page=0&size=50`

**Response:** Paginated `AuditLogResponse { id, actorId, actorRole, action, entityType, entityId, locationId, previousState, newState, occurredAt }`

> All records are scoped to the requesting user's location. Super Admin sees all locations.

---

## visitor-service — Visitors (Phase 2)

### POST /api/v1/visitors/pre-register
**Auth:** Employee  
**Body:** `VisitorPreRegisterRequest { visitorName, email, phone, expectedDate, locationId, purpose }`  
**Response:** `201`

### POST /api/v1/visitors/walk-in
**Auth:** Facilities Admin  
**Body:** Walk-in registration details  
**Response:** `201`

### PATCH /api/v1/visits/{id}/check-in
**Auth:** Facilities Admin  
**Body:** `{ "agreementTemplateVersionId": "uuid" }`  
**Response:** `200`

### PATCH /api/v1/visits/{id}/check-out
**Auth:** Facilities Admin  
**Response:** `200`

### GET /api/v1/visits/history
**Auth:** Employee (own)  
**Response:** Paginated own visitor history

### GET /api/v1/visits/location/{locationId}
**Auth:** HR, Facilities Admin  
**Response:** Paginated all visits at location

### POST /api/v1/agreement-templates
**Auth:** Facilities Admin  
**Body:** Template content + version label  
**Response:** `201`

### GET /api/v1/agreement-templates/active
**Auth:** Any role  
**Response:** Current active template

---

## event-service — Office Events (Phase 2)

### GET /api/v1/events
**Auth:** Any role  
**Query:** `?locationId=uuid&from=...&to=...`  
**Response:** Paginated upcoming events

### GET /api/v1/events/{id}
**Auth:** Any role  
**Response:** `EventResponse { id, title, date, capacity, rsvpCount, waitlistCount, organizerId }`

### POST /api/v1/events
**Auth:** Manager, HR, Facilities Admin  
**Body:** `CreateEventRequest { title, date, locationId, capacity, isRecurring, recurrencePattern? }`  
**Response:** `201`

### PATCH /api/v1/events/{id}
**Auth:** Event organiser, HR, Facilities Admin  
**Body:** `{ "editScope": "SINGLE | THIS_AND_FUTURE | ALL" }` + updated fields  
**Response:** `200`

### DELETE /api/v1/events/{id}
**Auth:** Event organiser, HR, Facilities Admin  
**Body:** `{ "editScope": "SINGLE | THIS_AND_FUTURE | ALL" }`  
**Response:** `200`

### POST /api/v1/events/{id}/rsvp
**Auth:** Employee  
**Response:** `200 { "status": "ATTENDING | WAITLISTED" }`

### DELETE /api/v1/events/{id}/rsvp
**Auth:** Employee  
**Response:** `204`

### GET /api/v1/events/{id}/attendees
**Auth:** Event organiser, HR, Facilities Admin  
**Response:** Attendee list with RSVP statuses

---

## supplies-service — Supplies (Phase 2)

### GET /api/v1/supplies/catalogue
**Auth:** Any role  
**Response:** Paginated supply catalogue items

### POST /api/v1/supply-requests
**Auth:** Employee  
**Body:** `SupplyRequestBody { items: [{ itemId, quantity }], locationId, justification }`  
**Response:** `201 { status: PENDING_MANAGER }`

### PATCH /api/v1/supply-requests/{id}/approve-stage1
**Auth:** Manager  
**Response:** `200 { status: PENDING_FACILITIES }`

### PATCH /api/v1/supply-requests/{id}/reject-stage1
**Auth:** Manager  
**Body:** `{ "reason": "..." }`  
**Response:** `200 { status: REJECTED }`

### PATCH /api/v1/supply-requests/{id}/fulfil
**Auth:** Facilities Admin  
**Response:** `200 { status: FULFILLED }`

### GET /api/v1/supplies/inventory-report
**Auth:** HR, Facilities Admin, Super Admin  
**Response:** Current stock levels per item per location

---

## assets-service — Assets (Phase 2)

### GET /api/v1/assets
**Auth:** HR, Facilities Admin  
**Response:** Paginated asset register

### POST /api/v1/assets
**Auth:** Facilities Admin  
**Body:** `CreateAssetRequest { categoryId, serialNumber, description, locationId, purchaseDate, value }`  
**Response:** `201`

### GET /api/v1/assets/my
**Auth:** Employee  
**Response:** Own assigned assets

### POST /api/v1/fault-reports
**Auth:** Employee  
**Body:** `FaultReportRequest { assetId, description, severity, locationId }`  
**Response:** `201`

### POST /api/v1/maintenance-records
**Auth:** Facilities Admin  
**Body:** `MaintenanceRecordRequest { assetId, type, description, date, locationId }`  
**Response:** `201`

### GET /api/v1/assets/{id}/audit-log
**Auth:** HR, Facilities Admin  
**Response:** Full chronological audit trail for the asset

---

## Standard Error Codes

| Code | HTTP Status | Description |
|------|------------|-------------|
| `UNAUTHORIZED` | 401 | Session missing, invalid, or expired |
| `FORBIDDEN` | 403 | Insufficient role or location access |
| `NOT_FOUND` | 404 | Resource does not exist or is out of scope |
| `CONFLICT` | 409 | Business conflict (seat unavailable, duplicate request) |
| `VALIDATION_ERROR` | 400 | DTO validation failed |
| `BOOKING_WINDOW_EXCEEDED` | 400 | Booking date outside the configured window |
| `CANCELLATION_CUTOFF_PASSED` | 400 | Too late to cancel |
| `POLICY_HARD_BLOCK` | 400 | Remote day request blocked by HARD_BLOCK policy |
| `INTERNAL_ERROR` | 500 | Unexpected server error (no stack trace exposed) |
