# OMS Sprint Plans
Sprint 1, Sprint 2, Sprint 3  ·  PI 1  ·  Weeks 1–6
Phase 1 core features: Attendance · Seating · Remote / OOO · Platform security

|  | Sprint 1 (Wks 1–2) | Sprint 2 (Wks 3–4) | Sprint 3 (Wks 5–6) | PI Objectives touched |
|---|---|---|---|---|
| Platform / infra | SSO, interceptor, audit log, Cloud Map (infra + code), API Gateway + Lambda Authorizer, VPC Link, CloudFront, SES email | — | — | PI-OBJ-04 (cross-cutting) |
| Attendance | Badge sync, sessions, records, Arms HR, employee self-view | Manager dashboard, thresholds, sync alerts | HR org view, super-admin override, public holidays | PI-OBJ-01 → complete end of Sprint 2; Sprint 3 extensions |
| Seating | Hierarchy, booking, cancellation | No-show job, block-reserve, visibility config, floor plan visualisation, permanent seat assignment | — | PI-OBJ-02 → complete end of Sprint 2 |
| Remote / OOO | — | Request, policy config, approval flow, weekly limits | OOO submission, recurring schedule, delegation, team calendar, history view | PI-OBJ-03 → complete end of Sprint 3 |
| Story count | 17 stories (9 + 5 + 3) | 11 stories (3 + 5 + 3) | 8 stories (3 + 5) | 36 stories across 3 sprints |

> **Note — original plan gap:** The original Sprint 1/2 plan listed ATT-P04 as "Multi-location data scoping" and ATT-P05 as "Notification infrastructure". Both labels were incorrect. ATT-P04 is the AWS API Gateway + Lambda Authorizer; ATT-P05 is AWS Cloud Map service discovery. Six platform infrastructure tickets (ATT-P04 through ATT-P09) were entirely absent from the original plan. AOMS-15 (Floor Plan Visualisation) and AOMS-20 (Permanent Seat Assignment) were also missing from Sprint 2. All are corrected below.

---

## Sprint 1  ·  Weeks 1–2  ·  PI-OBJ-01 start · PI-OBJ-02 start · PI-OBJ-04 complete

**Sprint goal:** Full AWS infrastructure layer provisioned and verified (Cloud Map, API Gateway, VPC Link, CloudFront, SES). SSO integrated. Badge event pipeline proven end-to-end. Seat hierarchy and booking flow operational. Platform foundation locked before Sprint 2 feature work begins.

---

### Platform / Auth / Infrastructure  —  PI-OBJ-04

> Infrastructure tickets must be sequenced within the sprint. ATT-P08 (Cloud Map namespace) has no service dependencies and must land first; ATT-P09 (VPC Link) depends on it; ATT-P06 (CloudFront) depends on ATT-P09. ATT-P07 (SES) can run in parallel with ATT-P09/P06 after ATT-P08 lands.

**ATT-P01  —  SSO / OAuth 2.0 Integration**
Integrate SSO provider via OAuth 2.0 / OIDC. Token verification entirely server-side. No client-side auth logic.

**ATT-P02  —  Role + Location Interceptor**
Global interceptor enforces per-location role model on every endpoint. Requests without valid role + location context are rejected at gateway.

**ATT-P03  —  Audit Log Infrastructure**
Immutable audit log. All state-changing operations write an entry. 24-month minimum retention. Never deleted.

**ATT-P08  —  AWS Cloud Map Namespace Provisioning**  *(infrastructure first — no service dependencies)*
Provision the `oms.local` private DNS namespace in the OMS VPC via Terraform. Create one `aws_service_discovery_service` per ECS service (8 total: auth, attendance, seating, remote, notification, audit, workplace, inventory). All ECS service definitions include `service_registries` block. DNS A records with 10-second TTL; `MULTIVALUE` routing. No application code runs — ECS handles registration and deregistration automatically. Must be the first M1 infrastructure ticket merged.

**ATT-P04  —  AWS API Gateway + Lambda Authorizer**  *(depends on ATT-P08)*
Lambda Authorizer (Python 3.12) validates the `SESSION` cookie against `auth-service`, injects `X-User-Id`, `X-User-Roles`, `X-Location-Ids`, `X-Correlation-ID`, and `Authorization: Bearer <internal-jwt>` headers into every downstream request. Strips any client-supplied `X-User-*` headers to prevent spoofing. Authorizer cache TTL: 300 seconds keyed on `SESSION` cookie value. Deployed as part of M1 Terraform — no ECS container.

**ATT-P09  —  AWS API Gateway HTTP API + VPC Link**  *(depends on ATT-P08, ATT-P04)*
Provision via Terraform: HTTP API (`oms-api`), VPC Link (`oms-vpclink`) targeting private subnets, CORS (allowCredentials: true, no wildcard origin in production), per-stage throttling (burst 150 / rate 100 rps), 14 route definitions proxying to Cloud Map DNS targets, Lambda Authorizer attached to all protected routes, WAF WebACL (OWASP + bad inputs rules, 1,000 req/5 min rate block), custom domain `api.oms.yourorg.com` with ACM certificate (eu-west-1), JSON access logging to CloudWatch. Open paths: `/health`, `/api/v1/auth/login`, `/api/v1/auth/callback`.

**ATT-P05  —  AWS Cloud Map — Service Code Migration**  *(depends on ATT-P08)*
Remove Netflix Eureka client from all Spring Boot services; configure `RestTemplate` / `WebClient` base URLs from `*_SERVICE_URL` env vars pointing to `http://service-name.oms.local:PORT`. Add `httpx.AsyncClient` base URL env vars for FastAPI services. No self-registration code — Cloud Map handles this via ECS task registration.

**ATT-P06  —  CloudFront Distribution**  *(depends on ATT-P09)*
S3 bucket (`oms-frontend-{env}`) with all public access blocked; Origin Access Control (OAC) restricts `s3:GetObject` to CloudFront only. Distribution with two origins: S3 (default `/*`) and API Gateway (`/api/*`). Caching: long TTL for hashed assets, `no-cache` for `index.html`, `CachingDisabled` for `/api/*`. Custom domain `app.oms.yourorg.com` with ACM certificate (us-east-1 required). Security headers policy (HSTS, X-Frame-Options: DENY, CSP, X-Content-Type-Options). WAF WebACL (scope CLOUDFRONT, us-east-1): OWASP + bad inputs rules, 2,000 req/5 min block. React Router fallback: 403/404 → `index.html` with HTTP 200. Jenkins `Deploy Frontend` stage: `aws s3 sync` + CloudFront cache invalidation for `index.html`.

**ATT-P07  —  Amazon SES Email Integration**  *(depends on ATT-P05 / ATT-P08; required before Sprint 2 REM-002)*
`notification-service` (FastAPI / Python 3.12) sends transactional email via Amazon SES — no API key. ECS task IAM role grants `ses:SendEmail`. SES domain identity `oms.yourorg.com` verified; DKIM CNAME records in Route 53; configuration set `oms-email-events` routes bounce/complaint events to SQS. `EmailService` class with `async def send(...)` using `aioboto3`: throttle errors re-raised (SQS retry); non-retriable errors logged and swallowed. Seven email templates (booking confirmation, remote approved/rejected, OOO approved/rejected, supply pending/fulfilled). SQS consumer maps each event type to the correct template. No PII in logs — only `userId` UUID.

⚠  **ART dependency:** SSO provider must deliver OAuth credentials and discovery document before ATT-P01 can begin. Escalate to ART board on day 1 if not received.
⚠  **Sequencing constraint:** ATT-P08 → ATT-P09 → ATT-P06 must be sequenced in that order. ATT-P07 can run in parallel with ATT-P09/P06 once ATT-P08 is merged.

---

### Attendance  —  PI-OBJ-01

**ATT-001 / AOMS-2  —  Badge Event Ingestion**
Nightly sync job ingests badge events from AWS S3 via Athena. `BadgeEvent` entities created with `location_id`, `employee_id`, `timestamp`, `direction`.

**ATT-002 / AOMS-3  —  Work Session Modelling**
Badge events grouped into `WorkSession` (first badge-in + last badge-out). Cross-midnight shift support: session owned by start date. Configurable gap threshold.

**ATT-003 / AOMS-4 + AOMS-5  —  AttendanceRecord Generation**
`AttendanceRecord` derived from `WorkSession`. Configurable lateness threshold applied. `INSUFFICIENT_HOURS` flagging for sub-threshold duration visits.

**ATT-004 / AOMS-?  —  Arms HR Integration**
Employee records, employment start dates, and org structure pulled from Arms HR system. Attendance history begins from HR employment start date.

**ATT-005 / AOMS-6  —  Employee Attendance History View**
Employee views their own attendance history from employment start date. Manager views team. Filter by date range.

⚠  **ART dependency:** Arms HR team must provide API access and schema before ATT-004 can be completed. Stub with mock employee service if not available by Sprint 1 day 3.

---

### Seating  —  PI-OBJ-02

**SEA-001 / AOMS-14  —  Seat Hierarchy Setup**
Office → Floor → Zone → Seat hierarchy. Admin creates and manages the hierarchy per location. All entities carry `location_id`. `Seat.assigned_user_id` column added in this migration (used by AOMS-20).

**SEA-002 / AOMS-16  —  Hot-Desk Booking**
Employee books a seat within the 2-week window (`hot_desk_booking_window_days` from `LocationConfig`). Real-time availability — not cached. Rejects booking of `PERMANENT` type seats regardless of `assigned_user_id`.

**SEA-003 / AOMS-17  —  Booking Cancellation**
Employee cancels up to 2 hours before booking. Seat released to pool immediately on cancellation.

---

### Definition of Done  —  Sprint 1 (all stories)

- Code reviewed and merged to `main`
- Unit tests written and passing (target: 80% coverage on new code)
- Role + location interceptor applied and verified on all new endpoints
- Audit log entry confirmed for all state-changing operations
- No hardcoded credentials, raw queries, or skipped auth checks
- Cloud Map namespace smoke-test: `nslookup auth-service.oms.local` resolves from within VPC
- API Gateway open-path smoke-test: `GET /health` returns 200 without session cookie
- CloudFront smoke-test: `app.oms.yourorg.com` serves SPA; `/api/*` proxied to API Gateway
- SES LocalStack integration test: SQS consumer dispatches email on `seat.booking.confirmed` event
- Acceptance criteria signed off by product owner

---

## Sprint 2  ·  Weeks 3–4  ·  PI-OBJ-01 complete · PI-OBJ-02 complete · PI-OBJ-03 start

**Sprint goal:** PI-OBJ-01 (Attendance) and PI-OBJ-02 (Seating) marked complete and demo-ready. Floor plan visualisation API live. Remote day scheduling in flight with email notifications working end-to-end via ATT-P07 (SES, delivered Sprint 1).

---

### Attendance  —  PI-OBJ-01 (final stories)

**ATT-006 / AOMS-7  —  Manager Attendance Dashboard**
Manager views team attendance summary. Flags for lateness, `INSUFFICIENT_HOURS`, and no-shows surfaced. Filters by date range and employee.

**ATT-007 / AOMS-9  —  Configurable Thresholds Admin**
Admin configures lateness threshold, minimum hours, session gap threshold, `seat_visibility_mode`, and `hot_desk_booking_window_days` per location via `LocationConfig`. Changes take effect from next sync run.

**ATT-008 / AOMS-12  —  Sync Monitoring and Failure Alerts**
Nightly sync job emits structured logs. Failed or partial sync raises CloudWatch alert. Admin notified via SES (ATT-P07). Manual re-run supported.

✓  **PI-OBJ-01 complete at end of Sprint 2** — demo-ready for ART sync

---

### Seating  —  PI-OBJ-02 (final stories)

**SEA-004 / AOMS-18  —  No-Show Auto-Release**
Scheduled daily job cross-references bookings against badge events. Unconfirmed bookings released at configurable cutoff time. No-shows tracked for admin view. First cross-feature integration point — reads badge event data from the attendance pipeline; treat as a mini integration test with explicit assertions.

**SEA-005 / AOMS-19  —  Manager Block-Reservation**
Manager reserves a block of seats for their team. Individual assignment flexible within the block. Block visible to team members only.

**SEA-006 / AOMS-21  —  Seat Visibility Configuration**
Admin configures `seat_visibility_mode` per office: `FULL` (all occupant names visible), `TEAM_ONLY` (same-team names only), or `AVAILABILITY_ONLY` (occupied/free only). Stored in `LocationConfig`. Read by AOMS-15 floor plan API.

**AOMS-15  —  Floor Plan Visualisation**  *(depends on SEA-001/AOMS-14, SEA-002/AOMS-16, SEA-006/AOMS-21)*
`GET /api/v1/locations/{locationId}/floor-plan?date=YYYY-MM-DD` returns the full nested floor → zone → seat structure with real-time per-seat availability for a given date. Single-response design — no round trips. Key rules:
- Availability derived in **one bulk query** (`SELECT seat_id, user_id FROM seat_bookings WHERE location_id = ? AND booking_date = ? AND status = 'CONFIRMED' AND deleted_at IS NULL`) — not a per-seat loop.
- Floor → Zone → Seat hierarchy fetched in one flat `JOIN`; assembled into nested response in Java.
- `occupantInfo` stripped or returned per `seat_visibility_mode`: `FULL` returns `{userId, name}` for all BOOKED seats; `TEAM_ONLY` returns `{userId, name}` only for same-team bookings (resolved via shared `manager_id`); `AVAILABILITY_ONLY` always returns `null`.
- Requesting employee's own permanent seat flagged `isMyPermanentSeat: true` (via `Seat.assigned_user_id`).
- `date` defaults to today; returns 400 if beyond `hot_desk_booking_window_days`.
- Only `is_active = true` floors and zones returned; inactive seats excluded.
- Employee must hold a role at the requested location — enforced by role + location interceptor.
- Response performant for 500+ seats.

**AOMS-20  —  Permanent Seat Assignment**  *(depends on AOMS-14, AOMS-15, AOMS-16)*
Admin assigns or unassigns a `PERMANENT` type seat to an employee. Three endpoints: `POST /api/v1/seats/{seatId}/permanent-assignment`, `DELETE /api/v1/seats/{seatId}/permanent-assignment`, `PATCH /api/v1/seats/{seatId}/type`. Guards: PERMANENT seat cannot be assigned if already assigned (409); cannot be converted to `HOT_DESK` while `assigned_user_id` is set (400); `HOT_DESK` seat cannot be converted to `PERMANENT` if it has future CONFIRMED bookings (400). All assignment and type-change operations written to `AuditLog`. `isMyPermanentSeat: true` in floor plan response (AOMS-15) verified by integration test.

✓  **PI-OBJ-02 complete at end of Sprint 2** — demo-ready for ART sync

---

### Remote / OOO  —  PI-OBJ-03 (start)

**REM-001 / AOMS-23  —  Remote Day Request Submission**
Employee submits a remote day request for a single date. Single manager approval triggers the request. Configurable weekly remote day limit enforced at submission.

**REM-002 / AOMS-25  —  Manager Approval Flow with Notifications**  *(depends on ATT-P07)*
Manager approves or rejects remote request. SES email notification sent to employee on decision (via ATT-P07 `notification-service`). `seat.booking.confirmed`, `remote.request.approved`, `remote.request.rejected` event types consumed from SQS by `notification-service`.

**REM-003 / AOMS-27  —  Remote Day Policy Configuration**
Admin configures weekly remote day limit per team/office. Hard block prevents submission over limit. Soft warning allows submission with acknowledgement. Configurable per `LocationConfig`.

⚠  **Cross-lane dependency:** REM-002 requires ATT-P07 (SES email integration, Sprint 1) to be live. Verify `notification-service` SQS consumer is wired and tested before marking REM-002 complete.

---

### Sprint 2 Risks

- Arms HR stub (Sprint 1) must be replaced with real integration before ATT-006 can display real team data. Validate this swap explicitly.
- SEA-004 no-show job is the first cross-feature integration point — reads badge event data from the attendance pipeline. Allocate explicit integration test time.
- AOMS-15 floor plan must not use a per-seat availability loop — single bulk query required. Enforce in code review.
- REM-001 recurring schedule parent/child pattern (AOMS-26, Sprint 3) is the most complex data model in PI 1. Review the data model design before Sprint 3 begins.

---

### Definition of Done  —  Sprint 2 (all stories)

- All Sprint 1 DoD criteria carry forward
- Integration tests for cross-feature data flows (badge events feeding SEA-004 no-show release; AOMS-15 floor plan availability verified against seed data)
- Notification emails verified end-to-end in staging (SES LocalStack or confirmed against real SES sandbox)
- AOMS-15 availability confirmed as single bulk query (no per-seat loop) — verified in code review and by query log assertion in integration test
- AOMS-20 `isMyPermanentSeat: true` confirmed in floor plan response via integration test
- Configurable thresholds tested at boundary values
- PI-OBJ-01 and PI-OBJ-02 demo-ready for ART sync at end of Sprint 2
- No open P1 or P2 defects carried forward into Sprint 3
- Arms HR real integration replacing stub — confirmed by integration test against real endpoint

---

## Sprint 3  ·  Weeks 5–6  ·  PI-OBJ-01 extensions · PI-OBJ-03 complete

**Sprint goal:** Remote / OOO feature set completed and demo-ready. Attendance admin-tier stories (HR org view, super-admin override, public holidays) delivered. PI-OBJ-03 marked complete at ART sync.

---

### Attendance  —  PI-OBJ-01 extensions

**AOMS-8  —  HR Org-Wide Attendance View**
HR role views attendance across the entire organisation. Aggregated by department/team. Exportable. Filtered by location, date range, and attendance status.

**AOMS-10  —  Super Admin Attendance Override**
Super admin manually overrides an `AttendanceRecord` status (e.g. corrects a missed badge-in). Override reason mandatory. Written to `AuditLog`. Employee notified.

**AOMS-11  —  Public Holiday Management**
Admin manages public holiday calendar per location. Public holidays excluded from attendance calculations (no `INSUFFICIENT_HOURS` or lateness flags on holiday dates). Configurable per `LocationConfig`.

---

### Remote / OOO  —  PI-OBJ-03 (final stories)

**AOMS-26  —  Recurring Remote Schedule**
Employee submits recurring remote day schedule (weekly pattern). Parent/child record pattern: single manager approval generates all child instances. Manager can revoke individual instances without cancelling the parent. Most complex data model in PI 1 — review schema before implementation begins.

**AOMS-24  —  OOO Request Submission**
Employee submits OOO (out of office) request with date range, reason, and delegate. Distinct from remote day — OOO blocks attendance calculations for the covered dates.

**AOMS-28  —  Approval Delegation**
Employee delegates approval authority to another employee during OOO. Delegated approver receives manager-level approval access for the delegation period. Delegation period validated against the delegate's own schedule.

**AOMS-29  —  Team Schedule Calendar View**
Manager and employee view a team calendar showing who is remote, OOO, or in-office per day. Seat bookings surfaced as in-office indicators. Filter by week and team.

**AOMS-30  —  Employee Request History View**
Employee views their own history of remote day requests, OOO requests, and outcomes (approved / rejected / pending). Filterable by date range and request type.

✓  **PI-OBJ-03 complete at end of Sprint 3** — demo-ready for ART sync

---

### Definition of Done  —  Sprint 3 (all stories)

- All Sprint 1 and Sprint 2 DoD criteria carry forward
- AOMS-26 recurring schedule parent/child pattern integration-tested: approve parent → verify child instances generated; revoke one child → verify parent and other children unaffected
- AOMS-28 delegation: verify delegated approver cannot approve their own requests
- AOMS-11 public holiday: verify no lateness/INSUFFICIENT_HOURS flag on holiday dates in attendance record
- PI-OBJ-03 demo-ready for ART sync at end of Sprint 3
- No open P1 or P2 defects

---

## Ticket Index by ID

| ID | Title | Sprint | Track |
|---|---|---|---|
| ATT-P01 | SSO / OAuth 2.0 Integration | 1 | Platform |
| ATT-P02 | Role + Location Interceptor | 1 | Platform |
| ATT-P03 | Audit Log Infrastructure | 1 | Platform |
| ATT-P04 | API Gateway + Lambda Authorizer | 1 | Platform |
| ATT-P05 | Cloud Map — Service Code Migration | 1 | Platform |
| ATT-P06 | CloudFront Distribution | 1 | Platform |
| ATT-P07 | Amazon SES Email Integration | 1 | Platform |
| ATT-P08 | Cloud Map Namespace Provisioning | 1 | Platform |
| ATT-P09 | API Gateway HTTP API + VPC Link | 1 | Platform |
| AOMS-2 | Badge Event Ingestion | 1 | Attendance |
| AOMS-3 | Work Session Modelling | 1 | Attendance |
| AOMS-4/5 | AttendanceRecord Generation | 1 | Attendance |
| AOMS-? | Arms HR Integration | 1 | Attendance |
| AOMS-6 | Employee Attendance History View | 1 | Attendance |
| AOMS-14 | Seat Hierarchy Setup | 1 | Seating |
| AOMS-16 | Hot-Desk Booking | 1 | Seating |
| AOMS-17 | Booking Cancellation | 1 | Seating |
| AOMS-7 | Manager Attendance Dashboard | 2 | Attendance |
| AOMS-9 | Configurable Thresholds Admin | 2 | Attendance |
| AOMS-12 | Sync Monitoring and Failure Alerts | 2 | Attendance |
| AOMS-18 | No-Show Auto-Release | 2 | Seating |
| AOMS-19 | Manager Block-Reservation | 2 | Seating |
| AOMS-21 | Seat Visibility Configuration | 2 | Seating |
| AOMS-15 | Floor Plan Visualisation | 2 | Seating |
| AOMS-20 | Permanent Seat Assignment | 2 | Seating |
| AOMS-23 | Remote Day Request Submission | 2 | Remote/OOO |
| AOMS-25 | Manager Approval Flow + Notifications | 2 | Remote/OOO |
| AOMS-27 | Remote Day Policy Configuration | 2 | Remote/OOO |
| AOMS-8 | HR Org-Wide Attendance View | 3 | Attendance |
| AOMS-10 | Super Admin Attendance Override | 3 | Attendance |
| AOMS-11 | Public Holiday Management | 3 | Attendance |
| AOMS-26 | Recurring Remote Schedule | 3 | Remote/OOO |
| AOMS-24 | OOO Request Submission | 3 | Remote/OOO |
| AOMS-28 | Approval Delegation | 3 | Remote/OOO |
| AOMS-29 | Team Schedule Calendar View | 3 | Remote/OOO |
| AOMS-30 | Employee Request History View | 3 | Remote/OOO |

---

## Notes

- Ticket IDs (ATT-Pxx, AOMS-xx) align with the OMS backlog Jira CSV. Import the CSV first, then reference these IDs in sprint planning.
- The six platform infrastructure tickets (ATT-P04 through ATT-P09) were absent from the original sprint plan and are added here per their ticket metadata (all Sprint 1). The original plan's ATT-P04 "Multi-location data scoping" and ATT-P05 "Notification infrastructure" labels were mismatches with the actual ticket files.
- ATT-P08 (Cloud Map namespace) must be the first M1 infrastructure ticket merged — ATT-P09, ATT-P06, and ATT-P04 all depend on it.
- ATT-P07 (SES) must land in Sprint 1 for REM-002 approval notifications to work in Sprint 2.
- AOMS-15 floor plan availability must be a single bulk query. This is a code-review gate — per-seat loops are not acceptable at 500+ seats.
- Sprint 1 carries 17 stories due to the platform infrastructure track. If actual team capacity is below this, split ATT-P06 (CloudFront) and ATT-P07 (SES) to Sprint 2 early — both are independent of the Sprint 2 feature stories.
- ART dependencies (SSO credentials, Arms HR API access) must be raised formally at PI Planning with committed delivery dates. The OMS team does not own resolution.
- Sprint velocity assumed at 11 feature stories per sprint. Platform infra stories (ATT-P04 through ATT-P09) are parallelisable by a dedicated infra engineer and may not count against feature team velocity.

OMS PI 1  ·  Sprint Plans  ·  April 2026
