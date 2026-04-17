# Data Models
## Office Management System (OMS)

**Version:** 3.0 — 8 services

---

## Schema Conventions

Every service applies these conventions uniformly. Deviating requires a formal ADR.

### Standard Fields — Every Entity

```java
@Id
@GeneratedValue(strategy = GenerationType.UUID)
private UUID id;                          // gen_random_uuid() at DB level

@Column(nullable = false, updatable = false)
private Instant createdAt;                // Set on INSERT only

@Column(nullable = false)
private Instant updatedAt;                // Set on INSERT and every UPDATE

@Column(nullable = false)
private UUID createdBy;                   // Resolved from SecurityContext JWT
```

### Location Scoping — Mandatory on All Feature Entities

```java
@Column(nullable = false)
private UUID locationId;                  // Present on every feature entity — never omit
```

Every repository query on a feature entity must include a `locationId` filter. No exceptions. This is enforced in code review and tested in every integration test.

### Soft Deletes

```java
private Instant deletedAt;               // NULL = active. Non-null = logically deleted.
```

All active-record queries append `WHERE deleted_at IS NULL`. Hard deletes are prohibited — they destroy audit traceability.

### UUID Primary Keys

Sequential integer IDs are forbidden. UUIDs prevent enumeration attacks and decouple ID generation from database auto-increment sequences. Generated with `gen_random_uuid()` at the database level.

### Column Naming

```
snake_case for all columns: user_id, location_id, created_at, first_badge_in
```

### Flyway Migration Conventions

- One file per schema change: `V{sequence}__{description}.sql`
- Forward-only — no rollback scripts (rollbacks are new forward migrations)
- All `NOT NULL` constraints declared explicitly
- Unique constraints declared in SQL migration, not only in JPA annotation
- Indexes created in a separate migration step (never blocking production deploys)

---

## auth-service — `auth_db`

### sessions (spring-session-jdbc, auto-generated schema)

| Column | Type | Notes |
|--------|------|-------|
| `primary_id` | VARCHAR(36) PK | Internal session ID |
| `session_id` | VARCHAR(36) UNIQUE | HTTP session key in the cookie |
| `creation_time` | BIGINT | Unix epoch ms |
| `last_access_time` | BIGINT | Updated on every request; used for inactivity expiry |
| `max_inactive_interval` | INT | Seconds; configured via `SESSION_EXPIRY_SECONDS` |
| `expiry_time` | BIGINT | Precomputed (`last_access_time + max_inactive_interval × 1000`) |
| `principal_name` | VARCHAR(100) | SSO subject claim (`sub`) |

**Why server-side sessions instead of stateless JWT:** HTTP-only server-side sessions are not accessible to JavaScript (XSS-safe). They can be revoked instantly (logout invalidates the session). Stateless JWTs cannot be revoked before expiry — a stolen JWT remains valid until it expires. For an internal office management tool, session-based auth provides stronger security guarantees.

### users

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `sso_sub` | VARCHAR(255) UNIQUE NOT NULL | SSO provider subject claim — primary lookup key |
| `email` | VARCHAR(255) UNIQUE NOT NULL | |
| `display_name` | VARCHAR(255) NOT NULL | |
| `personnel_id` | VARCHAR(100) | Arms HR system ID; used for badge sync mapping |
| `employment_start_date` | DATE | |
| `employment_end_date` | DATE | NULL = currently employed |
| `is_active` | BOOLEAN NOT NULL DEFAULT TRUE | |
| `created_at` | TIMESTAMPTZ NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ NOT NULL DEFAULT now() | |

### user_roles

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `user_id` | UUID NOT NULL | FK → users.id |
| `role` | VARCHAR(50) NOT NULL | `EMPLOYEE / MANAGER / HR / FACILITIES_ADMIN / SUPER_ADMIN` |
| `location_id` | UUID | NULL for SUPER_ADMIN (cross-location access) |
| `created_at` | TIMESTAMPTZ NOT NULL DEFAULT now() | |
| `deleted_at` | TIMESTAMPTZ | Soft-deleted when role is revoked |

**Unique constraint:** `(user_id, role, location_id)` — one role per location per user. NULL location_id is treated as a valid unique value for SUPER_ADMIN.

**Business rule — SUPER_ADMIN:** The only user with `location_id = NULL` on a `UserRole`. The `LocationRoleInterceptor` has an explicit bypass for this case. No other role may have `location_id = NULL`.

### locations

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `name` | VARCHAR(255) NOT NULL | e.g. "Accra HQ", "Lagos Office" |
| `country` | VARCHAR(100) NOT NULL | |
| `timezone` | VARCHAR(100) NOT NULL | IANA timezone, e.g. `Africa/Accra` |
| `is_active` | BOOLEAN NOT NULL DEFAULT TRUE | |
| `created_at` | TIMESTAMPTZ NOT NULL DEFAULT now() | |

### location_configs

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `location_id` | UUID UNIQUE NOT NULL | FK → locations.id; one config per location |
| `booking_window_days` | INT NOT NULL DEFAULT 14 | How far ahead a seat can be booked |
| `cancellation_cutoff_hours` | INT NOT NULL DEFAULT 2 | Hours before booking date when cancellation is blocked |
| `no_show_release_time` | TIME NOT NULL DEFAULT '10:00' | When the no-show job releases unclaimed bookings |
| `attendance_cutoff_time` | TIME NOT NULL DEFAULT '09:30' | Badge-in after this time = LATE |
| `max_remote_days_per_week` | INT NOT NULL DEFAULT 2 | Default remote day limit |
| `cross_midnight_gap_hours` | INT NOT NULL DEFAULT 6 | Badge gap that separates two sessions |
| `updated_at` | TIMESTAMPTZ | |

### public_holidays

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `location_id` | UUID NOT NULL | |
| `holiday_date` | DATE NOT NULL | |
| `name` | VARCHAR(255) | e.g. "Independence Day" |

**Unique constraint:** `(location_id, holiday_date)`

**Indexes:**
```sql
CREATE INDEX idx_users_sso_sub       ON users(sso_sub);
CREATE INDEX idx_users_email         ON users(email);
CREATE INDEX idx_users_personnel_id  ON users(personnel_id) WHERE personnel_id IS NOT NULL;
CREATE INDEX idx_user_roles_user     ON user_roles(user_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_user_roles_location ON user_roles(location_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_public_holidays_loc ON public_holidays(location_id, holiday_date);
```

---

## attendance-service — `attendance_db`

### badge_events

**Immutable after ingestion — no `updated_at`, no `deleted_at`. No UPDATE or DELETE ever.**

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `user_id` | UUID | NULL if `personnel_id` unresolved |
| `location_id` | UUID NOT NULL | |
| `personnel_id` | VARCHAR(100) NOT NULL | Raw Athena value — kept for traceability |
| `event_type` | VARCHAR(20) NOT NULL | `BADGE_IN` / `BADGE_OUT` |
| `occurred_at` | TIMESTAMPTZ NOT NULL | |
| `raw_payload` | JSONB | Full Athena row — for audit and re-processing |
| `ingested_at` | TIMESTAMPTZ NOT NULL DEFAULT now() | |

**Unique constraint:** `(personnel_id, occurred_at, location_id)` — deduplication on re-ingestion.

**Why JSONB raw_payload:** If the pipeline logic changes (e.g., new lateness calculation), the raw badge data is available for re-processing without re-querying Athena. This is the attendance equivalent of event sourcing.

### work_sessions

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `user_id` | UUID NOT NULL | |
| `location_id` | UUID NOT NULL | |
| `session_date` | DATE NOT NULL | Owned by the date of `first_badge_in` |
| `first_badge_in` | TIMESTAMPTZ NOT NULL | |
| `last_badge_out` | TIMESTAMPTZ | NULL if user is still in building |
| `total_duration_minutes` | INT | |
| `is_late` | BOOLEAN NOT NULL DEFAULT FALSE | `first_badge_in > attendance_cutoff_time` |
| `minutes_late` | INT NOT NULL DEFAULT 0 | |
| `crosses_midnight` | BOOLEAN NOT NULL DEFAULT FALSE | Session spans midnight (overnight shift) |
| `created_at` | TIMESTAMPTZ NOT NULL DEFAULT now() | |

### attendance_records

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `user_id` | UUID NOT NULL | |
| `location_id` | UUID NOT NULL | |
| `record_date` | DATE NOT NULL | |
| `status` | VARCHAR(20) NOT NULL | `PRESENT / LATE / ABSENT / REMOTE / OOO / PUBLIC_HOLIDAY` |
| `work_session_id` | UUID | FK → work_sessions.id (NULL for non-badge statuses like REMOTE) |
| `override_by` | UUID | SUPER_ADMIN user ID if manually overridden |
| `override_reason` | TEXT | |
| `created_at` | TIMESTAMPTZ NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | |

**Unique constraint:** `(user_id, record_date, location_id)` — one record per user per date.

**Why no deleted_at:** Attendance records are never logically deleted. Overrides update the status in place with an `override_by` and `override_reason` audit trail.

### badge_sync_job_logs

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `location_id` | UUID NOT NULL | |
| `sync_date` | DATE NOT NULL | The date the job processed |
| `started_at` | TIMESTAMPTZ NOT NULL | |
| `completed_at` | TIMESTAMPTZ | NULL while running |
| `records_queried` | INT | Total badge events from Athena |
| `records_inserted` | INT | New unique events inserted |
| `records_skipped` | INT | Duplicates (already in DB) |
| `records_failed` | INT | Unresolvable personnel IDs |
| `status` | VARCHAR(20) | `RUNNING / SUCCESS / PARTIAL / FAILED` |
| `error_message` | TEXT | Last error if status = FAILED |

### personnel_id_mapping (local read-model)

| Column | Type | Notes |
|--------|------|-------|
| `personnel_id` | VARCHAR(100) PK | Arms HR ID |
| `user_id` | UUID NOT NULL | OMS UUID |
| `synced_at` | TIMESTAMPTZ NOT NULL DEFAULT now() | |

**Indexes:**
```sql
CREATE INDEX idx_attendance_user_date
  ON attendance_records(user_id, record_date, location_id);

CREATE INDEX idx_badge_event_user_occurred
  ON badge_events(user_id, occurred_at, location_id);

CREATE INDEX idx_work_session_user_date
  ON work_sessions(user_id, session_date, location_id);
```

---

## seating-service — `seating_db`

### floors

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `location_id` | UUID NOT NULL | |
| `floor_number` | INT NOT NULL | |
| `name` | VARCHAR(100) | e.g. "Ground Floor", "Level 2" |
| `deleted_at` | TIMESTAMPTZ | |

**Unique constraint:** `(location_id, floor_number)`

### zones

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `floor_id` | UUID NOT NULL | FK → floors.id |
| `location_id` | UUID NOT NULL | Denormalised for query performance |
| `name` | VARCHAR(100) NOT NULL | e.g. "East Wing", "Quiet Zone" |
| `deleted_at` | TIMESTAMPTZ | |

### seats

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `zone_id` | UUID NOT NULL | FK → zones.id |
| `location_id` | UUID NOT NULL | Denormalised for query performance |
| `seat_label` | VARCHAR(50) NOT NULL | e.g. "E-14", "Q-03" |
| `seat_type` | VARCHAR(20) NOT NULL | `HOT_DESK / PERMANENT / BLOCKED` |
| `permanent_user_id` | UUID | Set when `seat_type = PERMANENT` |
| `is_bookable` | BOOLEAN NOT NULL DEFAULT TRUE | FALSE when `BLOCKED` |
| `deleted_at` | TIMESTAMPTZ | |

**Why denormalized `location_id` on zones and seats:** The floor plan query filters by `location_id` at every level. Denormalising avoids multi-level JOINs (location → floor → zone → seat) on the hottest read path.

### seat_bookings

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `seat_id` | UUID NOT NULL | FK → seats.id |
| `user_id` | UUID NOT NULL | |
| `location_id` | UUID NOT NULL | |
| `booking_date` | DATE NOT NULL | |
| `status` | VARCHAR(20) NOT NULL | `CONFIRMED / CANCELLED / RELEASED` |
| `block_reservation_id` | UUID | NULL for individual bookings |
| `created_by` | UUID NOT NULL | |
| `created_at` | TIMESTAMPTZ NOT NULL | |
| `deleted_at` | TIMESTAMPTZ | |
| `version` | BIGINT NOT NULL DEFAULT 0 | Optimistic locking (`@Version`) — prevents double-booking |

**Unique constraint:** `(seat_id, booking_date)` WHERE `status = 'CONFIRMED'` and `deleted_at IS NULL` — only one confirmed booking per seat per date.

### no_show_records

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `seat_booking_id` | UUID NOT NULL | FK → seat_bookings.id |
| `user_id` | UUID NOT NULL | |
| `location_id` | UUID NOT NULL | |
| `booking_date` | DATE NOT NULL | |
| `released_at` | TIMESTAMPTZ NOT NULL | |

**Indexes:**
```sql
-- Hottest query: real-time seat availability
CREATE INDEX idx_seat_booking_date_location
  ON seat_bookings(booking_date, location_id)
  WHERE deleted_at IS NULL AND status = 'CONFIRMED';

-- User's own bookings
CREATE INDEX idx_seat_booking_user
  ON seat_bookings(user_id, booking_date)
  WHERE deleted_at IS NULL;
```

---

## remote-service — `remote_db`

### remote_requests

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `user_id` | UUID NOT NULL | |
| `location_id` | UUID NOT NULL | |
| `request_date` | DATE NOT NULL | One row per requested date |
| `status` | VARCHAR(20) NOT NULL | `PENDING / APPROVED / REJECTED` |
| `recurring_schedule_id` | UUID | FK → recurring_remote_schedules.id (NULL for one-off) |
| `approved_by` | UUID | Approver user ID |
| `rejection_reason` | TEXT | |
| `created_at` | TIMESTAMPTZ NOT NULL | |
| `deleted_at` | TIMESTAMPTZ | |

### ooo_requests

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `user_id` | UUID NOT NULL | |
| `location_id` | UUID NOT NULL | |
| `start_date` | DATE NOT NULL | |
| `end_date` | DATE NOT NULL | |
| `status` | VARCHAR(20) NOT NULL | `PENDING / APPROVED / REJECTED` |
| `reason` | TEXT | |
| `approved_by` | UUID | |
| `created_at` | TIMESTAMPTZ NOT NULL | |

### approval_delegates

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `manager_id` | UUID NOT NULL | |
| `delegate_id` | UUID NOT NULL | |
| `location_id` | UUID NOT NULL | |
| `start_date` | DATE NOT NULL | |
| `end_date` | DATE NOT NULL | |
| `is_active` | BOOLEAN NOT NULL DEFAULT TRUE | |
| `created_at` | TIMESTAMPTZ NOT NULL | |

### remote_day_policies

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `location_id` | UUID NOT NULL | |
| `team_manager_id` | UUID | NULL = location-level default policy |
| `max_days_per_week` | INT NOT NULL | |
| `team_overlap_threshold_pct` | INT | Max % of team remote simultaneously |
| `policy_type` | VARCHAR(20) NOT NULL | `HARD_BLOCK / SOFT_WARNING` |
| `updated_at` | TIMESTAMPTZ | |

---

## audit-service — `audit_db`

### audit_logs

**Append-only. No `updated_at`. No `deleted_at`. DB user has INSERT privilege ONLY.**

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `actor_id` | UUID NOT NULL | User who performed the action |
| `actor_role` | VARCHAR(50) NOT NULL | Role at time of action (captured in event — role may change later) |
| `action` | VARCHAR(100) NOT NULL | e.g. `SEAT_BOOKED`, `ATTENDANCE_OVERRIDDEN`, `ASSET_ASSIGNED` |
| `entity_type` | VARCHAR(100) NOT NULL | e.g. `SeatBooking`, `AttendanceRecord` |
| `entity_id` | UUID NOT NULL | |
| `location_id` | UUID | |
| `previous_state` | JSONB | Full entity state before mutation (NULL for creates) |
| `new_state` | JSONB | Full entity state after mutation (NULL for deletes) |
| `occurred_at` | TIMESTAMPTZ NOT NULL | |
| `correlation_id` | UUID | Trace ID from originating HTTP request |

**Why JSONB for previous/new state:** Storing the full serialised entity at mutation time means audit records are self-contained. No JOIN to other tables is needed to understand what changed. This is critical for compliance investigations — the audit record is the source of truth for what happened, not a pointer to the current state of a possibly-modified record.

**Indexes:**
```sql
CREATE INDEX idx_audit_actor    ON audit_logs(actor_id, occurred_at DESC);
CREATE INDEX idx_audit_entity   ON audit_logs(entity_type, entity_id);
CREATE INDEX idx_audit_location ON audit_logs(location_id, occurred_at DESC);
CREATE INDEX idx_audit_action   ON audit_logs(action, occurred_at DESC);
```

### processed_events (idempotency)

| Column | Type | Notes |
|--------|------|-------|
| `event_id` | UUID PK | OmsEvent.eventId — deduplication key |
| `processed_at` | TIMESTAMPTZ NOT NULL | |

---

## notification-service — `notification_db`

### notifications

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `user_id` | UUID NOT NULL | Recipient |
| `location_id` | UUID | |
| `message` | TEXT NOT NULL | Rendered notification body |
| `type` | VARCHAR(100) NOT NULL | e.g. `REMOTE_APPROVED`, `SEAT_BOOKING_CONFIRMED` |
| `is_read` | BOOLEAN NOT NULL DEFAULT FALSE | |
| `created_at` | TIMESTAMPTZ NOT NULL | |
| `read_at` | TIMESTAMPTZ | |

**Index:**
```sql
CREATE INDEX idx_notification_user_unread
  ON notifications(user_id, created_at DESC)
  WHERE is_read = FALSE;
```

---

## inventory-service — `inventory_db`

### supply_categories

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `name` | VARCHAR(255) NOT NULL | e.g. "Stationery", "IT Peripherals" |
| `location_id` | UUID NOT NULL | |

### supply_items

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `category_id` | UUID NOT NULL | FK → supply_categories.id |
| `location_id` | UUID NOT NULL | |
| `name` | VARCHAR(255) NOT NULL | |
| `unit` | VARCHAR(50) NOT NULL | e.g. "box", "piece", "ream" |
| `reorder_threshold` | INT NOT NULL | Triggers stock.low event when crossed |
| `is_active` | BOOLEAN NOT NULL DEFAULT TRUE | |

### supply_stock_entries

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `item_id` | UUID NOT NULL | FK → supply_items.id |
| `location_id` | UUID NOT NULL | |
| `quantity_received` | INT NOT NULL | |
| `quantity_remaining` | INT NOT NULL | Decremented on fulfilment |
| `expiry_date` | DATE | NULL for non-perishable items |
| `received_at` | DATE NOT NULL | Used for FIFO ordering |
| `created_at` | TIMESTAMPTZ NOT NULL | |
| `written_off_at` | TIMESTAMPTZ | Set when batch is written off |

### supply_requests

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `user_id` | UUID NOT NULL | Requester |
| `location_id` | UUID NOT NULL | |
| `status` | VARCHAR(30) NOT NULL | `PENDING_MANAGER / PENDING_FACILITIES / FULFILLED / REJECTED` |
| `justification` | TEXT | |
| `approved_by_manager` | UUID | |
| `approved_by_facilities` | UUID | |
| `rejection_reason` | TEXT | |
| `created_at` | TIMESTAMPTZ NOT NULL | |
| `fulfilled_at` | TIMESTAMPTZ | |
| `deleted_at` | TIMESTAMPTZ | |

### supply_request_items

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `request_id` | UUID NOT NULL | FK → supply_requests.id |
| `item_id` | UUID NOT NULL | FK → supply_items.id |
| `quantity_requested` | INT NOT NULL | |
| `quantity_fulfilled` | INT | Set on fulfilment (may differ from requested) |

### asset_categories

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `name` | VARCHAR(255) NOT NULL | e.g. "Laptop", "Monitor", "Access Card" |
| `location_id` | UUID NOT NULL | |

### assets

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `category_id` | UUID NOT NULL | |
| `location_id` | UUID NOT NULL | |
| `serial_number` | VARCHAR(255) UNIQUE NOT NULL | |
| `description` | TEXT NOT NULL | |
| `status` | VARCHAR(20) NOT NULL | `AVAILABLE / ASSIGNED / UNDER_MAINTENANCE / RETIRED` |
| `purchase_date` | DATE | |
| `purchase_value` | NUMERIC(10,2) | |
| `created_at` | TIMESTAMPTZ NOT NULL | |
| `deleted_at` | TIMESTAMPTZ | |

### asset_assignments

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `asset_id` | UUID NOT NULL | FK → assets.id |
| `user_id` | UUID NOT NULL | Assigned user |
| `location_id` | UUID NOT NULL | |
| `status` | VARCHAR(20) NOT NULL | `ASSIGNED / ACKNOWLEDGED / RETURNED / RETIRED` |
| `assigned_by` | UUID NOT NULL | |
| `assigned_at` | TIMESTAMPTZ NOT NULL | |
| `acknowledged_at` | TIMESTAMPTZ | |
| `returned_at` | TIMESTAMPTZ | |
| `notes` | TEXT | |

### fault_reports

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `asset_id` | UUID NOT NULL | |
| `reported_by` | UUID NOT NULL | Employee user ID |
| `location_id` | UUID NOT NULL | |
| `description` | TEXT NOT NULL | |
| `severity` | VARCHAR(20) NOT NULL | `LOW / MEDIUM / HIGH / CRITICAL` |
| `status` | VARCHAR(20) NOT NULL | `OPEN / IN_PROGRESS / RESOLVED` |
| `created_at` | TIMESTAMPTZ NOT NULL | |
| `resolved_at` | TIMESTAMPTZ | |

### maintenance_records

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `asset_id` | UUID NOT NULL | |
| `location_id` | UUID NOT NULL | |
| `type` | VARCHAR(30) NOT NULL | `REPAIR / SCHEDULED_SERVICE / INSPECTION / UPGRADE` |
| `description` | TEXT NOT NULL | |
| `performed_by` | UUID NOT NULL | Facilities Admin |
| `performed_at` | DATE NOT NULL | |
| `created_at` | TIMESTAMPTZ NOT NULL | |

---

## workplace-service — `workplace_db`

### visitor_profiles

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `display_name` | VARCHAR(255) NOT NULL | |
| `email` | VARCHAR(255) UNIQUE NOT NULL | Natural key for returning visitors |
| `phone` | VARCHAR(50) | |
| `created_at` | TIMESTAMPTZ NOT NULL | |

### parent_visits

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `visitor_profile_id` | UUID NOT NULL | FK → visitor_profiles.id |
| `host_user_id` | UUID NOT NULL | OMS user who invited the visitor |
| `location_id` | UUID NOT NULL | |
| `expected_date` | DATE NOT NULL | |
| `purpose` | TEXT | |
| `status` | VARCHAR(20) NOT NULL | `EXPECTED / CHECKED_IN / CHECKED_OUT / NO_SHOW` |
| `created_at` | TIMESTAMPTZ NOT NULL | |

### visit_records

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `parent_visit_id` | UUID NOT NULL | FK → parent_visits.id |
| `location_id` | UUID NOT NULL | |
| `checked_in_at` | TIMESTAMPTZ | |
| `checked_out_at` | TIMESTAMPTZ | |
| `agreement_template_version_id` | UUID | Version of template signed at check-in |
| `signed_at` | TIMESTAMPTZ | |

### agreement_templates

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `location_id` | UUID NOT NULL | |
| `version_label` | VARCHAR(100) NOT NULL | e.g. "v2.1 – 2026 NDA" |
| `content` | TEXT NOT NULL | Full template text |
| `is_active` | BOOLEAN NOT NULL DEFAULT FALSE | Only ONE active template per location at a time |
| `created_by` | UUID NOT NULL | |
| `created_at` | TIMESTAMPTZ NOT NULL | |
| `activated_at` | TIMESTAMPTZ | When made active |

**Constraint:** Only one `is_active = TRUE` row per `location_id` enforced via a partial unique index.

```sql
CREATE UNIQUE INDEX idx_agreement_one_active_per_location
  ON agreement_templates(location_id) WHERE is_active = TRUE;
```

### event_series

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `location_id` | UUID NOT NULL | |
| `title` | VARCHAR(255) NOT NULL | |
| `recurrence_pattern` | JSONB | Recurrence rule (cron-like or `{ "frequency": "WEEKLY", "dayOfWeek": "MONDAY" }`) |
| `organizer_id` | UUID NOT NULL | |
| `created_at` | TIMESTAMPTZ NOT NULL | |
| `deleted_at` | TIMESTAMPTZ | |

### events

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `event_series_id` | UUID | FK → event_series.id (NULL for standalone) |
| `location_id` | UUID NOT NULL | |
| `title` | VARCHAR(255) NOT NULL | |
| `event_date` | DATE NOT NULL | |
| `start_time` | TIME NOT NULL | |
| `end_time` | TIME NOT NULL | |
| `capacity` | INT NOT NULL | |
| `description` | TEXT | |
| `organizer_id` | UUID NOT NULL | |
| `status` | VARCHAR(20) NOT NULL | `ACTIVE / CANCELLED` |
| `created_at` | TIMESTAMPTZ NOT NULL | |
| `deleted_at` | TIMESTAMPTZ | |

### event_invites

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `event_id` | UUID NOT NULL | FK → events.id |
| `user_id` | UUID NOT NULL | |
| `location_id` | UUID NOT NULL | |
| `status` | VARCHAR(20) NOT NULL | `ATTENDING / WAITLISTED / CANCELLED` |
| `waitlist_position` | INT | Set when status = WAITLISTED |
| `created_at` | TIMESTAMPTZ NOT NULL | |
| `updated_at` | TIMESTAMPTZ | |

**Unique constraint:** `(event_id, user_id)` — one RSVP per user per event.

---

## Entity Ownership Summary

No service accesses another service's database. Cross-service references use UUID values only — the UUID is a logical pointer, not a foreign key enforced by the database.

| Service | Primary Owned Tables |
|---------|---------------------|
| `auth-service` | sessions, users, user_roles, locations, location_configs, public_holidays |
| `attendance-service` | badge_events, work_sessions, attendance_records, badge_sync_job_logs, personnel_id_mapping |
| `seating-service` | floors, zones, seats, seat_bookings, block_reservations, no_show_records |
| `remote-service` | remote_requests, recurring_remote_schedules, ooo_requests, approval_delegates, remote_day_policies |
| `notification-service` | notifications, notification_templates |
| `audit-service` | audit_logs, processed_events |
| `inventory-service` | supply_categories, supply_items, supply_stock_entries, supply_requests, supply_request_items, asset_categories, assets, asset_assignments, asset_requests, fault_reports, maintenance_records |
| `workplace-service` | visitor_profiles, parent_visits, visit_records, agreement_templates, event_series, events, event_invites |

**Plus `user_summaries` and `processed_events` tables in every service that consumes RabbitMQ events.**
