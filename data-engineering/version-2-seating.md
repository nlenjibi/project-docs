```mermaid

erDiagram

  COUNTRY {
    uuid country_id PK
    string country_name
    string country_code
    string timezone
  }

  CITY {
    uuid city_id PK
    uuid country_id FK
    string city_name
  }

  OFFICE_BUILDING {
    uuid building_id PK
    uuid city_id FK
    string building_name
    string address
    int total_floors
  }

  LOCATION_CONFIG {
    uuid config_id PK
    uuid building_id FK
    time work_start_time
    int lateness_threshold_minutes
    int min_presence_duration_minutes
    time no_show_release_time
    int hot_desk_booking_window_days
    int booking_cancellation_cutoff_hours
    enum seat_visibility_mode
    int session_gap_threshold_hours
  }

  FLOOR {
    uuid floor_id PK
    uuid building_id FK
    string floor_name
    int floor_number
    bool is_active
  }

  ROOM {
    uuid room_id PK
    uuid floor_id FK
    uuid building_id FK
    string room_name
    int total_seats
    enum room_type
    bool is_active
  }

  SEAT {
    uuid seat_id PK
    uuid room_id FK
    uuid floor_id FK
    uuid building_id FK
    string seat_number
    enum seat_type
    enum status
    uuid assigned_user_id FK
    float x_position
    float y_position
  }

  EMPLOYEE {
    uuid employee_id PK
    string employee_code
    string sso_user_id
    string first_name
    string last_name
    string email
    uuid primary_building_id FK
    uuid manager_id FK
    enum employee_type
    string department
    string job_title
    string project
    date employment_start_date
    date employment_end_date
    bool is_active
    timestamp deleted_at
  }

  USER_ROLE {
    uuid role_id PK
    uuid employee_id FK
    uuid building_id FK
    enum role
    timestamp assigned_at
    uuid assigned_by FK
  }

  WORK_SCHEDULE {
    uuid schedule_id PK
    uuid employee_id FK
    uuid seat_id FK
    enum day_of_week
    bool is_remote
    date effective_from
    date effective_to
  }

  SEAT_BOOKING {
    uuid booking_id PK
    uuid seat_id FK
    uuid employee_id FK
    uuid building_id FK
    date booking_date
    enum status
    uuid block_reservation_id FK
    timestamp cancelled_at
    string cancellation_reason
    timestamp auto_released_at
  }

  BLOCK_RESERVATION {
    uuid reservation_id PK
    uuid manager_id FK
    uuid building_id FK
    uuid room_id FK
    date reservation_date
    int seat_count
    string notes
    enum status
  }

  REMOTE_REQUEST {
    uuid request_id PK
    uuid employee_id FK
    uuid building_id FK
    date request_date
    enum request_type
    uuid recurring_schedule_id FK
    enum status
    uuid reviewed_by FK
    timestamp reviewed_at
    string rejection_reason
    string notes
  }

  RECURRING_REMOTE_SCHEDULE {
    uuid schedule_id PK
    uuid employee_id FK
    uuid building_id FK
    enum day_of_week
    date effective_from
    date effective_to
    enum status
    uuid reviewed_by FK
    timestamp reviewed_at
  }

  OOO_REQUEST {
    uuid request_id PK
    uuid employee_id FK
    uuid building_id FK
    enum ooo_type
    date start_date
    date end_date
    enum status
    uuid reviewed_by FK
    timestamp reviewed_at
    string rejection_reason
    string notes
  }

  SEAT_OCCUPANCY_LOG {
    uuid log_id PK
    uuid seat_id FK
    uuid employee_id FK
    date occupancy_date
    bool is_occupied
    bool is_remote_day
    string notes
  }

  BADGE_EVENT {
    uuid event_id PK
    uuid employee_id FK
    uuid building_id FK
    string personnel_id
    enum event_type
    timestamp occurred_at
    jsonb raw_payload
    timestamp ingested_at
  }

  WORK_SESSION {
    uuid session_id PK
    uuid employee_id FK
    uuid building_id FK
    date session_date
    timestamp first_badge_in
    timestamp last_badge_out
    int total_duration_minutes
    bool is_late
    int minutes_late
    bool crosses_midnight
  }

  FLOOR_LEAD {
    uuid lead_id PK
    uuid employee_id FK
    uuid room_id FK
    date assigned_from
    date assigned_to
  }

  COUNTRY ||--o{ CITY : has
  CITY ||--o{ OFFICE_BUILDING : has
  OFFICE_BUILDING ||--|| LOCATION_CONFIG : configured_by
  OFFICE_BUILDING ||--o{ FLOOR : has
  FLOOR ||--o{ ROOM : contains
  ROOM ||--o{ SEAT : has
  OFFICE_BUILDING ||--o{ USER_ROLE : scopes
  EMPLOYEE ||--o{ USER_ROLE : holds
  EMPLOYEE }o--o| SEAT : primary_seat
  SEAT ||--o{ WORK_SCHEDULE : assigned_via
  EMPLOYEE ||--o{ WORK_SCHEDULE : has
  SEAT ||--o{ SEAT_BOOKING : booked_as
  EMPLOYEE ||--o{ SEAT_BOOKING : makes
  BLOCK_RESERVATION ||--o{ SEAT_BOOKING : generates
  EMPLOYEE ||--o{ BLOCK_RESERVATION : creates
  ROOM ||--o{ BLOCK_RESERVATION : targets
  SEAT ||--o{ SEAT_OCCUPANCY_LOG : tracked_in
  EMPLOYEE ||--o{ SEAT_OCCUPANCY_LOG : logged_for
  EMPLOYEE ||--o{ REMOTE_REQUEST : submits
  RECURRING_REMOTE_SCHEDULE ||--o{ REMOTE_REQUEST : generates
  EMPLOYEE ||--o{ RECURRING_REMOTE_SCHEDULE : has
  EMPLOYEE ||--o{ OOO_REQUEST : submits
  EMPLOYEE ||--o{ BADGE_EVENT : generates
  OFFICE_BUILDING ||--o{ BADGE_EVENT : occurs_at
  BADGE_EVENT }o--|| WORK_SESSION : resolved_into
  EMPLOYEE ||--o{ WORK_SESSION : has
  ROOM ||--o{ FLOOR_LEAD : led_by
  EMPLOYEE ||--o{ FLOOR_LEAD : serves_as