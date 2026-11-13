# P09 — Analytics, leads and automation

**Goal:** close the loop: measure what was published, learn from it, capture leads from Instagram comments, and send the right DM automatically. **Est:** 6–8 working days. **Depends on:** P05 (published posts, provider webhooks), P04. The comment-to-DM part depends on the P05 provider test result.

## Steps

### 1. Analytics sync
Job `analytics.sync` every 12 hours per active account (and "Refresh now", rate-limited): for each published post fetch metrics through the provider adapter (`impressions, reach, likes, comments, shares, saves, video_views, clicks, engagement, engagement_rate`) and upsert `post_analytics` on `(org_id, provider_post_id)`. Record `analytics_sync_state` per account (last run, status, count). Respect the plan's history window (90 d / 1 y / 2 y) when showing.

### 2. Analytics screens
Port from `Analytics.tsx`: four metric tiles (Total engagement, Likes, Comments, Views/reach), **Engagement by platform** bar chart, **Top posts** list, per-post drill-down, date range picker (new: 7/30/90 days, custom), channel filter, CSV export. Empty state until the first sync.

### 3. AI insights ("learning engine")
Job `insights.generate`: after enough measured posts, an LLM summarizes what works (content scores, winning hooks, best posting times, pillar performance vs target). Output stored in `ai_insights` and **fed back automatically**: best times into the scheduler suggestions (P05), winning hooks into copy prompts (P07), `pillar_balance` and `header_insights` into Strategy (P06). Port the prompt from "AI Insights". Require a minimum number of posts (e.g. 10) before generating; say so in the UI.

### 4. Leads and comment-to-DM — model
Tables: `comment_automations` (post reference, keyword list with fuzzy matching, DM message, optional asset link/file, **follow gate** config, **public reply** config, enabled), `leads`, `automation_events` (unique on the provider's comment id — the claim-before-send dedupe). Plan limits on number of enabled automations.

### 5. Comment-to-DM — flow (port the logic from the old "Comment-to-DM" workflow)
1. Provider webhook delivers a comment event → verify signature (P04 framework) → insert `automation_events` with the comment id; a duplicate returns immediately.
2. Ignore comments authored by our own account (a bug source: our public reply arrives as a comment).
3. Find the automation for that post; **fuzzy keyword match** (case-insensitive, trims punctuation, optional typos).
4. Insert/upsert the `lead` (username, text, matched keyword, `dm_status = pending`).
5. Send the DM through the adapter. **Only after the provider confirms**, set `dm_status = sent`; on failure set `failed` with the reason (never mark sent first — lesson #5).
6. If public reply enabled, post it (best effort; failure does not fail the DM).
7. **Follow gate (optional):** first DM asks to follow with a quick-reply button; when the person taps it, check follow status (only possible for someone in an active conversation) and send the link, else re-prompt (limit attempts). Idempotent on duplicate taps.
8. Private replies must go out within 7 days of the comment; a daily job marks older pending leads `expired`.
9. **Track an existing post by URL:** resolve the post through the adapter, then attach an automation.

### 6. Leads screen
Table: username, comment, post, when, keyword, DM status chip (sent/failed/pending/expired), follower info if available; filters by post, CSV export, "Sync comments now" (backfill through the adapter), retry failed. Mark rows clearly when the automation was disabled or the account needs reconnect.

### 7. Connection-health alerts
If the provider reports the account unusable, flip the automation to paused, notify owners/admins, and show a banner (daily token/health job from P05). (Lesson #6.)

### 8. Privacy
Leads are personal data: store only what is needed, retention setting per org (default 12 months), included in data export and deletion (P11).

## Deliverables
Analytics sync and screens; insights loop; automations, event dedupe, leads screen, follow gate.

## Acceptance
- [ ] After a real post gets engagement, numbers appear within one sync and match the platform
- [ ] A test comment with the keyword produces exactly one DM (send the same webhook twice: still one)
- [ ] A failed DM shows `failed` with a reason, never `sent`
- [ ] Our own public reply does not trigger another DM
- [ ] Expired token → automation paused, owner notified, UI shows reconnect
- [ ] Insights unlock only after the minimum post count and feed the scheduler suggestion
- [ ] Isolation suite covers leads, automations and analytics

## Traps
- Providers may send both comment and read-receipt/echo events; ignore what you don't handle.
- Fuzzy matching can cause false triggers; show a test box ("would this comment trigger?").
- Follow status is only available for existing conversations; do not promise a pre-DM follow check.

## If time is short
Launch with analytics + leads capture + basic keyword → DM (no follow gate, no public reply). Insights loop right after.

## Kickoff prompt for Claude Code
> Read `CLAUDE.md`, `phases/P09-analytics-leads-automation.md`, the P05 provider ADR, and the old repo's `docs`/workflow notes for Comment-to-DM (memory `comment-to-dm-instagram`). Implement analytics sync and screens first. Then the automation tables and the webhook flow with a duplicate-event test and a "mark sent only after provider confirms" test, then the leads screen. Ask me before enabling follow gate.
