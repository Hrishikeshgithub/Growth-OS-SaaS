# P04 — Job platform

**Goal:** the machinery every later feature uses: a durable queue and worker, the AI gateway with credit reservation, secure uploads to R2, signed webhooks in, and job progress that the browser can watch. **Est:** 8–10 working days. **Depends on:** P01 (tenancy). Build this **before** any feature that calls an AI model or posts to a platform.

## Steps

### 1. Queue and worker skeleton
1. Add **pg-boss** (Postgres-backed queue) with its own schema in the same database. Create `packages/shared/queues.ts` with the queue names from `02-architecture.md` §3 and typed payloads (`zod`).
2. `apps/worker`: boot, connect, register consumers, health endpoint, graceful shutdown (finish the current job, stop taking new ones). Structured logging with `job_id`, `org_id`.
3. Deploy the worker to **Railway** (new project, Hobby) with a Dockerfile in `infra/`. Copy **whole directories** in the Dockerfile (never file lists) and run under `tini`. Staging and production services separate.
4. A `jobs` table for **user-visible** state (separate from the queue's internal tables): `id, org_id, brand_id, kind, status (queued|running|done|failed|canceled), progress jsonb, input_ref, output_ref, error_code, error_message, cost_credits, created_by, created_at, started_at, finished_at`. The web app creates the row, then enqueues; the worker updates it. Realtime pushes changes to the browser.

### 2. Concurrency and fairness
Constants in `packages/shared/limits.ts`: `AI_JOBS_PER_ORG = 2`, `AI_JOBS_GLOBAL = 10`, `RENDER_JOBS_GLOBAL = 2`. A gate function checks counts of `running` jobs per org before a consumer starts; if full, the job is re-queued with a delay (the user sees "queued"). Per-queue retry policy: 3 tries with exponential back-off for provider errors, **no retry** for validation errors and credit failures. A dead-letter queue is alerting-only.

### 3. Job lifecycle rules
- Every handler: load row → re-verify org and status → do work → write `done` or `failed` (never leave `running`).
- **Idempotent**: running twice produces the same result or detects it already finished.
- A **rescue job** every minute moves `running` jobs with no heartbeat for 10 minutes to `failed` with a clear reason and releases their credits.
- A handler failure is **never** swallowed: it must set `failed` + a human-readable `error_message` (e.g. "The render service did not respond; nothing was charged"). (Lesson #3.)

### 4. AI gateway (`packages/ai`)
1. **Model registry** (`models.ts`): id, provider, capability (`text|image|video|speech|music|moderation`), aspect ratios, max variants, whether it accepts a reference image. Seed from `src/lib/studio.ts` (image models) and `videoStudio.ts` (Veo engines).
2. **Cost table** (`costs.ts`): USD per unit and the **credit rate** (credits per USD, default 100). Seed Veo rates from `videoStudio.ts` `PRICE_PER_SECOND`; token prices for text; per-image prices. Stored in the database (`ai_cost_table`) so numbers can change without deploy; the code reads and caches it.
3. Functions: `generateText`, `generateImage`, `generateVideoShot`, `generateSpeech`, `generateMusic`, `moderate`. Each: reserve credits → call provider (timeout, retry on 429/5xx) → moderate (text and image outputs) → settle at the real cost → write `ai_usage`.
4. **Prompt-injection hygiene:** user and scraped text go in clearly delimited data sections; system prompts state that such text is data; outputs that fail JSON parsing are retried once, then fail cleanly (strip code fences before parsing).
5. Cost caps: per job ceiling (default $5 for video, from the current system), per org per hour and per month.
6. Unit tests with a fake provider: reservation on start, release on failure, settle at lower actual cost, refusal on insufficient credits, per-member cap.

### 5. Credit ledger functions
Implement `private.reserve_credits`, `settle_credits`, `release_credits` per `03-data-model.md` §4 as `security definer` functions callable **only by the service role** (the worker/API server code). Locking: select the org row `for update` inside the function so concurrent jobs cannot overspend. Grants (`grant_credits`) are server-only too. Add a daily reconciliation job that checks `credit_balance` ≥ 0 and `ai_usage` totals match ledger settles.

### 6. Uploads to R2
1. API route `POST /api/uploads/presign`: check membership and feature level; validate `kind` (logo, post-image, post-video, pdf, reference-image), content type allow-list (no html/svg unsanitized/js), size caps (image 20 MB, video 200 MB, pdf 100 MB), per-user rate (50 per hour) and storage quota; return a presigned `PUT` URL (10 min) with path `org/{org}/brand/{brand}/{kind}/{uuid}.{ext}`.
2. After upload the browser calls `POST /api/uploads/confirm`; the server `HEAD`s the object, checks size and sniffed type, writes a `media_assets` row (path, kind, size, width, height, sha256), and queues an **image-normalize job** (resize to ≤ 2160 px longest side, strip metadata, generate a thumbnail).
3. Reads: signed GET URLs from the API, or the Cloudflare-cached custom domain for post media that providers must fetch (unguessable paths, optional expiry via a signed path token).
4. Configure the R2 **custom domain** (`media.<domain>`), CORS for the app origins only, and an R2 lifecycle rule for abandoned uploads (objects without a `media_assets` row after 24 h).

### 7. Webhook receiver framework
`packages/shared/webhooks.ts`: verify signature over the **raw** body (HMAC-SHA256 with a per-provider secret, constant-time compare), reject if the timestamp is older than 5 minutes, insert into `webhook_receipts (provider, event_id unique)` — a duplicate means "already handled, return 200". Handlers enqueue work and return fast (< 3 s). Used by P05 (social provider), P10 (billing).

### 8. Progress UI primitives
`useJob(jobId)` hook (Realtime subscription + initial fetch + polling fallback every 30 s). `JobProgress` component: label, percent, estimated time left, "queued behind N jobs", failed state with reason and retry. Estimates come from per-kind constants (like the current ETA helpers in `videoStudio.ts`), and are labelled as estimates.

### 9. Metering display
`/api/credits` returns balance and recent usage; fills the shell pill from P02.

## Deliverables
Queue + worker deployed on staging; `jobs` table and realtime; AI gateway with registry/cost table/ledger; presign + confirm + normalize; webhook framework; `useJob` and `JobProgress`.

## Acceptance
- [ ] Enqueue a dummy job from the UI; watch it run, progress and finish live
- [ ] Kill the worker mid-job: the rescue job fails it within 10 minutes, releases credits, and the UI shows why
- [ ] An AI call with a balance of 0 is refused **before** the provider is contacted
- [ ] A provider error releases the reservation (ledger shows reserve then release)
- [ ] Two concurrent jobs cannot overspend a balance (test with 20 parallel reservations)
- [ ] Per-org concurrency cap holds (third job waits, shows "queued")
- [ ] Upload: an `.html` file and a 500 MB file are refused; a 12 MP image is normalized to ≤ 2160 px
- [ ] A webhook with a bad signature, an old timestamp, or a replayed event id is refused or ignored
- [ ] No provider SDK is imported outside `packages/ai` (lint)

## Traps
- Don't record spend at click time; reserve at **acceptance** (lesson #4).
- Don't let the browser call a webhook or queue directly; only the API enqueues.
- Don't leave `running` rows after a crash.
- Presign only after checks; never return a long-lived write URL.

## If time is short
Postpone moderation and per-org hourly caps to P11 (keep hooks); implement video/music functions in P07. Keep reserve/settle/release and the rescue job.

## Kickoff prompt for Claude Code
> Read `CLAUDE.md`, `02-architecture.md` §3, `03-data-model.md` §4 and `phases/P04-job-platform.md`. Build, in order: the `jobs` table and Realtime hook, the pg-boss queue and worker with the rescue job, the credit ledger functions with concurrency tests, the AI gateway with a fake provider and tests, the R2 presign/confirm/normalize flow, and the webhook receiver framework. After each, run its tests. Do not call real AI providers yet except one smoke test of `generateText` with a tiny prompt.
