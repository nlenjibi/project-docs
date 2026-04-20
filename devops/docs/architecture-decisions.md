# OMS Infrastructure — Architecture Decision Records

This document records all major infrastructure decisions made for the Office Management System (OMS) AWS deployment. Each ADR captures the context, the decision taken, the reasoning behind it, and the trade-offs consciously accepted.

---

## ADR-001 — API Gateway + VPC Link Instead of API Gateway + ALB

### Context

The initial architecture sketch included an Application Load Balancer (ALB) sitting between API Gateway and the ECS Fargate private subnet. This is a common pattern for routing HTTP traffic from the internet into private ECS services.

### Decision

Remove the ALB entirely. Use **API Gateway with a VPC Link** to route traffic directly into the private subnet where ECS services run.

### Reasoning

API Gateway already handles everything an ALB would provide in this architecture:

- TLS termination
- Path-based routing (`/api/attendance/*` → Attendance Service, etc.)
- Authentication via JWT validation (delegated to SSO/OIDC)
- Rate limiting and throttling
- Request/response logging

A VPC Link is a direct private connection from API Gateway into the VPC without needing a load balancer in the middle. Adding an ALB on top of this would be a redundant hop — adding latency, cost, and an additional failure point with no compensating benefit.

### Trade-offs accepted

- API Gateway has a 29-second integration timeout — long-running requests (exports, reports) must be handled asynchronously
- API Gateway HTTP APIs support VPC Link but have fewer features than REST APIs — the team selected the HTTP API tier and accepts this constraint
- No per-service load balancer metrics from ALB — service-level traffic metrics come from Prometheus `/metrics` scrape instead

---

## ADR-002 — AWS Cloud Map for Service Discovery


### Context

Microservices need to call each other (e.g., Notification Service called by other services to trigger email, Attendance Service queried by Seating for occupancy). ECS Fargate tasks have dynamic IPs that change on restart or scale-out. A stable addressing mechanism is required.

### Decision

Use **AWS Cloud Map** with a private DNS namespace (`oms.local`) for service-to-service discovery. Every ECS service registers itself on startup and deregisters on shutdown. Services resolve each other by name (e.g., `notification.oms.local`).

### Reasoning

- Cloud Map is natively integrated with ECS — no sidecar or agent required
- DNS registration/deregistration is automatic with ECS service events
- No hardcoded IPs or URLs — scaling does not break inter-service communication
- Private DNS namespace keeps resolution inside the VPC
- Cheaper and simpler than a service mesh (no Envoy sidecars, no data plane)
- AWS charges only for namespace hosting ($0.50/namespace/month) and health checks if enabled

### Trade-offs accepted

- DNS caching can cause brief routing to a terminated task if TTL is not tuned — set ECS service DNS TTL to 10–30 seconds
- No built-in circuit breaking, retries, or mTLS — services implement their own retry logic; mTLS is deferred to a future version if required
- Not as feature-rich as a full service mesh (AWS App Mesh, Istio) — acceptable for this scale

---

## ADR-003 — SQS + SES Instead of RabbitMQ / Amazon MQ


### Context

The Notification Service needs to send transactional emails (booking confirmations, OOO approvals, visitor check-in alerts, etc.) asynchronously. Candidates evaluated: RabbitMQ (self-hosted), Amazon MQ (managed RabbitMQ/ActiveMQ), Amazon SQS + Amazon SES.

### Decision

Use **Amazon SQS** as the async queue and **Amazon SES** for email delivery. No message broker cluster is provisioned.

### Reasoning

The OMS notification pattern is strictly one-directional: a microservice enqueues a message, the Notification Service dequeues it and sends an email. There is no need for:

- Complex routing (topic exchanges, fan-out with multiple subscribers)
- Protocol richness (AMQP, STOMP, MQTT)
- Publish/subscribe patterns across multiple consumers

SQS provides:
- Serverless — no broker instance to manage or patch
- Pay-per-use (~$0.40/million requests — effectively free at OMS scale)
- Built-in Dead Letter Queue (DLQ) for failed messages
- At-least-once delivery with visibility timeout

SES provides:
- ~$0.10 per 1,000 emails
- No SMTP server to operate
- Delivery status tracking, bounce/complaint handling

Total estimated messaging cost: **under $5/month** across both environments.

### Trade-offs accepted

- SQS is not a full message broker — cannot do fan-out to multiple independent consumers without SNS in front of it; accepted because OMS only has one email consumer
- SES requires domain/email verification and may enter sandbox mode on new accounts — team must request production access before sending to external addresses
- No AMQP client support in the application — acceptable as SQS SDK is available for Java/Spring

---

## ADR-004 — ECS Fargate Instead of EC2 or EKS


### Context

OMS has seven microservices that need container orchestration. Options evaluated: EC2 with self-managed Docker, ECS on EC2, ECS Fargate, Amazon EKS (Kubernetes).

### Decision

Use **ECS Fargate** for all microservice deployments.

### Reasoning

- **Serverless compute** — no EC2 instances to patch, right-size, or manage
- **Per-task billing** — pay only for CPU/memory allocated to running tasks (no idle EC2 cost)
- **Auto-scaling** — ECS Application Auto Scaling adjusts task count per service based on CPU/memory metrics
- **Native CodeDeploy integration** — Blue/Green deployments are first-class in ECS Fargate
- **Native Cloud Map integration** — service discovery registers automatically
- **Simpler operational overhead** vs EKS — no control plane cost (~$72/month per EKS cluster), no Kubernetes expertise required in the initial team

EKS was rejected because:
- Control plane cost alone ($72/month × 2 environments = $144/month) would consume most of the $200 budget before any workload runs
- Kubernetes adds significant operational complexity for a team that does not yet have dedicated platform engineering resources

### Trade-offs accepted

- ECS Fargate has a cold-start latency on new task launches (~10–30 seconds) — acceptable for OMS workloads
- Less ecosystem tooling than Kubernetes — acceptable at current scale
- No native pod-level networking policies (ECS uses security groups instead) — acceptable

---

## ADR-005 — Two Database Variants (Shared Schema vs Per-Service RDS)


### Context

The team needed to decide how to provision PostgreSQL for seven microservices within a $200/month budget constraint for dev + staging. True microservices best practice recommends one database per service, but this has direct cost implications.

### Decision

Define and maintain **two infrastructure variants** in parallel:

- **v2** — One shared RDS PostgreSQL instance (`db.t3.micro`), seven logical schemas, separate credentials per service
- **v3** — Seven dedicated RDS PostgreSQL instances (`db.t3.micro` each), one per microservice

Both variants are fully documented and diagrammed. The team chooses one variant per deployment based on budget and isolation requirements.

### Reasoning

A single shared RDS instance at `db.t3.micro` costs ~$13–15/month. Seven separate instances cost ~$91–105/month — a difference of ~$76–90/month. Within a $200/month total budget for two environments, this difference is significant.

v2 provides logical isolation (schemas + credentials) which prevents accidental cross-service reads in application code, while remaining budget-friendly. v3 provides physical isolation, independent failover, independent backup/restore, and independent scaling at higher cost.

Maintaining both variants allows the team to start with v2 and migrate individual services to their own RDS instances as the product matures and the budget grows, without architectural rework of the surrounding infrastructure.

### Trade-offs accepted

- v2: A misconfigured credential could theoretically allow cross-schema access at the DB level — mitigated by IAM-based credential rotation and review
- v2: One instance failure affects all services simultaneously — mitigated by RDS Multi-AZ (available as an upgrade when needed)
- Maintaining two XML diagrams and two sets of Terraform modules adds documentation overhead
- v3 at $200–220/month is at or slightly over the current budget — Fargate Spot and dev scheduling are required to stay within budget

---

## ADR-006 — Blue/Green Deployment via AWS CodeDeploy


### Context

OMS needs zero-downtime deployments for all seven services. Jenkins builds and pushes Docker images to ECR. A deployment mechanism must shift traffic from the old version to the new version safely.

### Decision

Use **AWS CodeDeploy** to orchestrate Blue/Green deployments to ECS Fargate.

### Reasoning

- Native ECS + CodeDeploy integration — no custom scripting required
- Blue/Green ensures the new task set is healthy before any traffic is shifted
- Automatic rollback on failed health checks — the old task set remains running until the new set is confirmed healthy
- Configurable traffic shift (linear, canary, all-at-once) — all-at-once is used for dev; canary (e.g., 10% → 100%) is recommended for staging
- CodeDeploy is free for ECS deployments (no additional charge)
- Jenkins triggers CodeDeploy via the AWS CLI after a successful ECR push — no additional CI tooling needed

### Trade-offs accepted

- Blue/Green temporarily doubles the ECS task count during deployment — Fargate billing increases briefly per deploy
- CodeDeploy deployment groups must be configured per service (7 groups × 2 environments = 14 deployment groups to manage)
- Rollback is fast but not instantaneous (~1–2 minutes) — OMS accepts this for non-critical services

---

## ADR-007 — External Observability (Prometheus + Grafana, Company-Hosted)


### Context

OMS services need to expose metrics for monitoring. Options: AWS CloudWatch (native), AWS Managed Grafana, self-hosted Prometheus + Grafana on EC2, or the company's existing Prometheus + Grafana infrastructure.

### Decision

Use the **company's existing Prometheus and Grafana instances**. No new AWS observability infrastructure is provisioned. All ECS services expose `/metrics` endpoints that the company Prometheus scrapes.

### Reasoning

- Zero additional AWS cost — Prometheus and Grafana are already operated by the company
- No new EC2 instances or managed services to provision
- Grafana dashboards can be created without AWS Managed Grafana cost (~$9/user/month)
- The company already has alerting rules and on-call integrations configured in the existing Prometheus/Grafana stack — OMS benefits from these immediately

CloudWatch was considered but rejected for routine metrics because:
- CloudWatch custom metrics cost $0.30/metric/month — with 7 services × ~20 metrics each, this adds ~$42/month
- The company already has Grafana expertise and dashboards for Spring Boot services

CloudWatch Logs may still be used for ECS container log aggregation (TBD — not shown in current architecture).

### Trade-offs accepted

- Prometheus must be able to reach ECS task `/metrics` endpoints — requires a VPC peering or VPN connection from the company network to the OMS VPC (TBD with network team)
- If the company Prometheus instance is down, OMS has no metrics — no redundancy in the monitoring path
- Alerting is coupled to the company's Grafana alert configuration — OMS-specific alert rules must be added to the shared instance

---

## ADR-008 — NAT Gateway for Private Subnet Outbound Traffic


### Context

ECS Fargate tasks run in a private subnet with no public IP addresses. Several outbound calls must reach the internet or AWS public endpoints:

- ECR image pulls (during task startup)
- SSO / OAuth 2.0 token validation calls
- Amazon SES email delivery
- Arms HR system sync (external)
- AWS Athena queries (data engineering pipeline)

### Decision

Place a **NAT Gateway** in the public subnet. All private subnet outbound traffic routes through the NAT Gateway.

### Reasoning

- NAT Gateway is the standard AWS pattern for private subnet outbound internet access
- ECS tasks in a private subnet cannot have public IPs — a NAT Gateway is required
- One NAT Gateway per VPC (per environment) is sufficient at OMS scale
- NAT Gateway provides outbound-only connectivity — no inbound internet traffic can reach private subnet resources through it

VPC Endpoints for ECR and S3 were considered as a cost optimisation (they eliminate NAT Gateway data processing charges for ECR pulls) — this is noted as a future optimisation if data transfer costs become significant.

### Trade-offs accepted

- NAT Gateway costs ~$32/month per environment (hourly rate + data processing) — this is the largest fixed networking cost in the architecture
- A single NAT Gateway is a potential single point of failure — for staging and dev this is acceptable; prod should consider multi-AZ NAT Gateways
- Data transfer through NAT Gateway is charged at $0.045/GB — ECR pull sizes should be minimised with multi-stage Docker builds

---

## ADR-009 — Shared ElastiCache Redis with Per-Service Key Namespacing

### Context

Multiple OMS services benefit from a caching layer:

- Seating Service: real-time hot-desk availability (read-heavy, high contention)
- Notification Service: rate limiting per user/channel
- Potential future use: session token caching, deduplication

Options: one Redis instance per service, or one shared Redis with key namespaces.

### Decision

Use **one shared ElastiCache Redis instance** (`cache.t3.micro`) with a per-service **key prefix convention** (`seating:`, `notification:`, etc.).

### Reasoning

- A single `cache.t3.micro` costs ~$13/month — seven separate instances would cost ~$91/month
- Redis caches computed results and availability states, not private personal data — sharing is safe
- Key prefix namespacing provides logical isolation without requiring separate instances
- At OMS scale (7 services, dev + staging), one instance provides more than sufficient throughput

### Trade-offs accepted

- A Redis failure affects all services simultaneously — mitigated by the fact that Redis is a cache (services fall back to DB reads on cache miss)
- No memory isolation between services — a runaway cache usage in one service could evict another service's keys; mitigated by setting per-service key TTLs and monitoring eviction metrics
- No auth isolation — all services use the same Redis endpoint; mitigated by keeping Redis in the private subnet with security group restrictions

---

## ADR-010 — Data Warehouse as Separate RDS, Accessed via Internal Service APIs


### Context

The data engineering team needs access to OMS operational data for analytics, reporting, and BI. Options: direct DB connection to microservice databases, database replication, or API-based data export.

### Decision

Data engineers collect data exclusively via **`/internal/export` HTTP endpoints** exposed by each microservice. A dedicated **`data_warehouse_db`** RDS PostgreSQL instance holds the analytical copy. Data engineers never connect directly to any microservice database.

The pipeline is: Service APIs → (optional) S3 staging bucket → (optional) AWS Glue ETL → `data_warehouse_db`.

AWS Athena provides optional ad-hoc SQL on S3 staging data.

### Reasoning

- Direct DB connections from data engineering to microservice databases would violate service ownership boundaries — the service team controls the schema, and direct access bypasses that
- `/internal/export` endpoints allow the service team to control what data is exposed, in what shape, and at what rate
- The data warehouse is a read-only analytical copy — it never writes back to any microservice database
- S3 staging bucket + Glue ETL is optional — for simple cases, the Notification or ETL job can call APIs directly and write to `data_warehouse_db`
- Cloud Map DNS (`attendance.oms.local`, etc.) is used by the data engineering pipeline to resolve service endpoints — no hardcoded URLs

### Trade-offs accepted

- API-based extraction is slower than direct DB replication — acceptable for analytical (not real-time operational) reporting
- `/internal/export` endpoints must be maintained by each service team as first-class APIs
- AWS Glue and Athena are marked OPTIONAL — the data engineering pipeline can be built incrementally
- Athena query costs are per-scan ($5/TB scanned) — data engineers must use partitioned S3 layouts and column pruning to control cost
