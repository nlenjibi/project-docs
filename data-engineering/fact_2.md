```mermaid 
erDiagram

%% =========================
%% DIMENSIONS
%% =========================

DIM_EMPLOYEE {
  bigint employee_sk PK
  string employee_id
  string full_name
  string department
  string job_title
  date effective_from
  date effective_to
  bool is_current
}

DIM_LOCATION {
  bigint location_sk PK
  string country
  string city
  string building
}

DIM_DATE {
  bigint date_sk PK
  date full_date
  int year
  int month
  int day
}

DIM_TIME {
  bigint time_sk PK
  int hour
  int minute
}

DIM_SEAT {
  bigint seat_sk PK
  string seat_number
  string seat_type
  bool is_bookable
}

DIM_ASSET {
  bigint asset_sk PK
  string asset_type
  string status
}

DIM_SUPPLY_ITEM {
  bigint supply_item_sk PK
  string item_name
  string category
}

DIM_REQUEST_STATUS {
  bigint request_status_sk PK
  string status_name
  bool is_terminal
}

DIM_ACCESS_DEVICE {
  bigint device_sk PK
  string device_id
  string device_model
  string ip_address
  string zone
}


%% =========================
%% BRIDGE TABLES
%% =========================

BRIDGE_EMPLOYEE_ROLE {
  bigint employee_sk
  bigint location_sk
  string role
  date effective_from
  date effective_to
}

BRIDGE_EMPLOYEE_MANAGER {
  bigint employee_sk
  bigint manager_sk
  bigint location_sk
  date effective_from
  date effective_to
}

BRIDGE_EVENT_ATTENDEE {
  bigint event_id
  bigint employee_sk
  string rsvp_status
}


%% =========================
%% FACT TABLES
%% =========================

FACT_ATTENDANCE_EVENT {
  bigint attendance_event_sk PK
  bigint employee_sk FK
  bigint location_sk FK
  bigint device_sk FK
  bigint date_sk FK
  bigint time_sk FK
  string access_result
  string direction
}

FACT_ATTENDANCE_DAILY {
  bigint attendance_daily_sk PK
  bigint employee_sk FK
  bigint location_sk FK
  bigint date_sk FK
  float total_hours
  bool is_present
  bool is_late
  bool is_remote
}

FACT_SEAT_OCCUPANCY {
  bigint seat_occupancy_sk PK
  bigint seat_sk FK
  bigint employee_sk FK
  bigint location_sk FK
  bigint date_sk FK
  bool occupied_flag
  bool no_show_flag
}

FACT_REQUEST {
  bigint request_sk PK
  bigint employee_sk FK
  bigint approver_sk FK
  bigint location_sk FK
  bigint request_status_sk FK
  string request_type
  date submitted_at
  date approved_at
}

FACT_ASSET_EVENT {
  bigint asset_event_sk PK
  bigint asset_sk FK
  bigint employee_sk FK
  bigint location_sk FK
  bigint date_sk FK
  string event_type
}

FACT_SUPPLY_REQUEST {
  bigint supply_request_sk PK
  bigint supply_item_sk FK
  bigint employee_sk FK
  bigint location_sk FK
  bigint date_sk FK
  int quantity
}


%% =========================
%% RELATIONSHIPS
%% =========================

DIM_EMPLOYEE ||--o{ FACT_ATTENDANCE_EVENT : generates
DIM_EMPLOYEE ||--o{ FACT_ATTENDANCE_DAILY : summarized_in
DIM_EMPLOYEE ||--o{ FACT_REQUEST : submits
DIM_EMPLOYEE ||--o{ FACT_ASSET_EVENT : triggers
DIM_EMPLOYEE ||--o{ FACT_SUPPLY_REQUEST : requests

DIM_LOCATION ||--o{ FACT_ATTENDANCE_EVENT : occurs_in
DIM_LOCATION ||--o{ FACT_SEAT_OCCUPANCY : located_in
DIM_LOCATION ||--o{ FACT_REQUEST : scoped_to

DIM_DATE ||--o{ FACT_ATTENDANCE_EVENT : on
DIM_DATE ||--o{ FACT_ATTENDANCE_DAILY : on
DIM_DATE ||--o{ FACT_REQUEST : on

DIM_SEAT ||--o{ FACT_SEAT_OCCUPANCY : used_in

DIM_ASSET ||--o{ FACT_ASSET_EVENT : tracked_in
DIM_SUPPLY_ITEM ||--o{ FACT_SUPPLY_REQUEST : requested

DIM_ACCESS_DEVICE ||--o{ FACT_ATTENDANCE_EVENT : records

DIM_REQUEST_STATUS ||--o{ FACT_REQUEST : status

%% =========================
%% BRIDGES RELATIONSHIPS
%% =========================

DIM_EMPLOYEE ||--o{ BRIDGE_EMPLOYEE_ROLE : has
DIM_LOCATION ||--o{ BRIDGE_EMPLOYEE_ROLE : scoped_to

DIM_EMPLOYEE ||--o{ BRIDGE_EMPLOYEE_MANAGER : reports_to
DIM_EMPLOYEE ||--o{ BRIDGE_EMPLOYEE_MANAGER : manages

DIM_EMPLOYEE ||--o{ BRIDGE_EVENT_ATTENDEE : attends