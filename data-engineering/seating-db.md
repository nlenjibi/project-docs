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

  ROOM {
    uuid room_id PK
    uuid building_id FK
    string room_name
    int floor_number
    int total_seats
    enum room_type
  }

  SEAT {
    uuid seat_id PK
    uuid room_id FK
    string seat_number
    bool is_permanent
    bool is_bookable
    enum seat_type
  }

  EMPLOYEE {
    uuid employee_id PK
    string full_name
    string email
    string access_level
    string specialization
    uuid primary_building_id FK
    uuid primary_room_id FK
    uuid primary_seat_id FK
    enum employee_type
    enum department
    string project
    date start_date
    date end_date
    bool is_active
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

  SEAT_OCCUPANCY_LOG {
    uuid log_id PK
    uuid seat_id FK
    uuid employee_id FK
    date occupancy_date
    bool is_occupied
    bool is_remote_day
    string notes
  }

  FLOOR_LEAD { 
    uuid lead_id PK
    uuid employee_id FK
    uuid room_id FK
    date assigned_from
    date assigned_to
  }

  ZK_ATTENDANCE_RECORD {
    uuid event_id PK
    bigint event_sequence_number
    string employee_id FK
    string badge_id
    string employee_name
    string department
    string job_title
    string employment_status

    datetime event_timestamp
    datetime device_time
    string timezone
    string shift_id
    string attendance_type

    string device_id
    string device_name
    string device_type
    string device_model
    string device_ip

    string verification_method
    string verification_status
    float biometric_score

    string door_id
    string door_name
    string access_result
    string access_direction

    string location_id
    string building_id
    string floor
    string zone

    float temperature_value
    bool mask_detection

    string image_url
    datetime ingestion_timestamp
    string processing_status
  }

  COUNTRY ||--o{ CITY : has
  CITY ||--o{ OFFICE_BUILDING : has
  OFFICE_BUILDING ||--o{ ROOM : contains
  ROOM ||--o{ SEAT : has
  SEAT ||--o{ WORK_SCHEDULE : assigned_via
  EMPLOYEE ||--o{ WORK_SCHEDULE : has
  SEAT ||--o{ SEAT_OCCUPANCY_LOG : tracked_in
  EMPLOYEE ||--o{ SEAT_OCCUPANCY_LOG : logged_for
  EMPLOYEE }o--|| SEAT : primary_seat
  ROOM ||--o{ FLOOR_LEAD : led_by
  EMPLOYEE ||--o{ FLOOR_LEAD : is

  EMPLOYEE ||--o{ ZK_ATTENDANCE_RECORD : generates
  OFFICE_BUILDING ||--o{ ZK_ATTENDANCE_RECORD : occurs_at