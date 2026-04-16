# Observability
## Office Management System (OMS)

Every service implements the full observability stack. There are no exceptions — a service without all four pillars cannot go to production.

**Four pillars:**
1. Structured JSON logging → stdout
2. Prometheus metrics → `/actuator/prometheus`
3. Distributed tracing → OpenTelemetry + Jaeger
4. Health checks → `/actuator/health/liveness` and `/actuator/health/readiness`

---

## 1. Structured Logging (JSON to Stdout)

All logs output to **stdout** only — no local log files. A centralised log aggregator (CloudWatch Logs or ELK) consumes stdout.

### Log Format

```json
{
  "timestamp": "2026-03-01T09:15:42.123Z",
  "level": "INFO",
  "service": "seating-service",
  "correlationId": "d4e2f9a1-1234-5678-abcd-ef1234567890",
  "userId": "user-uuid",
  "locationId": "location-uuid",
  "action": "SEAT_BOOKED",
  "seatId": "seat-uuid",
  "message": "Hot-desk booking confirmed for 2026-03-02"
}
```

### Implementation

```java
log.info("Seat booking confirmed",
    kv("service", "seating-service"),
    kv("correlationId", correlationId),
    kv("userId", userId),
    kv("seatId", seatId),
    kv("locationId", locationId),
    kv("action", "SEAT_BOOKED"));
```

Use `logstash-logback-encoder` in every service to produce JSON structured output.

### Rules

- Output to stdout only — no local files.
- Every log entry must include `correlationId` (the request trace ID).
- Never log passwords, raw tokens, or sensitive PII.
- Log at appropriate levels: `DEBUG` for trace details (not enabled in production), `INFO` for normal operations, `WARN` for recoverable issues, `ERROR` for failures.

### Log Levels by Scenario

| Scenario | Level |
|----------|-------|
| Request received | DEBUG |
| Business operation completed | INFO |
| Fallback triggered (circuit breaker) | WARN |
| Duplicate Kafka event ignored | INFO |
| Scheduled job started / completed | INFO |
| Scheduled job failed | ERROR |
| Unresolved personnel ID (badge sync) | WARN |
| 403 Forbidden access denial | WARN |
| Unhandled exception | ERROR |

---

## 2. Prometheus Metrics

Every service exposes `/actuator/prometheus`. Prometheus scrapes all services every 15 seconds. Grafana dashboards visualise the metrics.

### Required Metrics (All Services)

```
http_request_duration_seconds{service, endpoint, method, status_code}
  → API latency by endpoint — track p50, p95, p99

http_requests_total{service, endpoint, method, status_code}
  → Request count; used to derive error rates

kafka_consumer_lag{service, topic, consumer_group}
  → Messages behind on each topic partition

circuit_breaker_state{service, name}
  → 0 = CLOSED (healthy), 1 = OPEN (failing), 2 = HALF_OPEN

db_connection_pool_active{service}
  → Active DB connections vs pool size
```

### Scheduled Job Metrics (attendance-service, seating-service, user-service)

```
job_duration_seconds{service, job_name}
  → How long the job took to run

job_failures_total{service, job_name}
  → Incremented on every job failure

job_records_processed_total{service, job_name}
  → Badge events inserted, no-show bookings released, etc.
```

### Spring Boot Actuator Configuration

```yaml
# application.yml — every service
management:
  endpoints:
    web:
      exposure:
        include: health, prometheus, info
  endpoint:
    health:
      probes:
        enabled: true
  metrics:
    export:
      prometheus:
        enabled: true
```

---

## 3. Distributed Tracing (OpenTelemetry)

Every request carries a `correlationId` (UUID v4). The API Gateway generates it for every inbound request and injects it as the `X-Correlation-ID` HTTP header. Every service:

- Reads `X-Correlation-ID` from inbound HTTP headers
- Propagates it on every outbound HTTP call
- Includes it in every Kafka event envelope
- Includes it in every log entry

```
Browser → API Gateway → seating-service → user-service
           [corr: abc]    [corr: abc]       [corr: abc]
                               ↓ Kafka event
                          audit-service [corr: abc]
```

A single `correlationId` traces the complete path of a request across all services and async event chains.

### Kafka Header Propagation

```java
ProducerRecord<String, OmsEvent<?>> record = new ProducerRecord<>(topic, event);
record.headers().add("X-Correlation-ID", correlationId.toString().getBytes(StandardCharsets.UTF_8));
```

### OpenTelemetry Setup

The OpenTelemetry SDK is included in every service via the Spring Boot auto-configuration. Traces are exported to Jaeger (dev/staging) or AWS X-Ray (production).

```yaml
# application.yml
spring:
  application:
    name: seating-service   # Appears in Jaeger service list

otel:
  exporter:
    otlp:
      endpoint: ${OTEL_EXPORTER_ENDPOINT}
  traces:
    sampler: always_on      # 100% sampling in dev; tune in prod
```

---

## 4. Health Checks

Every service exposes two health endpoints used by Kubernetes probes:

```
GET /actuator/health/liveness   → Is the process alive? (JVM running, app not deadlocked)
GET /actuator/health/readiness  → Is the service ready to serve traffic? (DB connected, Kafka reachable)
```

### Kubernetes Probe Configuration

```yaml
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3

readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 5
  failureThreshold: 3
```

A service failing its readiness probe is removed from the load balancer pool until it recovers — it receives no traffic.

---

## Grafana Dashboards

| Dashboard | Key Metrics |
|-----------|------------|
| **System Health** | Error rate (5xx/total) per service; p95 API latency per service |
| **Kafka Health** | Consumer group lag per topic per service; producer throughput |
| **Circuit Breakers** | Open / half-open state per service per downstream |
| **Scheduled Jobs** | Badge sync completion time; no-show release completion; HR sync status |
| **DB Health** | Connection pool utilisation per service; slow query count |
| **Deployment** | Pod restart count; rollout status; replica count per service |

---

## Alerting Rules

All alerts route to PagerDuty (critical) or Slack (warnings).

| Alert | Threshold | Severity | Route |
|-------|-----------|----------|-------|
| Badge sync job failure | Any failure | Critical | PagerDuty |
| Service 5xx error rate > 1% | Sustained 5 minutes | Critical | PagerDuty |
| Circuit breaker OPEN | Any service | Critical | PagerDuty |
| `identity-service` degraded | 401 error rate > 5% | Critical | PagerDuty |
| Kafka consumer lag > 10,000 | Any topic | Warning | Slack |
| DB connection pool > 80% | Any service | Warning | Slack |
| Pod restart count > 3 | Within 1 hour | Warning | Slack |
| No-show release job failure | Any failure | Warning | Slack |
| HR sync job failure | Any failure | Warning | Slack |
| Nightly badge sync partial failure | Any partial | Warning | Slack |

---

## Log Aggregation

All service stdout is forwarded to a centralised aggregator.

| Environment | Tool |
|-------------|------|
| Development | Docker Compose log driver → local |
| Staging | AWS CloudWatch Logs |
| Production | AWS CloudWatch Logs or ELK stack |

Query pattern (CloudWatch Logs Insights):
```sql
-- Find all logs for a specific correlation ID across all services
fields @timestamp, service, level, action, message
| filter correlationId = "d4e2f9a1-..."
| sort @timestamp asc
```

---

## Observability Checklist (Per Service)

Before a service goes to production, confirm:

- [ ] JSON structured logging enabled with `logstash-logback-encoder`
- [ ] `correlationId` included in every log entry
- [ ] No passwords, tokens, or PII in logs
- [ ] `/actuator/prometheus` reachable and scraped by Prometheus
- [ ] All required metrics emitting correctly
- [ ] `X-Correlation-ID` read from inbound headers and propagated on all outbound calls and Kafka events
- [ ] `/actuator/health/liveness` and `/actuator/health/readiness` returning correct states
- [ ] Kubernetes probes configured in the Helm chart / Deployment YAML
- [ ] Alert rules added to Grafana for this service
