# REST vs GraphQL API Gateway — Research Report

> **Context:** Office Management System (OMS), 8-service microservices architecture (post-merge),  
> Angular frontend, Java/Spring Boot services, AWS EKS, Kafka event bus, 2–5 locations, hundreds of employees.

---

## 1. What They Are

### REST (Representational State Transfer)
A **design style** for APIs built on top of HTTP. Each resource has a URL, and standard HTTP verbs (`GET`, `POST`, `PATCH`, `DELETE`) define the operation. The client requests a fixed endpoint and receives a fixed response shape.

```
GET  /api/v1/users/123               → returns user profile
POST /api/v1/seat-bookings           → creates a booking
GET  /api/v1/attendance/team/456     → returns team attendance
```

REST API Gateway = a reverse proxy that routes HTTP requests to the correct downstream service based on path prefix.

### GraphQL
A **query language for APIs** developed by Facebook. Instead of many fixed endpoints, there is typically **one endpoint** (`/graphql`). The client sends a query that describes exactly what data it wants — including nested relationships — and the server returns exactly that shape.

```graphql
query {
  user(id: "123") {
    name
    attendance(month: "2026-04") { date status }
    seatBookings(upcoming: true) { date seat { zone floor } }
    remoteRequests(status: PENDING) { dates }
  }
}
```

GraphQL API Gateway = a schema federation layer that stitches together multiple services' schemas into one unified graph and resolves each field by calling the appropriate downstream service.

---

## 2. Core Architectural Differences

| Dimension | REST | GraphQL |
|---|---|---|
| **Endpoints** | One URL per resource/operation | Single `/graphql` endpoint |
| **Data shape** | Fixed — server decides what comes back | Flexible — client specifies exactly what it needs |
| **Over-fetching** | Common — you get all fields whether you need them or not | Eliminated — client requests only what it needs |
| **Under-fetching** | Common — multiple round trips to assemble a view | Eliminated — one query can fetch nested data across services |
| **Versioning** | URL versioning (`/api/v1/`, `/api/v2/`) or headers | Schema evolution — fields deprecated, not versioned |
| **Contract testing** | Mature tooling (Pact, OpenAPI) | Limited tooling; Pact does not natively support GraphQL |
| **Caching** | HTTP cache (CDN, browser) works naturally on GET requests | HTTP caching broken — all queries are POST to one endpoint |
| **Error handling** | HTTP status codes (200, 400, 404, 500) | Always returns HTTP 200; errors are in the response body |
| **Gateway complexity** | Thin router — no business logic, no schema | Schema federation requires resolvers, a stitching layer, and runtime type resolution |
| **Security surface** | Per-endpoint RBAC, rate limiting by path | Must implement field-level auth; introspection can leak schema |
| **Tooling maturity** | 30+ years of tooling | Mature for single-service; federation is still evolving |

---

## 3. How Each Works as an API Gateway

### REST API Gateway

```
Angular SPA
    │  HTTP request: GET /api/v1/attendance/team/456
    ▼
API Gateway (AWS API Gateway / Kong)
  1. Validate session cookie → calls auth-service /validate
  2. Match path prefix → /api/v1/attendance/** → attendance-service
  3. Forward request with user context header
  4. Return response
    ▼
attendance-service
  → validates internal JWT
  → role + location check
  → returns data
```

The gateway has **zero knowledge of data shapes**. It authenticates, routes by path, and logs. Nothing else.

### GraphQL API Gateway

```
Angular SPA
    │  POST /graphql
    │  { query: "{ user { attendance seatBookings remoteRequests } }" }
    ▼
GraphQL Gateway (Apollo Federation / GraphQL Mesh)
  1. Parse and validate the query against the unified schema
  2. Plan the query — which subgraph resolves which field?
  3. Call auth-service for user fields
  4. Call attendance-service for attendance fields
  5. Call seating-service for seatBookings fields
  6. Call remote-service for remoteRequests fields
  7. Merge all results into one response
    ▼
Client receives one composed response
```

The gateway has **deep knowledge of every service's data model** and is responsible for orchestrating calls across services. It is no longer a thin router — it is a **data orchestration layer**.

---

## 4. Effectiveness

### REST Effectiveness for OMS

**Strong fit for OMS because:**

- **Service-per-bounded-context model** — each service owns clearly-bounded data. REST endpoints map cleanly to service boundaries. There is no need to "stitch" data at the gateway.
- **Pact contract testing** — OMS uses consumer-driven contract testing with Pact to guarantee API compatibility across 8 services. Pact has first-class REST support. The CI pipeline verifies that no service can deploy a breaking API change. This is a core architectural guarantee in OMS.
- **Thin gateway principle** — the OMS architecture mandates: *"the API Gateway contains zero business logic — it authenticates, routes, and logs."* REST lets the gateway stay thin. GraphQL forces the gateway to become a data orchestration layer.
- **Resilience4j circuit breakers** — each inter-service REST call has its own circuit breaker, retry, and fallback. These map naturally to individual REST endpoints.
- **Internal JWT per service** — Zero Trust requires every service to validate the caller's identity. With REST, the gateway forwards the request with a user context header and each service validates independently. Clean and explicit.

**Where REST has friction:**

- **API Composition for dashboards** — the manager dashboard requires data from attendance, remote, and seating services. With REST, the API Gateway or a BFF layer must make 3 parallel calls and aggregate the response. This is additional code.
- **Over-fetching on some responses** — a user list endpoint returns all fields even if the Angular component only needs name and email. Minor at this scale.

### GraphQL Effectiveness for OMS

**Where GraphQL helps:**

- **Dashboard queries** — one GraphQL query replaces 3 parallel REST calls for the manager dashboard. The client describes exactly what it needs and gets exactly that.
- **Angular flexibility** — different Angular components can request different field subsets from the same query. No duplicate endpoints for mobile vs desktop views.
- **Reduced over-fetching** — the Angular SPA never receives fields it does not render.

**Where GraphQL creates problems for OMS:**

- **Breaks Pact contract testing** — OMS's CI pipeline relies on Pact to catch breaking API changes between services. Pact does not natively support GraphQL schemas. Replacing REST contracts with GraphQL schema contracts requires a completely different toolchain (GraphQL schema linting, schema change detection), and the guarantees are weaker.
- **Gateway becomes a business logic layer** — the GraphQL gateway must understand the data models of all 8 services to write resolvers. This creates a central point that knows everything about every service — the opposite of microservices independence.
- **Federation complexity** — Apollo Federation or GraphQL Mesh requires sub-graphs, resolver configuration, schema stitching, and a runtime query planner. This is a non-trivial operational burden on top of already managing 8 services, Kafka, EKS, and 8 RDS instances.
- **HTTP caching breaks** — all GraphQL queries are POST requests to `/graphql`. CDN and browser caching (which work naturally on GET requests for seat availability, user profiles, etc.) are no longer available without custom caching logic.
- **Error handling complexity** — GraphQL always returns HTTP 200. Partial failures (e.g., attendance data available but seating data unavailable) return mixed success/error in the response body. OMS's existing circuit breaker + fallback model maps cleanly to HTTP status codes; it does not map cleanly to GraphQL's error model.
- **Security surface increases** — in GraphQL, introspection can expose the entire schema to any authenticated caller. Field-level authorisation (can this user query `attendance` for another user?) must be implemented inside resolvers, not at the HTTP layer. Role + location checks become significantly more complex to enforce consistently across a federated schema.
- **Team expertise** — the OMS team's expertise is Angular + Java/Spring Boot + REST. Introducing GraphQL federation requires new tooling (Apollo Server/Federation, or GraphQL Mesh) on top of the existing stack, with no identified team member who has deep GraphQL federation experience.

---

## 5. Cost Comparison

### REST API Gateway (AWS API Gateway or Kong)

**AWS API Gateway:**
- ~$3.50 per million API calls
- ~$0.09/GB data transfer
- No infrastructure to manage — fully serverless
- At OMS scale (hundreds of employees): cost is minimal (~$15–30/month estimated)

**Kong (self-hosted on EKS):**
- No per-request cost (already running on EKS nodes)
- Adds ~2 pods to the cluster (marginal cost within existing EKS node group)
- Requires ops overhead to manage Kong configuration

### GraphQL Gateway (Apollo Federation or GraphQL Mesh)

- Apollo Server / Federation: open source, runs as a container on EKS
- Marginal hosting cost (2 pods)
- **Hidden costs:**
  - Developer time to write and maintain resolvers for every service's schema
  - Developer time to write and maintain the federated schema
  - Additional testing — schema tests, resolver unit tests, integration tests against the gateway
  - Operational time when the query planner misbehaves or resolver N+1 problems appear
  - Apollo Studio (schema registry, performance monitoring): $0–$500/month depending on tier

**Cost verdict:** Direct infrastructure cost is similar. The real cost of GraphQL is **developer time** — building and maintaining a federation layer for 8 services is significant ongoing work that delivers limited benefit at OMS scale.

---

## 6. Enterprise and Organizational Benefits

### REST — Organizational Benefits

| Benefit | Detail |
|---|---|
| **Mature contract testing** | Pact has been used in enterprise microservices for years; full CI integration |
| **Standardised tooling** | OpenAPI 3.0 documents every endpoint; Swagger UI auto-generates API docs |
| **Easy onboarding** | Every backend developer understands REST; no learning curve |
| **Independent deployability** | Services can change their REST API (with Pact guard) without touching the gateway |
| **Audit/compliance simplicity** | HTTP access logs capture every endpoint call with path, method, status — easily parsed |
| **Rate limiting by endpoint** | Different rate limits per endpoint (`/api/v1/attendance/export` vs `/api/v1/users`) is trivial |

### GraphQL — Organizational Benefits

| Benefit | Detail |
|---|---|
| **Single API surface** | Frontend team queries one endpoint instead of knowing 8 service URLs |
| **Rapid UI iteration** | Frontend can add new data requirements without a backend endpoint change |
| **Self-documenting** | Introspection allows tools like GraphiQL to explore the schema interactively |
| **Reduced API versioning** | Fields are deprecated rather than versioned — smoother evolution |
| **Mobile-optimised** | If a mobile app arrives, it can request smaller payloads without backend changes |

---

## 7. The API Composition Problem — Does GraphQL Solve Something Real?

The one genuine REST weakness in OMS is the **manager dashboard**: it requires data from three services, meaning 3 parallel REST calls with aggregation.

```
Dashboard request
  ├── GET /api/v1/attendance/team/{id}   → attendance-service
  ├── GET /api/v1/teams/{id}/schedule    → remote-service
  └── GET /api/v1/seat-bookings/team/{id} → seating-service
       ↓
  Composed response
```

GraphQL would replace this with one query. However, the REST solution is not burdensome:

**Option A — BFF (Backend for Frontend) endpoint:**
A dedicated `GET /api/v1/dashboard/manager/{id}` endpoint in a lightweight BFF layer (or within the API Gateway) makes the 3 calls in parallel and aggregates. This is ~50 lines of Java. It does not require a GraphQL layer.

**Option B — Client-side composition:**
The Angular SPA makes 3 parallel calls using `forkJoin` and assembles the dashboard in the component. This is standard practice and keeps the backend stateless.

Both options solve the composition problem without introducing federation complexity. The composition requirement does **not** justify a GraphQL gateway for OMS.

---

## 8. When GraphQL Would Be the Right Choice

GraphQL is genuinely superior when:

- The **frontend teams are many and have rapidly changing, diverse data requirements** — different mobile apps, web apps, and partner integrations all need different subsets of the same data
- **You have a public API** used by external developers who benefit from self-service schema exploration
- **The team has existing GraphQL expertise** and the operational model is already understood
- **You have a content-heavy system** with deep object graphs (e.g., a CMS, a social network, an e-commerce catalogue with many nested relationships)
- **No contract testing requirement** — the organisation accepts schema registry as the compatibility mechanism

For the OMS — an internal workplace management system with a single Angular frontend, Pact contract testing as a CI requirement, and a team that is expert in Java/Spring Boot + REST — none of these conditions apply.

---

## 9. Decision: REST for OMS

### Confirmed reasons REST is correct for OMS

| Reason | Detail |
|---|---|
| **Pact contract testing** | Core CI pipeline requirement. Pact does not support GraphQL. Switching would require replacing the entire contract testing strategy. |
| **Thin gateway mandate** | Architecture rule: API Gateway has zero business logic. GraphQL federation violates this. |
| **Zero Trust + per-service JWT validation** | Each service validates every caller via internal JWT. This maps cleanly to REST request forwarding. It does not map cleanly to a federated GraphQL resolver chain. |
| **Team expertise** | Angular + Java/Spring Boot + REST is the mandated stack. No GraphQL expertise on the team. |
| **8 independently deployable services** | Each service owns its API contract (enforced by Pact). With GraphQL federation, the gateway schema couples all services together at the gateway level. |
| **HTTP caching** | Seat availability (`GET /api/v1/locations/{id}/floor-plan`) can be cached at CDN level with REST. Not possible with GraphQL POST. |
| **Operational simplicity** | 8 services + Kafka + EKS + 8 RDS instances is already a complex system to operate. Adding a GraphQL federation layer adds a new class of operational problems (resolver errors, schema validation, N+1 queries) without proportionate benefit. |
| **API Composition is solved simply** | The one place REST requires extra work (dashboard composition) is solved by a BFF endpoint or client-side `forkJoin` — not by introducing a full GraphQL layer. |

---

## 10. Summary Comparison Table

| Dimension | REST | GraphQL | Verdict for OMS |
|---|---|---|---|
| Contract testing (Pact) | Full support | Not supported | **REST** |
| Gateway complexity | Thin router | Data orchestration layer | **REST** |
| Dashboard / composition queries | Requires BFF or client composition | Single query | GraphQL (but solvable with REST) |
| HTTP caching | Native | Broken | **REST** |
| Security (RBAC + location checks) | Per-endpoint, well-understood | Field-level, complex in federation | **REST** |
| Zero Trust + internal JWT | Simple forwarding | Complex resolver chain | **REST** |
| Over-fetching | Yes | Eliminated | GraphQL |
| Team expertise | Java/Spring Boot REST — existing | New tooling required | **REST** |
| Independent service deployments | Preserved | Gateway schema couples services | **REST** |
| Error handling | HTTP status codes | Always 200; errors in body | **REST** |
| Operational overhead | Low | Moderate–high (federation layer) | **REST** |
| Future mobile app | Extra endpoints or BFF needed | One query | GraphQL (future consideration) |

**Score: REST wins 9 of 12 dimensions for OMS specifically.**

---

## 11. Recommended API Gateway Technology

Since REST is confirmed, the two gateway candidates are:

### AWS API Gateway
- Fully managed — no infrastructure to deploy or maintain
- Native integration with AWS Secrets Manager, CloudWatch, IAM
- Per-request pricing (~$15–30/month at OMS scale)
- Some limitations on custom plugin/middleware behaviour

### Kong (self-hosted on EKS)
- More flexible — custom plugins for auth, rate limiting, request transformation
- Already running on the EKS cluster — marginal additional cost
- Requires ops knowledge to manage Kong Admin API and declarative config
- Richer rate limiting (per-user, per-endpoint, per-location-header)

**Recommendation:** Start with **AWS API Gateway** for Phase 1 (lower ops overhead at launch). Migrate to **Kong** if the team needs custom gateway plugins, more granular rate limiting rules, or wants to avoid AWS API Gateway's per-call pricing at higher scale. This decision is recorded as an open item in the architecture (the gateway selection is listed as a pre-development action in the architecture docs).

---

## 12. Bottom Line

REST is the correct API Gateway pattern for OMS. The decision is not about REST being "better" than GraphQL in the abstract — GraphQL solves real problems for large public APIs with many consumer types. For OMS specifically:

- The Pact contract testing requirement alone settles the question (GraphQL is incompatible with Pact)
- The thin gateway mandate rules out federation
- The team's existing REST expertise avoids a significant learning curve
- The one genuine REST weakness (dashboard composition) is solved trivially with a BFF endpoint

GraphQL is worth reconsidering **only if** OMS later adds a mobile app with distinct data requirements and the team acquires GraphQL federation expertise — at which point a GraphQL layer could sit *in front of* the existing REST services without requiring them to change.
