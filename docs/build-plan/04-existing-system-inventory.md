# Existing system inventory (what to port, and from where)

Source repo: `G:\1 Scalepods Client Projects\ScalePods Growth OS` (GitHub `Scale-Pods/Scalepods_Marekting_Engine`). Paths below are relative to it. **Port logic, prompts and templates; do not port the hard-coded ScalePods values or the architecture shortcuts** (see `05-risks-and-lessons.md`).

Legend — **Port**: copy and adapt. **Rewrite**: take the behaviour, build fresh. **Drop**: not in the SaaS.

## 1. Screens and modules

| Module | Current route / page | Key source files | New home | Port or rewrite |
|---|---|---|---|---|
| Login, reset, pending/no-access walls | `/login`, `/reset-password` · `src/pages/Login.tsx`, `AccountPending.tsx`, `NoAccess.tsx`, `src/lib/auth.tsx` | | P01–P02 | Rewrite (Supabase SSR) |
| Team, roles, per-feature access, view-as | `/settings/team` · `TeamAccess.tsx`, `src/lib/team.ts`, `permissions.ts` (17 features, levels none/view/edit/full) | | P01, P08 | Port the feature list and level meanings |
| Business profile | `/clients`, `/clients/:id` · `BusinessProfile.tsx`, `Clients.tsx`, `src/lib/clients.ts` | | P03 | Port fields → `brand_profiles` |
| Intelligence | `/intelligence` · `Intelligence.tsx`, `IntelligenceReport.tsx`; n8n "AI Analysis" | prompt in the n8n workflow (7 sub-analyses) | P06 | Port prompt, rewrite as a job |
| Competitors + AI discovery | in Business profile; n8n "Competitor Search" (`jkyRS1nbiGOJ9BQn`) | two-pass search + verification | P06 | Port the algorithm |
| Trends | `/trends` · `Trends.tsx`, `src/lib/trends.ts`; n8n "Trend Intelligence" + "Daily Scheduler"; `trend-scraper/app.py` | 5 Apify sources, GPT ranking | P06 | Port prompts and source list; rewrite jobs |
| Trend Alerts | `/trend-alerts` · `TrendAlerts.tsx`, `src/lib/trendAlerts.ts`; `docs/radar-prd.md`; n8n `RUSAvRcKGy1H1zLA`; SQL pipeline in Postgres | clustering, spike detection, rules, delivery | P06 | Read the PRD first; rewrite |
| Strategy | `/strategy` · `Strategy.tsx`, `src/components/strategy/*`, `src/lib/strategy.ts`; n8n "Marketing Strategy", "Strategy Generation", "Strategy Section Regenerate" | 7-part plan, pillar balance, header insights, calendar | P06 | Port prompts and JSON shapes |
| Content Factory (hidden) | `/content` · `ContentFactory.tsx`; n8n "Content Text Engine", "Content Image Engine", "Branding Overlay" | calendar → copy → image → brand stamp | P07 | Rewrite as a batch job |
| AI Studio | `/studio` · `AIStudio.tsx` (1,457 lines), `src/lib/studio.ts`, `studioStyles.ts` (15 styles + art direction text); n8n "Studio Brief", "Studio Generate", "Studio Regenerate Slide" | image model registry, reference-image path, carousel (3–8 slides) | P07 | Port the style text, model registry, prompt rules |
| Carousel Studio | `/carousel-studio` · `CarouselStudio.tsx`, `src/lib/carousels.ts`; n8n "Carousel Outline", "Render Trigger"; worker `carousel-studio/gen.js` (HTML templates), `base.css`, `render.js`, `server.js` | HTML→frames→video/PNG | P07 | **Port the worker nearly as is**; make templates take brand kit |
| Video Studio | `/video-studio` · `VideoStudio.tsx`, `src/lib/videoStudio.ts` (pricing, ETA, errors), `videoTypes.ts` (8 types); n8n "Video Brief/Render/Regenerate Shot"; worker `render.js`, `veo.js`, `audio.js`, `server.js` | Veo 3.1, TTS, Lyria, ffmpeg assembly | P07 | Port worker + prompts + pricing; rewrite UI and orchestration |
| Creative Review | `/review` · `CreativeReview.tsx`, `src/components/postPreview.tsx`, `reviewHandoff.tsx`; n8n "Content Revision" | maker-checker, send-to-reviewer, attachments | P08 | Port rules (see `docs/board-maker-checker-test-plan.md`) |
| Board | `/board` · `Board.tsx`, `src/lib/board.ts`; `docs/board-list-view-plan.md` | Jira-style tickets | P13 (later) | Optional |
| Calendar | `/calendar` · `Calendar.tsx` | month grid, create post on a day | P05 | Rewrite |
| Publishing + composer | `/publishing` · `Publishing.tsx`, `CreatePostModal.tsx` (multi-platform, schedule picker), `src/lib/publishing.ts`; n8n "Publishing Engine" (205 nodes), "Publishing Scheduler", "Failure Handler", "Edit Published Post" | per-platform branches, state handling | P05 | Rewrite behind the provider adapter; port UX of the composer |
| Blog | `/blog` · `Blog.tsx`, `BlogPostEditor.tsx`, `docs/blog-module.md` | publishes to scalepods.co | — | **Drop** (internal) |
| Analytics + AI insights | `/analytics` · `Analytics.tsx`, `src/lib/analytics.ts`; n8n "Analytics Collector" (12 h), "AI Insights" | metrics per platform, learning loop | P09 | Port insight prompt; rewrite sync |
| Comment-to-DM + follow gate | Settings panel, `Analytics` leads table; n8n "Comment-to-DM", "Track External Post", "Sync Instagram Comments"; `docs/` none; see memory `comment-to-dm-instagram` | keyword match, claim-before-send dedupe, private reply, follow gate | P09 | Port the logic; verify provider support first |
| Notifications | bell; `src/lib/notifications.ts`; n8n "Team Notifications" | in-app + email | P08 | Rewrite as queue job |
| Spend caps | `src/lib/spend.ts`, `Gate.tsx` | per-member monthly cap | P10 | Rewrite on the credit ledger |
| Settings | `/settings` · `Settings.tsx` | account, theme, Instagram tracking | P02, P05 | Rewrite |
| User manual | `/manual` · `UserManual.tsx` | per-screen how-to | P12 | Port content, update |
| Dashboard | `/` · `Dashboard.tsx` | KPI tiles, week strip, activity, platform status | P02 | Rewrite |

## 2. Reusable assets and logic

| Asset | Where | Use in SaaS |
|---|---|---|
| Design tokens (dark `#04070D`, card `#080A0E`, panel `#10131C`, sage `#B1D997`, electric `#63A5E7`, terracotta `#CC6B49`; light `#F8FAF7` etc.) | `tailwind.config.ts`, `src/index.css` | Default theme of the product's own UI. **Customer brand colours are separate**, per brand |
| Fonts Inter + Instrument Serif Italic | `brand-kit/fonts`, `src/index.css` | Product UI |
| 15 AI Studio looks with art-direction wording | `src/lib/studioStyles.ts` | Seed data for `style_presets` |
| 8 video types with arc/direction/avoid lists | `src/lib/videoTypes.ts` | Seed data |
| Veo prompt rules (five-part formula, no in-frame text, no JSON) | `src/lib/videoTypes.ts`, n8n "Video Brief", `docs/video-studio-trd-phase2.md` | Prompt builder in `packages/shared` |
| Pricing and ETA constants | `src/lib/videoStudio.ts` | Seed the cost table in `packages/ai` |
| Slide templates (cover/step/stat/cta) | `carousel-studio/gen.js`, `base.css`, `assets/`, `icons/` | Per-brand template renderer |
| Render pipeline (persistent Chromium, tini, bounded ffmpeg) | `carousel-studio/render.js`, `Dockerfile` | `apps/worker` |
| Voice list (30 Gemini voices) | `src/lib/videoStudio.ts` | Seed |
| Platform specs (aspect ratios, caption limits) | `src/components/mediaUi.tsx`, `postPreview.tsx` | `packages/shared` |
| Maker-checker rules and tests | `docs/board-maker-checker-test-plan.md`, `team-collaboration-prd.md` | P08 spec |

## 3. Hard-coded ScalePods values to replace with per-brand data

Approved Pod stats (HR/Sales/Ops/Marketing) and the four content pillars → `brand_facts` and brand pillars. Banned words and approved CTAs → `brand_voice`. Handles, logo, tagline stamped on images → brand kit. LinkedIn accounts (Adnan, Raunak, Company Page) and the Facebook Page id → `social_accounts`. Notification emails to `info@scalepods.co` → member emails. The `ScalePods` strings in prompts → `{{brand.name}}`.

## 4. Existing database (about 40 tables) — mapping

| Current | New |
|---|---|
| `business_profiles` | `brands` + `brand_profiles` + `brand_kits` + `brand_voice` + `competitors` |
| `business_intelligence_reports` | `intelligence_reports` |
| `trend_runs`, `trend_signals`, trend-alert tables | same names, plus `org_id` |
| `marketing_strategies`, `strategy_generations` | `strategies` |
| `content_runs`, `content_items`, `content_item_attachments` | `content_items` (+ `content_attachments`); runs become job rows |
| `scheduled_posts`, `post_analytics`, `analytics_state`, `ai_insights`, `post_edits` | same, with the state machine in 03 |
| `instagram_leads`, `comment_dm_log`, `follow_gate_verifications` | `leads`, `automation_events`, follow state on `leads` |
| `studio_jobs`, `carousel_jobs`, `video_jobs` | same, plus `org_id`, cost in credits |
| `app_users`, permission rows, `spend_events` | `profiles`, `memberships`, `feature_access`, `credit_ledger` |
| `notifications`, board tables | `notifications`; board later |
| `canva_connections`, Instagram connection rows | `social_accounts` / `integrations` |

## 5. Existing n8n workflows (about 33) → new queues

| n8n workflow | New queue / code |
|---|---|
| AI Analysis | `ai.text` job `intelligence` |
| Competitor Search | `ai.text` + search job |
| Trend Intelligence, Daily Scheduler | `trends.scan` (scheduled per brand) |
| Trend Alerts Run | `alerts.run` |
| Marketing Strategy, Strategy Generation, Section Regenerate | `ai.text` jobs `strategy.*` |
| Content Text/Image Engine, Branding Overlay, Content Revision | `ai.text`, `ai.image`, `render` |
| Studio Brief/Generate/Regenerate Slide | `ai.text`, `ai.image` |
| Carousel Outline/Render Trigger/Generator | `ai.text`, `render.carousel` |
| Video Brief/Render/Regenerate Shot | `ai.text`, `ai.video`, `render.video` |
| Publishing Engine, Scheduler, Failure Handler, Edit Published Post | `publish.post`, `publish.scheduler` |
| Analytics Collector, AI Insights | `analytics.sync`, `ai.text` |
| Comment-to-DM, Track External Post, Sync Instagram Comments, Instagram Token Keeper | provider webhooks → `automation.*`; token checks only if direct apps |
| Team Notifications, Notifications, Error Handler | `notify.send`, Sentry |
| Blog Publish | dropped |
| Quick Post (inactive) | dropped |

## 6. Edge functions (11 in repo)

`brand-overlay`, `canva-*`, `figma-import` → worker/API routes (P07). `instagram-oauth-*`, `instagram-disconnect` → replaced by provider connect (P05). `r2-upload` → API presign route (P04). `r2-usage-check` → billing/usage monitor (P12).
