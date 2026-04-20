# ATT-P09 — AWS API Gateway HTTP API + VPC Link Infrastructure

| Field | Value |
|-------|-------|
| **Ticket** | ATT-P09 |
| **Epic** | Platform / Infrastructure — M1 |
| **Type** | Story |
| **Priority** | 🔴 Highest |
| **Story Points** | 3 |
| **Sprint** | Sprint 1 |
| **Labels** | platform, api-gateway, vpc-link, terraform, infrastructure, aws |
| **Perspective** | Infrastructure |

---

## Story

As the platform, I want the AWS API Gateway HTTP API, VPC Link, route definitions, CORS configuration, WAF association, and custom domain provisioned via Terraform so that all external traffic has a managed, secure entry point into the private ECS network — without running any gateway container.

---

## Background

This ticket covers the **Terraform infrastructure provisioning** for AWS API Gateway HTTP API and VPC Link. The Lambda Authorizer code (Python 3.12), unit tests, and deployment pipeline are tracked in **ATT-P04**.

**Why this is a separate ticket from ATT-P04:**
ATT-P04 covers the Lambda Authorizer implementation — writing Python code, handling edge cases, testing the auth logic. This ticket is the infrastructure provisioning work: creating the HTTP API resource, attaching the VPC Link, defining all routes, configuring CORS and WAF, and creating the custom domain. These are two different skill sets (Terraform vs Python) and two different PRs.

**Architecture:**
```
Internet
    ↓ HTTPS
Amazon CloudFront (ATT-P06)
    ↓ /api/**
AWS API Gateway HTTP API   ← this ticket
    + Lambda Authorizer    ← ATT-P04
    + VPC Link             ← this ticket
    ↓
ECS private subnets
    ↓ Cloud Map DNS (oms.local)
ECS Services               ← ATT-P08
```

**VPC Link:** A VPC Link connects the API Gateway to the private subnets where ECS tasks run. All integration targets are Cloud Map DNS names (`http://service-name.oms.local:PORT`) — no NLB per service, no static IP. The VPC Link routes traffic to the correct task IP via Cloud Map A record resolution.

---

## Acceptance Criteria

### AWS API Gateway HTTP API
- [ ] HTTP API (not REST API) created: name `oms-api`, description `OMS v4.0 HTTP API`.
- [ ] Auto-deploy disabled — deployments are managed explicitly via Terraform.
- [ ] API stage `$default` created with access logging enabled to CloudWatch Logs group `/aws/apigateway/oms-api`.
- [ ] Access log format (JSON):
  ```json
  {
    "requestId": "$context.requestId",
    "sourceIp": "$context.identity.sourceIp",
    "routeKey": "$context.routeKey",
    "status": "$context.status",
    "responseLength": "$context.responseLength",
    "durationMs": "$context.responseLatency",
    "userId": "$context.authorizer.userId",
    "correlationId": "$context.authorizer.correlationId"
  }
  ```

### VPC Link
- [ ] VPC Link created: name `oms-vpclink`, targeting the **private subnets** (`10.0.10.0/24`, `10.0.11.0/24`, `10.0.12.0/24`).
- [ ] VPC Link associated with security group `sg-api-gateway-vpclink`:
  - Outbound: TCP 8081–8090 to `sg-services` (ECS tasks)
  - No inbound rules (VPC Link is egress from API Gateway perspective)
- [ ] VPC Link status must reach `AVAILABLE` before routes are created (Terraform `depends_on`).

### Lambda Authorizer Attachment
- [ ] Lambda Authorizer `oms-session-authorizer` (created in ATT-P04) attached to the HTTP API as a **request-based** authorizer.
- [ ] Authorizer cache TTL: **300 seconds**, keyed on the `SESSION` cookie value (identity source: `$request.header.Cookie`).
- [ ] All routes (except open paths) reference this authorizer.

### Route Configuration
- [ ] All routes created as **HTTP proxy integrations** via VPC Link to Cloud Map DNS targets:

  | Route | Integration Target | Authorizer |
  |---|---|---|
  | `GET /api/v1/auth/login` | `http://auth-service.oms.local:8081/api/v1/auth/login` | None (open) |
  | `GET /api/v1/auth/callback` | `http://auth-service.oms.local:8081/api/v1/auth/callback` | None (open) |
  | `POST /api/v1/auth/logout` | `http://auth-service.oms.local:8081/api/v1/auth/logout` | Lambda Authorizer |
  | `ANY /api/v1/users/{proxy+}` | `http://auth-service.oms.local:8081/api/v1/users/{proxy}` | Lambda Authorizer |
  | `ANY /api/v1/locations/{proxy+}` | `http://auth-service.oms.local:8081/api/v1/locations/{proxy}` | Lambda Authorizer |
  | `ANY /api/v1/attendance/{proxy+}` | `http://attendance-service.oms.local:8083/api/v1/attendance/{proxy}` | Lambda Authorizer |
  | `ANY /api/v1/seat-bookings/{proxy+}` | `http://seating-service.oms.local:8084/api/v1/seat-bookings/{proxy}` | Lambda Authorizer |
  | `ANY /api/v1/seats/{proxy+}` | `http://seating-service.oms.local:8084/api/v1/seats/{proxy}` | Lambda Authorizer |
  | `ANY /api/v1/remote-requests/{proxy+}` | `http://remote-service.oms.local:8085/api/v1/remote-requests/{proxy}` | Lambda Authorizer |
  | `ANY /api/v1/ooo-requests/{proxy+}` | `http://remote-service.oms.local:8085/api/v1/ooo-requests/{proxy}` | Lambda Authorizer |
  | `ANY /api/v1/notifications/{proxy+}` | `http://notification-service.oms.local:8086/api/v1/notifications/{proxy}` | Lambda Authorizer |
  | `ANY /api/v1/audit-logs/{proxy+}` | `http://audit-service.oms.local:8087/api/v1/audit-logs/{proxy}` | Lambda Authorizer |
  | `ANY /api/v1/supplies/{proxy+}` | `http://inventory-service.oms.local:8090/api/v1/supplies/{proxy}` | Lambda Authorizer |
  | `ANY /api/v1/workplace/{proxy+}` | `http://workplace-service.oms.local:8088/api/v1/workplace/{proxy}` | Lambda Authorizer |
  | `GET /health` | `http://auth-service.oms.local:8081/actuator/health` | None (open) |

- [ ] Integration timeout: **29 seconds** on every route (HTTP API maximum). Downstream services enforce their own timeouts (3–5s) — the gateway timeout is a safety ceiling only.
- [ ] Header transformation on every authorised integration:
  - **Injected** (from authorizer context):
    - `overwrite:header.X-User-Id` → `$context.authorizer.userId`
    - `overwrite:header.X-User-Roles` → `$context.authorizer.userRoles`
    - `overwrite:header.X-Location-Ids` → `$context.authorizer.locationIds`
    - `overwrite:header.X-Correlation-ID` → `$context.authorizer.correlationId`
    - `overwrite:header.Authorization` → `Bearer $context.authorizer.internalJwt`
  - **Stripped** (client-supplied, must not reach services):
    - Any `X-User-*` header arriving from the browser is overwritten, not forwarded

### CORS
- [ ] CORS configured at the API stage level:
  - `allowOrigins`: `CORS_ALLOWED_ORIGINS` SSM parameter (comma-separated list; `*` forbidden in production)
  - `allowMethods`: `GET, POST, PUT, PATCH, DELETE, OPTIONS`
  - `allowHeaders`: `Content-Type, X-Correlation-ID, Cookie`
  - `allowCredentials: true` (required for `SESSION` cookie)
  - `maxAge: 3600`
- [ ] `OPTIONS` pre-flight requests return 200 immediately without invoking the Lambda Authorizer.

### Throttling
- [ ] API Gateway stage default throttling:
  - Burst limit: **150 requests**
  - Rate limit: **100 requests/second**
- [ ] Per-route throttling: same as stage defaults (can be overridden per route if needed post-deployment).
- [ ] API Gateway returns `429 Too Many Requests` with body `{ "message": "Too Many Requests" }` on throttle.

### Custom Domain
- [ ] API Gateway custom domain name `api.oms.yourorg.com` created.
- [ ] ACM certificate for `api.oms.yourorg.com` in `eu-west-1` attached (HTTP API uses regional certificates).
- [ ] API mapping: `api.oms.yourorg.com` → `oms-api` stage `$default`.
- [ ] Route 53 alias record: `api.oms.yourorg.com` → API Gateway regional domain name.

### WAF WebACL
- [ ] WAF WebACL `oms-api-waf` (scope: `REGIONAL`, in `eu-west-1`) created and associated with the API Gateway stage.
- [ ] Managed rule groups:
  - `AWSManagedRulesCommonRuleSet`
  - `AWSManagedRulesKnownBadInputsRuleSet`
- [ ] Rate-based rule: block IP sending > 1,000 requests per 5 minutes.
- [ ] WAF logs delivered to CloudWatch Logs group `/aws/waf/oms-api`.

### Tests
- [ ] `GET https://api.oms.yourorg.com/health` returns 200 (open path, no authorizer).
- [ ] `GET https://api.oms.yourorg.com/api/v1/attendance/` without session cookie → 401 (Lambda Authorizer rejects).
- [ ] `OPTIONS https://api.oms.yourorg.com/api/v1/attendance/` returns 200 with correct `Access-Control-Allow-Origin` header (CORS pre-flight bypasses authorizer).
- [ ] Valid session cookie → request reaches ECS service with `X-User-Id` and `Authorization: Bearer ...` headers injected.
- [ ] Client-supplied `X-User-Id: attacker-uuid` header → overwritten by authorizer context (spoofing prevented).
- [ ] Throttle test: 101 rapid requests from same client → 429 returned.
- [ ] VPC Link connectivity: API Gateway reaches `auth-service.oms.local:8081` via VPC Link (confirmed via access log showing non-5xx response).

---

## Terraform Resource Outline

```hcl
# VPC Link
resource "aws_apigatewayv2_vpc_link" "oms" {
  name               = "oms-vpclink"
  subnet_ids         = var.private_subnet_ids
  security_group_ids = [aws_security_group.apigw_vpclink.id]
}

resource "aws_security_group" "apigw_vpclink" {
  name        = "sg-api-gateway-vpclink"
  description = "API Gateway VPC Link security group"
  vpc_id      = var.vpc_id

  egress {
    from_port       = 8081
    to_port         = 8090
    protocol        = "tcp"
    security_groups = [aws_security_group.services.id]
  }
}

# HTTP API
resource "aws_apigatewayv2_api" "oms" {
  name          = "oms-api"
  protocol_type = "HTTP"
  description   = "OMS v4.0 HTTP API"

  cors_configuration {
    allow_origins     = split(",", data.aws_ssm_parameter.cors_origins.value)
    allow_methods     = ["GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"]
    allow_headers     = ["Content-Type", "X-Correlation-ID", "Cookie"]
    allow_credentials = true
    max_age           = 3600
  }
}

# Lambda Authorizer (Lambda function created in ATT-P04)
resource "aws_apigatewayv2_authorizer" "session" {
  api_id                            = aws_apigatewayv2_api.oms.id
  authorizer_type                   = "REQUEST"
  authorizer_uri                    = aws_lambda_function.session_authorizer.invoke_arn
  identity_sources                  = ["$request.header.Cookie"]
  name                              = "oms-session-authorizer"
  authorizer_result_ttl_in_seconds  = 300
  enable_simple_responses           = true
}

# Stage with access logging
resource "aws_apigatewayv2_stage" "default" {
  api_id      = aws_apigatewayv2_api.oms.id
  name        = "$default"
  auto_deploy = true

  access_log_settings {
    destination_arn = aws_cloudwatch_log_group.api_access.arn
    format = jsonencode({
      requestId       = "$context.requestId"
      sourceIp        = "$context.identity.sourceIp"
      routeKey        = "$context.routeKey"
      status          = "$context.status"
      responseLength  = "$context.responseLength"
      durationMs      = "$context.responseLatency"
      userId          = "$context.authorizer.userId"
      correlationId   = "$context.authorizer.correlationId"
    })
  }

  default_route_settings {
    throttling_burst_limit = 150
    throttling_rate_limit  = 100
  }
}

# Integration — reusable module pattern (example: attendance-service)
resource "aws_apigatewayv2_integration" "attendance" {
  api_id             = aws_apigatewayv2_api.oms.id
  integration_type   = "HTTP_PROXY"
  integration_uri    = "http://attendance-service.oms.local:8083/api/v1/attendance/{proxy}"
  integration_method = "ANY"
  connection_type    = "VPC_LINK"
  connection_id      = aws_apigatewayv2_vpc_link.oms.id
  timeout_milliseconds = 29000

  request_parameters = {
    "overwrite:header.X-User-Id"       = "$context.authorizer.userId"
    "overwrite:header.X-User-Roles"    = "$context.authorizer.userRoles"
    "overwrite:header.X-Location-Ids"  = "$context.authorizer.locationIds"
    "overwrite:header.X-Correlation-ID" = "$context.authorizer.correlationId"
    "overwrite:header.Authorization"   = "Bearer $context.authorizer.internalJwt"
  }
}

resource "aws_apigatewayv2_route" "attendance" {
  api_id             = aws_apigatewayv2_api.oms.id
  route_key          = "ANY /api/v1/attendance/{proxy+}"
  target             = "integrations/${aws_apigatewayv2_integration.attendance.id}"
  authorization_type = "CUSTOM"
  authorizer_id      = aws_apigatewayv2_authorizer.session.id
}

# Custom domain
resource "aws_apigatewayv2_domain_name" "api" {
  domain_name = "api.oms.yourorg.com"
  domain_name_configuration {
    certificate_arn = var.api_acm_certificate_arn
    endpoint_type   = "REGIONAL"
    security_policy = "TLS_1_2"
  }
}

resource "aws_apigatewayv2_api_mapping" "api" {
  api_id      = aws_apigatewayv2_api.oms.id
  domain_name = aws_apigatewayv2_domain_name.api.id
  stage       = aws_apigatewayv2_stage.default.id
}

# Route 53 alias for custom domain
resource "aws_route53_record" "api" {
  zone_id = var.route53_zone_id
  name    = "api.oms.yourorg.com"
  type    = "A"
  alias {
    name                   = aws_apigatewayv2_domain_name.api.domain_name_configuration[0].target_domain_name
    zone_id                = aws_apigatewayv2_domain_name.api.domain_name_configuration[0].hosted_zone_id
    evaluate_target_health = false
  }
}
```

---

## Terraform Outputs

```hcl
output "api_gateway_id" {
  value = aws_apigatewayv2_api.oms.id
}

output "api_gateway_endpoint" {
  value = aws_apigatewayv2_api.oms.api_endpoint
}

output "api_custom_domain" {
  value = aws_apigatewayv2_domain_name.api.domain_name_configuration[0].target_domain_name
}

output "vpc_link_id" {
  value = aws_apigatewayv2_vpc_link.oms.id
}
```

---

## Definition of Done

- [ ] Terraform code reviewed and merged
- [ ] HTTP API created; VPC Link status `AVAILABLE`
- [ ] All 14 routes configured with correct integration targets and authorizer attachment
- [ ] Open paths (`/health`, `/api/v1/auth/login`, `/api/v1/auth/callback`) bypass authorizer (confirmed: no 401 without session cookie)
- [ ] Header injection confirmed: `X-User-Id`, `X-User-Roles`, `Authorization: Bearer` present on requests reaching ECS
- [ ] Client-supplied `X-User-Id` header overwritten (spoofing test passes)
- [ ] CORS pre-flight returns 200; `Access-Control-Allow-Credentials: true`
- [ ] Throttle test: 101st request → 429
- [ ] Custom domain `api.oms.yourorg.com` resolves and serves HTTPS
- [ ] WAF WebACL attached; OWASP rule set active
- [ ] Access log group `/aws/apigateway/oms-api` receiving structured JSON logs
- [ ] CloudWatch Alarm: API Gateway 5xx rate > 1% for 5 minutes → SNS alert
- [ ] Jenkins pipeline green for Terraform plan and apply
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **ATT-P08** (Cloud Map Namespace) must be deployed — VPC Link integration targets (`http://service-name.oms.local`) are not resolvable until the `oms.local` namespace exists.
- **ATT-P04** (Lambda Authorizer code) must be deployed — the Lambda function ARN is referenced in `aws_apigatewayv2_authorizer.session.authorizer_uri`.
- ACM certificate for `api.oms.yourorg.com` in `eu-west-1` must be issued.
- Route 53 hosted zone for `oms.yourorg.com` must exist.
- `sg-services` security group (from ATT-P08) must exist before `sg-api-gateway-vpclink` egress rule can reference it.
- **ATT-P06** (CloudFront) depends on this ticket — CloudFront's `/api/*` origin targets the API Gateway custom domain.
