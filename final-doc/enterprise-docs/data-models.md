# Data Models
## Office Management System (OMS)

**Version:** 3.0 | Database: PostgreSQL 16 (AWS RDS) | One instance per service

---

## 1. Schema Conventions (All Services)

Every table follows these conventions:

```sql
-- Every entity
id             UUID PRIMARY KEY DEFAULT gen_random_uuid()
created_at     TIMESTAMPTZ NOT NULL DEFAULT now()
updated_at     TIMESTAMPTZ NOT NULL DEFAULT now()
created_by     UUID NOT NULL              -- User ID from session context

-- Every feature entity (not auth-service globals)
location_id    UUID NOT NULL              -- Location-scoped data isolation

-- Mutable entities (not audit, badge events, etc.)
deleted_at     TIMESTAMPTZ                -- Soft delete: NULL = active
```

Auto-update `updated_at` via trigger:
```sql
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN NEW.updated_at = now(); RETURN NEW; END;
$$ language 'plpgsql';
```

---

## 2. auth-service (`auth_db`)

### sessions
```sql
CREATE TABLE sessions (
    id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id        UUID NOT NULL,
    session_token  VARCHAR(255) NOT NULL UNIQUE,
    expires_at     TIMESTAMPTZ NOT NULL,
    created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    ip_address     INET,
    user_agent     TEXT
);
CREATE INDEX idx_sessions_token ON sessions(session_token) WHERE expires_at > now();
CREATE INDEX idx_sessions_user ON sessions(user_id);
```

### users
```sql
CREATE TABLE users (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    sso_sub           VARCHAR(255) NOT NULL UNIQUE,  -- SSO provider subject claim
    email             VARCHAR(255) NOT NULL UNIQUE,
    display_name      VARCHAR(255) NOT NULL,
    arms_employee_id  VARCHAR(100),                   -- Arms HR system ID
    employment_start  DATE NOT NULL,
    employment_end    DATE,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at        TIMESTAMPTZ
);
CREATE UNIQUE INDEX idx_users_sso ON users(sso_sub) WHERE deleted_at IS NULL;
```

### user_roles
```sql
CREATE TABLE user_roles (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id     UUID NOT NULL REFERENCES users(id),
    role        VARCHAR(50) NOT NULL,  -- EMPLOYEE | MANAGER | HR | FACILITIES_ADMIN | SUPER_ADMIN
    location_id UUID,                  -- NULL = SUPER_ADMIN (cross-location)
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_by  UUID NOT NULL,
    deleted_at  TIMESTAMPTZ
);
CREATE UNIQUE INDEX idx_user_roles_unique
    ON user_roles(user_id, role, location_id) WHERE deleted_at IS NULL;
```

### locations + location_configs
```sql
CREATE TABLE locations (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name       VARCHAR(255) NOT NULL UNIQUE,
    city       VARCHAR(100) NOT NULL,
    country    VARCHAR(100) NOT NULL,
    timezone   VARCHAR(100) NOT NULL,  -- e.g., 'Africa/Accra'
    created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE location_configs (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id           UUID NOT NULL UNIQUE REFERENCES locations(id),
    booking_window_days   INTEGER NOT NULL DEFAULT 14,
    cancellation_cutoff_h INTEGER NOT NULL DEFAULT 2,
    no_show_release_time  TIME NOT NULL DEFAULT '10:00',
    late_threshold_mins   INTEGER NOT NULL DEFAULT 30,
    remote_weekly_limit   INTEGER NOT NULL DEFAULT 3,
    updated_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_by            UUID NOT NULL
);
```

---

## 3. attendance-service (`attendance_db`)

### badge_events
```sql
CREATE TABLE badge_events (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id      UUID NOT NULL,
    location_id  UUID NOT NULL,
    occurred_at  TIMESTAMPTZ NOT NULL,
    event_type   VARCHAR(20) NOT NULL,  -- BADGE_IN | BADGE_OUT
    raw_payload  JSONB,                 -- original Athena row preserved
    sync_job_id  UUID NOT NULL,         -- which sync job ingested this
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
    -- No deleted_at — badge events are IMMUTABLE after ingestion
);
CREATE INDEX idx_badge_event_user_occurred ON badge_events(user_id, occurred_at, location_id);
CREATE INDEX idx_badge_event_sync ON badge_events(sync_job_id);
```

### work_sessions
```sql
CREATE TABLE work_sessions (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id                 UUID NOT NULL,
    location_id             UUID NOT NULL,
    session_date            DATE NOT NULL,
    first_badge_in          TIMESTAMPTZ NOT NULL,
    last_badge_out          TIMESTAMPTZ,
    total_duration_minutes  INTEGER,
    is_late                 BOOLEAN NOT NULL DEFAULT false,
    minutes_late            INTEGER NOT NULL DEFAULT 0,
    crosses_midnight        BOOLEAN NOT NULL DEFAULT false,
    created_at              TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX idx_work_session_user_date ON work_sessions(user_id, session_date, location_id);
```

### attendance_records
```sql
CREATE TABLE attendance_records (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id      UUID NOT NULL,
    location_id  UUID NOT NULL,
    record_date  DATE NOT NULL,
    status       VARCHAR(30) NOT NULL,  -- PRESENT|LATE|ABSENT|REMOTE|OOO|PUBLIC_HOLIDAY
    override_by  UUID,                  -- Super Admin who overrode, if any
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX idx_attendance_record_user_date
    ON attendance_records(user_id, record_date, location_id);
CREATE INDEX idx_attendance_record_location_date
    ON attendance_records(location_id, record_date, status);
```

---

## 4. seating-service (`seating_db`)

### floors, zones, seats
```sql
CREATE TABLE floors (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    location_id UUID NOT NULL,
    name        VARCHAR(100) NOT NULL,
    floor_number INTEGER NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_by  UUID NOT NULL,
    deleted_at  TIMESTAMPTZ
);

CREATE TABLE zones (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    floor_id   UUID NOT NULL REFERENCES floors(id),
    location_id UUID NOT NULL,
    name       VARCHAR(100) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at TIMESTAMPTZ
);

CREATE TABLE seats (
    id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    zone_id        UUID NOT NULL REFERENCES zones(id),
    location_id    UUID NOT NULL,
    label          VARCHAR(50) NOT NULL,     -- e.g., "A-204"
    seat_type      VARCHAR(30) NOT NULL,     -- HOT_DESK | PERMANENT | MEETING
    is_visible     BOOLEAN NOT NULL DEFAULT true,
    permanent_user UUID,                     -- NULL if hot-desk
    created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at     TIMESTAMPTZ
);
```

### seat_bookings
```sql
CREATE TABLE seat_bookings (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    seat_id      UUID NOT NULL REFERENCES seats(id),
    user_id      UUID NOT NULL,
    location_id  UUID NOT NULL,
    booking_date DATE NOT NULL,
    status       VARCHAR(20) NOT NULL DEFAULT 'CONFIRMED',  -- CONFIRMED|CANCELLED|RELEASED
    cancelled_at TIMESTAMPTZ,
    released_at  TIMESTAMPTZ,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_by   UUID NOT NULL,
    deleted_at   TIMESTAMPTZ
);
-- Real-time availability query index
CREATE INDEX idx_seat_booking_date_location
    ON seat_bookings(booking_date, location_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_seat_booking_user
    ON seat_bookings(user_id, booking_date) WHERE deleted_at IS NULL;
```

---

## 5. remote-service (`remote_db`)

### remote_requests
```sql
CREATE TABLE remote_requests (
    id                        UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id                   UUID NOT NULL,
    location_id               UUID NOT NULL,
    request_date              DATE NOT NULL,
    status                    VARCHAR(30) NOT NULL,  -- PENDING|APPROVED|REJECTED|CANCELLED
    reason                    TEXT,
    approver_id               UUID,
    rejection_reason          TEXT,
    recurring_schedule_id     UUID,
    created_at                TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at                TIMESTAMPTZ NOT NULL DEFAULT now(),
    created_by                UUID NOT NULL,
    deleted_at                TIMESTAMPTZ
);
CREATE INDEX idx_remote_request_user_date ON remote_requests(user_id, request_date, location_id);
```

### ooo_requests
```sql
CREATE TABLE ooo_requests (
    id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id       UUID NOT NULL,
    location_id   UUID NOT NULL,
    start_date    DATE NOT NULL,
    end_date      DATE NOT NULL,
    reason        TEXT,
    status        VARCHAR(20) NOT NULL DEFAULT 'APPROVED',
    created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    deleted_at    TIMESTAMPTZ
);
```

### remote_day_policies
```sql
CREATE TABLE remote_day_policies (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scope_type            VARCHAR(20) NOT NULL,  -- LOCATION | TEAM
    scope_id              UUID NOT NULL,          -- locationId or managerId
    weekly_limit          INTEGER NOT NULL DEFAULT 3,
    team_overlap_threshold INTEGER NOT NULL DEFAULT 50,  -- percentage
    enforcement           VARCHAR(20) NOT NULL,  -- HARD_BLOCK | SOFT_WARNING
    updated_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_by            UUID NOT NULL
);
```

---

## 6. notification-service (`notification_db`)

### notifications
```sql
CREATE TABLE notifications (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id      UUID NOT NULL,
    location_id  UUID NOT NULL,
    type         VARCHAR(50) NOT NULL,   -- e.g., REMOTE_REQUEST_APPROVED
    title        VARCHAR(255) NOT NULL,
    body         TEXT NOT NULL,
    is_read      BOOLEAN NOT NULL DEFAULT false,
    read_at      TIMESTAMPTZ,
    source_event_id UUID,               -- eventId from RabbitMQ message
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_notification_user ON notifications(user_id, is_read, created_at DESC);
```

---

## 7. audit-service (`audit_db`)

### audit_logs
```sql
CREATE TABLE audit_logs (
    id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_id       UUID NOT NULL UNIQUE,   -- from RabbitMQ event envelope
    actor_id       UUID NOT NULL,
    actor_role     VARCHAR(50) NOT NULL,
    action         VARCHAR(100) NOT NULL,  -- e.g., SEAT_BOOKED, USER_DEACTIVATED
    entity_type    VARCHAR(100) NOT NULL,  -- e.g., SeatBooking, User
    entity_id      UUID NOT NULL,
    location_id    UUID,
    previous_state JSONB,
    new_state      JSONB,
    correlation_id UUID,
    occurred_at    TIMESTAMPTZ NOT NULL
    -- No created_at, deleted_at, updated_at — this is append-only
    -- No PRIMARY KEY constraint outside event_id — no UPDATE allowed at DB level
);
-- DB user: INSERT only — no UPDATE, no DELETE granted

CREATE INDEX idx_audit_actor ON audit_logs(actor_id, occurred_at DESC);
CREATE INDEX idx_audit_entity ON audit_logs(entity_type, entity_id, occurred_at DESC);
CREATE INDEX idx_audit_location ON audit_logs(location_id, occurred_at DESC);
CREATE INDEX idx_audit_action ON audit_logs(action, occurred_at DESC);
```

---

## 8. Key Indexes Summary

| Table | Index | Purpose |
|---|---|---|
| `attendance_records` | `(user_id, record_date, location_id)` | Employee's own attendance lookup |
| `attendance_records` | `(location_id, record_date, status)` | HR location-wide reporting |
| `seat_bookings` | `(booking_date, location_id)` WHERE active | Real-time availability query — most critical |
| `seat_bookings` | `(user_id, booking_date)` WHERE active | Employee's own bookings |
| `badge_events` | `(user_id, occurred_at, location_id)` | WorkSession resolution |
| `remote_requests` | `(user_id, request_date, location_id)` | Employee's request history |
| `audit_logs` | `(entity_type, entity_id, occurred_at)` | Entity audit trail |
| `audit_logs` | `(actor_id, occurred_at)` | Actor activity |
| `notifications` | `(user_id, is_read, created_at)` | Unread notifications fetch |
| `sessions` | `(session_token)` WHERE not expired | Gateway validates on every request |

---

## 9. Flyway Migration Conventions

```
src/main/resources/db/migration/
  V1__create_schema.sql
  V2__add_policy_enforcement_column.sql
  V3__add_booking_block_table.sql

Rules:
- Never modify an existing migration file after it has run in any environment
- Always create a new version to change schema
- Migrations are run by Flyway at service startup
- Migration DB user has broader privileges; revoked after startup
```
