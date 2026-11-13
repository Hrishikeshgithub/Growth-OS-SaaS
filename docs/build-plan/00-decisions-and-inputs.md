# Decisions and inputs

Two lists: **decisions** (each has a recommended default; change only if you disagree) and **inputs** (accounts, keys and answers we need from you, by the phase that needs them).

## A. Decisions

| ID | Decision | Default (recommended) | Why | Alternative |
|---|---|---|---|---|
| D01 | Product name and domain | Working name "Growth OS"; separate domain from scalepods.co (e.g. `growthos.<yourdomain>`) | Public site, app, media and email domains must be set early (cookies, OAuth redirect URLs, email DNS) | Subdomain of scalepods.co |
| D02 | Market | **India-first, global-ready**: Razorpay first (UPI, cards, GST invoices), Stripe added in P10b | Razorpay is already open in your browser; GST needed in India | Global from day one with Stripe first |
| D03 | Frontend | **Next.js (App Router) + TypeScript + Tailwind + shadcn/ui** on Vercel | Matches the structural spec your team already has; server components and route handlers give us a real API layer | Keep Vite + React Router (the current stack) |
| D04 | Database, login | **Supabase** (Postgres, Auth, Realtime), **new project, Pro plan from day one**, Mumbai region | Row-level security per tenant, managed backups, pooler | Neon or RDS plus our own auth (more work) |
| D05 | Media storage | **Cloudflare R2 with our own custom domain** (`media.<domain>`), private by default, signed upload links | No egress fees; the free `r2.dev` address was slow and rate-limited | Supabase Storage (small private files only) |
| D06 | Background jobs | **Own job system in code**: Postgres-backed queue (pg-boss) in a Node worker on Railway; **n8n is not used in the SaaS core** | Multi-tenant credentials, per-tenant limits, secrets and testing are hard in n8n; webhooks without auth were our biggest finding | n8n in queue mode (keeps workflows, harder isolation) |
| D07 | Render worker | Same Node worker image, headless Chromium + ffmpeg (ported from `carousel-studio/`) on **Railway Hobby** | Already proven; upgrade or add instances on demand | Fly.io or AWS ECS Fargate later |
| D08 | Social connections | **Provider adapter** (`SocialProvider` interface). Choose **bundle.social or Zernio** after a 2-day test in P05; Ayrshare is the fallback. Own Meta/LinkedIn/Google apps in the background (P13) | Lets customers connect without our own app reviews; swappable | Option A only: own apps (1+ month wait) |
| D09 | AI providers | OpenAI (text, image, moderation) and Google (Gemini image, Veo video, TTS, Lyria music), **all via our AI gateway** with a cost table | Matches what works today; each model can be swapped | Add Anthropic for text later |
| D10 | Billing | Razorpay Subscriptions + credit packs; plan state changes **only** from signed webhooks | Browser can never grant itself a plan | Stripe only |
| D11 | Plans | Starter $49 / Pro $149 / Agency $399 / Enterprise custom, with credits (see 01-product-scope) | From the approved solution doc; INR prices set in P10 | Founders change numbers |
| D12 | Tenancy model | **Organization → Brand**; one shared database with `org_id` on every row | Standard SaaS model; supports agencies | Database per customer (Enterprise only) |
| D13 | Agency white-label | **Not in version 1** (P13) | Keeps launch small | In v1 if a customer commits |
| D14 | Email | Amazon SES for sending; React Email templates | Already set up; cheap | Resend |
| D15 | Monitoring | Sentry (errors), Better Stack (uptime and status page), Supabase and Cloudflare logs | From the approved solution doc | |
| D16 | Language | English only in v1; database and UI built so strings can be translated later | | Hindi in P13 |
| D17 | Repo | New monorepo (pnpm workspaces + Turborepo): `apps/web`, `apps/worker`, `packages/*` | One place for shared types and tests | Separate repos |
| D18 | Old system | Keeps running as an internal tool; ScalePods itself becomes **workspace #1** in the SaaS in P12 | No big-bang cutover | Retire it earlier |

## B. Inputs needed from you

Grouped by the phase that first needs them. You do not need to do these up front; the "Needed by" column says when.

### Accounts and access

| # | Input | Needed by | Notes |
|---|---|---|---|
| 1 | Final **product name** and **domain** (buy it; point DNS to **Cloudflare**) | P00 | DNS on Cloudflare also unlocks R2 custom domain, firewall, Turnstile |
| 2 | A new **GitHub** repo (org Scale-Pods) and a deploy identity | P00 | Name suggestion: `growth-os-saas` |
| 3 | New **Supabase** project (Pro plan, Mumbai) in a **separate organization** from the internal app | P00 | Keeps internal and SaaS billing, quotas and access apart |
| 4 | **Vercel** project for `apps/web` (Pro already) | P00 | |
| 5 | **Cloudflare** account: R2 bucket, API token scoped to it, Turnstile site | P00 / P04 | |
| 6 | **Railway** project for the worker (Hobby already) | P04 | A second project, not the current one |
| 7 | **OpenAI** and **Google AI Studio** platform keys with billing and alerts | P04 | Use separate keys for dev and production |
| 8 | **Amazon SES** domain verification for the SaaS sender domain | P03 | SPF, DKIM, DMARC records |
| 9 | **Razorpay** account (live mode needs KYC) and a test-mode key now | P10 | Start KYC early; it can take days |
| 10 | **Stripe** account (optional until P10b) | P10b | |
| 11 | **Sentry** and **Better Stack** accounts | P12 | |
| 12 | Social provider trial accounts: **bundle.social** and **Zernio** (small paid trial) | P05 | Approve spend; test with ScalePods' own accounts |

### Answers and content

| # | Input | Needed by |
|---|---|---|
| 13 | Confirm plan names, prices and credit amounts, or give new ones (INR too) | P10 (earlier is better; P01 stores limits) |
| 14 | Logo, colours, fonts for the **SaaS product's own** brand (can reuse ScalePods tokens) | P02 |
| 15 | Free trial rule: default 14 days, Pro features, 200 credits, no card | P10 |
| 16 | Signup cap (default 200 accounts, then waitlist) | P12 |
| 17 | A lawyer for terms, privacy policy and data processing agreement | P11 |
| 18 | Who approves what: names for the first internal testers (5–10 people) | P05 onward |
| 19 | Whether existing ScalePods data (profiles, posts, analytics) is migrated into workspace #1 | P12 |
| 20 | Support channel: email address and (optional) chat tool | P12 |

### Questions I would like answered

1. Do you want customers to sign up with **Google** as well as email? (Default: yes.)
2. Should the SaaS include the **Blog module** (publishing to a customer's own website) in v1? (Default: **no**; it only fits ScalePods' own site. Customers connect WordPress/Webflow later.)
3. Should **Trend Alerts** ship in v1? (Default: yes, but with low per-plan limits because it costs money to run.)
4. Is **Video Studio (AI clips)** in the Starter plan? (Default: motion graphics only; AI clips from Pro.)
5. Do you want **client approval links** (non-members approving a post) in v1? (Default: later, with Agency.)
6. Any **competitor products** you want to match feature by feature? (Default: none; match the current Growth OS.)
