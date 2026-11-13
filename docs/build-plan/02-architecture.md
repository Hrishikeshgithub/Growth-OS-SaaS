# Architecture

## 1. The picture

```
                 Visitors / customers
                         │
                ┌────────▼─────────┐
                │ Cloudflare edge   │  DNS, CDN cache, firewall, rate limits, Turnstile
                └───┬─────────┬─────┘
        pages/app   │         │  media.<domain>
        ┌───────────▼───┐   ┌─▼────────────┐
        │ Vercel         │   │ R2 (media)   │  private bucket, signed URLs, cached by Cloudflare
        │ apps/web       │   └──────▲───────┘
        │ Next.js + API  │          │ presigned PUT/GET
        └──┬──────────┬──┘          │
   auth/data│          │ enqueue     │
   ┌────────▼───┐   ┌──▼───────────┐ │   ┌────────────────────────┐
   │ Supabase    │◄──┤ Job queue    │ │   │ Outside services       │
   │ Postgres    │   │ (pg-boss in  │ │   │ OpenAI, Google (Veo,   │
   │ Auth        │   │  Postgres)   │ │   │ Gemini, TTS, Lyria),   │
   │ Realtime    │   └──┬───────────┘ │   │ social provider,       │
   └────▲───────┘      │ pull         │   │ Razorpay/Stripe, SES,  │
        │          ┌────▼─────────────▼┐  │ Apify                  │
        └──────────┤ apps/worker       ├─►│                        │
                   │ Railway: jobs +    │  └────────────────────────┘
                   │ Chromium + ffmpeg  │
                   └────────────────────┘
```

Heavy or slow work (AI generation, rendering, publishing, scraping) never runs inside a web request. The web app writes a job; the worker picks it up at a pace the database and outside APIs can take, and reports progress through the database; the browser sees it live through Realtime.

## 2. Repo layout (monorepo, pnpm + Turborepo)

```
growth-os-saas/
├─ apps/
│  ├─ web/            Next.js App Router: UI, route handlers (the API), webhooks in
│  └─ worker/         Node + TypeScript: queue consumers, render engine (Chromium, ffmpeg)
├─ packages/
│  ├─ db/             SQL migrations, generated types, seed, RLS tests
│  ├─ shared/         zod schemas, constants, plan limits, error types, prompt builders
│  ├─ ai/             AI gateway: model registry, cost table, moderation, retries
│  ├─ social/         SocialProvider interface + adapters (bundle.social / Zernio / direct)
│  ├─ billing/        Razorpay + Stripe adapters, webhook verification, entitlements
│  └─ ui/             shared components and design tokens (shadcn-based)
├─ infra/             Dockerfile for worker, Cloudflare rules (as text), runbooks
├─ docs/              architecture decision records (ADRs), runbooks, API notes
└─ CLAUDE.md          build rules for Claude Code (template in templates/)
```

## 3. Components

### Web app (`apps/web`)
- Next.js App Router, TypeScript strict, Tailwind, shadcn/ui, lucide-react, recharts, date-fns, Inter + Instrument Serif Italic for accent words.
- Auth with `@supabase/ssr` (cookie sessions). Server components read data with the **user's** session, so row-level security applies. The service key is used only in server-only modules for system tasks, and never imported by client code (enforce with an ESLint rule).
- The API is route handlers under `app/api/*`, each: authenticate → resolve org from the session (never from the request body) → check permission and plan limit → validate input with zod → act → return typed result.
- Realtime subscriptions only on screens that show live status, closed on leaving.

### Worker (`apps/worker`)
- Queues (names are the contract): `ai.text`, `ai.image`, `ai.video`, `render.carousel`, `render.video`, `publish.post`, `publish.scheduler` (every minute), `analytics.sync`, `trends.scan`, `alerts.run`, `notify.send`, `billing.reconcile`, `social.token_check`.
- Every job payload carries `org_id`, `brand_id`, `job_id`. The handler **re-loads** the row and re-checks the org from the database; it never trusts payload contents beyond the ids.
- Concurrency gates: per-queue limits, plus at most **2 AI jobs per org** and **10 overall** at launch (tunable constants in `packages/shared`).
- Every handler is idempotent (safe to run twice) and ends in an explicit state: `done` or `failed` with a human-readable reason. Nothing may swallow an error.
- Render engine ported from the current `carousel-studio/` (see inventory): persistent Chromium, `tini` as PID 1, ffmpeg with bounded audio padding.

### AI gateway (`packages/ai`)
One function per capability (`generateText`, `generateImage`, `generateVideoShot`, `generateSpeech`, `generateMusic`, `moderate`). Each:
1. Looks up the model in the **registry** (id, provider, capabilities, aspect ratios, max variants).
2. Computes the cost from the **cost table** (USD per unit) and converts to credits.
3. **Reserves** credits (fails fast if the balance is too low).
4. Calls the provider with retry on 429/5xx and a timeout.
5. Runs moderation on generated text/images before they can be stored as publishable.
6. **Settles** the reservation at the real cost, or **releases** it on failure; writes one `ai_usage` row.
No other code may call an AI provider directly (lint rule + code review).

### Social provider adapter (`packages/social`)
Interface (provider-agnostic): `createConnectLink(orgId, brandId, platform)`, `listAccounts(brandId)`, `publish(accountId, post)`, `getPostStatus(id)`, `getAnalytics(accountId, range)`, `listComments`, `replyToComment`, `sendDirectMessage`, `disconnect(accountId)`, plus `verifyWebhook(req)`. One adapter per provider, plus a `direct` adapter later (P13). We store only the provider's **account id**, never platform tokens (unless using direct apps, then tokens go to Supabase Vault).

### Storage
- **R2** holds all media. Path: `org/{org_id}/brand/{brand_id}/{kind}/{uuid}.{ext}`. Browsers upload with short-lived presigned URLs created by the API after type, size and quota checks. Reads use signed URLs or a public custom domain **only** for assets that must be public (post media for providers that fetch by URL; use unguessable paths and expire).
- Images are **resized on upload** (longest side ≤ 2160 px) before being handed to any social provider; huge originals caused publish failures in the current system.
- Supabase Storage is not used for customer media.

### Database
Supabase Postgres, one shared database. Full design in `03-data-model.md`.

## 4. Security layers (built in, not bolted on)

| Layer | Controls |
|---|---|
| Edge | Cloudflare proxy, managed WAF rules, rate limits (login 5/min/IP, signup 3/hour/IP, API 300/min/IP), Turnstile on signup/login/reset |
| Web | CSP, HSTS, X-Frame-Options, Referrer-Policy, no inline scripts, user text rendered as text |
| Auth | Email verification, leaked-password check, Google sign-in, 2FA required for owners/admins |
| API | Auth required, zod validation on every input, 1 MB body cap, 15 s timeout, per-user limits (120 reads, 30 writes per minute), org taken from session |
| Webhooks in | Signature verified over the raw body, timestamp window, replay table, idempotent handlers |
| Database | RLS on every table, deny by default, per-command policies, security-definer helpers in a private schema, statement timeout 10 s |
| Secrets | All keys in the host's secret store; social tokens (if any) in Vault; no keys in code, workflows or the browser |
| AI | Pasted/scraped text treated as data, never instructions; moderation before publish; per-hour and per-month credit caps |
| Monitoring | Sentry, uptime checks, alerts on error spikes, spend spikes, failed logins, cross-tenant test failures; audit log kept 1 year |

## 5. Environments

| Env | Web | Database | Worker | Purpose |
|---|---|---|---|---|
| local | `pnpm dev` | local Supabase (Docker) | local | daily work |
| preview | Vercel preview per PR | shared **staging** Supabase project | staging worker | review |
| staging | `staging.<domain>` | staging project | staging worker | release candidate, load test |
| production | `app.<domain>` | production project | production worker | customers |

Rules: migrations run automatically on staging, and on production only from a tagged release. Production secrets exist only in production. Test data never goes to production.

## 6. Observability

- Structured JSON logs with `org_id`, `job_id`, `request_id` on every line; no tokens or message bodies in logs.
- Dashboards: queue depth and wait time, job failure rate per queue, provider error rate, AI spend per day, publishing success rate, database CPU/connections.
- Alerts (page someone): publishing failure rate above 5% over 15 min; queue wait above 5 min; worker down; AI daily spend above budget; provider webhook failures; backup failed.

## 7. Why not n8n or AWS for the core

- **n8n:** great for one team's internal flows (it still runs ScalePods' own). For strangers we need per-tenant limits, credentials per customer, tests in CI, no unauthenticated webhooks and a single place for secrets. Code gives us all of that. n8n can still be offered later as an optional integration.
- **AWS (EC2, VPC, RDS, IAM):** months of infrastructure work for scale managed services already cover at 100–200 users. Revisit only for an enterprise customer needing a private deployment or data region.
