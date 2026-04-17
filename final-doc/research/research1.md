# Amazon SQS vs. Apache Kafka (AWS MSK) — Research Report

> **Context:** Office Management System (OMS), 11-service microservices architecture, AWS-hosted, ~hundreds of employees at 2–5 locations.

---

## 1. What They Are

### Amazon SQS (Simple Queue Service)
A **fully managed message queue** service by AWS. It decouples producers from consumers using a queue model. One message is delivered to **one consumer**, then deleted. Think of it as a post box — a letter goes in, one person picks it up, it's gone.

Two modes:
- **Standard Queue** — at-least-once delivery, best-effort ordering
- **FIFO Queue** — exactly-once processing, strict ordering (but lower throughput)

### Apache Kafka / AWS MSK (Managed Streaming for Apache Kafka)
A **distributed event streaming platform**. It stores messages in a **durable, ordered, replayable log** organized into topics and partitions. Multiple consumers independently read from the same log. Think of it as a newspaper — everyone gets their own copy; back-issues are kept for a configurable retention window.

AWS MSK is Apache Kafka hosted and managed by AWS (no self-managed brokers, ZooKeeper, etc.).

---

## 2. Core Architectural Differences

| Dimension | Amazon SQS | Kafka / AWS MSK |
|---|---|---|
| **Model** | Queue (point-to-point or pub/sub via SNS fan-out) | Distributed log (native pub/sub, multi-consumer) |
| **Message delivery** | One consumer per message, then deleted | All consumer groups independently receive all messages |
| **Message retention** | 1 minute – 14 days (default 4 days) | Hours to forever (configurable; S3 tiered storage available) |
| **Replay** | Not possible — consumed messages are gone | Full replay from any offset, at any time |
| **Ordering** | FIFO only with FIFO queues (limited to 300 TPS per message group) | Strict order within each partition, at scale |
| **Throughput** | Standard: near-unlimited; FIFO: 300 msg/sec per group | Millions of messages/sec per cluster |
| **Consumer model** | Push (long-polling) | Pull (consumer controls pace) |
| **Protocol** | AWS-proprietary (HTTP/HTTPS SDK) | Open standard (Apache Kafka protocol) |

---

## 3. Effectiveness

### SQS Effectiveness
- Excellent for **simple task queues** — e.g., trigger a Lambda, process a background job, send a single notification
- Dead-letter queues (DLQ) make failure handling straightforward
- Scales automatically — zero infrastructure to manage
- Ideal when **exactly one consumer** needs to handle a message
- Weak at fan-out natively — you need SNS + SQS (SNS fan-out pattern) to send one event to multiple queues

### Kafka / MSK Effectiveness
- Designed for **many consumers reading the same event** — exactly what a microservices system needs
- Native **fan-out**: publish once to `oms.remote.request.approved`, three services (`attendance-service`, `notification-service`, `audit-service`) independently consume it
- **Replay** enables: re-processing after a bug fix, replaying audit events, bootstrapping a new service from history
- **Ordering** is guaranteed per partition — critical for `user.deactivated` cascading to multiple services in order
- **Log compaction** allows maintaining latest state per key (e.g., latest user profile snapshot)
- Consumer groups allow services to track their own offset independently — one service falling behind doesn't affect others

---

## 4. Cost Comparison

### Amazon SQS
- Billed per **API request** (every `SendMessage`, `ReceiveMessage`, `DeleteMessage` = 1 request)
- Free tier: **1 million requests/month** free
- Standard: **~$0.40 per million requests** after free tier
- **No infrastructure cost** — fully serverless, zero idle cost
- Very cheap at low/moderate volume; cost grows linearly with message throughput

### AWS MSK (Managed Kafka)
- Billed per **broker instance-hour** + storage + data transfer
- Minimum viable cluster: 3 brokers (for multi-AZ, replication factor ≥ 3)
- Broker costs: **~$0.21–$1.10/hour per broker** depending on instance type (kafka.m5.large ≈ $0.21/hr)
- A 3-broker `kafka.m5.large` cluster = **~$450–$500/month** baseline, even with zero messages
- Storage: $0.10/GB-month; data transfer within VPC is mostly free
- **MSK Serverless** option exists: pay-per-throughput (~$0.10/GB) — bridges the gap for low-traffic orgs

**Cost verdict for OMS:**
- At OMS scale (hundreds of employees, ~15 topics), SQS would be significantly cheaper in raw message cost
- MSK carries a meaningful baseline cost (~$500/month minimum for production HA)
- **MSK Serverless** reduces this to near-zero at low volume — worth evaluating for Phase 1

---

## 5. Enterprise and Organizational Benefits

### SQS — Enterprise Benefits
| Benefit | Detail |
|---|---|
| Zero ops overhead | No cluster to manage, patch, resize, or monitor at broker level |
| AWS-native integration | Direct triggers for Lambda, ECS tasks, Step Functions |
| IAM integration | Fine-grained per-queue IAM policies, no separate auth system |
| Compliance | SOC1/2/3, PCI DSS, HIPAA-eligible |
| Simplicity | Junior developers can use it with minimal training |

### Kafka / MSK — Enterprise Benefits
| Benefit | Detail |
|---|---|
| Event sourcing foundation | The Kafka log *is* an audit trail — replay any event from day one |
| Regulatory audit capability | Immutable ordered log satisfies audit requirements with no extra tooling |
| Decoupled teams | Each service team owns its consumer group; zero coordination needed for new consumers |
| Schema evolution | With Schema Registry (Glue or Confluent), schema contracts are enforced across teams |
| Ecosystem | Kafka Connect (DB sync), Kafka Streams (real-time processing), ksqlDB (stream queries) |
| Vendor portability | Open standard — can migrate to self-managed Kafka or Confluent Cloud without rewriting producers/consumers |
| Cross-region replication | MirrorMaker 2 enables multi-region event replication |
| Operational visibility | Consumer lag metrics are first-class — you always know how far behind each service is |

---

## 6. Operational Complexity

| Factor | SQS | Kafka / MSK |
|---|---|---|
| **Setup time** | Minutes (create queue via console/CLI) | Hours–days (cluster sizing, security groups, IAM, MSK config) |
| **Maintenance** | Zero — fully managed | MSK handles brokers; you manage topics, partitions, retention, consumer groups |
| **Monitoring** | CloudWatch queue depth, age of oldest message | Consumer lag per group/partition — more granular but more to monitor |
| **Schema contracts** | None built-in | Schema Registry optional but strongly recommended at scale |
| **Failure modes** | Message goes to DLQ after N retries | Consumer lag grows; poison-pill messages require manual intervention |
| **Expertise needed** | Minimal | Moderate — partition strategy, rebalancing, offset management |

---

## 7. When to Use Each — Decision Framework

### Choose SQS when:
- You need a **simple task queue** (one producer, one consumer)
- Workload is **serverless or Lambda-based**
- You need **zero idle cost** (pay only when messages flow)
- Fan-out is not needed, or is handled by SNS upstream
- Team has limited messaging expertise
- The system is small, early-stage, or a standalone microservice

### Choose Kafka / MSK when:
- **Multiple independent consumers** need to receive the same event (fan-out by design)
- You need **event replay** — re-processing, bootstrapping new services, disaster recovery
- You need an **immutable audit log** that is inherently part of the messaging layer
- **Ordering within a domain** matters (e.g., all events for `user_id=X` must be processed in sequence)
- You are building a **long-lived event-driven microservices platform** with growing service count
- You need **consumer independence** — each service tracks its own offset, falls behind without affecting others
- **Throughput is high** or expected to grow significantly

---

## 8. Why OMS Chose Kafka (MSK) Over SQS

### Reason 1 — Fan-out is a first-class requirement
The OMS architecture has events that go to 3+ consumers simultaneously:
```
oms.remote.request.approved → attendance-service
                             → notification-service
                             → audit-service
```
With SQS, you would need SNS + 3 separate SQS queues + duplicate message processing logic. With Kafka, it is one topic, three consumer groups. This pattern repeats across all 15 topics.

### Reason 2 — Audit replay is an architectural guarantee
The audit-service requires an **immutable, append-only log**. With Kafka's configurable retention and replay, if the audit-service has a bug and misses events for 2 hours, it can rewind its offset and reprocess — **without any data loss**. SQS deletes messages after consumption; there is no replay.

### Reason 3 — Consumer independence
If `notification-service` is redeployed or crashes, it does not lose its position in the queue — its consumer group offset is stored in Kafka. When it restarts, it resumes exactly where it left off. With SQS, in-flight messages are held briefly and returned to the queue, but there is no durable per-consumer offset concept.

### Reason 4 — Architecture decision on record
The architecture documents explicitly state:
> *"Kafka over direct REST for state changes — Durable replayable log; loose coupling; audit replay capability"*

### Reason 5 — Future-proofing
The Kafka ecosystem (Kafka Connect for DB sync, Kafka Streams for real-time analytics, Schema Registry) gives OMS a growth path. Real-time badge streaming (currently nightly batch via Athena) is listed as a future consideration — Kafka is already the right infrastructure for it.

---

## 9. Summary Comparison Table

| Dimension | SQS | Kafka / MSK | Winner for OMS |
|---|---|---|---|
| Fan-out (1 event → N consumers) | Needs SNS + multiple queues | Native | **Kafka** |
| Message replay | Not possible | Full replay | **Kafka** |
| Cost at low scale | Near-zero | ~$450–500/month baseline | **SQS** |
| Cost at high scale | Linear per message | Fixed infra + low per-message | **Kafka** |
| Operational simplicity | Very simple | Moderate | **SQS** |
| Consumer independence | Limited | Full per consumer-group | **Kafka** |
| Ordering guarantees | FIFO queue only | Per-partition, at scale | **Kafka** |
| Audit log capability | No | Yes (log is the audit trail) | **Kafka** |
| AWS-native integration | Deep (Lambda, Step Fn) | Good (MSK Connect, IAM) | **SQS** |
| Vendor lock-in | High (AWS-proprietary) | Low (open standard) | **Kafka** |
| Team expertise needed | Low | Medium | **SQS** |
| Ecosystem (streams, connectors) | Limited | Rich | **Kafka** |

---

## 10. Bottom Line

SQS wins on simplicity and cost-at-launch. Kafka/MSK wins on fan-out, replay, consumer independence, and audit capability. For a **multi-service event-driven platform where the audit log and fan-out are architectural requirements** — as in OMS — Kafka is the correct choice, and the ~$500/month infrastructure cost is the price of those guarantees. MSK Serverless is worth evaluating for Phase 1 to reduce the baseline cost while retaining all Kafka capabilities.
