# RabbitMQ vs Kafka vs SQS — Cost, Effectiveness & Implementation

> **Context:** OMS — 8 services, AWS-hosted, ECS Fargate, Spring Boot, hundreds of employees, 2–5 locations.  
> Purpose: Side-by-side comparison to justify the final broker choice.

---

## 1. Cost Comparison

### 1.1 AWS Managed Options (Apples-to-Apples)

| Broker | AWS Service | Pricing Model |
|---|---|---|
| Kafka | AWS MSK | Per broker-hour + storage + data transfer |
| RabbitMQ | Amazon MQ for RabbitMQ | Per broker-hour + storage |
| SQS | Amazon SQS | Per API request (send + receive + delete) |

---

### 1.2 Kafka — AWS MSK Cost

**Minimum production cluster:** 3 brokers (multi-AZ, replication factor ≥ 3)

| Item | Unit Cost | OMS Estimate | Monthly |
|---|---|---|---|
| Brokers | `kafka.m5.large` = $0.21/hr × 3 | 3 brokers × 24hr × 30 days | **~$453** |
| Storage | $0.10/GB-month | 100GB across cluster | **~$10** |
| Data transfer (within VPC) | Free | — | $0 |
| **Total** | | | **~$463/month** |

> MSK Serverless: ~$0.10/GB + $0.016/partition-hour — cheaper at very low volume, but at moderate traffic it approaches or exceeds the cluster cost. Hard to predict.

**Kafka minimum: ~$450–500/month regardless of whether you send 1 message or 1 million.**

---

### 1.3 RabbitMQ — Amazon MQ Cost

**Production setup:** Active/standby broker pair (multi-AZ)

| Item | Unit Cost | OMS Estimate | Monthly |
|---|---|---|---|
| Primary broker | `mq.m5.large` = $0.317/hr | 24hr × 30 days | **~$228** |
| Standby broker | Included in active/standby pricing | — | $0 (bundled) |
| Storage | $0.10/GB-month | 20GB | **~$2** |
| **Total** | | | **~$230/month** |

> Single-instance (dev/staging): `mq.m5.large` = ~$115/month

**RabbitMQ minimum: ~$230/month for HA production.**

---

### 1.4 SQS — Amazon SQS Cost

**SQS has no baseline cost.** You pay only per API call.

| Operation | Cost |
|---|---|
| First 1 million requests/month | Free |
| Standard queue requests | $0.40 per million |
| FIFO queue requests | $0.50 per million |
| Data transfer (within AWS) | Free |

**Estimating OMS SQS cost:**

OMS generates roughly:
- ~500 employees × average 5 events/day = 2,500 messages/day
- Each message = 1 send + 1 receive + 1 delete = **3 API calls**
- 2,500 × 3 × 30 days = **225,000 calls/month**

At $0.40 per million: **225,000 / 1,000,000 × $0.40 = ~$0.09/month**

Even at 10× traffic: **~$0.90/month**

**SQS for OMS scale: near-zero cost. Well under $5/month.**

---

### 1.5 Cost Summary Table

| Broker | Monthly Cost (OMS) | Cost Model | Scales With |
|---|---|---|---|
| **Kafka (MSK)** | **~$463/month** | Fixed infrastructure | Brokers (not messages) |
| **RabbitMQ (Amazon MQ)** | **~$230/month** | Fixed infrastructure | Broker instances |
| **SQS** | **~$0.09–5/month** | Pay per request | Message volume |

**Cost winner: SQS by a massive margin at OMS scale.**  
**Best infrastructure value: RabbitMQ (~50% cheaper than Kafka for equivalent HA).**

---

## 2. Effectiveness Comparison

### 2.1 Fan-out (One Event → Multiple Consumers)

This is the most critical requirement for OMS. `remote.request.approved` must reach `attendance-service`, `notification-service`, and `audit-service` independently.

| Broker | Fan-out Support | How |
|---|---|---|
| **Kafka** | Native | Multiple consumer groups read the same topic independently |
| **RabbitMQ** | Native | Topic/Fanout exchange → one queue per consumer service |
| **SQS** | Not native | Requires SNS + one SQS queue per consumer (SNS fan-out pattern) |

**Fan-out winner: Kafka and RabbitMQ tied. SQS requires SNS workaround.**

SQS fan-out architecture:
```
remote-service publishes → SNS Topic: remote-request-approved
    → SQS Queue: attendance-remote-approved   ← attendance-service
    → SQS Queue: notification-remote-approved ← notification-service
    → SQS Queue: audit-remote-approved        ← audit-service
```
This works, but every event needs a separate SNS topic + 3 SQS queues. For 15+ event types that is 15 SNS topics + 30–45 SQS queues to manage.

---

### 2.2 Message Replay

| Broker | Replay Possible | Detail |
|---|---|---|
| **Kafka** | Yes — full | Rewind consumer offset to any point; replay all events from day one |
| **RabbitMQ** | No | Messages consumed and removed; durable queues hold unprocessed messages only |
| **SQS** | No | Consumed messages deleted; DLQ holds failed messages only |

**Replay winner: Kafka only. RabbitMQ and SQS cannot replay consumed messages.**

---

### 2.3 Message Ordering

| Broker | Ordering Guarantee |
|---|---|
| **Kafka** | Strict order per partition |
| **RabbitMQ** | FIFO per queue (single consumer); breaks with multiple consumers on same queue |
| **SQS Standard** | Best-effort (no guarantee) |
| **SQS FIFO** | Strict order per message group (but limited to 300 TPS per group) |

**Ordering winner: Kafka (strict, at scale). RabbitMQ is adequate for OMS. SQS FIFO is limited.**

---

### 2.4 Throughput

| Broker | Throughput |
|---|---|
| **Kafka** | Millions of messages/second per cluster |
| **RabbitMQ** | ~50,000–100,000 messages/second per node |
| **SQS Standard** | Unlimited (nearly) |
| **SQS FIFO** | 300 messages/second per message group |

**OMS needs: ~100–500 messages/day. All three are vastly over-spec for OMS traffic.**

---

### 2.5 Message Retention / Durability

| Broker | How Long Messages Are Kept |
|---|---|
| **Kafka** | Configurable — hours to forever; log compaction available |
| **RabbitMQ** | Until consumed (durable queues survive restart); TTL configurable |
| **SQS** | 1 minute to 14 days (default 4 days); consumed messages gone |

**Durability winner: Kafka. RabbitMQ is sufficient. SQS has a 14-day hard cap.**

---

### 2.6 Consumer Independence

| Broker | Each Consumer Tracks Its Own Position? |
|---|---|
| **Kafka** | Yes — consumer group offset stored in Kafka; each service is fully independent |
| **RabbitMQ** | Yes — each service has its own durable queue; services are fully independent |
| **SQS** | Yes (with SNS fan-out) — each service has its own queue |

**All three are equivalent here when properly configured.**

---

### 2.7 Dead Letter Queue (Failed Messages)

| Broker | DLQ Support |
|---|---|
| **Kafka** | Manual — dead-letter topic, configured in consumer code |
| **RabbitMQ** | Built-in — `x-dead-letter-exchange` per queue; auto-configured with Spring Cloud Stream |
| **SQS** | Built-in — DLQ is a separate SQS queue; configured with `maxReceiveCount` |

**DLQ winner: SQS and RabbitMQ (built-in, simple). Kafka requires manual setup.**

---

### 2.8 Effectiveness Summary

| Feature | Kafka | RabbitMQ | SQS |
|---|---|---|---|
| Fan-out | Native | Native | Needs SNS |
| Replay | Full | No | No |
| Ordering | Strict per partition | FIFO per queue | Best-effort (FIFO: limited) |
| Throughput | Extreme | High | Extreme |
| Retention | Forever | Until consumed | Max 14 days |
| DLQ | Manual | Built-in | Built-in |
| Consumer independence | Full | Full | Full (with SNS) |
| Audit trail capability | Inherent (log is the record) | Needs separate storage | Needs separate storage |

---

## 3. Implementation Comparison

### 3.1 Setup Complexity

| Broker | Initial Setup Effort |
|---|---|
| **Kafka (MSK)** | High — cluster sizing, broker count, topic creation, partition strategy, replication factor, consumer group design, Schema Registry (optional) |
| **RabbitMQ (Amazon MQ)** | Medium — broker provisioning, exchange/queue/binding design, DLQ config, TLS setup |
| **SQS** | Low — create queues via console or CDK; SNS topics for fan-out; zero infrastructure |

---

### 3.2 Spring Boot Integration

#### Kafka
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-stream-kafka</artifactId>
</dependency>
```
```yaml
spring:
  kafka:
    bootstrap-servers: broker1:9092,broker2:9092
  cloud:
    stream:
      bindings:
        remoteRequestApproved-in-0:
          destination: oms.remote.request.approved
          group: attendance-service
```
Spring Boot Kafka integration is mature, well-documented, and widely used.

#### RabbitMQ
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-stream-rabbit</artifactId>
</dependency>
```
```yaml
spring:
  rabbitmq:
    host: ${RABBITMQ_HOST}
    port: 5671
    ssl.enabled: true
  cloud:
    stream:
      bindings:
        remoteRequestApproved-in-0:
          destination: oms.remote
          group: attendance-service
      rabbit:
        bindings:
          remoteRequestApproved-in-0:
            consumer:
              exchange-type: topic
              binding-routing-key: request.approved
              durableSubscription: true
              auto-bind-dlq: true
```
More YAML configuration than Kafka. Exchange/queue/binding design requires upfront planning. DLQ wiring is more explicit.

#### SQS
```xml
<dependency>
    <groupId>io.awspring.cloud</groupId>
    <artifactId>spring-cloud-aws-starter-sqs</artifactId>
</dependency>
```
```java
// Producer
@Autowired
private SqsTemplate sqsTemplate;

public void publish(RemoteApprovedEvent event) {
    sqsTemplate.send("attendance-remote-approved-queue", event);
}

// Consumer
@SqsListener("attendance-remote-approved-queue")
public void handleRemoteApproved(RemoteApprovedEvent event) {
    attendanceService.overlayRemoteStatus(event);
}
```
Simplest code. But for fan-out you also wire SNS:
```java
snsTemplate.sendNotification("oms-remote-request-approved-topic", event, null);
// SNS then fans out to 3 SQS queues automatically
```
SNS + SQS adds queue/topic management overhead at the infrastructure level.

---

### 3.3 Operations and Monitoring

| Concern | Kafka | RabbitMQ | SQS |
|---|---|---|---|
| **Key metric to watch** | Consumer group lag | Queue depth | Approximate number of messages |
| **CloudWatch support** | MSK publishes broker metrics | Amazon MQ publishes queue depth + connection metrics | Native SQS metrics |
| **Visibility into lag** | Per consumer group, per partition — very granular | Per queue — simple and clear | Per queue — simple and clear |
| **Alerting threshold** | Consumer lag > 10,000 | Queue depth > 10,000 | Approximate messages > 10,000 |
| **Broker management** | AWS manages MSK brokers; you manage topic config | AWS manages Amazon MQ; you manage exchange/queue config | Zero — fully serverless |
| **Upgrade/patching** | AWS-managed on MSK | AWS-managed on Amazon MQ | Not applicable |

---

### 3.4 Implementation Effort (Time Estimate for OMS)

| Task | Kafka | RabbitMQ | SQS |
|---|---|---|---|
| Infrastructure provisioning | 4–8 hours | 2–4 hours | 30 minutes |
| Topic/exchange/queue design | 2–4 hours | 3–5 hours | 2–3 hours (SNS + SQS) |
| Spring Boot producer setup per service | 1 hour/service | 1.5 hours/service | 1 hour/service |
| Spring Boot consumer setup per service | 1 hour/service | 1.5 hours/service | 1 hour/service |
| DLQ configuration | 2–3 hours | 1–2 hours (auto-bind-dlq) | 1 hour |
| Testing (integration with TestContainers) | TestContainers Kafka — well supported | TestContainers RabbitMQ — well supported | LocalStack SQS — well supported |
| **Estimated total for 8 services** | **~30–40 hours** | **~35–45 hours** | **~20–25 hours** |

---

### 3.5 TestContainers Support (Local Development + CI)

All three work with TestContainers for local integration testing:

```java
// Kafka
@Container
static KafkaContainer kafka = new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:7.4.0"));

// RabbitMQ
@Container
static RabbitMQContainer rabbit = new RabbitMQContainer(DockerImageName.parse("rabbitmq:3.12-management"));

// SQS (via LocalStack)
@Container
static LocalStackContainer localstack = new LocalStackContainer(DockerImageName.parse("localstack/localstack"))
    .withServices(LocalStackContainer.Service.SQS, LocalStackContainer.Service.SNS);
```

**All three are equal here.**

---

## 4. Three-Way Decision Matrix (OMS Specific)

| Dimension | Kafka | RabbitMQ | SQS | Weight |
|---|---|---|---|---|
| **Monthly cost at OMS scale** | $463 | $230 | ~$1 | High |
| **Fan-out (native)** | Yes | Yes | Needs SNS | High |
| **Spring Cloud Stream support** | Yes (Kafka binder) | Yes (RabbitMQ binder) | No native binder | High |
| **Eureka + Spring Cloud fit** | Works but independent | Works but independent | Works but independent | Medium |
| **Implementation complexity** | High | Medium | Low | Medium |
| **Replay capability** | Full | No | No | Medium (audit concern) |
| **Operations overhead** | Medium | Low | Zero | Medium |
| **DLQ (built-in)** | Manual | Built-in | Built-in | Medium |
| **Team learning curve** | Medium | Low (already decided) | Low | High |
| **Message ordering** | Strict | Per-queue FIFO | Best-effort | Low (OMS doesn't need strict global order) |

### Scoring (1 = worst, 3 = best for OMS)

| Dimension | Kafka | RabbitMQ | SQS |
|---|---|---|---|
| Cost | 1 | 2 | 3 |
| Fan-out | 3 | 3 | 1 |
| Spring Cloud Stream | 3 | 3 | 1 |
| Implementation complexity | 1 | 2 | 3 |
| Replay | 3 | 1 | 1 |
| Operations | 2 | 2 | 3 |
| DLQ | 1 | 3 | 3 |
| Team fit | 2 | 3 | 2 |
| **Total** | **16** | **19** | **17** |

---

## 5. Final Recommendation

### RabbitMQ (Amazon MQ) is the right choice for OMS

Here is why it beats both alternatives for your specific context:

**vs Kafka:**
- ~50% cheaper ($230 vs $463/month)
- Simpler to operate — no partition strategy, no offset management, no Schema Registry
- DLQ is built-in and auto-configured by Spring Cloud Stream
- The only thing Kafka wins on is replay — which matters for audit. Mitigation: run `audit-service` with durable queues + 2 HA tasks so no messages are lost
- At OMS scale (hundreds of employees), Kafka's extreme throughput and replay are over-engineering

**vs SQS:**
- Native fan-out via Topic Exchange — no SNS + 45 SQS queues to manage
- Spring Cloud Stream has a native RabbitMQ binder — clean integration with the rest of the Spring Cloud stack (Eureka, Gateway, OpenFeign all in one ecosystem)
- SQS has no native Spring Cloud Stream binder — you would step outside the Spring Cloud ecosystem you have already committed to
- SQS fan-out with SNS adds infrastructure complexity that eliminates its simplicity advantage

**The Spring Cloud commitment is the deciding factor between RabbitMQ and SQS.**  
You are building on: Spring Cloud Gateway + Eureka + OpenFeign + Resilience4j.  
Spring Cloud Stream's RabbitMQ binder completes the stack cleanly.  
SQS does not fit this ecosystem.

---

## 6. One-Line Summary

| Broker | Best For | Not For |
|---|---|---|
| **Kafka** | High-throughput event streaming, audit replay, large-scale microservices | Small teams, cost-sensitive projects, simple fan-out |
| **RabbitMQ** | Spring Cloud microservices, reliable fan-out, moderate traffic, cost-aware teams | Systems requiring replay, extreme throughput |
| **SQS** | Serverless/Lambda architectures, simple task queues, zero-ops, minimal cost | Fan-out without SNS, Spring Cloud Stream integration |
