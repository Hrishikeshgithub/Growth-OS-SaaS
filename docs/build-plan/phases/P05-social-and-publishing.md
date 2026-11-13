# P05 — Social connections and publishing

**Goal:** a customer connects their own social accounts with one click, composes a post (single image, carousel, video, text, PDF where supported), schedules it, sees it on a calendar, and it publishes reliably — with visible, retryable failures. **Est:** 10–12 working days. **Depends on:** P01–P04.

## Step 0 — Choose the social provider (2 days, do this first)
1. Open trials with **bundle.social** and **Zernio** (inputs #12). Use ScalePods' own accounts.
2. Run the checklist from the approved solution doc; record results in `docs/adr/0003-social-provider.md`:
   - [ ] Instagram Reel, carousel and Story publish
   - [ ] Facebook photo, multi-photo and Reel publish, **visible to a signed-out visitor**
   - [ ] LinkedIn personal post and Company Page post, including a PDF document post
   - [ ] YouTube Short publish
   - [ ] X post with a link, and what it costs per post
   - [ ] Instagram **comment-to-DM**: read a new comment and DM the commenter (decides P09)
   - [ ] Analytics returned per platform
   - [ ] Which app name the customer sees on each consent screen
   - [ ] Webhooks for post status and comments; signature scheme
   - [ ] Rate limits, max media sizes, scheduling support (native or ours)
3. Pick one; keep **Ayrshare** as the tested-on-paper fallback. Whatever is picked, we build behind our own interface so it can be replaced.

## Steps

### 1. Provider adapter (`packages/social`)
Define the interface in 02-architecture §3. Implement the chosen adapter plus a **fake adapter** (in-memory) used in tests and local dev. Normalize errors to our codes: `AUTH_EXPIRED`, `RATE_LIMITED`, `MEDIA_INVALID`, `MEDIA_TOO_LARGE`, `PLATFORM_REJECTED`, `PROVIDER_DOWN`, `UNKNOWN`, each with a human message and a retryable flag.

### 2. Tables (migration)
`social_accounts` (org_id, brand_id, provider, provider_account_id, platform, display_name, avatar_url, status active|needs_reconnect|disconnected, last_checked_at, connected_by), `scheduled_posts`, `publish_attempts`, `post_edits`, `media_assets` links. RLS pattern + plan-limit trigger on channels. Provider webhook secrets in secrets store.

### 3. Connect flow
Settings → Channels → "Connect Instagram" etc.: API route asks the provider for a connect link scoped to `{org, brand}`; opens it; the provider redirects back to `/api/social/callback`; the route **verifies state** (signed, single-use, tied to the session), fetches accounts, and upserts `social_accounts`. A daily **health check job** (`social.token_check`) pings each account; failures flip status to `needs_reconnect`, notify owners/admins, and show a "Reconnect" button. (Lesson #6.)

### 4. Composer (port the UX of `CreatePostModal.tsx`)
Multi-platform select; per-platform previews with the right aspect ratio and caption limits (`PLATFORM_ASPECT` and limits from the old `mediaUi.tsx`/`postPreview.tsx`); media upload (P04 presign, normalize); carousel ordering; video; LinkedIn PDF document; Facebook Reel vs Video; hashtags; first-comment option; **smart scheduler**: only future slots (≥ 5 min ahead), time zone shown, past dates disabled. Drafts auto-save server-side (not only `localStorage`, which lost drafts before).

### 5. Scheduling and publishing engine (worker)
- `publish.scheduler` runs **every minute**: select due `scheduled` rows and **claim** them atomically (`update … where status='scheduled' … returning`), then enqueue `publish.post`.
- `publish.post`: re-load post and account, re-check org and account status, resize/validate media for the platform, call the adapter, store the provider post id **before** marking `published`, write a `publish_attempts` row.
- Failure handling: retryable errors → retry with back-off (max 3); `AUTH_EXPIRED` → `failed` + reconnect prompt, **no** retry; media errors → one automatic retry after re-normalizing; everything else → `failed` with reason.
- Rescue job for rows stuck in `publishing` > 10 min (03 §3). "Post now" uses the same path with `scheduled_time = now`.

### 6. Calendar and Publishing screens
Calendar: month grid, post chips colour-coded by status, "create on this day". Publishing: "Ready to publish" tile grid and "Recent activity" with filters (All, Published, Scheduled, Failed) and counts; preview modal with live post link, retry on failed, reschedule, cancel. Stuck/failed states are explicit. Port visuals from `Calendar.tsx`, `Publishing.tsx`, `postPreview.tsx`.

### 7. Edit live post
Only where the provider supports it (LinkedIn, YouTube today). Server records an edit row, calls the adapter, updates our copy only on success. Admins/owners only.

### 8. Notifications
Enqueue `notify.send` on publish success/failure (in-app always; email per preference) — full notification center in P08.

### 9. Webhooks in
`/api/webhooks/social` with the P04 framework: post status updates, comment events (stored for P09), account revoked events (set `needs_reconnect`).

## Deliverables
Provider decision record; adapter + fake; connect flow; composer; engine; calendar and publishing screens; health checks; webhooks.

## Acceptance
- [ ] A new test account connects Instagram, LinkedIn and Facebook and publishes to each without help
- [ ] A scheduled post publishes within 1 minute of its time; "Post now" within 30 s
- [ ] Killing the worker during publish never causes a duplicate post (test by retrying the job)
- [ ] A revoked token shows "Reconnect" within 24 h (simulate in the fake adapter) and sends a notification
- [ ] Past times cannot be scheduled; time zone is displayed
- [ ] A 14-megapixel image publishes (normalized) on every platform
- [ ] Org B cannot see or publish with org A's accounts (isolation suite extended)
- [ ] Failed posts show the reason and a working Retry

## Traps
- Never trust a provider's "ok" until the post id is stored.
- Don't poll providers in a tight loop; use webhooks and a few status polls.
- Platform visibility (Facebook, Instagram) can differ between a connected admin and a stranger; always test signed-out.
- Don't store social tokens ourselves with a provider; store the account id only.
- Buffer-style "share now" differences: confirm the provider publishes immediately when asked.

## If time is short
Order of value: Instagram image/carousel, LinkedIn text/image, Facebook photo; then video/Reels, PDF, Stories, YouTube, X. Postpone edit-live-post and first-comment.

## Kickoff prompt for Claude Code
> Read `CLAUDE.md`, `phases/P05-social-and-publishing.md`, `03-data-model.md` §3 and the old repo's `CreatePostModal.tsx`, `postPreview.tsx`, `publishing.ts` for UX. First create the `SocialProvider` interface, the fake adapter and the error taxonomy with tests. Then the tables with RLS, the connect/callback routes with signed state, the scheduler/claim logic with a duplicate-prevention test, and finally the composer, calendar and publishing screens. Do not wire the real provider until I confirm the ADR.
