```mermaid 

erDiagram

  %% ══════════════════════════════════════════════════════
  %% SECTION 1 — GEOGRAPHIC & LOCATION HIERARCHY
  %% ══════════════════════════════════════════════════════
  ORGANISATION {
    uuid id PK
    uuid office_id FK
    string country_name
    string country_code
    string timezone
    timestamp created_at
    timestamp updated_at
  }

  OFFICE {
    uuid id PK
    uuid organisation_id FK
    uuid building_id FK
    string city_name
    timestamp created_at
    timestamp updated_at
  }

  OFFICE_BUILDING {
    uuid id PK
    uuid office_id FK
    string building_name
    string address
    int total_floors
    bool is_active
    timestamp created_at
    timestamp updated_at
  }

  LOCATION_CONFIG {
    uuid id PK
    uuid building_id FK
    time work_start_time
    int lateness_threshold_minutes
    int min_presence_duration_minutes
    time no_show_release_time
    int hot_desk_booking_window_days
    int booking_cancellation_cutoff_hours
    enum seat_visibility_mode
    int session_gap_threshold_hours
    timestamp created_at
    timestamp updated_at
  }

  FLOOR {
    uuid id PK
    uuid building_id FK
    string floor_name
    int floor_number
    bool is_active
    timestamp created_at
    timestamp updated_at
  }

  ROOM {
    uuid id PK
    uuid floor_id FK
    uuid building_id FK
    string room_name
    int total_seats
    enum room_type
    bool is_active
    timestamp created_at
    timestamp updated_at
  }

  SEAT {
    uuid id PK
    uuid room_id FK
    uuid floor_id FK
    uuid building_id FK
    string seat_number
    enum seat_type
    enum status
    uuid assigned_user_id FK
    float x_position
    float y_position
    timestamp created_at
    timestamp updated_at
    timestamp deleted_at
  }

  %% ══════════════════════════════════════════════════════
  %% SECTION 2 — IDENTITY: USER & EMPLOYEE
  %% ══════════════════════════════════════════════════════

  USER {
    uuid id PK
    uuid employee_id FK
    string first_name
    string last_name
    string email
    string sso_user_id
    string phone
    uuid password 
    bool is_active
    timestamp created_at
    timestamp updated_at
    timestamp deleted_at
  }

  EMPLOYEE {
    uuid id PK
    uuid user_id FK
    uuid primary_building_id FK
    string language_preference
    string arms_emp_id
    string employee_code
    string department
    string job_title
    string project
    date employment_start_date
    date employment_end_date
    timestamp arms_synced_at
    timestamp created_at
    timestamp updated_at
  }

  USER_ROLE {
    uuid id PK
    uuid user_id FK
    uuid building_id FK
    enum role
    timestamp assigned_at
    uuid assigned_by FK
    timestamp created_at
    timestamp updated_at
  }

  %% ══════════════════════════════════════════════════════
  %% SECTION 3 — BADGE EVENTS & ATTENDANCE
  %% ══════════════════════════════════════════════════════

  BADGE_EVENT {
    uuid id PK
    uuid user_id FK
    uuid building_id FK
    string personnel_id
    enum event_type
    timestamp occurred_at
    jsonb raw_payload
    timestamp ingested_at
  }

  WORK_SESSION {
    uuid id PK
    uuid user_id FK
    uuid building_id FK
    date session_date
    timestamp first_badge_in
    timestamp last_badge_out
    int total_duration_minutes
    bool is_late
    int session_split_count
    int session_split_index
    int minutes_late
    bool crosses_midnight
    timestamp created_at
    timestamp updated_at
  }

  ATTENDANCE_RECORD {
    uuid id PK
    uuid user_id FK
    uuid building_id FK
    date record_date
    enum status
    uuid work_session_id FK
    uuid remote_request_id FK
    uuid ooo_request_id FK
    bool is_overridden
    uuid overridden_by FK
    string override_reason
    timestamp created_at
    timestamp updated_at
  }

  PUBLIC_HOLIDAY {
    uuid id PK
    uuid building_id FK
    date holiday_date
    string name
    timestamp created_at
    timestamp updated_at
  }

  %% ══════════════════════════════════════════════════════
  %% SECTION 4 — SEATING MANAGEMENT
  %% ══════════════════════════════════════════════════════

  WORK_SCHEDULE {
    uuid id PK
    uuid user_id FK
    uuid seat_id FK
    enum day_of_week
    bool is_remote
    date effective_from
    date effective_to
    timestamp created_at
    timestamp updated_at
  }

  SEAT_BOOKING {
    uuid id PK
    uuid seat_id FK
    uuid user_id FK
    uuid building_id FK
    date booking_date
    enum status
    uuid block_reservation_id FK
    timestamp cancelled_at
    string cancellation_reason
    timestamp auto_released_at
    timestamp created_at
    timestamp updated_at
  }

  BLOCK_RESERVATION {
    uuid id PK
    uuid manager_id FK
    uuid building_id FK
    uuid room_id FK
    date reservation_date
    int seat_count
    string notes
    enum status
    timestamp created_at
    timestamp updated_at
  }

  NO_SHOW_RECORD {
    uuid id PK
    uuid seat_booking_id FK
    uuid user_id FK
    uuid building_id FK
    date no_show_date
    timestamp created_at
  }

  SEAT_OCCUPANCY_LOG {
    uuid id PK
    uuid seat_id FK
    uuid user_id FK
    date occupancy_date
    bool is_occupied
    bool is_remote_day
    string notes
    timestamp created_at
  }

  FLOOR_LEAD {
    uuid id PK
    uuid user_id FK
    uuid room_id FK
    date assigned_from
    date assigned_to
    timestamp created_at
    timestamp updated_at
  }

  %% ══════════════════════════════════════════════════════
  %% SECTION 5 — REMOTE DAY & OOO MANAGEMENT
  %% ══════════════════════════════════════════════════════

  REMOTE_REQUEST {
    uuid id PK
    uuid user_id FK
    uuid building_id FK
    date request_date
    enum request_type
    uuid recurring_remote_schedule_id FK
    enum status
    uuid reviewed_by FK
    timestamp reviewed_at
    string rejection_reason
    string notes
    timestamp created_at
    timestamp updated_at
  }

  RECURRING_REMOTE_SCHEDULE {
    uuid id PK
    uuid user_id FK
    uuid building_id FK
    enum day_of_week
    date effective_from
    date effective_to
    enum status
    uuid reviewed_by FK
    timestamp reviewed_at
    string rejection_reason
    timestamp created_at
    timestamp updated_at
  }

  OOO_REQUEST {
    uuid id PK
    uuid user_id FK
    uuid building_id FK
    enum ooo_type
    date start_date
    date end_date
    enum status
    uuid reviewed_by FK
    timestamp reviewed_at
    string rejection_reason
    string notes
    timestamp created_at
    timestamp updated_at
  }

  REMOTE_DAY_POLICY {
    uuid id PK
    uuid building_id FK
    uuid manager_id FK
    int max_remote_days_per_week
    enum enforcement_mode
    timestamp created_at
    timestamp updated_at
  }

  APPROVAL_DELEGATE {
    uuid id PK
    uuid manager_id FK
    uuid delegated_to_user_id FK
    uuid building_id FK
    date effective_from
    date effective_to
    uuid ooo_request_id FK
    bool is_active
    timestamp created_at
    timestamp updated_at
  }

  %% ══════════════════════════════════════════════════════
  %% SECTION 6 — VISITOR MANAGEMENT
  %% ══════════════════════════════════════════════════════

  VISITOR_PROFILE {
    uuid id PK
    string first_name
    string last_name
    string email
    string phone
    string company
    uuid created_by FK
    timestamp created_at
    timestamp updated_at
    timestamp deleted_at
  }

  PARENT_VISIT {
    uuid id PK
    uuid visitor_profile_id FK
    uuid host_user_id FK
    uuid building_id FK
    string purpose
    date start_date
    date end_date
    enum access_level
    string notes
    timestamp created_at
    timestamp updated_at
  }

  VISIT_RECORD {
    uuid id PK
    uuid parent_visit_id FK
    uuid visitor_profile_id FK
    uuid host_user_id FK
    uuid building_id FK
    date visit_date
    time expected_arrival_time
    timestamp actual_check_in
    timestamp actual_check_out
    enum status
    bool is_walk_in
    bool agreement_signed
    timestamp agreement_signed_at
    uuid agreement_template_id FK
    uuid checked_in_by FK
    timestamp created_at
    timestamp updated_at
  }

  AGREEMENT_TEMPLATE {
    uuid id PK
    uuid building_id FK
    string title
    text content
    bool is_active
    uuid created_by FK
    timestamp created_at
    timestamp updated_at
  }

  %% ══════════════════════════════════════════════════════
  %% SECTION 7 — OFFICE EVENT MANAGEMENT
  %% ══════════════════════════════════════════════════════

  EVENT_SERIES {
    uuid id PK
    uuid building_id FK
    uuid organiser_id FK
    string title
    string description
    enum recurrence_type
    int recurrence_interval
    string recurrence_day
    date series_start_date
    date series_end_date
    timestamp created_at
    timestamp updated_at
  }

  EVENT {
    uuid id PK
    uuid event_series_id FK
    uuid building_id FK
    uuid organiser_id FK
    string title
    string description
    date event_date
    time start_time
    time end_time
    string room_reference
    int capacity
    enum status
    string cancellation_reason
    timestamp cancelled_at
    bool is_series_detached
    timestamp created_at
    timestamp updated_at
  }

  EVENT_INVITE {
    uuid id PK
    uuid event_id FK
    uuid user_id FK
    uuid visit_record_id FK
    enum rsvp_status
    timestamp rsvp_at
    bool is_waitlisted
    int waitlist_position
    timestamp waitlist_joined_at
    timestamp created_at
    timestamp updated_at
  }

  %% ══════════════════════════════════════════════════════
  %% SECTION 8 — OFFICE SUPPLIES MANAGEMENT
  %% ══════════════════════════════════════════════════════

  SUPPLY_CATEGORY {
    uuid id PK
    string name
    bool has_expiry
    timestamp created_at
    timestamp updated_at
  }

  SUPPLY_ITEM {
    uuid id PK
    uuid building_id FK
    uuid supply_category_id FK
    string name
    string description
    string unit_of_measure
    enum allocation_type
    decimal current_stock
    decimal low_stock_threshold
    decimal reorder_quantity
    bool is_active
    timestamp created_at
    timestamp updated_at
  }

  SUPPLY_STOCK_ENTRY {
    uuid id PK
    uuid supply_item_id FK
    uuid building_id FK
    decimal quantity
    date expiry_date
    int expiry_alert_days
    enum status
    timestamp disposed_at
    uuid disposed_by FK
    timestamp received_at
    timestamp created_at
    timestamp updated_at
  }

  SUPPLY_REQUEST {
    uuid id PK
    uuid supply_item_id FK
    uuid requester_id FK
    uuid building_id FK
    decimal quantity
    string purpose
    enum status
    uuid manager_reviewed_by FK
    timestamp manager_reviewed_at
    string manager_rejection_reason
    uuid facilities_reviewed_by FK
    timestamp facilities_reviewed_at
    string facilities_rejection_reason
    timestamp fulfilled_at
    timestamp created_at
    timestamp updated_at
  }

  REORDER_LOG {
    uuid id PK
    uuid supply_item_id FK
    uuid building_id FK
    timestamp triggered_at
    enum triggered_by
    decimal stock_level_at_trigger
    decimal reorder_quantity
    jsonb procurement_response
    enum status
    timestamp created_at
  }

  %% ══════════════════════════════════════════════════════
  %% SECTION 9 — OFFICE ASSETS MANAGEMENT
  %% ══════════════════════════════════════════════════════

  ASSET_CATEGORY {
    uuid id PK
    string name
    int default_lifespan_months
    timestamp created_at
    timestamp updated_at
  }

  ASSET {
    uuid id PK
    uuid asset_category_id FK
    uuid building_id FK
    string asset_tag
    string serial_number
    string name
    string description
    enum condition
    enum status
    date purchase_date
    decimal purchase_value
    date warranty_expiry
    date next_service_date
    timestamp retired_at
    enum retirement_reason
    timestamp created_at
    timestamp updated_at
    timestamp deleted_at
  }

  ASSET_ASSIGNMENT {
    uuid id PK
    uuid asset_id FK
    uuid assigned_to_user_id FK
    string assigned_to_space
    uuid building_id FK
    uuid assigned_by FK
    timestamp assigned_at
    timestamp acknowledged_at
    timestamp returned_at
    enum return_condition
    uuid asset_request_id FK
    bool is_active
    timestamp created_at
    timestamp updated_at
  }

  ASSET_REQUEST {
    uuid id PK
    uuid requester_id FK
    uuid asset_category_id FK
    uuid building_id FK
    string justification
    enum status
    uuid manager_reviewed_by FK
    timestamp manager_reviewed_at
    string manager_rejection_reason
    uuid facilities_reviewed_by FK
    timestamp facilities_reviewed_at
    string facilities_rejection_reason
    timestamp fulfilled_at
    timestamp created_at
    timestamp updated_at
  }

  MAINTENANCE_RECORD {
    uuid id PK
    uuid asset_id FK
    uuid building_id FK
    enum maintenance_type
    enum performed_by_type
    string performed_by_name
    string description
    enum outcome
    date maintenance_date
    date next_service_date
    decimal cost
    uuid fault_report_id FK
    timestamp created_at
    timestamp updated_at
  }

  FAULT_REPORT {
    uuid id PK
    uuid asset_id FK
    uuid reported_by FK
    uuid building_id FK
    enum fault_type
    string description
    enum status
    uuid resolved_by FK
    timestamp resolved_at
    enum resolution_outcome
    timestamp created_at
    timestamp updated_at
  }

  %% ══════════════════════════════════════════════════════
  %% SECTION 10 — GLOBAL: AUDIT & NOTIFICATIONS
  %% ══════════════════════════════════════════════════════

  AUDIT_LOG {
    uuid id PK
    uuid actor_id FK
    enum actor_role
    string action
    string entity_type
    uuid entity_id
    uuid building_id FK
    jsonb previous_state
    jsonb new_state
    timestamp occurred_at
  }

  NOTIFICATION {
    uuid id PK
    uuid recipient_id FK
    enum type
    string title
    string body
    string deep_link
    bool is_read
    timestamp created_at
  }

  %% ══════════════════════════════════════════════════════
  %% RELATIONSHIPS
  %% ══════════════════════════════════════════════════════

  %% ── Location Hierarchy ───────────────────────────────
  ORGANISATION ||--|{ OFFICE : has
  OFFICE ||--|{ OFFICE_BUILDING : has
  OFFICE_BUILDING ||--|| LOCATION_CONFIG : configured_by
  OFFICE_BUILDING ||--|{ FLOOR : has
  FLOOR ||--|{ ROOM : contains
  ROOM ||--|{ SEAT : has

  %% ── Identity: User & Employee ────────────────────────
  USER ||--o| EMPLOYEE : has_profile
  OFFICE_BUILDING ||--o{ USER : primary_location_for
  USER }o--o| USER : managed_by
  USER ||--o{ USER_ROLE : holds
  OFFICE_BUILDING ||--o{ USER_ROLE : scopes
  USER |o--o| SEAT : assigned_to

  %% ── Badge Events & Attendance ────────────────────────
  USER ||--o{ BADGE_EVENT : generates
  OFFICE_BUILDING ||--o{ BADGE_EVENT : occurs_at
  BADGE_EVENT }o--|{ WORK_SESSION : resolved_into
  USER ||--o{ WORK_SESSION : has
  OFFICE_BUILDING ||--o{ WORK_SESSION : at
  WORK_SESSION ||--o| ATTENDANCE_RECORD : populates
  USER ||--|{ ATTENDANCE_RECORD : has
  OFFICE_BUILDING ||--o{ ATTENDANCE_RECORD : scopes
  REMOTE_REQUEST ||--o| ATTENDANCE_RECORD : linked_to
  OOO_REQUEST ||--o| ATTENDANCE_RECORD : linked_to
  OFFICE_BUILDING ||--o{ PUBLIC_HOLIDAY : defines

  %% ── Seating Management ───────────────────────────────
  USER ||--o{ WORK_SCHEDULE : has
  SEAT ||--o{ WORK_SCHEDULE : assigned_via
  USER ||--o{ SEAT_BOOKING : makes
  SEAT ||--o{ SEAT_BOOKING : booked_as
  OFFICE_BUILDING ||--o{ SEAT_BOOKING : at
  BLOCK_RESERVATION ||--o{ SEAT_BOOKING : generates
  USER ||--o{ BLOCK_RESERVATION : creates
  ROOM ||--o{ BLOCK_RESERVATION : targets
  OFFICE_BUILDING ||--o{ BLOCK_RESERVATION : at
  SEAT_BOOKING ||--o| NO_SHOW_RECORD : triggers
  USER ||--o{ NO_SHOW_RECORD : recorded_for
  OFFICE_BUILDING ||--o{ NO_SHOW_RECORD : at
  SEAT ||--o{ SEAT_OCCUPANCY_LOG : tracked_in
  USER ||--o{ SEAT_OCCUPANCY_LOG : logged_for
  USER ||--o{ FLOOR_LEAD : serves_as
  ROOM ||--o{ FLOOR_LEAD : led_by

  %% ── Remote Day & OOO Management ──────────────────────
  USER ||--o{ RECURRING_REMOTE_SCHEDULE : has
  OFFICE_BUILDING ||--o{ RECURRING_REMOTE_SCHEDULE : at
  RECURRING_REMOTE_SCHEDULE ||--o{ REMOTE_REQUEST : generates
  USER ||--o{ REMOTE_REQUEST : submits
  OFFICE_BUILDING ||--o{ REMOTE_REQUEST : at
  USER ||--o{ OOO_REQUEST : submits
  OFFICE_BUILDING ||--o{ OOO_REQUEST : at
  OFFICE_BUILDING ||--o{ REMOTE_DAY_POLICY : configures
  USER ||--o{ REMOTE_DAY_POLICY : scoped_to_manager
  OOO_REQUEST ||--o| APPROVAL_DELEGATE : triggers
  USER ||--o{ APPROVAL_DELEGATE : delegates
  USER ||--o{ APPROVAL_DELEGATE : acts_as
  OFFICE_BUILDING ||--o{ APPROVAL_DELEGATE : scopes

  %% ── Visitor Management ───────────────────────────────
  USER ||--o{ VISITOR_PROFILE : registers
  VISITOR_PROFILE ||--o{ PARENT_VISIT : for
  USER ||--o{ PARENT_VISIT : hosts
  OFFICE_BUILDING ||--o{ PARENT_VISIT : at
  PARENT_VISIT ||--o{ VISIT_RECORD : includes
  VISITOR_PROFILE ||--o{ VISIT_RECORD : for
  USER ||--o{ VISIT_RECORD : hosts
  OFFICE_BUILDING ||--o{ VISIT_RECORD : at
  OFFICE_BUILDING ||--o{ AGREEMENT_TEMPLATE : owns
  AGREEMENT_TEMPLATE ||--o{ VISIT_RECORD : signed_under

  %% ── Office Event Management ──────────────────────────
  OFFICE_BUILDING ||--o{ EVENT_SERIES : hosts
  USER ||--o{ EVENT_SERIES : organises
  EVENT_SERIES ||--o{ EVENT : generates
  OFFICE_BUILDING ||--o{ EVENT : at
  USER ||--o{ EVENT : organises
  EVENT ||--o{ EVENT_INVITE : has
  USER ||--o{ EVENT_INVITE : invited
  VISIT_RECORD ||--o{ EVENT_INVITE : for_external_attendee

  %% ── Office Supplies Management ───────────────────────
  SUPPLY_CATEGORY ||--o{ SUPPLY_ITEM : categorises
  OFFICE_BUILDING ||--o{ SUPPLY_ITEM : stocks
  SUPPLY_ITEM ||--o{ SUPPLY_STOCK_ENTRY : tracked_in
  OFFICE_BUILDING ||--o{ SUPPLY_STOCK_ENTRY : at
  SUPPLY_ITEM ||--o{ SUPPLY_REQUEST : requested_for
  USER ||--o{ SUPPLY_REQUEST : submits
  OFFICE_BUILDING ||--o{ SUPPLY_REQUEST : at
  SUPPLY_ITEM ||--o{ REORDER_LOG : triggers
  OFFICE_BUILDING ||--o{ REORDER_LOG : at

  %% ── Office Assets Management ─────────────────────────
  ASSET_CATEGORY ||--o{ ASSET : categorises
  ASSET_CATEGORY ||--o{ ASSET_REQUEST : for_category
  OFFICE_BUILDING ||--o{ ASSET : located_at
  ASSET ||--o{ ASSET_ASSIGNMENT : assigned_via
  USER ||--o{ ASSET_ASSIGNMENT : assigned_to
  OFFICE_BUILDING ||--o{ ASSET_ASSIGNMENT : at
  ASSET_REQUEST ||--o| ASSET_ASSIGNMENT : originated_from
  USER ||--o{ ASSET_REQUEST : submits
  OFFICE_BUILDING ||--o{ ASSET_REQUEST : at
  ASSET ||--o{ FAULT_REPORT : reported_for
  USER ||--o{ FAULT_REPORT : reported_by
  OFFICE_BUILDING ||--o{ FAULT_REPORT : at
  ASSET ||--o{ MAINTENANCE_RECORD : has
  OFFICE_BUILDING ||--o{ MAINTENANCE_RECORD : at
  FAULT_REPORT ||--o{ MAINTENANCE_RECORD : spawns
  %% ── Audit & Notifications ────────────────────────────
  USER ||--o{ AUDIT_LOG : performed
  OFFICE_BUILDING ||--o{ AUDIT_LOG : scopes
  USER ||--o{ NOTIFICATION : receives
