# Observability
## Office Management System (OMS)

**Version:** 3.0 | Logs: CloudWatch | Metrics: Prometheus + Grafana | Traces: OpenTelemetry + Jaeger

---

## 1. Observability Stack

```
Every Spring Boot service
    ↓ stdout (structured JSON)        → CloudWatch Logs
    ↓ /actuator/prometheus            → Prometheus (scrapes every 15s)
    ↓ OpenTelemetry SDK (traces)      → Jaeger (via OTel Collector)
    ↓ /actuator/health/liveness       → ECS task health check
    ↓ /actuator/health/readiness      → ALB target group health check

Prometheus → Grafana (dashboards + alerting)
CloudWatch → CloudWatch Alarms + PagerDuty / Slack
Jaeger     → Distributed trace UI (search by correlationId or traceId)
```

No AWS X-Ray is used. Tracing is OpenTelemetry-native — vendor-independent.

---

## 2. Structured Logging (JSON)

All services output structured JSON to stdout. The ECS `awslogs` driver ships logs to CloudWatch Logs automatically.

### Log Format

```json
{
  "timestamp": "2026-04-17T09:15:42.123Z",
  "level": "INFO",
  "service": "seating-service",
  "traceId": "4e3a2b1c0d9f8e7a",
  "spanId": "1a2b3c4d",
  "correlationId": "550e8400-e29b-41d4-a716-446655440000",
  "userId": "660e8400-e29b-41d4-a716-446655440001",
  "locationId": "770e8400-e29b-41d4-a716-446655440002",
  "action": "SEAT_BOOKED",
  "seatId": "880e8400-e29b-41d4-a716-446655440003",
  "message": "Hot-desk booking confirmed for 2026-04-18",
  "durationMs": 45
}
```

### Logging Rules

- All logs output to **stdout** — never to files (12-Factor principle)
- Never log passwords, tokens, raw session cookies, or PII (email, name)
- Use structured fields (`userId`, `locationId`, `action`) — not string-embedded values
- Log at appropriate levels: INFO for business events, WARN for degraded fallbacks, ERROR for exceptions
- All mutations log: actor, action, entity type, entity ID, location

### Logback Configuration (logback-spring.xml)

```xml
<configuration>
  <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
    <encoder class="net.logstash.logback.encoder.LogstashEncoder">
      <includeMdcKeyName>correlationId</includeMdcKeyName>
      <includeMdcKeyName>userId</includeMdcKeyName>
      <includeMdcKeyName>locationId</includeMdcKeyName>
      <customFields>{"service":"${SERVICE_NAME}"}</customFields>
    </encoder>
  </appender>

  <root level="INFO">
    <appender-ref ref="STDOUT"/>
  </root>

  <logger name="org.springframework.web" level="WARN"/>
  <logger name="com.oms" level="INFO"/>
</configuration>
```

---

## 3. Metrics (Prometheus + Grafana)

Every service exposes `/actuator/prometheus`. Prometheus scrapes all services every 15 seconds.

### Key Metrics Per Service

```
# HTTP
http_server_requests_seconds_count{method, uri, status}   — request count
http_server_requests_seconds_sum{method, uri, status}      — total duration
http_server_requests_seconds_max{method, uri}              — max latency (p99 proxy)

# JVM
jvm_memory_used_bytes{area}                               — heap / non-heap usage
jvm_gc_pause_seconds_count                                — GC frequency

# Database
hikaricp_connections_active{pool}                         — active DB connections
hikaricp_connections_pending{pool}                        — queued DB requests

# RabbitMQ consumers
spring_rabbitmq_listener_seconds_count{listener_id}       — messages processed
spring_rabbitmq_listener_seconds_sum{listener_id}         — total consumer time

# RabbitMQ queue depth (via RabbitMQ Prometheus plugin via Amazon MQ)
rabbitmq_queue_messages{queue}                            — messages waiting

# Circuit breakers
resilience4j_circuitbreaker_state{name}                   — CLOSED/OPEN/HALF_OPEN
resilience4j_circuitbreaker_failure_rate{name}            — failure percentage

# Scheduled jobs (custom)
oms_badge_sync_duration_seconds                           — nightly sync duration
oms_badge_sync_last_success_timestamp                     — last successful run
oms_no_show_release_duration_seconds                      — no-show job duration
```

### Custom Business Metrics (Micrometer)

```java
// attendance-service — badge sync job
@Autowired
private MeterRegistry meterRegistry;

public void runBadgeSync(UUID locationId) {
    Timer.Sample sample = Timer.start(meterRegistry);
    try {
        // ... sync logic ...
        meterRegistry.gauge("oms.badge.sync.last.success",
            Tags.of("location", locationId.toString()),
            Instant.now().getEpochSecond());
    } finally {
        sample.stop(meterRegistry.timer("oms.badge.sync.duration",
            Tags.of("location", locationId.toString())));
    }
}
```

---

## 4. Distributed Tracing (OpenTelemetry + Jaeger)

### Configuration (all services — application.yml)

```yaml
management:
  tracing:
    sampling:
      probability: 1.0      # 100% sampling (reduce in production if needed)

spring:
  application:
    name: seating-service

# OTel exporters (set via environment variables in ECS Task Definition)
# OTEL_SERVICE_NAME=seating-service
# OTEL_EXPORTER_OTLP_ENDPOINT=http://jaeger:4317
# OTEL_TRACES_EXPORTER=otlp
```

### pom.xml dependency

```xml
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-tracing-bridge-otel</artifactId>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-otlp</artifactId>
</dependency>
```

### Trace Flow

```
Browser → GET /api/v1/seat-bookings  [traceId: abc123]
    → Gateway: X-Correlation-ID: req-456  [spanId: 0001]
    → seating-service                     [spanId: 0002, parent: 0001]
        → Feign: GET auth-service/users   [spanId: 0003, parent: 0002]
        → DB: SELECT seat_bookings        [spanId: 0004, parent: 0002]
        → RabbitMQ publish: booking.created [spanId: 0005, parent: 0002]
    → notification-service consumer       [spanId: 0006, parent: 0005]
    → audit-service consumer              [spanId: 0007, parent: 0005]

Jaeger: search traceId=abc123 → see complete waterfall
```

---

## 5. Health Checks

Every service exposes Spring Actuator liveness and readiness endpoints.

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health, prometheus, info
  endpoint:
    health:
      probes:
        enabled: true
      show-details: never    # Never expose DB details in production
  health:
    livenessstate:
      enabled: true
    readinessstate:
      enabled: true
    rabbitmq:
      enabled: true
    db:
      enabled: true
```

| Probe | Endpoint | Checks | Used By |
|---|---|---|---|
| Liveness | `/actuator/health/liveness` | Is JVM running? Any deadlock? | ECS task restart policy |
| Readiness | `/actuator/health/readiness` | DB connected? RabbitMQ connected? Eureka registered? | ALB target group (removes unhealthy tasks from rotation) |

ECS Task Definition health check:
```json
"healthCheck": {
  "command": ["CMD-SHELL", "curl -f http://localhost:8080/actuator/health/readiness || exit 1"],
  "interval": 30,
  "timeout": 5,
  "retries": 3,
  "startPeriod": 60
}
```

---

## 6. Alerting

### Critical Alerts (PagerDuty)

| Alert | Condition | Impact |
|---|---|---|
| Badge sync job failure | `oms_badge_sync_last_success_timestamp` > 26h ago | No attendance data for the day |
| Service error rate high | HTTP 5xx rate > 1% for 5 minutes | User-facing errors |
| Circuit breaker OPEN | `resilience4j_circuitbreaker_state == OPEN` | Service degraded |
| ECS task unhealthy | ALB target health check failing on all tasks | Service unavailable |
| RDS connection failure | `hikaricp_connections_pending > 10` for 2+ minutes | DB saturation |

### Warning Alerts (Slack)

| Alert | Condition | Impact |
|---|---|---|
| RabbitMQ queue depth high | Any queue depth > 10,000 messages | Consumer falling behind |
| No-show release job failure | Job exit code != 0 | Seats not released; employee impact next morning |
| Memory pressure | `jvm_memory_used_bytes` > 85% of limit for 5 minutes | Risk of OOM restart |
| DB connection pool near saturation | `hikaricp_connections_active` > 80% of max | Approaching bottleneck |
| ECS task restart > 3 times | Restart count in 1 hour > 3 | Crash loop risk |

### CloudWatch Alarm Configuration (example)

```json
{
  "AlarmName": "oms-badge-sync-failure",
  "MetricName": "oms_badge_sync_last_success_timestamp",
  "Namespace": "OMS/AttendanceService",
  "Statistic": "Maximum",
  "Period": 3600,
  "EvaluationPeriods": 1,
  "Threshold": 93600,
  "ComparisonOperator": "LessThanThreshold",
  "AlarmActions": ["arn:aws:sns:eu-west-1:ACCOUNT:oms-pagerduty"]
}
```

---

## 7. Grafana Dashboards

| Dashboard | Key Panels |
|---|---|
| **System Overview** | HTTP request rate, p95 latency, 5xx error rate — all services |
| **Service Detail** | Per-service: request rate, latency histogram, DB connections, JVM heap |
| **Kafka / RabbitMQ** | Queue depth per queue, consumer throughput, DLQ message count |
| **Circuit Breakers** | State per service (CLOSED/OPEN/HALF_OPEN), failure rate |
| **Nightly Jobs** | Badge sync duration, last success time, no-show release duration |
| **ECS Health** | Task count per service, restart count, CPU/memory utilisation |
| **Deployment Tracker** | Deployment events overlaid on error rate and latency charts |

---

## 8. Log Queries (CloudWatch Logs Insights)

### Find all errors for a correlation ID
```
fields @timestamp, service, level, message, correlationId
| filter correlationId = "550e8400-e29b-41d4-a716-446655440000"
| sort @timestamp asc
```

### Error rate by service (last 1 hour)
```
fields @timestamp, service, level
| filter level = "ERROR"
| stats count(*) as errorCount by service
| sort errorCount desc
```

### Slow requests (> 1 second)
```
fields @timestamp, service, action, durationMs, userId
| filter durationMs > 1000
| sort durationMs desc
| limit 50
```

### Badge sync job history
```
fields @timestamp, service, message, locationId
| filter service = "attendance-service" AND action = "BADGE_SYNC_COMPLETE"
| sort @timestamp desc
```
