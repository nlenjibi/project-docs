# OMS Infrastructure — Cost Estimate

> **Note:** All prices are approximate AWS us-east-1 pricing as of April 2026. Actual costs depend on usage, data transfer volume, and any reserved instance or savings plan commitments. Production costs are TBD — this document covers **dev + staging** only. This cost is AI estimated 

---

## Monthly Cost Table — v2 (Shared Database)

### Per-Environment Breakdown

| Resource | Instance / Tier | Cost/month (dev) | Cost/month (staging) | Notes |
|----------|----------------|-----------------|---------------------|-------|
| **ECS Fargate** | 7 services × 0.25 vCPU × 512 MB | ~$18 | ~$25 | Dev: 1 task/service; Staging: 1–2 tasks/service |
| **RDS PostgreSQL (shared)** | db.t3.micro, single-AZ, 20 GB gp2 | ~$15 | ~$15 | One instance, 7 schemas |
| **ElastiCache Redis** | cache.t3.micro | ~$13 | ~$13 | Shared, key prefix per service |
| **NAT Gateway** | 1 × hourly + data transfer | ~$35 | ~$35 | ~$0.045/hr + $0.045/GB processed |
| **API Gateway (HTTP API)** | Per-request pricing | ~$1 | ~$2 | ~$1/million requests |
| **Amazon SQS** | Pay per request | ~$0.50 | ~$0.50 | ~$0.40/million requests; DLQ included |
| **Amazon SES** | Per email | ~$0.10 | ~$0.50 | ~$0.10/1,000 emails |
| **Amazon ECR** | Storage + data transfer | ~$1 | ~$1 | ~$0.10/GB/month storage |
| **S3 (frontend assets)** | Standard storage + GET requests | ~$0.50 | ~$0.50 | React build ~50 MB |
| **CloudFront** | Data transfer out | ~$1 | ~$2 | First 1 TB/month ~$0.085/GB |
| **Route 53** | Hosted zone + queries | ~$0.50 | ~$0.50 | $0.50/zone/month + queries |
| **AWS CodeDeploy** | ECS deployments | $0 | $0 | Free for ECS |
| **AWS Cloud Map** | Namespace + health checks | ~$0.50 | ~$0.50 | $0.50/namespace/month |
| **Data warehouse RDS** | db.t3.micro (shared dev/staging) | ~$8 | ~$8 | `data_warehouse_db` — 1 instance |
| **Misc / CloudWatch Logs** | Log ingestion | ~$2 | ~$2 | ECS container logs |
| **Sub-total** | | **~$97** | **~$106** | |

### v2 Total: ~$95–110/month per environment · ~$145/month combined (with optimisations)

---

## Monthly Cost Table — v3 (Database per Service)

### Per-Environment Breakdown

| Resource | Instance / Tier | Cost/month (dev) | Cost/month (staging) | Notes |
|----------|----------------|-----------------|---------------------|-------|
| **ECS Fargate** | 7 services × 0.25 vCPU × 512 MB | ~$18 | ~$25 | Dev: 1 task/service; Staging: 1–2 tasks/service |
| **RDS — attendance_db** | db.t3.micro, single-AZ, 20 GB gp2 | ~$13 | ~$13 | Per service |
| **RDS — seating_db** | db.t3.micro, single-AZ, 20 GB gp2 | ~$13 | ~$13 | Per service |
| **RDS — remote_ooo_db** | db.t3.micro, single-AZ, 20 GB gp2 | ~$13 | ~$13 | Per service |
| **RDS — visitor_db** | db.t3.micro, single-AZ, 20 GB gp2 | ~$13 | ~$13 | Per service |
| **RDS — events_db** | db.t3.micro, single-AZ, 20 GB gp2 | ~$13 | ~$13 | Per service |
| **RDS — supplies_assets_db** | db.t3.micro, single-AZ, 20 GB gp2 | ~$13 | ~$13 | Per service |
| **RDS — notification_db** | db.t3.micro, single-AZ, 20 GB gp2 | ~$13 | ~$13 | Per service |
| **ElastiCache Redis** | cache.t3.micro | ~$13 | ~$13 | Shared |
| **NAT Gateway** | 1 × hourly + data transfer | ~$35 | ~$35 | Same as v2 |
| **API Gateway (HTTP API)** | Per-request pricing | ~$1 | ~$2 | Same as v2 |
| **Amazon SQS** | Pay per request | ~$0.50 | ~$0.50 | Same as v2 |
| **Amazon SES** | Per email | ~$0.10 | ~$0.50 | Same as v2 |
| **Amazon ECR** | Storage + data transfer | ~$1 | ~$1 | Same as v2 |
| **S3 (frontend assets)** | Standard storage + requests | ~$0.50 | ~$0.50 | Same as v2 |
| **CloudFront** | Data transfer out | ~$1 | ~$2 | Same as v2 |
| **Route 53** | Hosted zone + queries | ~$0.50 | ~$0.50 | Same as v2 |
| **AWS CodeDeploy** | ECS deployments | $0 | $0 | Free for ECS |
| **AWS Cloud Map** | Namespace + health checks | ~$0.50 | ~$0.50 | Same as v2 |
| **Data warehouse RDS** | db.t3.micro | ~$8 | ~$8 | Same as v2 |
| **Misc / CloudWatch Logs** | Log ingestion | ~$2 | ~$2 | Same as v2 |
| **Sub-total** | | **~$153** | **~$167** | |

### v3 Total: ~$150–170/month per environment · ~$200–220/month combined

---

## Budget Headroom Analysis

| Variant | Monthly Estimate | Budget | Headroom | Status |
|---------|-----------------|--------|----------|--------|
| **v2 (Shared DB)** | ~$145/month | $200/month | **~$55/month** | ✅ Comfortable |
| **v3 (DB per Service)** | ~$200–220/month | $200/month | **~$0 to -$20/month** | ⚠️ At/over limit |

> **Warning:** v3 at full capacity (dev + staging, both running 24/7) exceeds or exactly meets the $200 budget. Apply the cost optimisations below to stay within budget.

---

## Cost Optimisation Recommendations

### 1. Fargate Spot for Dev Environment

Fargate Spot instances use spare AWS capacity at up to 70% discount. Dev workloads can tolerate interruption (Fargate Spot tasks are terminated with 2-minute notice when capacity is reclaimed).

**Savings: ~$12–13/month on Fargate in dev.**

```bash
# Example: use FARGATE_SPOT capacity provider in dev ECS service
aws ecs create-service \
  --capacity-provider-strategy capacityProvider=FARGATE_SPOT,weight=1 \
  ...
```

### 2. Schedule Dev Environment Shutdown Outside Office Hours

Stop all ECS tasks and optionally stop RDS instances outside working hours (e.g., weekdays 18:00–07:00, weekends fully off). RDS instances can be stopped for up to 7 days at a time via the console or CLI.

**ECS tasks cost nothing when stopped (Fargate is per-second billing).**
**RDS stopped instances: no compute charge, but storage charge continues.**

**Estimated savings: ~$30–40/month on dev ECS + RDS (assuming 12 hours/day, weekdays only).**

```bash
# Stop all ECS tasks in dev by setting desired count to 0
aws ecs update-service --cluster oms-dev --service attendance-service --desired-count 0

# Stop dev RDS instance
aws rds stop-db-instance --db-instance-identifier oms-dev-attendance-db
```

> **Note:** Automate this with EventBridge Scheduler (formerly CloudWatch Events) — runs on cron schedule, no EC2 required.

### 3. Single NAT Gateway per Environment (Already Applied)

Using one NAT Gateway per VPC (not one per AZ) saves ~$32/month per environment vs multi-AZ. This is already the current design. For staging and dev, single-AZ NAT is acceptable. Prod should use multi-AZ NAT Gateways.

### 4. VPC Endpoints for ECR and S3

ECR image pulls during ECS task startup generate NAT Gateway data processing charges. VPC Endpoints for ECR (`com.amazonaws.<region>.ecr.dkr`) and S3 (`com.amazonaws.<region>.s3`) bypass the NAT Gateway entirely.

**Estimated savings: ~$3–8/month depending on image sizes and deployment frequency.**

```bash
# Create ECR VPC endpoint (interface endpoint)
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-xxxxxx \
  --service-name com.amazonaws.us-east-1.ecr.dkr \
  --vpc-endpoint-type Interface \
  --subnet-ids subnet-xxxxxx \
  --security-group-ids sg-xxxxxx
```

### 5. Optimise Docker Image Sizes

Multi-stage Docker builds reduce image sizes, reducing ECR storage costs and NAT Gateway data transfer on ECR pulls.

```dockerfile
# Example: Spring Boot multi-stage build
FROM eclipse-temurin:17-jdk-alpine AS build
WORKDIR /app
COPY . .
RUN ./mvnw package -DskipTests

FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=build /app/target/service.jar .
ENTRYPOINT ["java", "-jar", "service.jar"]
```

### 6. Reserved Instances for Staging RDS

If staging runs 24/7 (which it should for pre-production stability), purchasing 1-year Reserved Instances for RDS instances saves ~30–40% vs on-demand pricing.

**Staging RDS savings: ~$27–36/month (v2: 1 instance; v3: up to 7 instances).**

---

## Optional Components Cost

These components are marked OPTIONAL in the architecture and are not included in the baseline cost estimates above.

| Component | Pricing Model | Estimated Cost | Notes |
|-----------|--------------|---------------|-------|
| **AWS Glue ETL** | $0.44/DPU-hour | ~$5–20/month | Depends on job frequency and data volume |
| **AWS Athena** | $5.00/TB scanned | ~$1–10/month | Use partitioned S3 data and column projection to reduce scans |
| **S3 Staging Bucket** | $0.023/GB/month + requests | ~$1–3/month | Raw data landing zone for data engineering |
| **CloudWatch Logs (extended)** | $0.50/GB ingested + $0.03/GB/month stored | ~$5–15/month | If extended log retention is configured |

> **Athena cost control:** Always partition S3 data by date (`year=/month=/day=`) and use column-based formats (Parquet or ORC) to minimise bytes scanned per query.

---

## Cost by Environment

| Environment | v2 Monthly | v3 Monthly | Status |
|-------------|-----------|-----------|--------|
| **dev** | ~$70/month | ~$120/month | Active (optimisations recommended) |
| **staging** | ~$75/month | ~$100/month | Active |
| **prod** | TBD | TBD | Not yet provisioned |
| **Total (dev + staging)** | **~$145/month** | **~$200–220/month** | Budget: $200/month |

---

## Recommended Configuration for Staying Within $200/$300 Budget on v3

Apply all of the following:

1. Fargate Spot for all dev ECS tasks (~$12 saved)
2. EventBridge Scheduler to shut down dev ECS tasks and RDS instances nights and weekends (~$35 saved)
3. VPC Endpoints for ECR and S3 in both environments (~$5 saved)
4. 1-year Reserved Instances for staging RDS instances (~$30 saved on v3)

**Combined savings: ~$82/month → brings v3 from ~$220 → ~$138/month.** Well within budget with headroom for traffic growth.
