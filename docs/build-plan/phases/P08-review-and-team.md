# P08 — Review and team

**Goal:** a safe approval path from draft to ready-to-publish, notifications that reach people, and team management that matches how agencies and companies work. **Est:** 6–8 working days. **Depends on:** P01, P05, P07. Can run in parallel with P07.

## Steps

### 1. Review model
Statuses on `content_items`: `draft → in_review → approved | revision → …`, then `scheduled → published`. Table `review_requests` (content_item_id, requested_by, reviewer_id nullable, note, status, decided_by, decided_at, decision_note). Rules ported from `docs/board-maker-checker-test-plan.md` and `team-collaboration-prd.md`:
- **Maker-checker:** the person who created or last edited content cannot approve it (enforced in the database, not just the UI). Designers and writers can submit but not approve.
- **Send to a named reviewer** (only people with `approve` ability appear in the picker via a security-definer function, because makers cannot read others' permission rows).
- **Starter plan:** simple "Approve" by an owner/admin. **Pro+:** maker-checker. Plan flag `maker_checker` from `plan_limits`.
- Approved content goes to the Publishing "Ready" list; editing approved content after approval returns it to review (configurable).

### 2. Creative Review screen
Port from `CreativeReview.tsx`: tile grid with platform and content-type filters, status filter (all/ready/revision), multi-select with bulk approve/send back, preview modal (platform mockup), side-by-side caption, notes box for revisions, **attachments** (files or links for the reviewer, `content_attachments`), a "Sent to me" filter, and an **Import** button (upload, Canva or Figma link) that creates a draft.

### 3. AI revision
"Revise with AI" takes the reviewer's note and rewrites copy within the brand voice and approved facts (job `ai.text`, credits), incrementing `revision_count`. Port the prompt from the old "Content Revision" workflow.

### 4. Comments and activity
Threaded comments on a content item with @mentions; an activity feed (who approved, who sent back, when) from `audit_log` and review rows.

### 5. Notifications
- Table `notifications` + `notification_prefs` (per user per type: in-app, email).
- Producers enqueue `notify.send`: assigned for review, approved, sent back, mentioned, publish success/failure, social account needs reconnect, credits low (80%/100%), plan events.
- Consumer writes the in-app row (Realtime pushes it to the bell) and sends email via SES with React Email templates, honouring prefs and batching digests (daily/weekly) where chosen. Include an unsubscribe link and a one-click per-type opt-out.
- Bell UI: unread count, mark read, link to the item.

### 6. Team and access
Port `TeamAccess.tsx` behaviour: invite by email with role (`owner`, `admin`, `editor`, `reviewer`, `viewer`), per-feature access matrix (17 features, levels none/view/edit/full with the plain-language meanings from `permissions.ts`), per-member monthly credit cap (P10), suspend/restore, remove. Owner can transfer ownership. Roles' default levels documented in the UI. **Owner-only "view as" preview** (permission preview, not impersonation: actions still happen as the real user).

### 7. Board (optional here)
The Jira-style board is internal tooling; skip for v1 (P13) unless Agency customers ask.

## Deliverables
Review workflow and screen, AI revision, comments, notifications with email, team screens, view-as.

## Acceptance
- [ ] A maker cannot approve their own item, even by calling the API directly (database test)
- [ ] A reviewer notified by email and in-app within 1 minute; clicking opens the item
- [ ] A user who opted out of an email type receives none
- [ ] Removing a member revokes access on their next request
- [ ] Starter org sees simple approve; Pro org sees maker-checker
- [ ] Bulk approve/send-back works for 20 items and is audited
- [ ] Isolation suite covers `review_requests`, comments, attachments, notifications

## Traps
- Never let the client decide "who may approve"; compute it in a database function.
- Email deliverability: SPF, DKIM, DMARC must be correct before launch; send from a subdomain.
- A notifications table with a "for all" policy hid problems before; policies per command.

## If time is short
Minimum: submit, approve, send back with a note, in-app notifications, simple email. Postpone comments threads, bulk actions, digests.

## Kickoff prompt for Claude Code
> Read `CLAUDE.md`, `phases/P08-review-and-team.md`, `docs/board-maker-checker-test-plan.md` and `docs/team-collaboration-prd.md` in the old repo. Implement the review model with the maker-checker rule enforced in the database and a test that calls it directly, then the Creative Review screen, the notification pipeline (queue, in-app, SES email, prefs) and the team screens. Show failing tests first for any rule that is only enforced in the UI.
