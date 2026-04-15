# Solution Architecture Document
## Office Management System (OMS) — Microservices Edition

**Version:** 2.0  
**Status:** Draft  
**Last Updated:** March 2026  
**Author:** Paul Mensah  

> This document applies the **5-Layer Excellence Model** to the OMS microservices solution:
> 1. Business Architecture (WHY)
> 2. Capability Architecture (WHAT)
> 3. Logical Architecture (HOW – abstract)
> 4. Physical Architecture (TECHNOLOGY)
> 5. Execution Layer (IMPLEMENTATION & OPERATIONS)

---

## 1. Problem Understanding and Business Context

### 1.1 Formal Problem Statement

Organisations operating across multiple office locations currently manage employee attendance, workspace allocation, visitor access, supply inventories, and asset lifecycles through a combination of spreadsheets, email chains, and disconnected tools. This introduces operational risk, compliance gaps, and a poor employee experience.

The OMS replaces this fragmented landscape with a **distributed microservices platform** — 11 independently deployable services providing a unified, authoritative, location-aware system for all workplace operations. The microservices design ensures that no single capability's complexity, failure, or scaling requirement imposes a burden on any other.

### 1.2 Business Objectives

| Objective | Success Metric |
|-----------|---------------|
| Replace manual attendance tracking | 100% of attendance records auto-generated via nightly badge sync |
| Ensure compliance auditability | Every state change logged immutably via the event-driven audit pipeline |
| Improve management visibility | Managers access real-time team location and availability |
| Reduce operational overhead | Supply and asset workflows tracked end-to-end with zero manual reconciliation |
| Support multi-location growth | Designed for 2–5 locations at launch; each service scales independently |
| Enable independent evolution | Any service can be deployed, updated, or scaled without downtime to others |

### 1.3 Key Stakeholders

| Stakeholder | Role | Key Concern |
|-------------|------|-------------|
| Employees | End users, self-service | Ease of use, transparency of own data |
| Managers | Approvers, visibility | Team location visibility, approval efficiency |
| HR | Org-wide visibility | Compliance, reporting, offboarding accuracy |
| Facilities Admin | Operations | Inventory accuracy, visitor security, desk utilisation |
| Super Admin | Governance | Policy correctness, cross-location oversight |
| Engineering Team | Build and maintain | Service independence, clear contracts, testability |
| IT/Security | Infrastructure | SSO integration, Zero Trust, encryption |

### 1.4 Constraints

| Constraint | Detail |
|-----------|--------|
| Technology | Angular frontend, Java/Spring Boot per service — mandated by existing team expertise |
| Authentication | Must integrate with the existing SSO provider (OAuth 2.0 / OIDC) |
| Employee data | Arms HR system is the source of truth — `user-service` syncs from it |
| Badge data | Ingested from AWS Athena (nightly batch) — real-time streaming deferred |
| Database | PostgreSQL per service — confirmed during architecture phase |
| Procurement integration | API contract pending discovery session (AD-026) |

### 1.5 Assumptions

- The SSO provider supports OAuth 2.0 / OIDC and exposes a JWKS endpoint.
- The Arms HR API exposes employee identity, start dates, manager relationships, and org structure.
- AWS Athena contains badge event data with a `personnel_id` field mappable to OMS users.
- At launch, 2–5 locations with hundreds of employees — each service can be scaled to meet demand independently.
- The organisation can operate and monitor 11 services via a shared observability stack.

---

## 2. Stakeholder and Requirement Analysis

### 2.1 Functional Requirements (Service-Mapped)

| Service | Core Capability |
|---------|----------------|
| `identity-service` | SSO auth, token verification, session lifecycle |
| `user-service` | User profiles, role assignments, location config, HR sync |
| `attendance-service` | Badge ingestion, WorkSession resolution, two-pass AttendanceRecord |
| `seating-service` | Floor plan, hot-desk booking, block booking, no-show auto-release |
| `remote-service` | Remote/OOO requests, approval workflow, policy enforcement, delegation |
| `notification-service` | Event-driven in-app and email dispatch |
| `audit-service` | Immutable append-only audit log |
| `visitor-service` (P2) | Visitor lifecycle, check-in/out, agreement versioning |
| `event-service` (P2) | Events, RSVP, waitlist, recurring events |
| `supplies-service` (P2) | Supply catalogue, batch stock, two-stage Saga, reorder |
| `assets-service` (P2) | Asset register, assignment lifecycle, fault, maintenance, retirement |

### 2.2 Non-Functional Requirements

| Category | Requirement | Priority |
|----------|-------------|----------|
| Service independence | Deploy any service without downtime to others | Critical |
| Zero Trust security | mTLS + internal JWT on every service call | Critical |
| Data isolation | Database per service; no cross-service DB access | Critical |
| Audit completeness | Every mutation produces an immutable `audit.event` | Critical |
| Resilience | Circuit breaker, retry, timeout, fallback on all REST calls | Critical |
| Idempotency | All Kafka consumers safe under duplicate delivery | Critical |
| Observability | Logs, metrics, traces, health probes on every service | High |
| Scalability | Horizontal scaling per service via Kubernetes HPA | High |
| Location scoping | Schema-level data isolation between offices | High |
| API versioning | All APIs versioned from day one (`/api/v1/`) | High |

### 2.3 Prioritised Requirements

**Critical (must be present before any service reaches production):**
- Service mesh (mTLS) operational — no service is deployed without it
- Internal JWT validation on every service
- Audit event pipeline live — `audit-service` consuming before business services launch
- Parameterised SQL — enforced in code review; no exceptions

---

## 3. Business and Outcome Architecture (WHY Layer)

### 3.1 Why Microservices?

The microservices architecture is not chosen for novelty. It is chosen because the OMS has **11 distinct business capabilities** that have different:
- **Scaling requirements** — `seating-service` handles concurrent real-time bookings; `audit-service` handles high-volume event ingestion; `attendance-service` runs a heavy nightly batch job. These workloads should not compete for resources.
- **Deployment cadences** — the team managing supply requests should not be blocked by a deployment to the attendance pipeline.
- **Failure domains** — a failure in `visitor-service` must not affect seat booking or attendance tracking.
- **Data models** — each domain has fundamentally different data relationships and access patterns.

### 3.2 Business Value Delivered

```
Badge Events (Athena) + HR Data (Arms) + SSO Identity
                    ↓
            API Gateway (single entry)
                    ↓
      ┌─────────────┬──────────────────────────────┐
      │  Employee    │  Manager     │  Facilities / HR│
      │  Self-service│  Visibility  │  Operations     │
      │  (11 services│  + Approvals │  + Compliance   │
      │   as one UX) │              │                  │
      └─────────────┴──────────────────────────────┘
                    ↓
          Business Outcomes:
          - Accurate attendance data
          - Optimised desk utilisation
          - End-to-end audit compliance
          - Reduced operational overhead
          - Independently evolvable services
```

### 3.3 Risk Overview

| Risk | Impact | Likelihood | Mitigation |
|------|--------|-----------|------------|
| SSO provider outage | High — all logins blocked | Low | Accepted; shared org dependency; existing sessions continue |
| Kafka outage | High — async workflows stall | Low | Multi-AZ MSK; services continue operating; events queued and replayed |
| `user-service` unavailable | Medium — fallback responses used | Low | Circuit breaker returns cached user data |
| `attendance-service` nightly job failure | Medium — attendance data delayed | Medium | Alert fires; retry on next run; prior day data available |
| Arms data quality issues | Medium — propagates to OMS | Medium | Data quality agreement with HR team; validation alerts |
| Cross-service contract breaking changes | High — cascading failures | Medium | Pact contract tests catch breaking changes in CI before deployment |

---

## 4. Capability Architecture (WHAT Layer)

### 4.1 Capability Map

**Identity and Access:**
- External SSO authentication
- Server-side token verification
- Role-based access control per location
- Session lifecycle management

**Workforce Tracking:**
- Automated attendance resolution from badge data
- Manual oversight and override
- Remote and OOO status tracking
- Cross-midnight shift support

**Space Management:**
- Real-time desk availability
- Multi-mode floor plan visualisation
- Automated no-show seat release

**Request and Approval Workflows:**
- Remote day and OOO approval with delegation
- Supply request two-stage Saga
- Asset request two-stage Saga
- Approval delegation management

**Visitor Management:**
- Visitor pre-registration and walk-in handling
- Compliance-grade agreement versioning

**Events and Collaboration:**
- Event creation, RSVP, waitlist, recurring events

**Governance and Compliance:**
- Event-driven immutable audit logging
- In-app and email notification dispatch
- Multi-location configuration isolation
- Internationalisation

### 4.2 Capability-to-Service Mapping

| Capability | Service |
|-----------|---------|
| Authentication | `identity-service` |
| Role management | `user-service` |
| Attendance resolution | `attendance-service` |
| Space management | `seating-service` |
| Remote/OOO approvals | `remote-service` |
| Visitor lifecycle | `visitor-service` |
| Office events | `event-service` |
| Supply management | `supplies-service` |
| Asset lifecycle | `assets-service` |
| Notifications | `notification-service` |
| Audit trail | `audit-service` |

---

## 5. Logical Architecture (HOW – Abstract Design)

### 5.1 Architecture Style: Microservices with Event-Driven Backbone

The system uses **microservices architecture** with an **event-driven communication backbone**:

- **Synchronous REST** for user-facing queries and operations requiring immediate confirmation (with circuit breakers on every call)
- **Apache Kafka** for all state change events, cross-service workflows, notification dispatch, and audit logging

This separation means `notification-service` and `audit-service` are never called directly — they react to events. This is the most important coupling decision in the system. Removing direct calls to these services means any producing service can be deployed without considering the notification or audit service's availability.

### 5.2 Logical System Diagram

```
                    [Angular SPA]
                         │
               ┌─────────┴──────────┐
               │     API Gateway     │
               │  Auth · Route · Log │
               └─────────┬──────────┘
                         │ Internal mTLS + JWT
          ┌──────────────┼──────────────────────────┐
          │              │                           │
   [identity-]     [user-service]           [feature services]
    service]              │                  attendance · seating
          │               │                  remote · visitor
          │               │                  event · supplies · assets
          │               │                           │
          └───────────────┴───────────────────────────┘
                                    │
                           ┌────────┴────────┐
                           │   Apache Kafka   │
                           │  (event bus)     │
                           └────────┬─────────┘
                                    │
                    ┌───────────────┴────────────────┐
                    │                                 │
           [notification-service]            [audit-service]
                  (dispatch)                 (persist immutably)
```

### 5.3 Communication Decision Matrix

| Scenario | Pattern | Rationale |
|----------|---------|-----------|
| User requests seat availability | REST (sync) | Immediate response required |
| Seat booking confirmed | Kafka event | Notification and audit don't need to delay the response |
| Remote request approved | Kafka event | Attendance overlay + notification + audit all react independently |
| Attendance status overlay | Kafka consumer | Attendance service reacts asynchronously; no tight coupling to remote-service |
| Manager dashboard (team data aggregation) | API Composition | Parallel REST calls to attendance, remote, seating services |
| Supply request fulfilment workflow | Choreography Saga via Kafka | Multi-step workflow across supplies-service stages; no orchestrator SPOF |

### 5.4 Saga Pattern — Supply Request Fulfilment

```
Employee submits supply request
  ↓ supplies-service sets status = PENDING_MANAGER
  ↓ publishes supply.request.submitted
        ↓
  notification-service: notifies Manager (in-app + email)
  audit-service: persists audit event
        ↓
Manager approves
  ↓ supplies-service sets status = PENDING_FACILITIES
  ↓ publishes supply.request.approved (stage 1)
        ↓
  notification-service: notifies Facilities Admin
        ↓
Facilities Admin fulfils
  ↓ supplies-service sets status = FULFILLED; decrements stock (FIFO)
  ↓ publishes supply.request.fulfilled
        ↓
  notification-service: notifies Employee
  audit-service: persists full trail

Compensation (if Manager rejects):
  ↓ supplies-service sets status = REJECTED
  ↓ publishes supply.request.rejected
        ↓
  notification-service: notifies Employee with reason
```

---

## 6. Physical Architecture (Technology Decisions)

### 6.1 Technology Stack

| Layer | Technology | Justification |
|-------|-----------|---------------|
| Frontend | Angular (TypeScript) | Mandated by team expertise; strong component model for complex enterprise UI |
| Backend (per service) | Java 21, Spring Boot 3.x | Mandated by team expertise; mature ecosystem for REST, security, Kafka, scheduling |
| API Gateway | AWS API Gateway or Kong | Centralised auth, routing, rate limiting; no business logic |
| Message broker | Apache Kafka (AWS MSK) | Durable, replayable event log; exactly-once semantics available; supports audit replay |
| Databases (per service) | PostgreSQL (AWS RDS) | JSONB; relational integrity within service boundary; consistent tooling |
| Internal security | mTLS via Istio or AWS App Mesh | Zero Trust — mutual certificate authentication for every service call |
| Resilience | Resilience4j | Spring Boot native; circuit breaker, retry, timeout, bulkhead |
| Authentication | OAuth 2.0 / OIDC via existing SSO | No credentials stored; single source of identity |
| Containerisation | Docker | Consistent deployment across all environments |
| Orchestration | Kubernetes (AWS EKS) | Horizontal scaling, rolling deploys, health-probe-driven availability |
| Secrets | AWS Secrets Manager | Runtime secret injection; never in code or config files |
| Observability | Prometheus + Grafana, OpenTelemetry + Jaeger, CloudWatch Logs | Full observability stack per service |
| Contract testing | Pact | Consumer-driven contracts prevent silent breaking API changes across 11 services |

### 6.2 Technology Trade-Off Comparison

| Concern | Choice | Alternative Rejected | Reason Rejected |
|---------|--------|---------------------|----------------|
| Architecture | Microservices | Modular monolith | 11 distinct capabilities with different scaling/failure/deployment requirements |
| Event broker | Kafka | RabbitMQ | Kafka's durable replayable log is required for audit replay capability |
| Workflow pattern | Choreography Saga | Orchestration Saga | Central orchestrator becomes a SPOF; choreography keeps each service autonomous |
| Caching | None at launch | Redis | Scale doesn't justify it; adds operational overhead; can be added per-service later |
| API format | REST + JSON | GraphQL | REST is simpler to contract-test across 11 services; GraphQL adds gateway complexity |
| CQRS | Not at launch | CQRS per service | Read/write patterns not complex enough yet; introduce per-service if traffic diverges |
| DB per service | PostgreSQL | Polyglot (different DB per service) | Consistent operational model; all services share the same schema characteristics |

---

## 7. System Design Deep Dive

### 7.1 API Design

All APIs follow these standards:
- Versioned: `/api/v1/...` from day one
- Response envelope: `{ "success": true/false, "data": {}, "error": null }`
- All inputs validated via DTOs with `@Valid`
- Entities never returned in API responses — always mapped to DTOs
- Swagger (OpenAPI 3.0) documentation mandatory on every endpoint
- All list endpoints paginated — no unbounded queries

### 7.2 Cross-Service Data Access — API Composition Pattern

A manager's dashboard requires data from multiple services. The API Gateway (or a lightweight BFF layer) composes this with parallel REST calls:

```
GET /api/v1/dashboard/manager/{id}
        ↓ (parallel)
  ├── attendance-service: GET /api/v1/attendance/team/{id}
  ├── remote-service:     GET /api/v1/teams/{id}/schedule
  └── seating-service:    GET /api/v1/seat-bookings/team/{id}
        ↓
Composed response to client
```

Each call has its own circuit breaker and fallback — if one service is degraded, the others still return data.

### 7.3 Audit Event Schema

Every service publishes this structure to `oms.audit.event`:

```json
{
  "eventId": "uuid-v4",
  "eventType": "audit.event",
  "version": "1",
  "correlationId": "uuid-v4",
  "occurredAt": "2026-03-01T09:00:00Z",
  "actorId": "user-uuid",
  "actorRole": "MANAGER",
  "action": "SEAT_BOOKED",
  "entityType": "SeatBooking",
  "entityId": "entity-uuid",
  "locationId": "location-uuid",
  "previousState": null,
  "newState": { "seatId": "...", "date": "...", "status": "CONFIRMED" }
}
```

`audit-service` persists these records immutably. The database user has INSERT privilege only — UPDATE and DELETE are not granted at the database level.

---

## 8. Scalability and Performance Design

### 8.1 Scaling Model

Each service scales independently. A spike in seat bookings during Monday morning rush does not affect the attendance sync job or the procurement notification pipeline.

```
Users → API Gateway
             ↓
     ┌───────┴────────────────────────────────┐
     │  seating-service (high read load)       │
     │  Pods: 2 → 10 (HPA on CPU > 70%)       │
     └────────────────────────────────────────┘

     ┌───────────────────────────────────────┐
     │  attendance-service (high write load   │
     │  during nightly sync job)              │
     │  Pods: 2 → 6 (isolated via bulkhead)   │
     └───────────────────────────────────────┘

     ┌───────────────────────────────────────┐
     │  audit-service (high Kafka throughput) │
     │  Pods: 2 → 8 (HPA on consumer lag)    │
     └───────────────────────────────────────┘
```

### 8.2 Performance Strategies

| Service | Strategy |
|---------|---------|
| `seating-service` | Real-time availability from SeatBooking records; properly indexed on `(booking_date, location_id)` |
| `attendance-service` | Nightly sync job runs in bulkhead-isolated thread pool; does not compete with API request threads |
| `audit-service` | Append-only writes; no UPDATE contention; scales horizontally as Kafka consumer group |
| `notification-service` | Fire-and-forget Kafka consumer; email dispatch is async; never blocks producing services |
| All services | All list endpoints paginated; no unbounded queries ever |

### 8.3 Kafka Consumer Scaling

Kafka consumer groups scale horizontally — add pods to increase throughput per topic:

```
oms.audit.event topic (3 partitions)
  → audit-service pod 1  (partition 0)
  → audit-service pod 2  (partition 1)
  → audit-service pod 3  (partition 2)
```

Number of pods ≤ number of partitions for efficient parallelism.

---

## 9. Security Architecture

### 9.1 Five Security Layers

```
Layer 1: Network
  - TLS 1.3 on all external HTTPS
  - All services in private VPC subnets
  - Database instances not accessible from public internet
  - API Gateway is the sole public-facing entry point

Layer 2: Identity (identity-service)
  - SSO only; no local password management
  - OIDC token verified server-side via JWKS
  - HTTP-only session cookies; JavaScript cannot access them

Layer 3: Service Mesh (Zero Trust)
  - mTLS on every service-to-service call (Istio / App Mesh)
  - Internal JWT verified by every receiving service
  - No service trusted based on network position alone

Layer 4: Authorisation (per service)
  - Role check: does the user hold the required role?
  - Location check: does the user hold it at the target location?
  - Both checks enforced at the service layer — API Gateway does coarse filtering only

Layer 5: Data
  - Parameterised SQL only — no string concatenation anywhere
  - Soft deletes — historical records never permanently lost
  - Separate DB credentials per service — minimum privilege
  - Audit DB user: INSERT only — no UPDATE, no DELETE at DB level
  - Secrets via AWS Secrets Manager — never in code or config files
```

### 9.2 OWASP Top 10 Mitigation

| Risk | Mitigation |
|------|-----------|
| A01: Broken Access Control | Role+location checks in every service; global via API Gateway and local via `@PreAuthorize` |
| A02: Cryptographic Failures | TLS 1.3 in transit; RDS encryption at rest; HTTP-only cookies |
| A03: Injection | Parameterised queries in every service; `@Valid` DTO validation |
| A04: Insecure Design | ADRs document every security decision; Zero Trust mandated |
| A07: Identification Failures | SSO only; no local passwords; server-side token verification |
| A09: Logging Failures | Immutable `audit-service` log; centralised structured logging across all services |

---

## 10. Risk Assessment and Mitigation

| Risk | Impact | Likelihood | Mitigation |
|------|--------|-----------|------------|
| SSO provider outage | High — logins blocked | Low | Accepted; shared org dependency; existing sessions valid until expiry |
| Kafka cluster failure | High — async workflows stall | Low | Multi-AZ MSK; at-least-once delivery on reconnect; services degrade gracefully |
| `user-service` unavailable | Medium — fallback responses | Low | Circuit breaker on all callers; cached user data returned |
| Nightly badge sync failure | Medium — attendance delayed | Medium | Alert fires; retry on next scheduled run; prior data remains correct |
| Pact contract violation (breaking API change) | High — cascading failures | Medium | Pact tests catch violations in CI before deployment |
| Cross-location data leakage | Critical | Low | Schema-level `location_id` scoping; tested in security review |
| Service-to-service without mTLS | Critical | Low | Service mesh enforced; unauthenticated calls rejected at mesh layer |
| Arms data quality issues | Medium | Medium | Data quality agreement with HR team; validation alerts on sync |

---

## 11. Integration and Interoperability

### 11.1 Integration Architecture

```
OMS ←── Arms HR API (user-service; scheduled sync)
         - Auth: API key (via Secrets Manager)
         - Data: employees, start_date, manager_id, department
         - Resilience: circuit breaker; retry; alert on failure

OMS ←── AWS Athena (attendance-service; nightly batch)
         - Auth: IAM role
         - Query: badge events by date partition
         - Idempotent: duplicate ingestion of same event is safe

OMS ←→ SSO Provider (identity-service; per-login)
         - Protocol: OAuth 2.0 / OIDC
         - JWKS polled for key rotation
         - Token verification is entirely server-side

OMS ──→ Email Provider (notification-service; event-triggered)
         - Protocol: SMTP or provider REST API (TBD)
         - Auth: API key (via Secrets Manager)

OMS ──→ Procurement System (supplies-service; Phase 2)
         - Trigger: supply.reorder.triggered Kafka event
         - Protocol: REST API (contract TBD — AD-026)
```

### 11.2 Event Contract Governance

All Kafka event schemas are defined as versioned records. Breaking changes require a new event type (e.g. `remote.request.approved.v2`) — the old type continues to be published until all consumers migrate. This ensures no consumer is broken by a producer schema change.

---

## 12. Cost Analysis and Optimisation

### 12.1 AWS Cost Estimate (Monthly)

| Component | Service | Estimated Monthly Cost |
|-----------|---------|----------------------|
| EKS cluster | AWS EKS | ~$75 (control plane) |
| Application containers (11 services × 2 pods) | ECS Fargate / EC2 node groups | ~$300–600 |
| Databases (11 RDS instances, db.t3.small) | AWS RDS PostgreSQL | ~$300–500 |
| Kafka | AWS MSK (2-broker multi-AZ) | ~$200–350 |
| API Gateway | AWS API Gateway | ~$15–30 |
| Audit archival | S3 Glacier | ~$5–20 |
| Observability | CloudWatch + Prometheus (managed) | ~$50–100 |
| Secrets Manager | AWS Secrets Manager | ~$10 |
| **Total (estimated)** | | **~$955–1,685/month** |

_Costs are indicative. Actual costs depend on traffic, data volume, and right-sizing after load testing._

### 12.2 Cost Optimisation Strategies

- **Right-size RDS instances** — start with `db.t3.small` per service; scale only where load is demonstrated
- **Spot instances** for non-critical pods (notification-service, audit-service)
- **Kafka partition optimisation** — start with 3 partitions per topic; increase only where consumer lag is measured
- **Nightly batch over real-time streaming** — avoids continuous Athena and stream processing costs
- **Audit archival** — records > 24 months moved to S3 Glacier; prevents unbounded RDS growth
- **Consolidated monitoring** — single Prometheus + Grafana deployment scrapes all 11 services

---

## 13. Implementation and Execution Plan

### 13.1 Development Phases

| Phase | Duration | Services Delivered |
|-------|----------|--------------------|
| Phase 1 | Weeks 1–22 | Infrastructure + 7 core services + hardening + launch |
| Phase 2 | Weeks 23–40 | 4 secondary services + hardening + launch |

### 13.2 Build Order Rationale

The service build order respects dependency direction:

```
Infrastructure first (M1)        ← Everything depends on EKS, Kafka, RDS
  ↓
identity + user (M2)             ← All services depend on auth and user context
  ↓
audit + notification (M3)        ← Must exist before business services produce events
  ↓
attendance (M4)                  ← Publishes no_show events consumed by seating
  ↓
seating (M5)                     ← Consumes attendance events
  ↓
remote (M6)                      ← Publishes events consumed by attendance
  ↓
Phase 1 hardening + launch (M7)
  ↓
visitor · event · supplies · assets (M8–M11, sequential or parallel)
  ↓
Phase 2 hardening + launch (M12)
```

### 13.3 CI/CD and Testing

```
Per-service GitHub Actions pipeline:

PR created
  ↓ Lint → Unit tests → Integration tests (TestContainers) → Pact consumer tests
  ↓ OWASP scan → Docker build → push to ECR

Merge to develop    → auto-deploy to DEV
Merge to staging    → auto-deploy to STAGING + run Pact provider verification
Merge to main       → manual approval → PRODUCTION + rollback plan documented
```

---

## 14. Monitoring and Continuous Improvement

### 14.1 Observability Stack

```
11 Spring Boot services
  ↓ /actuator/prometheus (metrics)
  ↓ stdout structured JSON (logs)
  ↓ OpenTelemetry SDK (traces)

Prometheus   → scrapes all services every 15s
Grafana      → dashboards per service + system-wide view
CloudWatch   → centralised log aggregation from stdout
Jaeger       → distributed trace visualisation
PagerDuty    → alert routing for critical failures
Slack        → alert routing for warnings
```

### 14.2 Key Dashboards (Grafana)

| Dashboard | Metrics |
|-----------|---------|
| System health | Error rate (5xx), p95 latency per service |
| Kafka health | Consumer group lag per topic per service |
| Circuit breakers | Open/half-open state per service |
| Nightly jobs | Badge sync completion time, no-show release completion |
| DB health | Connection pool utilisation per service |
| Deployment | Pod restarts, rollout status per service |

### 14.3 Alerting Rules

| Alert | Threshold | Severity | Route |
|-------|-----------|----------|-------|
| Badge sync job failure | Any failure | Critical | PagerDuty |
| Service error rate > 1% | 5 minutes sustained | Critical | PagerDuty |
| Circuit breaker OPEN | Any service | Critical | PagerDuty |
| Kafka consumer lag > 10,000 | Any topic | Warning | Slack |
| DB connection pool > 80% | Any service | Warning | Slack |
| Pod restart count > 3 | 1 hour | Warning | Slack |

### 14.4 Continuous Improvement Cycle

```
Monitor → (Prometheus / Grafana / Jaeger)
     ↓
Analyse → (error trends, latency outliers, Kafka lag, user feedback)
     ↓
Prioritise → (service-scoped; impact vs effort)
     ↓
Implement → (feature branch → PR → contract tests → CI/CD pipeline)
     ↓
Deploy → (per-service independent deployment; no system-wide downtime)
     ↓
Monitor → (repeat)
```

---

## 15. Final Architecture Summary

### 15.1 End-to-End System Flow

```
Employee opens browser
  ↓ HTTPS
Angular SPA (Angular)
  ↓ REST with session cookie
API Gateway
  ↓ validates session via identity-service
  ↓ routes to target service
Service (e.g. seating-service)
  ↓ validates internal JWT (Zero Trust)
  ↓ role + location check
  ↓ service layer (business logic)
  ↓ repository layer (parameterised SQL with location filter)
  ↓ database (seating_db)
  ↓ publishes domain event to Kafka (post-transaction commit)
  ↓ publishes audit.event to Kafka (post-transaction commit)
  ↓ returns ApiResponse DTO to API Gateway
API Gateway → Angular SPA → Employee

In parallel (async):
  Kafka → notification-service → dispatches in-app + email notification
  Kafka → audit-service → persists immutable audit record
```

### 15.2 Key Architectural Decisions Summary

| Decision | Justification |
|---------|---------------|
| 11 microservices, not a monolith | 11 distinct capabilities with different scaling, failure, and deployment requirements |
| Kafka over direct REST for state changes | Loose coupling; durable event log; audit replay; notification dispatch without blocking |
| No direct calls to notification/audit services | Decouples all services from notification/audit availability; services never wait for them |
| Choreography-based Saga | Avoids a central orchestrator SPOF; each service reacts autonomously |
| Zero Trust (mTLS + internal JWT) | Network position grants no trust; every caller authenticated per request |
| Database per service | Service independence; no cross-service join complexity; no shared-schema coupling |
| Pact contract testing | 11 services require API compatibility guarantees without full integration environments |
| Location scoping at schema level | Prevents cross-location leakage even at raw SQL level; learned from POC incident |
| Immutable audit via Kafka consumer | Write access restricted to Kafka consumer; no API endpoint can write or delete audit records |
| Nightly badge ingestion (not real-time) | Stakeholders accepted T+1 day; avoids streaming infrastructure cost and complexity |

### 15.3 Design Philosophy

This architecture is built on three convictions:

**Own your data.** Every service owns its database. There are no cross-service joins, no shared schemas, and no cross-database credentials. This is the most important structural decision — it is what makes each service independently deployable.

**Publish events, not calls.** `notification-service` and `audit-service` are never called directly. Every service that needs to notify or audit publishes a Kafka event and moves on. This eliminates a whole class of coupling and failure modes.

**Verify everything.** Zero Trust means every service validates every caller on every request. mTLS, internal JWTs, role checks, and location checks are not optional layers — they are the baseline. A service running inside the cluster is not automatically trusted.

---

*End of Solution Architecture Document*
