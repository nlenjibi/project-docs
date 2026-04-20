# Cost Analysis and Optimisation
## Office Management System (OMS)

**Version:** 4.0 | Scale: 2–5 locations, hundreds of employees

---

## 1. Monthly AWS Cost Estimate (Production)

| Component | Service | Instance/Config | Est. Monthly |
|---|---|---|---|
| Container orchestration | ECS Fargate | 8 services, avg 2 tasks × 0.25 vCPU × 0.5 GB | **~$60–100** |
| API routing + auth | AWS API Gateway HTTP API | ~100K req/day + Lambda Authorizer (300s cache) | **~$3–8** |
| CDN + SPA delivery | Amazon CloudFront | ~50 GB/month transfer; SPA static assets | **~$5–15** |
| Service discovery | AWS Cloud Map | 8 service records, 10s TTL | **~$1** |
| Async messaging | Amazon SNS + SQS | ~2,500 msgs/day; 8 SQS queues; DLQs | **~$0.40** |
| Databases | AWS RDS PostgreSQL | 8 × `db.t3.small` Multi-AZ, 20 GB each | **~$300–480** |
| Email | Amazon SES | ~500 emails/month (notifications) | **~$1** |
| Audit archival | S3 Glacier Instant Retrieval | ~50 GB/year | **~$5–10** |
| Container registry | AWS ECR | 8 repositories, ~2 GB each | **~$2** |
| Log aggregation | CloudWatch Logs | ~10 GB/month log ingestion | **~$5–15** |
| Metrics | Prometheus + Grafana (ECS Fargate) | 2 tasks, 0.5 vCPU each | **~$20** |
| Tracing | Jaeger (ECS Fargate) | 1 task, 0.5 vCPU, 1 GB | **~$15** |
| Secrets | AWS Secrets Manager | 20 secrets × $0.40/secret/month | **~$8** |
| Edge security | AWS WAF | API Gateway WAF — 1 WebACL + OWASP rules | **~$10** |
| Lambda (authorizer) | AWS Lambda | ~600K invocations/month (with 300s cache) | **~$1–2** |
| **Total (estimated)** | | | **~$450–690/month** |

*Costs are indicative. Actual costs depend on traffic volume and right-sizing after load testing.*

Note: CloudFront replaces the public ALB. API Gateway HTTP API is ~70% cheaper than REST API. ECS has 2 fewer permanent services (no gateway/eureka containers).

---

## 2. Broker Cost Decision (Updated)

The message broker was the largest infrastructure decision by cost.

| Broker | AWS Service | Monthly Cost | Notes |
|---|---|---|---|
| Apache Kafka | AWS MSK | ~$463/month | 3 × `kafka.m5.large`, minimum HA |
| RabbitMQ | Amazon MQ | ~$230/month | Active/standby `mq.m5.large` |
| **SQS + SNS** | **Amazon SQS** | **~$0.40/month** | At ~2,500 msgs/day; near-zero |

**Decision for v4.0: SNS + SQS.** Previous concern was "no Spring Cloud Stream binder" — resolved by accepting polyglot messaging: FastAPI services use aioboto3 directly, Spring Boot services use AWS SDK v2 SnsClient/SqsClient. Fan-out pattern (publish once → multiple queues) is solved natively by SNS → multiple SQS subscriptions.

**Annual saving from switching RabbitMQ → SNS+SQS: ~$2,748/year**

**Why not Kafka despite its capabilities:**
- Replay capability (Kafka's primary advantage over RabbitMQ) is not required at OMS scale
- Operational overhead: partition strategy, consumer group offset management, Schema Registry
- Over 1,000× more expensive than SNS+SQS for equivalent HA ($463 vs $0.40/month)

---

## 3. Orchestration Cost Comparison

| Orchestration | Monthly Fixed Cost | Notes |
|---|---|---|
| AWS EKS | **~$75** (control plane) + EC2 nodes | Plus ~$200–400 for 2 t3.medium nodes |
| **AWS ECS Fargate** | **$0** control plane | Pay per task CPU/memory second |

**ECS Fargate container cost estimate (v4.0):**
- 8 services × 2 tasks + Prometheus + Grafana + Jaeger = ~19 tasks (vs 22 in v3.0 — no gateway/eureka ECS services)
- Average: 0.25 vCPU × 0.5 GB RAM per task
- Fargate: $0.04048/vCPU-hour + $0.004445/GB-hour
- 19 tasks × 0.25 vCPU × 730 hours = ~$140/month CPU
- 19 tasks × 0.5 GB × 730 hours = ~$31/month memory
- **Total: ~$171/month** (down from ~$180/month — 2 fewer ECS services: gateway + eureka)

**Annual savings from choosing ECS over EKS: ~$1,440–3,540/year**

---

## 4. Service Merge Cost Savings (ADR-003)

Merging 11 services to 8 reduces infrastructure by 3 units across every layer.

| Resource | Before (11 services) | After (8 services) | Saving |
|---|---|---|---|
| RDS instances | 11 × $40/month = $440 | 8 × $40/month = $320 | **$120/month** |
| ECS task definitions | 11 | 8 | 3 fewer pipelines |
| ECR repositories | 11 | 8 | Minor |
| CI/CD pipeline minutes | 11 pipelines | 8 pipelines | ~27% reduction |

**Annual savings from service merge: ~$1,440/year on RDS alone**

---

## 5. Total Technology Decision Savings (Updated)

| Decision | Annual Saving |
|---|---|
| SNS+SQS over Kafka | ~$5,556 ($463 → $0.40/month) |
| SNS+SQS over RabbitMQ | ~$2,748 ($230 → $0.40/month) |
| AWS API Gateway over ALB + Spring Cloud Gateway ECS | ~$420 (removes ~$35/month ALB + $15/month ECS tasks) |
| AWS Cloud Map over Eureka ECS | ~$120 (removes $10/month ECS task) |
| ECS over EKS | ~$1,440–3,540 |
| Service merge (11→8) | ~$1,440 |
| **Total annual saving vs original v3.0 estimate** | **~$8,124–11,244/year** |

---

## 6. Cost Optimisation Strategies

### 6.1 Right-Size RDS Instances
Start all services on `db.t3.small` ($40/month, Multi-AZ). After 3 months of production monitoring:
- Services with < 10% CPU and < 50% memory → downgrade to `db.t3.micro` ($20/month)
- Services with > 70% sustained CPU → upgrade to `db.t3.medium`

Target: reduce `db.t3.small` → `db.t3.micro` for at least 4 low-traffic services (inventory, workplace, notification, audit) → saves ~$80/month

### 6.2 ECS Fargate Spot for Non-Critical Services
`notification-service` and `audit-service` are SQS consumers with no synchronous user-facing latency requirement. Both can tolerate occasional task interruptions.

- Fargate Spot pricing: ~70% discount vs on-demand
- Apply to: `notification-service`, `audit-service`, `workplace-service`, `inventory-service`
- Estimated saving: ~$25–40/month

### 6.3 Scheduled Scaling Down for Off-Hours
Scale all services to minimum 1 task during off-hours (22:00–06:00 UTC) in non-production environments.

```
Production: min 2 tasks always (availability SLA)
Staging: min 1 task 22:00–08:00 (dev team offline)
Dev: scale to 0 tasks on weekends
```

Estimated saving on staging/dev: ~$30–50/month

### 6.4 SNS + SQS Messaging Cost
SQS is already near-zero cost (~$0.40/month at OMS traffic volumes) — no further optimisation needed for messaging. The combination of SNS fan-out with SQS queues and DLQs is the lowest-cost HA messaging solution available in AWS at this scale.

### 6.5 S3 Lifecycle Policies for Audit Archival
- Active audit records: RDS (`audit_db`) — 24 months
- After 24 months: S3 Glacier Instant Retrieval (~$0.004/GB-month)
- After 36 months: S3 Glacier Deep Archive (~$0.00099/GB-month)

At ~500 audit events/day × 500 bytes each:
- 2 years of data: ~180 MB active (negligible RDS cost)
- Archival: growing but minimal S3 cost for years

### 6.6 Nightly Batch over Real-Time Streaming
The attendance badge sync runs nightly rather than in real-time. This avoids:
- Kafka Streams or Spark Streaming infrastructure
- Continuous Athena query costs (Athena charges per TB scanned)
- Saving: ~$50–200/month vs a real-time streaming pipeline

---

## 7. Development Cost Considerations

| Factor | Impact |
|---|---|
| Polyglot (Java Spring Boot + Python FastAPI) — adds modest ramp-up cost offset by operational savings | Team learns two languages/frameworks; offset by ~$2,748/year messaging saving and simpler operations |
| 8 services (vs 11) | 3 fewer services to build, test, document, and operate — significant developer time saving |
| TestContainers | Real database and broker in CI — reduces production bugs from mock divergence |
| Pact contract testing | Catches breaking API changes in CI before deployment — avoids costly production incidents |
| ECS over EKS | No Kubernetes expertise required — reduces hiring bar and training cost |

---

## 8. Three-Year Total Cost of Ownership (Indicative)

| Cost Category | Year 1 | Year 2 | Year 3 |
|---|---|---|---|
| AWS Infrastructure | $4,920–7,800 | $4,920–7,800 | $4,920–7,800 |
| After optimisation (est. 20% reduction) | $3,936–6,240 | $3,936–6,240 | $3,936–6,240 |
| Development (team salaries) | Dominant cost | Dominant cost | Dominant cost |
| Infrastructure as % of total | < 5% typical for team of 5–8 | < 5% | < 5% |

Infrastructure cost is not the dominant concern for OMS — development velocity and operational simplicity are. Decisions were made to reduce operational complexity (ECS over EKS, SNS+SQS over RabbitMQ/Kafka) even where the infrastructure cost difference is small, because the saved developer-hours compound over the lifetime of the system.

---

## 9. Cost Monitoring

| Metric | Tool | Alert Threshold |
|---|---|---|
| Monthly AWS spend | AWS Cost Explorer + Budget Alert | > $800/month |
| RDS storage growth | CloudWatch `FreeStorageSpace` | < 5 GB remaining |
| ECS task count (scaling events) | CloudWatch ECS metrics | > 8 tasks on any service |
| SQS DLQ depth | CloudWatch `ApproximateNumberOfMessagesVisible` | > 0 on any DLQ → alert |
| Data transfer costs | AWS Cost Explorer | > $50/month |
