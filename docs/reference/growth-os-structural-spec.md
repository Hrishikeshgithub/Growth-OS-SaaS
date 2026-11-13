# ScalePods Growth OS — Structural Specification

> Source: the live app (https://scalepods-marekting-engine.vercel.app/) and this repository, read on 2026-10-27.
> **Read first:** the prompt this spec answers lists modules (Email Marketing, WhatsApp CRM, Voice Agent, LinkedIn Outreach, Campaigns, Lead Scraper, flat Leads list, Credentials) that **do not exist in this app**. Growth OS is ScalePods' own *content-marketing and publishing* system. This document specifies what actually exists, in the 8 requested sections, and maps the prompt's modules to the nearest real ones in Section 8. Where the prompt's template assumes something the app does not do (a module switcher, a mobile drawer, a date-range picker), the real behaviour is documented instead and flagged.

---

## 1. Tech Stack & Foundation

### 1.1 Stack to build on (as the prompt requires) vs. what exists today

| Concern | Build target (per prompt) | Existing app today |
|---|---|---|
| Framework | Next.js (App Router) + TypeScript | Vite 5 + React 18 + TypeScript, single-page app on Vercel (`vercel.json` rewrites every path to `index.html`) |
| Routing | App Router folders | `react-router-dom` v6, one `<Routes>` block in `src/App.tsx`, every page lazy-loaded |
| Styling | Tailwind CSS | Tailwind 3.4 + a CSS-variable theme in `src/index.css` |
| Components | shadcn/ui | Hand-rolled primitives in `src/components/ui.tsx` (Button, Card, Panel, Badge, Spinner, PageHeader, Modal, EmptyState) |
| Icons | lucide-react | lucide-react 0.441 |
| Charts | recharts | recharts 2.12 (bar charts only today) |
| Dates | date-fns | none; native `Date` / `toLocaleString` |
| Font | Inter | Inter (self-hosted woff2) + Instrument Serif Italic for hero accent words |
| Also present | — | TanStack Query (data + realtime invalidation), Supabase (Auth/Postgres/Storage/Realtime), framer-motion, lottie-react, react-easy-crop, Tiptap (blog editor), pdfjs-dist |

### 1.2 Design tokens (dark is default; light via `data-theme="light"`)

| Token | Dark | Light | Notes |
|---|---|---|---|
| background | `#04070D` | `#F8FAF7` | page |
| foreground | `rgba(255,255,255,0.92)` | `rgba(4,7,13,0.88)` | primary text; secondary 0.62 / 0.60, muted 0.38 / 0.36 |
| card | `#080A0E` | `#FFFFFF` | cards |
| panel | `#10131C` | `#EEF3EA` | nested surfaces, inputs, tiles |
| layer 3 | `#191D2A` | `#E4EBDD` | deepest surface |
| primary | Sage `#B1D997` | `#8FBC6A` | primary button fill (dark-green text on it), success, "live" |
| secondary / accent-2 | Electric `#63A5E7` | `#408CD6` | info, scheduled, secondary charts |
| destructive / warning | Terracotta `#CC6B49` | `#CC6B49` | failures, warnings, pending-review counts, partner badges |
| muted | white @ 0.38 | ink @ 0.36 | helper text |
| border | white @ 0.09 | ink @ 0.08 | hairlines |
| fill-tertiary | white @ 0.07 | ink @ 0.045 | bar tracks, chips |
| radius | card **20px**, panel **14px**, modal **28px**, xl2 1.25rem | same | |
| shadow | `glass`: `0 8px 32px rgba(0,0,0,.35)` + 1px inner top highlight | — | |
| easing | standard `cubic-bezier(.4,0,.2,1)`, spring `(.34,1.56,.64,1)` | — | |

Tailwind aliases: `page`, `card`, `panel`, `sage`, `electric`, `terracotta`, `ink`, `muted`, `line`. Utility classes in CSS: `.card`, `.panel`, `.badge`, `.btn-primary`, `.btn-ghost`, `.input`, `.label`, `.nav-item`, `.sidebar`, `.accent-serif`.

### 1.3 Accent-colour system

| Purpose / module | Colour |
|---|---|
| Primary action, Success, "Live/Connected", Published, Done | Sage (`--accent-green`) |
| Info, Scheduled, In-progress, secondary data | Electric blue (`--accent-blue`) |
| Failure, Stuck, Warning, Pending review, Error | Terracotta (`--accent-orange`) |
| Instagram | gradient (IG_GRADIENT) |
| Facebook | `#1877F2` |
| LinkedIn (also `linkedin_article`, `linkedin_pdf` types) | `#0A66C2` |
| YouTube | `#FF0000` |
| X | `#000000` |
| Content type chips | static_image → sage, carousel → electric, social_caption → terracotta, motion_graphics → electric, product_video → sage, ugc_video → terracotta |
| Trend source dots | Google Trends `#4285F4`, Google News `#EA4335`, Reddit `#FF4500`, Instagram `#E1306C`, YouTube `#FF0000`, LinkedIn `#0A66C2`, Facebook `#1877F2` |
| Team role accents | owner/admin → sage; designer/writer → electric |

Brand rule: only these tokens; primary button = sage fill + dark-green text; Instrument Serif Italic only for hero accent words.

---

## 2. Route Map

All routes except `/login` and `/reset-password` are wrapped in `<Protected>` (signed in → active team record → per-feature `view` permission). Missing permission renders an in-shell "No access" explanation, not a redirect.

| Path | Page Name | Parent Module (nav group) | Purpose |
|---|---|---|---|
| `/login` | Login | Public | Email+password and Google sign-in, password reset request. Signed-in users are bounced to `/` |
| `/reset-password` | Reset password | Public | Set a new password from the emailed link |
| `/` | Dashboard | Overview | KPI tiles, this-week calendar strip, recent activity, platform status, pipeline shortcuts |
| `/clients` | Business profiles | Marketing Strategy | List of brand knowledge bases (one profile seeds every engine) |
| `/clients/new` | New business profile | Marketing Strategy | Create a profile |
| `/clients/:id` | Business profile | Marketing Strategy | Edit brand, audience, competitors, social URLs, assets |
| `/intelligence` | Intelligence | Insight | Latest AI business analysis + history |
| `/intelligence/:id` | Intelligence report | Insight | One 7-part business-intelligence report |
| `/trends` | Trend Intelligence | Marketing Strategy | Scan Reddit/Instagram/YouTube/Google Search/Google Trends, rank signals, select trends |
| `/trend-alerts` | Trend Alerts | Marketing Strategy | Watch items (keywords × sources) and AI-filtered alerts |
| `/trend-alerts/:id` | Watch item detail | Marketing Strategy | One watch: events, rules, run history, stats |
| `/strategy` | Marketing Strategy | Marketing Strategy | Strategy generations (list ↔ detail), 7-part plan, calendar, approve |
| `/studio` | AI Studio | Content Generation | Brief → prompt → image(s)/carousel generation with style gallery |
| `/carousel-studio` | Carousel Studio | Content Generation | Outline → rendered branded slide carousel |
| `/video-studio` | Video Studio | Content Generation | Storyboard → motion-graphics or AI-generated video |
| `/content` | Content Factory | Content Generation | Bulk copy+image generation from the approved calendar (hidden from nav; route live) |
| `/review` | Creative Review | Content Generation | Approval gate: approve / send back, assign reviewer, bulk actions |
| `/board` | Board | Team | Jira-style Kanban/List of team tickets |
| `/board/:key` | Board ticket | Team | Same board with one ticket's detail window open (deep link) |
| `/calendar` | Calendar | Publishing Engine | Month grid of scheduled/published posts; create post on a day |
| `/publishing` | Publishing | Publishing Engine | Approved items ready to post/schedule + activity log |
| `/blog` | Blog | Publishing Engine | Posts for scalepods.co |
| `/blog/new` | New blog post | Publishing Engine | Rich-text editor |
| `/blog/:id` | Edit blog post | Publishing Engine | Same editor, existing post |
| `/analytics` | Analytics | Insight | Post performance, Instagram comment leads, AI insights |
| `/settings` | Settings | Insight | Account, appearance, credit-safety flags, track an Instagram post |
| `/settings/team` | Team & access | Insight (admin only) | Invite people, set role, per-feature access, monthly spend cap |
| `/manual` | User Manual | Insight (ungated) | One card per screen with how-to steps |
| `*` | — | — | Redirects to `/` |

Not routes but full-screen states: loader, **Account pending** (invited / suspended / no record / lookup failed) and **No access**.

---

## 3. App Shell & Navigation Model

> **Deviation from the prompt:** there is **no module-switcher dropdown and no contextual per-module nav**. The sidebar is one flat list, grouped by pipeline stage. The only switcher at the top is the **business-profile switcher** (which business all pages operate on).

### 3.1 Layout
- `flex min-h-screen`: **sidebar | main column**.
- **Sidebar** `sticky top-0 h-screen`, border-right hairline.
  - **Pinned: 256px** (`w-64`). **Unpinned: 72px icon-only rail** that expands to 256px on hover and collapses when the cursor leaves. The pin choice persists in `localStorage` (`sp-sidebar-pinned`, default pinned). Hover state is session-only.
  - Top block (padding 20px): logo (full wordmark when expanded, icon when collapsed) + pin/unpin button (visible only when expanded).
  - Business-profile switcher (below logo): button showing the active profile; dropdown (w-224px) lists profiles with a sage dot + check on the active one, and a "New profile" link to `/clients/new`.
  - Scrolling nav region (groups with 10px uppercase muted headers, hidden when collapsed).
- **Main column**: optional orange "Viewing as …" banner → **top bar** → `<main>`.
  - **Top bar:** `px-8 py-4`, bottom hairline (≈69px tall incl. 36px avatar). Left spacer, then right-aligned: notification bell → owner-only "View as" eye button → theme toggle (sun/moon) → name + role (hidden below `sm`) → avatar (image or gradient initials) → sign-out icon.
  - **Content:** `max-w-[1680px]`, centred, padding 32px (`p-8`).
- **Responsive:** *no mobile drawer or backdrop exists.* Below desktop widths the sidebar stays as the 72px rail; page grids collapse via `grid-cols-2 sm:grid-cols-4` style breakpoints. (A drawer is a gap to decide in Section 8.)

### 3.2 Nav list (single list, filtered by permission)

| Group header | Item (icon) → route | Notes |
|---|---|---|
| *(none)* | Dashboard (LayoutDashboard) → `/` | |
| Marketing Strategy | Business (Building2) → `/clients` — count badge = profiles | |
| | Trends (TrendingUp) → `/trends` | |
| | Trend Alerts (BellRing) → `/trend-alerts` | |
| | Strategy (Target) → `/strategy` | |
| Content Generation | AI Studio (Wand2) → `/studio` | |
| | Carousel Studio (Clapperboard) → `/carousel-studio` | |
| | Video Studio (Film) → `/video-studio` | |
| | Creative Review (CheckSquare) → `/review` — orange count badge = pending review | Content Factory is deliberately not in the nav |
| Team | Board (KanbanSquare) → `/board` | |
| Publishing Engine | Calendar (CalendarDays) → `/calendar` | |
| | Publishing (Send) → `/publishing` | |
| | Blog (Newspaper) → `/blog` | |
| Insight | Analytics (BarChart3) → `/analytics` | |
| | Intelligence (BrainCircuit) → `/intelligence` | |
| | Settings (Settings) → `/settings` | |
| | Team & access (Users) → `/settings/team` | admin/owner only |
| | User Manual (BookOpen) → `/manual` | always visible |
| | Support AI (MessageCircleQuestion) → external URL, new tab | |

- Items show only if the person has `view` on that feature; empty groups vanish. Active item uses `nav-item.active`; `/settings` uses exact matching so it is not highlighted on `/settings/team`.
- Count badges: collapsed rail shows a 8px orange dot instead of the number.

### 3.3 Bottom of the sidebar
The sidebar has **no pinned bottom actions** (no global Call/Dialer, no Logout there). Global actions live in the **top bar**: notification bell, View-as (owner), theme toggle, sign-out. Global *create* actions are in-page (e.g. "Create post" opens the composer modal on Calendar/Publishing).

### 3.4 Top-bar conditions
- **Notification bell**: always, for any signed-in active user (in-app notifications with unread count).
- **View-as eye**: only when the *real* role is `owner`.
- **Preview banner**: only while an owner is previewing a teammate's permissions.
- There are **no balance/usage pills** in the header. Spend is shown inside the generation pages (cost bar, "$X of your $Y monthly budget used") and in Settings/Team.

---

## 4. Page Anatomy Pattern

> The app has no single "module overview" template. Three repeatable templates exist; each is documented in the prompt's order where a real equivalent exists.

### 4.1 Standard page frame (every page)
1. **PageHeader**: optional accent `Badge` (icon + module name) → `h1` (24px) → optional subtitle (muted, max-w-2xl) → right-aligned `actions` (ghost buttons such as "Refresh now").
2. **No-profile guard**: if no business profile exists, the page shows `EmptyState` "No business profile yet — Create the business profile first." instead of content.
3. Content sections separated by 16–32px.

### 4.2 Overview/metric pattern (Dashboard, Analytics)
1. **Header**: title, subtitle, action. Dashboard: "Welcome back to *Growth OS*" (serif accent). Analytics: `Analytics — {business}` + "Last refreshed {time}" + **Refresh now**. *No date-range picker exists.*
2. **Metric card row**: `grid-cols-2 sm:grid-cols-4 gap-3`, **4 tiles**. Each tile = icon chip (accent-coloured), big number, label. Dashboard: *Approved & ready · Published this month · Total engagement · Pending review*. Analytics: *Total engagement · Likes · Comments · Views/reach*. Tiles are plain (`card metric-tile p-4`: small icon + muted label on one line, 24px light-weight number below); they are not click-through today.
3. **Breakdown**:
   - Dashboard: `grid lg:grid-cols-[1.6fr_1fr]` — left card "This week" (7-column day strip, up to 3 item chips/day, link "Full calendar →"); right column stacks **Recent activity** (platform badge + title + status) and **Connected platforms** (platform badge + "Live" dot or "Off").
   - Analytics: a **bar chart** "Engagement by platform" (recharts, 220px high, `chart-clean`, sage bars with 6px rounded tops, no vertical grid), then a **"Top posts"** list (one `Panel` per post: platform badge, post URL, then *likes · comments · engagement*).
4. **Per-sender breakdown table**: the nearest equivalent is the **Instagram leads table** (Section 4.3) and the per-platform tiles.
5. **"Actual content" list**: Dashboard "Recent activity"; Analytics post list; empty state = `EmptyState` with icon + title + hint (e.g. "No analytics yet — Click Refresh now once you have published posts").
6. **Usage/quota card**: not a standalone card; spend appears as a **CostBar** (shots · seconds · $/sec · total in USD + INR) and a "$X of $Y monthly budget used" line with orange text and disabled buttons when over cap.
7. **Pipeline shortcut grid** (Dashboard only): `sm:grid-cols-2 lg:grid-cols-3 2xl:grid-cols-4` cards (icon chip, title, one-line description, arrow that turns sage on hover), one per pipeline stage the user may view.

### 4.3 List / log page templates
- **Tile grid (Publishing, Creative Review, Video Studio "Recent videos")**: `grid-cols-3 sm:grid-cols-4 lg:grid-cols-5 2xl:grid-cols-6 gap-1.5` of `PostTile` (square media preview or text placeholder, platform badge, content-type chip, status badge top-right, relative-time badge bottom-right, optional cost badge). Click → preview modal. Sections: "Ready to publish" grid, then "Recent activity" with **filter pills** (All · Published · Scheduled · Failed, each with a count).
- **Table (Analytics → Instagram comments)**: columns *Username · Comment · Post · When · Keyword · DM · Follower*; header actions = post-filter `<select>`, **Export CSV**, **Sync now**; empty state when no comments.
- **Kanban/List (Board)**: Board ↔ List toggle; columns Backlog → Not Started → In Progress → Selected for Development → In Review → On Hold → Done; list columns *Work · Assignee · Reporter · Priority · Status · Resolution · Created · Updated · Due date*.
- **Status/direction badge rules**: `published` → green (check), `scheduled`/`publishing`/`pending` → blue (clock/spinner), `failed` and `stuck` (publishing > 10 min) → orange (x). Review status: ready / revision / approved. DM sent: boolean chip.

### 4.4 Details modal template
`Modal` (title, close, sizes md/lg/xl/2xl, optional 16:9 lock): **(a)** media block (image, carousel viewer with slide dots, or `<video controls preload="metadata">`) inside a platform mockup using `PLATFORM_ASPECT` (IG 1:1, FB/LinkedIn 1.91:1, YouTube/X 16:9); **(b)** caption/body block with "more" expansion; **(c)** meta strip (platform badge, type chip, status badge, scheduled/published time, link to the live post); **(d)** action row (Approve / Send back with notes / Schedule / Post now / Edit live post / Open in platform). Confirmation modals for spend show a large dollar figure, ≈ rupee figure and a one-line estimate note.

### 4.5 Studio (generation) page template — Video, AI, Carousel Studio
Form card (source picker: trend / strategy / topic; platform; shape; engine/model; counts) → **price bar** → "Write the storyboard/brief" (free) → editable draft cards (one per shot/slide/variant, with prompt, on-screen text, duration) → audio panel (Video) → caption editor (hook/body/CTA/hashtags) → **Generate** (spend confirmation) → progress card (spinner, step label, % and time left, bar) → result + "Send to Review". Per-shot cards carry their own progress bar; "Recent …" grid sits at the top.

---

## 5. Shared Components Inventory

| Component | Props / inputs | Reused in |
|---|---|---|
| `PageHeader` | `title`, `subtitle?`, `accent?`, `actions?` | every page |
| `Card` / `Panel` | `children`, `className` | everywhere (card 20px radius, panel 14px) |
| `Button` | `variant: primary \| ghost`, `loading`, `disabled` | everywhere; primary = sage fill |
| `Badge` | `tone: green \| blue \| orange \| grey`, `className` | status chips, page accents |
| `Spinner` | `size` | loaders, buttons (branded full-screen loader exists separately) |
| `EmptyState` | `icon`, `title`, `hint?` | every list/section |
| `Modal` | `title`, `onClose`, `wide?`, `size`, `aspectVideo?` | composer, previews, confirmations |
| `Tile` (metric card) | `icon`, `label`, `value`, `accent` | Dashboard, Analytics (local component today) |
| `PlatformBadge` | `platform`, `size: sm \| md` | tiles, lists, composer |
| `ContentTypeChip` | `type` | tiles, previews |
| `PostTile` | `img`, `platform`, `placeholder`, `topRight`, `bottomLeft`, `bottomRight`, `busyNote`, `onClick` | Publishing, Review, Video Studio |
| `TimeBadge` | `time` | tiles (relative time) |
| `PostPreviewModal` / `ActivityPreviewModal` / `ReadyPreviewActions` | item/post, `onDone` | Publishing, Calendar, Review |
| `CreatePostModal` (composer) | optional prefill date; multi-platform select, media upload, schedule picker | Calendar, Publishing, Review "Import" |
| `CarouselViewer`, `PlatformMockup` | slides, platform | previews |
| `AssetUploader` | `bucket`, `pathPrefix`, `accept`, `onUploaded` | composer, business profile, blog |
| `MediaEditor`, `MediaSourcePanel` | crop/replace; Upload/Canva/Figma tabs | composer, review |
| `NotificationBell` | — | top bar |
| `Toast` (`useToast`) | `success/error/info` | everywhere |
| `Gate` hooks (`useGate`, `useCan`, `useBudgetGate`) | feature, level / est. cost | any spend or permission-gated control |
| `CostBar` | engine, resolution, seconds, shots | Video Studio (analogue in AI/Carousel Studio) |
| `ShotEditor`, `ProgressBar`, `EtaNote` | shot, live progress, ETA seconds | Video Studio |
| `FeedbackControl` | thumbs + note | shots, audio, carousel slides |
| `VoicePicker` | selected voice, playable sample | Video Studio |
| `Markdown`, `JsonBlock` | text | reports, strategy sections |
| `StrategyCalendarView`, `PillarBalanceChart`, `PlatformCards`, `InsightHeaderStrip`, `SectionEditor`, `GenerateStrategyModal` | strategy data | Strategy only |
| `RichTextEditor`, `BlogPreview`, `SectionCardsNode`, `SectionImageNode` | Tiptap | Blog |

Prompt items with **no equivalent here:** Account/Sender colour pill, DateRangePicker, floating Dialer, Balance/Wallet modal.

---

## 6. Data Model Per Module

(Postgres via Supabase, row-level security on everything.)

**Business profile** — id, business_name, tagline, industry, description, products_services, target_audience, business_goals, brand_guidelines, brand_voice, target_platforms[], competitors, competitor_profiles[] (name, urls…), website_url, social_media_urls{}, assets[{name,url}], additional_notes, phone, email, address, hours, service_areas[], fb_page_id, logo_url, cover_url, status, created_at, updated_at.

**Intelligence (BI report)** — id, profile_id, 7 sub-analyses (JSON), created_at.

**Trend run / signal** — Run: id, profile_id, status, created_at. Signal: id, run_id, profile_id, source, topic, relevance_score, relevance_reason, url, meta{}, created_at.

**Trend alert** — Watch: id, profile_id, name, description, keywords[], exclude_keywords[], sources[] (google_news/reddit/web/youtube/rss), rss_urls[], region, results_per_source, frequency_minutes, lookback_days, relevance_threshold, monthly_budget_usd, status (active/paused), next_run_at, last_run_at, last_run_status, last_error. Event: id, watch_id, kind (story/spike), status, title, summary, explanation, primary_url/source, sources[], mention_count, source_count, relevance, sentiment (+intensity), importance (low…critical), credibility, spike{today, mean_7d, stddev_7d, ratio}, first_seen_at, last_seen_at. Rule: name, min relevance, delivery (immediate/daily/weekly), channel (app/slack/webhook). Run, Stat rows.

**Strategy generation** — id, profile_id, status (processing/completed/approved/failed), source_signal_ids[], source_signals_snapshot[], scope (day/week/month), platform, content_type, ai_summary, campaign_planning, weekly_content_strategy, content_pillars, content_calendar[] (scheduled_date, platform, content_type, title, hook, pillar), platform_strategy, lead_generation_strategy, cta_strategy, header_insights, pillar_balance (recommended vs actual HR/Sales/Ops/Marketing), error_detail, created_at.

**Content item** — id, run_id, profile_id, strategy_id, calendar_index, content_type (static_image, carousel, ugc_video, motion_graphics, product_video, blog, social_caption, linkedin_article, website_content, email, story, linkedin_pdf), status (pending, generating, ready, in_review, approved, revision, failed, published, scheduled, publishing), platform, scheduled_date, title, body, media_url, thumbnail_url, metadata{slides, hashtags, comment_automation, …}, review_notes, revision_count, error_message, approved_at, reviewer_id, submitted_by, sent_for_review_at. Attachments: item_id, file/link.

**Scheduled post** — id, content_item_id, profile_id, platform, caption, media_url, title, platform_post_id, post_url, post_type, status (scheduled/publishing/published/failed), scheduled_time, published_at, error, retry_count, ai_best_time.

**Post analytics** — id, content_item_id, scheduled_post_id, profile_id, platform, content_type, platform_post_id, post_url, impressions, reach, likes, comments, shares, saves, video_views, clicks, engagement, engagement_rate, fetched_at. **AI insight**: generated_at, posts_analyzed, overall_summary, content_scores, winning_hooks, best_posting_time.

**Instagram lead (comment)** — id, content_item_id, media_id, comment_id, commenter_id, commenter_username, comment_text, like_count, commented_at, matched_keyword, dm_sent, follow_gate_status, follower_count, is_verified_user, profile_pic_url, enriched_at.

**AI Studio job** — id, profile_id, status (drafting/draft_ready/generating/done/failed), source_kind (trend/strategy/topic), source_signal_id, topic, platform, aspect_ratio, style_id, model, reference_image_url, character_id, variant_count, copy_json{hook, body, hashtags, cta}, image_prompt, image_urls[], selected_image_url, post_type (single/carousel), slide_count, slides_json[], image_feedback, content_item_id, error_detail.

**Carousel job** — id, profile_id, topic, comment_keyword, platform, status, outline_json[] (type cover/step/stat/cta + fields), slide_urls[], render_progress, error_detail.

**Video job** — id, profile_id, source_kind, topic, platform, aspect_ratio, video_type (motion_graphics/generated_clips), status, outline_json, engine (veo-3.1 lite/fast/standard), resolution, video_style_id, shots_json[] (index, prompt, onScreenText, negativePrompt, durationS 4/6/8, status, clipUrl, costUsd, errorDetail, startedAt, feedback), estimated_cost_usd, copy_json, voiceover_script/url, voice, music_prompt/url, render_progress, final_video_url, error_detail, content_item_id.

**Blog post** — id, title, slug, category, excerpt, meta_title, meta_description, banner_url(+dark/light), sections[], faqs[], status, published_at.

**Board** — Column: id, name, position, is_review, is_terminal. Ticket: id, key, profile_id, title, description, type (task/design/copy/video/blog/bug), priority (highest…lowest), column_id, position, reporter_id, assignee_id, reviewer_id, start_date, due_date, labels[], linked_kind/linked_id, submitted_at, accepted_at, rejection_note. Comment, Activity, Attachment.

**Team & access** — AppUser: id, auth_user_id, email, full_name, avatar_url, role (owner/admin/designer/writer/client), status (invited/active/suspended), monthly_spend_cap_usd, email_notifications, last_seen_at. Permission map: feature → none/view/edit/full across 17 features. **Spend event**: source (studio/carousel_studio/video_studio), job_id, amount_usd, user_id. **Notification**: type, title, body, item_id, link, read_at.

---

## 7. Dummy Data Plan

| Area | Rows / shape |
|---|---|
| Business profiles | 3 (ScalePods plus two sample brands), one fully filled |
| Trend signals | 20–30 in one run, spread across 5 sources, relevance 0.4–0.95 |
| Trend alerts | 4 watch items; 12–20 events (mix of story/spike, 2 critical) |
| Strategies | 3 (one approved, one completed, one failed); 12–20 calendar items |
| Content items | 25–40 across all statuses; 6 in review, 3 in revision |
| Scheduled posts | 20–30: 12 published, 8 scheduled, 3 failed, 1 stuck |
| Post analytics | one row per published post; impressions 800–45,000; engagement rate 1.2–6.8% |
| Instagram leads | 30–60 comments over 5 posts; ~60% matched a keyword, ~45% DM'd |
| Studio / Carousel / Video jobs | 6–10 each; Video: 1 done, 1 rendering (shot 2 of 3 in progress), 2 draft, 1 failed with the "render service offline" message |
| Board | 7 columns, 15–25 tickets, 4 people |
| Team | 5 users: owner, admin, designer, writer, client (one invited, one suspended) |

Constraints so every teammate's mock data matches:
- **At least 3 platform "accounts"** where a multi-account view appears (Calendar/Publishing LinkedIn: *Adnan*, *Raunak*, *ScalePods Page*) with post counts that sum to the platform total.
- **Realistic funnels**, never round: e.g. 38,420 impressions → 4,113 engagements → 212 comments → 61 keyword matches → 44 DMs sent → 9 conversations.
- **Populated vs empty:** Dashboard, Publishing and Analytics fully populated; **Analytics → AI Insights** and **Creative Review → "sent back"** shown in the **empty** state; one profile with zero content to show the no-data screens.
- **Charts:** 30 days of engagement with a gentle upward trend (+8–15%), weekend dips and two spikes; never flat or uniformly random.
- Currency shown as USD with INR beside it (rate 94.6).

---

## 8. Open Questions / Assumptions

1. **Product mismatch (decide first).** The prompt describes an outreach/sales-engagement product (Email Marketing, WhatsApp CRM, Voice Agent + dialer, LinkedIn Outreach, Campaigns, Lead Scraper, Leads, Credentials). Growth OS has none of these. Is the team rebuilding **Growth OS** (this spec) or a **different product**? The n8n instance contains `[Spectra]` outreach workflows (WhatsApp, Email via Instantly, calls), which suggests that product exists separately — its code was not part of this analysis.
2. **Module mapping used here:** Leads → *Instagram comments* in Analytics; Campaigns → *Strategy calendar*; LinkedIn Outreach → *LinkedIn publishing*; Credentials → *Settings*; Email/WhatsApp/Voice/Lead Scraper → **no equivalent**.
3. **Target stack vs current stack.** The prompt says Next.js App Router + shadcn/ui + date-fns; the app is Vite + React Router with hand-rolled primitives. Section 1 lists both; teammates should confirm a true port is intended, since auth guards, lazy routing and Supabase Realtime sync would need re-expressing.
4. **No module switcher.** Section 3 documents the real flat, grouped sidebar. Confirm whether the contextual-per-module sidebar is a desired *change*.
5. **Mobile drawer + backdrop do not exist.** Choose: keep the 72px rail, or add a drawer.
6. **Header usage pills / Balance modal do not exist.** Spend lives inside generation pages and Settings/Team. Decide whether to add a global spend indicator.
7. **Date-range picker and dialer are absent.** Treated as out of scope unless requested.
8. **Colour for X** is black (`#000000`); on the dark theme it needs a border or light chip to stay visible.
9. **Permission model** (5 roles × 17 features × none/view/edit/full) is richer than the prompt implies; assumed to be in scope for the rebuild.
10. **Pricing/time figures** (Veo rates, time estimates) are code constants, not API-driven; mock data should reuse the same numbers.
11. **Hidden but live:** Content Factory (`/content`) has a route and permission but no nav link.
12. **Unverified here:** the live site sits behind a login, so page layouts were read from source, not seen rendered, apart from the Video Studio screen checked earlier.
