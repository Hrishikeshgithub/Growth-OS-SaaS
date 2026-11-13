# P00 — Foundations

**Goal:** everything that must exist before the first line of product code: accounts, repo, environments, pipelines, standards. **Est:** 3–4 working days. **Depends on:** decisions D01–D18 confirmed (or defaults accepted).

## Inputs needed from you
Inputs 1–5 in `00-decisions-and-inputs.md` (name/domain on Cloudflare DNS, GitHub repo, new Supabase Pro project, Vercel project, Cloudflare account). Everything else can wait.

## Steps

### 1. Accounts and access (day 1)
1. Buy the domain; move DNS to **Cloudflare** (nameservers). Create records later as needed.
2. Create the **Supabase** project in a **new organization**, Pro plan, Mumbai. Turn on daily backups. Create a second **staging** project (can be Free initially; upgrade before the load test).
3. Create the GitHub repo `growth-os-saas` in the org. Protect `main`: pull request required, one review, status checks required, no force push.
4. Create the Vercel project linked to the repo (`apps/web` as root); disable auto-deploy of `main` to production until P12 (production deploys from tags).
5. Create the Cloudflare **R2** bucket(s): `growthos-media-staging`, `growthos-media-prod`; a token limited to each bucket (object read and write).
6. Create a password manager vault "Growth OS SaaS". Every secret goes there first. **No secret is ever pasted into chat, code or a ticket.**

### 2. Repo scaffold (day 1–2)
1. `pnpm init`, workspaces `apps/*` and `packages/*`, Turborepo with `build`, `lint`, `typecheck`, `test` pipelines.
2. Create `apps/web` (Next.js App Router, TypeScript strict, Tailwind, shadcn/ui via `shadcn init`), `apps/worker` (Node 20 + TypeScript, `tsx` for dev), and empty `packages/{db,shared,ai,social,billing,ui}`.
3. Add `CLAUDE.md` from `templates/CLAUDE.md.template` and `docs/adr/` with the ADR template.
4. TypeScript config shared in `packages/shared/tsconfig.base.json`. ESLint with: no `console.log` in app code, no imports of the service-role client in `apps/web/**/client`, no direct AI provider SDK imports outside `packages/ai`.
5. Git hooks (lefthook or husky): format + lint on commit, typecheck on push.

### 3. Local development (day 2)
1. `supabase init`; `supabase start` runs local Postgres/Auth/Realtime in Docker. Put the config in git.
2. `.env.example` files for web and worker listing every variable (no values). Use `zod` to validate env at boot in `packages/shared/env.ts`; the app refuses to start with a missing variable.
3. A `pnpm dev` that starts web + worker + Supabase together.

### 4. CI/CD (day 2–3)
GitHub Actions on every pull request: install (cached) → lint → typecheck → unit tests → build → **database tests** (start Supabase in CI, apply migrations, run RLS tests) → dependency audit. On `main`: deploy to **staging** (Vercel + migrations + worker). On a git tag `v*`: deploy to production with a manual approval step.

### 5. Environments and secrets (day 3)
Set the four environments from `02-architecture.md` §5. Vercel env vars per environment; Railway variables per service; Supabase secrets via the CLI. Name secrets consistently (`OPENAI_API_KEY`, `GOOGLE_AI_API_KEY`, `R2_ACCESS_KEY_ID`, …). Production values exist only in production.

### 6. Observability stubs (day 3)
Add Sentry to web and worker (DSN per environment). Add a `/api/health` route (returns build sha, database reachable) and a request-id middleware. Better Stack uptime monitor on staging.

### 7. Design tokens (day 3–4)
Create the Tailwind theme from the current tokens (`02-architecture.md` and `04-existing-system-inventory.md` §2): dark default and light themes through CSS variables, Inter and Instrument Serif Italic self-hosted. Build `packages/ui` primitives needed everywhere (Button, Card, Panel, Badge, Spinner, Modal, EmptyState, PageHeader, Toast). Reference screens in `docs/reference/growth-os-structural-spec.md`.

## Deliverables
Repo with CI green; staging deployed and reachable; local `pnpm dev` works for a new machine in under 30 minutes (document in `README`); secrets in the vault; empty health route live.

## Acceptance
- [ ] A new developer can clone, copy `.env.example`, run `pnpm dev` and see the app
- [ ] A pull request cannot merge without CI passing and a review
- [ ] Staging deploys automatically from `main`; production only from a tag with approval
- [ ] Env validation fails loudly when a variable is missing
- [ ] Sentry receives a test error from web and worker
- [ ] Backups are enabled on the production database
- [ ] ESLint blocks a client import of the service-role key (prove with a deliberate bad import)

## Traps
- Do not create the production database as Free "for now": quotas and no backups bit us before.
- Do not make schema changes in the Supabase dashboard; migrations only.
- Keep the internal app's Supabase and the SaaS's in **different organizations** so quotas and access never mix.

## If time is short
Skip the light theme and the Better Stack monitor until P12. Never skip CI, branch protection or env validation.

## Kickoff prompt for Claude Code
> Read `CLAUDE.md` and `docs/build-plan/phases/P00-foundations.md` and `02-architecture.md`. Scaffold the monorepo exactly as described (pnpm workspaces, Turborepo, `apps/web` Next.js App Router with Tailwind and shadcn, `apps/worker`, empty packages). Add env validation, ESLint rules, the GitHub Actions workflow and the health route. Do not add product features. After each step, run lint and typecheck and show me the result.
