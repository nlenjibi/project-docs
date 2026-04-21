%% ============================================================
%% dataengineer-backend-data-models.mmd
%% OMS — Microservice Data Models  (version 3.0 · 8 services)
%%
%% Each service owns its own database. Cross-service references
%% are carried as plain UUID columns (no enforced FK) — resolved
%% at runtime via service APIs or event-driven sync.
%%
%% Services:
%%   1. auth-service          → auth_db
%%   2. attendance-service    → attendance_db   (owns EMPLOYEE)
%%   3. seating-service       → seating_db
%%   4. remote-service        → remote_db
%%   5. notification-service  → notification_db
%%   6. audit-service         → audit_db
%%   7. inventory-service     → inventory_db
%%   8. workplace-service     → workplace_db
%% ============================================================


```mermaid 
erDiagram

  %% ══════════════════════════════════════════════
  %% SERVICE 1 — auth-service (auth_db)
  %% ══════════════════════════════════════════════

  SESSIONS {
    string  primary_id      PK  "spring-session-jdbc"
    string  session_id          "HTTP cookie key (UNIQUE)"
    bigint  creation_time
    bigint  last_access_time
    int     max_inactive_interval
    bigint  expiry_time
    string  principal_name      "SSO sub claim"
  }

  USERS {
    uuid    id              PK
    string  sso_sub             "UNIQUE NOT NULL — SSO subject claim"
    string  email               "UNIQUE NOT NULL"
    string  display_name
    string  personnel_id        "Arms HR ID for badge sync"
    date    employment_start_date
    date    employment_end_date
    bool    is_active
    timestamp created_at
    timestamp updated_at
  }

  USER_ROLES {
    uuid    id              PK
    uuid    user_id         FK
    string  role                "EMPLOYEE | MANAGER | HR | FACILITIES_ADMIN | SUPER_ADMIN"
    uuid    location_id         "NULL for SUPER_ADMIN only"
    timestamp created_at
    timestamp deleted_at        "Soft-delete on role revoke"
  }

  OFFICES {
    uuid    id              PK
    string  name                "e.g. Accra HQ"
    string  country
    string  timezone            "IANA tz e.g. Africa/Accra"
    bool    is_active
    timestamp created_at
  }

  LOCATION_CONFIGS {
    uuid    id              PK
    uuid    location_id     FK  "UNIQUE — one config per location"
    int     booking_window_days
    int     cancellation_cutoff_hours
    time    no_show_release_time
    time    attendance_cutoff_time
    int     max_remote_days_per_week
    int     cross_midnight_gap_hours
    timestamp updated_at
  }

  PUBLIC_HOLIDAYS {
    uuid    id              PK
    uuid    location_id     FK
    date    holiday_date
    string  name
  }

  USERS         ||--o{ USER_ROLES        : holds
  LOCATIONS     ||--o{ USER_ROLES        : scopes
  LOCATIONS     ||--||  LOCATION_CONFIGS : configured_by
  LOCATIONS     ||--o{ PUBLIC_HOLIDAYS   : defines


  %% ══════════════════════════════════════════════
  %% SERVICE 2 — attendance-service (attendance_db)
  %% Note: EMPLOYEE record lives here per backend spec
  %% ══════════════════════════════════════════════

  EMPLOYEES {
    uuid    id              PK
    uuid    user_id             "Ref → auth_db.users.id (cross-service, no FK)"
    uuid    location_id         "Primary building location"
    uuid    manager_id          "Self-ref within this service"
    string  employee_code
    string  first_name
    string  last_name
    string  email
    string  department
    string  job_title
    string  project
    string  language_preference
    string  employee_type       "FULL_TIME | PART_TIME | CONTRACTOR | INTERN"
    date    employment_start_date
    date    employment_end_date
    bool    is_active
    timestamp created_at
    timestamp updated_at
    timestamp deleted_at
  }

  BADGE_EVENTS {
    uuid    id              PK
    uuid    user_id             "NULL if personnel_id unresolved"
    uuid    location_id
    string  personnel_id        "Raw badge reader value (NOT NULL)"
    string  event_type          "BADGE_IN | BADGE_OUT"
    timestamp occurred_at
    jsonb   raw_payload         "Full Athena row — for re-processing"
    timestamp ingested_at
  }

  WORK_SESSIONS {
    uuid    id              PK
    uuid    user_id
    uuid    location_id
    date    session_date
    timestamp first_badge_in
    timestamp last_badge_out    "NULL if still in building"
    int     total_duration_minutes
    bool    is_late
    int     minutes_late
    bool    crosses_midnight
    timestamp created_at
    timestamp updated_at
  }

  ATTENDANCE_RECORDS {
    uuid    id              PK
    uuid    user_id
    uuid    location_id
    date    record_date
    string  status              "PRESENT | LATE | ABSENT | REMOTE | OOO | PUBLIC_HOLIDAY"
    uuid    work_session_id FK
    uuid    override_by         "SUPER_ADMIN user_id if manually overridden"
    string  override_reason
    timestamp created_at
    timestamp updated_at
  }

  BADGE_SYNC_JOB_LOGS {
    uuid    id              PK
    uuid    location_id
    date    sync_date
    timestamp started_at
    timestamp completed_at
    int     records_queried
    int     records_inserted
    int     records_skipped
    int     records_failed
    string  status              "RUNNING | SUCCESS | PARTIAL | FAILED"
    string  error_message
  }

  PERSONNEL_ID_MAPPING {
    string  personnel_id    PK  "Arms HR ID"
    uuid    user_id             "OMS UUID"
    timestamp synced_at
  }

  EMPLOYEES             ||--o{ ATTENDANCE_RECORDS    : has
  WORK_SESSIONS         ||--o|  ATTENDANCE_RECORDS   : populates
  EMPLOYEES             ||--o{ WORK_SESSIONS         : has
  EMPLOYEES             ||--o{ BADGE_EVENTS          : generates


  %% ══════════════════════════════════════════════
  %% SERVICE 3 — seating-service (seating_db)
  %% ══════════════════════════════════════════════

  FLOORS {
    uuid    id              PK
    uuid    location_id
    int     floor_number
    string  name
    timestamp deleted_at
  }

  ZONES {
    uuid    id              PK
    uuid    floor_id        FK
    uuid    location_id         "Denormalised for query performance"
    string  name                "e.g. East Wing, Quiet Zone"
    timestamp deleted_at
  }

  SEATS {
    uuid    id              PK
    uuid    zone_id         FK
    uuid    location_id         "Denormalised for query performance"
    string  seat_label          "e.g. E-14, Q-03"
    string  seat_type           "HOT_DESK | PERMANENT | BLOCKED"
    uuid    permanent_user_id   "Set when seat_type = PERMANENT"
    bool    is_bookable
    timestamp deleted_at
  }

  SEAT_BOOKINGS {
    uuid    id              PK
    uuid    seat_id         FK
    uuid    user_id             "Ref → auth_db.users.id"
    uuid    location_id
    date    booking_date
    string  status              "CONFIRMED | CANCELLED | RELEASED"
    uuid    block_reservation_id
    uuid    created_by
    timestamp created_at
    timestamp deleted_at
    bigint  version             "Optimistic lock — prevents double-booking"
  }

  BLOCK_RESERVATIONS {
    uuid    id              PK
    uuid    manager_user_id     "Ref → auth_db.users.id"
    uuid    location_id
    uuid    zone_id         FK
    date    reservation_date
    int     seat_count
    string  notes
    string  status              "ACTIVE | CANCELLED | FULFILLED"
    timestamp created_at
    timestamp updated_at
  }

  NO_SHOW_RECORDS {
    uuid    id              PK
    uuid    seat_booking_id FK
    uuid    user_id
    uuid    location_id
    date    booking_date
    timestamp released_at
  }

  FLOOR_LEADS {
    uuid    id              PK
    uuid    user_id             "Ref → auth_db.users.id"
    uuid    zone_id         FK
    uuid    location_id
    date    assigned_from
    date    assigned_to
    timestamp created_at
    timestamp updated_at
  }

  FLOORS            ||--|{ ZONES              : contains
  ZONES             ||--|{ SEATS              : has
  SEATS             ||--o{ SEAT_BOOKINGS      : booked_as
  BLOCK_RESERVATIONS ||--o{ SEAT_BOOKINGS     : generates
  SEAT_BOOKINGS     ||--o| NO_SHOW_RECORDS    : triggers
  ZONES             ||--o{ FLOOR_LEADS        : led_by


  %% ══════════════════════════════════════════════
  %% SERVICE 4 — remote-service (remote_db)
  %% ══════════════════════════════════════════════

  REMOTE_REQUESTS {
    uuid    id              PK
    uuid    user_id             "Ref → auth_db.users.id"
    uuid    location_id
    date    request_date
    string  status              "PENDING | APPROVED | REJECTED"
    uuid    recurring_schedule_id FK
    uuid    approved_by
    string  rejection_reason
    timestamp created_at
    timestamp deleted_at
  }

  RECURRING_REMOTE_SCHEDULES {
    uuid    id              PK
    uuid    user_id
    uuid    location_id
    string  day_of_week         "monday … sunday"
    date    effective_from
    date    effective_to
    string  status              "PENDING | APPROVED | REJECTED"
    uuid    reviewed_by
    timestamp reviewed_at
    string  rejection_reason
    timestamp created_at
    timestamp updated_at
  }

  OOO_REQUESTS {
    uuid    id              PK
    uuid    user_id
    uuid    location_id
    string  ooo_type            "ANNUAL_LEAVE | SICK_LEAVE | MATERNITY | etc."
    date    start_date
    date    end_date
    string  status              "PENDING | APPROVED | REJECTED"
    uuid    approved_by
    string  rejection_reason
    string  notes
    timestamp created_at
    timestamp updated_at
  }

  APPROVAL_DELEGATES {
    uuid    id              PK
    uuid    manager_id
    uuid    delegate_id
    uuid    location_id
    date    start_date
    date    end_date
    bool    is_active
    timestamp created_at
  }

  REMOTE_DAY_POLICIES {
    uuid    id              PK
    uuid    location_id
    uuid    team_manager_id     "NULL = location-level default"
    int     max_days_per_week
    int     team_overlap_threshold_pct
    string  policy_type         "HARD_BLOCK | SOFT_WARNING"
    timestamp updated_at
  }

  RECURRING_REMOTE_SCHEDULES ||--o{ REMOTE_REQUESTS : generates


  %% ══════════════════════════════════════════════
  %% SERVICE 5 — notification-service (notification_db)
  %% ══════════════════════════════════════════════

  NOTIFICATIONS {
    uuid    id              PK
    uuid    user_id             "Recipient — ref → auth_db.users.id"
    uuid    location_id
    string  type                "REMOTE_APPROVED | SEAT_BOOKING_CONFIRMED | etc."
    string  message
    bool    is_read
    timestamp created_at
    timestamp read_at
  }


  %% ══════════════════════════════════════════════
  %% SERVICE 6 — audit-service (audit_db)
  %% Append-only — INSERT privilege only
  %% ══════════════════════════════════════════════

  AUDIT_LOGS {
    uuid    id              PK
    uuid    actor_id            "User who performed the action"
    string  actor_role          "Role captured at time of action"
    string  action              "SEAT_BOOKED | ATTENDANCE_OVERRIDDEN | etc."
    string  entity_type         "SeatBooking | AttendanceRecord | etc."
    uuid    entity_id
    uuid    location_id
    jsonb   previous_state      "Full entity before mutation"
    jsonb   new_state           "Full entity after mutation"
    timestamp occurred_at
    uuid    correlation_id      "HTTP request trace ID"
  }

  PROCESSED_EVENTS {
    uuid    event_id        PK  "Deduplication key"
    timestamp processed_at
  }


  %% ══════════════════════════════════════════════
  %% SERVICE 7 — inventory-service (inventory_db)
  %% ══════════════════════════════════════════════

  SUPPLY_CATEGORIES {
    uuid    id              PK
    uuid    location_id
    string  name                "e.g. Stationery, IT Peripherals"
  }

  SUPPLY_ITEMS {
    uuid    id              PK
    uuid    category_id     FK
    uuid    location_id
    string  name
    string  unit                "box | piece | ream"
    int     reorder_threshold
    bool    is_active
    timestamp created_at
    timestamp updated_at
  }

  SUPPLY_STOCK_ENTRIES {
    uuid    id              PK
    uuid    item_id         FK
    uuid    location_id
    int     quantity_received
    int     quantity_remaining
    date    expiry_date
    date    received_at
    timestamp created_at
    timestamp written_off_at
  }

  SUPPLY_REQUESTS {
    uuid    id              PK
    uuid    user_id             "Requester"
    uuid    location_id
    string  status              "PENDING_MANAGER | PENDING_FACILITIES | FULFILLED | REJECTED"
    string  justification
    uuid    approved_by_manager
    uuid    approved_by_facilities
    string  rejection_reason
    timestamp created_at
    timestamp fulfilled_at
    timestamp deleted_at
  }

  SUPPLY_REQUEST_ITEMS {
    uuid    id              PK
    uuid    request_id      FK
    uuid    item_id         FK
    int     quantity_requested
    int     quantity_fulfilled
  }

  ASSET_CATEGORIES {
    uuid    id              PK
    uuid    location_id
    string  name                "e.g. Laptop, Monitor, Access Card"
  }

  ASSETS {
    uuid    id              PK
    uuid    category_id     FK
    uuid    location_id
    string  serial_number       "UNIQUE NOT NULL"
    string  description
    string  status              "AVAILABLE | ASSIGNED | UNDER_MAINTENANCE | RETIRED"
    date    purchase_date
    decimal purchase_value
    timestamp created_at
    timestamp deleted_at
  }

  ASSET_ASSIGNMENTS {
    uuid    id              PK
    uuid    asset_id        FK
    uuid    user_id             "Assigned user"
    uuid    location_id
    string  status              "ASSIGNED | ACKNOWLEDGED | RETURNED | RETIRED"
    uuid    assigned_by
    timestamp assigned_at
    timestamp acknowledged_at
    timestamp returned_at
    string  notes
  }

  ASSET_REQUESTS {
    uuid    id              PK
    uuid    requester_id
    uuid    category_id     FK
    uuid    location_id
    string  justification
    string  status              "PENDING_MANAGER | PENDING_FACILITIES | FULFILLED | REJECTED"
    uuid    manager_reviewed_by
    timestamp manager_reviewed_at
    uuid    facilities_reviewed_by
    timestamp facilities_reviewed_at
    string  rejection_reason
    timestamp fulfilled_at
    timestamp created_at
    timestamp updated_at
  }

  FAULT_REPORTS {
    uuid    id              PK
    uuid    asset_id        FK
    uuid    reported_by
    uuid    location_id
    string  description
    string  severity            "LOW | MEDIUM | HIGH | CRITICAL"
    string  status              "OPEN | IN_PROGRESS | RESOLVED"
    timestamp created_at
    timestamp resolved_at
  }

  MAINTENANCE_RECORDS {
    uuid    id              PK
    uuid    asset_id        FK
    uuid    location_id
    string  type                "REPAIR | SCHEDULED_SERVICE | INSPECTION | UPGRADE"
    string  description
    uuid    performed_by
    date    performed_at
    uuid    fault_report_id FK
    timestamp created_at
  }

  SUPPLY_CATEGORIES   ||--o{ SUPPLY_ITEMS         : categorises
  SUPPLY_ITEMS        ||--o{ SUPPLY_STOCK_ENTRIES  : tracked_in
  SUPPLY_ITEMS        ||--o{ SUPPLY_REQUEST_ITEMS  : included_in
  SUPPLY_REQUESTS     ||--|{ SUPPLY_REQUEST_ITEMS  : contains
  ASSET_CATEGORIES    ||--o{ ASSETS               : categorises
  ASSET_CATEGORIES    ||--o{ ASSET_REQUESTS        : for_category
  ASSETS              ||--o{ ASSET_ASSIGNMENTS     : assigned_via
  ASSET_REQUESTS      ||--o| ASSET_ASSIGNMENTS     : originated_from
  ASSETS              ||--o{ FAULT_REPORTS         : reported_for
  FAULT_REPORTS       ||--o{ MAINTENANCE_RECORDS   : spawns
  ASSETS              ||--o{ MAINTENANCE_RECORDS   : has


  %% ══════════════════════════════════════════════
  %% SERVICE 8 — workplace-service (workplace_db)
  %% ══════════════════════════════════════════════

  VISITOR_PROFILES {
    uuid    id              PK
    string  display_name
    string  email               "UNIQUE NOT NULL — natural key"
    string  phone
    string  company
    uuid    created_by          "Ref → auth_db.users.id"
    timestamp created_at
    timestamp updated_at
    timestamp deleted_at
  }

  AGREEMENT_TEMPLATES {
    uuid    id              PK
    uuid    location_id
    string  title
    string  content
    bool    is_active
    uuid    created_by
    timestamp created_at
    timestamp updated_at
  }

  PARENT_VISITS {
    uuid    id              PK
    uuid    visitor_profile_id  FK
    uuid    host_user_id        "Ref → auth_db.users.id"
    uuid    location_id
    string  purpose
    date    start_date
    date    end_date
    string  access_level        "ESCORTED | UNESCORTED | RESTRICTED"
    string  notes
    timestamp created_at
    timestamp updated_at
  }

  VISIT_RECORDS {
    uuid    id              PK
    uuid    parent_visit_id     FK
    uuid    visitor_profile_id  FK
    uuid    host_user_id
    uuid    location_id
    date    visit_date
    time    expected_arrival_time
    timestamp actual_check_in
    timestamp actual_check_out
    string  status              "SCHEDULED | CHECKED_IN | CHECKED_OUT | NO_SHOW | CANCELLED"
    bool    is_walk_in
    bool    agreement_signed
    timestamp agreement_signed_at
    uuid    agreement_template_id FK
    uuid    checked_in_by
    timestamp created_at
    timestamp updated_at
  }

  EVENT_SERIES {
    uuid    id              PK
    uuid    location_id
    uuid    organiser_id        "Ref → auth_db.users.id"
    string  title
    string  description
    string  recurrence_type     "NONE | DAILY | WEEKLY | MONTHLY"
    int     recurrence_interval
    string  recurrence_day
    date    series_start_date
    date    series_end_date
    timestamp created_at
    timestamp updated_at
  }

  EVENTS {
    uuid    id              PK
    uuid    event_series_id FK
    uuid    location_id
    uuid    organiser_id
    string  title
    string  description
    date    event_date
    time    start_time
    time    end_time
    string  room_reference
    int     capacity
    string  status              "SCHEDULED | CANCELLED | COMPLETED"
    string  cancellation_reason
    timestamp cancelled_at
    bool    is_series_detached
    timestamp created_at
    timestamp updated_at
  }

  EVENT_INVITES {
    uuid    id              PK
    uuid    event_id        FK
    uuid    user_id             "NULL for external attendees"
    uuid    visit_record_id FK  "NULL for internal attendees"
    string  rsvp_status         "PENDING | ACCEPTED | DECLINED | TENTATIVE"
    timestamp rsvp_at
    bool    is_waitlisted
    int     waitlist_position
    timestamp created_at
    timestamp updated_at
  }

  VISITOR_PROFILES    ||--o{ PARENT_VISITS         : for
  PARENT_VISITS       ||--|{ VISIT_RECORDS          : includes
  VISITOR_PROFILES    ||--o{ VISIT_RECORDS          : for
  AGREEMENT_TEMPLATES ||--o{ VISIT_RECORDS          : signed_under
  EVENT_SERIES        ||--o{ EVENTS                 : generates
  EVENTS              ||--o{ EVENT_INVITES          : has
  VISIT_RECORDS       ||--o{ EVENT_INVITES          : for_external_attendee