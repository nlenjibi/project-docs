```mermaid
erDiagram

%% =========================
%% RAW LAYER
%% =========================
oms_raw_ZK_ATTENDANCE {
  bigint event_sequence_number
  string employee_id
  datetime event_timestamp
  string device_id
}

oms_raw_HR_EMPLOYEE {
  string employee_id
  string full_name
  string department
  date start_date
}

oms_raw_SHAREPOINT_SEATING {
  string seat_id
  string employee_id
  string room_id
}

oms_raw_ASSET_EVENTS {
  string asset_id
  string event_type
  datetime event_time
}

oms_raw_SUPPLY_REQUESTS {
  string request_id
  string item_id
  string status
}

oms_raw_VISITOR_VISITS {
  string visitor_id
  datetime checkin_time
  datetime checkout_time
}

%% =========================
%% SILVER LAYER (CLEANED)
%% =========================
oms_silver_EMPLOYEE {
  string employee_id
  string department
  string role
}

oms_silver_SEAT {
  string seat_id
  string room_id
  bool is_bookable
}

oms_silver_DEVICE {
  string device_id
  string device_type
  string ip_address
}

%% =========================
%% GOLD LAYER - DIMENSIONS
%% =========================
DIM_EMPLOYEE {
  int employee_sk PK
  string employee_id
  string department
  string role
  date effective_from
  date effective_to
}

DIM_LOCATION {
  int location_sk PK
  string country
  string city
  string building
}

DIM_SEAT {
  int seat_sk PK
  string seat_id
  string seat_type
  bool is_bookable
}

DIM_ROOM {
  int room_sk PK
  string room_id
  int floor_number
}

DIM_DATE {
  int date_sk PK
  date full_date
  int year
  int month
}

DIM_TIME {
  int time_sk PK
  int hour
  int minute
}

DIM_ACCESS_DEVICE {
  int device_sk PK
  string device_id
  string model
  string ip
}

DIM_REQUEST_STATUS {
  int status_sk PK
  string status_name
}

DIM_ASSET {
  int asset_sk PK
  string asset_id
  string asset_type
}

DIM_VISITOR {
  int visitor_sk PK
  string visitor_id
  string company
}

%% =========================
%% GOLD LAYER - FACTS
%% =========================
FACT_ATTENDANCE {
  int attendance_sk PK
  int employee_sk FK
  int date_sk FK
  int device_sk FK
  datetime first_in
  datetime last_out
  bool is_present
  bool is_late
}

FACT_SEAT_OCCUPANCY {
  int seat_occ_sk PK
  int seat_sk FK
  int employee_sk FK
  int date_sk FK
  bool occupied
}

FACT_REMOTE_OOO {
  int request_sk PK
  int employee_sk FK
  int date_sk FK
  string request_type
  string status
}

FACT_SUPPLY_REQUEST {
  int supply_request_sk PK
  int employee_sk FK
  int item_sk FK
  int status_sk FK
  date request_date
}

FACT_ASSET_EVENT {
  int asset_event_sk PK
  int asset_sk FK
  int employee_sk FK
  int date_sk FK
  string event_type
}

FACT_VISITOR_VISIT {
  int visit_sk PK
  int visitor_sk FK
  int employee_sk FK
  int date_sk FK
  datetime checkin_time
}

%% =========================
%% BRIDGE LAYER
%% =========================
BRIDGE_EMPLOYEE_ROLE_LOCATION {
  int employee_sk FK
  int location_sk FK
  string role
}

BRIDGE_EMPLOYEE_SEAT {
  int employee_sk FK
  int seat_sk FK
  date start_date
  date end_date
}

BRIDGE_EVENT_ATTENDEE {
  int visitor_sk FK
  int employee_sk FK
  int visit_sk FK
}

%% =========================
%% META LAYER
%% =========================
AUDIT_LOG {
  int audit_id PK
  string entity_type
  string entity_id
  string action
  datetime action_time
}

ETL_JOB_RUN {
  int job_run_id PK
  string job_name
  datetime start_time
  datetime end_time
  string status
  int rows_processed
}

DATA_QUALITY_ISSUES {
  int issue_id PK
  string table_name
  string issue_type
  string severity
}

%% =========================
%% RELATIONSHIPS (CORE FACTS)
%% =========================

DIM_EMPLOYEE ||--o{ FACT_ATTENDANCE : has
DIM_DATE ||--o{ FACT_ATTENDANCE : records
DIM_ACCESS_DEVICE ||--o{ FACT_ATTENDANCE : captures

DIM_SEAT ||--o{ FACT_SEAT_OCCUPANCY : used_in
DIM_EMPLOYEE ||--o{ FACT_SEAT_OCCUPANCY : occupies
DIM_DATE ||--o{ FACT_SEAT_OCCUPANCY : on

DIM_EMPLOYEE ||--o{ FACT_REMOTE_OOO : submits
DIM_DATE ||--o{ FACT_REMOTE_OOO : scheduled

DIM_EMPLOYEE ||--o{ FACT_SUPPLY_REQUEST : requests
DIM_REQUEST_STATUS ||--o{ FACT_SUPPLY_REQUEST : status

DIM_ASSET ||--o{ FACT_ASSET_EVENT : changes
DIM_EMPLOYEE ||--o{ FACT_ASSET_EVENT : performs

DIM_VISITOR ||--o{ FACT_VISITOR_VISIT : has
DIM_EMPLOYEE ||--o{ FACT_VISITOR_VISIT : hosts