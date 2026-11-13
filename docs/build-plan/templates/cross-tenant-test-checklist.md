# Cross-tenant test checklist

Goal: prove a person in organization B can never read, change, delete or infer anything of organization A. Automate everything below in `packages/db/tests/`; run on every pull request. **One failure blocks the release.**

## Setup (seed script)
- Org **A** with users: `a_owner`, `a_admin`, `a_editor`, `a_viewer`.
- Org **B** with users: `b_owner`, `b_editor`.
- A user `multi` who is a viewer in A and an owner in B.
- Rows in **every** table for A and for B (the seed iterates over the table catalogue).
- Unauthenticated (anon) client.

## Table catalogue
Generated from `information_schema.tables` for schema `public`. The suite **fails if a table is missing from the policy list or has RLS disabled**, so new tables cannot slip through.

## For each table, as `b_owner` against A's rows
- [ ] `select` returns 0 rows
- [ ] `insert` with A's `org_id` is rejected
- [ ] `insert` with B's `org_id` but a foreign key pointing at an A row is rejected
- [ ] `update` of an A row changes nothing (0 rows)
- [ ] `update` moving a B row to A's `org_id` is rejected
- [ ] `delete` of an A row deletes nothing

## Role matrix inside the same org (A)
- [ ] `a_viewer`: select allowed where the feature level ≥ view; insert/update/delete refused
- [ ] `a_editor`: write allowed only for features with level ≥ edit; delete refused unless owner/admin
- [ ] Maker-checker: the author of an item cannot approve it (direct SQL/API call)
- [ ] `multi` acting in org A has only viewer rights; switching to B gives owner rights; neither leaks into the other

## Anonymous
- [ ] Every table: select/insert/update/delete as anon all fail
- [ ] Every `security definer` function: executable by anon only if intended (default: no)

## Functions and views
- [ ] Each function in `private` and `public` called as the wrong org returns nothing or errors
- [ ] Views (e.g. `credit_balance`) respect RLS (`security_invoker = true`)
- [ ] `credit_ledger` cannot be inserted into from a client session at all

## Storage and media
- [ ] A presign request for a path under another org's prefix is refused
- [ ] A signed URL for A's object does not work with a changed path
- [ ] After org deletion, objects under `org/{id}/` are gone

## Jobs and webhooks
- [ ] A job payload with B's ids but A's `org_id` (forged) is refused: the handler reloads rows and compares orgs
- [ ] Provider webhooks resolve the org from our stored account id, never from payload fields alone

## API
- [ ] A request body containing `org_id`/`orgId` is ignored or rejected on every route (contract test over the route list)
- [ ] Passing another org's resource id in the URL returns 404 (not 403) to avoid confirming existence

## Reporting
The run prints a table: table × command × result, plus a count of tables covered. CI uploads it as an artifact. Coverage must be **100% of public tables**.
