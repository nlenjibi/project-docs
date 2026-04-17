# Cost Analysis and Optimisation
## Office Management System (OMS)

**Version:** 3.0 | Scale: 2–5 locations, hundreds of employees

---

## 1. Monthly AWS Cost Estimate (Production)

| Component | Service | Instance/Config | Est. Monthly |
|---|---|---|---|
| Container orchestration | ECS Fargate | 8 services + gateway + Eureka, avg 2 tasks × 0.25 vCPU × 0.5 GB | **~$80–150** |
| Message broker | Amazon MQ (RabbitMQ) | `mq.m5.large` active/standby multi-AZ | **~$230** |
| Databases | AWS RDS PostgreSQL | 8 × `db.t3.small` Multi-AZ, 20 GB each | **~$300–480** |
| API Gateway (ALB) | AWS ALB | 2 LCUs baseline | **~$20–35** |
| Service discovery | Eureka Server (ECS Fargate) | 2 tasks, 0.25 vCPU, 512 MB | **~$10** |
| Rate limiting cache | ElastiCache Redis | `cache.t3.micro`, single node | **~$15** |
| Audit archival | S3 Glacier Instant Retrieval | ~50 GB/year | **~$5–10** |
| Container registry | AWS ECR | 10 repositories, ~2 GB each | **~$2** |
| Observability | CloudWatch Logs | ~10 GB/month log ingestion | **~$5–15** |
| Secrets | AWS Secrets Manager | 20 secrets × $0.40/secret/month | **~$8** |
| Tracing | Jaeger (ECS Fargate) | 1 task, 0.5 vCPU, 1 GB | **~$15** |
| Metrics | Prometheus + Grafana (ECS) | 2 tasks, 0.5 vCPU each | **~$20** |
| **Total (estimated)** | | | **~$710–970/month** |

*Costs are indicative. Actual costs depend on traffic volume and right-sizing after load testing.*

---

## 2. Broker Cost Comparison (Key Decision)

The message broker was the largest infrastructure decision by cost.

| Broker | AWS Service | Monthly Cost | Notes |
|---|---|---|---|
| Apache Kafka | AWS MSK | **~$463/month** | 3 × `kafka.m5.large` brokers, minimum HA |
| **RabbitMQ** | **Amazon MQ** | **~$230/month** | Active/standby `mq.m5.large` |
| SQS + SNS | Amazon SQS | **~$0.09/month** | At ~2,500 msgs/day; near-zero |

**Why not SQS despite the cost advantage:**
- SQS has no native Spring Cloud Stream binder — breaks the Spring Cloud ecosystem consistency
- Fan-out requires SNS + 3 SQS queues per event type → ~45 queues for 15 event types
- Each SQS consumer is a separate queue to monitor, configure DLQs for, and manage IAM permissions on

**Why not Kafka despite its capabilities:**
- Replay capability (Kafka's primary advantage over RabbitMQ) is not required at OMS scale
- Operational overhead: partition strategy, consumer group offset management, Schema Registry
- 50% more expensive than RabbitMQ for equivalent HA ($463 vs $230/month)

**Annual savings from choosing RabbitMQ over Kafka: ~$2,796/year**

---

## 3. Orchestration Cost Comparison

| Orchestration | Monthly Fixed Cost | Notes |
|---|---|---|
| AWS EKS | **~$75** (control plane) + EC2 nodes | Plus ~$200–400 for 2 t3.medium nodes |
| **AWS ECS Fargate** | **$0** control plane | Pay per task CPU/memory second |

**ECS Fargate container cost estimate:**
- 10 services × 2 tasks each = 20 tasks
- Average: 0.25 vCPU × 0.5 GB RAM per task
- Fargate: $0.04048/vCPU-hour + $0.004445/GB-hour
- 20 tasks × 0.25 vCPU × 730 hours = ~$148/month CPU
- 20 tasks × 0.5 GB × 730 hours = ~$32/month memory
- **Total: ~$180/month** (vs EKS ~$300–475/month)

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

## 5. Total Technology Decision Savings

| Decision | Annual Saving |
|---|---|
| RabbitMQ over Kafka | ~$2,796 |
| ECS over EKS | ~$1,440–3,540 |
| Service merge (11→8) | ~$1,440 |
| **Total annual saving** | **~$5,676–7,776/year** |

---

## 6. Cost Optimisation Strategies

### 6.1 Right-Size RDS Instances
Start all services on `db.t3.small` ($40/month, Multi-AZ). After 3 months of production monitoring:
- Services with < 10% CPU and < 50% memory → downgrade to `db.t3.micro` ($20/month)
- Services with > 70% sustained CPU → upgrade to `db.t3.medium`

Target: reduce `db.t3.small` → `db.t3.micro` for at least 4 low-traffic services (inventory, workplace, notification, audit) → saves ~$80/month

### 6.2 ECS Fargate Spot for Non-Critical Services
`notification-service` and `audit-service` are Kafka consumers with no synchronous user-facing latency requirement. Both can tolerate occasional task interruptions.

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

### 6.4 Amazon MQ Serverless (Future)
Amazon MQ for RabbitMQ Serverless (if available in region) charges per-GB transferred rather than per broker-hour. At OMS traffic (~2,500 msgs/day × 1KB average = ~2.5 MB/day), serverless would cost ~$0.75/month. Worth evaluating once the feature reaches general availability in your region.

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
| Spring Cloud ecosystem (all Java) | Team does not need to learn multiple languages or frameworks — reduced ramp-up cost |
| 8 services (vs 11) | 3 fewer services to build, test, document, and operate — significant developer time saving |
| TestContainers | Real database and broker in CI — reduces production bugs from mock divergence |
| Pact contract testing | Catches breaking API changes in CI before deployment — avoids costly production incidents |
| ECS over EKS | No Kubernetes expertise required — reduces hiring bar and training cost |

---

## 8. Three-Year Total Cost of Ownership (Indicative)

| Cost Category | Year 1 | Year 2 | Year 3 |
|---|---|---|---|
| AWS Infrastructure | $8,520–11,640 | $8,520–11,640 | $8,520–11,640 |
| After optimisation (est. 20% reduction) | $6,816–9,312 | $6,816–9,312 | $6,816–9,312 |
| Development (team salaries) | Dominant cost | Dominant cost | Dominant cost |
| Infrastructure as % of total | < 5% typical for team of 5–8 | < 5% | < 5% |

Infrastructure cost is not the dominant concern for OMS — development velocity and operational simplicity are. Decisions were made to reduce operational complexity (ECS over EKS, RabbitMQ over Kafka) even where the infrastructure cost difference is small, because the saved developer-hours compound over the lifetime of the system.

---

## 9. Cost Monitoring

| Metric | Tool | Alert Threshold |
|---|---|---|
| Monthly AWS spend | AWS Cost Explorer + Budget Alert | > $1,200/month |
| RDS storage growth | CloudWatch `FreeStorageSpace` | < 5 GB remaining |
| ECS task count (scaling events) | CloudWatch ECS metrics | > 8 tasks on any service |
| Amazon MQ queue depth | CloudWatch MQ metrics | > 10,000 messages any queue |
| Data transfer costs | AWS Cost Explorer | > $50/month |
