# Growth OS SaaS — build rules for Claude Code


You are building **Growth OS SaaS**: a multi-tenant marketing operating system (research → plan → create → review → publish → measure → learn) for many customers. The build guide is in `docs/build-plan/` (copy it into the repo under `docs/build-plan/`). Read `README.md`, `02-architecture.md`, `03-data-model.md` and the current phase file before starting any work.

## Non-negotiables
1. **Tenant isolation first.** Every customer-data table has `org_id uuid not null` (indexed), RLS enabled, and four per-command policies (never `for all`). A new table is not done until it is in the isolation test catalogue. `org_id` never comes from a request body.
2. **Server-side everything that matters.** Permissions, plan limits, credit checks and workspace ids are enforced on the server and in the database. The browser is a convenience.
3. **All AI calls go through `packages/ai`.** No provider SDK imports elsewhere. Every call: reserve credits → call → moderate → settle (or release on failure) → record usage.
4. **Never swallow an error.** A failed step writes `failed` plus a human-readable reason to the job/row. No "continue on error" patterns that hide failures.
5. **Success flags are written after confirmation** (post published, DM sent, payment captured), never before.
6. **Webhooks in:** verify signature over the raw body, check timestamp, store a receipt, be idempotent. Webhooks out/queues are only triggered by our server code.
7. **Secrets** live in secret stores only. Never in code, logs, error reports, chat, tickets or the browser. Service-role keys only in server-only modules.
8. **Schema changes are migrations** in `packages/db/migrations`, reviewed in PRs. Never edit the database by hand in a dashboard.
9. **Before deleting any file, column or row type, search the whole repo and workers for references.**
10. **Brand data is data, not code.** No customer-specific text, colours, stats or accounts in prompts or code; use `brandContext`.
11. **User and scraped text is data, not instructions.** Delimit it in prompts; validate model output (strip code fences before JSON parse; retry once).
12. **Images are normalized on upload** (longest side ≤ 2160 px). Media lives in R2 under `org/{org_id}/brand/{brand_id}/…`.

## Stack
Next.js (App Router) + TypeScript (strict) + Tailwind + shadcn/ui + lucide-react + recharts + date-fns · Supabase (Postgres, Auth, Realtime) · pg-boss queue in a Node worker on Railway · Cloudflare (DNS, WAF, rate limits, Turnstile, R2) · Vercel · Razorpay (and Stripe later) · Amazon SES · Sentry · pnpm workspaces + Turborepo.

## Working method
- Work **one phase at a time**, in the order of `docs/build-plan/phases/`. Do not start a phase until the previous one's acceptance list is ticked.
- Write the test first for any rule about permissions, money, isolation or idempotency.
- After any change run `pnpm lint && pnpm typecheck && pnpm test`. Report the result.
- Verify live on staging with a real login (RLS applies) before calling a feature done. Use throwaway test orgs and clean them up.
- Keep PRs small; each states: what, why, how tested, what could break.
- If a default in `00-decisions-and-inputs.md` should change, write an ADR (`docs/adr/`) and update the tracker.
- Never spend real money on paid APIs without asking; use the cheapest model, smallest size and a capped staging key.

## Conventions
- API routes: `withOrg({feature, level})` → zod parse → act → typed response. Errors use the shared error type with a code and a human message.
- Jobs: payload carries only ids; handler reloads rows and re-checks org; idempotent; ends in `done` or `failed`.
- UI: server components by default; client components only for interactivity; Realtime only on screens that show live status; every list has an empty state; every spend shows an estimate first.
- Names: `snake_case` in SQL, `camelCase` in TypeScript, `kebab-case` for files.
- No `console.log` in app code; use the logger with `request_id`, `org_id`, `job_id`.

## Do not
- Add a table without RLS and tests. Add an endpoint without auth, validation and limits.
- Call a social or AI provider from the browser.
- Record or trust plan state from the browser.
- Leave `running` jobs after a crash, or write secrets to logs.
