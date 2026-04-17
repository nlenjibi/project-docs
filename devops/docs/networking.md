# OMS Infrastructure — Networking Guide


## Table of Contents

1. [VPC Structure](#1-vpc-structure)
2. [Full Request Flow](#2-full-request-flow)
3. [NAT Gateway](#3-nat-gateway)
4. [API Gateway Path-Based Routing](#4-api-gateway-path-based-routing)
5. [AWS Cloud Map Service Discovery](#5-aws-cloud-map-service-discovery)
6. [Security Groups Overview](#6-security-groups-overview)
7. [Environment Separation](#7-environment-separation)

---

## 1. VPC Structure

Each environment (`dev`, `staging`) has its own **dedicated VPC**. The two environments are fully isolated — no VPC peering between them.

```
┌─────────────────────────────────────────────────────────────────┐
│  AWS VPC  (one per environment: dev / staging)                  │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  PUBLIC SUBNET                                          │   │
│  │                                                         │   │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │   │
│  │  │  VPC Link    │  │ NAT Gateway  │  │  S3 App      │  │   │
│  │  │ (API GW →    │  │ (outbound    │  │  Assets      │  │   │
│  │  │  private)    │  │  only)       │  │  (React SPA) │  │   │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  PRIVATE SUBNET — ECS Fargate                           │   │
│  │                                                         │   │
│  │  Attendance · Seating · Remote/OOO · Visitor            │   │
│  │  Events · Supplies & Assets · Notification              │   │
│  │                                                         │   │
│  │  AWS Cloud Map (oms.local DNS namespace)                │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  DATABASE / CACHE LAYER (Private)                       │   │
│  │                                                         │   │
│  │  RDS PostgreSQL (v2: shared · v3: per service)          │   │
│  │  ElastiCache Redis (shared, t3.micro)                   │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                 │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  MESSAGING / DATA ENGINEERING (Private)                 │   │
│  │                                                         │   │
│  │  Amazon SQS · Amazon SES                                │   │
│  │  S3 Staging Bucket · AWS Glue · Athena · data_warehouse_db │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### What lives in the Public Subnet and why

| Resource | Reason for public subnet |
|----------|--------------------------|
| **VPC Link** | Bridges API Gateway (outside VPC) into the private subnet. Must be accessible from the AWS API Gateway service endpoint |
| **NAT Gateway** | Requires a public IP (Elastic IP) to forward outbound traffic from private resources to the internet |
| **S3 App Assets** | S3 is a regional service endpoint; CloudFront fetches React build artifacts via HTTPS. Not strictly "in" the subnet but associated via bucket policy and origin access identity |

### What lives in the Private Subnet and why

All ECS Fargate tasks, RDS instances, ElastiCache Redis, and SQS consumers run in the private subnet. They have **no public IP addresses**. This means:

- Inbound access is only possible via VPC Link → API Gateway (controlled)
- Outbound access to the internet routes through the NAT Gateway (controlled, outbound-only)
- Database and cache resources are unreachable from the public internet

---

## 2. Full Request Flow

### API Request (user-initiated)

```
Step 1: User sends HTTPS request
        └─► Route 53
              Purpose: DNS resolution for the OMS domain
              Returns: CloudFront distribution IP

Step 2: Route 53 → CloudFront
              Purpose: CDN, SSL termination, edge caching
              Static assets: served directly from S3 origin (React build)
              API requests: forwarded to API Gateway origin

Step 3: CloudFront → API Gateway
              Purpose: Auth, rate limiting, path-based routing
              JWT validation: API Gateway calls SSO/OIDC token introspection endpoint
              (outbound via NAT Gateway if SSO is external)
              Throttle: per-IP and per-key limits configured in API Gateway usage plans

Step 4: API Gateway → VPC Link
              Purpose: Secure private channel from API Gateway into the VPC
              Replaces: ALB (removed as redundant — see ADR-001)
              Protocol: HTTP (API Gateway terminates TLS; internal traffic is HTTP)

Step 5: VPC Link → ECS Service (Private Subnet)
              Path /api/attendance/*   → Attendance Service tasks
              Path /api/seating/*      → Seating Service tasks
              Path /api/remote/*       → Remote/OOO Service tasks
              Path /api/visitor/*      → Visitor Service tasks
              Path /api/events/*       → Events Service tasks
              Path /api/supplies/*     → Supplies & Assets Service tasks
              Path /api/notifications/*→ Notification Service tasks

Step 6: ECS Service → RDS / Redis / SQS
              ECS task reads/writes its own RDS schema (v2) or instance (v3)
              ECS task reads/writes Redis for cache (Seating, Notification)
              Notification Service enqueues to SQS asynchronously

Step 7: SQS → SES (for email)
              Notification Service consumer polls SQS
              On dequeue: calls SES to deliver email
              On failure: message moved to Dead Letter Queue (DLQ)
```

### Static Asset Request (frontend)

```
User → Route 53 → CloudFront → S3 (React build: JS · CSS · HTML · images)
```

CloudFront caches static assets at the edge. S3 is the origin. No API Gateway or VPC Link involved.

### Service-to-Service Request (internal)

```
Service A (e.g., Attendance)
  │  HTTP call to notification.oms.local
  ▼
AWS Cloud Map DNS resolution
  │  Returns: private IP of a running Notification Service task
  ▼
Notification Service (ECS task in private subnet)
  │  enqueues to SQS
  ▼
SQS → SES (email delivery)
```

---

## 3. NAT Gateway

### What it is

A **NAT Gateway** sits in the public subnet with an Elastic IP address. It translates outbound TCP/UDP traffic from private subnet resources (which have no public IP) to the public internet, and maps return traffic back to the originating private resource.

It is **strictly outbound-only** — no inbound connections can be initiated from the internet through the NAT Gateway.

### Why it is needed

ECS Fargate tasks run in the private subnet with no public IP. Several runtime operations require internet or AWS public endpoint access:

| Outbound call | Source | Destination | Why via NAT |
|---------------|--------|-------------|-------------|
| **ECR image pull** | ECS Fargate task (on startup) | `ecr.amazonaws.com` | ECS must pull the Docker image from ECR on every task start. ECR is a public endpoint unless a VPC endpoint is configured |
| **SSO / OAuth 2.0 token validation** | API Gateway Lambda authorizer / ECS service | Company SSO provider | JWT validation requires calling the OIDC well-known endpoint or token introspection URL |
| **Amazon SES email delivery** | Notification Service | `email.us-east-1.amazonaws.com` | SES is called via the AWS SDK from inside the ECS task |
| **Arms HR system sync** | TBD service | Arms HR API (external) | Integration with the company HR system for employee data sync |
| **AWS Athena queries** | Data engineering pipeline / ETL job | `athena.amazonaws.com` | Athena is a public-endpoint service; queries from inside the VPC go through NAT unless a VPC endpoint is added |

### Cost optimisation note

NAT Gateway charges ~$0.045/hour ($32.40/month) per gateway plus $0.045/GB of data processed. Two optimisations reduce this cost:

1. **VPC Endpoints for ECR and S3** — ECR pulls and S3 access do not go through the NAT Gateway, saving data processing charges on image pulls. This is a recommended future optimisation.
2. **One NAT Gateway per environment** — v2 and v3 each use one NAT Gateway per VPC (one for dev, one for staging), not one per AZ. For dev/staging this is acceptable; prod should use per-AZ NAT Gateways for high availability.

---

## 4. API Gateway Path-Based Routing

API Gateway routes requests to the correct ECS service based on the URL path. All routes pass through VPC Link into the private subnet.

| Path Pattern | Target ECS Service | Example |
|-------------|-------------------|---------|
| `/api/attendance/*` | Attendance Service | `GET /api/attendance/sessions` |
| `/api/seating/*` | Seating Service | `POST /api/seating/bookings` |
| `/api/remote/*` | Remote/OOO Service | `GET /api/remote/requests` |
| `/api/visitor/*` | Visitor Service | `POST /api/visitor/pre-register` |
| `/api/events/*` | Events Service | `GET /api/events/{id}/rsvp` |
| `/api/supplies/*` | Supplies & Assets Service | `POST /api/supplies/requests` |
| `/api/notifications/*` | Notification Service | `GET /api/notifications/preferences` |
| `/*` (non-API) | CloudFront → S3 | React SPA (HTML/JS/CSS) |

It might change base on the services I get from the backend

### Authentication at the Gateway

Every API request passes through API Gateway's JWT authorizer before reaching a service. The JWT authorizer:

1. Extracts the `Authorization: Bearer <token>` header
2. Validates the token signature against the SSO OIDC JWK Set endpoint
3. Checks token expiry, issuer, and audience claims
4. On success: forwards the request with `X-User-*` headers stripped in to the downstream service
5. On failure: returns `401 Unauthorized` — the request never reaches ECS

Token validation is **server-side only** — the React frontend never exposes token validation logic.

---

## 5. AWS Cloud Map Service Discovery

### Namespace

| Property | Value |
|----------|-------|
| Namespace name | `oms.local` |
| Namespace type | Private DNS (VPC-scoped) |
| DNS resolution | Internal to the VPC only |
| Registration | Automatic via ECS service integration |

### Service DNS Names

| Microservice | Cloud Map DNS Name | Port |
|-------------|-------------------|------|
| Attendance | `attendance.oms.local` | 8080 |
| Seating | `seating.oms.local` | 8080 |
| Remote/OOO | `remote.oms.local` | 8080 |
| Visitor | `visitor.oms.local` | 8080 |
| Events | `events.oms.local` | 8080 |
| Supplies & Assets | `supplies.oms.local` | 8080 |
| Notification | `notification.oms.local` | 8080 |

### How Registration Works

1. ECS starts a new Fargate task for a service
2. ECS registers the task's private IP with Cloud Map under the service's DNS name
3. Cloud Map updates the DNS A record
4. Other services can immediately resolve the new task IP
5. When ECS stops or replaces a task, it deregisters the old IP from Cloud Map
6. Cloud Map removes the stale DNS record

Because multiple tasks can run simultaneously (auto-scaling), Cloud Map returns multiple A records for the same service name. The calling service's HTTP client performs client-side load balancing across returned IPs (round-robin by default in most HTTP clients).

### Internal API Call Example

```java
// Notification Service → Seating Service (Java Spring Boot)
RestTemplate restTemplate = new RestTemplate();
String url = "http://seating.oms.local:8080/internal/availability";
AvailabilityResponse response = restTemplate.getForObject(url, AvailabilityResponse.class);
```

No hardcoded IPs. No service registry client library. Standard DNS resolution, resolved inside the VPC by Cloud Map.

### Data Engineering Pipeline Usage

The data engineering pipeline resolves service `/internal/export` endpoints via Cloud Map:

```
attendance.oms.local/internal/export
seating.oms.local/internal/export
remote.oms.local/internal/export
visitor.oms.local/internal/export
events.oms.local/internal/export
supplies.oms.local/internal/export
```

---

## 6. Security Groups Overview

Security groups act as virtual firewalls for each resource layer. The following table describes the intended allow-rules. All rules are **deny by default** — only explicitly listed traffic is permitted.

| Resource | Inbound From | Port | Purpose |
|----------|-------------|------|---------|
| **ECS Tasks** | VPC Link | 8080 | API Gateway requests via VPC Link |
| **ECS Tasks** | ECS Tasks (same VPC) | 8080 | Service-to-service calls via Cloud Map |
| **ECS Tasks** | Prometheus (company network) | 8080 | `/metrics` scrape (requires VPN/peering) |
| **RDS PostgreSQL** | ECS Tasks | 5432 | Database connections from services |
| **ElastiCache Redis** | ECS Tasks | 6379 | Cache read/write from services |
| **SQS** | ECS Tasks (Notification) | 443 (HTTPS) | SQS enqueue and dequeue via AWS SDK |
| **NAT Gateway** | ECS Tasks | All | Outbound internet (egress only) |

> **Note:** Security group rules should follow the principle of least privilege. Each service's ECS task should only be permitted to reach its own RDS instance (v3) or the shared RDS (v2), not other services' databases.

---

## 7. Environment Separation

| Property | dev | staging | prod |
|----------|-----|---------|------|
| **VPC** | Dedicated | Dedicated | TBD (planned) |
| **Subnets** | Separate | Separate | TBD |
| **RDS** | Separate instance(s) | Separate instance(s) | TBD |
| **Redis** | Separate instance | Separate instance | TBD |
| **NAT Gateway** | 1 (single-AZ) | 1 (single-AZ) | TBD (multi-AZ for HA) |
| **ECS Task Count** | 1 per service | 1–2 per service | TBD |
| **CodeDeploy traffic shift** | All-at-once | Canary (10% → 100%) | TBD |
| **Cost** | ~$72/month (v2) | ~$72/month (v2) | TBD |

**How environments are isolated:**

- Separate VPCs ensure no cross-environment network traffic
- Separate RDS instances ensure data isolation
- Separate ECS clusters (or ECS services in the same cluster with environment-prefixed names — TBD)
- Separate ECR image tags: `dev-<commit>`, `staging-<commit>`, `prod-<commit>`
- Separate API Gateway stages: `/dev`, `/staging`, `/prod`

> **Note:** Production environment networking design (Multi-AZ, WAF on CloudFront, multi-AZ NAT Gateways, RDS Multi-AZ) is TBD and will be documented when prod provisioning begins.
