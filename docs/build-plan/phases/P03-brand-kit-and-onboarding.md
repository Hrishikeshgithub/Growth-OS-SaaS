# P03 — Brand kit and onboarding

**Goal:** a brand holds everything the AI and the renderers need (profile, voice, banned words, approved facts, CTAs, logo, colours, fonts), and a new customer is guided from signup to "ready to create" in minutes. **Est:** 5–6 working days. **Depends on:** P01, P02. (Draft-from-website needs P04's AI gateway; build the form first, plug AI in at the end of P04.)

## What replaces the hard-coded ScalePods values
See `04-existing-system-inventory.md` §3. In the SaaS, **nothing brand-specific is in code or prompts**. Prompts take a `brand_context` object assembled at run time by `packages/shared/brandContext.ts`.

## Steps

### 1. Migrations
`brand_profiles`, `brand_kits`, `brand_voice`, `brand_facts`, `competitors`, `brand_pillars` (name, description, target share %), each with `org_id`, `brand_id`, RLS pattern, indexes, and entries in the isolation catalogue. Plan-limit trigger on `brands`.

### 2. Brand profile screens
- **Profile**: name, tagline, industry, description, products/services, audience, goals, website, social URLs, contact details, service areas (port field list from `src/lib/clients.ts` `BusinessProfile`).
- **Brand kit**: upload logo (light and dark variants), cover image, up to 6 colours (primary, secondary, accent, background, text, plus custom), heading and body font choice (curated list of Google fonts; custom upload later), image style notes.
- **Voice**: tone words (chips), do/don't examples, **banned words** list, **approved CTAs** list, hashtags to prefer.
- **Facts**: statements the AI may cite (e.g. "5 hours saved per recruiter per week") each with an optional source; a toggle **"Only cite approved facts"** (default on). Everything outside the list is treated as not allowed to appear as a statistic.
- **Pillars**: 3–6 content pillars with target mix (replaces the fixed HR/Sales/Ops/Marketing set).
- **Competitors**: add/edit with name, website, social handles, notes (AI discovery comes in P06).

### 3. Uploads
Use the presign route from P04 once it exists; until then, a stub storing to a dev bucket. Validate type (png/jpg/webp/svg-sanitized), size ≤ 5 MB for logos, resize to sane bounds. **SVG logos are sanitized** (no scripts) or converted to PNG.

### 4. `brandContext` builder
A pure function returning: `name`, `tagline`, `industry`, `audience`, `voice` (tone, banned words, CTAs), `facts` (approved list), `pillars`, `colors`, `fonts`, `logoUrl`, `handles`. Unit-tested. It is the **only** way prompts get brand data. Include a `renderPromptHeader(brandContext)` helper that produces the standard system-prompt block: voice, banned words, the literal list of approved facts and the rule "if no approved fact fits, use no number". (The current system learned that the *literal list* must be embedded, not just referred to.)

### 5. Onboarding wizard (`/onboarding`)
Steps, each saved as it goes (resume later): **1** Workspace name and your role → **2** Brand basics (enter website; if P04 is ready, "Draft from my website" fills profile, voice and 5 candidate facts for the customer to approve; otherwise manual) → **3** Brand kit (logo, colours, fonts; "pull from website" later) → **4** Connect channels (links to P05; skippable, with a clear "you can't publish until you connect") → **5** Invite teammates (skippable) → **6** Guided first run (links to P06/P07; shows a checklist). Progress shown as a checklist on the dashboard until complete (`onboarding_state` jsonb on the org).

### 6. Brand preview
A "How your posts will look" card renders a sample post using the brand kit (logo, colours, font) — the same renderer P07 will use.

### 7. Brand-scoped navigation
Everything under `/b/[brandSlug]/…`. Switching brands never mixes data (cache keys include `brand_id`).

## Deliverables
Brand tables and policies; profile/kit/voice/facts/pillars/competitor screens; `brandContext`; onboarding wizard with resume; preview card.

## Acceptance
- [ ] A new account completes onboarding steps 1–3 and 5 without help in under 10 minutes
- [ ] `brandContext` output contains the literal approved facts and banned words (unit test snapshots)
- [ ] A second brand's data never appears under the first (UI and isolation test)
- [ ] Logo uploads reject scripts, wrong types and oversize files
- [ ] Plan limit: Starter cannot create a second brand (database refusal)
- [ ] Onboarding can be left and resumed at the right step

## Traps
- Don't let customers paste unlimited text into facts or voice; cap lengths so prompts stay bounded.
- Treat everything the customer enters (and later, scraped website text) as **data**, never as instructions (prompt-injection rule).
- Fonts: loading arbitrary font files is a security and performance risk; start with a curated list.

## If time is short
Ship profile, kit, voice, facts; postpone pillars (default to a single "General" pillar) and the preview card.

## Kickoff prompt for Claude Code
> Read `CLAUDE.md`, `phases/P03-brand-kit-and-onboarding.md`, and the `BusinessProfile` type in the old repo's `src/lib/clients.ts` for field ideas. Create the brand tables with RLS and add them to the isolation catalogue. Build the brand profile, kit, voice, facts, pillars and competitor screens and the `brandContext` builder with snapshot tests. Then build the resumable onboarding wizard. Leave the "draft from website" button disabled with a TODO until P04.
