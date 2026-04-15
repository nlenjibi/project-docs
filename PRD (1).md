# Product Requirements Document (PRD)
## Office Management System (OMS)

**Version:** 2.0 — Microservices Edition  
**Status:** Draft  
**Last Updated:** March 2026  
**Author:** Paul Mensah  

---

## 1. Executive Summary

The Office Management System (OMS) is an internal enterprise platform built as a **distributed microservices system** serving 2–5 office locations. It unifies attendance management, seat booking, remote day scheduling, visitor management, event management, supply tracking, and asset management into a single authoritative, location-aware platform.

The system is composed of **11 independently deployable services**, each owning its business capability and its database. All external traffic enters through an API Gateway. Services communicate synchronously via REST (with circuit breakers) for user-facing queries, and asynchronously via Apache Kafka for state changes and cross-service workflows.

**Core Principles:**
- **Transparency** — every stakeholder sees exactly the data relevant to them, no more and no less
- **Auditability** — every action is traceable to a person, a time, and a reason via an immutable, event-driven audit trail
- **Configurability** — policies differ across offices and teams; the system accommodates this without code changes
- **Independence** — every service is deployable, scalable, and maintainable in isolation

---

## 2. Problem Statement

The organisation currently manages attendance, desk booking, visitor access, supply requests, and asset assignments through spreadsheets, email chains, and disconnected tools. This creates:

- No single source of truth for employee presence, seating, or asset ownership
- Compliance risk from unauditable, manual record-keeping
- Poor employee experience due to friction in basic self-service tasks
- Lack of visibility for managers and HR into team location and availability
- Operational burden on Facilities Admins from uncoordinated manual processes

The OMS solves this with a platform where every action is traceable, every workflow is automated, and every service can evolve independently without impacting the rest of the system.

---

## 3. Goals and Success Metrics

| Goal | Success Metric |
|------|---------------|
| Replace manual attendance tracking | 100% of attendance records auto-generated via nightly badge sync |
| Audit every action | All state changes logged immutably via the audit event pipeline |
| Real-time desk availability | Seat availability query latency < 200ms under normal load |
| Reduce supply request turnaround | Average request-to-fulfilment time < 3 business days |
| Support multi-location operations | System live across 2–5 locations with independently configured policies |
| Independent service deployability | Any single service deployable without downtime to other services |
| Resilient to service failures | No single service failure causes a full system outage |

---

## 4. Users and Roles

Roles are **location-scoped** — a user may hold different roles in different offices. Role enforcement is applied at both the API Gateway and within each individual service.

| Role | Description |
|------|-------------|
| **Employee** | Standard user. Self-service access to their own data and requests. |
| **Manager** | All Employee capabilities plus visibility into direct reports and approval authority over team requests. |
| **HR** | Org-wide visibility within their location. Manages employee lifecycle events. Read access across all features. |
| **Facilities Admin** | Manages physical space configuration, supplies inventory, asset register, and visitor reception. Fulfilment authority across operational features. |
| **Super Admin** | Full system configuration access across all locations. Manages office setup, system-wide policies, and user role assignments. |

---

## 5. System Services Overview

The OMS is delivered as 11 microservices. Each service maps to one bounded context.

| Service | Business Capability | Phase |
|---------|-------------------|-------|
| `identity-service` | SSO integration, token verification, session management | 1 |
| `user-service` | User profiles, role assignments, HR sync, location config | 1 |
| `attendance-service` | Badge ingestion, WorkSession resolution, AttendanceRecord management | 1 |
| `seating-service` | Floor plans, seat booking, no-show auto-release | 1 |
| `remote-service` | Remote/OOO requests, approval workflow, policy enforcement | 1 |
| `notification-service` | In-app and email notification dispatch | 1 |
| `audit-service` | Immutable audit log ingestion and query | 1 |
| `visitor-service` | Visitor lifecycle, check-in/out, agreement templates | 2 |
| `event-service` | Office events, RSVP, waitlist, recurring events | 2 |
| `supplies-service` | Supply catalogue, batch stock management, procurement reorder | 2 |
| `assets-service` | Asset register, assignments, fault reports, maintenance | 2 |

---

## 6. Scope

### 6.1 Phase 1 — Core Services

#### 6.1.1 Identity Service
SSO-based authentication. Server-side token verification. HTTP-only session cookie issuance. No local credential management.

**User Stories:**
- As any user, I want to log in via SSO so I can access the OMS without managing a separate password.
- As any user, I want my session to expire after a configured period so that stale sessions cannot be exploited.

**Functional Requirements:**

| ID | Requirement |
|----|-------------|
| IDN-01 | Authentication is handled exclusively via the organisation's SSO provider (OAuth 2.0 / OIDC) |
| IDN-02 | Token verification is performed entirely server-side; the frontend never handles raw tokens |
| IDN-03 | The backend issues an HTTP-only session cookie to the client after successful token verification |
| IDN-04 | Sessions have a configurable expiry and are invalidated on logout |
| IDN-05 | The `identity-service` exposes a `/validate` endpoint consumed internally by the API Gateway on every inbound request |

---

#### 6.1.2 User Service
Authoritative store for user profiles, per-location role assignments, and location configuration. Syncs from the Arms HR system.

**User Stories:**
- As a **Super Admin**, I want to assign and revoke roles per user per location.
- As a **Super Admin**, I want to configure office hours, lateness thresholds, and booking policies per location.

**Functional Requirements:**

| ID | Requirement |
|----|-------------|
| USR-01 | User records are synced from the Arms HR system via a scheduled job; OMS is not the source of truth for employees |
| USR-02 | Roles are assigned per user per location (a user may hold different roles in different offices) |
| USR-03 | Super Admin has a NULL location on their UserRole — the only cross-location role |
| USR-04 | `user-service` publishes `user.created`, `user.updated`, and `user.deactivated` events to Kafka |
| USR-05 | Location configuration (office hours, thresholds, booking policies) is managed here and read by other services via REST |

---

#### 6.1.3 Attendance Service
Nightly badge event ingestion from AWS Athena, WorkSession resolution, two-pass AttendanceRecord computation.

**User Stories:**
- As an **Employee**, I want to view my attendance history so I can verify my records.
- As a **Manager**, I want to view my team's attendance to identify absence patterns.
- As a **Super Admin**, I want to override an incorrect attendance record with a documented reason.

**Functional Requirements:**

| ID | Requirement |
|----|-------------|
| ATT-01 | Badge events are ingested from AWS Athena via a nightly scheduled sync job |
| ATT-02 | Badge events are resolved into WorkSessions using a configurable session gap threshold |
| ATT-03 | The sync job runs a two-pass resolution: Pass 1 (badge → WorkSession → AttendanceRecord), Pass 2 (leave/remote/holiday overlay via Kafka events consumed from `remote-service`) |
| ATT-04 | Any user-date with no badge event and no leave/remote/holiday overlay is stamped ABSENT |
| ATT-05 | WorkSessions handle cross-midnight shifts — a session is owned by its start date |
| ATT-06 | Attendance history begins at `employment_start_date` from the HR sync; no records before this date |
| ATT-07 | The service consumes `remote.request.approved` and `ooo.request.approved` Kafka events to overlay REMOTE/OOO statuses |
| ATT-08 | Managers can export attendance reports; HR can export org-wide reports |
| ATT-09 | Super Admin can override any attendance record with a mandatory reason |
| ATT-10 | The service publishes `attendance.no_show.detected` events consumed by `seating-service` |

---

#### 6.1.4 Seating Service
Hot-desk booking with real-time availability, floor plan management, block bookings, permanent seat assignment, and no-show auto-release.

**User Stories:**
- As an **Employee**, I want to view the floor plan and book an available hot-desk for a specific date.
- As a **Manager**, I want to block-book seats for my team ahead of a collaboration day.
- As a **Facilities Admin**, I want to configure the floor plan and no-show release policy.

**Functional Requirements:**

| ID | Requirement |
|----|-------------|
| SEAT-01 | The service provides an interactive floor plan with real-time seat availability |
| SEAT-02 | Seat availability is computed in real-time from `SeatBooking` records — no pre-computed cache |
| SEAT-03 | Employees can book a hot-desk within the configurable booking window |
| SEAT-04 | Employees can cancel their own booking up to the configurable cancellation cutoff |
| SEAT-05 | Managers can block-book seats for their team |
| SEAT-06 | HR and Facilities Admin can assign permanent seats |
| SEAT-07 | A scheduled no-show release job runs daily; it consumes `attendance.no_show.detected` events and releases unattended bookings |
| SEAT-08 | Released bookings create a `NoShowRecord` and publish `seat.booking.released` for `notification-service` |
| SEAT-09 | Floor plan visibility mode (FULL / TEAM_ONLY / AVAILABILITY_ONLY) is read from `user-service` location config |

---

#### 6.1.5 Remote Service
Remote day and OOO request workflow with configurable per-team policy enforcement and manager delegation.

**User Stories:**
- As an **Employee**, I want to submit a remote day or OOO request.
- As a **Manager**, I want to approve or reject my team's requests with a reason.
- As a **Manager**, I want to nominate a delegate when I go OOO so requests are never blocked.

**Functional Requirements:**

| ID | Requirement |
|----|-------------|
| REM-01 | Employees can submit remote day and OOO requests |
| REM-02 | Managers approve or reject requests; rejections require a mandatory reason |
| REM-03 | Remote day limits are configurable per team or per location (default) |
| REM-04 | Enforcement mode is HARD_BLOCK (reject submission) or SOFT_WARNING (allow with warning) |
| REM-05 | Managers configure a team overlap threshold (max % of team remote on the same day) |
| REM-06 | Recurring remote schedules use the parent/child pattern |
| REM-07 | Managers must nominate a delegate as part of their OOO submission |
| REM-08 | If no delegate is nominated, HR at that location becomes the fallback approver |
| REM-09 | On approval, the service publishes `remote.request.approved` and `ooo.request.approved` Kafka events consumed by `attendance-service` and `notification-service` |

---

#### 6.1.6 Notification Service
Event-driven notification dispatch. Consumes Kafka events. Dispatches in-app and email notifications. Not called directly by any other service.

**Functional Requirements:**

| ID | Requirement |
|----|-------------|
| NOTIF-01 | The notification service is a Kafka consumer only — never called directly by other services |
| NOTIF-02 | Notifications are dispatched for all significant state changes across all services |
| NOTIF-03 | In-app notifications are stored in `notification_db` and exposed via a read API to the frontend |
| NOTIF-04 | Email notifications are dispatched via the configured email provider |
| NOTIF-05 | All notifications include a deep link to the relevant record |
| NOTIF-06 | Employees can view, read, and dismiss their in-app notifications |

---

#### 6.1.7 Audit Service
Immutable, append-only audit log. Kafka-driven writes. Read-only query API for authorised roles.

**Functional Requirements:**

| ID | Requirement |
|----|-------------|
| AUD-01 | Every service publishes an `audit.event` Kafka message for every create, update, delete, or status change |
| AUD-02 | The audit service is Kafka-only for writes — no service can call a write API directly |
| AUD-03 | Audit records include: actor ID, actor role, action, entity type, entity ID, location context, previous state (JSONB), new state (JSONB), timestamp |
| AUD-04 | Audit records are never updated or deleted — the database user has no UPDATE or DELETE privileges |
| AUD-05 | Audit logs are retained for a minimum of 24 months; older records are archived to cold storage (S3) |
| AUD-06 | HR, Facilities Admin, and Super Admin can query audit logs filtered by actor, entity type, and location |

---

### 6.2 Phase 2 — Secondary Services

#### 6.2.1 Visitor Service

| ID | Requirement |
|----|-------------|
| VIS-01 | Employees can pre-register expected visitors |
| VIS-02 | Facilities Admin can register walk-in visitors |
| VIS-03 | Facilities Admin performs check-in/check-out; `visitor.checked_in` Kafka event triggers host notification |
| VIS-04 | Agreement signing is recorded at the visit level capturing the exact template version |
| VIS-05 | Returning visitors must re-sign when the active template has changed |
| VIS-06 | Full visitor audit log accessible to HR, Facilities Admin, and Super Admin |

---

#### 6.2.2 Event Service

| ID | Requirement |
|----|-------------|
| EVT-01 | Managers, HR, and Facilities Admin can create events with capacity |
| EVT-02 | All employees can view and RSVP to events; waitlist is automatically managed |
| EVT-03 | Waitlist promotions publish `event.rsvp.promoted`; `notification-service` dispatches the alert |
| EVT-04 | Recurring events follow the EventSeries/Event parent/child pattern supporting three edit scopes |
| EVT-05 | Event cancellation publishes `event.cancelled`; `notification-service` alerts all attendees |

---

#### 6.2.3 Supplies Service

| ID | Requirement |
|----|-------------|
| SUP-01 | Supply requests follow a two-stage Choreography-based Saga: Manager (stage 1) → Facilities Admin (stage 2) |
| SUP-02 | Stock is tracked at batch level with per-batch expiry; fulfilment uses FIFO |
| SUP-03 | Low stock triggers `supply.reorder.triggered` Kafka event; procurement integration consumes it (AD-026) |

---

#### 6.2.4 Assets Service

| ID | Requirement |
|----|-------------|
| ASSET-01 | Asset requests follow a two-stage Choreography-based Saga: Manager approval → Facilities Admin fulfilment |
| ASSET-02 | Employees submit fault reports; `asset.fault_reported` event triggers Facilities Admin notification |
| ASSET-03 | All asset lifecycle events (assigned, returned, retired, maintained) are published to the audit pipeline |

---

## 7. Cross-Cutting Requirements

### 7.1 Event-Driven Architecture
All cross-service workflows use Kafka. Services never call `notification-service` or `audit-service` directly — they publish events. This ensures loose coupling and independent scalability.

### 7.2 Resilience on Every REST Call
Every synchronous inter-service REST call implements a circuit breaker, bounded retry with exponential backoff, a 3-second timeout, and a defined fallback response.

### 7.3 Observability on Every Service
Every service implements structured JSON logging (stdout), Prometheus metrics, OpenTelemetry distributed tracing with correlation IDs, and liveness/readiness health probes.

### 7.4 Multi-Location Data Isolation
All feature data is location-scoped at the schema level. Every API query filters by the authenticated user's location context. Super Admin is the sole exception.

### 7.5 Internationalisation
Language preference is set per user in `user-service`. All UI strings are externalised. Date, time, and number formats follow the user's locale.

---

## 8. Non-Functional Requirements

| Category | Requirement |
|----------|-------------|
| **Deployability** | Each service independently deployable with zero downtime to other services |
| **Scalability** | Each service scales horizontally via Kubernetes HPA |
| **Authentication** | SSO only via OAuth 2.0 / OIDC; managed by `identity-service` |
| **Authorisation** | Role + location checks enforced at API Gateway AND within each service |
| **Internal security** | mTLS between all services; internal JWTs verified per request (Zero Trust) |
| **Data isolation** | Database per service; no shared schemas; no cross-service DB access |
| **SQL safety** | Parameterised queries only; no string concatenation in any service |
| **Resilience** | Circuit breaker, retry, timeout, fallback on every inter-service REST call |
| **Idempotency** | All Kafka consumers idempotent; duplicate delivery is safe |
| **Audit retention** | Minimum 24 months; archived to S3 thereafter |

---

## 9. System Integrations

| System | Consuming Service | Purpose | Method |
|--------|------------------|---------|--------|
| Door Access Hardware / AWS Athena | `attendance-service` | Badge event ingestion | Nightly Athena query |
| HR System (Arms) | `user-service` | Employee records, org structure | Scheduled API sync |
| SSO Provider | `identity-service` | User identity and login | OAuth 2.0 / OIDC |
| Email Provider | `notification-service` | Outbound email notifications | SMTP or REST API (TBD) |
| Procurement System | `supplies-service` | Low-stock reorder trigger | REST API (contract TBD — AD-026) |

---

## 10. Tech Stack

| Layer | Technology |
|-------|------------|
| Frontend | Angular (TypeScript) |
| Backend (all services) | Java 21, Spring Boot |
| API Gateway | AWS API Gateway or Kong |
| Message broker | Apache Kafka (AWS MSK) |
| Databases (per service) | PostgreSQL (AWS RDS) |
| Authentication | SSO via OAuth 2.0 / OIDC |
| Containerisation | Docker |
| Orchestration | Kubernetes (AWS EKS) |
| Internal security | mTLS + internal JWT (Istio or AWS App Mesh) |
| Secrets management | AWS Secrets Manager |
| Observability | Prometheus + Grafana · OpenTelemetry + Jaeger · CloudWatch Logs |

---

## 11. Out of Scope (Launch)

- Payroll system integration
- Leave balance management
- Room booking / calendar system integration
- Mobile native application (web-responsive at launch)
- Real-time badge event streaming (nightly batch only)
- Self-service role management by non-Super Admins
- Public holiday calendar integration (manually configured per location)
- CQRS (can be introduced per service if read/write traffic diverges)
- GraphQL (REST at launch)

---

## 12. Open Questions

| ID | Question | Owner |
|----|----------|-------|
| OQ-01 | Which email provider will `notification-service` use? | Infrastructure team |
| OQ-02 | Procurement system API contract for reorder triggers? | Procurement team (AD-026) |
| OQ-03 | SSO provider — confirm JWKS endpoint and OIDC compliance | IT/Security team |
| OQ-04 | API Gateway: AWS API Gateway vs Kong? | Architecture team |
| OQ-05 | Service mesh: Istio vs AWS App Mesh for mTLS? | Infrastructure team |

---

*End of Product Requirements Document*
