# P01 — Tenancy and auth

**Goal:** people can sign up, create an organization, invite members, create brands — and the database itself guarantees one organization can never touch another's data. **Est:** 7–9 working days. **Depends on:** P00. This is the most important phase; do not rush it.

## Steps

### 1. Auth configuration (Supabase Auth)
1. Enable email + password and **Google** sign-in. Require email confirmation. Turn on **leaked password protection** and set a minimum length of 10.
2. Enable **CAPTCHA protection with Cloudflare Turnstile** (Supabase Auth setting) and render the widget on signup, login and reset.
3. Enable MFA (TOTP). Policy: required for `owner` and `admin` (enforced in the app after first login, with a setup screen).
4. Set redirect URLs per environment (local, staging, production). Configure the SaaS sender domain with SES for auth emails (custom SMTP in Supabase) and brand the templates.
5. Session lifetime and refresh settings: short access token, refresh rotation on.

### 2. Core schema (migration `0001_tenancy.sql`)
Create everything in `03-data-model.md` §1: `organizations`, `profiles`, `memberships`, `feature_access`, `brands`, the `private` schema helpers, and `audit_log`. A trigger creates a `profiles` row for every new auth user. Index `memberships(user_id)`, `brands(org_id)`.

### 3. Row-level security
1. Enable RLS and write the four per-command policies for each table using the standard pattern.
2. Policies by table:
   - `organizations`: select if member; update if owner/admin; insert only via the `create_organization` function; no delete from the client.
   - `memberships`: select if member of the same org; insert/update/delete only owner/admin (an owner cannot be removed or demoted by an admin; the last owner cannot leave).
   - `brands`: select if member; insert/update with feature level ≥ `edit` **and** within the plan limit; delete owner/admin.
   - `audit_log`: select owner/admin only; **no** client insert, update or delete (written by server code).
3. `create_organization(name)` (security definer): creates the org, an `owner` membership for the caller, default `feature_access` rows, a trial `plan_id`, and one empty brand; returns the org id. Rate-limit by user.

### 4. Invitations
Table `invitations (org_id, email, role, token_hash, expires_at, invited_by, accepted_at)`. API route creates a token (random, store only its hash), emails a link; accepting requires being signed in with the invited email. Expire in 7 days; resend and revoke.

### 5. API layer conventions (`apps/web/app/api/*`)
Create the helper `withOrg(handler, { feature, level })` that: gets the user from the cookie session, reads the **active org id** from a signed cookie or path param, verifies membership and feature level on the server, applies per-user rate limits, validates the body with zod, and attaches `ctx = { user, orgId, role }`. **No handler may read `org_id` from the request body.** Add the same helper for server actions.

### 6. UI for this phase (minimal; P02 builds the shell)
Signup → verify email → "Create your workspace" → land on an empty dashboard. Settings → Team (list, invite, change role, remove). Org switcher for users in several orgs. An "Account pending/suspended" wall when a membership is not active.

### 7. Audit log
Server helper `audit(ctx, action, target, meta)` records sign-ins, role changes, invites, brand creation/deletion. Never log secrets.

### 8. Cross-tenant test suite (the gate)
Create `packages/db/tests/isolation.test.ts` that: seeds orgs A and B (users `a_owner`, `a_viewer`, `b_owner`), then for **every table listed in a generated catalogue** tries `select`, `insert`, `update`, `delete` as `b_owner` against A's rows and as `a_viewer` for writes, and calls each `security definer` function as the wrong org. Anonymous requests are also tried. Every attempt must fail or return zero rows. The catalogue is generated from `information_schema` so a new table without a test fails CI. See `templates/cross-tenant-test-checklist.md`.

## Deliverables
Migrations, policies, helpers, invitation flow, auth screens, `withOrg`, audit log, isolation test suite in CI.

## Acceptance
- [ ] Signup with email and with Google works; unverified email cannot proceed
- [ ] Turnstile blocks a scripted signup (test with the widget disabled → rejected)
- [ ] Owners and admins are forced through MFA setup
- [ ] User B cannot read, insert, update or delete anything in org A, on every table (suite green)
- [ ] An admin cannot demote the last owner; a viewer cannot invite
- [ ] Brand creation beyond the plan limit is refused by the database (test by direct insert)
- [ ] A table with RLS disabled makes CI fail
- [ ] No API route reads `org_id` from the body (lint rule or code search in CI)

## Tests to write
RLS suite (above); invitation expiry and reuse; role matrix per feature level; `create_organization` rate limit; membership removal revokes access instantly (next request).

## Traps
- `for all` policies — never (lesson #2).
- Forgetting `(select auth.uid())` wrapping makes policies slow at scale.
- `security definer` functions without `set search_path = ''` are injectable.
- Do not put org lists in the JWT yet; look memberships up (indexed) so removals take effect immediately.
- Remember `profiles.id` is a user, not an org member: never expose all profiles; expose only members of the caller's orgs through a view.

## If time is short
Postpone Google sign-in and MFA enforcement to P11 (keep the schema ready). Never postpone the isolation suite.

## Kickoff prompt for Claude Code
> Read `CLAUDE.md`, `03-data-model.md` and `phases/P01-tenancy-and-auth.md`. Create migration 0001 with the tenancy tables, private helper functions and per-command RLS policies exactly as specified. Then build the invitation flow, `create_organization`, the `withOrg` API helper, and the isolation test suite that auto-generates its table list from the database. Run the suite and show failures first, then fix them. Do not build any other feature tables yet.
