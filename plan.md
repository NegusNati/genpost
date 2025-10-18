# GenImage Implementation Plan (Monorepo, PWA-first) — ENHANCED

This plan translates the PRD into an executable, phased roadmap with production-grade best practices. It adopts a pnpm + Turborepo monorepo, React + Vite frontend, Hono backend, Drizzle ORM, Chapa billing (ETB), Google Gemini for image generation, local disk storage (self‑hosted via Docker Compose), and PWA + SEO best practices using TanStack Router with SSR for the marketing site (no Next.js).

**Enhanced with**: Redis caching, BullMQ job queue, OpenTelemetry observability, circuit breakers, advanced security, comprehensive testing strategy, performance optimization, and enterprise-grade monitoring.

References: prd.md:1, prd.md:35, prd.md:113

## 🎯 Success Metrics & SLOs

### Performance SLOs
- **Frontend**: LCP < 1.5s, FID < 100ms, CLS < 0.1, TTI < 2s
- **Backend**: p95 < 300ms (excluding Gemini), p99 < 1s
- **Availability**: 99.9% uptime (43min/month downtime budget)
- **Reliability**: Error rate < 0.1%, no data loss

### Quality Metrics
- **Test Coverage**: 80%+ line, 90%+ critical paths, 80%+ mutation score
- **Security**: 0 critical/high vulnerabilities, WCAG 2.1 AA compliant
- **Performance Budgets**: JS < 200KB, CSS < 50KB, FCP < 1.5s (CI enforced)


## 🏗️ Enhanced Architecture Overview

### Core Technology Stack
- **Frontend**: React 18 + Vite + TanStack Router + Tailwind CSS + Shadcn UI
- **Backend**: Hono (Node.js) + TypeScript strict mode
- **Database**: PostgreSQL 16 + Drizzle ORM + pgBouncer connection pooling
- **Caching**: Redis 7 (sessions, rate limiting, multi-tier cache)
- **Queue**: BullMQ (async jobs: generation, PDFs, emails, cron)
- **Observability**: OpenTelemetry + Jaeger (traces) + Prometheus (metrics) + Grafana (dashboards) + Sentry (errors)
- **Auth**: Better Auth (email/password + Google OAuth)
- **Payments**: Chapa (ETB currency, monthly subscriptions + add-ons)
- **AI**: Google Gemini via `@google/genai` SDK (image generation + alt text)
- **Storage**: Local disk (self-hosted) with nginx reverse proxy
- **Infrastructure**: Docker Compose + nginx-proxy-manager (Let's Encrypt TLS)
- **Monorepo**: pnpm workspaces + Turborepo (build caching)

### New Infrastructure Components (Phase 0.5)
- **Redis**: Multi-tier caching (L1 memory, L2 Redis, L3 DB), session store, rate limiting counters, pub/sub for real-time features
- **BullMQ**: Job queue for generation requests, PDF generation, email sending, credit resets; dead letter queue for failures
- **pgBouncer**: Connection pooling (transaction mode, 20 connections) to prevent DB exhaustion under load
- **OpenTelemetry**: Distributed tracing across HTTP requests, DB queries, external APIs, queue jobs; W3C Trace Context propagation
- **Jaeger**: Trace visualization and analysis (spans, latencies, dependencies)
- **Grafana**: Custom dashboards for API latency (p50/p95/p99), error rates, queue metrics, cache hit ratio, credit consumption
- **Sentry**: Error tracking with source maps, release tracking, user context, breadcrumbs, performance monitoring

### Architecture Layers
```
┌─────────────────────────────────────────────────────────────┐
│                      Client Layer (PWA)                      │
│  React + TanStack Router + Service Worker + Web Vitals RUM  │
│  Code Splitting • Virtual Scrolling • Prefetching • MSW     │
└─────────────────────────────────────────────────────────────┘
                            ↓ HTTPS (TLS 1.3)
┌─────────────────────────────────────────────────────────────┐
│              nginx-proxy-manager (Reverse Proxy)            │
│  Let's Encrypt • HTTP/2 • Gzip • Rate Limiting • WAF        │
└─────────────────────────────────────────────────────────────┘
            ↓                    ↓                    ↓
┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
│   apps/site      │  │   apps/web       │  │   apps/api       │
│  (SSR + SEO)     │  │   (PWA Static)   │  │  (Hono Server)   │
│  Vite SSR + Hono │  │   Vite Build     │  │  REST API v1     │
└──────────────────┘  └──────────────────┘  └──────────────────┘
                                                      ↓
                             ┌────────────────────────────────┐
                             │   Middleware Layer             │
                             │  Auth • Rate Limit • Circuit   │
                             │  Breaker • Cache • Observ.     │
                             └────────────────────────────────┘
                                      ↓
                ┌─────────────────────┴─────────────────────┐
                ↓                     ↓                     ↓
      ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
      │  PostgreSQL 16   │  │    Redis 7       │  │    BullMQ        │
      │  via pgBouncer   │  │  Cache + Session │  │  Job Queues      │
      │  Drizzle ORM     │  │  Rate Limiting   │  │  Workers         │
      └──────────────────┘  └──────────────────┘  └──────────────────┘
                                                            ↓
                                               ┌────────────────────┐
                                               │  External Services │
                                               │  • Google Gemini   │
                                               │  • Chapa Payments  │
                                               │  • Email Provider  │
                                               └────────────────────┘
                             
┌─────────────────────────────────────────────────────────────┐
│                    Observability Layer                       │
│  OpenTelemetry → Jaeger (traces) + Prometheus (metrics)     │
│  Sentry (errors) + Grafana (dashboards) + Pino (logs)       │
└─────────────────────────────────────────────────────────────┘
```

### Key Design Decisions
1. **Async Processing**: BullMQ for long-running operations (image generation) prevents HTTP timeout issues
2. **Circuit Breaker**: Protect against Gemini API failures with automatic fallback and retry
3. **Multi-Tier Caching**: L1 (memory) for hot data, L2 (Redis) for distributed, L3 (DB) as source of truth
4. **Request Deduplication**: Prevent duplicate generations within 5s window (same user + prompt + style)
5. **Graceful Degradation**: System remains partially functional when Redis/Gemini unavailable
6. **Connection Pooling**: pgBouncer prevents DB connection exhaustion under high load
7. **Distributed Tracing**: End-to-end request tracking for debugging and performance optimization
8. **Progressive Enhancement**: PWA works offline with queued operations synced when online
9. **Provider Interface**: All Gemini interactions flow through the shared `@google/genai` client to centralize retries, safety controls, and telemetry


## Phase 0 — Foundations & Monorepo Setup

Definition of Done
- Monorepo initialized with workspaces, build pipelines, code standards, CI.
- Shared configs/packages extracted (tsconfig, eslint, env, ui primitives, db).

Tasks
- [ ] Toolchain & package manager
  - [ ] Enable `corepack` and pin Node `>=20.x`, pnpm `>=9.x`.
  - [ ] Initialize Turborepo (`turbo.json`) with pipelines: `lint`, `typecheck`, `build`, `test`, `e2e`.
  - [ ] Configure `pnpm-workspace.yaml` for `apps/*` and `packages/*`.
  - [ ] Add `@google/genai` to workspace deps; create smoke script (mocked provider) verifying SDK wiring.
  - [ ] Bootstrap root `package.json` with workspace scripts (`pnpm -w lint|typecheck|test|build|e2e`).
  - [ ] Configure `.npmrc`/`.pnpmfile.cjs` to enforce pnpm version and frozen lockfile in CI.
  - [ ] Verify toolchain by running `pnpm -w exec node -v`, `pnpm -w dlx turbo run lint --dry` after initialization.
- [ ] Repo hygiene
  - [ ] Add `.editorconfig`, `.gitignore`, `LICENSE` (if applicable), `CODEOWNERS`.
  - [ ] Conventional commits (`@commitlint/config-conventional`) and `changesets` for versioning.
  - [ ] Pre-commit hooks via `husky` + `lint-staged` (run `eslint --fix`, `prettier --check`, `typecheck`).
  - [ ] Automate Husky install via `prepare` script; ensure `lint-staged` config covers TS/JSON/MD.
  - [ ] Add `renovate.json` or Dependabot config for dependency updates (npm + GitHub Actions).
  - [ ] Verify hooks by crafting dummy commit to confirm lint-staged + commitlint enforcement.
- [ ] TypeScript & linting
  - [ ] Strict TypeScript across repo (`noImplicitAny`, `exactOptionalPropertyTypes`).
  - [ ] `packages/tsconfig` with `base.json` + app-specific extends; path aliases.
  - [ ] ESLint (flat config) + Prettier; import/order rules; no default exports.
  - [ ] Configure `tsconfig.base.json` with incremental builds, `paths`, and DOM/node lib separation per app.
  - [ ] Set up `tsc --build` project references (root `tsconfig.json`) to support incremental compilation.
  - [ ] ESLint flat config extends internal presets (typescript, react, security); enable `eslint-plugin-security`, `eslint-plugin-promise`, `eslint-plugin-import`.
  - [ ] Integrate Prettier via `eslint-config-prettier`; add formatting script `pnpm -w format`.
  - [ ] Add `types/` directory for global ambient types (e.g., PWA service worker) referenced via `typeRoots`.
- [ ] Environment management
  - [ ] `packages/env` with Zod-validated schemas for server and client.
  - [ ] Separate `.env.example` for root, `apps/api`, `apps/web`.
  - [ ] Add Gemini-related env vars (`GOOGLE_GENAI_API_KEY`, default model, safety settings) consumed by SDK.
  - [ ] Enforce runtime validation: throw on missing/invalid env in bootstrap; provide `validateEnv()` helper per app.
  - [ ] Document secret storage process (1Password/Vault) and `.envrc` template for direnv (optional).
  - [ ] Add `.env.test` schema ensuring deterministic values for integration tests (no real API keys).
 - [ ] CI/CD (GitHub Actions)
  - [ ] Concurrency: cancel in-progress runs per branch/PR (`ci-${{ github.ref }}`).
  - [ ] Setup: Node 20.x, pnpm, `actions/cache` for pnpm store and Turbo cache keyed by lockfile + OS.
  - [ ] Pipelines: `lint`, `typecheck`, `build`, `test` (unit), optional `e2e` with Postgres service and seeded DB.
  - [ ] Selective execution: Turbo affected-only on PRs; full on `main`.
  - [ ] Artifacts: upload coverage (lcov), build outputs, and test results; annotate ESLint/TS.
  - [ ] Security: least-privilege permissions; pinned action SHAs/versions; Dependabot for npm and Actions.
  - [ ] Workflows: `ci.yml` (PR), `main.yml` (push), `nightly.yml` (cron); reuse job matrix with shared setup composite action.
  - [ ] Secrets management: document required GitHub secrets (`POSTGRES_URL`, `SENTRY_DSN`, `CHAPA_KEY`, `GOOGLE_GENAI_API_KEY` placeholder) and ensure encrypted env usage.
  - [ ] Add status badges (CI, coverage) once pipelines green; configure branch protection to require CI.
- [ ] Documentation
  - [ ] `README.md` (root) explaining workspace, commands, contribution.
  - [ ] ADRs in `docs/adr/` for major architecture choices (monorepo, auth, ORM, payments).
  - [ ] ADR: Adopt `@google/genai` as the unified AI provider layer (retries, observability, safety policies).
  - [ ] Onboarding guide: include prerequisite installs, repo bootstrap steps, `docker-compose` services list.
  - [ ] CONTRIBUTING.md with coding standards, commit message style, review checklist.
  - [ ] Diagram README section linking to architecture overview and runbooks.


## Phase 0.5 — Critical Infrastructure (NEW)

Definition of Done
- Redis, BullMQ, pgBouncer, observability foundation operational. Performance monitoring baseline established.

This phase adds production-grade infrastructure components that should be in place before development begins.

Tasks
- [ ] Redis Setup
  - [ ] Add `redis:7-alpine` service to Docker Compose with persistent volume (`redis-data:/data`).
  - [ ] Configure Redis in `packages/cache`: session store, cache client, rate limiter.
  - [ ] Implement cache invalidation patterns (TTL + event-driven).
  - [ ] Add Redis connection health check to `/readyz` endpoint.
  - [ ] Create cache warming scripts for startup (plans, common queries).
- [ ] BullMQ Job Queue
  - [ ] Install `bullmq` + Redis connection in `apps/api`.
  - [ ] Create queue modules: `generation-queue`, `pdf-queue`, `email-queue`, `cron-queue`.
  - [ ] Implement job processors with retry logic (exponential backoff: 1s, 5s, 30s, 5m).
  - [ ] Add Bull Board dashboard at `/admin/queues` (admin-only, basic auth).
  - [ ] Worker separation strategy: run queue workers in dedicated containers for horizontal scaling.
  - [ ] Configure dead letter queue for failed jobs (max 3 retries).
- [ ] Database Connection Pooling
  - [ ] Add `pgbouncer:1.21` service to Docker Compose.
  - [ ] Configure transaction pooling mode (pool_size=20, max_client_conn=100).
  - [ ] Update Drizzle connection strings to use PgBouncer port (6432).
  - [ ] Add connection pool metrics to Prometheus (active, idle, waiting).
  - [ ] Test pooling under load (100 concurrent connections).
- [ ] Observability Foundation
  - [ ] Install `@opentelemetry/sdk-node` + auto-instrumentation packages.
  - [ ] Add Jaeger service to Compose (`jaegertracing/all-in-one:1.54`) for trace visualization.
  - [ ] Configure trace context propagation: HTTP headers (W3C Trace Context) + queue job metadata.
  - [ ] Create custom spans: DB queries, external API calls (Gemini), job processing, cache hits/misses.
  - [ ] Set up Sentry SDK with release tracking, source maps, user context, breadcrumbs.
  - [ ] Configure sampling: 100% errors, 10% success traces in prod.
- [ ] Performance Monitoring
  - [ ] Install `web-vitals` library for Real User Monitoring (RUM).
  - [ ] Send Web Vitals to Umami custom events: LCP, FID, CLS, TTFB, INP.
  - [ ] Add performance budgets to CI: fail if JS > 200KB, CSS > 50KB, FCP > 1.5s.
  - [ ] Set up Lighthouse CI with GitHub checks (mobile + desktop, fail if score < 90).
  - [ ] Create bundle size tracking: `vite-bundle-visualizer` reports uploaded to artifacts.
- [ ] Grafana Dashboards
  - [ ] Add `grafana:10` service to Compose with Prometheus data source.
  - [ ] Create dashboards: API latency (p50/p95/p99), error rates, queue metrics, cache hit ratio.
  - [ ] Set up alerting rules: error rate > 5%, disk > 80%, DB connections > 80%.


## Phase 1 — Repository Structure & Shared Packages

Definition of Done
- Folder structure created and documented. Shared code extracted. Bootstrapped builds.

Monorepo layout (proposed)
- apps/
  - web/ — React + Vite PWA client
  - api/ — Hono server (Node)
  - site/ — TanStack Router SSR marketing site (Vite SSR + Hono)
- packages/
  - ui/ — Shadcn+Tailwind components; a11y-first primitives
  - db/ — Drizzle ORM (schema, migrations, seed, client)
  - env/ — Zod-validated environment loaders (server/client)
  - config/ — shared ESLint, TS, Tailwind, PostCSS configs
  - auth/ — Better Auth adapters + shared types
  - i18n/ — translation resources, keys, helpers
  - shared/ — shared types (DTOs), utilities, API client
  - og/ — shared utilities for dynamic OG image composition (Satori + Resvg)
  - docs/ — invoice/receipt HTML templates, email templates, branding assets
  - ai/ — `@google/genai` client config, model presets, retry/safety policies (NEW)
  - cache/ — Redis client, caching decorators, multi-tier strategies (NEW)
  - queue/ — BullMQ queue definitions, job processors, retry policies (NEW)
  - observability/ — OpenTelemetry setup, custom metrics, tracing (NEW)
  - resilience/ — Circuit breakers, retry logic, request deduplication (NEW)

Tasks
- [ ] Scaffolding
  - [ ] Create `apps/web` (Vite, React, TS, TanStack Router, Tailwind, Shadcn UI, VitePWA).
  - [ ] Create `apps/api` (Hono, TS, pino logging).
  - [ ] Create `apps/site` (TanStack Router SSR with Vite SSR + Hono server; MDX content; dynamic OG endpoint).
- [ ] `packages/db` (Drizzle + Postgres)
  - [ ] Install `drizzle-orm`, `drizzle-kit`, Postgres driver (`pg`).
  - [ ] Add Drizzle config (`drizzle.config.ts`) for Postgres in dev and prod.
  - [ ] Export `db` client factory, transaction helpers, testing harness (optionally `testcontainers` for integration tests).
- [ ] `packages/auth`
  - [ ] Wrap Better Auth server + client integrations; shared user/session types; Google OAuth provider wiring.
- [ ] `packages/ui`
  - [ ] Establish design tokens, theme provider, dark/light, RTL readiness.
- [ ] `packages/i18n`
  - [ ] Base `react-i18next` setup, namespaces, ICU formatting, number/currency helpers.
- [ ] `packages/shared`
  - [ ] DTOs: `AuthDTO`, `CreditDTO`, `GenerationDTO`, `SubscriptionDTO`, `MediaDTO`, `PaymentDTO` (provider: `chapa`).
- [ ] `packages/config`
  - [ ] `eslint.config.mjs`, `tsconfig` presets, `tailwind.config.ts` extendable presets.
- [ ] `packages/ai` (NEW)
  - [ ] Install and wrap `@google/genai` with centralized configuration (API key, model ids, safety settings).
  - [ ] Expose typed clients for image and text generation with shared retry/backoff policies.
  - [ ] Emit OpenTelemetry spans/metrics and structured logging hooks around SDK calls.
  - [ ] Provide test utilities/mock adapters for offline development and CI.
- [ ] `packages/cache` (NEW)
  - [ ] Redis client factory with connection pooling and reconnection logic.
  - [ ] Cache decorators: `@Cacheable(key, ttl)`, `@CacheEvict(key)`, `@CacheUpdate(key)`.
  - [ ] Multi-tier caching: L1 (in-memory LRU) → L2 (Redis) → L3 (database).
  - [ ] Cache warming utilities for startup (preload plans, user profiles).
  - [ ] Cache invalidation patterns: event-driven + TTL-based.
- [ ] `packages/queue` (NEW)
  - [ ] BullMQ queue factory and typed job definitions.
  - [ ] Base processor class with error handling, logging, and metrics.
  - [ ] Retry policies: exponential backoff with jitter.
  - [ ] Job status tracking helpers (pending/active/completed/failed).
  - [ ] Queue health checks and monitoring helpers.
- [ ] `packages/observability` (NEW)
  - [ ] OpenTelemetry tracer factory with auto-instrumentation.
  - [ ] Custom metrics registry: `credit_balance_gauge`, `generation_success_rate`, `cache_hit_ratio`.
  - [ ] Log correlation middleware (inject trace IDs into Pino logs).
  - [ ] Performance monitoring utilities for Web Vitals.
  - [ ] Sentry integration helpers (user context, breadcrumbs, tags).
- [ ] `packages/resilience` (NEW)
  - [ ] Circuit breaker implementation using `opossum` (configurable thresholds).
  - [ ] Retry helpers with exponential backoff and max attempts.
  - [ ] Request deduplication: hash-based (user_id + prompt + style) with 5s window.
  - [ ] Graceful degradation patterns (fallback responses, cached results).
  - [ ] Timeout enforcement utilities (AbortController-based).


## Phase 2 — Data Model & Drizzle ORM

Definition of Done
- Drizzle schema, migrations, seed data exist. Credit ledger and subscription plans modeled. Tests green.

Core tables (Postgres)
- users: id, email, password_hash, display_name, avatar_url, locale, created_at, updated_at
- sessions: id, user_id, expires_at, created_at
- plans: id, code, name, monthly_price_cents, currency (ETB), monthly_credits
- subscriptions: id, user_id, plan_id, status, current_period_start, current_period_end, cancel_at_period_end
- credit_ledger: id, user_id, delta, reason, ref_type, ref_id, created_at (derived balance via sum)
- generation_requests: id, user_id, prompt, style, input_image_url, status, error_code, created_at, started_at, completed_at
- images: id, request_id, url, width, height, seed, metadata_json, created_at
- attachments: id, user_id, url, mime, size_bytes, created_at
- payments: id, user_id, amount_cents, currency (ETB), provider ("chapa"), type ("subscription"|"addon"), provider_ref, status, created_at
- audit_logs: id, actor_id, action, target_type, target_id, metadata_json, created_at
 - invoices: id, user_id, invoice_number, issue_date, due_date, currency (ETB), subtotal_cents, tax_cents, total_cents, status (draft|issued|paid|void), pdf_url, billing_name, billing_address, created_at
 - invoice_lines: id, invoice_id, product_code (PRO150|ADDON30), description, quantity, unit_amount_cents, total_amount_cents
 - receipts: id, payment_id, receipt_number, issue_date, currency (ETB), amount_cents, invoice_id (nullable), pdf_url, created_at

Tasks
- [ ] Define schema in `packages/db/src/schema.ts` with relations, enums, indexes.
- [ ] Add computed views (or queries) for credit balance; create drizzle query helpers.
- [ ] Write `drizzle-kit` migrations and seed script for dev users, plans.
- [ ] Add repository layer per module (users, credits, subscriptions, generation, media).
- [ ] Unit tests for repositories using testcontainers (Postgres) with transactional rollback.
- [ ] Migration strategy doc (versioned migrations), zero-downtime checklist.
- [ ] Seed data: plan `PRO150` (150 credits/month, 50 ETB); add-on `ADDON30` (30 credits, 10 ETB) as payments + ledger grants.
- [ ] Database Optimizations (NEW)
  - [ ] Composite indexes: `(user_id, created_at DESC)` on generation_requests, images, credit_ledger.
  - [ ] Partial indexes: `(status) WHERE status IN ('pending', 'processing')` on generation_requests.
  - [ ] GIN index: `(prompt) gin_trgm_ops` for fuzzy search (enable pg_trgm extension).
  - [ ] Materialized view: `user_credit_balances` with refresh trigger on credit_ledger writes.
  - [ ] Materialized view: `user_generation_stats` for analytics (hourly refresh via cron job).
  - [ ] Add BRIN index on `created_at` for audit_logs table (time-series optimization).
  - [ ] Table partitioning: `audit_logs` by month (retain 7 years for compliance).
- [ ] Soft Delete Pattern (NEW)
  - [ ] Add `deleted_at TIMESTAMP` column to users, generation_requests, images tables.
  - [ ] Create views excluding soft-deleted: `users_active`, `generation_requests_active`, `images_active`.
  - [ ] Repository methods: `softDelete()`, `restore()`, `hardDelete()`, `findIncludingDeleted()`.
  - [ ] Cron job: purge hard-deleted records after 30-day grace period.
- [ ] Query Performance (NEW)
  - [ ] Add `EXPLAIN ANALYZE` tests in repository test suite (fail if seq scan on large tables).
  - [ ] Implement query timeout: 5s for reads, 10s for writes (via Drizzle config).
  - [ ] Use prepared statements for frequently executed queries.
  - [ ] Add query performance logging: log queries > 500ms with EXPLAIN plan.


## Phase 3 — Backend API Skeleton (Hono)

Definition of Done
- Hono app boots locally. Versioned API `/v1`. Health, metrics, error handling, security headers, rate limits.

Module structure (feature-based)
- src/
  - app.ts (bootstrap), server.ts
  - modules/
    - auth/ (controller, service, repo, routes, validators)
    - users/
    - credits/
    - subscriptions/
    - generation/
    - media/
    - i18n/
    - notifications/
  - lib/ (db, logger, env, errors, middlewares, http)
  - tests/

Tasks
- [ ] App bootstrap
  - [ ] Hono server with global middlewares: request id, pino logger, error mapper, timing.
  - [ ] Helmet-like security headers, CORS, compression.
  - [ ] Rate limiting per IP and per user (credits endpoints stricter).
- [ ] Health & metrics
  - [ ] `/healthz` (liveness: basic health) and `/readyz` (readiness: DB + Redis connectivity).
  - [ ] `/metrics` (Prometheus format): request count, latency histograms, error rates, custom metrics.
- [ ] Error & validation
  - [ ] Zod schemas for all DTOs; typed responses; error codes list.
- [ ] Auth module (Better Auth)
  - [ ] Email/password, Google OAuth; sessions via httpOnly, secure cookies.
  - [ ] Password reset, email verification stubs.
- [ ] Users module
  - [ ] CRUD endpoints (self-service: get/update profile, locale, avatar).
- [ ] Credits module
  - [ ] Ledger write helper; get balance; invariants enforced in transaction.
- [ ] Subscriptions module (Chapa stub)
  - [ ] Plans listing from DB; Chapa checkout initialization; verify endpoint; webhook handler skeleton.
  - [ ] Recurring billing: confirm Chapa recurring support; if unavailable, implement monthly renewal reminders + one‑click pay links.
- [ ] Invoices module
  - [ ] Invoice creation on subscription purchase and add-on purchases; line items from plan/add-on catalog.
  - [ ] Numbering scheme (prefix + YYYYMM + sequence); prevent gaps; store counters in DB.
  - [ ] HTML → PDF rendering (choose: Playwright/Chromium or PDFKit ADR); store PDFs on `docs` volume and expose signed URLs.
  - [ ] Email delivery with attached PDF (transactional email provider stub; env-controlled send).
- [ ] Receipts module
  - [ ] Generate receipt on successful payment verification; link to invoice when present.
  - [ ] HTML → PDF rendering and email to user; admin re-issue endpoint.
- [ ] Generation module (stub)
  - [ ] `POST /generate` validates prompt, style; enqueue job; returns request id.
  - [ ] `GET /generate/:id` returns status and image URL when complete.
  - [ ] Use `packages/ai` `@google/genai` client for all Gemini calls; ensure circuit breaker wraps SDK invocations.
- [ ] Media module
  - [ ] Local disk storage with safe path join; MIME + size checks; virus scan hook (optional).
  - [ ] Static image server with range requests and caching headers (via reverse proxy).
- [ ] Contract tests with `supertest` and Jest.
- [ ] Enhanced Middleware (NEW)
  - [ ] **Resilience middleware**: Circuit breaker for Gemini API (5 failures → 30s open, 50% success → close).
  - [ ] **Request deduplication**: Hash(user_id + prompt + style), 5s window, return existing job_id if duplicate.
  - [ ] **Timeout enforcement**: 30s for generation endpoints, 10s for standard APIs, 5s for health checks.
  - [ ] **Caching middleware**: ETag generation (hash response body), support `If-None-Match` → 304.
  - [ ] **Rate limiting via Redis**: Token bucket algorithm (auth: 5/min/IP, generation: 10/min/user, global: 100/min/IP).
  - [ ] **Progressive auth delays**: 1s after 1st failure, 5s after 2nd, 30s after 3rd, require hCaptcha after 5th.
  - [ ] **Cache-Control headers**: per route type (auth: no-cache, generation status: max-age=5s, images: max-age=1y).
- [ ] OpenAPI Documentation (NEW)
  - [ ] Generate spec from Zod schemas using `@asteasolutions/zod-to-openapi`.
  - [ ] Serve Swagger UI at `/docs` (protected with basic auth in prod, public in dev).
  - [ ] Include rate limit info, error code definitions, and examples in spec.
  - [ ] Auto-generate TypeScript API client for frontend using `openapi-typescript-codegen`.
  - [ ] Version API spec (v1.0.0), include changelog for breaking changes.
- [ ] Generation Module Enhancement (NEW)
  - [ ] **Async via BullMQ**: `POST /generate` → enqueue job, return `job_id` immediately (200 Accepted).
  - [ ] **Job status polling**: `GET /generate/:id` → return status (pending/processing/completed/failed) + result.
  - [ ] **WebSocket real-time**: `WS /ws/generate/:id` → push status updates (optional alternative to polling).
  - [ ] **Image optimization pipeline**: On completion, generate WebP + AVIF formats + blurhash placeholder.
  - [ ] **Multi-size generation**: Create thumbnail (400px), medium (1024px), full resolution with lazy loading.
  - [ ] **Alt text generation**: Auto-generate accessibility description using Gemini text model.
- [ ] Security Enhancements (NEW)
  - [ ] **CSP nonce generation**: Inject nonce for inline scripts, strict CSP policy.
  - [ ] **Signed URLs**: For invoice/receipt PDFs (HMAC-SHA256, 1-hour expiry, owner-only access).
  - [ ] **Input sanitization**: HTML escape user-generated content, strip dangerous characters.
  - [ ] **Output encoding**: JSON responses properly escaped, prevent XSS in error messages.


## Phase 3b — Self‑Hosted Infra (Docker Compose & Local Storage)

Definition of Done
- Backend + storage run via Docker Compose with persistent volumes, reverse proxy, HTTPS.

Tasks
- [ ] Compose services: `api` (Node), `site` (SSR, Hono + Vite SSR), `web` (static PWA via nginx), `db` (Postgres 16), `redis` (7-alpine), `pgbouncer` (1.21), `proxy` (nginx-proxy-manager + DB), `images` volume, `docs` volume (PDFs), `umami` (analytics), `jaeger` (tracing), `grafana` (dashboards).
- [ ] nginx-proxy-manager
  - [ ] Add `nginx-proxy-manager` and its MariaDB service in Compose; configure Let’s Encrypt HTTP‑01 certificates.
  - [ ] Map vhosts: `genimage.com` → `site`; `app.genimage.com` → `web`; `api.genimage.com` → `api`.
- [ ] Reverse proxy
  - [ ] Enable HTTP/2, gzip/brotli, per‑route rate limiting, access logs.
  - [ ] Static file route `/media/*` served from images volume with caching, range requests (via `api` or separate minimal file server).
  - [ ] Protect `/admin` and `/invoices`/`/receipts` signed URLs; disable directory listing.
  - [ ] SSR catch‑all: forward `GET /*` (excluding `/media/*`, `/og`, `/sitemap.xml`, `/robots.txt`, assets) to `site` Hono SSR handler.
- [ ] Backups
  - [ ] Nightly volume snapshots for Postgres and images; restore runbook.
- [ ] CI deploy job to server (SSH or GitHub Actions runner) and `.env` management.
- [ ] Umami analytics
  - [ ] Deploy `umami` container connected to Postgres; create `umami` database/user.
  - [ ] Create website entries for `genimage.com` and `app.genimage.com`; set tracker script domain.
- [ ] Deployment Enhancements (NEW)
  - [ ] **Blue-Green deploys**: Use Docker labels (`environment=blue|green`), nginx upstream switching without downtime.
  - [ ] **Health check gates**: Wait for `/readyz` → 200 before routing traffic to new version.
  - [ ] **Graceful shutdown**: Handle SIGTERM (stop accepting requests → drain 30s → close connections).
  - [ ] **Queue worker scaling**: Run BullMQ workers in separate containers, scale independently from API.
  - [ ] **Rollback automation**: Monitor error rate for 5min post-deploy, auto-rollback if rate > 5%.
- [ ] Backup & DR (NEW)
  - [ ] **Automated backups**: Daily cron for Postgres (pg_dump), Redis (RDB), volumes (restic to S3-compatible).
  - [ ] **Backup verification**: Monthly automated restore to staging environment, validate data integrity.
  - [ ] **Retention policy**: 7 daily, 4 weekly, 12 monthly backups with lifecycle rules.
  - [ ] **Disaster recovery runbook**: RTO < 4h, RPO < 15min, document step-by-step restore process.
  - [ ] **Backup alerting**: Notify if backup fails or storage usage > 80%.


## Phase 4 — Frontend App Shell (Vite, PWA)

Definition of Done
- Web app boots with routing, theming, i18n, auth context, PWA manifest & SW.

Feature-based structure (client)
- src/
  - app/ (providers: theme, query, i18n, auth)
  - routes/ (TanStack Router route files)
  - features/
    - auth/ (login, signup, reset)
    - generation/
    - subscriptions/
    - billing/ (invoices, receipts, plans)
    - credits/
    - gallery/
    - profile/
    - i18n/
    - notifications/
  - components/ (UI: forms, modals, toasts)
  - lib/ (api client, query keys, analytics)
  - assets/, styles/

Tasks
- [ ] Vite setup with TS, Tailwind, shadcn/ui; design tokens and themes.
- [ ] VitePWA plugin: manifest (name, icons incl. maskable, theme color), basic service worker.
- [ ] react-i18next with namespaces: `common`, `auth`, `generation`, `billing` for en/am/om; locale switcher.
- [ ] Auth client wiring (Better Auth) + protected routes.
- [ ] Query library (TanStack Query) and API client with typed responses.
- [ ] A11y baseline: focus styles, semantic landmarks, keyboard nav.
- [ ] Billing UI shell: invoices list and receipts list pages with download actions.
- [ ] Performance Optimizations (NEW)
  - [ ] **Code splitting**: Route-based (all routes lazy via TanStack Router) + component-based (heavy components).
  - [ ] **Vendor chunking**: Separate chunks for React, UI library, utilities (better caching).
  - [ ] **Prefetching strategies**: Prefetch next route on hover, gallery next page on scroll proximity (90%).
  - [ ] **Preload critical assets**: Fonts (woff2), above-fold images, critical CSS.
  - [ ] **Virtual scrolling**: Use `@tanstack/react-virtual` for gallery (handles 10k+ items efficiently).
  - [ ] **Web Workers**: Move image processing (crop, resize, filters) off main thread.
  - [ ] **Suspense boundaries**: Strategic placement per route + heavy components (lazy image editor).
  - [ ] **Bundle analysis**: `vite-bundle-visualizer` in CI, fail if bundle size increase > 10%.
- [ ] Developer Experience (NEW)
  - [ ] **Storybook**: Setup with Vite, install a11y/viewport/dark mode addons.
  - [ ] **Component stories**: Create stories for all `packages/ui` components with variants.
  - [ ] **MSW (Mock Service Worker)**: Mock all API endpoints with realistic data for offline dev.
  - [ ] **Visual regression**: Percy/Chromatic integration for component screenshot testing.
  - [ ] **Dev tooling**: Install React DevTools, TanStack Query DevTools, Network inspector.
- [ ] Service Worker Strategy (NEW)
  - [ ] **App shell**: Precache HTML, CSS, JS (CacheFirst with cache busting on deploy).
  - [ ] **API calls**: NetworkFirst with 5s timeout, fallback to cache if offline.
  - [ ] **Images**: CacheFirst with LRU eviction (50MB quota), serve stale while revalidating.
  - [ ] **Avatars**: StaleWhileRevalidate (always show cached, update in background).
  - [ ] **Quota management**: Monitor storage usage, proactive cleanup when > 80% full.


## Phase 4b — Marketing Site (TanStack Router SSR + SEO)

Definition of Done
- Server‑rendered marketing pages with excellent SEO, dynamic OG, and internationalization using TanStack Router SSR (no Next.js).

Tasks
- [ ] Hono + Vite SSR server in `apps/site`; shared UI from `packages/ui`.
- [ ] Pages: `/`, `/features`, `/pricing`, `/faq`, `/blog` (MDX via `@mdx-js/mdx` or `mdx-bundler`), `/legal/privacy`, `/legal/terms`.
- [ ] SEO
  - [ ] Metadata per route (title/description/canonical/alternates) via `react-helmet-async`.
  - [ ] Sitemap.xml + robots.txt generator; hreflang for en/am/om.
  - [ ] Structured data (JSON‑LD) for Organization, Product, FAQ, BlogPosting.
- [ ] Dynamic OG
  - [ ] `GET /og` route using Satori + Resvg to render branded OG images (framework-agnostic replacement for @vercel/og).
  - [ ] Cache fonts and templates; validate on major platforms.
- [ ] Performance
  - [ ] Image optimization, font subsetting, route‑level caching where safe.
 - [ ] Analytics
   - [ ] Inject Umami script (env-controlled) for `genimage.com`; respect DNT and anonymize IP.

SSR Implementation (non‑streaming first, streaming optional)
- [ ] Router creation (shared)
  - [ ] `apps/site/src/router.tsx` exports `createRouter()` using `createRouter as createTanstackRouter` and the generated `routeTree`.
  - [ ] Add `@tanstack/router-plugin` for route tree codegen; commit `routeTree.gen.tsx`.
- [ ] Server render (non‑streaming)
  - [ ] `apps/site/src/entry-server.tsx` uses `createRequestHandler({ request, createRouter })` and `defaultRenderHandler` from `@tanstack/react-router/ssr/server`.
  - [ ] Wire Hono to call `render({ request })` and return the `Response` (Hono uses Web Request/Response, no adapters needed).
- [ ] Client hydration
  - [ ] `apps/site/src/entry-client.tsx` hydrates with `hydrateRoot(document, <RouterClient router={router} />)`.
- [ ] Streaming (optional, per‑route toggle)
  - [ ] Add alternate handler with `defaultStreamHandler` (or `renderRouterToStream` + `RouterServer`) behind env flag `SSR_STREAMING=1` for high‑latency pages.
- [ ] Data loading & serialization
  - [ ] Use TanStack Router loaders; only return serializable data (Dates, Errors, FormData are supported). Avoid Maps/Sets/BigInt in loader outputs.
  - [ ] For non‑serializable data, return IDs/URLs and fetch on client.
- [ ] Vite SSR config
  - [ ] `vite.config.ts` exposes SSR entry `src/entry-server.tsx` and client entry `src/entry-client.tsx`.
  - [ ] Dev: Vite SSR middleware; Prod: Node server created via Hono exporting a `fetch` handler.
- [ ] Error/fallbacks
  - [ ] Render 404/500 pages server‑side; never leak stack traces in prod; log with request id.
- [ ] Testing
  - [ ] SSR render test: `renderRouterToString` returns HTML containing `<title>` and canonical; snapshot per route.
  - [ ] E2E: verify hydration and SEO tags.


## Phase 5 — Authentication Flows

Definition of Done
- Email/password + Google OAuth working end-to-end. Password reset and email verification flows functional.

Tasks
- [ ] Screens: Login, Signup, Forgot Password, Reset Password, Verify Email.
- [ ] Server: token generation, expiry, secure cookie settings, CSRF protection where applicable.
- [ ] Rate-limit per account + per IP for auth endpoints.
- [ ] UI tests and server tests; error messages localized.


## Phase 6 — Credits & Subscription Billing (Chapa paused)

Definition of Done
- Monthly plan and on‑demand add‑ons tracked in the ledger; ETB prices displayed; Chapa endpoints scaffolded behind a feature flag and disabled until keys are provided. Admin console can grant credits for testing.

Tasks
- [ ] Plans API + UI (list, select plan).
- [ ] Chapa integration (paused)
  - [ ] Stub checkout init + verify endpoints; feature flag off by default.
  - [ ] Webhook handler skeleton with signature verification; idempotent store.
  - [ ] When enabled: map events → DB writes (subscriptions, payments, credit_ledger grants).
- [ ] Credit balance in header; “X left” component.
- [ ] Purchase add-on credits (small packages); confirm screens; receipts page.
- [ ] Billing error handling + retries; ETB formatting.
 - [ ] Monthly credit reset job (UTC midnight 1st) with proration rules and manual override.
 - [ ] Invoices & receipts
  - [ ] Invoice issuance on checkout init (draft) → on verification (issued/paid) with PDF.
  - [ ] Receipt generation from payment; PDF with receipt number; email to user.
  - [ ] Admin billing console: search users, reissue invoices/receipts, manual credit grants with document issuance.
  - [ ] Localized templates (en/am/om), ETB currency; date/number formatting per locale.
  - [ ] Storage: persist PDFs to `docs` volume; signed download links via API.


## Phase 7 — Image Generation (MVP)

Definition of Done
- Prompt-to-image with style presets; one credit per successful generation; progress/status polling.

Tasks
- [ ] Presets: Vintage, Celebration, Cartoonish, Cinematic, Meme & Trendy, Dreamy, Renaissance Art.
- [ ] Backend: Google Gemini image generation provider behind interface; API key in env; safety settings and rate limits.
  - [ ] Provider: `@google/genai` SDK; env `GOOGLE_GENAI_API_KEY`, `GEMINI_IMAGE_MODEL`; request timeouts/retries and exponential backoff.
  - [ ] Request lifecycle
  - [ ] Validate credits > 0; reserve credit; on success finalize; on failure release.
  - [ ] Persist request + result; store image URL and metadata.
- [ ] Frontend: prompt form, style selector, generate button, result view (download/share/edit).
- [ ] Auto-generate alt text/description for accessibility and sharing.


## Phase 8 — Attachments, History & Gallery

Definition of Done
- Image-to-image supported via attachment; gallery shows last 100 images per user with actions.

- Tasks
- [ ] Direct uploads to API; server stores to local disk (`/var/app/images`); size/type validation; optional malware scan.
- [ ] Generation requests accept `input_image_url`; backend passes to provider.
- [ ] Gallery route with filters and pagination; actions: download, re-generate, share.
- [ ] Data retention policy doc; deletion flow; export my data.


## Phase 9 — PWA Offline, Sync & Notifications

Definition of Done
- App works offline for browsing history and UI; queue generation requests while offline; push notifications for completion and low credits.

Tasks
- [ ] Service worker
  - [ ] Workbox or VitePWA strategies (precaching app shell, stale-while-revalidate for API reads, background sync for writes to `/generate`).
  - [ ] Cache images with quota management and cleanup.
- [ ] Background sync queue for generation requests; replay when online; optimistic UI.
- [ ] Web push (service worker) for generation complete, low credits, renewal reminders.
- [ ] “Add to Home Screen” UX and PWA validation (Lighthouse 95+).
- [ ] Analytics events (Umami) for key actions (prompt submit, success/failure, add-on purchase intent), respecting DNT.
- [ ] Advanced PWA Features (NEW)
  - [ ] **Share Target API**: Register PWA to receive shared images and text from OS share menu.
  - [ ] **Share handler**: Pre-fill generation form with shared content (image as input_image, text as prompt).
  - [ ] **Background Sync API**: Queue failed generation requests, auto-retry when online (exponential backoff).
  - [ ] **Push notifications with actions**: "View Image" → open gallery, "Share" → native share, "Generate Another" → new form.
  - [ ] **Notification customization**: Per-user settings (enable/disable by type), respect system DND.
  - [ ] **Periodic Background Sync**: Daily check for low credits (< 10), subscription renewal reminders (3 days before).
  - [ ] **Offline UI indicators**: Toast when offline, show sync queue status, disable unavailable features gracefully.
  - [ ] **Install prompt optimization**: Show after 2 successful generations or 3 visits, track dismissal (don't spam).
  - [ ] **App shortcuts**: Add app shortcuts in manifest (New Image, Gallery, Profile, Credits).


## Phase 10 — Localization & Accessibility

Definition of Done
- Full i18n coverage (English, Amharic, Oromo). Content, errors, legal pages localized. WCAG 2.1 AA (target AAA where feasible); screen reader tested.

Tasks
- [ ] Translate all namespaces; QA with native speakers; RTL checks for Amharic (if required).
- [ ] Currency/number/date formatting per locale.
- [ ] End-to-end i18n tests; language switch persists in profile.
- [ ] A11y audits (axe, Lighthouse); keyboard traps fixed; focus order verified.
- [ ] Enhanced Accessibility (NEW)
  - [ ] **WCAG 2.1 AAA targets**: Contrast ratio 7:1 (where feasible), enhanced focus indicators (3px outline).
  - [ ] **Automated a11y testing**: Integrate `axe-core` in Cypress E2E tests, fail on violations.
  - [ ] **Screen reader testing**: Manual testing checklist for NVDA (Windows) and VoiceOver (Mac/iOS).
  - [ ] **Keyboard shortcuts**: Global `Cmd/Ctrl+K` command palette, `G` shortcuts for navigation (gh, gg, gp = home/gallery/profile).
  - [ ] **Skip links**: "Skip to main content", "Skip to navigation" at page top.
  - [ ] **Focus management**: Trap focus in modals, restore focus on close, manage focus on route changes.
  - [ ] **ARIA live regions**: Announce dynamic content changes (generation complete, credit update).
  - [ ] **Reduced motion**: Respect `prefers-reduced-motion`, disable animations for accessibility.
  - [ ] **High contrast mode**: Support Windows High Contrast Mode via CSS custom properties.


## Phase 11 — Observability, Security & Hardening

Definition of Done
- Centralized logs, metrics, tracing; security baselines in place; backups configured.

Tasks
- [ ] Observability
  - [ ] Pino logs with request ids; log redaction of PII.
  - [ ] Prometheus metrics (requests, latency, generation success rate); uptime alerts.
  - [ ] Instrument `@google/genai` client usages (latency, model, quota) via OpenTelemetry and export to Prometheus/Grafana dashboards.
- [ ] Security
  - [ ] CSP headers, `X-Frame-Options`, `Referrer-Policy`, `Permissions-Policy`.
  - [ ] Input validation everywhere (Zod); output encoding; file scanning.
  - [ ] Rate limits per route group; account lockout after repeated failures.
  - [ ] Secrets via platform KMS; no secrets in repo; rotate keys policy.
  - [ ] Document signing: signed, expiring URLs for invoice/receipt PDFs; owner-only access; admin override with audit.
- [ ] Data
  - [ ] Daily backups (DB + storage metadata); restore runbook; retention policy.
- [ ] OWASP Top 10 Mitigation (NEW)
  - [ ] **A01 Broken Access Control**: RBAC middleware (user/admin roles), resource ownership checks on mutations, signed PDF URLs.
  - [ ] **A02 Cryptographic Failures**: Docker secrets (not .env), bcrypt cost 12, quarterly API key rotation policy documented.
  - [ ] **A03 Injection**: Parameterized queries (Drizzle enforced), Zod validation on all inputs, HTML escape user content.
  - [ ] **A05 Security Misconfiguration**: Strict CSP with nonces, remove X-Powered-By, nginx directory listing off.
  - [ ] **A07 Auth Failures**: Account lockout (5 failures → 15min), session timeout (7d idle, 30d absolute), 2FA consideration.
  - [ ] **A08 Data Integrity**: File upload validation (MIME + magic bytes), size limits, optional ClamAV scanning.
  - [ ] **A09 Logging Failures**: PII redaction in logs, audit trail for billing, log retention 90 days.
  - [ ] **A10 SSRF**: Validate/allowlist external URLs, no user-controlled redirect destinations.
- [ ] Security Scanning (NEW)
  - [ ] **Pre-commit**: `gitleaks` for secret scanning (block commits with secrets).
  - [ ] **CI pipeline**: `npm audit` + Snyk scan (fail on high/critical vulnerabilities).
  - [ ] **Runtime**: Dependabot alerts enabled, CodeQL analysis for JS/TS.
  - [ ] **Quarterly**: Manual penetration testing, vulnerability disclosure program consideration.
- [ ] Data Privacy & Compliance (NEW)
  - [ ] **GDPR compliance**: Right to erasure (`DELETE /users/me`), data portability (`GET /users/me/export`).
  - [ ] **Consent management**: Track privacy policy version acceptance, cookie consent banner.
  - [ ] **Data retention**: Auto-delete generations > 1 year (configurable per plan), 30-day soft-delete grace period.
  - [ ] **Anonymization**: Use Faker.js for test data, scrub PII from logs and error messages.


## Phase 12 — QA, Performance & Launch

Definition of Done
- Test suites pass, performance budgets met, deployments configured, beta launched.

Tasks
- [ ] Testing
  - [ ] Unit: 80%+ coverage; E2E (Cypress) for top flows (auth, generate, subscribe).
  - [ ] Load tests for `/generate` path; graceful degradation when provider slow.
- [ ] Performance
  - [ ] Code-splitting, prefetching critical routes; image lazy loading; TTI < 2s.
  - [ ] API p95 < 500ms (excluding provider time). Caching for GETs.
- [ ] Enhanced Testing Strategy (NEW)
  - [ ] **Contract testing**: Pact consumer tests (web), provider verification (api), contract versioning.
  - [ ] **Visual regression**: Percy/Chromatic snapshots for all Storybook stories, fail CI if diff > 0.5%.
  - [ ] **Mutation testing**: Stryker.js to verify test quality, target 80%+ mutation score.
  - [ ] **Performance regression**: Lighthouse CI on every PR, fail if LCP > 2.5s or score drops > 5 points.
  - [ ] **Load testing**: k6 scripts for auth (50 login/s), generation (100 concurrent), gallery (200 req/s).
  - [ ] **Chaos engineering**: Toxiproxy scenarios (Gemini 5s latency, Redis disconnect, DB slow queries).
  - [ ] **Smoke tests**: Post-deploy validation (login → generate → download → logout) in prod.
  - [ ] **E2E CI**: Run Cypress on every PR with Postgres service, seeded data, parallel execution.
- [ ] Deployment
  - [ ] apps/site (SSR, TanStack Router) → self‑hosted via Docker Compose; domain `genimage.com`.
  - [ ] apps/web (PWA) → static hosting/CDN or self‑hosted behind proxy; domain `app.genimage.com`.
  - [ ] apps/api → self‑hosted via Docker Compose; domain `api.genimage.com`.
  - [ ] Domain, TLS, redirects, `robots.txt`, `sitemap.xml` on SSR site.
- [ ] Beta rollout
  - [ ] Feature flags for risky features; analytics funnels; feedback form.
- [ ] Pre-Launch Checklist (NEW)
  - [ ] **Performance**: Lighthouse score ≥ 95 (mobile + desktop), LCP < 1.5s, TTI < 2s, API p95 < 300ms.
  - [ ] **Accessibility**: axe-core 0 violations, WCAG 2.1 AA compliant, manual screen reader testing complete.
  - [ ] **Security**: Penetration test report clean, vulnerability scan 0 high/critical, OWASP Top 10 mitigation verified.
  - [ ] **Compliance**: GDPR audit complete (data export/deletion/consent working), privacy policy published.
  - [ ] **Monitoring**: Alerts configured (error rate, disk, DB), on-call rotation established, runbooks documented.
  - [ ] **Disaster Recovery**: Backup restore drill completed, RTO < 4h and RPO < 15min verified.
  - [ ] **Load testing**: System handles target load (100 RPS generation, 1000 concurrent users), auto-scaling tested.
  - [ ] **Documentation**: API docs published, user guides complete, admin runbooks ready, incident response plan.
  - [ ] **Legal**: Terms of service published, cookie policy, GDPR privacy notice, content policy for AI generations.


## Backend API Surface (v1) — Initial

- Auth: `POST /v1/auth/login`, `POST /v1/auth/signup`, `POST /v1/auth/logout`, `POST /v1/auth/password/reset`, `POST /v1/auth/password/confirm`, `GET /v1/auth/session`.
- Users: `GET /v1/users/me`, `PATCH /v1/users/me` (display_name, locale, avatar).
- Credits: `GET /v1/credits/me`, `GET /v1/credits/ledger`.
- Plans: `GET /v1/plans`.
- Subscriptions: `POST /v1/subscriptions/checkout`, `POST /v1/webhooks/chapa`.
 - Invoices: `GET /v1/billing/invoices`, `GET /v1/billing/invoices/:id`, `GET /v1/billing/invoices/:id/pdf`.
 - Receipts: `GET /v1/billing/receipts`, `GET /v1/billing/receipts/:id/pdf`.
- Generation: `POST /v1/generate`, `GET /v1/generate/:id`.
- Media: `POST /v1/media/sign`.
- Health: `GET /healthz`, Metrics: `GET /metrics`.
 
Site SSR server
- OG Images: `GET /og` (dynamic OG for marketing routes)


## Frontend Routes (TanStack Router)

- `/` — Home/Generate
- `/auth/login`, `/auth/signup`, `/auth/reset`
- `/profile` — profile + language
- `/billing` — plan selection, receipts
- `/billing/invoices` — list + view/download
- `/billing/receipts` — list + download
- `/gallery` — history
- `/legal/privacy`, `/legal/terms`

Site (TanStack Router SSR)
- `/` (landing), `/features`, `/pricing`, `/faq`, `/blog`, `/legal/privacy`, `/legal/terms`
- `/og` (dynamic OG images via Satori + Resvg)


## Non-Functional Requirements Mapping

- Performance: code-splitting, caching, DB indexes, provider timeouts + retries, CDN or reverse‑proxy caching for images; SSR TTFB optimizations.
- Security: HTTPS everywhere, secure cookies, CSRF where needed, CSP, validation.
- Scalability: stateless api; background jobs optional later; DB sharding not needed initially.
- Compatibility: mobile-first UI; PWA install banners; multi-browser testing.
- Testing: Jest unit, Cypress E2E; contract tests for API.
- Deployment: CI/CD via GitHub Actions; preview deployments per PR.


## Risks & Mitigations (Enhanced)

- **Provider downtime** → Circuit breaker (5 failures → 30s open), exponential backoff to 1h, BullMQ retry queue, graceful degradation with cached results.
- **Payment failures** → Idempotent webhook handlers (Redis deduplication), dead letter queue for failed webhooks, manual reconciliation dashboard, user email notifications.
- **Translation quality** → Human review checklist per locale, native speaker QA, fallback to English with locale indicator.
- **Cost overrun from abuse** → Redis-based rate limiting (token bucket), per-user generation quotas, hCaptcha on suspicious activity, cost alerting thresholds.
- **SSR complexity without Next** → Vite SSR with proven patterns, extensive E2E tests for SSR routes, route-level caching with edge invalidation.
- **TanStack Router SSR API is experimental** → Pin exact package versions; add SSR smoke tests on each CI run; maintain a non‑streaming fallback path; track upstream changes and schedule dependency updates behind feature flags.
- **PDF generation reliability** → Deterministic HTML templates, preflight render tests in CI, retry queue for failures, on-demand regeneration endpoint.
- **Database overload** → PgBouncer connection pooling (20 connections), read replicas for reporting, query timeouts (5s reads, 10s writes), slow query logging.
- **Redis failure** → Graceful degradation (no caching → direct DB, no rate limiting → in-memory fallback), health check monitoring, auto-restart on failure.
- **Disk full** → Monitoring alerts at 80%, automated cleanup of old generations (> 1 year), image optimization to reduce storage, quota per user.
- **DDoS attacks** → CloudFlare WAF with rate limiting, IP reputation filtering, challenge pages on spikes, geographic restrictions if needed.
- **Secret leakage** → Gitleaks pre-commit hooks, Docker secrets in prod, quarterly key rotation, secret scanning in CI, no secrets in logs.
- **Data loss** → Daily automated backups (Postgres + Redis + volumes), monthly restore drills, 7 daily + 4 weekly + 12 monthly retention, RPO < 15min.
- **Gemini API quota exhaustion** → Request queuing (BullMQ), user notifications on quota near-limit, fallback to alternative providers (architecture supports).
- **Service worker bugs** → Versioned SW with cache invalidation on deploy, SW update notifications, kill switch to disable problematic SW versions.


## Open Questions

- Confirm Better Auth version and Google OAuth app credentials (client id/secret, redirect URIs).
- Confirm Gemini model(s) and quotas; safety settings required for public app.
- VAT/tax handling for ETB pricing and invoice requirements.
- Chapa: confirm support for recurring subscriptions; if not, define renewal UX and reminders.
- Domains confirmed: `genimage.com` → site SSR; `app.genimage.com` → PWA; `api.genimage.com` → API; `analytics.genimage.com` → Umami. Please confirm there’s no typo (we saw "api.gemimage.com") and confirm TLS via nginx‑proxy‑manager (Let’s Encrypt).
- Image retention policy (how long to keep originals and generated assets).


## Commands Cheat Sheet (Enhanced)

### Development
- Setup: `pnpm i`, `pnpm -w run build`, `pnpm -w run dev`
- Dev apps: `pnpm --filter @genimage/web dev`, `pnpm --filter @genimage/api dev`, `pnpm --filter @genimage/site dev`
- Storybook: `pnpm --filter @genimage/ui storybook`

### Database
- Migrations: `pnpm --filter @genimage/db drizzle:generate`, `drizzle:migrate`
- Dev sync: `pnpm --filter @genimage/db drizzle:push`
- Seed: `pnpm --filter @genimage/db db:seed`
- Backup: `pnpm --filter @genimage/db db:backup`, `db:restore`

### Quality Assurance
- Lint: `pnpm -w lint` (ESLint with auto-fix)
- Type: `pnpm -w typecheck` (TypeScript strict mode)
- Test: `pnpm -w test` (Jest unit tests with coverage)
- E2E: `pnpm --filter @genimage/web e2e` (Cypress)
- A11y: `pnpm -w a11y` (axe-core automated checks)

### Performance & Load Testing
- Lighthouse: `pnpm run perf:lighthouse` (CI lighthouse checks)
- Load test: `pnpm run perf:load` (k6 scenarios)
- Bundle analysis: `pnpm run perf:analyze` (visualize bundle size)

### Security
- Audit: `pnpm run security:audit` (npm audit + Snyk)
- Secrets scan: `pnpm run security:scan` (gitleaks)
- OWASP check: `pnpm run security:owasp` (dependency-check)

### Queue Management
- Start workers: `pnpm --filter @genimage/api queue:start`
- Queue status: `pnpm --filter @genimage/api queue:status`
- Queue dashboard: `pnpm --filter @genimage/api queue:ui` (Bull Board)
- Clear failed: `pnpm --filter @genimage/api queue:clear-failed`

### Observability
- Logs: `docker-compose logs -f api` (Pino structured logs)
- Traces: `pnpm run obs:traces` (Jaeger UI at :16686)
- Metrics: `pnpm run obs:metrics` (Prometheus at :9090)
- Dashboards: `pnpm run obs:dashboards` (Grafana at :3000)

### Docker & Deployment
- Start all: `docker-compose up -d`
- Stop all: `docker-compose down`
- Rebuild: `docker-compose up -d --build api web site`
- View logs: `docker-compose logs -f [service]`
- Health check: `curl localhost/readyz`


## Appendix — CI Pipeline Outline (GitHub Actions)

- Triggers
  - `pull_request` to run affected-only; `push` to `main` for full run; nightly `cron` for dependency checks.
- Concurrency
  - `group: ci-${{ github.ref }}`, `cancel-in-progress: true`.
- Jobs
  - `setup`: checkout (shallow), setup Node 20.x, pnpm, restore caches (pnpm store + Turbo), install deps.
  - `lint`: run `turbo run lint --cache-dir .turbo` and upload ESLint report.
  - `typecheck`: run `turbo run typecheck` with `TSC_COMPILE_ON_ERROR=false`.
  - `test`: run `turbo run test -- --coverage` and upload lcov.
  - `build`: run `turbo run build` with `--filter=!@genimage/site` or include site when SSR assets needed.
  - `e2e` (optional): start `apps/api` with Postgres service, run Cypress on `apps/web`.
- Caching
  - Cache keys: `${{ runner.os }}-pnpm-${{ hashFiles('pnpm-lock.yaml') }}` and `${{ runner.os }}-turbo-${{ hashFiles('pnpm-lock.yaml') }}`.
- Security
  - Permissions minimal; pin actions; Dependabot alerts; CodeQL optional.
- Artifacts
  - Upload dist bundles, coverage, and test results for PR review.

## Definition of “Great PWA” Checklist

- [ ] Lighthouse PWA score ≥ 95 on mobile and desktop
- [ ] Manifest with maskable icons, theme colors, share target
  - [ ] Web Share Target API configured for receiving shared images/text
- [ ] Offline caching of shell + recent gallery; background sync for generate
- [ ] Push notifications with action buttons; localized
- [ ] Install prompts and proper A2HS UX
- [ ] Responsiveness from 320px → desktop; fast inputs; skeletons
- [ ] Accessibility AA: focus visible, roles, labels, color contrast


## Appendix — Example Feature Folders

Frontend (`apps/web/src`)
- app/
- routes/
- features/
  - auth/
  - generation/
  - subscriptions/
  - credits/
  - gallery/
  - profile/
  - i18n/
  - notifications/
- components/
- lib/

Backend (`apps/api/src/modules`)
- auth/
- users/
- credits/
- subscriptions/
- generation/
- media/
- i18n/
- notifications/

Each module: `routes.ts`, `controller.ts`, `service.ts`, `repository.ts`, `schema.ts`, `types.ts`, `__tests__/`.
