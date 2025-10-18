# AGENTS.md — Working in the GenImage Monorepo

Authoritative guidelines for contributors and tooling agents working in this repository. These rules apply to the entire repo unless an app/package contains a more specific AGENTS.md.


## Purpose
- Ship a secure, fast PWA for AI image generation (apps/web) and an SEO‑focused SSR marketing site (apps/site) backed by a typed API (apps/api).
- Follow plan.md phases and checklists; keep scope tight and changes minimal per PR.


## Repo Layout (must keep)
- apps/
  - web/ — React + Vite PWA (TanStack Router)
  - api/ — Hono (Node) REST API, versioned under `/v1`
  - site/ — TanStack Router SSR using Vite SSR + Hono (marketing)
- packages/
  - ui/ — shadcn + Tailwind components; design tokens; a11y‑first primitives
  - db/ — Drizzle ORM (schema, migrations, seed, client for Postgres)
  - env/ — Zod‑validated environment loaders (server/client)
  - config/ — shared ESLint, TS, Tailwind, PostCSS configs
  - auth/ — Better Auth adapters + shared types; Google OAuth
  - i18n/ — translation resources and helpers (en/am/om)
  - shared/ — shared DTOs, utilities, API client, feature flags
  - og/ — Satori + Resvg templates and assets for dynamic OG images
  - docs/ — invoice/receipt HTML templates, email templates, branding

Never introduce Next.js; SSR is implemented with TanStack Router + Vite SSR + Hono only.


## Tooling & Prereqs
- Node 20.x, pnpm 9.x (enable with `corepack enable`).
- Turborepo orchestrates `lint`, `typecheck`, `build`, `test`, `e2e`.
- Postgres 16 (dev/prod) via Docker Compose.
- Do not use npm/yarn; use pnpm only.

Key commands (run at repo root)
- `pnpm i` — install
- `pnpm -w dev` — recommended dev script (if present) or use per‑app dev:
  - `pnpm --filter @genimage/web dev`
  - `pnpm --filter @genimage/api dev`
  - `pnpm --filter @genimage/site dev`
- `pnpm -w build` — build all
- `pnpm -w lint` / `pnpm -w typecheck` / `pnpm -w test`


## Branching, Commits, Reviews
- Branch naming: `feat/*`, `fix/*`, `chore/*`, `docs/*`, `test/*`.
- Conventional Commits required (commitlint enforced). Small PRs (prefer < 400 LOC changed).
- Every PR must: pass CI, include tests for new logic, and update docs/plan if scope changes.


## Coding Standards (all packages)
- TypeScript strict mode. No `any`. Prefer explicit return types.
- No default exports. Use named exports and explicit interfaces.
- Module boundaries: frontend never calls providers or DB directly; only via API.
- Absolute imports via TS path aliases exposed by packages/config.
- ESLint (flat) + Prettier; import/order enabled; unused imports fail CI.
- Public APIs should include JSDoc. Keep inline comments minimal and purposeful.


## Frontend — apps/web (PWA)
- Architecture: feature‑based folders under `src/features/*` plus shared `components/`, `lib/`, and `routes/` (TanStack Router).
- State/data: TanStack Query. Never store secrets in the client.
- i18n: All UI text must live in packages/i18n JSON resources. Use `react-i18next`. Support en/am/om.
- Accessibility: keyboard navigable, focus visible, roles/labels. Alt text on images; forms labeled.
- PWA:
  - VitePWA with manifest (maskable icons, theme, share target).
  - Workbox strategies: app‑shell precache; SWR for reads; background sync for `/generate`.
  - Web Push for generation completion and low credits.
- Analytics: Umami (respect DNT). Never block UX on analytics.


## Marketing — apps/site (SSR)
- TanStack Router SSR with Vite SSR + Hono. No Next.js.
- SEO: react-helmet-async for metadata; sitemap/robots; hreflang.
- Content: MDX allowed for `/blog`. Keep content in repo under `content/`.
- Dynamic OG: use packages/og (Satori + Resvg). Bundle fonts locally; cache prudently.


## Backend — apps/api (Hono)
- Structure: `src/modules/<name>/{routes,controller,service,repository,schema,types}.ts` plus `src/lib/*`.
- Routing: versioned under `/v1`; use Zod for all input/output validation.
- Logging: pino with request id; no PII in logs; redaction enabled.
- Errors: map to typed error responses; never leak stack traces to clients in prod.
- Rate limits: IP‑based and user‑based (stricter on credits/generate/auth).
- Auth: Better Auth with Google OAuth; secure, httpOnly cookies; CSRF where relevant.
- Image generation: provider is Google AI Studio (Gemini). All calls go server‑side with timeouts, retries, and safety settings.
- Media storage: local disk under images volume; validate MIME and size; sanitize filenames; serve with range + cache headers via API/proxy; signed URLs where appropriate.


## Database — Drizzle + Postgres
- Define schema in `packages/db/src/schema.ts` with clear relations and indexes.
- Migrations via `drizzle-kit`. Never edit old migrations; add new ones.
- Use transactions for multi‑write flows (credits, generation, payments).
- Seed data includes plan `PRO150` (150 credits, 50 ETB) and `ADDON30` (30 credits, 10 ETB).
- Testing uses Testcontainers Postgres; tests must rollback state.


## Credits, Plans, Billing, Documents
- Credits: maintain a ledger table; balance is `SUM(delta)`.
- Plans and add‑ons must be cataloged; changes require migrations and updated seeds.
- Payments: Chapa integration is feature‑flagged (paused). Implement idempotent verification and webhook handling when enabled.
- Invoices/Receipts:
  - Create invoices (draft → issued/paid) for subscriptions and add‑ons.
  - Generate receipts on successful payments.
  - HTML → PDF via chosen renderer (ADR: Playwright/Chromium recommended for fidelity). Store PDFs in `docs` volume.
  - Signed, expiring URLs for downloads; owner‑only; admin override with audit logs.
  - Email PDFs (provider configurable; stubbed in dev).


## i18n & Accessibility
- All strings in i18n resources. No hard‑coded UI copy in components.
- Localize invoice/receipt templates and legal pages. Currency formatting (ETB) and locale dates.
- Maintain AA contrast and keyboard accessibility; run axe/Lighthouse.


## Observability & Security
- Metrics: Prometheus `/metrics` (requests, latency, generation success rate).
- Logs: include request id; redact secrets; ship JSON logs.
- CSP, Referrer‑Policy, Permissions‑Policy headers enabled. HTTPS required.
- Secrets only via environment; validated via packages/env.
- Rate‑limit auth and generation; account lockout on brute force attempts.


## Testing
- Unit tests for every service/repository; API contract tests with supertest.
- E2E (Cypress) for core flows: auth, generate, subscribe, invoices.
- Add regression tests for every bug fix.
- Minimal coverage target: 80% statements/branches.


## CI (GitHub Actions)
- Use provided reusable workflows (see plan.md appendix). Concurrency cancellation on PRs.
- Cache pnpm store + Turbo. Affected‑only builds on PRs; full on main.
- Upload coverage and build artifacts; annotate ESLint/TS diagnostics.


## Docker & Deploy
- Compose services: api, site (SSR), web (static), Postgres, nginx‑proxy‑manager (+ DB), Umami, images volume, docs volume.
- Domains: genimage.com (site), app.genimage.com (PWA), api.genimage.com (API), analytics.genimage.com (Umami).
- TLS via nginx‑proxy‑manager (Let’s Encrypt). Enable HSTS when ready.
- Nightly backups for Postgres and images/docs volumes. Test restores.


## Adding a Feature (checklist)
1) Create/adjust schema + migrations in packages/db; run and test.
2) Backend module: repository → service → controller → routes with Zod.
3) Frontend feature: routes, components, i18n strings, tests.
4) Wire analytics events (Umami), a11y checks, and translations (en/am/om).
5) Update plan.md if scope differs; add ADR if architecture changes.


## Do / Don’t
- Do keep changes atomic and tested. Don’t mix unrelated refactors with features.
- Do follow strict typing. Don’t use `any` or default exports.
- Do validate and sanitize all inputs. Don’t trust client data.
- Do keep secrets out of code. Don’t commit `.env` files (use `.env.example`).
- Do consider offline behavior and background sync impact for PWA.


## Contact & Ownership
- Code owners must review sensitive areas: auth, billing, migrations, invoices.
- If guidelines conflict, plan.md phases take precedence; otherwise favor security and correctness.

