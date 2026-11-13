# P11 — Security and compliance

**Goal:** the product is safe to open to strangers, and we can prove it. This phase is a **checklist that runs alongside P01–P10** (each earlier phase already adds its tests) and a final hardening sweep. **Est:** 6–8 working days of dedicated work plus ongoing. **Depends on:** most of P01–P10 existing.

Standard to aim for: **OWASP ASVS Level 2**. An external penetration test follows within 30 days of launch and before 50 paying customers.

## A. Hardening checklist

### Edge and web
- [ ] Domain behind Cloudflare proxy; managed WAF ruleset on; rate limits: login 5/min/IP and 10/hour/email, signup and reset 3/hour/IP, API 300/min/IP
- [ ] Turnstile on signup, login, password reset, and public forms
- [ ] Security headers in `next.config` / middleware: CSP (no inline scripts; nonces), HSTS (preload), `X-Frame-Options: DENY`, `Referrer-Policy`, `Permissions-Policy`, `X-Content-Type-Options`
- [ ] Cookies `Secure`, `HttpOnly`, `SameSite=Lax`; CSRF protection on state-changing routes
- [ ] User text always rendered as text; markdown renderer never injects raw HTML

### Auth and accounts
- [ ] Email verification, leaked-password check, min length 10, Google sign-in
- [ ] MFA enforced for owners and admins; recovery codes
- [ ] "Sign out of all devices"; session revocation on role change or removal
- [ ] Account enumeration safe messages on login/reset

### API
- [ ] Every route uses `withOrg`; org never read from body (CI search)
- [ ] zod validation on every input; 1 MB body cap; 15 s timeout; consistent error shape without stack traces
- [ ] Per-user limits 120 reads / 30 writes per minute; per-org AI limits
- [ ] SSRF protection for any server-side fetch of user-supplied URLs (block private IP ranges, redirects limited, size/time caps) — website drafting, competitor checks, reference images
- [ ] File uploads: type sniffing, size caps, sanitized SVG or rasterized, no executable types

### Database
- [ ] RLS on every table; isolation suite green; new-table-without-RLS fails CI
- [ ] Every `security definer` function reviewed: fixed `search_path`, `revoke … from public, anon`, grant only what is needed
- [ ] Service role key used only in server-only modules and the worker; **scoped** database roles for the worker where possible
- [ ] Statement timeout 10 s for API roles; connection pooling; no direct DB access from the browser except through RLS-protected tables
- [ ] Daily backups, point-in-time recovery when plan allows; **restore drill** performed and documented

### Secrets and tokens
- [ ] All keys in secret stores; rotation calendar; immediate rotation when a person leaves
- [ ] Customer social tokens (only if we hold any) encrypted with Supabase Vault, decrypted only inside the posting worker
- [ ] No secrets in logs, error reports, or client bundles (CI scan with gitleaks; bundle check)

### AI-specific
- [ ] Prompt-injection rules in every prompt; scraped text delimited as data; outputs validated
- [ ] Moderation before anything is publishable; refuse and log policy violations
- [ ] Credit and hourly/monthly caps enforced in the gateway; spend-spike alert

### Supply chain
- [ ] `pnpm audit` in CI; Renovate/Dependabot weekly; lockfile committed; no postinstall scripts from unknown packages
- [ ] Docker base image pinned and rebuilt monthly; worker runs as non-root

### Logging and response
- [ ] Audit log: sign-in, role change, invite, plan change, publish, delete, export; kept 1 year; viewable by owner/admin
- [ ] Sentry alerts; error budgets; on-call contact written down
- [ ] **Incident response plan** (who is called, how customers are told, 72-hour notice where law requires)

## B. Automated security testing
- [ ] Isolation suite on every PR (P01)
- [ ] OWASP ZAP baseline scan against staging nightly
- [ ] Dependency and secret scans on every PR
- [ ] A scripted "abuse" test: 1,000 signups/hour, login stuffing, upload bombs, webhook replays, oversized bodies, parallel credit reservations — all handled without data exposure or runaway cost

## C. Privacy and legal (needs the lawyer: input #17)
- [ ] Terms of service and acceptable-use policy (no spam, impersonation, banned content, platform rule compliance)
- [ ] Privacy policy naming every processor (Supabase, Cloudflare, Vercel, Railway, OpenAI, Google, Razorpay/Stripe, the social provider, Amazon SES, Sentry)
- [ ] Data processing agreement customers can sign; sub-processor list page
- [ ] Cookie consent on the public site
- [ ] **Data export** (zip of the org's data) and **account deletion** (soft delete 30 days, then hard delete of rows, R2 objects and provider connections) from settings
- [ ] Meta-compliant **data deletion** page and callback (for our own app later)
- [ ] AI-generated content disclosure where a platform or law requires
- [ ] Applicable laws: India DPDP Act, EU GDPR, US state privacy laws; a data-subject request process (30-day SLA)

## D. Abuse and trust
Rate limits per plan on posts per day per account; automated checks for spam patterns; ability for us to suspend an org instantly (admin console, P10) which also pauses queued jobs and scheduled posts; a report-abuse email.

## Deliverables
The checklist above fully ticked, a written security overview page, runbooks (`docs/runbooks/`: incident, restore, key rotation), and the external pen-test booking.

## Acceptance
- [ ] ZAP baseline: no high issues; headers score A on a public header checker
- [ ] Isolation suite and abuse tests green in CI
- [ ] Restore drill: a staging copy restored from backup and verified
- [ ] Export and deletion work end to end on a test org and leave no objects in R2
- [ ] All findings in `05-risks-and-lessons.md` §A closed or explicitly accepted in an ADR

## Traps
- Security by obscurity: unguessable media URLs are a layer, not a control. The API still checks membership.
- Do not weaken a control to hit a date; cut features instead.

## Kickoff prompt for Claude Code
> Read `CLAUDE.md` and `phases/P11-security-and-compliance.md`. Audit the repo against section A and give me a table of each item as pass/fail/not-applicable with file references. Then fix failures in priority order (tenant isolation, secrets, headers, SSRF, uploads, rate limits), adding a test for each. Do not touch product features.
