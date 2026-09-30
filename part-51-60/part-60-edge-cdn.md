# Part 60: Edge Computing และ CDN Deployment

## บทนำ

Edge Computing และ CDN (Content Delivery Network) เป็นเทคโนโลยีที่นำ computation และ content ให้ใกล้กับ users มากที่สุด แทนที่จะรวมศูนย์ไว้ที่ data center เดียว

ในโลก CI/CD สมัยใหม่ การ deploy ไปยัง edge locations ต้องมีกระบวนการที่ automated และ reliable เพราะมีหลาย locations ที่ต้องจัดการ

บทนี้จะครอบคลุม:
- CDN Concepts
- Cloudflare Workers/Pages
- AWS CloudFront
- Edge Deployments
- Cache Invalidation ใน CI/CD
- Workshop และ Exercises

---

## 1. CDN และ Edge Computing Concepts

### 1.1 CDN ทำงานอย่างไร?

```
Without CDN:
User (Bangkok) ──────────────────────────→ Origin Server (New York)
                    ~200ms round trip

With CDN:
User (Bangkok) ──→ CDN PoP (Singapore) ──→ Origin (New York)
                    ~20ms cache hit          (only on cache miss)

CDN Points of Presence (PoPs):
Cloudflare: 310+ cities
AWS CloudFront: 600+ edge locations
Fastly: 80+ PoPs
```

### 1.2 Edge vs CDN

```
CDN (Content Delivery Network):
├── Cache static assets
├── Reduce origin load
├── Faster content delivery
└── DDoS protection

Edge Computing:
├── Run code at edge
├── Real-time personalization
├── A/B testing at edge
├── Auth/Rate limiting at edge
└── Geolocation-based routing
```

---

## 2. Cloudflare Workers

### 2.1 Cloudflare Workers คืออะไร?

Cloudflare Workers คือ serverless platform ที่รัน JavaScript/TypeScript/WebAssembly ที่ edge locations ของ Cloudflare (310+ cities)

```javascript
// worker.js — ตัวอย่าง Worker อย่างง่าย

export default {
  async fetch(request, env, ctx) {
    const url = new URL(request.url);
    
    // Routing
    if (url.pathname.startsWith('/api/')) {
      return handleAPI(request, env);
    }
    
    if (url.pathname.startsWith('/static/')) {
      return handleStatic(request, env);
    }
    
    // Serve from origin
    return fetch(request);
  }
};

async function handleAPI(request, env) {
  // Get from cache
  const cache = caches.default;
  const cacheKey = new Request(request.url, request);
  let response = await cache.match(cacheKey);
  
  if (!response) {
    // Fetch from origin
    response = await fetch(request);
    
    // Clone สำหรับ cache (response body can only be consumed once)
    const clonedResponse = response.clone();
    
    // Cache 5 นาที
    const headers = new Headers(clonedResponse.headers);
    headers.set('Cache-Control', 'public, max-age=300');
    
    ctx.waitUntil(
      cache.put(cacheKey, new Response(clonedResponse.body, {
        status: clonedResponse.status,
        headers: headers
      }))
    );
  }
  
  return response;
}
```

### 2.2 Wrangler Configuration

```toml
# wrangler.toml
name = "myapp-worker"
main = "src/worker.ts"
compatibility_date = "2024-01-01"

# Environment variables
[vars]
ENVIRONMENT = "production"
API_VERSION = "v2"

# Secrets (จะถูก set ด้วย wrangler secret put)
# DB_URL, JWT_SECRET

# KV Namespaces
[[kv_namespaces]]
binding = "CACHE"
id = "abc123def456"

# R2 Storage
[[r2_buckets]]
binding = "ASSETS"
bucket_name = "myapp-assets"

# Durable Objects
[[durable_objects.bindings]]
name = "RATE_LIMITER"
class_name = "RateLimiter"

# Routes
[[routes]]
pattern = "api.example.com/*"
zone_name = "example.com"

# Environments
[env.staging]
name = "myapp-worker-staging"
vars = { ENVIRONMENT = "staging" }

[[env.staging.routes]]
pattern = "staging-api.example.com/*"
zone_name = "example.com"
```

### 2.3 Advanced Worker Features

```typescript
// src/worker.ts
// Advanced Cloudflare Worker with multiple features

interface Env {
  CACHE: KVNamespace;
  ASSETS: R2Bucket;
  RATE_LIMITER: DurableObjectNamespace;
  ORIGIN_URL: string;
  JWT_SECRET: string;
}

export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext): Promise<Response> {
    const url = new URL(request.url);
    
    // Rate Limiting
    const rateLimitResult = await checkRateLimit(request, env);
    if (!rateLimitResult.allowed) {
      return new Response('Too Many Requests', { status: 429 });
    }
    
    // Geolocation-based routing
    const country = request.cf?.country as string;
    if (country === 'CN') {
      // Redirect ไป China-specific endpoint
      return Response.redirect(`https://cn.example.com${url.pathname}`, 302);
    }
    
    // A/B Testing
    const variant = getABVariant(request);
    
    // Authentication
    if (url.pathname.startsWith('/api/protected')) {
      const authResult = await verifyJWT(request, env.JWT_SECRET);
      if (!authResult.valid) {
        return new Response('Unauthorized', { status: 401 });
      }
    }
    
    // Route to origin with modifications
    const modifiedRequest = new Request(request, {
      headers: {
        ...Object.fromEntries(request.headers),
        'X-AB-Variant': variant,
        'X-Country': country || 'unknown',
        'X-Worker-Version': '1.0.0',
        'X-CF-Ray': request.headers.get('CF-Ray') || '',
      }
    });
    
    // Cache API responses
    const cacheResponse = await serveFromCache(modifiedRequest, env, ctx);
    if (cacheResponse) return cacheResponse;
    
    // Forward to origin
    const originResponse = await fetch(`${env.ORIGIN_URL}${url.pathname}${url.search}`, {
      method: request.method,
      headers: modifiedRequest.headers,
      body: request.body,
    });
    
    // Cache successful responses
    if (originResponse.ok && request.method === 'GET') {
      const responseToCache = originResponse.clone();
      ctx.waitUntil(cacheResponse_put(modifiedRequest, responseToCache, env));
    }
    
    return originResponse;
  }
};

// Rate Limiter (Durable Object)
export class RateLimiter {
  private state: DurableObjectState;
  
  constructor(state: DurableObjectState) {
    this.state = state;
  }
  
  async fetch(request: Request): Promise<Response> {
    const key = new URL(request.url).searchParams.get('key') || 'default';
    const windowMs = 60000; // 1 นาที
    const maxRequests = 100;
    
    const now = Date.now();
    const windowStart = now - windowMs;
    
    // Get current count
    const stored = await this.state.storage.get<number[]>(key) || [];
    
    // Filter ให้เฉพาะ requests ใน window ปัจจุบัน
    const recentRequests = stored.filter(time => time > windowStart);
    
    if (recentRequests.length >= maxRequests) {
      return new Response(JSON.stringify({ allowed: false, remaining: 0 }), {
        status: 429,
        headers: { 'Content-Type': 'application/json' }
      });
    }
    
    // บันทึก request ใหม่
    recentRequests.push(now);
    await this.state.storage.put(key, recentRequests);
    
    return new Response(JSON.stringify({
      allowed: true,
      remaining: maxRequests - recentRequests.length
    }), {
      headers: { 'Content-Type': 'application/json' }
    });
  }
}

async function checkRateLimit(request: Request, env: Env): Promise<{ allowed: boolean }> {
  const clientIP = request.headers.get('CF-Connecting-IP') || 'unknown';
  const id = env.RATE_LIMITER.idFromName(clientIP);
  const rateLimiter = env.RATE_LIMITER.get(id);
  
  const response = await rateLimiter.fetch(`https://rate-limiter/?key=${clientIP}`);
  const result = await response.json() as { allowed: boolean };
  return result;
}

function getABVariant(request: Request): string {
  // ใช้ cookie ถ้ามี
  const cookie = request.headers.get('Cookie');
  if (cookie?.includes('ab_variant=')) {
    return cookie.split('ab_variant=')[1].split(';')[0];
  }
  
  // Random assignment
  const clientIP = request.headers.get('CF-Connecting-IP') || '';
  const hash = simpleHash(clientIP);
  return hash % 2 === 0 ? 'A' : 'B';
}

function simpleHash(str: string): number {
  let hash = 0;
  for (let i = 0; i < str.length; i++) {
    hash = ((hash << 5) - hash) + str.charCodeAt(i);
    hash |= 0;
  }
  return Math.abs(hash);
}

async function verifyJWT(request: Request, secret: string): Promise<{ valid: boolean }> {
  const authHeader = request.headers.get('Authorization');
  if (!authHeader?.startsWith('Bearer ')) {
    return { valid: false };
  }
  
  const token = authHeader.substring(7);
  
  try {
    // Verify JWT signature
    const encoder = new TextEncoder();
    const keyData = encoder.encode(secret);
    const key = await crypto.subtle.importKey(
      'raw', keyData,
      { name: 'HMAC', hash: 'SHA-256' },
      false, ['verify']
    );
    
    const [header, payload, signature] = token.split('.');
    const data = encoder.encode(`${header}.${payload}`);
    const sig = Uint8Array.from(atob(signature.replace(/-/g, '+').replace(/_/g, '/')), c => c.charCodeAt(0));
    
    const valid = await crypto.subtle.verify('HMAC', key, sig, data);
    
    if (valid) {
      // ตรวจสอบ expiry
      const payloadData = JSON.parse(atob(payload));
      if (payloadData.exp && payloadData.exp < Date.now() / 1000) {
        return { valid: false };
      }
    }
    
    return { valid };
  } catch {
    return { valid: false };
  }
}

async function serveFromCache(
  request: Request,
  env: Env,
  ctx: ExecutionContext
): Promise<Response | null> {
  if (request.method !== 'GET') return null;
  
  const cache = caches.default;
  const response = await cache.match(request);
  return response || null;
}

async function cacheResponse_put(
  request: Request,
  response: Response,
  env: Env
): Promise<void> {
  const cache = caches.default;
  
  // Cache 5 นาทีสำหรับ API responses
  const headers = new Headers(response.headers);
  headers.set('Cache-Control', 'public, max-age=300');
  
  await cache.put(request, new Response(response.body, {
    status: response.status,
    headers
  }));
}
```

---

## 3. Deploy Cloudflare Workers ใน CI/CD

### 3.1 GitHub Actions Pipeline สำหรับ Workers

```yaml
# .github/workflows/deploy-cloudflare.yml
name: Deploy Cloudflare Workers

on:
  push:
    branches: [main, staging]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install Dependencies
        run: npm ci

      - name: Run Tests
        run: npm test

      - name: Type Check
        run: npm run type-check

      - name: Lint
        run: npm run lint

  deploy-staging:
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/staging'
    environment:
      name: staging
      url: https://staging-api.example.com

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install Dependencies
        run: npm ci

      - name: Build Worker
        run: npm run build

      - name: Deploy to Staging
        run: |
          npx wrangler deploy \
            --env staging \
            --compatibility-date $(date +%Y-%m-%d)
        env:
          CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          CLOUDFLARE_ACCOUNT_ID: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}

      - name: Run Smoke Tests
        run: |
          sleep 10  # รอให้ deploy propagate
          
          # Test staging endpoint
          STATUS=$(curl -sS -o /dev/null -w "%{http_code}" https://staging-api.example.com/health)
          
          if [ "${STATUS}" != "200" ]; then
            echo "❌ Smoke test failed: ${STATUS}"
            exit 1
          fi
          
          echo "✅ Smoke tests passed"

  deploy-production:
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://api.example.com

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install Dependencies
        run: npm ci

      - name: Build Worker
        run: npm run build

      - name: Deploy with Canary
        run: |
          # Deploy ใน staged rollout mode
          npx wrangler deploy \
            --env production \
            --compatibility-date $(date +%Y-%m-%d)
        env:
          CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          CLOUDFLARE_ACCOUNT_ID: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}

      - name: Run Production Smoke Tests
        run: |
          sleep 30  # รอให้ propagate ไปทุก edge locations
          
          ENDPOINTS=(
            "https://api.example.com/health"
            "https://api.example.com/api/v2/status"
          )
          
          for endpoint in "${ENDPOINTS[@]}"; do
            STATUS=$(curl -sS -o /dev/null -w "%{http_code}" "${endpoint}")
            if [ "${STATUS}" != "200" ]; then
              echo "❌ ${endpoint}: ${STATUS}"
              exit 1
            fi
            echo "✅ ${endpoint}: ${STATUS}"
          done
```

---

## 4. Cloudflare Pages

### 4.1 Deploy Static Site

```yaml
# .github/workflows/cloudflare-pages.yml
name: Deploy to Cloudflare Pages

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      deployments: write  # สำหรับ GitHub Deployments API

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install Dependencies
        run: npm ci

      - name: Build
        run: npm run build
        env:
          NODE_ENV: production
          NEXT_PUBLIC_API_URL: ${{ github.ref == 'refs/heads/main' && 'https://api.example.com' || 'https://staging-api.example.com' }}

      - name: Deploy to Cloudflare Pages
        uses: cloudflare/pages-action@v1
        with:
          apiToken: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          accountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
          projectName: myapp
          directory: out  # Next.js static export directory
          # Branch deployments
          gitHubToken: ${{ secrets.GITHUB_TOKEN }}
          # Preview สำหรับ PRs, Production สำหรับ main
          branch: ${{ github.head_ref || github.ref_name }}
```

---

## 5. AWS CloudFront

### 5.1 CloudFront Distribution

```hcl
# terraform/cloudfront/main.tf

resource "aws_cloudfront_distribution" "main" {
  enabled             = true
  is_ipv6_enabled     = true
  http_version        = "http2and3"
  price_class         = "PriceClass_All"
  aliases             = ["www.example.com", "example.com"]
  
  # S3 Origin สำหรับ static assets
  origin {
    domain_name              = aws_s3_bucket.static.bucket_regional_domain_name
    origin_id                = "S3-static"
    origin_access_control_id = aws_cloudfront_origin_access_control.s3.id
  }
  
  # ALB Origin สำหรับ API
  origin {
    domain_name = aws_lb.api.dns_name
    origin_id   = "ALB-api"
    
    custom_origin_config {
      http_port              = 80
      https_port             = 443
      origin_protocol_policy = "https-only"
      origin_ssl_protocols   = ["TLSv1.2"]
    }
    
    # Custom headers สำหรับ security
    custom_header {
      name  = "X-Origin-Verify"
      value = var.origin_verify_secret
    }
  }
  
  # Default cache behavior สำหรับ static assets
  default_cache_behavior {
    allowed_methods  = ["GET", "HEAD", "OPTIONS"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "S3-static"
    
    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }
    
    viewer_protocol_policy = "redirect-to-https"
    min_ttl                = 0
    default_ttl            = 86400     # 1 วัน
    max_ttl                = 31536000  # 1 ปี
    compress               = true
    
    # Lambda@Edge สำหรับ security headers
    lambda_function_association {
      event_type   = "viewer-response"
      lambda_arn   = aws_lambda_function.security_headers.qualified_arn
      include_body = false
    }
  }
  
  # Cache behavior สำหรับ API
  ordered_cache_behavior {
    path_pattern     = "/api/*"
    allowed_methods  = ["DELETE", "GET", "HEAD", "OPTIONS", "PATCH", "POST", "PUT"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "ALB-api"
    
    forwarded_values {
      query_string = true
      headers      = ["Authorization", "Accept", "Accept-Encoding"]
      
      cookies {
        forward = "all"
      }
    }
    
    viewer_protocol_policy = "https-only"
    min_ttl                = 0
    default_ttl            = 0  # ไม่ cache API responses โดย default
    max_ttl                = 0
    compress               = true
  }
  
  # Cache behavior สำหรับ static assets ที่มี hash (immutable)
  ordered_cache_behavior {
    path_pattern     = "/_next/static/*"
    allowed_methods  = ["GET", "HEAD"]
    cached_methods   = ["GET", "HEAD"]
    target_origin_id = "S3-static"
    
    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }
    
    viewer_protocol_policy = "redirect-to-https"
    min_ttl                = 31536000  # 1 ปี (immutable)
    default_ttl            = 31536000
    max_ttl                = 31536000
    compress               = true
  }
  
  # Geographic restrictions
  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }
  
  # SSL Certificate
  viewer_certificate {
    acm_certificate_arn      = aws_acm_certificate.main.arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }
  
  # Web Application Firewall
  web_acl_id = aws_wafv2_web_acl.main.arn
  
  tags = {
    Environment = var.environment
    Project     = var.project_name
  }
}
```

### 5.2 Cache Invalidation ใน CI/CD

```yaml
# .github/workflows/deploy-with-cdn.yml
name: Deploy with CloudFront Cache Invalidation

on:
  push:
    branches: [main]

jobs:
  deploy-and-invalidate:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/DeployRole
          aws-region: ap-southeast-1

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - name: Install and Build
        run: |
          npm ci
          npm run build

      - name: Deploy Static Assets to S3
        run: |
          # Deploy immutable assets (hashed filenames)
          # ใช้ long cache headers
          aws s3 sync \
            out/_next/static/ \
            s3://myapp-assets/_next/static/ \
            --cache-control "public, max-age=31536000, immutable" \
            --delete
          
          # Deploy non-hashed files (index.html, etc.)
          # ใช้ short cache headers
          aws s3 sync \
            out/ \
            s3://myapp-assets/ \
            --exclude "_next/static/*" \
            --cache-control "public, max-age=0, must-revalidate" \
            --delete

      - name: Invalidate CloudFront Cache
        run: |
          # Invalidate เฉพาะ non-hashed files
          INVALIDATION_ID=$(aws cloudfront create-invalidation \
            --distribution-id ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }} \
            --paths \
              "/*" \
              "/index.html" \
              "/api/*" \
            --query 'Invalidation.Id' \
            --output text)
          
          echo "Invalidation ID: ${INVALIDATION_ID}"
          
          # รอให้ invalidation เสร็จ
          echo "Waiting for invalidation to complete..."
          aws cloudfront wait invalidation-completed \
            --distribution-id ${{ secrets.CLOUDFRONT_DISTRIBUTION_ID }} \
            --id "${INVALIDATION_ID}"
          
          echo "✅ Cache invalidation complete"

      - name: Verify Deployment
        run: |
          # ตรวจสอบว่า assets deploy ถูกต้อง
          
          # Check index.html
          RESPONSE=$(curl -sS -D - https://www.example.com/ 2>&1)
          
          # ตรวจสอบ cache headers
          CACHE_CONTROL=$(echo "${RESPONSE}" | grep -i "cache-control:" | head -1)
          echo "Cache-Control: ${CACHE_CONTROL}"
          
          # ตรวจสอบว่า X-Cache แสดง MISS (fresh content)
          X_CACHE=$(echo "${RESPONSE}" | grep -i "x-cache:" | head -1)
          echo "X-Cache: ${X_CACHE}"
          
          HTTP_STATUS=$(echo "${RESPONSE}" | head -1 | grep -oP '\d{3}')
          
          if [ "${HTTP_STATUS}" != "200" ]; then
            echo "❌ Unexpected status: ${HTTP_STATUS}"
            exit 1
          fi
          
          echo "✅ Deployment verified"
```

---

## 6. Edge Functions และ Lambda@Edge

### 6.1 Lambda@Edge สำหรับ Security Headers

```javascript
// lambda/security-headers/index.js
// Lambda@Edge function เพิ่ม security headers ทุก response

'use strict';

exports.handler = (event, context, callback) => {
  const response = event.Records[0].cf.response;
  
  // Security Headers
  const headers = response.headers;
  
  // Strict Transport Security
  headers['strict-transport-security'] = [{
    key: 'Strict-Transport-Security',
    value: 'max-age=63072000; includeSubDomains; preload'
  }];
  
  // Content Security Policy
  headers['content-security-policy'] = [{
    key: 'Content-Security-Policy',
    value: [
      "default-src 'self'",
      "script-src 'self' 'unsafe-inline' https://cdn.jsdelivr.net",
      "style-src 'self' 'unsafe-inline' https://fonts.googleapis.com",
      "font-src 'self' https://fonts.gstatic.com",
      "img-src 'self' data: https:",
      "connect-src 'self' https://api.example.com",
      "frame-ancestors 'none'",
      "base-uri 'self'",
      "form-action 'self'"
    ].join('; ')
  }];
  
  // Clickjacking protection
  headers['x-frame-options'] = [{
    key: 'X-Frame-Options',
    value: 'DENY'
  }];
  
  // MIME sniffing protection
  headers['x-content-type-options'] = [{
    key: 'X-Content-Type-Options',
    value: 'nosniff'
  }];
  
  // XSS Protection
  headers['x-xss-protection'] = [{
    key: 'X-XSS-Protection',
    value: '1; mode=block'
  }];
  
  // Referrer Policy
  headers['referrer-policy'] = [{
    key: 'Referrer-Policy',
    value: 'strict-origin-when-cross-origin'
  }];
  
  // Permissions Policy
  headers['permissions-policy'] = [{
    key: 'Permissions-Policy',
    value: 'camera=(), microphone=(), geolocation=(self), interest-cohort=()'
  }];
  
  callback(null, response);
};
```

---

## 7. Edge Caching Strategies

### 7.1 Cache Strategy Configuration

```javascript
// cloudflare/cache-strategy.js
// กำหนด cache rules ตาม content type

const CACHE_RULES = [
  // Static assets (hashed) — cache forever
  {
    pattern: /\/_next\/static\//,
    maxAge: 365 * 24 * 60 * 60,  // 1 ปี
    cacheControl: 'public, max-age=31536000, immutable'
  },
  
  // Images
  {
    pattern: /\.(jpg|jpeg|png|gif|webp|avif|svg|ico)$/,
    maxAge: 30 * 24 * 60 * 60,  // 30 วัน
    cacheControl: 'public, max-age=2592000, stale-while-revalidate=86400'
  },
  
  // Fonts
  {
    pattern: /\.(woff|woff2|ttf|eot)$/,
    maxAge: 365 * 24 * 60 * 60,
    cacheControl: 'public, max-age=31536000, immutable'
  },
  
  // API responses — cache short term
  {
    pattern: /^\/api\/public\//,
    maxAge: 5 * 60,  // 5 นาที
    cacheControl: 'public, max-age=300, stale-while-revalidate=60'
  },
  
  // API responses ที่มี auth — ไม่ cache
  {
    pattern: /^\/api\//,
    maxAge: 0,
    cacheControl: 'no-store, no-cache'
  },
  
  // HTML pages
  {
    pattern: /\.html?$|\/$/,
    maxAge: 0,
    cacheControl: 'public, max-age=0, must-revalidate'
  }
];

function getCacheRule(path) {
  for (const rule of CACHE_RULES) {
    if (rule.pattern.test(path)) {
      return rule;
    }
  }
  return { maxAge: 3600, cacheControl: 'public, max-age=3600' };
}

// Cloudflare Worker middleware
export async function withCaching(request, env, ctx, next) {
  const url = new URL(request.url);
  const cacheRule = getCacheRule(url.pathname);
  
  // ไม่ cache requests ที่มี Authorization header
  if (request.headers.has('Authorization')) {
    return next(request);
  }
  
  // ตรวจสอบ cache
  const cache = caches.default;
  const cacheKey = new Request(url.toString(), {
    method: request.method,
    headers: {
      'Accept-Encoding': request.headers.get('Accept-Encoding') || ''
    }
  });
  
  if (cacheRule.maxAge > 0) {
    const cachedResponse = await cache.match(cacheKey);
    if (cachedResponse) {
      // เพิ่ม header บอกว่ามาจาก cache
      const response = new Response(cachedResponse.body, cachedResponse);
      response.headers.set('X-Cache', 'HIT');
      return response;
    }
  }
  
  // Fetch จาก origin
  const response = await next(request);
  
  // Cache response ถ้า maxAge > 0
  if (cacheRule.maxAge > 0 && response.ok) {
    const responseToCache = new Response(response.clone().body, {
      status: response.status,
      headers: {
        ...Object.fromEntries(response.headers),
        'Cache-Control': cacheRule.cacheControl,
        'X-Cache': 'MISS'
      }
    });
    
    ctx.waitUntil(cache.put(cacheKey, responseToCache));
  }
  
  return response;
}
```

### 7.2 Cache Invalidation Strategy

```python
# scripts/cdn_cache_invalidation.py
"""
Automated cache invalidation สำหรับหลาย CDN providers
"""
import os
import json
import hashlib
import urllib.request
import urllib.error
from typing import List, Optional


class CloudflareCacheManager:
    """จัดการ Cloudflare cache"""
    
    def __init__(self, api_token: str, zone_id: str):
        self.api_token = api_token
        self.zone_id = zone_id
        self.base_url = "https://api.cloudflare.com/client/v4"
    
    def _make_request(self, method: str, path: str, data: dict = None) -> dict:
        """ทำ API request"""
        url = f"{self.base_url}{path}"
        body = json.dumps(data).encode() if data else None
        
        req = urllib.request.Request(
            url,
            data=body,
            headers={
                'Authorization': f'Bearer {self.api_token}',
                'Content-Type': 'application/json'
            },
            method=method
        )
        
        with urllib.request.urlopen(req, timeout=30) as resp:
            return json.loads(resp.read())
    
    def purge_all(self) -> bool:
        """Purge ทุก cache"""
        result = self._make_request(
            'POST',
            f'/zones/{self.zone_id}/purge_cache',
            {'purge_everything': True}
        )
        return result.get('success', False)
    
    def purge_by_url(self, urls: List[str]) -> bool:
        """Purge specific URLs"""
        # Cloudflare จำกัด 30 URLs ต่อ request
        chunks = [urls[i:i+30] for i in range(0, len(urls), 30)]
        
        all_success = True
        for chunk in chunks:
            result = self._make_request(
                'POST',
                f'/zones/{self.zone_id}/purge_cache',
                {'files': chunk}
            )
            if not result.get('success', False):
                all_success = False
        
        return all_success
    
    def purge_by_tag(self, tags: List[str]) -> bool:
        """Purge โดย cache tag"""
        result = self._make_request(
            'POST',
            f'/zones/{self.zone_id}/purge_cache',
            {'tags': tags}
        )
        return result.get('success', False)


class AWSCloudFrontManager:
    """จัดการ AWS CloudFront cache"""
    
    def __init__(self, distribution_id: str):
        self.distribution_id = distribution_id
    
    def create_invalidation(self, paths: List[str]) -> str:
        """สร้าง CloudFront invalidation"""
        import subprocess
        
        paths_json = json.dumps({'Quantity': len(paths), 'Items': paths})
        caller_ref = hashlib.md5(json.dumps(paths).encode()).hexdigest()
        
        result = subprocess.run(
            ['aws', 'cloudfront', 'create-invalidation',
             '--distribution-id', self.distribution_id,
             '--invalidation-batch',
             f'{{"Paths": {paths_json}, "CallerReference": "{caller_ref}"}}'],
            capture_output=True, text=True
        )
        
        if result.returncode != 0:
            raise Exception(f"Invalidation failed: {result.stderr}")
        
        data = json.loads(result.stdout)
        return data['Invalidation']['Id']
    
    def wait_for_invalidation(self, invalidation_id: str) -> None:
        """รอให้ invalidation เสร็จ"""
        import subprocess
        
        subprocess.run(
            ['aws', 'cloudfront', 'wait', 'invalidation-completed',
             '--distribution-id', self.distribution_id,
             '--id', invalidation_id],
            check=True
        )


def invalidate_all_cdns(
    changed_files: List[str],
    cloudflare_token: Optional[str] = None,
    cloudflare_zone: Optional[str] = None,
    cloudfront_distribution: Optional[str] = None,
    base_url: str = "https://www.example.com"
) -> dict:
    """
    Invalidate cache ใน CDN ทุกตัวที่กำหนด
    
    Args:
        changed_files: รายการ files ที่เปลี่ยนแปลง
        cloudflare_token: Cloudflare API token
        cloudflare_zone: Cloudflare zone ID
        cloudfront_distribution: CloudFront distribution ID
        base_url: Base URL สำหรับ construct full URLs
    
    Returns:
        dict with results for each CDN
    """
    results = {}
    
    # สร้าง URLs จาก paths
    urls = [f"{base_url}/{f.lstrip('/')}" for f in changed_files]
    
    # สำหรับ non-hashed files เพิ่ม paths พิเศษ
    paths_to_invalidate = list(set(
        [f"/{f.lstrip('/')}" for f in changed_files] +
        ['/index.html', '/', '/sitemap.xml']
    ))
    
    # Cloudflare
    if cloudflare_token and cloudflare_zone:
        print(f"Invalidating Cloudflare cache for {len(urls)} URLs...")
        cf = CloudflareCacheManager(cloudflare_token, cloudflare_zone)
        
        success = cf.purge_by_url(urls)
        results['cloudflare'] = 'success' if success else 'failed'
        print(f"Cloudflare: {results['cloudflare']}")
    
    # CloudFront
    if cloudfront_distribution:
        print(f"Invalidating CloudFront for {len(paths_to_invalidate)} paths...")
        cf_manager = AWSCloudFrontManager(cloudfront_distribution)
        
        invalidation_id = cf_manager.create_invalidation(paths_to_invalidate)
        print(f"CloudFront invalidation ID: {invalidation_id}")
        
        # รอให้เสร็จ
        cf_manager.wait_for_invalidation(invalidation_id)
        results['cloudfront'] = 'success'
        print(f"CloudFront: success")
    
    return results


if __name__ == '__main__':
    import sys
    
    # อ่านรายการ changed files จาก git
    import subprocess
    changed = subprocess.run(
        ['git', 'diff', '--name-only', 'HEAD~1', 'HEAD'],
        capture_output=True, text=True
    ).stdout.strip().split('\n')
    
    results = invalidate_all_cdns(
        changed_files=changed,
        cloudflare_token=os.getenv('CLOUDFLARE_API_TOKEN'),
        cloudflare_zone=os.getenv('CLOUDFLARE_ZONE_ID'),
        cloudfront_distribution=os.getenv('CLOUDFRONT_DISTRIBUTION_ID')
    )
    
    print(f"\nInvalidation Results: {results}")
    
    if any(v == 'failed' for v in results.values()):
        sys.exit(1)
```

---

## 8. Workshop: CDN Deployment Lab

### Lab 1: Deploy Static Site ไปยัง Cloudflare Pages

```bash
#!/bin/bash
# workshop/lab1-cloudflare-pages.sh

echo "=== Lab 1: Cloudflare Pages Deployment ==="

# Prerequisites
command -v node >/dev/null || { echo "Install Node.js first"; exit 1; }
command -v npx >/dev/null || { echo "Install npm first"; exit 1; }

# 1. สร้าง demo static site
mkdir -p /tmp/cf-pages-lab/public
cd /tmp/cf-pages-lab

cat > public/index.html << 'EOF'
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <title>Edge Computing Lab</title>
  <style>
    body { font-family: sans-serif; max-width: 800px; margin: 50px auto; }
    .info { background: #f0f8ff; padding: 20px; border-radius: 8px; }
  </style>
</head>
<body>
  <h1>🌐 Edge Computing Lab</h1>
  <div class="info">
    <p>Version: <strong id="version">1.0.0</strong></p>
    <p>Build Time: <strong id="build-time"></strong></p>
    <p>Your location: <strong id="location">Loading...</strong></p>
  </div>
  <script>
    document.getElementById('build-time').textContent = new Date().toISOString();
    fetch('https://ipapi.co/json/')
      .then(r => r.json())
      .then(d => {
        document.getElementById('location').textContent = 
          `${d.city}, ${d.country_name}`;
      });
  </script>
</body>
</html>
EOF

# 2. สร้าง Wrangler config
cat > wrangler.toml << 'EOF'
name = "edge-lab"

[site]
bucket = "./public"
EOF

# 3. Deploy
echo "Deploying to Cloudflare Pages..."
if [ -z "${CLOUDFLARE_API_TOKEN}" ]; then
    echo "⚠️  CLOUDFLARE_API_TOKEN not set"
    echo "Set it with: export CLOUDFLARE_API_TOKEN=your-token"
    echo "Then run this script again"
    exit 0
fi

npx wrangler pages deploy public \
    --project-name edge-lab \
    --branch main

echo "✅ Lab 1 Complete!"
echo "Visit your Cloudflare Pages dashboard to see the deployment"
```

### Lab 2: CloudFront + S3 Static Hosting

```bash
#!/bin/bash
# workshop/lab2-cloudfront.sh

echo "=== Lab 2: CloudFront Static Hosting ==="

# Variables
BUCKET_NAME="cf-lab-$(date +%Y%m%d%H%M%S)"
REGION="ap-southeast-1"

# 1. สร้าง S3 bucket
echo "Creating S3 bucket: ${BUCKET_NAME}"
aws s3 mb "s3://${BUCKET_NAME}" --region "${REGION}"

# 2. ตั้งค่า bucket สำหรับ static website
aws s3 website "s3://${BUCKET_NAME}" \
    --index-document index.html \
    --error-document error.html

# 3. Upload sample content
cat > /tmp/index.html << 'EOF'
<!DOCTYPE html>
<html>
<head><title>CloudFront Lab</title></head>
<body>
  <h1>Hello from CloudFront Edge!</h1>
  <p>Served at: <script>document.write(new Date())</script></p>
</body>
</html>
EOF

aws s3 cp /tmp/index.html "s3://${BUCKET_NAME}/index.html" \
    --content-type "text/html" \
    --cache-control "max-age=300"

# 4. สร้าง CloudFront Origin Access Control
OAC_ID=$(aws cloudfront create-origin-access-control \
    --origin-access-control-config '{
        "Name": "lab-oac",
        "Description": "Lab OAC",
        "SigningProtocol": "sigv4",
        "SigningBehavior": "always",
        "OriginAccessControlOriginType": "s3"
    }' \
    --query 'OriginAccessControl.Id' \
    --output text)

echo "OAC ID: ${OAC_ID}"

echo "✅ Lab 2 Setup Complete!"
echo "Next: Create CloudFront distribution in AWS Console"
echo "S3 Bucket: ${BUCKET_NAME}"
echo "OAC ID: ${OAC_ID}"
```

---

## 9. สรุปและ Best Practices

### Edge Computing Checklist

```markdown
## Edge/CDN Deployment Checklist

### Static Assets
- [ ] Hashed filenames สำหรับ immutable cache
- [ ] Separate cache policies สำหรับ static vs dynamic
- [ ] Brotli/Gzip compression enabled
- [ ] HTTP/3 enabled

### Cache Strategy
- [ ] Document cache rules
- [ ] Implement cache invalidation ใน pipeline
- [ ] Monitor cache hit rates
- [ ] Test cache behavior

### Security
- [ ] Security headers ผ่าน Lambda@Edge หรือ Workers
- [ ] WAF rules ที่ edge
- [ ] DDoS protection
- [ ] Origin authentication (shared secret)

### Performance
- [ ] Pre-warming cache หลัง deploy
- [ ] Monitor edge performance metrics
- [ ] Implement stale-while-revalidate
- [ ] Use CDN for API responses ที่เหมาะสม

### CI/CD
- [ ] Automated deployment ไป edge
- [ ] Cache invalidation automation
- [ ] Smoke tests หลัง edge deployment
- [ ] Rollback procedure
```

---

## 10. Summary: Part 51-60

เราได้เรียนรู้ topics สำคัญในช่วง Part 51-60:

| Part | Topic | Key Concepts |
|------|-------|-------------|
| 51 | CI/CD Security | Least Privilege, OIDC, Secret Scanning |
| 52 | Supply Chain Security | SLSA, SBOM, Sigstore |
| 53 | Code Signing | Cosign, Keyless Signing, Transparency Logs |
| 54 | Zero-Trust | SPIFFE/SPIRE, OIDC, mTLS |
| 55 | Advanced Kubernetes | Operators, CRDs, OPA, Kyverno |
| 56 | Istio Service Mesh | Traffic Management, mTLS, Observability |
| 57 | Chaos Engineering | LitmusChaos, Experiments, Game Days |
| 58 | Disaster Recovery | Velero, RTO/RPO, DR Testing |
| 59 | Multi-Cloud | Terraform, Cross-Cloud, Cost Optimization |
| 60 | Edge/CDN | Cloudflare Workers, CloudFront, Cache Strategy |

---

## อ้างอิง

- [Cloudflare Workers Documentation](https://developers.cloudflare.com/workers/)
- [Cloudflare Pages](https://developers.cloudflare.com/pages/)
- [AWS CloudFront Documentation](https://docs.aws.amazon.com/cloudfront/)
- [Lambda@Edge](https://docs.aws.amazon.com/lambda/latest/dg/lambda-edge.html)
- [Edge Computing - CNCF](https://tag-runtime.cncf.io/wgs/iot-edge/)
- [Web Performance Optimization](https://web.dev/performance/)
