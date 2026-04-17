# Solution Architecture Document
## Office Management System (OMS) — 5-Layer Excellence Model

**Version:** 3.0 | **Author:** OMS Architecture Team | **Date:** April 2026

---

## 1. Problem Understanding and Business Context

### 1.1 Formal Problem Statement

Organisations operating across multiple office locations currently manage employee attendance, workspace allocation, visitor access, supply inventories, and asset lifecycles through a combination of spreadsheets, email chains, and disconnected tools. This introduces:

- **Operational risk** — no single source of truth for employee presence, desk occupancy, or asset ownership
- **Compliance gaps** — manual record-keeping is unauditable; regulatory requirements for access logs and asset tracking cannot be met
- **Poor employee experience** — friction in basic self-service tasks (booking a desk, requesting supplies, submitting remote-day requests) drives non-compliance and workarounds
- **Manager blindness** — managers lack real-time visibility into team location, remote distribution, and availability
- **Facilities burden** — Facilities Admins manage uncoordinated, manual processes for visitors, supplies, and assets

The OMS replaces this fragmented landscape with a **cloud-native microservices platform** providing a unified, authoritative, location-aware system for all workplace operations.

### 1.2 Business Objectives

| Objective | Success Metric |
|---|---|
| Replace manual attendance tracking | 100% of attendance records auto-generated via nightly badge sync |
| Ensure compliance auditability | Every state change logged immutably via the event-driven audit pipeline |
| Improve management visibility | Managers access real-time team location and availability without spreadsheets |
| Reduce operational overhead | Supply and asset workflows tracked end-to-end; zero manual reconciliation |
| Support multi-location growth | System live across 2–5 locations; each service scales independently |
| Enable independent evolution | Any service deployed, updated, or scaled without downtime to others |

### 1.3 Key Stakeholders

| Stakeholder | Role | Key Concern |
|---|---|---|
| Employees | End users — self-service | Ease of use; transparency of own data |
| Managers | Approvers; team visibility | Approval efficiency; team location dashboard |
| HR | Org-wide visibility | Compliance reporting; employee lifecycle accuracy |
| Facilities Admin | Operations | Inventory accuracy; visitor security; desk utilisation |
| Super Admin | Governance | Policy correctness; cross-location oversight |
| Engineering Team | Build and maintain | Service independence; clear contracts; testability |
| IT/Security | Infrastructure | SSO integration; Zero Trust; encryption |

### 1.4 Assumptions and Constraints

| Item | Detail |
|---|---|
| Technology | React frontend, Java 21/Spring Boot backend — team expertise |
| Auth | SSO provider supports OAuth 2.0 / OIDC and exposes a JWKS endpoint |
| Employee data | Arms HR system is the authoritative source; OMS syncs from it |
| Badge data | AWS Athena nightly batch — same-day attendance not required at launch |
| Procurement | API contract for Phase 2 reorder integration is pending |
| Tracing | AWS X-Ray excluded; OpenTelemetry + Jaeger |

---

## 2. Stakeholder and Requirement Analysis

### 2.1 Prioritised Functional Requirements

**Critical — must be complete before Phase 1 launch:**
- SSO authentication, session management, role + location enforcement
- Automated attendance computation from badge data
- Real-time seat booking with no-show auto-release
- Remote/OOO request and approval workflow
- Event-driven notification dispatch
- Immutable audit logging of all mutations

**High — Phase 2:**
- Visitor lifecycle management (compliance-grade agreement versioning)
- Office event RSVP and waitlist
- Supply catalogue and stock management
- Asset register and assignment lifecycle

### 2.2 Non-Functional Requirements (with targets)

| NFR | Target | Measurement |
|---|---|---|
| Seat availability latency | < 200ms p95 | Prometheus `http_server_requests_seconds` |
| Service independence | Zero-downtime independent deployment | CI/CD pipeline per service |
| Fault isolation | Service failure → degraded mode, not outage | Resilience4j circuit breakers |
| Audit completeness | 100% of mutations produce an audit event | RabbitMQ DLQ monitoring |
| Scalability | Horizontal scaling per service | ECS Service Auto Scaling |
| Security | Zero Trust; no shared secrets | Internal JWT; security groups; Secrets Manager |

---

## 3. Business Architecture (WHY Layer)

### 3.1 Why This System Exists

The OMS exists because the organisation's operational processes have outgrown manual tooling. The cost of the current state is:

- **Compliance risk** — unauditable manual records; regulatory exposure
- **Wasted manager time** — manually tracking team location, approving requests via email
- **Facilities inefficiency** — unknown desk utilisation; manual supply reconciliation
- **Employee friction** — inconsistent processes across locations

### 3.2 Value Delivery Model

```
Badge Events (Athena) + HR Data (Arms) + SSO Identity
                        ↓
                  API Gateway (single entry)
                        ↓
      ┌────────────────┬──────────────────────────────┐
      │  Employee       │  Manager        │  Facilities/HR│
      │  Self-service   │  Visibility     │  Operations   │
      │  Attendance     │  + Approvals    │  + Compliance │
      │  Desk booking   │  Team calendar  │  Supply/Asset │
      │  Requests       │  Dashboard      │  management   │
      └────────────────┴──────────────────────────────┘
                        ↓
                Business Outcomes:
                - Accurate, automated attendance data
                - Optimised desk utilisation
                - End-to-end audit compliance
                - Reduced operational overhead
                - Independent service evolution
```

### 3.3 Risk Overview

| Risk | Impact | Likelihood | Mitigation |
|---|---|---|---|
| SSO provider outage | High — all logins blocked | Low | Accepted; existing sessions continue until expiry |
| RabbitMQ cluster failure | High — async workflows stall | Low | Amazon MQ active/standby; services continue; events queue and drain on reconnect |
| auth-service unavailable | Medium — fallback responses | Low | Circuit breaker returns cached user data |
| Badge sync job failure | Medium — attendance delayed by 1 day | Medium | PagerDuty alert; retry on next scheduled run |
| Pact contract violation | High — cascading API failures | Medium | Pact tests catch violations in CI before deployment |
| Cross-location data leakage | Critical | Low | Schema-level `location_id` scoping; tested in security review |

---

## 4. Capability Architecture (WHAT Layer)

### 4.1 Capability Map

**Identity and Access**
- External SSO authentication (OAuth 2.0 / OIDC)
- Server-side token verification via JWKS
- Role-based access control scoped per location
- Session lifecycle management and revocation

**Workforce Tracking**
- Automated attendance resolution from badge data
- Manual oversight and Super Admin override
- Remote day and OOO status tracking
- Cross-midnight shift support

**Space Management**
- Real-time desk availability without caching
- Multi-mode floor plan visualisation
- Hot-desk booking, permanent assignment, block reservation
- Automated no-show seat release

**Request and Approval Workflows**
- Remote day and OOO approval with delegation
- Supply request two-stage Saga
- Asset request two-stage Saga
- Configurable policy enforcement (HARD_BLOCK / SOFT_WARNING)

**Governance and Compliance**
- Event-driven immutable audit logging of every mutation
- 24-month active retention with S3 archival
- Location-scoped data isolation

**Notifications**
- Event-driven in-app and email dispatch
- Never called directly — always via RabbitMQ

---

## 5. Logical Architecture (HOW — Abstract Design)

### 5.1 Architecture Style

**Microservices with event-driven backbone.**

- **Synchronous REST** for user-facing queries requiring immediate response (with Resilience4j circuit breakers on every call)
- **RabbitMQ** for all state change events, cross-service workflows, notification dispatch, and audit logging

### 5.2 Logical System Diagram

```
                    [React SPA]
                         │
               ┌─────────┴───────────┐
               │    API Gateway       │
               │  Auth · Route · Log  │
               └─────────┬───────────┘
                         │ internal JWT + user context headers
          ┌──────────────┼────────────────────────────────┐
          │              │                                 │
      [auth-]       [feature services]             [feature services]
      service]       attendance · seating           remote · inventory
          │           workplace                        │
          └──────────────┴────────────────────────────┘
                                    │
                           ┌────────┴────────┐
                           │   RabbitMQ       │
                           │  Topic Exchanges  │
                           └────────┬─────────┘
                                    │
                    ┌───────────────┴────────────────┐
                    │                                 │
           [notification-service]            [audit-service]
                 (dispatch)                 (persist immutably)
```

### 5.3 Communication Decision Matrix

| Scenario | Pattern | Rationale |
|---|---|---|
| User requests seat availability | REST (sync) | Immediate response required |
| Seat booking confirmed | RabbitMQ event | Notification + audit don't delay the response |
| Remote request approved | RabbitMQ event | Attendance overlay + notification + audit react independently |
| Manager dashboard aggregation | API Composition (parallel REST) | Attendance + remote + seating data composed in BFF |
| Supply request fulfilment workflow | Choreography Saga via RabbitMQ | Multi-step; no orchestrator SPOF |

---

## 6. Physical Architecture (Technology Decisions)

### 6.1 Technology Stack

| Layer | Technology | Justification | Alternative Rejected | Reason Rejected |
|---|---|---|---|---|
| Frontend | React (TypeScript) | Team expertise; ecosystem; flexible component model | Angular | Team expertise shifted; less flexible |
| Backend | Java 21, Spring Boot 3.x | Team expertise; Spring Cloud ecosystem integration | Node.js | No Spring Cloud; weaker type safety for complex domain |
| API Gateway | Spring Cloud Gateway | Native Eureka integration; Java auth filter; no Lambda overhead | AWS API Gateway | Session cookie auth requires Lambda Authorizer; no Eureka support |
| Service Discovery | Netflix Eureka | Native Spring Boot support; works with dynamic ECS IPs | AWS Cloud Map | Extra infrastructure; no OpenFeign native integration |
| Message Broker | RabbitMQ (Amazon MQ) | Spring Cloud Stream binder; fan-out native; 50% cheaper than Kafka | Kafka (MSK) | Replay not needed; $463/month vs $230/month; higher ops overhead |
| Message Broker | RabbitMQ (Amazon MQ) | Ecosystem fit with Spring Cloud Stream | SQS + SNS | No Spring Cloud Stream binder; SNS fan-out pattern adds 45+ queues |
| Databases | PostgreSQL (AWS RDS) | JSONB; relational; consistent tooling | Polyglot | No benefit at this scale; adds ops overhead |
| Orchestration | AWS ECS Fargate | No K8s control plane ops; right-sized for team | AWS EKS | $75/month K8s control plane; Kubernetes expertise required |
| Tracing | OpenTelemetry + Jaeger | Vendor-neutral; Spring Boot 3 native | AWS X-Ray | Excluded by team decision |
| Auth model | OAuth 2.0 / OIDC via SSO | No local password management | Local auth | Org SSO is the identity source of truth |

### 6.2 Trade-off Comparison — Broker

| Dimension | Kafka | RabbitMQ | SQS |
|---|---|---|---|
| Monthly cost (OMS scale) | ~$463 | ~$230 | ~$0.09 |
| Fan-out (native) | Yes | Yes | Needs SNS |
| Spring Cloud Stream binder | Yes | Yes | No |
| Message replay | Full | No | No |
| Operations complexity | High | Medium | Zero |
| DLQ (built-in) | Manual | Built-in | Built-in |
| **OMS verdict** | Over-spec + expensive | Right fit | No ecosystem fit |

---

## 7. System Design Deep Dive

### 7.1 Choreography Saga — Supply Request

```
Employee submits request
  ↓ inventory-service: status = PENDING_MANAGER
  ↓ publishes supply.request.submitted (oms.inventory exchange)
        ↓
  notification-service: notifies Manager (in-app + email)
        ↓
Manager approves via inventory-service API
  ↓ inventory-service: status = PENDING_FACILITIES
  ↓ publishes supply.request.approved (stage 1)
        ↓
  notification-service: notifies Facilities Admin
        ↓
Facilities Admin fulfils
  ↓ inventory-service: status = FULFILLED; decrements FIFO stock
  ↓ publishes supply.request.fulfilled
        ↓
  notification-service: notifies Employee
  audit-service: persists full trail

Compensation (Manager rejects):
  ↓ inventory-service: status = REJECTED
  ↓ publishes supply.request.rejected
  ↓ notification-service: notifies Employee with reason
```

### 7.2 API Composition — Manager Dashboard

```
GET /api/v1/dashboard/manager/{id}
        ↓ (parallel Feign calls with circuit breakers)
  ├── attendance-service: GET /api/v1/attendance/team/{id}
  ├── remote-service:     GET /api/v1/teams/{id}/schedule
  └── seating-service:    GET /api/v1/seat-bookings/team/{id}
        ↓
Composed response to React SPA

Each call has its own Resilience4j circuit breaker and fallback.
If seating-service is degraded, attendance and remote data still return.
```

---

## 8. Scalability and Performance Design

### 8.1 Independent Scaling Model

```
Peak seat booking (Monday morning rush):
  seating-service: 2 → 8 tasks (HPA on CPU > 70%)
  No impact on attendance-service, remote-service, etc.

Nightly badge sync (02:00 UTC):
  attendance-service: 2 → 4 tasks (scheduled scale-up at 01:45)
  Sync runs in bulkhead-isolated thread pool
  No impact on API request threads

High audit throughput (end of month reporting):
  audit-service: 2 → 6 tasks (HPA on CPU > 70%)
  RabbitMQ queue depth metrics trigger scale-out
```

### 8.2 Performance Strategies

| Service | Strategy |
|---|---|
| `seating-service` | Real-time availability from properly indexed `seat_bookings(booking_date, location_id)` — sub-200ms |
| `attendance-service` | Nightly sync in bulkhead thread pool; API threads unaffected |
| `audit-service` | Append-only writes; no UPDATE contention; horizontal scale via consumer group |
| `notification-service` | Fire-and-forget RabbitMQ consumer; email dispatch async; never blocks producers |
| All services | Paginated list endpoints; no unbounded queries |

---

## 9. Security Architecture

See [security.md](security.md) for full detail.

**Five security layers:**
1. **Network** — VPC private subnets; security groups; TLS 1.3 external
2. **Edge** — Rate limiting; CORS; session cookie validation at gateway
3. **Identity** — SSO only; HTTP-only cookies; immediate revocation
4. **Service** — Internal JWT on every call; role + location check in every service
5. **Data** — Parameterised SQL; separate DB users; INSERT-only for audit; Secrets Manager

---

## 10. Risk Assessment and Mitigation

| Risk | Impact | Likelihood | Mitigation |
|---|---|---|---|
| SSO provider outage | High | Low | Existing sessions continue; no mitigation for new logins |
| Amazon MQ failure | High | Low | Active/standby HA; services continue; messages drain on recovery |
| auth-service degraded | Medium | Low | Circuit breaker; cached user context in session |
| Badge sync failure | Medium | Medium | PagerDuty alert; retry next night; prior data unaffected |
| Pact contract breaking change | High | Medium | Pact in CI catches before deployment |
| Cross-location data leak | Critical | Low | `location_id` in all queries; tested in security review |
| RabbitMQ DLQ accumulation | Medium | Medium | CloudWatch DLQ depth alert; ops runbook for reprocessing |
| Replay lost (vs Kafka) | Low | N/A | Mitigated by durable queues; audit-service runs 2 HA tasks |

---

## 11. Integration and Interoperability

```
OMS ← Arms HR API (auth-service; scheduled sync)
       Auth: API key (Secrets Manager)
       Data: employees, start dates, manager relationships
       Resilience: circuit breaker; retry; alert on failure

OMS ← AWS Athena (attendance-service; nightly batch)
       Auth: IAM role (ECS task role)
       Query: badge events by date partition
       Idempotent: duplicate ingestion of same event is safe

OMS ↔ SSO Provider (auth-service; per-login)
       Protocol: OAuth 2.0 / OIDC
       JWKS polled for key rotation

OMS → Email Provider (notification-service; event-triggered)
       Protocol: REST API (provider TBD)
       Auth: API key (Secrets Manager)

OMS → Procurement System (inventory-service; Phase 2)
       Trigger: supply.reorder.triggered RabbitMQ event
       Protocol: REST API (contract TBD — pending discovery session)
```

---

## 12. Cost Analysis and Optimisation

See [cost-analysis.md](cost-analysis.md) for full detail.

**Estimated monthly AWS infrastructure cost: ~$710–970/month**

**Key cost decisions:**
- RabbitMQ over Kafka: saves ~$2,796/year
- ECS Fargate over EKS: saves ~$1,440–3,540/year
- Service merge 11→8: saves ~$1,440/year (3 fewer RDS instances)
- **Total technology decisions save: ~$5,676–7,776/year**

---

## 13. Implementation and Execution Plan

### Phase 1 (Weeks 1–22)

| Milestone | Work | Blocker |
|---|---|---|
| M1 | Infrastructure: ECS, Amazon MQ, RDS, ALB, Eureka, Gateway, observability | None |
| M2 | auth-service: SSO, sessions, users, roles, HR sync | SSO JWKS endpoint, Arms API |
| M3 | audit-service + notification-service: RabbitMQ consumers live | M1 complete |
| M4 | attendance-service: badge pipeline, two-pass resolution | Athena schema, M3 |
| M5 | seating-service: floor plans, booking, no-show release | M4 (consumes no_show events) |
| M6 | remote-service: requests, approvals, policy engine | M5 (attendance consumes approved) |
| M7 | Phase 1 hardening: security review, performance testing, UAT, launch | M6 |

### Testing Strategy

| Level | Tool | Scope |
|---|---|---|
| Unit | JUnit 5 + Mockito | Business logic, policy enforcement, event mapping |
| Integration | Spring Boot Test + TestContainers | Repository queries, RabbitMQ producer/consumer pairs, full API flows |
| Contract | Pact | API contracts between consumer and provider services |
| Security | MockMvc + role simulation | Every endpoint against every role + location combination |
| Performance | k6 | Seat availability under concurrency; nightly sync throughput |

---

## 14. Monitoring and Continuous Improvement

See [observability.md](observability.md) for full detail.

```
Monitor → Prometheus (metrics) + CloudWatch (logs) + Jaeger (traces)
     ↓
Analyse → error trends, latency outliers, RabbitMQ queue depth, user feedback
     ↓
Prioritise → service-scoped; impact vs effort
     ↓
Implement → feature branch → PR → Pact tests → CI/CD pipeline
     ↓
Deploy → per-service independent deployment; no system-wide downtime
     ↓
Monitor → repeat
```

**Key alerts:**
- Badge sync failure → PagerDuty (critical)
- Service error rate > 1% for 5 min → PagerDuty (critical)
- Circuit breaker OPEN → PagerDuty (critical)
- RabbitMQ queue depth > 10,000 → Slack (warning)

---

## 15. Final Architecture Summary

### End-to-End System Flow

```
Employee opens React SPA
  ↓ HTTPS
React SPA
  ↓ REST with session cookie
Spring Cloud Gateway (port 8080)
  ↓ validates session via auth-service (Eureka lb://)
  ↓ generates internal JWT + injects X-User-* headers
  ↓ routes to target service (lb://service-name)
Service (e.g. seating-service)
  ↓ validates internal JWT (Spring Security)
  ↓ role + location check (@PreAuthorize + service layer)
  ↓ business logic
  ↓ parameterised SQL query (PostgreSQL, seating_db)
  ↓ publishes domain event to oms.seating (RabbitMQ)
  ↓ publishes audit.event to oms.audit (RabbitMQ)
  ↓ returns ApiResponse DTO
Gateway → React SPA → Employee

In parallel (async, within 2 seconds):
  RabbitMQ → notification-service → dispatches in-app + email
  RabbitMQ → audit-service → persists immutable audit record
```

### Design Philosophy

**Own your data.** Every service owns its database. No cross-service joins, no shared schemas, no shared credentials. This is the most important structural decision — it is what makes each service independently deployable.

**Publish events, not calls.** notification-service and audit-service are never called directly. Every service that needs to notify or audit publishes a RabbitMQ event and returns immediately. This eliminates a whole class of coupling and failure modes.

**Verify everything.** Zero Trust means every service validates every caller on every request. Internal JWT, role checks, and location checks are not optional layers — they are the baseline.

---

*End of Solution Architecture Document*  
*See [README.md](README.md) for the full documentation index.*
