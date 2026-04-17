# API Reference
## Office Management System (OMS)

**Version:** 3.0 — 8 services

---

## Global Conventions

| Convention | Value |
|-----------|-------|
| Base path | `/api/v1/` — all APIs versioned from day one |
| Auth | HTTP-only session cookie on every request; validated by Spring Cloud Gateway → `auth-service /internal/validate` |
| Response envelope | `{ "success": true, "data": {} }` |
| Error envelope | `{ "success": false, "data": null, "error": { "code": "...", "message": "...", "field": "..." } }` |
| Content type | `application/json` |
| Pagination | `?page=0&size=20&sort=createdAt,desc` — all list endpoints return paginated results |
| Versioning strategy | `/api/v1/` from day one; breaking changes → new `/api/v2/` path; old version supported for one release cycle |
| Internal header | `X-Correlation-ID` injected by gateway on every request; propagated by all services and Feign clients |

### Roles

| Role | Scope | Description |
|------|-------|-------------|
| `EMPLOYEE` | Location-scoped | Standard access — own data only |
| `MANAGER` | Location-scoped | Team visibility and approval authority |
| `HR` | Location-scoped | Org-wide visibility at assigned locations |
| `FACILITIES_ADMIN` | Location-scoped | Operational controls (floor plan, stock, assets) |
| `SUPER_ADMIN` | Cross-location (NULL location_id) | Governance — all locations, all data |

**Location scoping:** Role checks and location checks are inseparable. A Manager in Accra cannot access Lagos data. `SUPER_ADMIN` holds a NULL `location_id` on their `UserRole` record — the only cross-location exception, handled explicitly in the `LocationRoleInterceptor`.

---

## API Gateway Routing

Spring Cloud Gateway routes all requests via Eureka service discovery (`lb://` prefix).

```yaml
# Gateway route table
/api/v1/auth/**              → lb://auth-service
/api/v1/users/**             → lb://auth-service
/api/v1/locations/**         → lb://auth-service
/api/v1/attendance/**        → lb://attendance-service
/api/v1/seat-bookings/**     → lb://seating-service
/api/v1/seats/**             → lb://seating-service
/api/v1/remote-requests/**   → lb://remote-service
/api/v1/ooo-requests/**      → lb://remote-service
/api/v1/teams/**             → lb://remote-service
/api/v1/approval-delegates/**→ lb://remote-service
/api/v1/remote-day-policies/**→ lb://remote-service
/api/v1/notifications/**     → lb://notification-service
/api/v1/audit-logs/**        → lb://audit-service
/api/v1/supplies/**          → lb://inventory-service
/api/v1/supply-requests/**   → lb://inventory-service
/api/v1/assets/**            → lb://inventory-service
/api/v1/asset-assignments/** → lb://inventory-service
/api/v1/asset-requests/**    → lb://inventory-service
/api/v1/fault-reports/**     → lb://inventory-service
/api/v1/maintenance-records/**→ lb://inventory-service
/api/v1/visitors/**          → lb://workplace-service
/api/v1/visits/**            → lb://workplace-service
/api/v1/agreement-templates/**→ lb://workplace-service
/api/v1/events/**            → lb://workplace-service
```

**Internal path (NOT routed to frontend — gateway uses it internally):**
```
/internal/validate           → auth-service  (called by gateway's SessionAuthFilter, not exposed externally)
```

---

## auth-service — Authentication, Users & Locations

### POST /api/v1/auth/login
Initiates the SSO redirect. The React SPA calls this when no session exists.

**Auth required:** No
**Response:** `302 Found` → SSO provider authorisation URL
**Design note:** The redirect URL contains a `state` parameter (CSRF token) that is verified in the callback. The React SPA never handles or stores the SSO token — all token handling is server-side.

---

### GET /api/v1/auth/callback
Exchanges OIDC authorisation code for tokens, verifies against JWKS endpoint, resolves OMS user and roles, and issues an HTTP-only session cookie.

**Auth required:** No
**Query params:** `code` (OIDC auth code), `state` (CSRF token from login redirect)
**Success:** `200` + `Set-Cookie: SESSION=...; HttpOnly; SameSite=Strict; Secure`
**Errors:**
- `401 UNAUTHORIZED` — OIDC token invalid, expired, wrong issuer, or wrong audience
- `403 FORBIDDEN` — SSO user not provisioned in OMS (no matching User record)

```json
// Success response body
{
  "success": true,
  "data": {
    "userId": "550e8400-e29b-41d4-a716-446655440000",
    "email": "alice@org.com",
    "displayName": "Alice Mensah",
    "roles": [
      { "role": "MANAGER", "locationId": "accra-location-uuid" }
    ]
  }
}
```

---

### POST /api/v1/auth/logout
Invalidates the server-side session and clears the session cookie.

**Auth required:** Yes (any role)
**Response:** `200 { "success": true, "data": null }`

---

### GET /api/v1/auth/me
Returns the authenticated user's identity and all roles from the session context.

**Auth required:** Yes (any role)
**Response:**
```json
{
  "success": true,
  "data": {
    "userId": "uuid",
    "email": "user@org.com",
    "displayName": "Alice Mensah",
    "roles": [
      { "role": "MANAGER", "locationId": "uuid", "locationName": "Accra HQ" }
    ]
  }
}
```

---

### GET /api/v1/users/{id}
**Auth:** EMPLOYEE (own profile only) · HR, SUPER_ADMIN (any)
**Response:** `UserResponse { id, email, displayName, personnelId, locationId, isActive, employmentStartDate }`
**Errors:** `404 NOT_FOUND`, `403 FORBIDDEN`

---

### GET /api/v1/users/{id}/roles
**Auth:** HR, SUPER_ADMIN
**Response:**
```json
{
  "success": true,
  "data": [
    { "roleId": "uuid", "role": "MANAGER", "locationId": "uuid", "locationName": "Accra HQ" }
  ]
}
```

---

### GET /api/v1/users/{id}/direct-reports
**Auth:** MANAGER (own team), HR, SUPER_ADMIN — also called internally by attendance-service via Feign
**Query:** `?locationId=uuid`
**Response:** `List<UserSummaryResponse> { userId, displayName, email }`

---

### GET /api/v1/locations
**Auth:** Any role
**Response:** Paginated `LocationResponse { id, name, country, timezone, isActive }`

---

### GET /api/v1/locations/{id}/config
**Auth:** Any role (own location) or SUPER_ADMIN
**Response:**
```json
{
  "success": true,
  "data": {
    "locationId": "uuid",
    "bookingWindowDays": 14,
    "cancellationCutoffHours": 2,
    "noShowReleaseTime": "10:00",
    "attendanceCutoffTime": "09:30",
    "maxRemoteDaysPerWeek": 2,
    "crossMidnightGapHours": 6
  }
}
```

---

### PATCH /api/v1/locations/{id}/config
**Auth:** SUPER_ADMIN
**Body:** Partial update — any subset of `LocationConfigUpdateRequest` fields
**Response:** `200` updated config object
**Note:** Changes take effect on the next job execution cycle for attendance and seating. The service publishes `oms.location.config.updated` so dependent services can refresh cached config.

---

### POST /api/v1/users/{id}/roles
**Auth:** SUPER_ADMIN
**Body:**
```json
{ "role": "MANAGER", "locationId": "uuid" }
```
**Response:** `201 UserRoleResponse`
**Error:** `409 CONFLICT` if user already holds this role at this location

---

### DELETE /api/v1/users/{id}/roles/{roleId}
**Auth:** SUPER_ADMIN
**Response:** `204`

---

## attendance-service — Attendance

### GET /api/v1/attendance/{userId}
**Auth:** EMPLOYEE (own only) · MANAGER (own team) · HR, SUPER_ADMIN (any)
**Query:** `?from=2026-01-01&to=2026-01-31&page=0&size=30&locationId=uuid`
**Response:** Paginated `AttendanceRecordResponse`:
```json
{
  "success": true,
  "data": {
    "content": [
      {
        "id": "uuid",
        "recordDate": "2026-01-15",
        "status": "PRESENT",
        "workDurationMinutes": 510,
        "isLate": false,
        "minutesLate": 0,
        "workSessionId": "uuid",
        "overrideBy": null
      }
    ],
    "page": 0,
    "size": 30,
    "totalElements": 20
  }
}
```

---

### GET /api/v1/attendance/team/{managerId}
**Auth:** MANAGER (own team), HR, SUPER_ADMIN
**Query:** `?from=2026-01-01&to=2026-01-31&locationId=uuid`
**Response:** Paginated team attendance records enriched with `displayName`, `email` (from local `user_summaries`)
**Implementation note:** Calls auth-service `GET /api/v1/users/{managerId}/direct-reports` via Feign (once) to get the team member IDs, then queries own DB. No N+1 calls.

---

### GET /api/v1/attendance/location/{locationId}
**Auth:** HR, SUPER_ADMIN
**Query:** `?from&to&page&size`
**Response:** Paginated location-wide attendance

---

### POST /api/v1/attendance/{id}/override
**Auth:** SUPER_ADMIN
**Body:**
```json
{ "status": "PRESENT", "reason": "Badge reader fault on 2026-01-15" }
```
**Response:** `200` updated record
**Audit:** Publishes `oms.audit.event` with `previousState` and `newState`.

---

### GET /api/v1/attendance/reports/no-shows
**Auth:** HR, FACILITIES_ADMIN, SUPER_ADMIN
**Query:** `?locationId=uuid&from=2026-01-01&to=2026-01-31`
**Response:** Paginated no-show records with user display names

---

### GET /api/v1/attendance/export
**Auth:** HR, SUPER_ADMIN
**Query:** `?locationId=uuid&from=...&to=...&format=CSV` (CSV or EXCEL)
**Response:** File download (`text/csv` or `application/vnd.ms-excel`)
**Trade-off:** Export is synchronous for date ranges ≤ 90 days. For longer ranges, returns `202 Accepted` and the file is delivered via a notification when ready. This prevents request timeouts on large exports.

---

### GET /api/v1/attendance/sync/logs
**Auth:** SUPER_ADMIN, FACILITIES_ADMIN
**Response:** Paginated `BadgeSyncJobLogResponse { jobId, locationId, syncDate, startedAt, status, recordsInserted, recordsSkipped, recordsFailed, errorMessage }`

---

### POST /api/v1/attendance/sync/trigger
**Auth:** SUPER_ADMIN
**Body:** `{ "locationId": "uuid", "date": "2026-01-15" }`
**Response:** `202 Accepted { "jobId": "uuid" }` — job runs asynchronously
**Use case:** Re-run sync for a specific date (e.g., after Athena data correction).

---

## seating-service — Seats & Bookings

### GET /api/v1/locations/{id}/floor-plan
**Auth:** Any role
**Query:** `?date=2026-01-15` (defaults to today)
**Response:**
```json
{
  "success": true,
  "data": {
    "locationId": "uuid",
    "date": "2026-01-15",
    "floors": [
      {
        "floorId": "uuid",
        "floorNumber": 2,
        "availableSeats": 12,
        "totalBookableSeats": 30,
        "zones": [
          {
            "zoneId": "uuid",
            "zoneName": "East Wing",
            "seats": [
              {
                "seatId": "uuid",
                "seatLabel": "E-14",
                "seatType": "HOT_DESK",
                "isBookable": true,
                "bookingId": null,
                "occupantName": null
              }
            ]
          }
        ]
      }
    ]
  }
}
```

---

### POST /api/v1/seat-bookings
**Auth:** EMPLOYEE (own), MANAGER (team members)
**Body:**
```json
{ "seatId": "uuid", "bookingDate": "2026-01-20", "locationId": "uuid" }
```
**Response:** `201 SeatBookingResponse { bookingId, seatId, seatLabel, bookingDate, status: "CONFIRMED" }`
**Errors:**
- `409 SEAT_UNAVAILABLE` — seat already booked for that date
- `400 BOOKING_WINDOW_EXCEEDED` — date is beyond `bookingWindowDays` in LocationConfig
- `400 VALIDATION_ERROR` — missing or invalid fields

---

### DELETE /api/v1/seat-bookings/{id}
**Auth:** EMPLOYEE (own booking) · MANAGER (team booking) · FACILITIES_ADMIN (any)
**Response:** `204`
**Errors:** `400 CANCELLATION_CUTOFF_PASSED` — too late to cancel (within `cancellationCutoffHours`)

---

### GET /api/v1/seat-bookings/my
**Auth:** EMPLOYEE
**Query:** `?from=2026-01-01&to=2026-01-31`
**Response:** Paginated own bookings with seat labels and statuses

---

### POST /api/v1/seat-bookings/block
**Auth:** MANAGER
**Body:**
```json
{
  "seatIds": ["uuid1", "uuid2"],
  "dateRange": { "from": "2026-01-20", "to": "2026-01-24" },
  "teamUserIds": ["user-uuid1", "user-uuid2"],
  "locationId": "uuid"
}
```
**Response:** `201 BlockReservationResponse { reservationId, status, bookedDates[] }`
**Note:** Block booking reserves seats for the specified team members across a date range. Each seat-date pair is validated individually. Partial success not permitted — all or nothing.

---

### POST /api/v1/seats/{id}/permanent
**Auth:** HR, FACILITIES_ADMIN
**Body:** `{ "userId": "uuid", "locationId": "uuid" }`
**Response:** `200 SeatResponse`
**Note:** Marks seat as `PERMANENT` type; sets `permanent_user_id`. Hot-desk bookings on this seat by other users are now rejected.

---

### DELETE /api/v1/seats/{id}/permanent
**Auth:** HR, FACILITIES_ADMIN
**Response:** `204` — seat reverts to `HOT_DESK` type

---

### PUT /api/v1/locations/{id}/floor-plan
**Auth:** FACILITIES_ADMIN
**Body:** Full floor plan configuration (floors, zones, seat layout)
**Response:** `200`
**Trade-off:** This is a full replacement (`PUT`), not a partial update. The frontend sends the complete current plan with modifications. This avoids complex partial-update merge logic for spatial data but requires the client to always send the full state.

---

## remote-service — Remote & OOO

### POST /api/v1/remote-requests
**Auth:** EMPLOYEE
**Body:**
```json
{ "dates": ["2026-01-20", "2026-01-21"], "locationId": "uuid", "reason": "Team-agreed WFH" }
```
**Response:**
```json
{
  "success": true,
  "data": {
    "requestId": "uuid",
    "status": "APPROVED",
    "policyResult": {
      "blocked": false,
      "warning": false,
      "reason": null
    }
  }
}
```
**Note on status:** Auto-approved if policy allows; `PENDING` if manager approval required by policy config; `BLOCKED` (400) if HARD_BLOCK threshold exceeded.

---

### POST /api/v1/ooo-requests
**Auth:** EMPLOYEE
**Body:** `{ "startDate": "2026-02-10", "endDate": "2026-02-14", "locationId": "uuid", "reason": "Medical leave" }`
**Response:** `201 OOORequestResponse { requestId, startDate, endDate, status: "PENDING" }`

---

### PATCH /api/v1/remote-requests/{id}/approve
**Auth:** MANAGER, active ApprovalDelegate, HR fallback approver
**Response:** `200 { "status": "APPROVED" }`
**Side effect:** Publishes `oms.remote.request.approved` → attendance-service overlays REMOTE status.

---

### PATCH /api/v1/remote-requests/{id}/reject
**Auth:** MANAGER, active ApprovalDelegate
**Body:** `{ "reason": "Team coverage insufficient for this date" }`
**Response:** `200 { "status": "REJECTED" }`

---

### GET /api/v1/teams/{managerId}/schedule
**Auth:** MANAGER (own team), HR, SUPER_ADMIN
**Query:** `?weekOf=2026-01-20`
**Response:**
```json
{
  "success": true,
  "data": {
    "weekOf": "2026-01-20",
    "days": [
      {
        "date": "2026-01-20",
        "inOffice": ["Alice Mensah", "Bob Asante"],
        "remote": ["Claire Osei"],
        "ooo": [],
        "pendingApproval": ["David Larbi"]
      }
    ]
  }
}
```

---

### POST /api/v1/approval-delegates
**Auth:** MANAGER
**Body:** `{ "delegateUserId": "uuid", "startDate": "2026-02-10", "endDate": "2026-02-14", "locationId": "uuid" }`
**Response:** `201 ApprovalDelegateResponse`
**Note:** Only one active delegate per manager per location at a time. Creating a new one deactivates the previous.

---

### GET /api/v1/remote-day-policies/{locationId}
**Auth:** Any role
**Response:** `RemoteDayPolicyResponse { maxDaysPerWeek, teamOverlapThresholdPct, policyType: "HARD_BLOCK | SOFT_WARNING" }`

---

### PATCH /api/v1/remote-day-policies/{managerId}
**Auth:** HR, SUPER_ADMIN
**Body:** Updated policy fields
**Response:** `200` updated policy

---

### GET /api/v1/remote-requests/history
**Auth:** EMPLOYEE (own), MANAGER (team), HR, SUPER_ADMIN
**Query:** `?locationId=uuid&from&to&status=APPROVED|PENDING|REJECTED`
**Response:** Paginated request history with status and approval chain

---

## notification-service — Notifications

### GET /api/v1/notifications/my
**Auth:** Any role (own notifications only)
**Query:** `?unreadOnly=true&page=0&size=20`
**Response:**
```json
{
  "success": true,
  "data": {
    "content": [
      {
        "id": "uuid",
        "message": "Your remote-day request for 2026-01-20 has been approved.",
        "type": "REMOTE_APPROVED",
        "read": false,
        "createdAt": "2026-01-18T14:32:00Z"
      }
    ],
    "unreadCount": 3
  }
}
```

---

### PATCH /api/v1/notifications/{id}/read
**Auth:** Notification owner only
**Response:** `200 { "success": true }`

---

### PATCH /api/v1/notifications/read-all
**Auth:** Any role
**Response:** `200 { "success": true, "data": { "markedRead": 5 } }`

---

## audit-service — Audit Logs

### GET /api/v1/audit-logs
**Auth:** HR, FACILITIES_ADMIN, SUPER_ADMIN

**Query parameters (all optional):**
| Param | Type | Example |
|-------|------|---------|
| `actorId` | UUID | Filter by user who performed the action |
| `entityType` | String | `SeatBooking`, `AttendanceRecord`, `SupplyRequest` |
| `entityId` | UUID | Specific entity instance |
| `locationId` | UUID | Location scope (SUPER_ADMIN can omit for all) |
| `action` | String | `SEAT_BOOKED`, `ATTENDANCE_OVERRIDDEN`, `ASSET_ASSIGNED` |
| `from` | ISO date | Start of date range |
| `to` | ISO date | End of date range |
| `page` | int | Default 0 |
| `size` | int | Default 50, max 100 |

**Response:**
```json
{
  "success": true,
  "data": {
    "content": [
      {
        "id": "uuid",
        "actorId": "uuid",
        "actorName": "Alice Mensah",
        "actorRole": "MANAGER",
        "action": "SEAT_BOOKED",
        "entityType": "SeatBooking",
        "entityId": "uuid",
        "locationId": "uuid",
        "previousState": null,
        "newState": { "seatId": "...", "date": "2026-01-20", "status": "CONFIRMED" },
        "occurredAt": "2026-01-18T14:30:00Z",
        "correlationId": "uuid"
      }
    ]
  }
}
```

> All records are scoped to the requesting user's `location_id`. SUPER_ADMIN sees all locations (omit `locationId` parameter or the service accepts NULL).

---

## inventory-service — Supplies & Assets (Phase 2)

### GET /api/v1/supplies/catalogue
**Auth:** Any role
**Query:** `?locationId=uuid&page&size`
**Response:** Paginated `SupplyItemResponse { itemId, name, category, unit, currentStock, reorderThreshold }`

---

### POST /api/v1/supply-requests
**Auth:** EMPLOYEE
**Body:**
```json
{
  "items": [{ "itemId": "uuid", "quantity": 5 }],
  "locationId": "uuid",
  "justification": "Team stationery replenishment"
}
```
**Response:** `201 { "requestId": "uuid", "status": "PENDING_MANAGER" }`

---

### PATCH /api/v1/supply-requests/{id}/approve-stage1
**Auth:** MANAGER
**Response:** `200 { "status": "PENDING_FACILITIES" }`

---

### PATCH /api/v1/supply-requests/{id}/reject-stage1
**Auth:** MANAGER
**Body:** `{ "reason": "Budget limit reached for this quarter" }`
**Response:** `200 { "status": "REJECTED" }`

---

### PATCH /api/v1/supply-requests/{id}/fulfil
**Auth:** FACILITIES_ADMIN
**Response:** `200 { "status": "FULFILLED" }`
**Side effect:** Decrements stock from oldest non-expired `SupplyStockEntry` (FIFO). If resulting stock < `reorderThreshold` → publishes `oms.inventory.stock.low`.

---

### GET /api/v1/supplies/inventory-report
**Auth:** HR, FACILITIES_ADMIN, SUPER_ADMIN
**Query:** `?locationId=uuid`
**Response:** Stock levels per item per location with reorder status flags

---

### GET /api/v1/assets
**Auth:** HR, FACILITIES_ADMIN
**Query:** `?locationId=uuid&categoryId=uuid&page&size`
**Response:** Paginated `AssetResponse { assetId, serialNumber, description, category, status, assignedTo, locationId, purchaseDate, value }`

---

### POST /api/v1/assets
**Auth:** FACILITIES_ADMIN
**Body:**
```json
{
  "categoryId": "uuid",
  "serialNumber": "SN-20260001",
  "description": "Dell Latitude 7420",
  "locationId": "uuid",
  "purchaseDate": "2026-01-01",
  "value": 1200.00
}
```
**Response:** `201 AssetResponse`

---

### POST /api/v1/asset-assignments
**Auth:** FACILITIES_ADMIN
**Body:** `{ "assetId": "uuid", "userId": "uuid", "locationId": "uuid", "notes": "Issued for remote work" }`
**Response:** `201 AssetAssignmentResponse { assignmentId, assetId, userId, status: "ASSIGNED" }`

---

### PATCH /api/v1/asset-assignments/{id}/acknowledge
**Auth:** Assigned EMPLOYEE
**Response:** `200 { "status": "ACKNOWLEDGED" }`
**Note:** Employee confirmation that they have received the asset. Required before the assignment is considered complete.

---

### POST /api/v1/fault-reports
**Auth:** EMPLOYEE
**Body:**
```json
{
  "assetId": "uuid",
  "description": "Screen flickering intermittently",
  "severity": "MEDIUM",
  "locationId": "uuid"
}
```
**Response:** `201 FaultReportResponse`
**Severity values:** `LOW` · `MEDIUM` · `HIGH` · `CRITICAL`

---

### POST /api/v1/maintenance-records
**Auth:** FACILITIES_ADMIN
**Body:** `{ "assetId": "uuid", "type": "REPAIR", "description": "Screen replaced", "date": "2026-01-20", "locationId": "uuid" }`
**Response:** `201 MaintenanceRecordResponse`
**Type values:** `REPAIR` · `SCHEDULED_SERVICE` · `INSPECTION` · `UPGRADE`

---

### GET /api/v1/assets/{id}/audit-log
**Auth:** HR, FACILITIES_ADMIN
**Response:** Full chronological timeline: registrations, assignments, acknowledgements, returns, fault reports, maintenance, retirement

---

## workplace-service — Visitors & Events (Phase 2)

### POST /api/v1/visitors/pre-register
**Auth:** EMPLOYEE
**Body:**
```json
{
  "visitorName": "John Smith",
  "email": "john.smith@external.com",
  "phone": "+44 7700 123456",
  "expectedDate": "2026-01-22",
  "locationId": "uuid",
  "purpose": "Partnership meeting"
}
```
**Response:** `201 { "visitId": "uuid", "visitorCode": "VIS-20260122-001" }`
**Note:** Calls auth-service via Feign to validate that the host employee (the caller) is active at the specified location. Returns `400 INVALID_HOST` if not.

---

### POST /api/v1/visitors/walk-in
**Auth:** FACILITIES_ADMIN
**Body:** Walk-in registration details (same fields as pre-register)
**Response:** `201` — immediately proceeds to check-in flow

---

### PATCH /api/v1/visits/{id}/check-in
**Auth:** FACILITIES_ADMIN
**Body:** `{ "agreementTemplateVersionId": "uuid" }`
**Response:** `200 { "checkedInAt": "2026-01-22T09:15:00Z" }`
**Agreement version check:** If the visitor's last-signed version differs from the current active template, returns `409 AGREEMENT_SIGNATURE_REQUIRED` with the new template content for the visitor to sign before check-in can proceed.

---

### PATCH /api/v1/visits/{id}/check-out
**Auth:** FACILITIES_ADMIN
**Response:** `200 { "checkedOutAt": "2026-01-22T17:30:00Z", "durationMinutes": 495 }`

---

### GET /api/v1/visits/location/{locationId}
**Auth:** HR, FACILITIES_ADMIN
**Query:** `?from&to&page&size`
**Response:** Paginated visit records with visitor details and check-in/out times

---

### GET /api/v1/events
**Auth:** Any role
**Query:** `?locationId=uuid&from=2026-01-01&to=2026-01-31&page&size`
**Response:** Paginated `EventResponse { eventId, title, date, startTime, capacity, rsvpCount, waitlistCount, organizerName, isRecurring }`

---

### POST /api/v1/events
**Auth:** MANAGER, HR, FACILITIES_ADMIN
**Body:**
```json
{
  "title": "Q1 All-Hands",
  "date": "2026-01-30",
  "startTime": "14:00",
  "endTime": "16:00",
  "locationId": "uuid",
  "capacity": 100,
  "isRecurring": false,
  "description": "Quarterly company update"
}
```
**Response:** `201 EventResponse`

---

### POST /api/v1/events/{id}/rsvp
**Auth:** EMPLOYEE
**Response:**
```json
{
  "success": true,
  "data": {
    "inviteId": "uuid",
    "status": "ATTENDING",
    "position": null
  }
}
```
If capacity is full: `status: "WAITLISTED"`, `position: 3` (place in waitlist queue).

---

### DELETE /api/v1/events/{id}/rsvp
**Auth:** EMPLOYEE (own RSVP)
**Response:** `204`
**Side effect:** If event had a waitlist, automatically promotes the next waitlisted attendee and publishes `oms.workplace.event.rsvp.promoted`.

---

### GET /api/v1/events/{id}/attendees
**Auth:** Event organiser, HR, FACILITIES_ADMIN
**Response:** List of attendees (ATTENDING + WAITLISTED) with RSVP timestamps and statuses

---

## Standard Error Codes

| Code | HTTP Status | When It Occurs |
|------|------------|----------------|
| `UNAUTHORIZED` | 401 | Session missing, invalid, or expired |
| `FORBIDDEN` | 403 | Insufficient role or wrong location |
| `NOT_FOUND` | 404 | Resource does not exist or is outside the caller's location scope |
| `CONFLICT` | 409 | Business conflict — seat unavailable, duplicate request, agreement signature required |
| `VALIDATION_ERROR` | 400 | DTO field validation failed; `error.field` indicates the failing field |
| `BOOKING_WINDOW_EXCEEDED` | 400 | Booking date is beyond the configured `bookingWindowDays` |
| `CANCELLATION_CUTOFF_PASSED` | 400 | Cancellation attempted within `cancellationCutoffHours` |
| `POLICY_HARD_BLOCK` | 400 | Remote day request blocked by HARD_BLOCK policy threshold |
| `AGREEMENT_SIGNATURE_REQUIRED` | 409 | Visitor must sign the updated agreement template before check-in |
| `SEAT_UNAVAILABLE` | 409 | Seat already booked for that date (concurrent booking) |
| `INVALID_HOST` | 400 | Pre-registration host employee not active at specified location |
| `SERVICE_UNAVAILABLE` | 503 | Circuit breaker is OPEN on a required downstream service |
| `INTERNAL_ERROR` | 500 | Unexpected server error — no stack trace or internal detail in response |

---

## Request/Response Conventions

### Successful response envelope
```json
{
  "success": true,
  "data": { ... }
}
```

### Error response envelope
```json
{
  "success": false,
  "data": null,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "bookingDate must be a future date",
    "field": "bookingDate"
  }
}
```

### Paginated response
```json
{
  "success": true,
  "data": {
    "content": [ ... ],
    "page": 0,
    "size": 20,
    "totalElements": 87,
    "totalPages": 5,
    "last": false
  }
}
```

### Accepted (async) response
```json
{
  "success": true,
  "data": {
    "jobId": "uuid",
    "message": "Sync job accepted. Track progress via GET /api/v1/attendance/sync/logs/{jobId}"
  }
}
```
