# Data model and tenant isolation

Principles:
1. **Every customer-data table has `org_id uuid not null`**, indexed, with a foreign key to `organizations`. No exceptions, including child tables (denormalize `org_id` so policies never need joins).
2. **Row-level security is enabled on every table** in `public`, with **no policy meaning no access**. Policies are written **per command** (`select`, `insert`, `update`, `delete`). Never `for all`: in the current system a `for all` policy once masked a missing comment rule.
3. Policies check **membership first, role second**.
4. Helper functions live in a **private schema** (not exposed by the API) and are `security definer` with a fixed `search_path`.
5. `org_id` is never accepted from the client on writes; a trigger sets it from the parent row or the API sets it from the session.
6. Soft facts (plan limits, prices, credit costs) are **data**, not code.

## 1. Core tenancy tables

```sql
create schema if not exists private;

create table public.organizations (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  slug text unique not null,
  plan_id text not null default 'trial',
  status text not null default 'active',          -- active | suspended | deleted
  trial_ends_at timestamptz,
  created_by uuid not null references auth.users(id),
  created_at timestamptz not null default now()
);

create table public.profiles (                      -- one per auth user
  id uuid primary key references auth.users(id) on delete cascade,
  full_name text, avatar_url text,
  email_notifications boolean not null default true,
  created_at timestamptz not null default now()
);

create type public.org_role as enum ('owner','admin','editor','reviewer','viewer');

create table public.memberships (
  org_id uuid not null references public.organizations(id) on delete cascade,
  user_id uuid not null references auth.users(id) on delete cascade,
  role public.org_role not null,
  status text not null default 'active',            -- invited | active | suspended
  monthly_credit_cap int,                           -- per-member cap (nullable = none)
  created_at timestamptz not null default now(),
  primary key (org_id, user_id)
);
create index on public.memberships (user_id);

create table public.feature_access (                -- per-feature levels, as in today's app
  org_id uuid not null, user_id uuid not null,
  feature text not null,                            -- 'studio','review','publishing',...
  level text not null check (level in ('none','view','edit','full')),
  primary key (org_id, user_id, feature),
  foreign key (org_id, user_id) references public.memberships(org_id, user_id) on delete cascade
);

create table public.brands (
  id uuid primary key default gen_random_uuid(),
  org_id uuid not null references public.organizations(id) on delete cascade,
  name text not null,
  status text not null default 'active',
  created_at timestamptz not null default now()
);
create index on public.brands (org_id);
```

### Helper functions (private schema)

```sql
create or replace function private.is_member(p_org uuid)
returns boolean language sql stable security definer set search_path = '' as $$
  select exists (select 1 from public.memberships m
    where m.org_id = p_org and m.user_id = (select auth.uid()) and m.status = 'active');
$$;

create or replace function private.has_role(p_org uuid, p_roles public.org_role[])
returns boolean language sql stable security definer set search_path = '' as $$
  select exists (select 1 from public.memberships m
    where m.org_id = p_org and m.user_id = (select auth.uid())
      and m.status = 'active' and m.role = any(p_roles));
$$;

create or replace function private.feature_level(p_org uuid, p_feature text)
returns int language sql stable security definer set search_path = '' as $$
  select coalesce((select case fa.level when 'full' then 3 when 'edit' then 2
                    when 'view' then 1 else 0 end
     from public.feature_access fa
     where fa.org_id = p_org and fa.user_id = (select auth.uid()) and fa.feature = p_feature),
     -- default by role when no explicit row:
     (select case m.role when 'owner' then 3 when 'admin' then 3 when 'editor' then 2
        when 'reviewer' then 1 else 1 end
      from public.memberships m
      where m.org_id = p_org and m.user_id = (select auth.uid()) and m.status = 'active'), 0);
$$;
revoke all on function private.is_member, private.has_role, private.feature_level from public, anon;
grant execute on function private.is_member, private.has_role, private.feature_level to authenticated;
```

### The standard policy pattern (copy for every table)

```sql
alter table public.some_table enable row level security;

create policy some_table_select on public.some_table for select to authenticated
  using (private.is_member(org_id));
create policy some_table_insert on public.some_table for insert to authenticated
  with check (private.is_member(org_id) and private.feature_level(org_id,'some_feature') >= 2);
create policy some_table_update on public.some_table for update to authenticated
  using (private.is_member(org_id) and private.feature_level(org_id,'some_feature') >= 2)
  with check (private.is_member(org_id));
create policy some_table_delete on public.some_table for delete to authenticated
  using (private.has_role(org_id, array['owner','admin']::public.org_role[]));
```

Performance: always wrap `auth.uid()` as `(select auth.uid())`, index `org_id` plus the main filter column, and keep policy functions `stable`.

## 2. Tables by area

Every table below also has `org_id`, `created_at`, and `updated_at` (trigger). Brand-scoped tables also have `brand_id`.

| Area | Tables (columns beyond the common ones) |
|---|---|
| **Brand** | `brand_profiles` (tagline, industry, description, products_services, target_audience, goals, website_url, contact fields, service_areas[]), `brand_kits` (logo_path, colors jsonb, fonts jsonb, voice_notes), `brand_voice` (tone, banned_words[], approved_ctas[]), `brand_facts` (statement, source, approved bool; replaces hard-coded "approved stats"), `competitors` (name, urls, notes) |
| **Intelligence** | `intelligence_reports` (sections jsonb, status) |
| **Trends** | `trend_runs`, `trend_signals` (source, topic, relevance_score, reason, url, meta jsonb), `trend_watches`, `trend_events`, `trend_rules`, `trend_watch_runs` |
| **Strategy** | `strategies` (status, scope, platform, content_type, sections jsonb, calendar jsonb, header_insights jsonb, pillar_balance jsonb), `strategy_signals` (link) |
| **Content** | `content_items` (type, status, platform, title, body, media paths, metadata jsonb, reviewer_id, submitted_by, revision_count), `content_item_media` (ordered slides/assets), `content_attachments`, `content_comments` |
| **Creation jobs** | `studio_jobs`, `carousel_jobs`, `video_jobs` (each: status, inputs jsonb, outputs jsonb, progress jsonb, cost_credits, error), `style_presets` (per org overrides) |
| **Publishing** | `social_accounts` (provider, provider_account_id, platform, display_name, status, brand_id, connected_by), `scheduled_posts` (state machine below), `publish_attempts` (one row per try), `post_edits` |
| **Analytics** | `post_analytics`, `analytics_sync_state`, `ai_insights` |
| **Leads and automation** | `comment_automations` (post_id, keyword, message, asset, follow_gate jsonb, public_reply jsonb, enabled), `leads` (commenter, text, matched_keyword, dm_status, follow_status), `automation_events` (dedupe by provider comment id; unique) |
| **Review and team** | `review_requests`, `tickets`, `ticket_columns`, `ticket_comments`, `ticket_activity` (Board is P13) |
| **Notifications** | `notifications` (user_id, type, title, body, link, read_at), `notification_prefs` |
| **Billing** | `plans`, `plan_limits`, `subscriptions`, `billing_events` (raw webhook log, unique event id), `credit_ledger`, `credit_packs`, `invoices` (mirror) |
| **AI metering** | `ai_usage` (org_id, user_id, model, units, cost_usd, credits, job_id) |
| **System** | `audit_log` (actor, action, target, ip, meta), `webhook_receipts` (replay protection), `api_keys` (later), `feature_flags` |

## 3. Publishing state machine (fixes the "stuck publishing" bug)

`scheduled_posts.status`: `draft → scheduled → publishing → published | failed`, plus `canceled`.

- Transition to `publishing` uses a single atomic update that **claims** the row (`update … set status='publishing', locked_at=now() where id=… and status='scheduled' returning *`). Only one worker can win.
- Every attempt writes a `publish_attempts` row (started, finished, provider response summary, error).
- A rescue job returns rows stuck in `publishing` for more than 10 minutes to `scheduled` (up to 3 tries) then `failed`.
- The same logical post is never sent twice: store the provider post id before marking `published`; on retry, check it first.

## 4. Credits ledger (append-only)

```sql
create table public.credit_ledger (
  id bigint generated always as identity primary key,
  org_id uuid not null references public.organizations(id),
  kind text not null check (kind in ('grant','purchase','reserve','adjust','release','refund','expire')),
  amount int not null,                   -- signed: grants positive, reserves negative
  job_id uuid, ai_usage_id uuid, ref text,
  created_at timestamptz not null default now()
);
create view public.credit_balance as
  select org_id, sum(amount) as balance from public.credit_ledger group by org_id;
```

Flow: `reserve` writes `-N`. On success an `adjust` writes `N - actual` (positive or zero). On failure a `release` writes `+N`. Balance = sum. Inserts are possible **only** through security-definer functions called by the gateway (no client insert policy). A function `private.reserve_credits(org, job, n)` locks the org row, checks the balance and member cap, and inserts atomically.

## 5. Limits and entitlements

`plan_limits(plan_id, key, value)` holds numbers (`brands`, `channels`, `seats`, `credits_month`, `automations`, `watches`, `history_days`, flags like `video_clips`, `maker_checker`). A function `private.limit_for(org, key)` reads the org's plan. **Triggers** on `brands`, `social_accounts`, `memberships`, `comment_automations`, `trend_watches` refuse inserts above the limit, so a modified browser cannot exceed a plan.

## 6. Storage paths and policies

R2 is not behind Postgres policies, so the **API** enforces: path prefix must equal `org/{org_id}/…` for an org the caller belongs to; presigned URLs expire in 10 minutes; type and size are checked before signing. Any metadata table row holding a path must be in an org-scoped table.

## 7. Migrations approach

- All schema changes are SQL files in `packages/db/migrations`, ordered, reviewed in PRs, applied by the CLI. **No** changes made by hand in the dashboard (the current system lost track of changes that way).
- Every migration that adds a table also adds: RLS enabled, the four policies, the `org_id` index, and an entry in the cross-tenant test list.
- Generate TypeScript types from the schema in CI and fail the build if they are stale.
- Seed script creates two demo organizations (A and B) with data in every table, used by the isolation tests.

## 8. Cross-tenant test suite (runs on every PR)

For each table in a generated list: sign in as a user of org B and try `select`, `insert` (with org A's id), `update` and `delete` on org A's rows → all must return zero rows or an error. Also test: unauthenticated access, a member of org A with role `viewer` trying writes, storage path guessing, and calling every `security definer` function as the wrong org. One failure blocks the release. Details in `templates/cross-tenant-test-checklist.md`.
