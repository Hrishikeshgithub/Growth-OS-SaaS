# P12 — Launch readiness

**Goal:** prove the system holds at launch size, make it observable, move ScalePods in as workspace #1, and open the doors in a controlled soft launch. **Est:** 5–6 working days. **Depends on:** P01–P11 (P11 mostly done).

## Launch size targets
100 signed-in users at normal load, **200 at worst**; open signup stops at **200 accounts**, then a waitlist (input #16). A burst of **1,000 public visitors** must not affect the app.

## Steps

### 1. Capacity settings
Upgrade staging to the production-grade plan for the test. Supabase compute: start Small (production), raise only if the load test needs it. Confirm connection limits (Small: 90 direct / 400 pooled) and that all API traffic goes through the pooler. Worker: 2 render slots, 10 AI jobs globally; Railway plan sized for it.

### 2. Load test (k6 or Artillery) against staging
Scenarios and pass criteria:
- 200 simulated signed-in users browsing (reads) and doing writes at the documented per-user limits → **95% of API reads under 500 ms, errors under 1%**
- 1,000-visitor spike on the public pages → served from cache, no database load
- 50 concurrent AI jobs submitted → per-org and global caps hold; none lost; users see "queued"
- 100 posts scheduled for the same minute → all published within 3 minutes, none duplicated
- Webhook flood (replays) → all replays ignored
Record results in `docs/load-test-report.md`; fix and rerun until green.

### 3. Monitoring and alerts
Sentry (web, worker); Better Stack uptime for `app`, `api/health`, worker health; **status page** (public) with provider components (social provider, AI providers, payments); dashboards from `02-architecture.md` §6; alert routes to email/phone for: worker down, queue wait > 5 min, publish failure rate > 5%, AI spend over daily budget, backup failure, webhook failures, database CPU > 80%. Write the on-call note (who, how to reach).

### 4. Email and domains
SES production access, SPF/DKIM/DMARC on the sender subdomain, bounce and complaint handling (suppress addresses), transactional templates tested in major clients.

### 5. Public site and legal pages
Landing page, pricing (reads `plans`), signup, login, terms, privacy, sub-processors, security overview, status link, data-deletion page, contact. Cookie consent. Cache aggressively at Cloudflare; only signup/login reach the backend.

### 6. Support and docs
User manual (port `UserManual.tsx` content and update for SaaS flows: onboarding, connecting channels, credits), a "Getting started in 15 minutes" guide, FAQ, support email and a tool for tickets. In-app help link and a feedback button.

### 7. Backups and recovery drill
Verify daily backups; perform a restore of production into a scratch project; time it; document RPO/RTO in `docs/runbooks/restore.md`.

### 8. Move ScalePods in (workspace #1)
1. Decide scope (input #19): profile, brand kit, facts, posts and analytics history, leads. Skip jobs history and old media unless wanted.
2. Write an idempotent import script (`packages/db/scripts/import-scalepods.ts`) reading from the **old Supabase** with a read-only key; maps per `04-existing-system-inventory.md` §4; copies needed media to the new R2 paths; **dry-run mode** first with a report of counts.
3. Reconnect ScalePods' channels through the provider; keep the old internal app and n8n flows running until the SaaS is verified, then switch internal use over.

### 9. Soft launch
1. **Internal beta** (5–10 testers, input #18) for 3–5 days: scripted journey sheet (signup → onboarding → connect → create → review → publish → analytics → upgrade → cancel). Triage into blockers, fixes, later.
2. **Friends and design partners** (20–30 accounts) with a feedback form; watch dashboards daily.
3. Open signup with the cap and waitlist; announce.
4. Day-1 to day-7 daily review: errors, queue waits, publish failures, support questions, spend vs revenue.

### 10. Go / no-go checklist
- [ ] Load test report green and attached
- [ ] Isolation, abuse and ZAP tests green on the release tag
- [ ] Backup restore drill done; runbooks written
- [ ] Alerts firing to a real human (test each)
- [ ] Legal pages live and reviewed by the lawyer
- [ ] Test purchase end to end in live mode with a real small payment, then refunded
- [ ] Support email monitored; status page live
- [ ] Rollback plan: previous release tag redeployable in 10 minutes; migrations backward compatible for one release
- [ ] External penetration test booked (within 30 days)

## Deliverables
Load test report, monitoring, public site, docs, import script and ScalePods workspace, soft launch run.

## Acceptance
All items in the go/no-go checklist ticked.

## Traps
- Don't launch on a Friday.
- Don't test load against production.
- Don't import data into production without a dry run and a backup taken first.

## Kickoff prompt for Claude Code
> Read `CLAUDE.md` and `phases/P12-launch-readiness.md`. First write the k6 load-test scripts for the five scenarios and the report template, run them against staging and show me results. Then the import script for ScalePods in dry-run mode only (read-only access to the old database), producing a counts report for my review.
