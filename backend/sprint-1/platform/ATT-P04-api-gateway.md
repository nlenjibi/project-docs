# ATT-P04 — AWS API Gateway + Lambda Authorizer (Edge Layer)

| Field | Value |
|-------|-------|
| **Ticket** | ATT-P04 |
| **Epic** | Platform / Infrastructure — M1 |
| **Type** | Story |
| **Priority** | 🔴 Highest |
| **Story Points** | 5 |
| **Sprint** | Sprint 1 |
| **Labels** | platform, api-gateway, auth, security, aws |
| **Perspective** | Infrastructure / Backend |

---

## Story

As the platform, I want an AWS API Gateway (HTTP API) backed by a VPC Link and a Lambda Authorizer so that all external requests are authenticated, rate-limited, enriched with a correlation ID, and routed to the correct ECS service via Cloud Map DNS — without running any custom gateway container.

---

## Background

AWS API Gateway (HTTP API) replaces Spring Cloud Gateway. There is no ECS service to deploy for this component — it is a fully managed AWS service configured via Terraform / AWS CDK as part of M1 infrastructure.

**Why HTTP API over REST API:**
- HTTP API is ~70% cheaper ($1/million requests vs $3.50) and has lower latency
- HTTP API supports Lambda Authorizers, JWT Authorizers, VPC Links, and CORS natively
- Sufficient for OMS — REST API features not needed (no API keys per resource, no request/response transformations beyond headers)

**Three things happen on every non-open request:**

1. **Lambda Authorizer** — extracts the `SESSION` HTTP-only cookie, calls `auth-service /internal/validate` (via Cloud Map DNS inside the VPC), receives `UserContext`, returns an IAM ALLOW policy + user context as authorizer context. Cached for 300 seconds per session token — `auth-service` is not called on every single request.

2. **Internal JWT injection** — the API Gateway stage (via a response template or a dedicated Lambda step) signs a short-lived (300s) HMAC-SHA256 JWT containing the `UserContext` and injects it as the `Authorization: Bearer` header before forwarding to the ECS service.

3. **Header enrichment** — `X-User-Id`, `X-User-Roles`, `X-Location-Ids`, and `X-Correlation-ID` are injected from the authorizer context into every forwarded request.

**VPC Link:** Provides private connectivity from the API Gateway to the private VPC subnets where ECS tasks run. All ECS services are accessed via Cloud Map DNS (`http://service-name.oms.local:PORT`) — no public IPs, no ALB in the routing path.

---

## Acceptance Criteria

### Infrastructure (Terraform / CDK)
- [ ] AWS API Gateway **HTTP API** created: `oms-api`.
- [ ] **VPC Link** created targeting the private VPC subnets (`10.0.10.0/24`, `10.0.11.0/24`, `10.0.12.0/24`). The VPC Link is associated with a security group `sg-api-gateway-vpclink` that can reach `sg-services` on ports 8081–8090.
- [ ] **Custom domain** configured: `api.oms.yourorg.com` → API Gateway stage. TLS certificate via AWS Certificate Manager.
- [ ] **CloudFront distribution** created: static assets (S3 bucket) + `/api/*` forwarded to the API Gateway custom domain.

### Lambda Authorizer
- [ ] A Python 3.12 Lambda function `oms-session-authorizer` is created and associated with the API Gateway as a **request-based** Lambda Authorizer.
- [ ] Authorizer extracts the `SESSION` cookie from the `Cookie` header.
- [ ] Makes a synchronous HTTPS call to `http://auth-service.oms.local:8081/internal/validate` with the session cookie value. Uses `urllib3` (no external dependencies — Lambda layer not required).
- [ ] On `200` from auth-service: returns an `Allow` IAM policy for all resources (`arn:aws:execute-api:*:*:*/*/*/*`) with context:
  ```json
  {
    "userId": "uuid",
    "userRoles": "MANAGER,EMPLOYEE",
    "locationIds": "uuid1,uuid2",
    "correlationId": "generated-uuid"
  }
  ```
- [ ] On `401` from auth-service: returns an `Unauthorized` response (raises `Exception("Unauthorized")`).
- [ ] On auth-service unavailability (connection error, timeout): returns `Deny` policy. The ECS service never receives the request.
- [ ] Authorizer **cache TTL: 300 seconds** keyed on the `SESSION` cookie value. Reduces auth-service load dramatically at scale.
- [ ] Authorizer timeout: **3 seconds** maximum (Lambda timeout configured to 5s to account for cold starts).
- [ ] `INTERNAL_JWT_SECRET` is retrieved from **AWS Secrets Manager** at Lambda cold start and cached in memory for the Lambda execution environment lifetime. Never hardcoded.
- [ ] The Lambda generates the **internal JWT** in the authorizer context: signs `{ sub, roles, locationIds, correlationId, iss: "oms-gateway", iat, exp: iat+300 }` with HMAC-SHA256.
- [ ] Lambda IAM role: `AWSLambdaVPCAccessExecutionRole` + `secretsmanager:GetSecretValue` on `oms/internal-jwt-secret`.
- [ ] Lambda deployed inside the VPC (same private subnets as ECS) to reach Cloud Map DNS.

### Open Paths (bypass Lambda Authorizer)
- [ ] The following routes bypass the authorizer entirely (configured as routes with no authorizer):
  - `GET /api/v1/auth/login`
  - `GET /api/v1/auth/callback`
  - `GET /health`
- [ ] All other routes have the Lambda Authorizer attached.

### Header Injection (via API Gateway integration)
- [ ] The following headers are injected into every forwarded request (configured in the API Gateway integration's request parameters):
  - `X-User-Id: $context.authorizer.userId`
  - `X-User-Roles: $context.authorizer.userRoles`
  - `X-Location-Ids: $context.authorizer.locationIds`
  - `X-Correlation-ID: $context.authorizer.correlationId`
  - `Authorization: Bearer $context.authorizer.internalJwt`
- [ ] Any `X-User-*` or `Authorization` headers arriving from the browser are **stripped** before forwarding (configured via `overwrite:header.Authorization` in the integration).

### Route Configuration
- [ ] All routes are configured as HTTP proxy integrations to Cloud Map DNS targets via VPC Link:

  | Route | Integration Target |
  |---|---|
  | `ANY /api/v1/auth/{proxy+}` | `http://auth-service.oms.local:8081/api/v1/auth/{proxy}` |
  | `ANY /api/v1/users/{proxy+}` | `http://auth-service.oms.local:8081/api/v1/users/{proxy}` |
  | `ANY /api/v1/locations/{proxy+}` | `http://auth-service.oms.local:8081/api/v1/locations/{proxy}` |
  | `ANY /api/v1/attendance/{proxy+}` | `http://attendance-service.oms.local:8083/api/v1/attendance/{proxy}` |
  | `ANY /api/v1/seat-bookings/{proxy+}` | `http://seating-service.oms.local:8084/api/v1/seat-bookings/{proxy}` |
  | `ANY /api/v1/seats/{proxy+}` | `http://seating-service.oms.local:8084/api/v1/seats/{proxy}` |
  | `ANY /api/v1/remote-requests/{proxy+}` | `http://remote-service.oms.local:8085/api/v1/remote-requests/{proxy}` |
  | `ANY /api/v1/ooo-requests/{proxy+}` | `http://remote-service.oms.local:8085/api/v1/ooo-requests/{proxy}` |
  | `ANY /api/v1/notifications/{proxy+}` | `http://notification-service.oms.local:8086/api/v1/notifications/{proxy}` |
  | `ANY /api/v1/audit-logs/{proxy+}` | `http://audit-service.oms.local:8087/api/v1/audit-logs/{proxy}` |

- [ ] Timeout on each integration: **29 seconds** (API Gateway HTTP API maximum). Service-level timeouts (3s) are enforced by the downstream services — the gateway timeout is a safety ceiling only.

### Rate Limiting
- [ ] API Gateway **usage plan** created: 100 requests/second burst, 50 requests/second steady rate per user (keyed on `X-User-Id` injected by authorizer).
- [ ] API Gateway returns `429 Too Many Requests` with body: `{ "success": false, "error": { "code": "RATE_LIMITED" } }`.

### CORS
- [ ] CORS configured at the API Gateway stage level:
  - `allowedOrigins`: `CORS_ALLOWED_ORIGINS` parameter from SSM (comma-separated; `*` forbidden in production).
  - `allowedMethods`: `GET, POST, PUT, PATCH, DELETE, OPTIONS`.
  - `allowedHeaders`: `Content-Type, X-Correlation-ID, Cookie`.
  - `allowCredentials: true` (required for session cookie).
  - `maxAge: 3600`.
- [ ] `OPTIONS` pre-flight requests return `200` immediately without hitting the Lambda Authorizer.

### Lambda Authorizer Code

```python
import json
import os
import urllib3
import boto3
import hmac
import hashlib
import base64
import time
import uuid

# Cached at Lambda execution environment level
_jwt_secret = None
http = urllib3.PoolManager()

def get_jwt_secret():
    global _jwt_secret
    if _jwt_secret is None:
        sm = boto3.client("secretsmanager")
        _jwt_secret = sm.get_secret_value(SecretId="oms/internal-jwt-secret")["SecretString"]
    return _jwt_secret

def generate_internal_jwt(user_context: dict, correlation_id: str) -> str:
    secret = get_jwt_secret()
    now = int(time.time())
    payload = {
        "sub": user_context["userId"],
        "roles": user_context["roles"],
        "locationIds": user_context["locationIds"],
        "correlationId": correlation_id,
        "iss": "oms-gateway",
        "iat": now,
        "exp": now + 300,
    }
    header = base64.urlsafe_b64encode(json.dumps({"alg": "HS256", "typ": "JWT"}).encode()).rstrip(b"=")
    body = base64.urlsafe_b64encode(json.dumps(payload).encode()).rstrip(b"=")
    sig_input = header + b"." + body
    sig = base64.urlsafe_b64encode(
        hmac.new(secret.encode(), sig_input, hashlib.sha256).digest()
    ).rstrip(b"=")
    return (sig_input + b"." + sig).decode()

def lambda_handler(event, context):
    cookies = event.get("headers", {}).get("cookie", "")
    session_token = None
    for part in cookies.split(";"):
        if part.strip().startswith("SESSION="):
            session_token = part.strip()[len("SESSION="):]
            break

    if not session_token:
        raise Exception("Unauthorized")

    auth_url = os.environ["AUTH_SERVICE_VALIDATE_URL"]  # http://auth-service.oms.local:8081/internal/validate
    try:
        resp = http.request(
            "GET", auth_url,
            headers={"X-Session-Token": session_token},
            timeout=3.0,
        )
    except Exception:
        return generate_policy("Deny", event["routeArn"], {})

    if resp.status != 200:
        raise Exception("Unauthorized")

    user_context = json.loads(resp.data.decode())
    correlation_id = str(uuid.uuid4())
    internal_jwt = generate_internal_jwt(user_context, correlation_id)

    return {
        "isAuthorized": True,
        "context": {
            "userId": user_context["userId"],
            "userRoles": ",".join(user_context["roles"]),
            "locationIds": ",".join(user_context["locationIds"]),
            "correlationId": correlation_id,
            "internalJwt": internal_jwt,
        },
    }
```

### Tests
- [ ] Unit tests (mocked `urllib3`): valid session → ALLOW policy with correct context; no SESSION cookie → `Unauthorized` exception; auth-service 401 → `Unauthorized`; auth-service unreachable → `Deny` policy; JWT generation includes correct claims and expiry.
- [ ] Integration test: deploy Lambda to dev environment; send request with valid session cookie; assert `X-User-Id` header reaches the downstream ECS service; assert 401 on missing cookie; assert 429 on exceeded rate limit.

---

## Environment Variables (Lambda)

```bash
AUTH_SERVICE_VALIDATE_URL=http://auth-service.oms.local:8081/internal/validate
AWS_REGION=eu-west-1
# INTERNAL_JWT_SECRET fetched from Secrets Manager at cold start
```

---

## Definition of Done

- [ ] Terraform / CDK code reviewed and merged
- [ ] API Gateway stage deployed in DEV; all routes reachable via VPC Link
- [ ] Lambda Authorizer deployed; cache TTL confirmed (300s)
- [ ] `INTERNAL_JWT_SECRET` fetched from Secrets Manager — never in Lambda env vars or code
- [ ] Spoofed `X-User-Id` / `Authorization` headers from browser stripped (tested)
- [ ] CORS pre-flight returns 200 without hitting authorizer (tested)
- [ ] Rate limiting confirmed: 101st request → 429 (tested)
- [ ] CloudWatch log group created for Lambda (`/aws/lambda/oms-session-authorizer`)
- [ ] CloudWatch Alarm created: Lambda error rate > 1% → SNS alert
- [ ] Jenkins pipeline green for Lambda deployment stage
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **ATT-P05** (AWS Cloud Map) must be deployed — Lambda Authorizer calls `auth-service.oms.local` and cannot resolve the DNS until Cloud Map namespace exists.
- **auth-service M2** must expose `GET /internal/validate` before the Lambda Authorizer can be tested end-to-end. In Sprint 1, stub the validate endpoint with a mock response.
- `oms/internal-jwt-secret` must be created in AWS Secrets Manager before Lambda cold start.
- VPC subnets and security group `sg-api-gateway-vpclink` must be provisioned as part of M1 network setup.
