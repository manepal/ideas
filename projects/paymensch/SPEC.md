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

## Subscription & Pricing (v2 — Not Built in v1)

### Tiers

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

## Dev Environment

### One Command
```bash
git clone <repo-url> && cd paymensch && cp .env.example .env && npm run setup
```

`npm run setup` chains: infra:up → install → db:migrate → db:seed → dev

### npm Scripts
```
setup, infra:up, infra:down, infra:restart, infra:logs, infra:status
dev, dev:api, dev:dashboard, build
test, test:api, test:dashboard, test:e2e
db:migrate, db:migrate:test, db:seed
lint, typecheck, graphify
```

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

Built in 6 milestones, each fully testable before the next begins:

| # | Milestone | What You Can Do | ~Hours |
|---|-----------|----------------|--------|
| **M1** | Scaffold & Testing Infrastructure | Monorepo builds, tests run with real PG/Redis, Docker spins up everything | 4 |
| **M2** | Database & Core Types | Migrations run, seed data loads, shared types compile, can query tables | 3 |
| **M3** | Payment Pipeline (eSewa only) | Initiate → callback → verify → refund end-to-end with eSewa sandbox | 10 |
| **M4** | Remaining Gateways + Webhooks | Khalti and Fonepay work identically, webhooks deliver to merchant URLs | 8 |
| **M5** | Merchant Dashboard | Login, view transactions, configure gateways, manage API keys, see analytics | 12 |
| **M6** | Monitoring, Audit, Polish | Dashboards live, alerts fire, audit logs queryable, accessibility passes | 6 |
| **Total** | | | **~43 hours** |

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
