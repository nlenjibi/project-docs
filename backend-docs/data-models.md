# Data Models
## Office Management System (OMS)

---

## Schema Conventions

Every service applies these conventions consistently. Deviating from them requires an ADR.

### Standard Fields — Every Entity

```java
@Id
@GeneratedValue(strategy = GenerationType.UUID)
private UUID id;

private Instant createdAt;
private Instant updatedAt;
private UUID createdBy;           // Resolved from session context; FK → User (logical)
```

### Feature Entities (all services except identity-service and user-service globals)

```java
private UUID locationId;          // Mandatory. Never omit. Every query filters by it.
```

### Soft Delete (most mutable entities)

```java
private Instant deletedAt;        // NULL = active. Non-null = logically deleted.
```

All active-record queries use `WHERE deleted_at IS NULL`. Hard deletes are never performed.

### UUID Primary Keys

```java
@Id
@GeneratedValue(strategy = GenerationType.UUID)
private UUID id;
```

Never use sequential integers. UUIDs prevent enumeration attacks.

### DB Column Naming

```sql
snake_case for all columns:
  user_id, location_id, created_at, deleted_at, first_badge_in
```

---

## Flyway Migration Conventions

- One migration file per schema change: `V{number}__{description}.sql`
- Migrations are forward-only — no rollback scripts.
- Every table created with explicit `NOT NULL` constraints where appropriate.
- Unique constraints declared in migration, not only in JPA.

---

## identity-service — `identity_db`

### sessions

Managed by `spring-session-jdbc`. Schema generated automatically.

| Column | Type | Notes |
|--------|------|-------|
| `primary_id` | VARCHAR(36) PK | Session ID |
| `session_id` | VARCHAR(36) UNIQUE | HTTP session key |
| `creation_time` | BIGINT | Unix epoch ms |
| `last_access_time` | BIGINT | Used for expiry checks |
| `max_inactive_interval` | INT | Seconds; configurable via `SESSION_EXPIRY_SECONDS` |
| `expiry_time` | BIGINT | Precomputed expiry |
| `principal_name` | VARCHAR(100) | SSO subject (`sub`) claim |

---

## user-service — `user_db`

### users

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `sso_sub` | VARCHAR(255) UNIQUE NOT NULL | SSO provider subject claim |
| `email` | VARCHAR(255) UNIQUE NOT NULL | |
| `display_name` | VARCHAR(255) NOT NULL | |
| `personnel_id` | VARCHAR(100) | Arms HR system ID |
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
| `role` | VARCHAR(50) NOT NULL | EMPLOYEE / MANAGER / HR / FACILITIES_ADMIN / SUPER_ADMIN |
| `location_id` | UUID | NULL for Super Admin (cross-location access) |
| `created_at` | TIMESTAMPTZ NOT NULL DEFAULT now() | |
| `deleted_at` | TIMESTAMPTZ | |

**Unique constraint:** `(user_id, role, location_id)` — one role per location.

### locations

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `name` | VARCHAR(255) NOT NULL | |
| `country` | VARCHAR(100) | |
| `timezone` | VARCHAR(100) NOT NULL | e.g. `Africa/Accra` |
| `is_active` | BOOLEAN NOT NULL DEFAULT TRUE | |
| `created_at` | TIMESTAMPTZ | |

### location_configs

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `location_id` | UUID UNIQUE NOT NULL | FK → locations.id |
| `booking_window_days` | INT NOT NULL DEFAULT 14 | |
| `cancellation_cutoff_hours` | INT NOT NULL DEFAULT 2 | |
| `no_show_release_time` | TIME NOT NULL DEFAULT '10:00' | |
| `attendance_cutoff_time` | TIME NOT NULL DEFAULT '09:30' | |
| `max_remote_days_per_week` | INT NOT NULL DEFAULT 2 | |
| `cross_midnight_gap_hours` | INT NOT NULL DEFAULT 6 | |
| `updated_at` | TIMESTAMPTZ | |

### public_holidays

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `location_id` | UUID NOT NULL | |
| `holiday_date` | DATE NOT NULL | |
| `name` | VARCHAR(255) | |

---

## attendance-service — `attendance_db`

### badge_events

**Immutable after ingestion. No `deleted_at`.**

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `user_id` | UUID | NULL if personnel_id unresolved |
| `location_id` | UUID NOT NULL | |
| `personnel_id` | VARCHAR(100) NOT NULL | Raw Athena value |
| `event_type` | VARCHAR(20) NOT NULL | `BADGE_IN` / `BADGE_OUT` |
| `occurred_at` | TIMESTAMPTZ NOT NULL | |
| `raw_payload` | JSONB | Full Athena row for traceability |
| `ingested_at` | TIMESTAMPTZ NOT NULL DEFAULT now() | |

**Unique constraint:** `(personnel_id, occurred_at, location_id)` — deduplication.

### work_sessions

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `user_id` | UUID NOT NULL | |
| `location_id` | UUID NOT NULL | |
| `session_date` | DATE NOT NULL | Owned by start date |
| `first_badge_in` | TIMESTAMPTZ NOT NULL | |
| `last_badge_out` | TIMESTAMPTZ | NULL if still in building |
| `total_duration_minutes` | INT | |
| `is_late` | BOOLEAN NOT NULL DEFAULT FALSE | |
| `minutes_late` | INT NOT NULL DEFAULT 0 | |
| `crosses_midnight` | BOOLEAN NOT NULL DEFAULT FALSE | |
| `created_at` | TIMESTAMPTZ NOT NULL DEFAULT now() | |

### attendance_records

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `user_id` | UUID NOT NULL | |
| `location_id` | UUID NOT NULL | |
| `record_date` | DATE NOT NULL | |
| `status` | VARCHAR(20) NOT NULL | `PRESENT / LATE / ABSENT / REMOTE / OOO / PUBLIC_HOLIDAY` |
| `work_session_id` | UUID | FK → work_sessions.id (NULL for non-badge statuses) |
| `override_by` | UUID | Super Admin user ID if manually overridden |
| `override_reason` | TEXT | |
| `created_at` | TIMESTAMPTZ NOT NULL DEFAULT now() | |
| `updated_at` | TIMESTAMPTZ | |

**Unique constraint:** `(user_id, record_date, location_id)` — one record per user per date.

### badge_sync_job_logs

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `location_id` | UUID NOT NULL | |
| `sync_date` | DATE NOT NULL | The date the job ran for |
| `started_at` | TIMESTAMPTZ NOT NULL | |
| `completed_at` | TIMESTAMPTZ | |
| `records_queried` | INT | |
| `records_inserted` | INT | |
| `records_skipped` | INT | |
| `records_failed` | INT | |
| `status` | VARCHAR(20) | `SUCCESS / PARTIAL / FAILED` |
| `error_message` | TEXT | |

---

## seating-service — `seating_db`

### floors

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `location_id` | UUID NOT NULL | |
| `floor_number` | INT NOT NULL | |
| `name` | VARCHAR(100) | |
| `deleted_at` | TIMESTAMPTZ | |

### zones

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `floor_id` | UUID NOT NULL | |
| `location_id` | UUID NOT NULL | |
| `name` | VARCHAR(100) | |
| `deleted_at` | TIMESTAMPTZ | |

### seats

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `zone_id` | UUID NOT NULL | |
| `location_id` | UUID NOT NULL | |
| `seat_label` | VARCHAR(50) | e.g. `A-12` |
| `seat_type` | VARCHAR(20) | `HOT_DESK / PERMANENT / BLOCKED` |
| `permanent_user_id` | UUID | Set when assigned permanently |
| `is_bookable` | BOOLEAN NOT NULL DEFAULT TRUE | |
| `deleted_at` | TIMESTAMPTZ | |

### seat_bookings

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `seat_id` | UUID NOT NULL | |
| `user_id` | UUID NOT NULL | |
| `location_id` | UUID NOT NULL | |
| `booking_date` | DATE NOT NULL | |
| `status` | VARCHAR(20) NOT NULL | `CONFIRMED / CANCELLED / RELEASED` |
| `block_reservation_id` | UUID | NULL for individual bookings |
| `created_by` | UUID NOT NULL | |
| `created_at` | TIMESTAMPTZ NOT NULL | |
| `deleted_at` | TIMESTAMPTZ | |

### no_show_records

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `seat_booking_id` | UUID NOT NULL | |
| `user_id` | UUID NOT NULL | |
| `location_id` | UUID NOT NULL | |
| `booking_date` | DATE NOT NULL | |
| `released_at` | TIMESTAMPTZ NOT NULL | |

---

## remote-service — `remote_db`

### remote_requests

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `user_id` | UUID NOT NULL | |
| `location_id` | UUID NOT NULL | |
| `request_date` | DATE NOT NULL | |
| `status` | VARCHAR(20) NOT NULL | `PENDING / APPROVED / REJECTED` |
| `recurring_schedule_id` | UUID | Parent schedule (if recurring) |
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
| `team_manager_id` | UUID | NULL = location-level default |
| `max_days_per_week` | INT NOT NULL | |
| `team_overlap_threshold_pct` | INT | % of team that can be remote simultaneously |
| `policy_type` | VARCHAR(20) NOT NULL | `HARD_BLOCK / SOFT_WARNING` |
| `updated_at` | TIMESTAMPTZ | |

---

## audit-service — `audit_db`

### audit_logs

**Append-only. No `deleted_at`. DB user has INSERT privilege only — no UPDATE, no DELETE.**

| Column | Type | Notes |
|--------|------|-------|
| `id` | UUID PK | |
| `actor_id` | UUID NOT NULL | User performing the action |
| `actor_role` | VARCHAR(50) NOT NULL | Role at time of action |
| `action` | VARCHAR(100) NOT NULL | e.g. `SEAT_BOOKED`, `ATTENDANCE_OVERRIDDEN` |
| `entity_type` | VARCHAR(100) NOT NULL | e.g. `SeatBooking`, `AttendanceRecord` |
| `entity_id` | UUID NOT NULL | |
| `location_id` | UUID | |
| `previous_state` | JSONB | State before mutation (NULL for creates) |
| `new_state` | JSONB | State after mutation (NULL for deletes) |
| `occurred_at` | TIMESTAMPTZ NOT NULL | |
| `correlation_id` | UUID | Trace ID from originating HTTP request |

**Indexes:**
```sql
CREATE INDEX idx_audit_log_actor      ON audit_logs(actor_id, occurred_at);
CREATE INDEX idx_audit_log_entity     ON audit_logs(entity_type, entity_id);
CREATE INDEX idx_audit_log_location   ON audit_logs(location_id, occurred_at);
```

### processed_audit_events

Idempotency table for the SQS / Kafka consumer.

| Column | Type | Notes |
|--------|------|-------|
| `event_id` | UUID PK | Deduplication key |
| `processed_at` | TIMESTAMPTZ NOT NULL | |

---

## Key Database Indexes

The following indexes are critical for query performance and must be present in every service's migrations.

```sql
-- All feature tables — location-scoped lookups (present in all services)
CREATE INDEX idx_{table}_location ON {table}(location_id) WHERE deleted_at IS NULL;

-- attendance-service — most queried patterns
CREATE INDEX idx_attendance_record_user_date
  ON attendance_records(user_id, record_date, location_id);

CREATE INDEX idx_badge_event_user_occurred
  ON badge_events(user_id, occurred_at, location_id);

-- seating-service — real-time availability query (hottest path)
CREATE INDEX idx_seat_booking_date_location
  ON seat_bookings(booking_date, location_id) WHERE deleted_at IS NULL;

-- audit-service — query patterns
CREATE INDEX idx_audit_log_actor    ON audit_logs(actor_id, occurred_at);
CREATE INDEX idx_audit_log_entity   ON audit_logs(entity_type, entity_id);
CREATE INDEX idx_audit_log_location ON audit_logs(location_id, occurred_at);

-- user-service — role lookups
CREATE INDEX idx_user_roles_user ON user_roles(user_id) WHERE deleted_at IS NULL;
```

---

## Entity Ownership Summary

No service accesses another service's database. Cross-service data references use UUID values only.

| Service | Owned Tables |
|---------|-------------|
| `identity-service` | sessions |
| `user-service` | users, user_roles, locations, location_configs, public_holidays |
| `attendance-service` | badge_events, work_sessions, attendance_records, badge_sync_job_logs |
| `seating-service` | floors, zones, seats, seat_bookings, block_reservations, no_show_records |
| `remote-service` | remote_requests, recurring_remote_schedules, ooo_requests, approval_delegates, remote_day_policies |
| `notification-service` | notifications, notification_templates |
| `audit-service` | audit_logs, processed_audit_events |
| `visitor-service` | visitor_profiles, parent_visits, visit_records, agreement_templates |
| `event-service` | event_series, events, event_invites |
| `supplies-service` | supply_categories, supply_items, supply_stock_entries, supply_requests, reorder_logs |
| `assets-service` | asset_categories, assets, asset_assignments, asset_requests, maintenance_records, fault_reports |
