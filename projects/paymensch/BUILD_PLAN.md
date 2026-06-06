# Paymensch — Build Plan

Ordered, file-by-file implementation plan in 6 milestones. Each milestone is a hard stop — the agent presents a summary, provides testing instructions, and waits for human sign-off before the next milestone begins.

**Before writing any code:** read `CLAUDE.md` for conventions, skills requirements, and non-negotiable rules.

---

## Milestone 1: Scaffold & Testing Infrastructure

**Goal:** Monorepo builds, tests run with real PostgreSQL + Redis, Docker spins up everything. Zero features — just infrastructure.

### Step 1.1: Create monorepo scaffold

```bash
mkdir paymensch && cd paymensch
npm init -y
```

Create root `package.json` with workspaces and the scripts from CLAUDE.md.

**Create:** `turbo.json`
```json
{
  "$schema": "https://turbo.build/schema.json",
  "globalDependencies": ["**/.env.*local"],
  "tasks": {
    "build": { "dependsOn": ["^build"], "outputs": ["dist/**"] },
    "dev": { "cache": false, "persistent": true },
    "test": { "dependsOn": ["build"] },
    "test:e2e": { "dependsOn": ["build"] },
    "lint": {},
    "typecheck": { "dependsOn": ["^build"] },
    "db:migrate": { "cache": false },
    "db:migrate:test": { "cache": false },
    "db:seed": { "cache": false }
  }
}
```

**Create:** `tsconfig.base.json` with strict TypeScript settings.

**Create:** `docker-compose.yml` from CLAUDE.md — PostgreSQL 16 + Redis 7 + Prometheus + Grafana + Loki + promtail.

**Create:** `.env.example` from CLAUDE.md.

**Verify:** `docker compose up -d` → `docker compose ps` shows all services healthy.

### Step 1.2: Scaffold shared package

```bash
mkdir -p packages/shared/src && cd packages/shared && npm init -y
```

**Create:** `packages/shared/package.json` — name `@paymensch/shared`, type `module`.

**Create:** `packages/shared/src/types.ts` — all TypeScript interfaces:
```ts
export type Player = 'tiger' | 'goat'; // (template — replace with actual types)
export type TransactionStatus = 'initiated' | 'processing' | 'completed' | 'failed' | 'refunded' | 'expired';
export type TransactionType = 'payment' | 'refund' | 'payout' | 'subscription_charge';
export type PlanTier = 'starter' | 'growth' | 'business';
export type GatewayName = 'esewa' | 'khalti' | 'fonepay' | 'imepay' | 'connectips';

export interface Merchant { id: string; name: string; email: string; ... }
export interface Transaction { id: string; merchantId: string; amountSubunits: number; ... }
export interface PaymentRequest { amountSubunits: number; currency: string; orderId: string; ... }
export interface PaymentResponse { success: boolean; gatewayTxnId?: string; redirectUrl?: string; ... }
export interface PaymentGateway { readonly name: string; initiatePayment(...): Promise<...>; ... }
```

**Create:** `packages/shared/src/schemas.ts` — Zod schemas for all request/response payloads.

**Create:** `packages/shared/src/constants.ts` — status enums, error codes, currency definitions.

**Create:** `packages/shared/src/errors.ts` — typed error classes:
```ts
export class PaymentError extends Error {
  constructor(
    public code: string,
    public statusCode: number,
    public retryable: boolean,
    message: string
  ) { super(message); this.name = 'PaymentError'; }
}
export class GatewayError extends PaymentError { ... }
export class ValidationError extends PaymentError { ... }
```

**Create:** `packages/shared/tsconfig.json` extending `../../tsconfig.base.json`.

**Verify:** `npm run typecheck` passes. Types importable from other packages.

### Step 1.3: Scaffold API package

```bash
mkdir -p packages/api/src && cd packages/api && npm init -y
```

**Create:** `packages/api/package.json` — name `@paymensch/api`, dependencies on `@paymensch/shared`, Fastify, Knex, BullMQ, pino, prom-client.

**Create:** `packages/api/src/server.ts` — Fastify bootstrap that starts on `API_PORT`:
```ts
import Fastify from 'fastify';
import { logger } from './logger';
export async function buildServer() {
  const app = Fastify({ logger });
  app.get('/health', async () => ({ status: 'ok', uptime: process.uptime() }));
  app.get('/health/ready', async () => ({ status: 'ready' }));
  app.get('/health/live', async () => ({ status: 'alive' }));
  return app;
}
```

**Create:** `packages/api/src/app.ts` — registers plugins, routes, starts listening.

**Create:** `packages/api/src/config.ts` — typed environment variable loader using Zod.

**Create:** `packages/api/src/logger.ts` — pino instance with redaction from CLAUDE.md.

**Create:** `packages/api/src/metrics.ts` — prom-client counters, histograms, gauges.

**Create:** `packages/api/src/db/client.ts` — Knex instance with connection pool config.

**Create:** `packages/api/knexfile.ts` — Knex configuration for dev/test.

**Create:** `packages/api/tsconfig.json`.

**Verify:** `npm run dev:api` starts. `curl localhost:4000/health` returns `{"status":"ok"}`.

### Step 1.4: Scaffold dashboard package

```bash
npx create-next-app@latest packages/dashboard --typescript --tailwind --app --src-dir
```

**Create:** `packages/dashboard/package.json` — name `@paymensch/dashboard`, dependencies on `@paymensch/shared`, React Query, React Hook Form, shadcn/ui.

**Verify:** `npm run dev:dashboard` starts Next.js on :3000.

### Step 1.5: Set up Vitest with Testcontainers

**Install:** `vitest`, `@vitest/coverage-v8`, `testcontainers` in root and each package.

**Create:** `vitest.config.ts` in each package — configured to use V8 coverage provider.

**Create:** `packages/api/src/__tests__/health.test.ts`:
```ts
import { describe, it, expect, beforeAll, afterAll } from 'vitest';
import { buildServer } from '../server';

describe('health', () => {
  const app = await buildServer();
  afterAll(() => app.close());

  it('returns ok when database is reachable', async () => {
    const res = await app.inject({ method: 'GET', url: '/health' });
    expect(res.statusCode).toBe(200);
    expect(res.json().status).toBe('ok');
  });
});
```

**Create:** `packages/api/vitest.config.ts` — test database setup with `paymensch_test`, migrations before suite, transaction rollback per test.

**Verify:** `npm test` runs, finds the test, passes. Real PostgreSQL hit, not mocked.

### Step 1.6: Set up Playwright for E2E

**Install:** `@playwright/test` in root.

**Create:** `playwright.config.ts` — Chromium + Firefox, base URL `http://localhost:3000`.

**Create:** `e2e/health.spec.ts`:
```ts
import { test, expect } from '@playwright/test';
test('health endpoint returns ok', async ({ request }) => {
  const res = await request.get('http://localhost:4000/health');
  expect(res.status()).toBe(200);
  expect(await res.json()).toMatchObject({ status: 'ok' });
});
```

**Verify:** `npm run test:e2e` passes.

### Step 1.7: Commit and present M1 summary

```bash
git add -A && git commit -m "feat: scaffold monorepo with testing infrastructure"
```

**Milestone 1 complete.** Human test:
```bash
npm run setup        # One command, everything works
npm test             # Tests pass with real PG/Redis
npm run test:e2e     # Playwright passes
npm run dev          # API on :4000, Dashboard on :3000, no crashes
```

---

## Milestone 2: Database & Core Types

**Goal:** Migrations run, seed data loads, all types compile, shared package is the single source of truth.

### Step 2.1: Write first migration — merchants

**Create:** `packages/api/src/db/migrations/001_create_merchants.ts`

```ts
import { Knex } from 'knex';

export async function up(knex: Knex): Promise<void> {
  await knex.schema.createTable('merchants', (t) => {
    t.uuid('id').primary().defaultTo(knex.raw('gen_random_uuid()'));
    t.string('name').notNullable();
    t.string('email').notNullable().unique();
    t.string('password_hash').notNullable();
    t.string('business_name').notNullable();
    t.string('api_key_live_hash').unique();
    t.string('api_key_test_hash').unique();
    t.string('webhook_url');
    t.string('webhook_secret');
    t.string('plan_tier').notNullable().defaultTo('starter');
    t.string('status').notNullable().defaultTo('active');
    t.jsonb('metadata').defaultTo('{}');
    t.timestamps(true, true);
  });
}

export async function down(knex: Knex): Promise<void> {
  await knex.schema.dropTableIfExists('merchants');
}
```

### Step 2.2: Write migrations — gateway_configs, transactions

**Create:** `packages/api/src/db/migrations/002_create_gateway_configs.ts`
**Create:** `packages/api/src/db/migrations/003_create_transactions.ts`

Transactions table includes future `subscription_id` and `invoice_id` columns (nullable FKs).

### Step 2.3: Write migrations — gateway_events, audit_logs, refunds

**Create:** `packages/api/src/db/migrations/004_create_gateway_events.ts`
**Create:** `packages/api/src/db/migrations/005_create_audit_logs.ts` — append-only, partitioned by month.
**Create:** `packages/api/src/db/migrations/006_create_refunds.ts`

### Step 2.4: Create seed script

**Create:** `packages/api/src/db/seeds/development.ts`

Creates:
- 1 test merchant (`test@paymensch.com` / `paymensch_test`)
- Live and test API keys (hashed in DB, shown as plaintext in console output)
- Sandbox gateway configs with test credentials from `.env`
- 20 sample transactions across all 3 gateways with various statuses

### Step 2.5: Wire migration and seed into npm scripts

**Modify:** `packages/api/package.json` — add `db:migrate`, `db:migrate:test`, `db:seed` scripts using Knex CLI.

**Create:** `packages/api/src/db/migrate.ts` — runs migrations programmatically for tests.

### Step 2.6: Add DB connection test

**Create:** `packages/api/src/__tests__/db/connection.test.ts` — verifies migrations run, tables exist, seed data loads.

### Step 2.7: Commit and present M2 summary

**Milestone 2 complete.** Human test:
```bash
npm run db:migrate          # All 6 migrations run clean
npm run db:seed             # Test merchant created, API key printed
npm test                    # DB connection test passes
psql -d paymensch_dev -c "SELECT * FROM merchants;"  # Real data
```

---

## Milestone 3: Payment Pipeline (eSewa Only)

**Goal:** Initiate → callback → verify → refund works end-to-end with eSewa sandbox. Zero dashboard. Zero other gateways. Just the pipeline, one gateway, working.

### Step 3.1: Write gateway adapter contract tests

**Create:** `packages/api/src/__tests__/gateways/contract.test.ts`

Shared test suite that every gateway adapter must pass:
```ts
export function testGatewayContract(name: string, adapter: PaymentGateway) {
  describe(`${name} gateway contract`, () => {
    it('initiatePayment returns a redirect URL', async () => { ... });
    it('initiatePayment fails gracefully with invalid credentials', async () => { ... });
    it('verifyPayment returns completed for a successful payment', async () => { ... });
    it('verifyPayment throws GatewayError for invalid reference', async () => { ... });
    it('processRefund returns refund status', async () => { ... });
    it('healthCheck returns latency and status', async () => { ... });
    it('handles gateway timeout with GatewayError (retryable=true)', async () => { ... });
    it('handles malformed response from gateway', async () => { ... });
  });
}
```

### Step 3.2: Implement eSewa adapter

**Create:** `packages/api/src/gateways/interface.ts` — `PaymentGateway` interface.

**Create:** `packages/api/src/gateways/esewa.ts` — implements `PaymentGateway`:
```ts
export class EsewaAdapter implements PaymentGateway {
  readonly name = 'esewa';
  async initiatePayment(params: PaymentRequest): Promise<PaymentResponse> {
    // HMAC signature generation
    // POST to eSewa sandbox endpoint
    // Parse XML/JSON response
    // Return standardized PaymentResponse
  }
  async verifyPayment(reference: string): Promise<VerificationResponse> { ... }
  async processRefund(params: RefundRequest): Promise<RefundResponse> { ... }
  async healthCheck(): Promise<GatewayStatus> { ... }
}
```

### Step 3.3: Write eSewa adapter tests against real fixtures

**Create:** `packages/api/src/__tests__/gateways/esewa.test.ts`

Uses nock to intercept HTTP with real fixture responses:
```ts
import { testGatewayContract } from './contract.test';
import { EsewaAdapter } from '../../gateways/esewa';
import esewaSuccess from '../../../test-utils/fixtures/esewa-success.json';
import esewaFailure from '../../../test-utils/fixtures/esewa-failure.json';

testGatewayContract('eSewa', new EsewaAdapter(testConfig));
```

**Create:** `test-utils/fixtures/esewa-success.json` — real eSewa sandbox response.
**Create:** `test-utils/fixtures/esewa-failure.json` — real eSewa error response.

### Step 3.4: Write pipeline stage tests

**Create:** `packages/api/src/__tests__/pipeline/validate.test.ts` — tests JSON schema validation via Fastify schemas.
**Create:** `packages/api/src/__tests__/pipeline/auth.test.ts` — tests API key extraction, hashing comparison, merchant lookup.
**Create:** `packages/api/src/__tests__/pipeline/rate-limit.test.ts` — tests Redis-backed rate limiting (100 req/min, burst 200).
**Create:** `packages/api/src/__tests__/pipeline/idempotency.test.ts` — tests concurrent requests with same key, cache TTL, lock per merchant.

### Step 3.5: Implement pipeline stages

**Create:** `packages/api/src/pipeline/validate.ts`
**Create:** `packages/api/src/pipeline/auth.ts`
**Create:** `packages/api/src/pipeline/rate-limit.ts`
**Create:** `packages/api/src/pipeline/idempotency.ts`
**Create:** `packages/api/src/pipeline/route.ts` — strategy pattern, picks gateway by merchant config priority.

### Step 3.6: Write payment route tests

**Create:** `packages/api/src/__tests__/routes/payments.test.ts`

```ts
describe('POST /v1/payments/initiate', () => {
  it('returns 200 with redirect URL on valid request', async () => {
    const res = await app.inject({
      method: 'POST', url: '/v1/payments/initiate',
      headers: { authorization: `Bearer ${testApiKey}` },
      payload: { amount_subunits: 150000, currency: 'NPR', order_id: 'TEST-1', callback_url: 'https://example.com/cb' }
    });
    expect(res.statusCode).toBe(200);
    expect(res.json().redirect_url).toBeTruthy();
  });
  it('returns 401 with invalid API key', async () => { ... });
  it('returns 400 with missing required fields', async () => { ... });
  it('returns 409 on duplicate idempotency key', async () => { ... });
  it('returns 429 when rate limit exceeded', async () => { ... });
});

describe('GET /v1/payments/:id/status', () => { ... });
describe('POST /v1/payments/:id/refund', () => { ... });
```

### Step 3.7: Implement payment routes

**Create:** `packages/api/src/routes/payments.ts` — `POST /initiate`, `GET /:id/status`, `POST /:id/refund`.

Wires pipeline: validate → auth → rate-limit → idempotency → route(esewa) → execute → audit → respond.

### Step 3.8: Write webhook receiver

**Create:** `packages/api/src/routes/webhooks.ts` — `POST /v1/webhooks/:gateway`

Verifies incoming gateway callbacks (HMAC for eSewa), updates transaction status, triggers merchant webhook delivery.

**Create:** `packages/api/src/__tests__/routes/webhooks.test.ts`

### Step 3.9: Implement webhook delivery to merchants

**Create:** `packages/api/src/services/webhook-delivery.ts` — sends POST to merchant webhook URL with HMAC signature. Retry with exponential backoff via BullMQ.

**Create:** `packages/api/src/queue/jobs/webhook-delivery.ts` — BullMQ job definition.

### Step 3.10: Write amount math utility tests

**Create:** `packages/api/src/__tests__/lib/amounts.test.ts`

```ts
describe('amount handling', () => {
  it('converts NPR float to subunits', () => {
    expect(toSubunits(1500.50, 'NPR')).toBe(150050);
  });
  it('converts JPY with 0 decimal places', () => {
    expect(toSubunits(500, 'JPY')).toBe(500);
  });
  it('rejects fractional subunits', () => {
    expect(() => toSubunits(10.555, 'NPR')).toThrow(ValidationError);
  });
  it('formats subunits back for display', () => {
    expect(formatSubunits(150050, 'NPR')).toBe('NPR 1,500.50');
  });
  it('adds amounts safely without floating point', () => {
    expect(addAmounts(100050, 50450)).toBe(150500);
  });
  it('handles all supported currencies', () => { ... });
});

// Property-based with fast-check
test.prop([float({ min: 0.01, max: 999999.99 }), currencyCode()])(
  'round-trip conversion is lossless for any valid amount',
  (amount, currency) => {
    const subunits = toSubunits(amount, currency);
    const back = fromSubunits(subunits, currency);
    expect(back).toBeCloseTo(amount, getDecimals(currency));
  }
);
```

**Create:** `packages/api/src/lib/amounts.ts` — implemented to pass the tests.

### Step 3.11: Write E2E test — full payment lifecycle

**Create:** `e2e/payment-flow.spec.ts`

```ts
test('eSewa sandbox payment completes end-to-end', async ({ request }) => {
  // 1. Initiate payment
  const initRes = await request.post('/v1/payments/initiate', { ... });
  expect(initRes.status()).toBe(200);
  const { id, redirect_url } = await initRes.json();

  // 2. Simulate gateway callback (we control the sandbox)
  const callbackRes = await request.post('/v1/webhooks/esewa', {
    transaction_id: id,
    status: 'completed',
    ...esewaCallbackPayload,
  });
  expect(callbackRes.status()).toBe(200);

  // 3. Verify payment status
  const statusRes = await request.get(`/v1/payments/${id}/status`);
  expect(await statusRes.json()).toMatchObject({ status: 'completed' });

  // 4. Refund
  const refundRes = await request.post(`/v1/payments/${id}/refund`, { amount_subunits: 50000 });
  expect(refundRes.status()).toBe(200);
});
```

### Step 3.12: Commit and present M3 summary

**Milestone 3 complete.** Human test:
```bash
# Start server, run sandbox payment:
curl -X POST http://localhost:4000/v1/payments/initiate \
  -H "Authorization: Bearer sk_test_xxx" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{"amount_subunits": 100000, "currency": "NPR", "order_id": "test-1", "callback_url": "https://example.com/cb"}'

# Check status:
curl http://localhost:4000/v1/payments/<id>/status -H "Authorization: Bearer sk_test_xxx"

# Test idempotency:
# Run same request twice with same Idempotency-Key → same result, one DB row
```

---

## Milestone 4: Remaining Gateways + Webhooks

**Goal:** Khalti and Fonepay work identically. Webhooks deliver reliably. Merchant webhook management.

### Step 4.1: Implement Khalti adapter (TDD)

**Create:** `packages/api/src/gateways/khalti.ts`
**Create:** `packages/api/src/__tests__/gateways/khalti.test.ts`
**Create:** `test-utils/fixtures/khalti-success.json`, `khalti-failure.json`

Passes the shared contract test suite from M3.

### Step 4.2: Implement Fonepay adapter (TDD)

**Create:** `packages/api/src/gateways/fonepay.ts`
**Create:** `packages/api/src/__tests__/gateways/fonepay.test.ts`
**Create:** `test-utils/fixtures/fonepay-soap-request.xml`, `fonepay-soap-response.xml`

Fonepay uses SOAP/XML — adapter handles XML parsing and wraps it in the standard interface.

### Step 4.3: Update routing strategy

**Modify:** `packages/api/src/pipeline/route.ts`

Priority routing: merchant sets gateway priority. Health-aware: if gateway health check fails, skip to next. Update tests.

### Step 4.4: Implement merchant webhook management

**Create:** `packages/api/src/routes/merchants.ts` — `GET /me`, `PATCH /me` (update webhook URL, secret).

### Step 4.5: Implement gateway config endpoints

**Create:** `packages/api/src/routes/gateways.ts` — `GET /`, `POST /`, `PATCH /:id`, `POST /:id/test`.

`POST /:id/test` sends a ping to the gateway with stored credentials and returns success/failure + latency.

### Step 4.6: E2E test — all three gateways

**Create:** `e2e/multi-gateway-flow.spec.ts` — tests routing, priority, fallback.

### Step 4.7: Commit and present M4 summary

**Milestone 4 complete.** Human test: test all three gateways via API, verify gateway config CRUD, test connection button works.

---

## Milestone 5: Merchant Dashboard

**Goal:** Full dashboard — login, transactions, gateways, API keys, analytics. Every component handles all 6 states.

### Step 5.1: Build auth (login, session, middleware)

**Create:** `packages/dashboard/src/app/login/page.tsx`
**Create:** `packages/dashboard/src/lib/auth.ts` — JWT + refresh token logic.
**Create:** `packages/dashboard/src/middleware.ts` — redirect unauthenticated to login.

### Step 5.2: Build layout shell

**Create:** `packages/dashboard/src/app/dashboard/layout.tsx` — sidebar nav, header with merchant name + plan tier, mobile hamburger.

### Step 5.3: Build Overview page

**Create:** `packages/dashboard/src/app/dashboard/page.tsx`

Shows: today's volume, success rate, gateway health cards, recent transactions summary.

### Step 5.4: Build Transactions page

**Create:** `packages/dashboard/src/app/dashboard/transactions/page.tsx`

Paginated table, filters (gateway, status, date range), sortable columns, detail drawer/modal per transaction.

**Create:** `packages/dashboard/src/components/TransactionTable.tsx` — handles loading skeleton, empty state, error state, single row edge case.

### Step 5.5: Build Gateways page

**Create:** `packages/dashboard/src/app/dashboard/gateways/page.tsx`

Per-gateway cards with status badges, configure button, test connection button.

**Create:** `packages/dashboard/src/components/GatewayConfigForm.tsx` — validates required fields per gateway type, shows connection test result.

### Step 5.6: Build API Keys page

**Create:** `packages/dashboard/src/app/dashboard/api-keys/page.tsx`

Shows live and test keys (masked by default), copy to clipboard, reveal on demand with confirmation, rotate with instant invalidation.

**Create:** `packages/dashboard/src/components/APIKeyCard.tsx` — all 6 states tested.

### Step 5.7: Build Webhooks page

**Create:** `packages/dashboard/src/app/dashboard/webhooks/page.tsx`

Set webhook URL, view recent deliveries with status (delivered/failed/pending), retry button.

### Step 5.8: Build Settings page

**Create:** `packages/dashboard/src/app/dashboard/settings/page.tsx` — business name, email, password change, plan tier display.

### Step 5.9: Connect dashboard to API

**Create:** `packages/dashboard/src/lib/api-client.ts` — typed fetch wrapper with auth headers, error handling, 401 → redirect to login.

Wire all pages to real API endpoints.

### Step 5.10: Write dashboard integration tests (each page)

**Create:** `packages/dashboard/src/__tests__/` — one test file per page, using React Testing Library + MSW for API mocking.

Each page test covers: loading state, loaded state, empty state, error state, edge case, submitting state.

### Step 5.11: Write dashboard E2E tests

**Create:** `e2e/dashboard-flow.spec.ts`

Full flow: sign up → login → view transactions → configure gateway → copy API key → logout → login → everything persists.

### Step 5.12: Accessibility audit

Run axe-core on every page. Fix all violations. Test keyboard navigation through full dashboard flow.

### Step 5.13: Commit and present M5 summary

**Milestone 5 complete.** Human test: full dashboard flow in browser, mobile responsive, all error states triggerable.

---

## Milestone 6: Marketing Site & Developer Docs

**Goal:** Landing, pricing, blog. VitePress docs with auto-generated API reference. One command generates docs.

### Step 6.1: Build marketing landing page

**Create:** `packages/dashboard/src/app/(marketing)/page.tsx` — hero, value proposition, CTA, "How it works" section.

### Step 6.2: Build pricing page

**Create:** `packages/dashboard/src/app/(marketing)/pricing/page.tsx` — three tiers, feature comparison table, FAQ accordion.

### Step 6.3: Build blog shell

**Create:** `packages/dashboard/src/app/(marketing)/blog/page.tsx` — list of posts.
**Create:** `packages/dashboard/src/app/(marketing)/blog/[slug]/page.tsx` — post detail, MDX support.

### Step 6.4: Build about page

**Create:** `packages/dashboard/src/app/(marketing)/about/page.tsx`

### Step 6.5: Scaffold VitePress docs package

```bash
mkdir -p packages/docs && cd packages/docs && npm init -y
npx vitepress init
```

**Create:** `packages/docs/package.json` — `@paymensch/docs`, depends on `@paymensch/shared`.

### Step 6.6: Write quickstart guide

**Create:** `packages/docs/src/quickstart.md` — 5-minute guide: get API key, make first payment, verify status.

### Step 6.7: Set up OpenAPI auto-generation

**Modify:** `packages/api/src/app.ts` — add `@fastify/swagger` and `@fastify/swagger-ui` plugins. Fastify auto-generates `openapi.json` from route schemas.

**Create:** `packages/docs/scripts/generate-api-docs.ts` — fetches `openapi.json` from running API, generates VitePress pages.

Add to root `package.json`:
```json
"docs:generate": "turbo docs:generate --filter=@paymensch/docs"
```

### Step 6.8: Write error code reference

**Create:** `packages/docs/src/errors/index.md` — every error code with status, message, cause, resolution steps.

### Step 6.9: Write gateway setup guides

**Create:** `packages/docs/src/guides/esewa-setup.md`
**Create:** `packages/docs/src/guides/khalti-setup.md`
**Create:** `packages/docs/src/guides/fonepay-setup.md`
**Create:** `packages/docs/src/guides/webhook-testing.md`
**Create:** `packages/docs/src/guides/idempotency.md`

### Step 6.10: Write SDK quickstarts

**Create:** `packages/docs/src/sdks/node.md` — Node.js quickstart with code examples.
**Create:** `packages/docs/src/sdks/python.md` — Python quickstart.

### Step 6.11: Commit and present M6 summary

**Milestone 6 complete.** Human test:
```bash
open http://localhost:3000          # Landing page
open http://localhost:3000/pricing  # Pricing page
open http://localhost:3000/blog     # Blog
npm run docs:dev                    # Docs on localhost:5173
```

---

## Milestone 7: Super Admin Dashboard

**Goal:** Separate Next.js app for Paymensch staff. Merchant management, impersonation, revenue, feature flags, audit log viewer.

### Step 7.1: Scaffold admin package

```bash
npx create-next-app@latest packages/admin --typescript --tailwind --app --src-dir
```

**Create:** `packages/admin/package.json` — `@paymensch/admin`, depends on `@paymensch/shared`.

### Step 7.2: Create admins table and auth

**Create:** `packages/api/src/db/migrations/007_create_admins.ts` — `admins` table with role, email, password_hash, totp_secret.
**Create:** `packages/api/src/routes/admin-auth.ts` — separate login endpoint, separate JWT secret.
**Create:** `packages/admin/src/lib/auth.ts` — admin auth client.

### Step 7.3: Build admin login page

**Create:** `packages/admin/src/app/login/page.tsx` — email + password + TOTP field (shown only if 2FA enabled).

### Step 7.4: Build admin layout shell

**Create:** `packages/admin/src/app/(authenticated)/layout.tsx` — sidebar nav, role-based menu visibility.

### Step 7.5: Build Merchants page

**Create:** `packages/admin/src/app/(authenticated)/merchants/page.tsx` — table with status/plan/date filters, approve/suspend/reactivate actions.
**Create:** `packages/admin/src/app/(authenticated)/merchants/[id]/page.tsx` — detail: profile, gateways, transaction summary, activity timeline, impersonate button.

### Step 7.6: Build impersonation flow

**Create:** `packages/admin/src/app/(authenticated)/merchants/[id]/impersonate/route.ts` — generates a time-limited JWT scoped to the merchant, redirects to dashboard with token.

### Step 7.7: Build Transactions page

**Create:** `packages/admin/src/app/(authenticated)/transactions/page.tsx` — global transaction view, advanced filters, manual refund capability, anomaly flag.

### Step 7.8: Build Revenue page

**Create:** `packages/admin/src/app/(authenticated)/revenue/page.tsx` — MRR, ARR, churn, plan distribution, gateway usage, new signups trend. Charts via Recharts or Tremor.

### Step 7.9: Build System Health page

**Create:** `packages/admin/src/app/(authenticated)/system-health/page.tsx` — gateway status (aggregated), queue depth/rate, DB pool, Redis memory, error trends.

### Step 7.10: Build Audit Log page

**Create:** `packages/admin/src/app/(authenticated)/audit-log/page.tsx` — filterable by merchant/entity/action/date, CSV/JSON export.

### Step 7.11: Build Feature Flags page

**Create:** `packages/admin/src/app/(authenticated)/settings/feature-flags/page.tsx` — toggle per merchant or globally, manage rollout %. Backend: `packages/api/src/services/feature-flags.ts`.

### Step 7.12: Build admin settings

**Create:** `packages/admin/src/app/(authenticated)/settings/page.tsx` — rate limit config, admin user management (add/remove/role change).

### Step 7.13: Commit and present M7 summary

**Milestone 7 complete.** Human test:
```bash
open http://localhost:4002          # Admin login
# Login as super_admin → merchants list → impersonate a merchant → view all pages
```

---

## Milestone 8: Monitoring, Audit, Polish

**Goal:** Dashboards live, alerts fire, audit logs queryable, everything production-ready.

### Step 8.1: Create Grafana dashboards as code

**Create:** `monitoring/grafana-dashboards/payment-overview.json`
**Create:** `monitoring/grafana-dashboards/gateway-health.json`
**Create:** `monitoring/grafana-dashboards/api-performance.json`
**Create:** `monitoring/grafana-dashboards/queue-health.json`
**Create:** `monitoring/grafana-dashboards/business-metrics.json`

### Step 8.2: Create Prometheus datasource and alerting config

**Create:** `monitoring/grafana-datasources.yml`
**Create:** `monitoring/prometheus-alerts.yml` — 5 alert rules from CLAUDE.md.
**Create:** `monitoring/prometheus.yml` — scrape config for paymensch-api:9090.
**Create:** `monitoring/loki-config.yml`
**Create:** `monitoring/promtail-config.yml`

### Step 8.3: Implement audit log service

**Create:** `packages/api/src/services/audit.ts` — `auditLog()` function called inline within services.

**Create:** `packages/api/src/__tests__/services/audit.test.ts` — verifies append-only, no UPDATE/DELETE works.

### Step 8.4: Add Sentry integration

**Create:** `packages/api/src/plugins/sentry.ts` — Fastify plugin, captures unhandled errors.

### Step 8.5: Performance hardening

- [ ] Connection pooling: Knex pool min=2 max=20 idle timeout 10s
- [ ] Redis connection pooling: ioredis with max retries
- [ ] API response compression: Fastify compress plugin
- [ ] Static asset caching: CDN headers for dashboard, docs, admin assets
- [ ] Cold start <2 seconds

### Step 8.6: Final test pass

- [ ] Full test suite: `npm test` — all packages, all tests pass
- [ ] Full E2E suite: `npm run test:e2e` — Chromium + Firefox pass
- [ ] Coverage: 100% on gateways/, services/. 90%+ overall.
- [ ] `npm run setup` on fresh clone — one command, everything works
- [ ] Grafana dashboards load at localhost:3001, show real metrics
- [ ] No console warnings or errors
- [ ] Accessibility audit passes on all sites
- [ ] Never logs API keys or credentials at any log level

### Step 8.7: Commit and present M8 summary

**Milestone 8 complete.** Platform is production-ready across all four sites.

---

## File Manifest

```
paymensch/
├── CLAUDE.md
├── docs/
│   ├── SPEC.md
│   └── BUILD_PLAN.md
├── packages/
│   ├── shared/
│   │   └── src/
│   │       ├── types.ts
│   │       ├── schemas.ts
│   │       ├── constants.ts
│   │       └── errors.ts
│   ├── api/
│   │   └── src/
│   │       ├── server.ts
│   │       ├── app.ts
│   │       ├── config.ts
│   │       ├── logger.ts
│   │       ├── metrics.ts
│   │       ├── plugins/
│   │       │   └── sentry.ts
│   │       ├── routes/
│   │       │   ├── payments.ts
│   │       │   ├── webhooks.ts
│   │       │   ├── merchants.ts
│   │       │   └── gateways.ts
│   │       ├── pipeline/
│   │       │   ├── validate.ts
│   │       │   ├── auth.ts
│   │       │   ├── rate-limit.ts
│   │       │   ├── idempotency.ts
│   │       │   └── route.ts
│   │       ├── gateways/
│   │       │   ├── interface.ts
│   │       │   ├── esewa.ts
│   │       │   ├── khalti.ts
│   │       │   └── fonepay.ts
│   │       ├── services/
│   │       │   ├── transaction.ts
│   │       │   ├── webhook-delivery.ts
│   │       │   └── audit.ts
│   │       ├── lib/
│   │       │   └── amounts.ts
│   │       ├── db/
│   │       │   ├── client.ts
│   │       │   ├── migrate.ts
│   │       │   ├── migrations/
│   │       │   │   ├── 001_create_merchants.ts
│   │       │   │   ├── 002_create_gateway_configs.ts
│   │       │   │   ├── 003_create_transactions.ts
│   │       │   │   ├── 004_create_gateway_events.ts
│   │       │   │   ├── 005_create_audit_logs.ts
│   │       │   │   └── 006_create_refunds.ts
│   │       │   └── seeds/
│   │       │       └── development.ts
│   │       └── queue/
│   │           └── jobs/
│   │               └── webhook-delivery.ts
│   ├── dashboard/
│   │   └── src/
│   │       └── app/
│   │           ├── (marketing)/
│   │           │   ├── page.tsx (Landing)
│   │           │   ├── pricing/
│   │           │   ├── blog/
│   │           │   └── about/
│   │           ├── (auth)/
│   │           │   └── login/
│   │           └── (dashboard)/
│   │               ├── page.tsx (Overview)
│   │               ├── transactions/
│   │               ├── gateways/
│   │               ├── api-keys/
│   │               ├── webhooks/
│   │               └── settings/
│   ├── docs/
│   │   ├── src/
│   │   │   ├── quickstart.md
│   │   │   ├── api/
│   │   │   ├── sdks/
│   │   │   ├── guides/
│   │   │   └── errors/
│   │   └── scripts/
│   │       └── generate-api-docs.ts
│   └── admin/
│       └── src/
│           └── app/
│               ├── login/
│               └── (authenticated)/
│                   ├── merchants/
│                   ├── transactions/
│                   ├── revenue/
│                   ├── system-health/
│                   ├── audit-log/
│                   └── settings/
├── test-utils/
│   ├── factories.ts
│   ├── mocks.ts
│   └── fixtures/
│       ├── esewa-success.json
│       ├── esewa-failure.json
│       ├── khalti-success.json
│       ├── khalti-failure.json
│       ├── fonepay-soap-request.xml
│       └── fonepay-soap-response.xml
├── e2e/
│   ├── health.spec.ts
│   ├── payment-flow.spec.ts
│   ├── multi-gateway-flow.spec.ts
│   └── dashboard-flow.spec.ts
├── monitoring/
│   ├── prometheus.yml
│   ├── prometheus-alerts.yml
│   ├── loki-config.yml
│   ├── promtail-config.yml
│   ├── grafana-datasources.yml
│   └── grafana-dashboards/
│       ├── payment-overview.json
│       ├── gateway-health.json
│       ├── api-performance.json
│       ├── queue-health.json
│       └── business-metrics.json
├── docker-compose.yml
├── turbo.json
├── vitest.config.ts
├── playwright.config.ts
├── tsconfig.base.json
├── .env.example
└── package.json
```

## Build Time Estimate

| Milestone | Steps | ~Hours |
|-----------|-------|--------|
| M1: Scaffold & Testing | 1.1–1.7 | 4 |
| M2: Database & Types | 2.1–2.7 | 3 |
| M3: Payment Pipeline | 3.1–3.12 | 10 |
| M4: Remaining Gateways | 4.1–4.7 | 8 |
| M5: Merchant Dashboard | 5.1–5.13 | 12 |
| M6: Marketing Site & Docs | 6.1–6.11 | 8 |
| M7: Super Admin Dashboard | 7.1–7.13 | 10 |
| M8: Monitoring & Polish | 8.1–8.7 | 6 |
| **Total** | **80 steps** | **~61 hours** |
