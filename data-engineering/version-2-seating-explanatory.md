
---
# Office Management System (OMS) — Data Model Documentation

**Version:** 1.0 | **Status:** Draft | **Last Updated:** April 2026
**Source of truth:** `oms-final-data-model.md`

---

## 1. Overview

This data model supports a **multi-country, multi-office workplace management system**.
It is designed to run across all company locations — Ghana (Accra, Kumasi, Takoradi), Rwanda, Germany, and the US — from a single unified schema.

The model handles:

* Physical office layout from country down to individual seat
* Employee master data and role-based access per location
* Badge-driven attendance tracking and session resolution
* Seating assignment, hot-desk booking, and occupancy visibility
* Remote day and out-of-office request workflows
* Visitor registration and reception management
* Office events and invitations
* Consumable supply tracking and reordering
* Physical asset lifecycle management
* Immutable audit logging and in-app notifications

The schema is structured into **ten functional domains**:

| # | Domain | Key entities |
|---|---|---|
| 1 | Geographic & Location Hierarchy | COUNTRY, CITY, OFFICE_BUILDING, LOCATION_CONFIG, FLOOR, ROOM, SEAT |
| 2 | People & Access Control | EMPLOYEE, USER_ROLE |
| 3 | Badge Events & Attendance | BADGE_EVENT, WORK_SESSION, ATTENDANCE_RECORD, PUBLIC_HOLIDAY |
| 4 | Seating Management | WORK_SCHEDULE, SEAT_BOOKING, BLOCK_RESERVATION, NO_SHOW_RECORD, SEAT_OCCUPANCY_LOG, FLOOR_LEAD |
| 5 | Remote Day & OOO Management | REMOTE_REQUEST, RECURRING_REMOTE_SCHEDULE, OOO_REQUEST, REMOTE_DAY_POLICY, APPROVAL_DELEGATE |
| 6 | Visitor Management | VISITOR_PROFILE, PARENT_VISIT, VISIT_RECORD, AGREEMENT_TEMPLATE |
| 7 | Office Event Management | EVENT_SERIES, EVENT, EVENT_INVITE |
| 8 | Office Supplies Management | SUPPLY_CATEGORY, SUPPLY_ITEM, SUPPLY_STOCK_ENTRY, SUPPLY_REQUEST, REORDER_LOG |
| 9 | Office Assets Management | ASSET_CATEGORY, ASSET, ASSET_ASSIGNMENT, ASSET_REQUEST, MAINTENANCE_RECORD, FAULT_REPORT |
| 10 | Global: Audit & Notifications | AUDIT_LOG, NOTIFICATION |

### Core design principles

* **UUID primary keys** — safe for distributed systems; avoids sequential ID enumeration.
* **Soft deletes via `deleted_at`** — nothing is permanently destroyed; full history is preserved.
* **`building_id` on every non-global entity** — enforces data isolation between offices at the schema level.
* **Standard audit fields** — every table carries `created_at` and `updated_at`; mutable entities also carry `deleted_at`.
* **Immutable logs** — `BADGE_EVENT`, `AUDIT_LOG`, `NO_SHOW_RECORD`, and `REORDER_LOG` are append-only; they are never updated or deleted.

---

# 2. Geographic & Location Hierarchy

These seven tables form the **spine of the entire model**. Every other entity traces back to `OFFICE_BUILDING`.

---

## COUNTRY

### Purpose

Top-level geographic grouping. The company operates in Ghana, Rwanda, Germany, and the US.

### Stores

* Country name and ISO 3166-1 alpha-2 code (e.g. `GH`, `RW`, `DE`, `US`)
* IANA timezone string (e.g. `Africa/Accra`, `Europe/Berlin`)

### Why it exists

* Provides timezone-aware attendance calculation — a badge event timestamped at 08:55 means different things in Accra versus Berlin.
* Enables country-level analytics and reporting rollups.

---

## CITY

### Purpose

Represents cities within a country where the company has offices.

### Stores

* City name
* Foreign key to `COUNTRY`

### Why it exists

* Organises buildings geographically below country level.
* Ghana, for example, has offices in three cities — Kumasi, Accra, and Takoradi — so city is a meaningful grouping.

---

## OFFICE_BUILDING

### Purpose

Represents a single physical office building. This is the **primary scoping unit** of the entire system.

### Stores

* Building name (e.g. "Kumasi Office", "Accra Office")
* Street address
* Total number of floors
* Active flag

### Why it exists

* Every attendance record, seat booking, policy, and request is scoped to a building.
* Enables separate configurations, policies, and reporting per office.
* The `is_active` flag allows a building to be retired without deleting its historical data.

---

## LOCATION_CONFIG

### Purpose

Defines **operational rules for a specific office**. One config row per building.

### Stores

* `work_start_time` — the official start of the working day (e.g. 09:00)
* `lateness_threshold_minutes` — how many minutes after start time before an employee is marked late
* `min_presence_duration_minutes` — minimum in-office minutes to count as present (not just a brief visit)
* `no_show_release_time` — the time at which an unchecked-in seat booking is auto-released
* `hot_desk_booking_window_days` — how far in advance a hot desk can be booked
* `booking_cancellation_cutoff_hours` — deadline before a booking to allow cancellation
* `seat_visibility_mode` — controls what employees can see (`FULL`, `TEAM_ONLY`, `AVAILABILITY_ONLY`)
* `session_gap_threshold_hours` — gap between badge events that triggers a new session rather than extending the current one

### Why it exists (critical table)

Badge events are raw timestamps. Without `LOCATION_CONFIG`, the system cannot answer:

* Is this employee late?
* Does this visit count as presence?
* When should an unchecked-in seat be released?

Every attendance calculation reads this table first.

---

## FLOOR

### Purpose

Represents a floor within a building.

### Stores

* Floor number and display name (e.g. "Floor 3", "Ground Floor")
* Active flag

### Why it exists

* In the Kumasi Office, for example, workspaces run from the 3rd to the 6th floor. Floor is a meaningful physical boundary.
* Supports floor-level analytics, navigation, and facilities management.
* Intermediate link in the hierarchy: `OFFICE_BUILDING → FLOOR → ROOM → SEAT`.

---

## ROOM

### Purpose

Represents a named office space within a floor — what staff call a room or wing (e.g. "Effiekuma", "Kweikuma", "Adiembra", "Apremdo").

### Stores

* Room name and type (e.g. open-plan workspace, private office, meeting room)
* Total seat count in the room
* Active flag

### Why it exists

* Seats are not just "in the building" — they are in specific rooms. A seat number like A-12 only makes sense in context of a room.
* Block reservations target rooms; floor leads are assigned to rooms.
* Enables room-level occupancy and utilisation analytics.

> **Mapping note:** `ROOM` in this model maps to the `Zone` concept in the OMS specification. The word "room" better reflects how staff refer to their workspace.

---

## SEAT

### Purpose

Represents an individual seat — a numbered workstation that can be assigned, booked, or tracked.

### Stores

* `seat_number` — display identifier (e.g. "A-12")
* `seat_type` — `PERMANENT` (assigned to one person) or `HOT_DESK` (open for booking)
* `status` — `AVAILABLE`, `UNAVAILABLE`, or `MAINTENANCE`
* `assigned_employee_id` — set only for permanent seats
* `x_position`, `y_position` — canvas coordinates for floor plan visualisation
* `deleted_at` — soft delete when a seat is decommissioned

### Why it exists

* The seat is the atomic unit of physical presence. The shared-point tracking of who sits where starts and ends here.
* Coordinates enable an interactive floor plan view showing which seats are occupied in real time.
* `assigned_employee_id` determines whose seat is "empty" on a remote day without requiring a booking.

---

# 3. People & Access Control

---

## EMPLOYEE

### Purpose

Stores the master record for every person with access to the OMS — full-time staff, employees on probation, NSP (National Service Personnel), and interns. Synced from the HR system (Arms).

### Stores

* Identity: `first_name`, `last_name`, `email`, `employee_code`, `sso_user_id`
* Organisational: `department`, `job_title`, `project`, `manager_id` (self-referencing FK)
* `employee_type` enum — distinguishes FULL_TIME, PROBATION, NSP, INTERN
* `primary_building_id` — home office
* `employment_start_date` / `employment_end_date` — sourced from Arms; attendance history begins at start date
* `language_preference` — ISO 639-1 code for UI localisation
* `deleted_at` — soft delete on offboarding

### Why it exists

* Central entity that everything else links back to. An attendance record, seat booking, leave request, asset assignment — all reference `employee_id`.
* The `employee_type` field directly captures the categories tracked in the SharePoint file (probation, NSP, intern, full staff).
* `deleted_at` rather than hard delete preserves historical records — a departed employee's attendance history remains queryable.

---

## USER_ROLE

### Purpose

Assigns a role to an employee within a specific building. An employee may have different roles in different offices.

### Stores

* `role` — `EMPLOYEE`, `MANAGER`, `HR`, `FACILITIES_ADMIN`, `SUPER_ADMIN`
* `building_id` — the building where this role applies (NULL for Super Admin, which is global)
* `assigned_by` and `assigned_at`

### Why it exists

* Role determines what actions a person can take in the system (approve requests, configure policies, manage assets).
* Building-scoped roles mean a person can be a Manager in Accra but a regular Employee in Kumasi — their approval permissions only apply in the office where the role is granted.

---

# 4. Badge Events & Attendance

This domain processes the raw physical entry/exit stream into meaningful daily attendance records.

---

## BADGE_EVENT

### Purpose

Stores every raw badge-in or badge-out event ingested from the door access hardware (via Athena). Immutable once written.

### Stores

* `personnel_id` — raw ID from the Athena device (e.g. "acc-42")
* `employee_id` — resolved via identity mapping after ingestion
* `event_type` — `BADGE_IN` or `BADGE_OUT`
* `occurred_at` — exact timestamp from the device
* `raw_payload` (JSONB) — full Athena payload for traceability
* `ingested_at` — when the sync job picked up the event

### Why it exists

* This is the **bronze layer** — the immutable source of truth for all physical presence.
* Separating raw ingestion from processed output means a bug in the processing logic can be corrected by reprocessing, not by altering the source.
* `raw_payload` preserves the full device data even if the parsed fields miss something.

---

## WORK_SESSION

### Purpose

A resolved work session derived from one or more badge events for a single employee on a single day.

### Stores

* `first_badge_in` — timestamp of the first entry event of the session
* `last_badge_out` — timestamp of the last exit (nullable if session is still open)
* `total_duration_minutes` — computed from the above
* `is_late` / `minutes_late` — derived against `LOCATION_CONFIG.work_start_time` and `lateness_threshold_minutes`
* `crosses_midnight` — true if a session spans two calendar days
* `session_date` — anchored to the calendar date of the first badge-in

### Why it exists

* This is the **silver/gold layer** — raw events transformed into business-readable time.
* Multiple badge events (e.g., leave for lunch and return) collapse into one session for the day, controlled by `session_gap_threshold_hours`.
* Lateness and duration calculations are pre-computed here so reports do not need to recalculate them at query time.

---

## ATTENDANCE_RECORD

### Purpose

The **resolved daily attendance status** for an employee. One row per employee per date — the single source of truth for "what did this person do today?"

### Stores

* `status` — `PRESENT`, `ABSENT`, `LATE`, `INSUFFICIENT_HOURS`, `REMOTE`, `ON_LEAVE`, `PUBLIC_HOLIDAY`
* `work_session_id` — linked session if the employee was physically present
* `remote_request_id` — linked approved remote request if status is REMOTE
* `leave_request_id` — linked approved OOO request if status is ON_LEAVE
* `is_overridden` / `override_by` / `override_reason` — manual correction by Super Admin

### Why it exists

* Consolidates all inputs (badge events, remote approvals, OOO approvals, holidays) into one clean row per day.
* Attendance dashboards, payroll integrations, and HR reports all read from this table — not from raw badge events.
* The override mechanism supports legitimate edge cases (hardware failure, manual correction) without touching immutable source data.

---

## PUBLIC_HOLIDAY

### Purpose

Manually configured public holidays per office location.

### Stores

* `holiday_date` and `name` (e.g. "Independence Day")
* Scoped to a `building_id`

### Why it exists

* Different offices observe different holidays. Ghana's Independence Day is not observed in Germany.
* When attendance records are generated, a date that falls on a public holiday produces a `PUBLIC_HOLIDAY` status automatically.
* Prevents employees from being wrongly marked absent on national holidays.

---

### Attendance processing flow

```text
Door access hardware
        │
        ▼
  BADGE_EVENT          ← immutable, raw, bronze layer
  (ingested via Athena)
        │
        ▼
  WORK_SESSION         ← processed session, silver layer
  (gap-threshold logic │ lateness calculation)
        │
        ├── + REMOTE_REQUEST (if approved)
        ├── + OOO_REQUEST    (if approved)
        ├── + PUBLIC_HOLIDAY (if date matches)
        │
        ▼
  ATTENDANCE_RECORD    ← single status per person per day, gold layer
```

---

# 5. Seating Management

This domain manages physical seat allocation, hot-desk booking, team reservations, and occupancy tracking — including visibility into which seats are empty because an employee is on a remote day.

---

## WORK_SCHEDULE

### Purpose

Defines an employee's planned weekly seat assignment — which days they come in, to which seat, and whether a given day is a remote day.

### Stores

* `seat_id` — the seat they plan to use
* `day_of_week` — e.g. MONDAY, TUESDAY
* `is_remote` — if true, the seat is expected to be empty that day
* `effective_from` / `effective_to` — supports schedule changes over time

### Why it exists

* The shared-point file this system replaces tracked exactly this: who sits where and on which days.
* `is_remote = true` drives `SEAT_OCCUPANCY_LOG.is_remote_day = true` without requiring the employee to make a booking or request.
* Enables proactive space planning — facilities can see which desks are free before the day starts.

---

## SEAT_BOOKING

### Purpose

Represents a hot-desk booking for a specific date.

### Stores

* `seat_id`, `employee_id`, `booking_date`
* `status` — `CONFIRMED`, `CANCELLED`, `NO_SHOW`, `RELEASED`
* `block_reservation_id` — set if this booking was generated by a manager's block reservation
* `auto_released_at` — timestamp set when the no-show release job fires

### Why it exists

* Allows employees who do not have a permanent seat to reserve a desk.
* `auto_released_at` frees up seats that were booked but never used, improving space utilisation.
* Links to `BLOCK_RESERVATION` so team bookings remain traceable to the manager who created them.

---

## BLOCK_RESERVATION

### Purpose

Allows a manager to bulk-reserve a number of seats in a specific room for their team on a given date.

### Stores

* `manager_id`, `room_id`, `reservation_date`, `seat_count`
* `status` — `ACTIVE` or `CANCELLED`

### Why it exists

* When a team has an in-office collaboration day, a manager needs to guarantee enough desks without booking individual seats one by one.
* Generates individual `SEAT_BOOKING` rows for the team members as they confirm attendance.

---

## NO_SHOW_RECORD

### Purpose

Tracks no-show events for reporting and accountability. Created automatically by the auto-release job when a booking is not checked in by `LOCATION_CONFIG.no_show_release_time`.

### Stores

* `seat_booking_id`, `employee_id`, `building_id`, `no_show_date`
* Immutable — append-only log

### Why it exists

* Identifies employees who repeatedly book seats and do not show up, consuming capacity.
* Feeds into admin dashboards and can trigger policy enforcement (e.g., booking privileges restricted after N no-shows).

---

## SEAT_OCCUPANCY_LOG

### Purpose

Tracks actual seat usage day by day — whether a seat was physically occupied, or was empty because the assigned employee was working remotely.

### Stores

* `occupancy_date`, `is_occupied`, `is_remote_day`
* `employee_id` — who was expected to be in that seat

### Why it exists

* This is the real-time equivalent of the SharePoint seating chart — it answers "is seat A-12 empty today?"
* `is_remote_day = true` means the seat is intentionally empty because the employee has an approved remote day, not abandoned.
* Drives the live floor plan view and daily space utilisation metrics.

---

## FLOOR_LEAD

### Purpose

Assigns an employee as the responsible lead for a specific room within a floor.

### Stores

* `employee_id`, `room_id`, `assigned_from`, `assigned_to`

### Why it exists

* The project describes named rooms (Effiekuma, Kweikuma, etc.) — each of these needs a point of contact for facilities issues, headcount, and access.
* Time-bounded assignment supports staff rotation without losing history.

---

# 6. Remote Day & OOO Management

This domain manages all forms of planned absence from the office — one-off remote days, recurring schedules, leave, and travel — and ensures approvals flow correctly even when a manager is absent.

---

## REMOTE_REQUEST

### Purpose

A single remote work day request submitted by an employee for manager approval.

### Stores

* `request_date`, `request_type` (`ONE_OFF` or `RECURRING_INSTANCE`)
* `recurring_schedule_id` — set if this is an instance generated by an approved recurring schedule
* `status` — `PENDING`, `APPROVED`, `REJECTED`, `CANCELLED`
* `reviewed_by`, `reviewed_at`, `rejection_reason`

### Why it exists

* An approved remote request marks the employee's seat as intentionally empty and sets their `ATTENDANCE_RECORD.status` to `REMOTE`.
* Prevents false absence flags for employees on approved remote days.

---

## RECURRING_REMOTE_SCHEDULE

### Purpose

A standing approval for an employee to work remotely on a specific day of the week (e.g. every Friday).

### Stores

* `day_of_week`, `effective_from`, `effective_to`
* `status` — follows the same approval lifecycle as `REMOTE_REQUEST`

### Why it exists

* A single approval covers all future instances — the employee does not need to submit a request every week.
* When active, the schedule automatically generates `REMOTE_REQUEST` instances with `request_type = RECURRING_INSTANCE`.
* `effective_to` allows a recurring schedule to expire naturally (e.g. end of a project phase).

---

## OOO_REQUEST

### Purpose

Represents an out-of-office period — annual leave, sick leave, personal day, business travel, or other.

### Stores

* `ooo_type` — `ANNUAL_LEAVE`, `SICK_LEAVE`, `PERSONAL_DAY`, `BUSINESS_TRAVEL`, `OTHER`
* `start_date`, `end_date`
* `status` — `PENDING`, `APPROVED`, `REJECTED`, `CANCELLED`

### Why it exists

* Approved OOO requests drive `ATTENDANCE_RECORD.status = ON_LEAVE` for all covered dates.
* Business travel is tracked separately from personal leave for reporting purposes.
* An approved OOO request can trigger `APPROVAL_DELEGATE` creation — ensuring another manager can approve requests while this one is away.

---

## REMOTE_DAY_POLICY

### Purpose

Configurable limit on remote days per week, set at building level or per manager's team.

### Stores

* `max_remote_days_per_week`
* `enforcement_mode` — `HARD_BLOCK` (system rejects the request) or `SOFT_WARNING` (system warns but allows)
* `manager_id` — NULL means the policy applies to the whole building

### Why it exists

* Prevents excessive remote day requests without manager involvement.
* `HARD_BLOCK` enforces limits automatically; `SOFT_WARNING` flags the request for manager attention.

---

## APPROVAL_DELEGATE

### Purpose

Records a manager's nominated deputy for approving requests during their absence.

### Stores

* `manager_id` — the absent manager
* `delegated_to_id` — the employee acting as deputy (must be a Manager or HR role)
* `effective_from` / `effective_to`
* `ooo_request_id` — the manager's own OOO that triggered the delegation
* `is_active`

### Why it exists

* When a manager takes leave, their team's remote and OOO requests cannot sit in a pending queue indefinitely.
* Delegation is tied to a specific OOO period — it activates and expires with the manager's absence.

---

# 7. Visitor Management

This domain handles the full visitor lifecycle — from pre-registration to check-in and security agreement signing.

---

## VISITOR_PROFILE

### Purpose

A reusable profile for a known external visitor that persists across multiple visits.

### Stores

* `first_name`, `last_name`, `email`, `phone`, `company`
* `created_by` — the employee who first registered this visitor
* `deleted_at` — soft delete for GDPR compliance

### Why it exists

* Avoids re-entering the same visitor's details for every visit.
* Provides a searchable history of all visits from a specific person or company.

---

## PARENT_VISIT

### Purpose

Represents a multi-day or recurring visit engagement (e.g., an external contractor on site for two weeks).

### Stores

* `visitor_profile_id`, `host_employee_id`, `building_id`
* `purpose`, `access_level` — `ESCORTED_ONLY`, `RECEPTION_ONLY`, `FLOOR_ACCESS`, `FULL_ACCESS`
* `start_date`, `end_date`

### Why it exists

* A long-term visitor should not require a new visit record registered by reception every single day.
* `access_level` drives what doors and areas the visitor badge will grant access to during the engagement.

---

## VISIT_RECORD

### Purpose

Represents a single day's visit — either a child of `PARENT_VISIT` (multi-day) or a standalone single-day visit.

### Stores

* `visit_date`, `expected_arrival_time`
* `actual_check_in` / `actual_check_out` — set by facilities admin at reception
* `status` — `EXPECTED`, `CHECKED_IN`, `CHECKED_OUT`, `NO_SHOW`, `CANCELLED`
* `is_walk_in` — true for unregistered visitors arriving without a prior booking
* `agreement_signed` / `agreement_signed_at` / `agreement_template_id` — NDA/entry terms tracking
* `checked_in_by` — the facilities admin who processed the check-in

### Why it exists

* Reception needs a simple per-day row to process arrivals — "who is expected, who has arrived, who has left."
* Walk-in tracking ensures all visitors are logged even if they were not pre-registered.
* Agreement signing is recorded with a timestamp and linked to the specific template version in use at the time.

---

## AGREEMENT_TEMPLATE

### Purpose

Configurable entry terms or NDA text that visitors sign on arrival, per office location.

### Stores

* `title`, `content` (full template text)
* `is_active` — only one active template per building at a time
* `created_by`

### Why it exists

* Legal requirements differ by country. Ghana, Germany, and the US may have different NDA templates.
* Versioning through `is_active` ensures a visit record's `agreement_template_id` always points to the exact wording the visitor signed.

---

# 8. Office Event Management

---

## EVENT_SERIES

### Purpose

Parent record for a recurring event (e.g., weekly all-hands, monthly town hall).

### Stores

* `organiser_id`, `building_id`
* `recurrence_type` — `DAILY`, `WEEKLY`, `MONTHLY`
* `recurrence_interval` — e.g. every 2 weeks
* `series_start_date` / `series_end_date`

### Why it exists

* A single approval and configuration covers all future event instances.
* Cancelling or editing a series propagates to all future events without touching past records.

---

## EVENT

### Purpose

Represents a single event instance — either a one-off event or a child of `EVENT_SERIES`.

### Stores

* `event_date`, `start_time`, `end_time`, `capacity`
* `room_reference` — informational room name (free text, not a strict FK, as events may use spaces not in the seat inventory)
* `status` — `SCHEDULED` or `CANCELLED`
* `is_series_detached` — true if this instance has been independently edited from its series

### Why it exists

* Tracks all office events for scheduling, RSVP collection, and capacity management.
* `is_series_detached` supports the standard "edit this event only / edit all future / edit series" pattern.

---

## EVENT_INVITE

### Purpose

Tracks invitations and RSVPs for an event — both internal employees and external visitors.

### Stores

* `employee_id` — set for internal attendees
* `visit_record_id` — set for external attendees (links to their Visitor Management record)
* `rsvp_status` — `PENDING`, `ATTENDING`, `NOT_ATTENDING`, `MAYBE`
* `is_waitlisted` / `waitlist_position` / `waitlist_joined_at` — for capacity-limited events

### Why it exists

* A single table covers both internal staff and external guests attending the same event.
* Waitlist logic is handled directly in the schema without a separate table.

---

# 9. Office Supplies Management

This domain tracks consumable supplies — stationery, cleaning products, kitchen items, first aid — from catalogue to request to reorder.

---

## SUPPLY_CATEGORY

### Purpose

Groups supply items into logical categories.

### Stores

* `name` (e.g. "Stationery", "Cleaning", "Kitchen", "First Aid")
* `has_expiry` — whether items in this category can expire (e.g. medicine vs. staples)

### Why it exists

* Enables category-level reporting ("how much is spent on stationery per office?").
* `has_expiry = true` triggers expiry-date fields and alerts on `SUPPLY_STOCK_ENTRY`.

---

## SUPPLY_ITEM

### Purpose

The supply catalogue — one row per distinct item per building.

### Stores

* `name`, `description`, `unit_of_measure` (e.g. "ream", "litre", "unit")
* `allocation_type` — `PERSONAL` (issued to an employee) or `SHARED` (communal stock)
* `current_stock`, `low_stock_threshold`, `reorder_quantity`

### Why it exists

* `current_stock` and `low_stock_threshold` drive automatic reorder alerts.
* Scoped to `building_id` — Accra and Kumasi maintain separate stock counts.

---

## SUPPLY_STOCK_ENTRY

### Purpose

Tracks individual stock batches, enabling per-batch expiry management.

### Stores

* `quantity`, `expiry_date` (nullable for non-perishable items), `expiry_alert_days`
* `status` — `ACTIVE`, `EXPIRED`, `DISPOSED`
* `disposed_at` / `disposed_by` — tracks who removed expired stock and when
* `received_at`

### Why it exists

* A single medicine stock might contain two batches with different expiry dates.
* Tracking batches separately prevents expired stock from being unknowingly used.
* Disposal creates an audit trail for health and safety compliance.

---

## SUPPLY_REQUEST

### Purpose

An employee's request for a supply item.

### Stores

* `supply_item_id`, `requester_id`, `quantity`, `purpose`
* Two-stage approval: `PENDING_MANAGER → PENDING_FACILITIES → APPROVED → FULFILLED`
* Separate reviewer and rejection-reason fields for each stage

### Why it exists

* Some items (e.g., a large stationery order) need both manager and facilities sign-off.
* Two-stage approval tracks where in the pipeline a request is and who is responsible.

---

## REORDER_LOG

### Purpose

Immutable record of every reorder trigger sent to the procurement system.

### Stores

* `triggered_by` — `AUTO` (system detected low stock) or `MANUAL` (admin initiated)
* `stock_level_at_trigger`, `reorder_quantity`
* `procurement_response` (JSONB) — full response from the procurement system
* `status` — `SENT`, `ACKNOWLEDGED`, `FAILED`

### Why it exists

* Provides a full audit trail of restocking activity.
* `procurement_response` in JSONB preserves whatever the external system returns, even if its schema changes.

---

# 10. Office Assets Management

This domain manages physical assets — laptops, monitors, chairs, access cards — from acquisition through assignment, maintenance, faults, and retirement.

---

## ASSET_CATEGORY

### Purpose

Groups assets by type.

### Stores

* `name` (e.g. "Laptop", "Monitor", "Chair", "Access Card")
* `default_lifespan_months` — used to schedule age-based retirement alerts

### Why it exists

* Category drives depreciation schedules and replacement planning.
* Asset requests reference a category, not a specific asset, allowing facilities to fulfil from available inventory.

---

## ASSET

### Purpose

Represents a single physical asset in the register.

### Stores

* `asset_tag` (unique internal identifier), `serial_number`, `name`
* `condition` — `NEW`, `GOOD`, `FAIR`, `POOR`
* `status` — `AVAILABLE`, `ASSIGNED`, `PENDING_ACKNOWLEDGEMENT`, `UNDER_REPAIR`, `RETIRED`, `WRITTEN_OFF`
* `purchase_date`, `purchase_value`, `warranty_expiry`, `next_service_date`
* `retired_at`, `retirement_reason` — `END_OF_LIFE`, `LOST`, `DAMAGED`, `DONATED`, `SOLD`
* `deleted_at` — soft delete when the asset record is archived

### Why it exists

* Central register of every piece of physical equipment in the company.
* Status transitions (`AVAILABLE → ASSIGNED → UNDER_REPAIR → RETIRED`) give a full lifecycle view.

---

## ASSET_ASSIGNMENT

### Purpose

Records the assignment of an asset to an employee or a shared space.

### Stores

* `assigned_to_employee_id` (nullable) — personal assignment
* `assigned_to_space` (string, nullable) — shared space assignment (e.g. "Kumasi - Floor 4 Printer Station")
* `acknowledged_at` — set when the employee confirms receipt
* `returned_at`, `return_condition`
* `asset_request_id` — set if this assignment originated from a formal request
* `is_active` — false when the asset has been returned

### Why it exists

* Maintains a clear chain of custody. At any point, you can query who holds which asset.
* `acknowledged_at` protects the company — an employee cannot claim they never received a laptop if the record shows they signed for it.

---

## ASSET_REQUEST

### Purpose

An employee-initiated request for an asset (e.g. a new laptop or a second monitor).

### Stores

* `category_id` — what type of asset is requested
* `justification`
* Same two-stage approval pattern as `SUPPLY_REQUEST` (Manager → Facilities)

### Why it exists

* Provides a formal request trail instead of informal messages to IT or facilities.
* When approved, the request is linked to the resulting `ASSET_ASSIGNMENT` via `asset_request_id`.

---

## FAULT_REPORT

### Purpose

An employee-submitted report of a lost, damaged, or malfunctioning asset.

### Stores

* `fault_type` — `LOST`, `DAMAGED`, `MALFUNCTIONING`
* `status` — `OPEN`, `UNDER_INVESTIGATION`, `RESOLVED`, `WRITTEN_OFF`
* `resolution_outcome` — `REPAIRED` or `WRITTEN_OFF`
* `resolved_by`, `resolved_at`

### Why it exists

* Creates a formal record the moment a fault is identified.
* A fault report is the trigger for creating a `MAINTENANCE_RECORD`; linking both preserves the full investigation and resolution trail against the asset.

---

## MAINTENANCE_RECORD

### Purpose

Logs a maintenance or repair event carried out on an asset.

### Stores

* `maintenance_type` — `SCHEDULED_SERVICE`, `REPAIR`, `INSPECTION`, `FAULT_RESOLUTION`
* `performed_by_type` — `INTERNAL` or `EXTERNAL_VENDOR`
* `outcome` — `RESOLVED`, `ONGOING`, `WRITE_OFF_RECOMMENDED`
* `maintenance_date`, `next_service_date`, `cost`
* `fault_report_id` — nullable FK to the fault that triggered this maintenance

### Why it exists

* `WRITE_OFF_RECOMMENDED` outcome feeds back to the asset's status lifecycle.
* Scheduled service records exist independently of fault reports — a laptop can have a maintenance record with no associated fault.
* `cost` enables total cost-of-ownership analysis per asset over its lifespan.

---

### Asset lifecycle flow

```text
ASSET created
    │
    ├── ASSET_REQUEST (employee requests it)
    │         │
    │         └── ASSET_ASSIGNMENT (fulfilled, acknowledged)
    │
    ├── FAULT_REPORT (problem reported)
    │         │
    │         └── MAINTENANCE_RECORD (repair/investigation)
    │                   │
    │                   └── outcome: WRITE_OFF_RECOMMENDED
    │                                      │
    │                                      ▼
    └──────────────────────────── ASSET.status = WRITTEN_OFF
```

---

# 11. Global: Audit & Notifications

---

## AUDIT_LOG

### Purpose

Immutable record of every significant action taken in the system.

### Stores

* `actor_id`, `actor_role` — who did it and in what capacity
* `action` — e.g. `SEAT_BOOKED`, `REQUEST_APPROVED`, `ASSET_ASSIGNED`
* `entity_type` / `entity_id` — which record was affected
* `previous_state` / `new_state` (JSONB) — the full before and after snapshot
* `occurred_at`
* `building_id` — location context of the action

### Why it exists

* Provides a tamper-evident trail for compliance, debugging, and incident investigation.
* JSONB state snapshots mean even if the entity is later edited, the log shows exactly what the record looked like at the time of each action.
* Retained for a minimum of 24 months. Never soft-deleted.

---

## NOTIFICATION

### Purpose

Stores in-app notifications delivered to employees.

### Stores

* `recipient_id`, `type`, `title`, `body`
* `deep_link` — URL to the relevant record in the application
* `is_read`

### Why it exists

* Centralises notification state — the app reads from this table to badge unread counts and render the notification panel.
* Email notifications are handled by the email provider and are not stored here.

---

# 12. End-to-End System Flows

### Daily attendance processing

```text
1. Employee badges in
        → BADGE_EVENT (raw, immutable)
2. Session resolver groups events
        → WORK_SESSION (is_late computed against LOCATION_CONFIG)
3. Attendance engine runs per employee per day
        → checks: REMOTE_REQUEST (approved?), OOO_REQUEST (approved?), PUBLIC_HOLIDAY (date match?)
        → writes: ATTENDANCE_RECORD (one row, one status)
4. Seat occupancy engine runs
        → cross-references: SEAT_BOOKING, WORK_SCHEDULE, ATTENDANCE_RECORD
        → writes: SEAT_OCCUPANCY_LOG (is_occupied, is_remote_day)
5. No-show release job runs at LOCATION_CONFIG.no_show_release_time
        → updates: SEAT_BOOKING.status = RELEASED
        → writes: NO_SHOW_RECORD
```

### Remote day approval

```text
Employee submits REMOTE_REQUEST
        → REMOTE_DAY_POLICY checked (HARD_BLOCK: reject | SOFT_WARNING: flag)
        → APPROVAL_DELEGATE checked (if manager has active OOO, delegate receives it)
        → Manager/delegate approves
        → ATTENDANCE_RECORD.status = REMOTE for that date
        → SEAT_OCCUPANCY_LOG.is_remote_day = true for assigned seat
```

### Visitor arrival

```text
Visitor pre-registered → VISITOR_PROFILE + PARENT_VISIT + VISIT_RECORD (status: EXPECTED)
Visitor arrives at reception
        → VISIT_RECORD.actual_check_in set (status: CHECKED_IN)
        → AGREEMENT_TEMPLATE presented and signed
        → VISIT_RECORD.agreement_signed = true
Visitor attends event → EVENT_INVITE.visit_record_id linked
Visitor leaves → VISIT_RECORD.actual_check_out set (status: CHECKED_OUT)
```

### Asset fault resolution

```text
Employee reports problem → FAULT_REPORT (status: OPEN)
Facilities investigates → FAULT_REPORT (status: UNDER_INVESTIGATION)
Repair carried out → MAINTENANCE_RECORD (fault_report_id linked)
        Outcome: RESOLVED → FAULT_REPORT (status: RESOLVED), ASSET.condition updated
        Outcome: WRITE_OFF_RECOMMENDED → ASSET.status = WRITTEN_OFF
```

---

# 13. Why This Model Works Globally

| Principle | How it is implemented |
|---|---|
| **Location-aware** | `building_id` on every non-global entity; data is always isolated per office |
| **Policy-driven** | `LOCATION_CONFIG` gives each office its own attendance rules; `REMOTE_DAY_POLICY` gives each team its own remote limits |
| **Timezone-safe** | `COUNTRY.timezone` ensures all timestamp comparisons are made in the correct local time |
| **Role-based access** | `USER_ROLE` is scoped per building — a manager's approval authority is limited to their office |
| **Hybrid work ready** | `REMOTE_REQUEST`, `RECURRING_REMOTE_SCHEDULE`, `WORK_SCHEDULE.is_remote`, and `SEAT_OCCUPANCY_LOG.is_remote_day` fully model remote days at every level |
| **Audit-complete** | Immutable `BADGE_EVENT`, `NO_SHOW_RECORD`, `REORDER_LOG`, and `AUDIT_LOG` mean nothing is silently changed |
| **Soft-delete everywhere** | `deleted_at` on `EMPLOYEE`, `SEAT`, `VISITOR_PROFILE`, and `ASSET` preserves history on offboarding or decommission |
| **Extensible** | JSONB fields on `BADGE_EVENT.raw_payload`, `AUDIT_LOG.previous_state / new_state`, and `REORDER_LOG.procurement_response` absorb schema changes in external systems without a migration |

---
