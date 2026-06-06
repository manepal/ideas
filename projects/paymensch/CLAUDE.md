# CLAUDE.md — Paymensch

Payment aggregation platform for Nepal. Single API to accept payments via eSewa, Khalti, Fonepay, and more. Evolving into a Stripe-like billing and subscription platform.

## Skills Required

You MUST use these skills at the appropriate times. Do NOT skip skill checks.

### Always Active
- `superpowers:test-driven-development` — EVERY feature starts with failing tests. No exceptions.
- `superpowers:verification-before-completion` — run tests and confirm output before claiming success
- `superpowers:using-git-worktrees` — isolate feature work, never work on main directly

### Per-Phase
- `superpowers:brainstorming` — before designing any new feature
- `superpowers:writing-plans` — after spec is approved
- `superpowers:requesting-code-review` — before merging any branch
- `security-review` — on all payment code, auth, and API key handling
- `simplify` — after feature complete, before merge
- `impeccable` — on dashboard UI components (reference `docs/BRAND.md`)
- `ui-ux-pro-max` — on dashboard UX decisions (reference `docs/BRAND.md`)
- `graphify` — after every feature commit to update knowledge graph
- **ALWAYS read `docs/BRAND.md` before writing any UI code** — never use raw colors, fonts, or spacing

### When Stuck
- `superpowers:systematic-debugging` — before proposing any fix
- Query `/graphify` — understand existing code before changing it

### Skill Discovery (Ongoing)

**The project's needs will grow faster than this skill list.** New domains (billing, PSP licensing, multi-country compliance, performance profiling, database optimization, incident response) will require new skills that don't exist yet or aren't listed here.

**When to search for new skills:**

| Trigger | Example |
|---------|---------|
| Starting a new domain area | Billing engine, PSP licensing, India expansion |
| Encountering a problem with no obvious existing skill | Database query optimization, Redis memory profiling |
| Before major architectural decisions | "Should we use Kafka instead of BullMQ?" |
| A human says "is there a skill for..." | Just search for it |
| At major version boundaries | Expo SDK 56, Fastify 6, Next.js 16 |

**How to discover and install skills:**

Use `find-skills` to search the skill marketplace. Be specific about what you need:

```bash
# Example invocations as the project grows:
/find-skills "database migration tools for PostgreSQL"
/find-skills "payment gateway integration testing"
/find-skills "Kubernetes deployment automation"
/find-skills "subscription billing engine"
/find-skills "PCI compliance checklist"
```

**Rule:** If you're about to write 200+ lines of code in a domain no existing skill covers, search for a skill first. The skill may already exist and save hours of wheel-reinventing. If no skill is found, proceed without one — but document what you built so a future skill can replace it.

Periodically (every ~5-10 features), review the installed skills against the current project needs. Remove unused skills. Update outdated ones.

## Tech Stack

### Backend (packages/api/)
- **Runtime:** Node.js 22+
- **Framework:** Fastify 5
- **Language:** TypeScript 5.x (strict mode)
- **Database:** PostgreSQL 16 via Knex.js migrations
- **Cache/Queue:** Redis 7 + BullMQ
- **Validation:** Fastify built-in JSON schemas + Zod for complex logic

### Frontend (packages/dashboard/)
- **Framework:** Next.js 15 (App Router)
- **Styling:** Tailwind CSS + shadcn/ui
- **State:** React Query (TanStack Query) for server state
- **Forms:** React Hook Form + Zod

### Shared (packages/shared/)
- **Types:** All TypeScript interfaces shared between API and dashboard
- **Schemas:** Zod schemas for validation on both sides
- **Constants:** Status enums, error codes, currency definitions

### Testing
- **Runner:** Vitest (all packages)
- **DB/Redis:** Testcontainers (real PostgreSQL + Redis, never mocked)
- **API:** fastify.inject() for route tests (no server start needed)
- **Gateway mocking:** nock with real fixture responses
- **Frontend unit:** React Testing Library + MSW
- **E2E:** Playwright
- **Property-based:** fast-check for idempotency and amount math
- **Coverage target:** 100% on gateways/, services/. 90%+ overall.

## Non-Negotiable Rules

### Testing
1. **No mock database.** Every test hits real PostgreSQL + Redis via Testcontainers.
2. **TDD always.** Write the test, watch it fail, then implement.
3. **Gateway contract tests.** Every new gateway adapter passes the shared contract suite.
4. **Every component has 6 states tested:** loading, loaded, empty, error, edge case, submitting.

### Security (Non-Negotiable)

Payment platforms are attack targets from day one. Every item on this list is a hard requirement — no exceptions, no shortcuts, no "we'll add it later."

#### Code-Level Security

1. **Never log API keys, credentials, or signatures.** pino `redact` list covers `authorization`, `x-api-key`, `credentials`, `secret`, `signature`, `token`. Every new sensitive field extends the redact list. Test: `grep` your logs after a test run. If an API key appears, you failed.

2. **Amounts are integers (subunits).** Never `float`, never `double`, never `Number` with decimal points. `1500.50 NPR` = `150050` in the database. Every arithmetic operation is integer math. Test: property-based testing with fast-check verifies round-trip conversion for any valid amount.

3. **Idempotency keys are enforced server-side.** Not client-dependent. Not best-effort. Concurrent requests with the same key arrive at the same Redis lock. First one processes. Second one returns the cached result. Test: fire 10 concurrent requests with the same key, verify exactly 1 database row and 1 gateway call.

4. **All gateway credentials are encrypted at rest.** AES-256-GCM with a key that never appears in code, config files, or `.env` committed to git. Encryption key is injected at runtime via environment variable or KMS. Test: query `gateway_configs` — `credentials` column must be unreadable ciphertext.

5. **Audit log is append-only.** No UPDATE. No DELETE. No TRUNCATE. Database-level enforcement via `REVOKE UPDATE, DELETE, TRUNCATE ON audit_logs FROM app_user`. Test: attempt an UPDATE query — it must fail with a permission error.

6. **Input validation on every endpoint.** Fastify JSON schemas define exact shapes. Unknown fields are stripped. Strings have max lengths. Integers have min/max. Enums have allowed values. No `additionalProperties: true` anywhere. Test: send extra fields, oversized strings, negative amounts — all rejected with 400.

7. **SQL injection prevention.** Never string-interpolate user input into queries. Knex parameterized queries 100% of the time. Test: send `'; DROP TABLE merchants; --` as an email address — it's stored as a string, not executed.

8. **XSS prevention.** Dashboard uses React (auto-escapes by default). API never returns raw HTML. CSP headers restrict script sources. Test: attempt to store `<script>alert(1)</script>` as a merchant name — it renders as text, not code.

#### Infrastructure Security

9. **Helmet headers on all responses.** `X-Content-Type-Options: nosniff`, `X-Frame-Options: DENY`, `X-XSS-Protection: 0`, `Referrer-Policy: strict-origin-when-cross-origin`, `Strict-Transport-Security: max-age=31536000; includeSubDomains`. Fastify `@fastify/helmet` plugin.

10. **CORS locked down.** Only `api.paymensch.io`, `app.paymensch.io`, `paymensch.io`, `docs.paymensch.io`, `admin.paymensch.io` in the allowlist. No wildcard. No `*`. Test: `curl -H "Origin: evil.com"` — response has no `Access-Control-Allow-Origin` header.

11. **Rate limiting per IP and per merchant.** 100 req/min per merchant API key. 20 req/min per IP on login endpoint. 5 req/min on failed login attempts (brute force protection). Redis-backed counters. Test: fire 101 requests — 101st returns 429.

12. **HTTPS only.** Traefik handles TLS termination with Let's Encrypt. HTTP requests are redirected to HTTPS (301). HSTS header set. Test: `curl http://api.paymensch.io` — returns 301 to HTTPS.

13. **Docker images run as non-root.** Every `Dockerfile` has `USER node`. No `USER root`. Minimal base images (Alpine or distroless). No dev tools in production images. Test: `docker exec <container> whoami` — returns `node`, not `root`.

14. **Secrets never in git.** `.env` is `.gitignore`'d. `.env.example` has placeholder values only (never real credentials). CI injects secrets from environment variables or a secrets manager. Pre-commit hook runs `gitleaks` or `trufflehog` to catch accidental secret commits. Test: `gitleaks detect --source .` in CI — fails the build if any secret is found.

#### Runtime Security

15. **JWT best practices.** Short-lived access tokens (15 min). Long-lived refresh tokens (7 days, stored hashed in DB, one per device). Refresh token rotation on use. Admin JWT signing secret is different from merchant JWT signing secret. Test: use a merchant JWT on an admin endpoint — 401.

16. **2FA for admin accounts.** TOTP-based. Required for `super_admin` role. Optional for `support_agent`. Not available for `read_only`. Test: attempt admin login with 2FA enabled but no TOTP code — rejected.

17. **Brute force protection on login.** 5 failed attempts → account locked for 15 minutes. Separate counter per IP and per email. Lock is enforced at the application level (not just Redis rate limit). Test: 6 rapid login attempts with wrong password — 6th returns 429 with "Account temporarily locked" message.

18. **Session management.** Refresh tokens are single-use (rotate on use). Logout invalidates the refresh token. Password change invalidates all refresh tokens for that account. Admin can force-logout all sessions for a merchant. Test: logout → attempt to use old refresh token → 401.

19. **Webhook signature verification.** Every outgoing webhook includes `X-Paymensch-Signature` (HMAC-SHA256 of the payload body). Merchants can verify the payload came from us and wasn't tampered with. Incoming gateway callbacks are verified with the gateway's signature scheme before any state change. Test: POST a webhook with a forged signature — rejected with 401.

20. **Error messages don't leak information.** Production errors return generic messages ("Invalid credentials", not "User not found" or "Password incorrect"). Stack traces never in production responses. Detailed errors only in logs. Test: attempt login with non-existent email — response is identical to wrong-password response.

#### CI/CD Security

21. **Dependency vulnerability scanning.** `npm audit` runs on every PR. Critical/High vulnerabilities block merge. CI runs daily `npm audit` on main and opens issues for new vulnerabilities. Dependabot or Renovate configured for automated patch PRs.

22. **SAST (Static Analysis).** ESLint security plugin (`eslint-plugin-security`). TypeScript strict mode. No `any`, no `as` casts without validation. SonarQube or CodeQL if available in CI.

23. **Container image scanning.** Trivy or Docker Scout scans images for known CVEs before push to registry. Critical CVEs block deploy.

#### Human-Triggered Security Reviews

24. **Full security review before every production deployment.** Run `security-review` skill. Peer reviews all auth code, payment code, credential handling code, and database migration code. No solo merges in these paths.

25. **Penetration test before processing real payments.** Hire an external firm or use a bug bounty platform. Fix all Critical/High findings before going live.

26. **Security.txt at `paymensch.io/.well-known/security.txt`.** Contact for security researchers. PGP key. Acknowledgment policy.

27. **NRB compliance maintained.** Transaction logs retained 5+ years. Audit log is queryable and exportable. Cross-border payments require explicit NRB approval — never route to a foreign gateway without sign-off.

### Code Quality
1. **One file = one responsibility.** If a file exceeds 300 lines, split it.
2. **Pure functions for business logic.** No side effects in game engine, amount math, or validation.
3. **Explicit error types.** Every error path returns a typed error, never a generic string.
4. **No `any`.** TypeScript strict mode. `unknown` if truly uncertain, then narrow.

### Git
1. **Conventional commits.** `feat:`, `fix:`, `test:`, `refactor:`, `docs:`, `chore:`
2. **Never commit to main.** Work in feature branches or worktrees.
3. **Commit after each BUILD_PLAN step.** Frequent, small commits.

## Development Environment

### One-Command Setup

Everything runs in Docker. No manual PostgreSQL installs. No "it works on my machine."

```bash
git clone <repo-url> && cd paymensch && cp .env.example .env && npm run setup
```

That single command:
1. Starts PostgreSQL 16, Redis 7, Prometheus, Grafana, Loki (Docker)
2. Installs all workspace dependencies
3. Runs database migrations
4. Seeds test merchant, sandbox gateways, and demo transactions
5. Starts API on :4000 and Dashboard on :3000 with hot-reload

**The agent must verify this exact sequence works before writing any feature code.** If the human can't run `npm run dev` and see a working setup, the foundation is broken.

### docker-compose.yml (Provided in Repo)

```yaml
version: '3.8'
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: paymensch
      POSTGRES_PASSWORD: paymensch_dev
      POSTGRES_DB: paymensch
    ports:
      - '5432:5432'
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U paymensch']
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports:
      - '6379:6379'
    healthcheck:
      test: ['CMD', 'redis-cli', 'ping']
      interval: 5s
      timeout: 3s
      retries: 5

volumes:
  pgdata:
```

### .env.example (Provided in Repo)

```bash
# Database
DATABASE_URL=postgresql://paymensch:paymensch_dev@localhost:5432/paymensch
DATABASE_TEST_URL=postgresql://paymensch:paymensch_dev@localhost:5432/paymensch_test

# Redis
REDIS_URL=redis://localhost:6379

# API
API_PORT=4000
API_HOST=0.0.0.0
NODE_ENV=development
LOG_LEVEL=debug

# Dashboard
NEXT_PUBLIC_API_URL=http://localhost:4000/v1
DASHBOARD_URL=http://localhost:3000

# Encryption (generate with: node -e "console.log(require('crypto').randomBytes(32).toString('hex'))")
ENCRYPTION_KEY=change_me_generate_with_crypto_randombytes

# Test Gateway Credentials (sandbox only — never commit real ones)
ESEWA_SANDBOX_MERCHANT_CODE=TEST123
KHALTI_SANDBOX_SECRET_KEY=test_secret_key_123
FONEPAY_SANDBOX_MERCHANT_CODE=TESTFM01
FONEPAY_SANDBOX_SECRET=test_secret_456
```

### npm Scripts (in root package.json)

```json
{
  "scripts": {
    "setup": "npm run infra:up && npm install && npm run db:migrate && npm run db:seed && npm run dev",
    "dev": "turbo dev",
    "dev:api": "turbo dev --filter=@paymensch/api",
    "dev:dashboard": "turbo dev --filter=@paymensch/dashboard",
    "dev:docs": "turbo dev --filter=@paymensch/docs",
    "dev:admin": "turbo dev --filter=@paymensch/admin",
    "build": "turbo build",
    "test": "turbo test",
    "test:api": "turbo test --filter=@paymensch/api",
    "test:dashboard": "turbo test --filter=@paymensch/dashboard",
    "test:admin": "turbo test --filter=@paymensch/admin",
    "test:e2e": "playwright test",
    "db:migrate": "turbo db:migrate --filter=@paymensch/api",
    "db:migrate:test": "turbo db:migrate:test --filter=@paymensch/api",
    "db:seed": "turbo db:seed --filter=@paymensch/api",
    "docs:generate": "turbo docs:generate --filter=@paymensch/docs",
    "lint": "turbo lint",
    "typecheck": "turbo typecheck",
    "graphify": "graphify",
    "infra:up": "docker compose up -d",
    "infra:down": "docker compose down",
    "infra:restart": "docker compose down && docker compose up -d",
    "infra:logs": "docker compose logs -f",
    "infra:status": "docker compose ps"
  }
}
```

### Seed Data (db/seeds/development.ts)

The seed script creates a test merchant the human can use immediately:

```typescript
// Running `npm run db:seed` creates:
// 1. A test merchant with id: '00000000-0000-0000-0000-000000000001'
// 2. API keys: sk_test_paymensch_xxxxxxxx (shown in console output)
// 3. Sandbox gateway configs pre-filled with test credentials
// 4. 20 sample transactions across all gateways for dashboard demo
```

After seeding, the human can immediately:
```bash
curl -H "Authorization: Bearer sk_test_paymensch_xxxxxxxx" \
     http://localhost:4000/v1/transactions
```

### Agent Verification Checklist (Run Before Writing Any Code)

Before starting ANY feature work, the agent must verify:

```bash
# 1. Infrastructure is healthy
npm run infra:status            # All services show "healthy"

# 2. Database is reachable
npm run db:migrate              # Runs without errors

# 3. Tests pass (even if zero tests exist, the infrastructure works)
npm test                        # Vitest runs, finds test files, no infra errors

# 4. API starts
npm run dev:api                 # Fastify starts on :4000, no crash

# 5. Dashboard starts
npm run dev:dashboard           # Next.js starts on :3000, no crash

# 6. Seed data loads
npm run db:seed                 # Creates test merchant, prints test API key
```

**If any check fails, fix the infrastructure before writing feature code.** The human should never debug Docker, Postgres, or npm configs.

### Test Database

Tests use a separate database (`paymensch_test`) on the same PostgreSQL instance. Testcontainers is NOT used for local dev — it spins up containers programmatically in CI. For local testing speed, tests hit the existing Postgres in Docker:

```
DATABASE_TEST_URL=postgresql://paymensch:paymensch_dev@localhost:5432/paymensch_test
```

Each test suite:
1. Creates test database if missing
2. Runs migrations on it
3. Wraps in a transaction that rolls back after each test (fast, no cleanup needed)
4. Drops and recreates for integration/E2E tests

This gives Testcontainers-level isolation with local dev speed.

---

## Logging & Monitoring

### Structured Logging (pino)

All logging uses pino (Fastify's native logger). JSON-structured, machine-parseable, human-readable in dev.

```typescript
// packages/api/src/logger.ts
import pino from 'pino';

export const logger = pino({
  level: process.env.LOG_LEVEL || 'info',
  redact: [
    'req.headers.authorization',        // Never log API keys
    'req.headers["x-api-key"]',
    'body.credentials',                  // Gateway credentials
    'body.merchantCode',
    'body.secret',
    'response.redirectUrl',              // May contain tokens
  ],
  serializers: {
    req: (req) => ({
      method: req.method,
      url: req.url,
      headers: { 'content-type': req.headers['content-type'] },
      query: req.query,
    }),
    res: (res) => ({
      statusCode: res.statusCode,
      responseTime: `${res.responseTime}ms`,
    }),
  },
  formatters: {
    level(label) {
      return { level: label };          // No "level": 30 — use strings
    },
  },
});
```

### Log Levels & Usage

| Level | When | Example |
|-------|------|---------|
| `fatal` | Process cannot continue | DB connection lost, Redis down, cannot start server |
| `error` | Something broke, needs attention | Gateway returned 500, webhook verification failed, refund failed |
| `warn` | Unexpected but recoverable | Gateway timeout on first attempt (retrying), rate limit hit, deprecated API version used |
| `info` | Normal business events | Payment initiated, payment completed, refund processed, merchant onboarded |
| `debug` | Detailed flow for investigation | Pipeline stage timings, gateway request/response summaries (redacted), idempotency key hits |
| `trace` | Everything (never in production) | Full request bodies (redacted), raw gateway responses, BullMQ job payloads |

**Rule:** `info` level logs must tell the complete story of every payment lifecycle. Given a transaction ID, you can trace initiate → gateway request → gateway response → callback → verify → complete without `debug` level.

### What Gets Logged

**Every API request (Fastify hook):**
```json
{
  "level": "info",
  "time": "2026-06-06T14:23:45.123Z",
  "msg": "request completed",
  "req": { "method": "POST", "url": "/v1/payments/initiate" },
  "res": { "statusCode": 200, "responseTime": "342ms" },
  "merchantId": "uuid-xxx",
  "transactionId": "uuid-yyy",
  "gateway": "esewa",
  "idempotencyKey": "ik_abc123",
  "amountSubunits": 150000,
  "currency": "NPR"
}
```

**Every gateway call:**
```json
{
  "level": "debug",
  "msg": "gateway request",
  "gateway": "khalti",
  "action": "initiate",
  "durationMs": 487,
  "statusCode": 200,
  "success": true
}
```

**Every error:**
```json
{
  "level": "error",
  "msg": "gateway verification failed",
  "transactionId": "uuid-yyy",
  "gateway": "fonepay",
  "gatewayStatusCode": 502,
  "gatewayErrorMessage": "Bank server unreachable",
  "retryable": true,
  "attempt": 2,
  "stack": "Error: ...\n    at ..."
}
```

### Application Metrics

Exposed via a `/health` endpoint and (in production) Prometheus-compatible `/metrics`:

```typescript
// packages/api/src/metrics.ts
// Exposed on a separate port (9090) for internal monitoring

export const metrics = {
  // Counters
  paymentsInitiated: new Counter({ name: 'paymensch_payments_initiated_total' }),
  paymentsCompleted: new Counter({ name: 'paymensch_payments_completed_total' }),
  paymentsFailed: new Counter({ name: 'paymensch_payments_failed_total' }),
  refundsProcessed: new Counter({ name: 'paymensch_refunds_processed_total' }),

  // Histograms
  gatewayLatency: new Histogram({
    name: 'paymensch_gateway_latency_seconds',
    labelNames: ['gateway', 'action'],
    buckets: [0.1, 0.25, 0.5, 1, 2, 5, 10],
  }),
  requestDuration: new Histogram({
    name: 'paymensch_request_duration_seconds',
    labelNames: ['method', 'route', 'status'],
  }),

  // Gauges
  activeConnections: new Gauge({ name: 'paymensch_active_connections' }),
  queueDepth: new Gauge({ name: 'paymensch_queue_depth', labelNames: ['queue'] }),
};
```

### Health Check Endpoint

```
GET /health
→ { "status": "ok", "uptime": 48239, "postgres": "ok", "redis": "ok" }
→ 200 if all checks pass
→ 503 if any dependency is down

GET /health/ready
→ 200 if server is accepting requests (used for k8s readiness)

GET /health/live
→ 200 if process is alive (used for k8s liveness)
```

### Audit Logging (Separate from Operational Logs)

Audit logs are stored in the database (append-only `audit_logs` table), NOT in the log stream. This keeps compliance data separate from operational noise.

```typescript
// packages/api/src/services/audit.ts
export async function auditLog(params: {
  entityType: 'merchant' | 'transaction' | 'api_key' | 'gateway_config' | 'refund' | 'settings';
  entityId: string;
  action: 'created' | 'updated' | 'deleted' | 'accessed' | 'executed';
  merchantId: string;
  actor: string;          // 'system' | 'merchant:{id}' | 'admin:{id}' | 'gateway:{name}'
  changes?: { before: unknown; after: unknown };
  ipAddress?: string;
}) { ... }
```

Called inline within service functions — NOT a middleware. Auditing is business logic, not infrastructure.

### Agent Rules for Logging

1. **Every new route logs at `info` level** with `merchantId`, `transactionId`, and outcome
2. **Every new gateway adapter logs at `debug` level** with duration, status, and redacted payload
3. **Every error logs with `retryable` flag** so operations can distinguish transient from fatal
4. **Never log credentials** — pino `redact` list is updated when new sensitive fields are added
5. **Audit log entries are created synchronously** within the transaction — if the transaction rolls back, the audit entry rolls back too

### Monitoring Infrastructure

#### Stack (Self-Hosted, Budget-Conscious)

Everything runs on the same VPS as the app for v1. No SaaS monitoring bills until revenue justifies it.

| Component | Tool | Purpose | Port |
|-----------|------|---------|------|
| **Metrics** | prom-client (npm) | Expose `/metrics` endpoint from the API | 9090 |
| **Collection** | Prometheus | Scrape metrics every 15s, store time-series | 9090 |
| **Dashboards** | Grafana | Visualize metrics, no-code dashboards | 3001 |
| **Logs** | Loki + promtail | Aggregate and search structured logs | 3100 |
| **Alerting** | Grafana Alertmanager | Alert on SLO breaches, error spikes | (built into Grafana) |
| **Uptime** | UptimeRobot (free tier) | External HTTP health check every 5 min | SaaS |
| **Errors** | Sentry (free tier) | Exception tracking with stack traces | SaaS |

#### docker-compose.yml Additions

```yaml
# Monitoring stack — added to docker-compose.yml
prometheus:
  image: prom/prometheus:latest
  volumes:
    - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
    - prometheus_data:/prometheus
  ports:
    - '9090:9090'
  command:
    - '--config.file=/etc/prometheus/prometheus.yml'
    - '--storage.tsdb.retention.time=30d'

grafana:
  image: grafana/grafana:latest
  environment:
    GF_SERVER_HTTP_PORT: 3001
    GF_SECURITY_ADMIN_USER: admin
    GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD:-paymensch_admin}
  volumes:
    - ./monitoring/grafana-dashboards:/etc/grafana/provisioning/dashboards
    - ./monitoring/grafana-datasources.yml:/etc/grafana/provisioning/datasources/datasources.yml
    - grafana_data:/var/lib/grafana
  ports:
    - '3001:3001'

loki:
  image: grafana/loki:latest
  ports:
    - '3100:3100'
  volumes:
    - ./monitoring/loki-config.yml:/etc/loki/local-config.yaml
    - loki_data:/loki

promtail:
  image: grafana/promtail:latest
  volumes:
    - ./monitoring/promtail-config.yml:/etc/promtail/config.yml
    - /var/log/paymensch:/var/log/paymensch  # Structured logs written here
  command: -config.file=/etc/promtail/config.yml
```

#### Critical Dashboards (Pre-Built, Committed as Code)

These dashboards are committed to `monitoring/grafana-dashboards/` as JSON — provisioning auto-loads them:

| Dashboard | What It Shows |
|-----------|---------------|
| **Payment Overview** | Transactions/min, success rate, avg latency, revenue volume (NPR), error breakdown by gateway |
| **Gateway Health** | Per-gateway: latency p50/p95/p99, error rate, uptime %, last error message |
| **API Performance** | Request rate, response time by route, status code distribution, active connections |
| **Queue Health** | BullMQ queue depth, processing rate, failed jobs, retry count, oldest pending job |
| **Business Metrics** | Active merchants, new signups, transaction volume by plan tier, MRR estimate |

#### Alerting Rules (Prometheus)

```yaml
# monitoring/prometheus-alerts.yml
groups:
  - name: paymensch_critical
    rules:
      - alert: PaymentSuccessRateDrop
        expr: |
          rate(paymensch_payments_completed_total[5m]) /
          rate(paymensch_payments_initiated_total[5m]) < 0.80
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Payment success rate dropped below 80%"
          description: "{{ $value | humanizePercentage }} success rate in last 5 minutes"

      - alert: GatewayLatencySpike
        expr: histogram_quantile(0.95, rate(paymensch_gateway_latency_seconds_bucket[5m])) > 5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Gateway {{ $labels.gateway }} p95 latency > 5s"

      - alert: DatabaseConnectionPoolNearLimit
        expr: paymensch_active_connections > 15
        for: 2m
        labels:
          severity: warning

      - alert: QueueBacklog
        expr: paymensch_queue_depth > 100
        for: 10m
        labels:
          severity: warning

      - alert: ServiceDown
        expr: up{job="paymensch-api"} == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Paymensch API is down"
```

#### External Monitoring (SaaS Free Tiers)

| Tool | What | Free Tier |
|------|------|-----------|
| **UptimeRobot** | HTTP ping to `/health` every 5 min | 50 monitors, email alert |
| **Sentry** | Exception tracking, stack traces, release tracking | 5K errors/month, 1 team member (enough for v1) |

#### Agent Rules for Monitoring

1. **Every new metric is added to the Payment Overview dashboard** — don't add metrics without visibility
2. **Alert thresholds are conservative** — better a false positive than a missed outage for payments
3. **Dashboards are committed as code** — `monitoring/grafana-dashboards/` is version controlled
4. **New gateway adapter = new panel in Gateway Health dashboard** — latency, error rate, uptime for every gateway
5. **Sentry is configured before first production deploy** — not after the first bug

---

## Development Cadence

### Build in Small, Testable Increments

**Rule: never build more than the human can test in one sitting.** Every increment must be a working, demonstrable feature — not a half-finished subsystem.

An increment is:
- A single API endpoint that returns real data (not mocked)
- A single dashboard page that loads and displays something
- A single gateway adapter that processes a test payment end-to-end
- A single pipeline stage wired into the request lifecycle

An increment is NOT:
- "All database migrations for the entire platform" — do one table at a time
- "All gateway adapters stubbed out" — build one fully, test it, then the next
- "The dashboard shell with all routes" — build one page end-to-end, then the next

### Milestone Management

Work is organized into milestones. Each milestone delivers a cohesive, testable slice of the product.

**Milestone structure (defined in BUILD_PLAN.md):**
```
Milestone 1: Project Scaffold & Testing Infrastructure
  └── Human test: monorepo builds, test suite runs, PG/Redis spin up

Milestone 2: Database Schema
  └── Human test: migrations run clean, seed data loads, can query tables

Milestone 3: Payment Pipeline (single gateway)
  └── Human test: initiate → callback → verify completes with eSewa sandbox

Milestone 4: Remaining Gateways
  └── Human test: Khalti and Fonepay work identically

Milestone 5: Merchant Dashboard (core pages)
  └── Human test: login, view transactions, configure gateways

Milestone 6: Webhooks & Audit
  └── Human test: webhook fires on payment, audit log is queryable

...etc
```

**Milestone gates are HARD stops.** The agent must:
1. Announce milestone completion
2. Present a summary of what was built
3. Provide exact steps for the human to test (curl commands, browser URLs, test credentials)
4. Wait for human sign-off before starting the next milestone

### Task Tracking Within Milestones

Every milestone starts with the agent creating tasks using TaskCreate:

```
Milestone 3: Payment Pipeline
  Task 1: Write gateway adapter contract tests
  Task 2: Implement eSewa adapter
  Task 3: Write pipeline stage tests (validate, auth, idempotency)
  Task 4: Implement pipeline stages
  Task 5: Write POST /v1/payments/initiate route test
  Task 6: Implement the route
  Task 7: Wire webhook receiver
  Task 8: End-to-end sandbox test
```

Tasks are marked `completed` as each finishes. Human can see progress at any time via TaskList.

### Commit Strategy

```
git log for Milestone 3 might look like:
  feat: add gateway adapter contract test suite
  test: add eSewa adapter tests
  feat: implement eSewa gateway adapter
  test: add pipeline idempotency tests
  feat: implement idempotency middleware
  feat: add POST /v1/payments/initiate endpoint
  test: add webhook receiver tests
  feat: implement webhook receiver
  fix: handle eSewa callback timeout gracefully
```

One commit per meaningful change. No "WIP" or "fix stuff" commits.

## Project Structure

```
paymensch/
├── CLAUDE.md                     # This file
├── docs/
│   ├── SPEC.md                   # Full product spec
│   ├── BUILD_PLAN.md             # Ordered implementation plan
│   └── BRAND.md                  # Design system and brand guide
├── packages/
│   ├── shared/                   # @paymensch/shared
│   │   └── src/
│   │       ├── types.ts
│   │       ├── schemas.ts
│   │       ├── constants.ts
│   │       └── errors.ts
│   ├── api/                      # @paymensch/api
│   │   └── src/
│   │       ├── server.ts
│   │       ├── app.ts
│   │       ├── config.ts
│   │       ├── plugins/
│   │       ├── routes/
│   │       ├── pipeline/
│   │       ├── gateways/
│   │       ├── services/
│   │       ├── db/
│   │       │   └── migrations/
│   │       └── queue/
│   ├── dashboard/                # @paymensch/dashboard — merchants + marketing
│   │   └── src/
│   │       └── app/
│   │           ├── (marketing)/   # Public: landing, pricing, blog
│   │           ├── (auth)/        # Login, signup
│   │           └── (dashboard)/   # Protected: merchant pages
│   ├── docs/                      # @paymensch/docs — VitePress developer docs
│   │   └── src/
│   │       ├── quickstart.md
│   │       ├── api/               # Auto-generated from OpenAPI spec
│   │       ├── guides/
│   │       └── errors/
│   └── admin/                     # @paymensch/admin — super admin dashboard
│       └── src/
│           └── app/
│               ├── merchants/
│               ├── transactions/
│               ├── revenue/
│               ├── system-health/
│               ├── audit-log/
│               └── settings/
├── graphify-out/                 # Knowledge graph (auto-generated)
├── docker-compose.yml
├── turbo.json
├── package.json
└── tsconfig.base.json
```

## Key Patterns

### Gateway Adapter Interface
Every gateway implements:
```typescript
interface PaymentGateway {
  readonly name: string;
  initiatePayment(params: PaymentRequest): Promise<PaymentResponse>;
  verifyPayment(reference: string): Promise<VerificationResponse>;
  processRefund(params: RefundRequest): Promise<RefundResponse>;
  healthCheck(): Promise<GatewayStatus>;
}
```

### Pipeline (Request Lifecycle)
```
validate schema → authenticate → rate limit → idempotency check → route gateway → execute → log audit → respond
```
Each stage is a standalone function with typed input/output. Testable in isolation.

### Database
- UUIDs for all primary keys
- Amounts in subunits (integer), never float
- Every row has `created_at`, `updated_at`
- `gateway_events` stores raw gateway request/response as JSONB
- `audit_logs` is append-only, partitioned by month, retained 5+ years

## Agent Workflow

When implementing any feature:

1. **Check for new skills** — `find-skills` for the domain you're about to work in
2. Read `docs/BRAND.md` for design tokens — every color, font, and spacing value must reference a semantic token
3. Read relevant section of `docs/SPEC.md`
4. Read relevant milestone in `docs/BUILD_PLAN.md`
5. **Create tasks** for the milestone using TaskCreate (one per step)
6. Query `/graphify` to understand existing code
7. Create isolated worktree (`using-git-worktrees`)
8. For each task in the milestone:
   - Write tests first (`test-driven-development`)
   - Implement until tests pass
   - Mark task completed
9. Run full test suite (`verification-before-completion`)
10. Run `/graphify` to update knowledge graph
11. **Present milestone summary** — what was built, how to test it
12. **Wait for human sign-off** before next milestone
13. After sign-off: request code review (`requesting-code-review`)
14. Run security review on payment/auth code (`security-review`)
15. Commit and merge (`finishing-a-development-branch`)

**Never skip the human gate between milestones.** Even if everything compiles and tests pass, the human tests the milestone before the next one starts.

### Security Gates

At the end of every milestone, the agent runs this checklist. Security is not a Phase 6 item — it's continuous:

```bash
# 1. Secrets check
gitleaks detect --source . --no-git
# Expected: no leaks found

# 2. Dependency audit
npm audit --audit-level=high
# Expected: no critical or high vulnerabilities

# 3. TypeScript strict
npm run typecheck
# Expected: no errors, no `any` escapes

# 4. Security lint
npm run lint  # includes eslint-plugin-security
# Expected: no security warnings

# 5. Log cleanliness check
npm test && grep -r "sk_live_\|sk_test_\|secret" logs/ || true
# Expected: NO matches. If any API key or secret appears in logs, FIX BEFORE CONTINUING.

# 6. Container scan (if Docker images built)
trivy image paymensch-api:latest
# Expected: no critical CVEs
```

## Deployment

### Docker Compose (Dev + Prod)

Same infrastructure definition. `docker-compose.yml` for all services. `docker-compose.prod.yml` for production overrides (resource limits, restart policies, actual TLS certs, proper secrets). No k8s day 1.

**Dev:** `npm run infra:up` — everything starts with hot-reload.
**Prod:** `docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d`

Reverse proxy: Traefik handles routing, TLS termination (Let's Encrypt), and IP restrictions.

### k8s Migration Path

Migrate when multi-server is needed (est. 12-18 months). Compose services map 1:1 to k8s resources. App code doesn't change.

### Domain Map

| Site | URL | Access |
|------|-----|--------|
| Marketing | paymensch.io | Public |
| API | api.paymensch.io | Public (rate-limited) |
| Docs | docs.paymensch.io | Public |
| Merchant App | app.paymensch.io | Public (behind login) |
| Super Admin | admin.paymensch.io | IP-restricted (office/VPN) |
| Grafana | grafana.paymensch.io | IP-restricted |
| Prometheus | prometheus.paymensch.io | IP-restricted |

## Domain Knowledge

### Nepali Payment Gateways
- **eSewa:** REST API, HMAC signature auth, returns redirect URL
- **Khalti:** REST API, secret key auth, returns pidx for verification
- **Fonepay:** SOAP/XML, merchant code + secret, bank redirect flow
- **ConnectIPS:** Bank-direct, account-based, different flow entirely

### NRB Compliance
- We never hold customer funds (PSP license not required)
- Transaction logs retained minimum 5 years
- PCI: never store card numbers
- Cross-border: domestic-only for v1

### Currencies
- NPR: 2 decimal places, subunit = paisa (divisor 100)
- Future: USD, INR, JPY
- Subunit math only. `1500.50 NPR` = `150050` in DB.
