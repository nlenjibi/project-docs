# API Reference
## Office Management System (OMS)

**Version:** 3.0 | Base: `http://localhost:8080` (dev) | All routes via Spring Cloud Gateway port 8080

---

## Global Conventions

| Convention | Value |
|---|---|
| Base path | `/api/v1/` |
| Auth | HTTP-only session cookie (set by auth-service on login) |
| Content-Type | `application/json` |
| Response envelope | `{ "success": true/false, "data": {}, "error": null }` |
| Pagination | `?page=0&size=20` on all list endpoints |
| Versioning | URL-based (`/api/v1/`) — breaking changes increment to `/api/v2/` |
| Timestamps | ISO 8601 UTC — `2026-04-17T09:15:00Z` |
| IDs | UUID v4 |
| Error format | `{ "success": false, "data": null, "error": { "code": "ERR_001", "message": "..." } }` |

---

## 1. auth-service (port 8081)

### Authentication

| Method | Path | Auth Required | Role | Description |
|---|---|---|---|---|
| `POST` | `/api/v1/auth/login` | No | — | Initiates SSO redirect |
| `GET` | `/api/v1/auth/callback` | No | — | SSO callback; issues session cookie |
| `POST` | `/api/v1/auth/logout` | Yes | Any | Invalidates session |
| `GET` | `/api/v1/auth/validate` | Internal | — | Called by gateway on every request |
| `GET` | `/api/v1/auth/me` | Yes | Any | Returns current user identity |

### Users

| Method | Path | Auth Required | Role | Description |
|---|---|---|---|---|
| `GET` | `/api/v1/users/{id}` | Yes | Any | Get user profile |
| `GET` | `/api/v1/users/{id}/roles` | Yes | HR, SUPER_ADMIN | Get user's roles across locations |
| `POST` | `/api/v1/users/{id}/roles` | Yes | SUPER_ADMIN | Assign role at location |
| `DELETE` | `/api/v1/users/{id}/roles/{roleId}` | Yes | SUPER_ADMIN | Revoke role |

### Locations

| Method | Path | Auth Required | Role | Description |
|---|---|---|---|---|
| `GET` | `/api/v1/locations` | Yes | Any | List all locations |
| `GET` | `/api/v1/locations/{id}/config` | Yes | Any | Get location configuration |
| `PATCH` | `/api/v1/locations/{id}/config` | Yes | SUPER_ADMIN | Update location configuration |

**Sample response — GET /api/v1/auth/me:**
```json
{
  "success": true,
  "data": {
    "userId": "550e8400-e29b-41d4-a716-446655440000",
    "email": "john.doe@company.com",
    "displayName": "John Doe",
    "roles": [
      { "role": "MANAGER", "locationId": "660e8400-e29b-41d4-a716-446655440001" },
      { "role": "EMPLOYEE", "locationId": "770e8400-e29b-41d4-a716-446655440002" }
    ]
  },
  "error": null
}
```

---

## 2. attendance-service (port 8083)

| Method | Path | Auth Required | Role | Description |
|---|---|---|---|---|
| `GET` | `/api/v1/attendance/{userId}` | Yes | EMPLOYEE (own), MANAGER, HR | Attendance history |
| `GET` | `/api/v1/attendance/team/{managerId}` | Yes | MANAGER, HR | Team attendance |
| `GET` | `/api/v1/attendance/location/{locationId}` | Yes | HR, SUPER_ADMIN | Location-wide attendance |
| `POST` | `/api/v1/attendance/{id}/override` | Yes | SUPER_ADMIN | Override attendance record |
| `GET` | `/api/v1/attendance/reports/no-shows` | Yes | HR, FACILITIES_ADMIN | No-show report |
| `GET` | `/api/v1/attendance/export` | Yes | HR, SUPER_ADMIN | Export CSV/Excel |
| `GET` | `/api/v1/attendance/sync/logs` | Yes | SUPER_ADMIN, FACILITIES_ADMIN | Badge sync logs |
| `POST` | `/api/v1/attendance/sync/trigger` | Yes | SUPER_ADMIN | Manual sync trigger |

**Attendance record statuses:** `PRESENT` `LATE` `ABSENT` `REMOTE` `OOO` `PUBLIC_HOLIDAY`

**Query parameters for list endpoints:** `?from=2026-04-01&to=2026-04-30&status=PRESENT&page=0&size=20`

**Sample response — GET /api/v1/attendance/{userId}:**
```json
{
  "success": true,
  "data": {
    "content": [
      {
        "id": "uuid",
        "userId": "uuid",
        "recordDate": "2026-04-17",
        "status": "PRESENT",
        "arrivalTime": "08:45:00",
        "departureTime": "17:30:00",
        "isLate": false,
        "locationId": "uuid"
      }
    ],
    "totalElements": 22,
    "totalPages": 2,
    "page": 0,
    "size": 20
  },
  "error": null
}
```

---

## 3. seating-service (port 8084)

| Method | Path | Auth Required | Role | Description |
|---|---|---|---|---|
| `GET` | `/api/v1/locations/{id}/floor-plan` | Yes | Any | Floor plan with real-time availability |
| `POST` | `/api/v1/seat-bookings` | Yes | EMPLOYEE | Book a hot-desk |
| `DELETE` | `/api/v1/seat-bookings/{id}` | Yes | EMPLOYEE (own) | Cancel booking |
| `GET` | `/api/v1/seat-bookings/my` | Yes | Any | Own upcoming bookings |
| `POST` | `/api/v1/seat-bookings/block` | Yes | MANAGER | Block-book seats for team |
| `POST` | `/api/v1/seats/{id}/permanent` | Yes | HR, FACILITIES_ADMIN | Assign permanent seat |
| `DELETE` | `/api/v1/seats/{id}/permanent` | Yes | HR, FACILITIES_ADMIN | Release permanent seat |
| `PUT` | `/api/v1/locations/{id}/floor-plan` | Yes | FACILITIES_ADMIN | Configure floor plan |

**Sample request — POST /api/v1/seat-bookings:**
```json
{
  "seatId": "550e8400-e29b-41d4-a716-446655440000",
  "bookingDate": "2026-04-18",
  "locationId": "660e8400-e29b-41d4-a716-446655440001"
}
```

**Sample response:**
```json
{
  "success": true,
  "data": {
    "bookingId": "uuid",
    "seatId": "uuid",
    "seatLabel": "A-204",
    "zone": "Open Plan",
    "floor": "2nd Floor",
    "bookingDate": "2026-04-18",
    "status": "CONFIRMED",
    "locationId": "uuid"
  },
  "error": null
}
```

---

## 4. remote-service (port 8085)

| Method | Path | Auth Required | Role | Description |
|---|---|---|---|---|
| `POST` | `/api/v1/remote-requests` | Yes | EMPLOYEE | Submit remote day request |
| `POST` | `/api/v1/ooo-requests` | Yes | EMPLOYEE | Submit OOO request |
| `PATCH` | `/api/v1/remote-requests/{id}/approve` | Yes | MANAGER | Approve request |
| `PATCH` | `/api/v1/remote-requests/{id}/reject` | Yes | MANAGER | Reject with reason |
| `GET` | `/api/v1/teams/{managerId}/schedule` | Yes | MANAGER, HR | Team schedule calendar |
| `POST` | `/api/v1/approval-delegates` | Yes | MANAGER | Nominate approval delegate |
| `GET` | `/api/v1/remote-day-policies/{locationId}` | Yes | MANAGER, HR | Get location policy |
| `PATCH` | `/api/v1/remote-day-policies/{managerId}` | Yes | HR, SUPER_ADMIN | Override team policy |
| `POST` | `/api/v1/remote-requests/{id}/override` | Yes | SUPER_ADMIN | Override past dates |
| `GET` | `/api/v1/remote-requests/history` | Yes | EMPLOYEE | Own request history |

**Policy enforcement response (SOFT_WARNING example):**
```json
{
  "success": true,
  "data": {
    "requestId": "uuid",
    "status": "PENDING_APPROVAL",
    "warning": "This request brings your team remote count to 4/5 (80% threshold). Submitted with warning."
  },
  "error": null
}
```

---

## 5. notification-service (port 8086)

| Method | Path | Auth Required | Role | Description |
|---|---|---|---|---|
| `GET` | `/api/v1/notifications/my` | Yes | Any | Own unread notifications |
| `PATCH` | `/api/v1/notifications/{id}/read` | Yes | Any | Mark as read |
| `PATCH` | `/api/v1/notifications/read-all` | Yes | Any | Mark all as read |

*No write endpoints accessible to other services. Populated exclusively via RabbitMQ consumers.*

---

## 6. audit-service (port 8087)

| Method | Path | Auth Required | Role | Description |
|---|---|---|---|---|
| `GET` | `/api/v1/audit-logs` | Yes | HR, FACILITIES_ADMIN, SUPER_ADMIN | Paginated audit log |
| `GET` | `/api/v1/audit-logs?entityType=SeatBooking` | Yes | HR, SUPER_ADMIN | Filter by entity type |
| `GET` | `/api/v1/audit-logs?actorId={userId}` | Yes | HR, SUPER_ADMIN | Filter by actor |
| `GET` | `/api/v1/audit-logs?locationId={id}` | Yes | HR, FACILITIES_ADMIN | Filter by location |
| `GET` | `/api/v1/audit-logs?action=SEAT_BOOKED` | Yes | HR, SUPER_ADMIN | Filter by action |

*No write endpoints. Populated exclusively via RabbitMQ consumer.*

---

## 7. inventory-service (port 8090) — Phase 2

### Supplies

| Method | Path | Auth Required | Role | Description |
|---|---|---|---|---|
| `GET` | `/api/v1/supplies/catalogue` | Yes | Any | Browse supply catalogue |
| `POST` | `/api/v1/supplies/catalogue` | Yes | FACILITIES_ADMIN | Add supply item |
| `PATCH` | `/api/v1/supplies/catalogue/{id}` | Yes | FACILITIES_ADMIN | Update item |
| `POST` | `/api/v1/supply-requests` | Yes | EMPLOYEE | Submit supply request |
| `PATCH` | `/api/v1/supply-requests/{id}/approve-stage1` | Yes | MANAGER | Manager approve |
| `PATCH` | `/api/v1/supply-requests/{id}/reject-stage1` | Yes | MANAGER | Manager reject |
| `PATCH` | `/api/v1/supply-requests/{id}/fulfil` | Yes | FACILITIES_ADMIN | Fulfil request |
| `PATCH` | `/api/v1/supply-requests/{id}/reject-stage2` | Yes | FACILITIES_ADMIN | Reject at fulfilment |
| `GET` | `/api/v1/supplies/inventory-report` | Yes | HR, FACILITIES_ADMIN | Inventory report |
| `POST` | `/api/v1/supplies/stock-entries` | Yes | FACILITIES_ADMIN | Add stock batch |

### Assets

| Method | Path | Auth Required | Role | Description |
|---|---|---|---|---|
| `GET` | `/api/v1/assets` | Yes | HR, FACILITIES_ADMIN | Asset register |
| `POST` | `/api/v1/assets` | Yes | FACILITIES_ADMIN | Add asset |
| `GET` | `/api/v1/assets/{id}` | Yes | HR, FACILITIES_ADMIN | Asset detail |
| `PATCH` | `/api/v1/assets/{id}/retire` | Yes | FACILITIES_ADMIN | Retire asset |
| `GET` | `/api/v1/assets/my` | Yes | EMPLOYEE | Own assigned assets |
| `POST` | `/api/v1/asset-assignments` | Yes | FACILITIES_ADMIN | Assign to user |
| `PATCH` | `/api/v1/asset-assignments/{id}/acknowledge` | Yes | EMPLOYEE | Acknowledge receipt |
| `PATCH` | `/api/v1/asset-assignments/{id}/return` | Yes | EMPLOYEE, FACILITIES_ADMIN | Return asset |
| `POST` | `/api/v1/asset-requests` | Yes | EMPLOYEE | Request an asset |
| `PATCH` | `/api/v1/asset-requests/{id}/approve` | Yes | MANAGER | Approve request |
| `PATCH` | `/api/v1/asset-requests/{id}/fulfil` | Yes | FACILITIES_ADMIN | Fulfil request |
| `POST` | `/api/v1/fault-reports` | Yes | EMPLOYEE | Report fault |
| `POST` | `/api/v1/maintenance-records` | Yes | FACILITIES_ADMIN | Log maintenance |
| `GET` | `/api/v1/assets/{id}/audit-log` | Yes | HR, FACILITIES_ADMIN | Asset history |

---

## 8. workplace-service (port 8088) — Phase 2

### Visitors

| Method | Path | Auth Required | Role | Description |
|---|---|---|---|---|
| `POST` | `/api/v1/visitors/pre-register` | Yes | EMPLOYEE | Pre-register expected visitor |
| `POST` | `/api/v1/visitors/walk-in` | Yes | FACILITIES_ADMIN | Register walk-in visitor |
| `PATCH` | `/api/v1/visits/{id}/check-in` | Yes | FACILITIES_ADMIN | Check visitor in |
| `PATCH` | `/api/v1/visits/{id}/check-out` | Yes | FACILITIES_ADMIN | Check visitor out |
| `GET` | `/api/v1/visits/history` | Yes | EMPLOYEE | Own visitor history |
| `GET` | `/api/v1/visits/location/{locationId}` | Yes | HR, FACILITIES_ADMIN | All visits at location |
| `POST` | `/api/v1/agreement-templates` | Yes | FACILITIES_ADMIN | Create agreement template |
| `GET` | `/api/v1/agreement-templates/active` | Yes | Any | Current active template |

### Events

| Method | Path | Auth Required | Role | Description |
|---|---|---|---|---|
| `GET` | `/api/v1/events` | Yes | Any | Upcoming events |
| `GET` | `/api/v1/events/{id}` | Yes | Any | Event detail |
| `POST` | `/api/v1/events` | Yes | MANAGER, HR, FACILITIES_ADMIN | Create event |
| `PATCH` | `/api/v1/events/{id}` | Yes | Organiser, FACILITIES_ADMIN | Update event |
| `DELETE` | `/api/v1/events/{id}` | Yes | Organiser, FACILITIES_ADMIN | Cancel event |
| `POST` | `/api/v1/events/{id}/rsvp` | Yes | EMPLOYEE | RSVP |
| `DELETE` | `/api/v1/events/{id}/rsvp` | Yes | EMPLOYEE | Cancel RSVP |
| `GET` | `/api/v1/events/{id}/attendees` | Yes | Organiser, MANAGER, HR | Attendee list |

---

## Error Codes

| Code | HTTP Status | Meaning |
|---|---|---|
| `ERR_AUTH_001` | 401 | No valid session |
| `ERR_AUTH_002` | 403 | Insufficient role or wrong location |
| `ERR_VAL_001` | 400 | Validation error — see `details` array |
| `ERR_NOT_FOUND` | 404 | Resource not found |
| `ERR_CONFLICT` | 409 | Conflict (e.g., seat already booked) |
| `ERR_POLICY` | 422 | Business rule violation (e.g., HARD_BLOCK on remote policy) |
| `ERR_INTERNAL` | 500 | Internal server error |

**Validation error response:**
```json
{
  "success": false,
  "data": null,
  "error": {
    "code": "ERR_VAL_001",
    "message": "Validation failed",
    "details": [
      { "field": "bookingDate", "message": "must not be in the past" },
      { "field": "seatId", "message": "must not be null" }
    ]
  }
}
```
