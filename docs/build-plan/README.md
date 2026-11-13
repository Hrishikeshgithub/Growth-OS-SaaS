# Growth OS SaaS — Build Plan (from scratch)

This folder is the single guide for building the Growth OS SaaS as a **new system** (new repo, new database, new infrastructure), replicating what the internal ScalePods Growth OS does, but multi-tenant, billable and safe for strangers.

The current internal app keeps running untouched while this is built. Nothing here edits it.

## How to use this folder

1. Read **00-decisions-and-inputs.md** first. It lists every decision with a recommended default, and everything we need from you. Unanswered decisions use the default, so you are never blocked.
2. Skim **01-product-scope.md** (what is in version 1), **02-architecture.md** (how it fits together) and **03-data-model.md** (the database).
3. Work through the phases **in order**: `phases/P00` to `phases/P13`. Each phase file is self-contained: goal, steps, acceptance checks, tests, traps to avoid, and a ready-to-paste kickoff prompt for Claude Code.
4. Tick progress in **tracker.md** as you go.
5. Keep **05-risks-and-lessons.md** open. It collects the real mistakes the current system made, so the new one does not repeat them.
6. When porting a feature, look it up in **04-existing-system-inventory.md**. It names the existing file, table or workflow to copy logic from.

## Folder map

| File | What it is |
|---|---|
| `00-decisions-and-inputs.md` | Decisions (with defaults) and the accounts, keys and answers needed from you |
| `01-product-scope.md` | What version 1 includes, what is later, plans and limits |
| `02-architecture.md` | Stack, services, repo layout, queue, AI gateway, storage, security layers |
| `03-data-model.md` | Tenancy tables, isolation rules, core tables, ledger, migrations approach |
| `04-existing-system-inventory.md` | Every current module mapped to its new home, with source paths to port from |
| `05-risks-and-lessons.md` | Cross-cutting traps and the fixes learned the hard way |
| `tracker.md` | Phase-by-phase progress board |
| `phases/P00` … `P13` | The step-by-step build guides |
| `templates/` | `CLAUDE.md` for the new repo, decision-record template, cross-tenant test checklist |

## The phases at a glance

| # | Phase | Outcome | Est. working days* |
|---|---|---|---|
| P00 | Foundations | Accounts, repo, environments, CI, design tokens | 3–4 |
| P01 | Tenancy and auth | Organizations, members, brands, row-level security, cross-tenant tests | 7–9 |
| P02 | App shell | Next.js shell, navigation, permissions UI, theming | 3–4 |
| P03 | Brand kit and onboarding | Brand profile, voice, approved facts, onboarding wizard | 5–6 |
| P04 | Job platform | Queue, worker, AI gateway, credit ledger, uploads | 8–10 |
| P05 | Social connections and publishing | Provider adapter, connect, composer, calendar, scheduler | 10–12 |
| P06 | Research and strategy | Business intelligence, competitors, trends, alerts, strategy | 8–10 |
| P07 | Content creation | Text engine, AI Studio, Carousel Studio, Video Studio | 12–15 |
| P08 | Review and team | Maker-checker review, board, notifications | 6–8 |
| P09 | Analytics, leads, automation | Analytics, AI insights, comment-to-DM, leads | 6–8 |
| P10 | Billing and entitlements | Plans, Razorpay/Stripe, credits, usage page | 6–8 |
| P11 | Security and compliance | Hardening, tests, privacy, penetration test | 6–8 (and ongoing) |
| P12 | Launch readiness | Load test, monitoring, legal pages, soft launch | 5–6 |
| P13 | After launch | Own platform apps, Agency tier, scale steps | ongoing |

\*One or two engineers working with Claude Code. Total is roughly **16–20 weeks** to a soft launch. P11 security work is done continuously, not only at the end. Phases P06, P08 and P09 can run in parallel with P07 if two people are building.

## Ground rules for the whole build

1. **Tenant isolation is never an afterthought.** Every table has `org_id` from the first migration, and every phase adds cross-tenant tests for what it created.
2. **Nothing paid runs without a credit check.** Every AI call goes through the gateway (P04).
3. **The browser is never trusted.** Limits, permissions and workspace ids are enforced on the server and in the database.
4. **Every outside call can fail.** Retry with back-off, never swallow errors, always leave a visible failed state.
5. **Ship small, verify live.** Each phase ends with a real end-to-end check, not just passing tests.
6. **Record decisions.** Use `templates/adr-template.md` whenever a default in `00-decisions-and-inputs.md` is changed.
