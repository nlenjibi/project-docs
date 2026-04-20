# ATT-P06 — CloudFront Distribution Setup

| Field | Value |
|-------|-------|
| **Ticket** | ATT-P06 |
| **Epic** | Platform / Infrastructure — M1 |
| **Type** | Story |
| **Priority** | 🔴 Highest |
| **Story Points** | 3 |
| **Sprint** | Sprint 1 |
| **Labels** | platform, cloudfront, cdn, infrastructure, aws |
| **Perspective** | Infrastructure |

---

## Story

As the platform, I want an Amazon CloudFront distribution that serves the React SPA from S3 and proxies `/api/**` requests to the API Gateway, so that static assets are globally cached with long TTLs, the SPA is delivered over HTTPS without a load balancer, and security headers are applied at the edge.

---

## Background

CloudFront replaces the public Application Load Balancer (ALB) in v4.0. There is no ECS service to deploy for this component — it is a fully managed AWS CDN layer provisioned via Terraform as part of M1 infrastructure.

**Two origins, two behaviors:**

| Behavior | Origin | Caching |
|---|---|---|
| Default (`/*`) | S3 bucket (`oms-frontend-assets`) | Long TTL for hashed assets; `no-cache` for `index.html` |
| `/api/*` | API Gateway HTTP API endpoint | Caching disabled; cookies and headers forwarded |

**Why CloudFront over a public ALB:**
- Eliminates ~$35/month public ALB cost
- SPA assets are globally cached at CloudFront edge locations — faster for users outside the primary region
- WAF WebACL can be attached to CloudFront at no extra infrastructure cost
- `SameSite=Strict` cookies pass through CloudFront correctly to API Gateway

**React SPA deployment flow:**
```
Jenkins CI build  →  aws s3 sync  →  S3 bucket  →  CloudFront (CDN)
                                                    ↓ /api/*
                                               API Gateway HTTP API
```

---

## Acceptance Criteria

### S3 Origin (SPA Assets)
- [ ] S3 bucket `oms-frontend-{env}` created with **all public access blocked**.
- [ ] **Origin Access Control (OAC)** created; bucket policy grants `s3:GetObject` only to the CloudFront OAC principal — no public `s3:GetObject`.
- [ ] Bucket versioning enabled — allows rollback to previous deploy.
- [ ] Bucket has no website hosting enabled — CloudFront serves files directly via S3 API.

### CloudFront Distribution
- [ ] CloudFront distribution created with two origins:
  - **Origin 1 (S3):** `oms-frontend-{env}.s3.eu-west-1.amazonaws.com` — default cache behavior (`/*`)
  - **Origin 2 (API Gateway):** API Gateway custom domain (e.g., `api.oms.yourorg.com`) — `/api/*` behavior
- [ ] **Default (`/*`) cache behavior:**
  - Origin: S3
  - Cache policy: managed `CachingOptimized` (TTL 86400s default; overridden per object by `Cache-Control` response header)
  - Origin request policy: `CORS-S3Origin`
  - Viewer protocol: **HTTPS only** (redirect HTTP → HTTPS)
- [ ] **`/api/*` cache behavior:**
  - Origin: API Gateway endpoint
  - Cache policy: `CachingDisabled` (TTL 0) — responses must never be cached (session-based auth)
  - Origin request policy: `AllViewerExceptHostHeader` — forwards all headers (including `Cookie`) and query strings to API Gateway
  - Viewer protocol: HTTPS only
- [ ] Custom domain `app.oms.yourorg.com` configured with an **ACM certificate in `us-east-1`** (CloudFront requires certificates in us-east-1).
- [ ] IPv6 enabled.
- [ ] Default root object: `index.html`.
- [ ] Custom error pages: 403 and 404 → return `index.html` with HTTP 200 (required for React Router client-side routing).

### Security Headers (CloudFront Response Headers Policy)
- [ ] A custom Response Headers Policy `oms-security-headers` applied to the default (`/*`) behavior with the following headers:

  | Header | Value |
  |---|---|
  | `Strict-Transport-Security` | `max-age=31536000; includeSubDomains; preload` |
  | `X-Content-Type-Options` | `nosniff` |
  | `X-Frame-Options` | `DENY` |
  | `Referrer-Policy` | `strict-origin-when-cross-origin` |
  | `Content-Security-Policy` | `default-src 'self'; script-src 'self'; connect-src 'self' https://api.oms.yourorg.com; img-src 'self' data:; style-src 'self' 'unsafe-inline'` |

- [ ] Security headers policy does **not** apply to the `/api/*` behavior (API Gateway sets its own headers).

### WAF WebACL
- [ ] WAF WebACL `oms-waf` created (scope: `CLOUDFRONT`, must be in `us-east-1`) and associated with the distribution.
- [ ] Managed rule groups enabled:
  - `AWSManagedRulesCommonRuleSet` — OWASP Core Rule Set
  - `AWSManagedRulesKnownBadInputsRuleSet`
- [ ] Rate-based rule: **block** any IP sending > 2,000 requests per 5-minute window.
- [ ] WAF logs delivered to CloudWatch Logs group `/aws/waf/oms`.

### CI/CD — React SPA Deployment (Jenkins)
- [ ] Jenkins pipeline stage `Deploy Frontend` runs after successful React build:
  ```bash
  # Upload all assets with long-lived cache (hashed filenames)
  aws s3 sync build/ s3://oms-frontend-${ENV}/ \
    --delete \
    --cache-control "max-age=31536000, immutable" \
    --exclude "index.html"

  # Upload index.html with no-cache (always re-fetched)
  aws s3 cp build/index.html s3://oms-frontend-${ENV}/index.html \
    --cache-control "no-cache, no-store, must-revalidate"

  # Invalidate CloudFront cache for index.html
  aws cloudfront create-invalidation \
    --distribution-id ${CLOUDFRONT_DISTRIBUTION_ID} \
    --paths "/index.html"
  ```
- [ ] `CLOUDFRONT_DISTRIBUTION_ID` stored in Jenkins credentials store (not hardcoded).
- [ ] Deployment fails build if `aws s3 sync` or invalidation command exits non-zero.

### Tests
- [ ] `curl https://app.oms.yourorg.com/` returns HTTP 200 with `Content-Type: text/html`.
- [ ] `curl https://app.oms.yourorg.com/some/spa/route` returns HTTP 200 with `index.html` (React Router fallback works).
- [ ] `curl https://app.oms.yourorg.com/api/v1/health` proxied to API Gateway; returns 200 (confirms `/api/*` behavior).
- [ ] `curl http://app.oms.yourorg.com/` redirects to HTTPS (301/302).
- [ ] `index.html` response includes `Cache-Control: no-cache`.
- [ ] A static JS asset (hashed filename) includes `Cache-Control: max-age=31536000`.
- [ ] Response headers include `X-Content-Type-Options: nosniff` and `X-Frame-Options: DENY`.
- [ ] `SESSION` cookie is forwarded through CloudFront to API Gateway without stripping (confirmed via API Gateway access log showing `Cookie` header).

---

## Terraform Resource Outline

```hcl
# S3 bucket
resource "aws_s3_bucket" "frontend" {
  bucket = "oms-frontend-${var.env}"
}

resource "aws_s3_bucket_public_access_block" "frontend" {
  bucket                  = aws_s3_bucket.frontend.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Origin Access Control
resource "aws_cloudfront_origin_access_control" "frontend" {
  name                              = "oms-frontend-oac-${var.env}"
  origin_access_control_origin_type = "s3"
  signing_behavior                  = "always"
  signing_protocol                  = "sigv4"
}

# CloudFront distribution
resource "aws_cloudfront_distribution" "oms" {
  enabled             = true
  default_root_object = "index.html"
  aliases             = ["app.oms.yourorg.com"]

  origin {
    domain_name              = aws_s3_bucket.frontend.bucket_regional_domain_name
    origin_id                = "S3-frontend"
    origin_access_control_id = aws_cloudfront_origin_access_control.frontend.id
  }

  origin {
    domain_name = var.api_gateway_domain   # e.g. api.oms.yourorg.com
    origin_id   = "APIGateway"
    custom_origin_config {
      http_port              = 80
      https_port             = 443
      origin_protocol_policy = "https-only"
      origin_ssl_protocols   = ["TLSv1.2"]
    }
  }

  # /api/* → API Gateway (no cache)
  ordered_cache_behavior {
    path_pattern           = "/api/*"
    target_origin_id       = "APIGateway"
    viewer_protocol_policy = "https-only"
    allowed_methods        = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods         = ["GET", "HEAD"]
    cache_policy_id        = data.aws_cloudfront_cache_policy.caching_disabled.id
    origin_request_policy_id = data.aws_cloudfront_origin_request_policy.all_viewer_except_host.id
  }

  # Default → S3 (long cache)
  default_cache_behavior {
    target_origin_id       = "S3-frontend"
    viewer_protocol_policy = "redirect-to-https"
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]
    cache_policy_id        = data.aws_cloudfront_cache_policy.caching_optimized.id
    response_headers_policy_id = aws_cloudfront_response_headers_policy.security.id
  }

  custom_error_response {
    error_code         = 403
    response_code      = 200
    response_page_path = "/index.html"
  }
  custom_error_response {
    error_code         = 404
    response_code      = 200
    response_page_path = "/index.html"
  }

  viewer_certificate {
    acm_certificate_arn      = var.acm_certificate_arn  # Must be in us-east-1
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }

  web_acl_id = aws_wafv2_web_acl.oms.arn
}

# S3 bucket policy — CloudFront OAC access only
resource "aws_s3_bucket_policy" "frontend" {
  bucket = aws_s3_bucket.frontend.id
  policy = jsonencode({
    Statement = [{
      Effect    = "Allow"
      Principal = { Service = "cloudfront.amazonaws.com" }
      Action    = "s3:GetObject"
      Resource  = "${aws_s3_bucket.frontend.arn}/*"
      Condition = {
        StringEquals = {
          "AWS:SourceArn" = aws_cloudfront_distribution.oms.arn
        }
      }
    }]
  })
}
```

---

## Terraform Outputs

```hcl
output "cloudfront_distribution_id" {
  value = aws_cloudfront_distribution.oms.id
}

output "cloudfront_domain_name" {
  value = aws_cloudfront_distribution.oms.domain_name
}

output "frontend_bucket_name" {
  value = aws_s3_bucket.frontend.bucket
}
```

---

## Definition of Done

- [ ] Terraform code reviewed and merged
- [ ] S3 bucket created with public access blocked; OAC configured
- [ ] CloudFront distribution deployed; SPA served at `app.oms.yourorg.com`
- [ ] `/api/*` requests proxied to API Gateway with `SESSION` cookie intact (confirmed via API Gateway log)
- [ ] `index.html` served with `Cache-Control: no-cache`; hashed assets with `max-age=31536000`
- [ ] React Router fallback: 404 → `index.html` with HTTP 200
- [ ] Security headers policy applied (HSTS, X-Frame-Options, CSP confirmed in browser DevTools)
- [ ] WAF WebACL attached; OWASP Common Rule Set active
- [ ] Jenkins `Deploy Frontend` stage deploys to S3 and invalidates CloudFront successfully
- [ ] CloudWatch Alarm created: CloudFront 5xx rate > 1% → SNS alert
- [ ] ACM certificate valid and attached (no SSL warnings in browser)
- [ ] Acceptance criteria signed off by PO

---

## Dependencies

- **ATT-P09** (API Gateway + VPC Link) must be deployed — CloudFront needs the API Gateway custom domain as the `/api/*` origin.
- ACM certificate for `app.oms.yourorg.com` must be issued in **`us-east-1`** (CloudFront requirement) before distribution can be created.
- WAF WebACL scope `CLOUDFRONT` must be provisioned in `us-east-1`.
- React SPA build process must output to a `build/` directory with hashed asset filenames for correct cache behaviour.
