# OMS Service Merge Recommendations

> **Context:** Reduce 11 services to a more manageable count while preserving architectural integrity.  
> **Constraint noted:** AWS X-Ray will not be used — tracing is OpenTelemetry → Jaeger only.

---

## Current State: 11 Services

| # | Service | Phase | Role |
|---|---------|-------|------|
| 1 | `identity-service` | 1 | SSO, session management |
| 2 | `user-service` | 1 | User profiles, roles, locations, HR sync |
| 3 | `attendance-service` | 1 | Badge ingestion, WorkSession, AttendanceRecord |
| 4 | `seating-service` | 1 | Floor plans, seat bookings, no-show release |
| 5 | `remote-service` | 1 | Remote/OOO requests, approval workflow, policy |
| 6 | `notification-service` | 1 | Kafka consumer — in-app + email dispatch |
| 7 | `audit-service` | 1 | Kafka consumer — immutable audit log |
| 8 | `visitor-service` | 2 | Visitor lifecycle, check-in/out, agreements |
| 9 | `event-service` | 2 | Office events, RSVP, waitlist, recurring |
| 10 | `supplies-service` | 2 | Supply catalogue, stock, two-stage approval |
| 11 | `assets-service` | 2 | Asset register, assignments, fault, maintenance |

---

## Recommended Target State: 8 Services

**3 merges proposed.** No core architectural rules are violated — every merged service still owns its own database schema, and all Kafka event patterns remain intact.

---

## Merge 1 — `identity-service` + `user-service` → `auth-service`

### What merges
| Before | After |
|--------|-------|
| `identity-service` (Session) | `auth-service` |
| `user-service` (User, UserRole, Location, LocationConfig) | `auth-service` |

### Why this merge is justified

**The dependency is synchronous on every request.** `identity-service` calls `user-service` via REST on every `/api/v1/auth/validate` call to resolve the authenticated user's roles. This means:
- If `user-service` is degraded, authentication degrades too — the circuit breaker separation is misleading isolation
- Every authentication validation crosses a network hop, adding latency and a potential failure point
- Merging removes this intra-service REST call entirely and makes auth validation a direct in-process call

**The bounded context is the same.** "Who you are" (identity) and "what you are allowed to do" (roles, locations) are one cohesive domain — Identity and Access Management. Separating them was over-engineering for this scale.

**`identity-service` is a thin service.** It owns one table (`sessions`) and three endpoints (`/login`, `/callback`, `/validate`). It has no Kafka events, no scheduled jobs, and no business logic beyond session lifecycle. It does not warrant a separate service, database, and deployment pipeline.

**At this scale (hundreds of employees, 2–5 locations), independent scaling is not needed.** Authentication requests and user profile reads share the same workload characteristics and traffic patterns.

### What changes
- **Database:** `identity_db` + `user_db` → `auth_db` (single schema, two logical groups of tables)
- **Kafka events:** unchanged — `user-service` events (`user.created`, `user.deactivated`, etc.) are now published by `auth-service`
- **API routes:** unchanged — gateway still routes `/api/v1/auth/**` and `/api/v1/users/**` to the same service
- **HR sync job:** moved to `auth-service` (was in `user-service`)
- **Removed:** one inter-service REST call (the internal `GET /validate` hop from identity → user)

### Savings
- 1 fewer RDS instance
- 1 fewer K8s Deployment + HPA
- 1 fewer CI/CD pipeline
- 1 fewer Pact contract
- Eliminates the highest-frequency inter-service REST call in the system

---

## Merge 2 — `supplies-service` + `assets-service` → `inventory-service`

### What merges
| Before | After |
|--------|-------|
| `supplies-service` (consumable stock, two-stage approval, reorder) | `inventory-service` |
| `assets-service` (non-consumable register, assignment lifecycle, fault, maintenance) | `inventory-service` |

### Why this merge is justified

**They share the same domain: physical office inventory.** Supplies are consumable items (pens, paper, coffee). Assets are non-consumable items (laptops, chairs, monitors). Both are physical things that the Facilities team manages in the office. The bounded context is "things the office owns and tracks."

**Identical approval workflow shape.** Both use a two-stage approval Saga:
```
Employee submits request
  → Stage 1: Manager approves/rejects
  → Stage 2: Facilities Admin fulfils/rejects
  → Kafka event published
  → notification-service + audit-service react
```
This logic is almost line-for-line identical between the two services. Sharing it in one service avoids duplicating the workflow engine.

**Same stakeholder.** The Facilities Admin is the owner of both domains. In most organisations, the same person or team manages supply requests and asset assignments. A single service means a single Facilities Admin view.

**Both are Phase 2.** Neither is on the critical path for launch. Merging them simplifies Phase 2 delivery — one service to build, test, and deploy instead of two.

**The data models are compatible.** Both follow a `Category → Item → Request → Approval → Fulfilment` shape. Supplies adds batch-level stock tracking (FIFO expiry); Assets adds assignment lifecycle and fault/maintenance records. These live cleanly in separate table groups within one schema.

### What changes
- **Database:** `supplies_db` + `assets_db` → `inventory_db`
- **Kafka events:** unchanged — all events from both services continue to be published, now by `inventory-service`
- **API routes:** gateway routes `/api/v1/supplies/**` and `/api/v1/assets/**` to the same service
- **Separation within service:** supply logic and asset logic remain as distinct packages/modules — this is an implementation concern, not an architecture concern

### Savings
- 1 fewer RDS instance
- 1 fewer K8s Deployment + HPA
- 1 fewer CI/CD pipeline
- Shared two-stage approval workflow code

---

## Merge 3 — `visitor-service` + `event-service` → `workplace-service`

### What merges
| Before | After |
|--------|-------|
| `visitor-service` (visitor lifecycle, check-in/out, agreement versioning) | `workplace-service` |
| `event-service` (events, RSVP, waitlist, recurring events) | `workplace-service` |

### Why this merge is justified

**Both are Phase 2 with low complexity relative to other services.** Neither has external integrations, scheduled jobs, or Kafka consumers. Both are primarily REST APIs that produce Kafka events to notification and audit.

**Identical communication pattern.** Both services:
- Call `user-service` via REST (to resolve users/hosts)
- Produce Kafka events consumed only by `notification-service` and `audit-service`
- Have no other inter-service dependencies

**Both handle "non-routine office activity."** Visitors coming to the office and office events (talks, meetings, socials) are both in the domain of "things happening in the office beyond daily work." The boundary between them from a Facilities Admin perspective is thin.

**Low combined data model complexity.** Visitor tables: `visitor_profiles`, `parent_visits`, `visit_records`, `agreement_templates`. Event tables: `event_series`, `events`, `event_invites`. These fit cleanly in one `workplace_db` schema with no overlap.

**Compliance concern is manageable.** Visitor agreement signing and audit trail requirements are handled correctly within one service — the agreement signing logic, versioning rules, and audit events continue to work identically. No compliance guarantee is broken by merging.

### What changes
- **Database:** `visitor_db` + `event_db` → `workplace_db`
- **Kafka events:** unchanged — visitor and event events continue to be published, now by `workplace-service`
- **API routes:** gateway routes `/api/v1/visitors/**`, `/api/v1/visits/**`, and `/api/v1/events/**` to the same service
- **Agreement versioning logic:** unchanged — lives as a module within `workplace-service`

### Savings
- 1 fewer RDS instance
- 1 fewer K8s Deployment + HPA
- 1 fewer CI/CD pipeline
- 1 fewer Pact contract

---

## What NOT to Merge

### Do NOT merge `notification-service` + `audit-service`
These two look similar on the surface (both are Kafka-consumer-only services with no REST callers) but have fundamentally different requirements:

| Concern | notification-service | audit-service |
|---|---|---|
| DB user privilege | Standard read/write | INSERT only — no UPDATE, no DELETE (enforced at DB level) |
| Failure criticality | Acceptable degraded (notification can be delayed) | Critical — missed audit events are a compliance gap |
| Retention policy | Short-term (notifications expire) | 24-month active + S3 archival |
| External dependency | Email provider API | None |
| Scaling driver | Email throughput | Kafka message volume (all audit events from all services) |

Merging them would either require weakening the audit DB's INSERT-only guarantee (a security regression) or adding complexity to separate the DB connections inside one service. Neither is acceptable.

### Do NOT merge `attendance-service` with anything
Attendance has unique characteristics that make it a standalone:
- Only service with an external data source dependency (AWS Athena nightly batch)
- The only scheduled job that is operationally critical (failure = delayed attendance data for all users)
- Bulkhead isolation of the sync job from API threads is an explicit design requirement
- Scaling requirements differ significantly from other services (burst CPU/memory during nightly sync vs low-traffic REST API)

### Do NOT merge `seating-service` with anything
Seating has distinct operational characteristics:
- Highest read traffic of any service (every employee checks seat availability)
- Real-time availability queries with strict sub-200ms latency requirement
- Has its own scheduled job (no-show auto-release)
- Consumes Kafka events (`attendance.no_show.detected`, `user.deactivated`) that affect booking state in real time

### Do NOT merge `remote-service` with anything
Remote scheduling is a distinct workflow domain:
- Contains a policy engine (HARD_BLOCK vs SOFT_WARNING, per-team and per-location policies)
- Manages approval delegation (manager OOO fallback to HR)
- Produces events consumed by `attendance-service` (the overlay mechanism) — tight semantic contract with attendance that should stay explicit at the Kafka level

---

## Final Service Inventory: 8 Services

| # | Service | Replaces | Phase |
|---|---------|----------|-------|
| 1 | `auth-service` | `identity-service` + `user-service` | 1 |
| 2 | `attendance-service` | unchanged | 1 |
| 3 | `seating-service` | unchanged | 1 |
| 4 | `remote-service` | unchanged | 1 |
| 5 | `notification-service` | unchanged | 1 |
| 6 | `audit-service` | unchanged | 1 |
| 7 | `inventory-service` | `supplies-service` + `assets-service` | 2 |
| 8 | `workplace-service` | `visitor-service` + `event-service` | 2 |

---

## Impact Summary

| Metric | Before | After | Change |
|---|---|---|---|
| Total services | 11 | 8 | -3 |
| RDS instances | 11 | 8 | -3 |
| K8s Deployments | 11 | 8 | -3 |
| CI/CD pipelines | 11 | 8 | -3 |
| Pact contracts | Multiple | Fewer | -2 minimum |
| Estimated monthly infra cost | ~$955–$1,685 | ~$760–$1,330 | ~-20% |

---

## Note on AWS X-Ray

The architecture documents listed OpenTelemetry + Jaeger as the primary tracing solution with AWS X-Ray as an alternative. Per the decision to not use X-Ray, the tracing stack is confirmed as:

```
OpenTelemetry SDK (in every service)
    ↓ exports traces
Jaeger (centralised trace store + visualisation)
```

This is already the primary path in the architecture. No services need to change — simply ensure no service configures the `OTEL_TRACES_EXPORTER=xray` environment variable. The `correlationId` propagation via `X-Correlation-ID` header remains unchanged.

AWS CloudWatch Logs and Prometheus + Grafana continue to handle logging and metrics respectively.
