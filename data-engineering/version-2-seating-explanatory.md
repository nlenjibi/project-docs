
---
# Office Management System (OMS) – Data Model Documentation

## 1. Overview

This data model supports a **multi-location attendance and workplace management system**.
It is designed to:

* Track employee attendance using badge events
* Support hybrid work (office, remote, OOO)
* Manage office seating and space utilization
* Enforce location-specific policies
* Provide auditability and analytics

The schema is structured into **six functional domains**:

1. Location & Configuration
2. Physical Workspace
3. Employee & Access Control
4. Work Planning
5. Seating & Occupancy
6. Attendance Processing

---

#  2. Location & Configuration Domain

These tables enable **global, multi-office support**.

---

## COUNTRY

### Purpose

Defines the top-level geographic grouping.

### Stores

* Country name and code
* Timezone configuration

### Why it exists

* Ensures **timezone-aware attendance tracking**
* Enables global reporting (e.g., attendance by country)

---

## CITY

### Purpose

Represents cities within a country.

### Stores

* City name
* Reference to country

### Why it exists

* Organizes offices geographically
* Supports regional analytics and filtering

---

## OFFICE_BUILDING

### Purpose

Represents a physical office location.

### Stores

* Office name and address
* Number of floors
* Associated city

### Why it exists

* Core unit of **attendance tracking and policy enforcement**
* All attendance events are tied to a building

---

## LOCATION_CONFIG

### Purpose

Defines **attendance and operational rules per location**

### Stores

* Work start time
* Lateness threshold
* Minimum presence duration
* Booking and cancellation rules

### Why it exists (CRITICAL)

This table controls how attendance is interpreted:

* Determines if an employee is **late**
* Defines **what qualifies as “present”**
* Controls **seat booking policies**

Without this, badge data has **no business meaning**

---

# 3. Physical Workspace Domain

These tables model the office layout.

---

## FLOOR

### Purpose

Represents floors within a building

### Stores

* Floor number and name
* Active status

### Why it exists

* Supports navigation and floor-level analytics
* Enables floor-based access and management

---

## ROOM

### Purpose

Represents rooms or zones within a floor

### Stores

* Room name
* Room type (e.g., workspace, meeting room)
* Seat capacity

### Why it exists

* Groups seats logically
* Enables room-level analytics and reservations

---

## SEAT

### Purpose

Represents individual workspaces

### Stores

* Seat identifier and type
* Status (available, assigned, etc.)
* Coordinates for floor plan visualization

### Why it exists

* Enables **seat booking and assignment**
* Supports **occupancy tracking**
* Links physical presence to a location

---

#  4. Employee & Access Control Domain

---

## EMPLOYEE

### Purpose

Stores employee master data

### Stores

* Identity (name, email, employee code)
* Organizational data (department, manager)
* Employment lifecycle

### Why it exists

* Central entity for all system operations
* Links attendance, bookings, and requests to a person

---

## USER_ROLE

### Purpose

Defines user roles per location

### Stores

* Role type (Employee, Manager, HR, etc.)
* Associated building

### Why it exists

* Enables **location-scoped permissions**
* Supports different roles across offices

Example:
An employee can be:

* Manager in Accra
* Regular employee in Lagos

---

# 5. Work Planning Domain

These tables define **expected behavior vs actual behavior**

---

## WORK_SCHEDULE

### Purpose

Defines planned office attendance

### Stores

* Assigned days of work
* Remote vs in-office status
* Optional seat assignment

### Why it exists

* Determines **expected attendance**
* Enables comparison with actual badge data

---

## REMOTE_REQUEST

### Purpose

Tracks ad-hoc remote work requests

### Stores

* Request date and type
* Approval status

### Why it exists

* Explains absence from office
* Prevents false “absence” flags

---

## RECURRING_REMOTE_SCHEDULE

### Purpose

Defines recurring remote work patterns

### Stores

* Weekly remote schedule (e.g., every Friday)

### Why it exists

* Supports hybrid work policies
* Reduces repeated manual requests

---

## OOO_REQUEST (Out-of-Office)

### Purpose

Tracks leave and absence requests

### Stores

* Leave period
* Approval status

### Why it exists

* Differentiates between:

  * Absence
  * Approved leave
* Critical for accurate attendance reporting

---

#  6. Seating & Occupancy Domain

---

## SEAT_BOOKING

### Purpose

Tracks seat reservations

### Stores

* Booking date
* Status (active, cancelled, auto-released)

### Why it exists

* Enables hot-desking
* Detects **no-shows**

---

## BLOCK_RESERVATION

### Purpose

Allows bulk seat reservations

### Stores

* Reserved seat count
* Target room
* Manager responsible

### Why it exists

* Supports team bookings
* Improves space planning

---

## SEAT_OCCUPANCY_LOG

### Purpose

Tracks actual seat usage

### Stores

* Whether seat was occupied
* Associated employee

### Why it exists

* Validates physical presence
* Supports:

  * utilization analytics
  * capacity planning

---

#  7. Attendance Processing Domain (Core)

---

## BADGE_EVENT  (Raw Data Layer)

### Purpose

Stores raw badge data from access control systems

### Stores

* Employee ID
* Timestamp
* Event type (entry/exit)
* Raw device payload

### Why it exists

* Acts as the **source of truth**
* Captures all physical access events

This is your **Bronze layer**

---

## WORK_SESSION  (Processed Attendance)

### Purpose

Represents a complete workday session

### Stores

* First entry time
* Last exit time
* Total duration
* Lateness status

### Why it exists

Transforms raw badge events into **business-readable attendance**

This is your **Silver/Gold layer**

---

## How They Work Together

```text
BADGE_EVENT → processed → WORK_SESSION
```

---

#  8. Operational Support

---

## FLOOR_LEAD

### Purpose

Assigns responsibility for specific areas

### Stores

* Employee assigned to a room
* Assignment duration

### Why it exists

* Supports facilities management
* Enables accountability for spaces

---

# 🔗 9. End-to-End Attendance Flow

1. Employee badges in → stored in `BADGE_EVENT`
2. System processes events using:

   * `LOCATION_CONFIG`
   * `WORK_SCHEDULE`
   * `REMOTE_REQUEST` / `OOO_REQUEST`
3. System generates → `WORK_SESSION`
4. Data is enriched with:

   * Employee info
   * Location info
5. Final outputs:

   * Attendance reports
   * Late arrivals
   * No-show detection

---

#  10. Why This Model Works Globally

### Location-aware

* Every entity is tied to a building

### Policy-driven

* Each office has its own rules (`LOCATION_CONFIG`)

### Timezone-safe

* Country-level timezone support

###  Role-based access

* `USER_ROLE` enforces per-location permissions

###  Hybrid work ready

* Remote + OOO fully integrated

---
