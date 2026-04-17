# Office Management System (OMS) — Infrastructure

## Overview

The Office Management System (OMS) is a microservices-based platform deployed on AWS, designed to manage core office operations including attendance tracking, hot-desk booking, remote/OOO management, visitor registration, event scheduling, supplies management, and notifications.

The infrastructure is defined in two variants — **v2** and **v3** — that differ only in how PostgreSQL databases are provisioned. Everything else (networking, compute, CI/CD, observability, messaging) is identical between the two.

> **Note:** Current active environments are `dev` and `staging` only.

---

## Architecture Variants

### v2 — Shared Database (Budget-Optimised)

![OMS AWS Architecture v2 — Shared RDS](diagrams/xyz%20v2%20update.jpg)

In v2, all seven microservices share **one RDS PostgreSQL instance** (`db.t3.micro`). Each service gets its own **logical schema** within that instance, with dedicated credentials and independent migrations.

**Key characteristics:**
- 1 × RDS PostgreSQL instance (db.t3.micro)
- 7 schemas: `attendance` | `seating` | `remote` | `visitor` | `events` | `supplies` | `notification`
- Services connect with their own DB credentials — no cross-schema access
- Migrations per service are independent
- Estimated cost: **~$145/month** for dev + staging combined

**When to use v2:**
- When a team is starting out and is on a tight budget pressure
- Services are unlikely to need independent DB scaling
- Operational simplicity is preferred over strict isolation

---

### v3 — Database per Service (Full Isolation)

![OMS AWS Architecture v3 — DB per Service](diagrams/xyz%20v3%20update.jpg)

In v3, every microservice gets its own **dedicated RDS PostgreSQL instance** (`db.t3.micro` each). There is zero shared DB infrastructure between services.

**Key characteristics:**
- 7 × RDS PostgreSQL instances (one per service)
- No cross-service DB access — enforced at infrastructure level
- Services communicate only via Cloud Map DNS + internal API calls
- Estimated cost: **~$200–220/month** for dev + staging combined

**When to use v3:**
- Full microservices isolation is required
- Services need independent DB scaling, failover, or backup policies
- Budget allows the additional ~$55–75/month over v2

> **Warning:** v3 sits at or slightly above the $200/month budget. Use Fargate Spot and schedule dev environment shutdowns outside office hours to stay within budget — see [Cost Estimate](docs/cost-estimate.md).

---

## Quick Comparison Table (v2 vs v3)

| Dimension | v2 — Shared DB | v3 — DB per Service |
|-----------|---------------|---------------------|
| **DB model** | 1 shared RDS, 7 schemas | 7 dedicated RDS instances |
| **Estimated cost (dev+staging)** | ~$145/month ✅ | ~$200–220/month ⚠️ |
| **DB isolation level** | Logical (schema + credentials) | Physical (separate instance) |
| **Cross-service DB risk** | Possible if misconfigured | Impossible at infra level |
| **Independent DB scaling** | Not possible | Per service |
| **Independent backup/restore** | Not supported | Per service |
| **Migration management** | Per service (independent) | Per service (independent) |
| **Operational complexity** | Low | Medium |
| **Recommended for** | Budget-constrained start | Full isolation / scale-ready |
| **Migration path** | Promote schemas to own RDS when ready | Already separated |
| **Budget headroom** | ~$55/month remaining | ~$0–negative |

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| **Frontend** | React (static assets served via S3 + CloudFront) |
| **Backend** | Java / Spring Boot |
| **Containerisation** | Docker on ECS Fargate |
| **Database** | PostgreSQL on Amazon RDS |
| **Cache** | Redis on Amazon ElastiCache |
| **Authentication** | SSO / OAuth 2.0 / OIDC (server-side token verification) |
| **CI/CD** | Jenkins → Amazon ECR → AWS CodeDeploy → ECS Blue/Green |
| **Service Discovery** | AWS Cloud Map (`oms.local` namespace) |
| **API Gateway** | Amazon API Gateway + VPC Link (no ALB) |
| **Messaging** | Amazon SQS + Amazon SES |
| **Observability** | Prometheus + Grafana (company-hosted, external) |
| **Container Registry** | Amazon ECR |
| **DNS** | Amazon Route 53 |
| **CDN** | Amazon CloudFront |
| **Data Engineering** | AWS Glue ETL + AWS Athena (optional) |
| **Data Warehouse** | Dedicated RDS PostgreSQL (`data_warehouse_db`) |
| **IaC** | Terraform  |

---

## Microservices

| Service | Emoji | Responsibility | v2 Schema | v3 RDS Instance |
|---------|-------|---------------|-----------|-----------------|
| Attendance | 🪑 | Badge events, WorkSession, AttendanceRecord, two-pass nightly resolution | `attendance` | `attendance_db` |
| Seating | 📋 | Hot-desk booking, real-time availability, floor plan | `seating` | `seating_db` |
| Remote/OOO | 🏠 | Request and approval workflow, manager delegation | `remote` | `remote_ooo_db` |
| Visitor | 🚶 | Pre-registration, walk-in, NDA per visit | `visitor` | `visitor_db` |
| Events | 📅 | RSVP, capacity management, recurring series (parent/child pattern) | `events` | `events_db` |
| Supplies & Assets | 📦 | Two-stage approval, batch stock tracking | `supplies` | `supplies_assets_db` |
| Notification | 🔔 | Async email queue via SQS → SES | `notification` | `notification_db` |

All services expose `/internal/export` endpoints for the data engineering pipeline. No service accesses another service's database directly.
The services may increase or descrease base on the info from backend

---

## Request Flow

```
User
  │  HTTPS
  ▼
Route 53 (DNS resolution)
  │
  ▼
CloudFront (CDN · SSL termination)
  │  Static assets ──────────────────────────────────────► S3 (React build output)
  │  API requests
  ▼
API Gateway (auth · rate limiting · path-based routing)
  │  JWT validation ◄──────────────────────────────────── SSO / OAuth 2.0 / OIDC
  │  VPC Link (no ALB)
  ▼
VPC Link (bridges API Gateway into the private subnet)
  │  /api/attendance/*  ──► Attendance Service (ECS Fargate)  ──► RDS / Redis
  │  /api/seating/*     ──► Seating Service    (ECS Fargate)  ──► RDS / Redis
  │  /api/remote/*      ──► Remote/OOO Service (ECS Fargate)  ──► RDS
  │  /api/visitor/*     ──► Visitor Service    (ECS Fargate)  ──► RDS
  │  /api/events/*      ──► Events Service     (ECS Fargate)  ──► RDS
  │  /api/supplies/*    ──► Supplies Service   (ECS Fargate)  ──► RDS
  │  /api/notifications/*► Notification Service(ECS Fargate)  ──► RDS / Redis / SQS
  ▼
SQS (async email queue)
  │
  ▼
SES (email delivery)
```

> **Note — Why ALB was removed:** An Application Load Balancer between API Gateway and ECS would be redundant. API Gateway with VPC Link already handles TLS termination, authentication, rate limiting, and path-based routing. Adding an ALB would add ~$16–20/month per environment with no functional benefit in this architecture.

> **Note — Why SQS + SES instead of RabbitMQ / Amazon MQ:** The OMS notification pattern is one-directional email dispatch. RabbitMQ and Amazon MQ are designed for complex routing, fan-out, and protocol richness (AMQP) that is not needed here. SQS is serverless, pay-per-use (~$0.40/million requests), has built-in DLQ support, and SES handles email delivery at ~$0.10/1,000 emails. This avoids operating a broker cluster entirely.

---

## Service Discovery (AWS Cloud Map)

All ECS Fargate services register themselves in the `oms.local` private DNS namespace via **AWS Cloud Map** on startup and deregister on shutdown.

Services call each other using stable DNS names — no hardcoded IPs or URLs. Auto-scaling does not break inter-service communication because Cloud Map updates DNS records automatically.

**Example DNS names:**

| Service | Cloud Map DNS |
|---------|--------------|
| Attendance | `attendance.oms.local` |
| Seating | `seating.oms.local` |
| Remote/OOO | `remote.oms.local` |
| Visitor | `visitor.oms.local` |
| Events | `events.oms.local` |
| Supplies & Assets | `supplies.oms.local` |
| Notification | `notification.oms.local` |

See [Networking Guide](docs/networking.md) for full Cloud Map DNS details.

---

## CI/CD Pipeline

```
Developer
  │  git push
  ▼
GitHub (Source Control)
  │  webhook
  ▼
Jenkins (company-hosted — no new AWS instance provisioned)
  │  docker build + docker push
  ▼
Amazon ECR (Container Registry)
  │  trigger deploy
  ▼
AWS CodeDeploy
  │  Blue/Green deployment
  ▼
ECS Fargate (traffic shifted after health checks pass)
```

**Blue/Green deployment flow:**
1. CodeDeploy registers the new task definition as the "green" deployment
2. ECS starts new tasks alongside existing "blue" tasks
3. Health checks pass on the green deployment
4. Traffic is shifted from blue → green
5. Blue tasks are terminated after the configurable wait period

---

## Environments

| Environment | Status | VPC | Notes |
|-------------|--------|-----|-------|
| `dev` | Active | Dedicated VPC | Used for active development |
| `staging` | Active | Dedicated VPC | Pre-production validation |
| `prod` |  Not yet provisioned |TBD | Not yet provisioned |

Each environment has its own separate VPC to prevent cross-environment traffic. Cost estimates cover dev + staging combined. Prod costs are TBD.

---

## Cost Estimates

| Variant | Monthly Estimate | Budget | Status |
|---------|-----------------|--------|--------|
| v2 (Shared DB) | ~$145/month | $200/$300 | ✅ $55 headroom |
| v3 (DB per Service) | ~$200–220/month | $200/$300 | ⚠️ At/over budget |

See [Cost Estimate](docs/cost-estimate.md) for full line-item breakdown and optimisation recommendations.

---

## Getting Started

### Prerequisites

- AWS CLI configured with appropriate credentials
- Docker installed locally
- Access to the company Jenkins instance
- Access to Amazon ECR (credentials via Jenkins IAM role)

### Deploying to dev

```bash
# Build and tag the image
docker build -t attendance-service .
docker tag attendance-service:latest <account-id>.dkr.ecr.<region>.amazonaws.com/oms/attendance-service:latest

# Push to ECR (Jenkins handles this in CI — manual push for local testing)
aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <account-id>.dkr.ecr.<region>.amazonaws.com
docker push <account-id>.dkr.ecr.<region>.amazonaws.com/oms/attendance-service:latest
```

> **Note:** Normal deployments are triggered automatically by Jenkins on push to the appropriate branch. Manual pushes are for emergency hotfixes only.

---

## Repository Structure

```
devops/
├── README.md                          # This file
├── oms_architecture_v2_updated.xml    # draw.io source — shared DB variant
├── oms_architecture_v3_updated.xml    # draw.io source — DB per service variant
├── diagrams/
│   ├── xyz v2 update.jpg              # Rendered diagram — v2
│   └── xyz v3 update.jpg              # Rendered diagram — v3
└── docs/
    ├── architecture-decisions.md      # ADRs for all major infrastructure decisions
    ├── networking.md                  # VPC layout, request flow, Cloud Map, security groups
    ├── cost-estimate.md               # Line-item cost breakdown for v2 and v3
    └── services.md                    # Service catalogue — responsibilities, endpoints, config
```
