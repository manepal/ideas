# Paymensch — Spec

Bring Your Own Payment Gateway (BYOPG) platform. Single integration to accept payments via eSewa, Khalti, Fonepay, and more — starting in Nepal, expanding across South Asia. Universal adapter model: add any gateway in any country by implementing one TypeScript interface. Developer-first: clean REST API, sandbox environment, typed SDKs, webhook management. Evolves into aggregated PSP settlement and Stripe-like billing/subscriptions.

## Stack

- **API:** Node.js 22+, Fastify 5, TypeScript 5.x (strict mode)
- **Dashboard:** Next.js 15 (App Router), Tailwind CSS, shadcn/ui
- **Database:** PostgreSQL 16 + Redis 7
- **Queue:** BullMQ (Redis-backed)
- **Monorepo:** Turborepo — `packages/shared/`, `packages/api/`, `packages/dashboard/`
- **Testing:** Vitest, Testcontainers, nock, React Testing Library, MSW, Playwright, fast-check
- **Logging:** pino (structured JSON) + Loki + promtail
- **Metrics:** prom-client → Prometheus → Grafana (dashboards as code)
- **Error tracking:** Sentry (free tier)
- **Uptime:** UptimeRobot (free tier)
- **Hosting:** Dockerized, Hetzner or DigitalOcean (Singapore region)
- **No blockchain. No cryptocurrency. No holding customer funds.**

## Product Philosophy

We are a technology provider, not a Payment Service Provider. We never hold, pool, or settle customer funds. Money flows directly through licensed gateways (eSewa, Khalti, Fonepay) to merchant bank accounts. We charge a monthly SaaS fee for the API and dashboard — no per-transaction percentage. This avoids NRB PSP licensing requirements and aligns our incentives with merchant success.

## How Money Moves

There is no inter-wallet payment routing infrastructure in Nepal. No universal settlement system exists. This means:

**Each merchant connects their own gateway accounts to Paymensch.** The merchant signs up for eSewa Merchant, Khalti Merchant, and Fonepay Merchant independently. They provide us their credentials (merchant code, secret key, etc.) via the dashboard. We call those APIs on the merchant's behalf. Money flows directly:

```
Customer pays via eSewa
  → Money lands in Merchant's eSewa account (NOT Paymensch's account)
  → Merchant withdraws from eSewa to their bank

Customer pays via Khalti
  → Money lands in Merchant's Khalti account
  → Merchant withdraws from Khalti to their bank
```

Paymensch never appears in the money path. We are the API layer, the audit trail, and the analytics engine — not the settlement layer.

**What we do:**
- Route payment requests to the correct gateway using the merchant's credentials
- Normalize every gateway's response into a standard format
- Maintain a complete audit trail of every transaction across all gateways
- Provide analytics: which gateway processes fastest, success rates, volume trends
- Deliver webhooks so the merchant's system stays in sync

**What we do NOT do:**
- Hold or pool customer funds
- Settle between wallets or banks
- Accept eSewa and route to a Khalti account (inter-wallet settlement requires a PSP license)
- Process cross-border payments (requires separate NRB approval)

This is the same model that early Stripe used before they obtained licenses — merchant-owned gateway accounts, unified behind one clean API.

**Per-gateway settlement:**

| Gateway | Type | Money Lands In | Settlement to Bank |
|---------|------|---------------|-------------------|
| eSewa | Wallet | Merchant's eSewa wallet | Manual withdrawal by merchant |
| Khalti | Wallet | Merchant's Khalti wallet | Manual withdrawal by merchant |
| Fonepay | Bank redirect | Merchant's linked bank account | Automatic |
| ConnectIPS | Interbank transfer | Merchant's bank account directly | Automatic |

## Target Customer

**Who Paymensch is for (v1):**
- SaaS companies with recurring billing (monthly integration pain, not one-time)
- E-commerce platforms processing 500+ transactions/month across multiple gateways
- Any business where a developer's time costs more than NPR 2,500/month
- Businesses that already have gateway accounts and are tired of maintaining integration code

**Who it's NOT for (v1):**
- A shop doing 20 transactions/month — they can manually check each gateway
- A business that only accepts one wallet (common in Nepal)
- Anyone expecting Paymensch to provide a gateway account — BYOPG means bring your own

## BYOPG: Universal Model, Any Market

The BYOPG (Bring Your Own Payment Gateway) model is country-agnostic. The `PaymentGateway` interface doesn't know about Nepal, rupees, or any specific gateway. It knows about initiating a payment and verifying a payment — primitives that exist in every payment system on earth.

```
                 Paymensch API (one interface)
                           │
       ┌───────────────────┼───────────────────┐
       │                   │                   │
   🇳🇵 Nepal              🇮🇳 India          🇧🇩 Bangladesh
  ┌────┼────┐          ┌────┼────┐          ┌────┼────┐
 eSewa  Khalti      Razorpay PayU       bKash  Nagad
 Fonepay IME Pay    Cashfree PhonePe    DBBL   Upay
 ConnectIPS         Instamojo Paytm     
```

**What changes per country:** A new adapter file, new currency row, new fixture files for tests.

**What doesn't change:** The pipeline, the API contract, the dashboard, the merchant model.

**Why BYOPG wins for international expansion:** No PSP license needed in any country (we never hold funds). No regulatory moat to cross. Just add adapters. India alone has ~10,000 mid-market SaaS and e-commerce companies who maintain multi-gateway integrations — and no self-serve platform for them.

## PSP Evolution Path

BYOPG is v1. PSP is a future product, not a replacement. Both can coexist.

```
Phase 1 — BYOPG (v1):        Phase 2 — PSP (v2+)         Phase 3 — Full Platform
Technology Provider           Aggregated Settlement        "Stripe for South Asia"

Merchant owns gateway         Paymensch holds master       Choice: BYOPG or PSP.
accounts. We route APIs.      accounts. We pool +          Same API. Same dashboard.
                              net-settle to banks.         Billing + subscriptions.
```

**What a PSP license unlocks:**
- Single merchant onboarding — sign up with Paymensch, not 5 gateways
- Inter-gateway routing — customer pays Khalti, merchant's bank gets the money
- Net settlement — one daily deposit, not N separate withdrawals
- Better volume rates — master accounts negotiate harder

**Architecture is built for it.** When the license arrives, swap credentials on `gateway_configs` from merchant-owned to Paymensch-owned. The API doesn't change. The dashboard doesn't change.

## Core Product

### Unified Payment API

A single REST endpoint that routes to the best available Nepali payment gateway:

```
POST /v1/payments/initiate
Authorization: Bearer sk_live_xxx

{
  "amount_subunits": 150000,
  "currency": "NPR",
  "order_id": "ORDER-123",
  "callback_url": "https://merchant.com/payment/callback",
  "customer_email": "customer@email.com",
  "customer_phone": "98XXXXXXXX"
}
```

### Gateway Support

| Gateway | MVP | Auth Type | Integration Style |
|---------|-----|-----------|-------------------|
| eSewa | Yes | HMAC signature | REST, redirect URL |
| Khalti | Yes | Secret key | REST, pidx verification |
| Fonepay | Yes | Merchant code + secret | SOAP/XML, bank redirect |
| IME Pay | Post-MVP | TBD | REST |
| ConnectIPS | Post-MVP | TBD | Bank-direct |

### Pipeline Architecture

Every payment request flows through a typed, testable pipeline:

```
validate schema → authenticate → rate limit → idempotency check → route gateway → execute → log audit → respond
```

Each stage is a standalone function. Each stage is independently testable. The pipeline composes them.

### Gateway Abstraction

Adapter pattern. Every gateway implements a shared TypeScript interface:

```typescript
interface PaymentGateway {
  readonly name: string;
  initiatePayment(params: PaymentRequest): Promise<PaymentResponse>;
  verifyPayment(reference: string): Promise<VerificationResponse>;
  processRefund(params: RefundRequest): Promise<RefundResponse>;
  healthCheck(): Promise<GatewayStatus>;
}
```

New gateway = new adapter file. Passes a shared contract test suite. No other code changes.

### Idempotency

Every payment initiation requires an `Idempotency-Key` header. Concurrent requests with the same key return the same result. The first request processes; subsequent requests (within TTL) return the cached response. This is the hardest problem in payment APIs to get right, and it's built into the pipeline from day one.

### Webhooks

Merchants register a webhook URL. We POST payment status changes:

```
POST {merchant.webhook_url}
X-Paymensch-Signature: {HMAC of payload}

{
  "event": "payment.completed",
  "transaction_id": "uuid",
  "order_id": "ORDER-123",
  "amount_subunits": 150000,
  "currency": "NPR",
  "gateway": "esewa",
  "status": "completed",
  "completed_at": "2026-06-06T14:23:45Z"
}
```

Retry with exponential backoff (1m, 5m, 15m, 1h, 4h, 12h, 24h). After 3 days of failures, webhook is disabled and merchant is notified.

### API Keys

Two types per merchant:
- `sk_live_xxx_xxxxxxxxxxxxx` — live payments, real money
- `sk_test_xxx_xxxxxxxxxxxxx` — sandbox, fake money, simulated gateways

Keys are prefixed for easy identification in logs and are hashed in the database. Only shown once at creation time. Key rotation is instant — old key invalidated, new key active immediately.

## Merchant Dashboard

### Pages

| Page | What It Shows |
|------|---------------|
| **Login** | Email + password. No social login for v1. |
| **Overview** | Transaction volume today/this month, success rate, gateway health summary |
| **Transactions** | Paginated table, filterable by gateway/status/date, detail view per transaction |
| **Gateways** | Configure eSewa/Khalti/Fonepay credentials per gateway, test connection button |
| **API Keys** | View live/test keys (masked), copy to clipboard, rotate, reveal on demand |
| **Webhooks** | Set webhook URL, view recent deliveries, retry status, manual resend |
| **Settings** | Business name, email, password change, plan tier display |

### Component States

Every component handles 6 states:
1. Loading (first fetch, skeleton UI)
2. Loaded (data displayed correctly)
3. Empty (no data yet, helpful illustration + CTA)
4. Error (API failure, retry button, error message)
5. Edge case (single item, max items, very long text)
6. Submitting (button disabled, spinner, success/error feedback)

### Accessibility

- All interactive elements keyboard-navigable
- Forms show accessible error messages (not just red borders)
- Screen reader labels on all icons and actions
- Color is never the only differentiator (status badges use shape + color)
- Contrast ratios meet WCAG AA

## Data Model

### Core Tables

```
merchants
├── id (UUID, PK)
├── name, email (unique), password_hash
├── business_name
├── api_key_live (hashed), api_key_test (hashed)
├── webhook_url, webhook_secret
├── plan_tier: 'starter' | 'growth' | 'business'
├── status: 'active' | 'suspended' | 'closed'
├── metadata (JSONB)
└── created_at, updated_at

gateway_configs
├── id (UUID, PK)
├── merchant_id (FK)
├── gateway: 'esewa' | 'khalti' | 'fonepay' | 'imepay' | 'connectips'
├── credentials (JSONB, encrypted at rest)
├── enabled (bool), priority (int)
├── unique(merchant_id, gateway)
└── created_at, updated_at

transactions
├── id (UUID, PK)
├── merchant_id (FK)
├── idempotency_key (unique per merchant)
├── amount_subunits (int, NOT NULL)
├── currency (varchar(3), NOT NULL)
├── gateway (enum)
├── gateway_txn_id
├── status: 'initiated' | 'processing' | 'completed' | 'failed' | 'refunded' | 'expired'
├── type: 'payment' | 'refund' | 'payout' | 'subscription_charge'
├── subscription_id (nullable FK, future)
├── invoice_id (nullable FK, future)
├── customer_email, customer_phone
├── description, metadata (JSONB)
├── settled_at
└── created_at, updated_at

gateway_events
├── id (UUID, PK)
├── transaction_id (FK)
├── event_type: 'request' | 'response' | 'callback' | 'reconciliation'
├── raw_payload (JSONB)
└── created_at

audit_logs
├── id (UUID, PK)
├── entity_type, entity_id
├── action: 'created' | 'updated' | 'deleted' | 'accessed' | 'executed'
├── actor: 'system' | 'merchant:{id}' | 'admin:{id}' | 'gateway:{name}'
├── changes (JSONB)
├── ip_address, user_agent
├── merchant_id (FK, for RLS)
└── created_at

refunds
├── id (UUID, PK)
├── transaction_id (FK)
├── amount_subunits (int)
├── reason, status
└── created_at, updated_at
```

### Future Tables (Not Built in v1, But Columns Above Are Ready)

```
plans, subscriptions, invoices, invoice_items
```

### Data Rules

- All monetary values are integers (subunits). 1500.50 NPR = 150050 in DB.
- All primary keys are UUIDs.
- Every row has `created_at` and `updated_at`.
- `audit_logs` is append-only — no UPDATE, no DELETE. Partitioned by month. Retained 5+ years per NRB.
- Gateway credentials are encrypted at rest using AES-256-GCM with a per-environment key.
- API keys are hashed (SHA-256) before storage. Only the prefix (`sk_live_` / `sk_test_`) and last 4 chars are shown after creation.

## Subscription & Billing (v2 — Works With BYOPG)

Billing is independent of the PSP license. Plans, invoices, proration, dunning, and subscription lifecycle run entirely within Paymensch. Payment capture happens through the merchant's existing BYOPG adapters — the same gateways they already use for one-time payments.

```
Subscription created → Invoice generated → Charge via eSewa/Khalti adapter → Money in merchant's gateway account
```

The billing engine doesn't know or care who owns the gateway credentials.

### Tiers (Paymensch's Own Pricing)

| Tier | Monthly Fee | Volume | What's Different |
|------|------------|--------|-----------------|
| **Starter** | NPR 2,500 | Up to NPR 5L/month | 3 gateways, API + dashboard, sandbox |
| **Growth** | NPR 7,500 | Up to NPR 25L/month | + Priority routing, custom webhooks, email support |
| **Business** | NPR 20,000 | Unlimited | + SLA, dedicated support, white-label option |

### What Every Plan Gets
- Full sandbox environment (test API keys + simulated gateway responses)
- API docs with code samples (curl, Node, Python)
- Dashboard with transaction history and settlement reports
- Webhook retry logic (exponential backoff, up to 3 days)
- Audit logs and analytics

## Security

### Non-Negotiable
- API keys never in logs (pino `redact` list catches them)
- Credentials encrypted at rest (AES-256-GCM)
- No card numbers stored (PCI DSS: we route to gateways, they handle cards)
- Audit log is immutable and complete
- Idempotency enforced server-side (not client-dependent)
- Rate limiting per merchant (100 req/min, burst 200)
- CORS restricted to registered dashboard URLs
- Helmet headers on all responses

### NRB Compliance
- We never hold, pool, or settle customer funds — PSP license not required
- Transaction logs retained minimum 5 years
- Domestic-only for v1 (cross-border requires separate NRB approval)
- All financial data stored in Nepal-friendly Singapore region

## Monitoring

### Self-Hosted Stack (docker-compose)
- prom-client → Prometheus → Grafana (metrics + dashboards)
- Loki + promtail (log aggregation)
- Grafana Alertmanager (alerting)

### Free SaaS
- UptimeRobot (health check pings every 5 min)
- Sentry (exception tracking, 5K errors/month free)

### Critical Dashboards (Committed as Code)
- Payment Overview: transactions/min, success rate, revenue volume, error breakdown
- Gateway Health: per-gateway latency (p50/p95/p99), error rate, uptime
- API Performance: request rate, response time by route, status codes
- Queue Health: BullMQ depth, processing rate, failed jobs
- Business Metrics: active merchants, new signups, volume by plan tier

## Sites & Packages

Monorepo with five packages. Three sites, three audiences, one shared codebase.

### Package Map

| Package | Site | Audience | Framework |
|---------|------|----------|-----------|
| `packages/api` | api.paymensch.io | Machine clients | Fastify |
| `packages/dashboard` | app.paymensch.io + paymensch.io | Merchants + public | Next.js |
| `packages/admin` | admin.paymensch.io | Paymensch staff | Next.js |
| `packages/docs` | docs.paymensch.io | Developers | VitePress |
| `packages/shared` | (internal) | All packages | TypeScript |

### Marketing Site (paymensch.io)

Built into `packages/dashboard` as a public route group `(marketing)`. Same Next.js app, different layouts, no auth required.

```
paymensch.io/
├── /              Landing — "One API for every payment gateway in South Asia"
├── /pricing       Three tiers, feature comparison table, FAQ
├── /docs          Links to docs.paymensch.io
├── /blog          Content marketing (SEO), changelog posts
├── /about         Team, mission, contact
├── /login         Redirects to app.paymensch.io
└── /signup        Merchant registration
```

### Developer Docs (docs.paymensch.io)

Separate VitePress app in `packages/docs/`. Static, MDX-native, built-in search. No JS overhead for readers. API reference auto-generated from Fastify OpenAPI spec.

```
docs.paymensch.io/
├── /                    Quickstart — first payment in 5 minutes
├── /api/                Auto-generated from openapi.json
│   ├── /payments
│   ├── /refunds
│   ├── /webhooks
│   ├── /gateway-configs
│   └── /api-keys
├── /sdks/               Node.js, Python quickstart
├── /guides/             Gateway setup, webhook testing, idempotency deep-dive
├── /errors/             Every error code with resolution steps
└── /changelog/          API version history, breaking changes, deprecations
```

API reference workflow: write route → Fastify OpenAPI plugin generates `openapi.json` → VitePress consumes it → reference pages auto-render. New endpoint = docs page appears automatically. No manual sync.

### Super Admin Dashboard (admin.paymensch.io)

Separate Next.js app in `packages/admin/`. Completely independent auth domain from merchant dashboard. Separate `admins` table with role-based access. **Not publicly accessible** — IP-restricted to office/VPN IP ranges via Traefik middleware.

**Roles:** `super_admin` (full access), `support_agent` (read merchants, impersonate), `read_only` (view analytics).

| Page | What It Shows |
|------|---------------|
| **Merchants** | List all, filter by status/plan/date, approve/suspend/reactivate, impersonate for support debugging |
| **Merchant Detail** | Full profile, gateway configs, transaction summary, activity timeline, impersonation entry point |
| **Transactions** | All transactions across all merchants, advanced filters, manual refund capability, flagged anomaly review |
| **Revenue** | MRR, ARR, churn, revenue by plan tier, new signups over time, gateway usage distribution across install base |
| **System Health** | Per-gateway status (aggregated), queue depth/processing rate, DB connection pool, Redis memory, error rate trends |
| **Audit Log** | Complete append-only trail, filterable by merchant/entity/action/date range, CSV/JSON export for compliance |
| **Feature Flags** | Toggle features per merchant or globally, manage rollout percentage |
| **Settings** | Global rate limits, gateway-level config overrides, admin user management |

**Auth:** Email + password + TOTP 2FA. Separate `admins` table. Separate JWT signing secret. No overlap with merchant auth.

## Dev Environment

### One Command
```bash
git clone <repo-url> && cd paymensch && cp .env.example .env && npm run setup
```

`npm run setup` chains: infra:up → install → db:migrate → db:seed → dev

### npm Scripts
```
setup, infra:up, infra:down, infra:restart, infra:logs, infra:status
dev, dev:api, dev:dashboard, dev:docs, dev:admin
build, test, test:api, test:dashboard, test:e2e
db:migrate, db:migrate:test, db:seed
lint, typecheck, graphify, docs:generate
```

## Deployment

### Docker Compose (Dev + Prod)

Same infrastructure definition for both environments. Single `docker-compose.yml` with production overrides in `docker-compose.prod.yml`. No k8s day 1.

```
┌──────────────────────────────────────────────────┐
│                 Docker Compose                    │
│                                                   │
│  ┌─────────┐ ┌───────────┐ ┌───────┐ ┌────────┐ │
│  │   API   │ │    App     │ │ Docs  │ │ Admin  │ │
│  │  :4000  │ │   :3000   │ │ :5173 │ │ :4002  │ │
│  └─────────┘ └───────────┘ └───────┘ └────────┘ │
│  ┌──────────┐ ┌──────┐ ┌───────────────────────┐ │
│  │PostgreSQL│ │Redis │ │Traefik (reverse proxy)│ │
│  │  :5432   │ │:6379 │ │:80/:443 + auto TLS    │ │
│  └──────────┘ └──────┘ └───────────────────────┘ │
│  ┌──────────┐ ┌──────┐ ┌────────┐               │
│  │Prometheus│ │Grafana│ │ Loki   │               │
│  │  :9090   │ │:3001 │ │ :3100  │               │
│  └──────────┘ └──────┘ └────────┘               │
└──────────────────────────────────────────────────┘
```

**Dev:** `npm run infra:up` — builds and starts everything locally, hot-reload on all services.

**Prod:** `docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d` — same services, production config (resource limits, restart policies, actual TLS certs via Traefik + Let's Encrypt, proper secrets from environment).

**Why not k8s day 1:** For a pre-revenue product with zero customers, k8s costs 1-2 weeks of infra work before any payment code ships. Docker Compose gives real dev/prod parity (same Postgres version, same Redis version, same network topology) without the YAML overhead or learning curve.

### Kubernetes Migration Path (When Needed)

Migrate when: multi-server deployment is required, zero-downtime deploys become revenue-critical, or auto-scaling is needed. Estimated trigger: 12-18 months post-launch.

Migration is straightforward because Docker Compose services map 1:1 to k8s resources:

| Compose | k8s |
|---------|-----|
| `services.api` | Deployment + Service + Ingress |
| `services.postgres` | StatefulSet + PersistentVolumeClaim (or migrate to managed DB) |
| `services.redis` | StatefulSet (or migrate to managed Redis) |
| `services.traefik` | Ingress Controller |
| `volumes` | PersistentVolume + PersistentVolumeClaim |
| `networks` | NetworkPolicy + CNI |

The app code doesn't change. Only the deployment layer.

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

## API Endpoints (v1)

```
Merchant:
  POST   /v1/merchants/register
  POST   /v1/merchants/login
  GET    /v1/merchants/me
  PATCH  /v1/merchants/me

Payments:
  POST   /v1/payments/initiate
  GET    /v1/payments/:id/status
  POST   /v1/payments/:id/refund

Transactions:
  GET    /v1/transactions
  GET    /v1/transactions/:id

Webhooks:
  POST   /v1/webhooks/:gateway

API Keys:
  POST   /v1/keys/rotate

Gateway Configs:
  GET    /v1/gateways
  POST   /v1/gateways
  PATCH  /v1/gateways/:id
  POST   /v1/gateways/:id/test

Health:
  GET    /health
  GET    /health/ready
  GET    /health/live

Metrics:
  GET    /metrics          (Prometheus scrape target, port 9090)
```

## Build Milestones

Built in 8 milestones, each fully testable before the next begins:

| # | Milestone | What You Can Do | ~Hours |
|---|-----------|----------------|--------|
| **M1** | Scaffold & Testing Infrastructure | Monorepo builds, tests run with real PG/Redis, Docker spins up everything | 4 |
| **M2** | Database & Core Types | Migrations run, seed data loads, shared types compile, can query tables | 3 |
| **M3** | Payment Pipeline (eSewa only) | Initiate → callback → verify → refund end-to-end with eSewa sandbox | 10 |
| **M4** | Remaining Gateways + Webhooks | Khalti and Fonepay work identically, webhooks deliver to merchant URLs | 8 |
| **M5** | Merchant Dashboard | Login, view transactions, configure gateways, manage API keys, see analytics | 12 |
| **M6** | Marketing Site & Docs | Landing, pricing, blog, quickstart, auto-generated API reference, error code docs | 8 |
| **M7** | Super Admin Dashboard | Merchant management, impersonation, revenue analytics, feature flags, audit log viewer | 10 |
| **M8** | Monitoring, Audit, Polish | Dashboards live, alerts fire, accessibility passes, performance hardening | 6 |
| **Total** | | | **~61 hours** |

## Verification

- [ ] `npm run setup` completes end-to-end on a fresh clone
- [ ] Test suite passes (`npm test`) — real PG/Redis, no mocks
- [ ] eSewa sandbox: initiate → callback → verify returns `completed`
- [ ] Khalti and Fonepay: same flow, different gateway, same result
- [ ] Idempotency: concurrent requests with same key return identical result
- [ ] Webhook: payment status change → POST to merchant URL with valid HMAC
- [ ] Dashboard: login → view transactions → filter by gateway → configure eSewa → rotate API key
- [ ] All dashboard components handle loading, loaded, empty, error, edge, submitting states
- [ ] Audit logs are append-only, no UPDATE or DELETE works
- [ ] Grafana dashboards load from committed JSON, show real metrics
- [ ] Playwright E2E tests pass on both Chromium and Firefox
- [ ] Accessibility audit passes (axe-core, keyboard navigation, screen reader)
- [ ] Never logs API keys, credentials, or signatures in any log level
