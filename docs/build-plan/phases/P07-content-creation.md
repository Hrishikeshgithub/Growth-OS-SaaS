# P07 — Content creation

**Goal:** turn ideas and calendar items into publishable assets: copy, images, carousels and videos — all in the customer's brand, all metered. **Est:** 12–15 working days (largest build phase). **Depends on:** P03, P04, P05 (to send to review/publish). Sub-phases can ship separately: **7a** text + AI Studio, **7b** Carousel Studio, **7c** Video Studio.

## 7a — Content engine and AI Studio

1. **Migrations:** `content_items`, `content_item_media`, `studio_jobs`, `style_presets` (seeded from `src/lib/studioStyles.ts`: 15 looks with `direction` text, default aspect ratios, whether the look renders text), per-org overrides allowed.
2. **Brief step (free):** user picks a source (trend, strategy item, or free topic), platform, aspect ratio, style, model, variants, optional reference image, character. A `ai.text` job writes copy (hook, body, hashtags, CTA) and a **subject** description for the image. Port the rules from the "Studio Brief" workflow: the model writes a physical, photographable scene (no text, no logos, no colours or medium words); the **style's art direction is appended by code**, so it cannot be paraphrased away; for text-led looks the poster line (≤ 6 words) is typeset instead.
3. **Generate step (paid):** `ai.image` job; models from the registry (OpenAI image models, Gemini image models); reference-image path (OpenAI edit vs Gemini inline image part: they behave differently, see `ai-studio` notes); carousels of 3–8 slides, one prompt per slide with per-slide regenerate; cost shown first and confirmed.
4. **Brand stamping:** apply logo, tagline and handles server-side in the worker (port `brandStamp.ts` logic and the `brand-overlay` function: do compositing in the worker with `sharp` or Chromium, **not** WASM in an edge function — it was unreliable).
5. **Content engine (batch):** "Generate this week from the approved calendar": for each calendar item create copy, then image, then stamp, producing `content_items` in `ready` state. Only image/carousel/caption types are batchable; **video is never auto-chained** (credit safety).
6. **UI:** style gallery (thumbnails shipped in the app, not fetched from storage), editable image prompt, variants grid, send to review, recent jobs grid. Port from `AIStudio.tsx`.

## 7b — Carousel Studio (branded slide renderer)

1. Port the worker engine from `carousel-studio/` (`gen.js` templates cover/step/stat/cta, `base.css`, `render.js`, `server.js` routes) into `apps/worker/src/render/`. Keep: persistent Chromium, `tini`, deterministic frame capture (absolute progress per frame), per-slide progress, encode with ffmpeg.
2. **Make templates brand-aware:** inputs `brandContext` (colours, fonts, logo, handles); replace baked-in ScalePods assets (`assets/brand`, poses, icons) with per-brand uploads and a neutral default set. Fonts loaded from the curated list only.
3. Outline step: `ai.text` job drafts slides (cover, steps, one stat using only approved facts, CTA) → editable → render job `render.carousel` → PNG/MP4 → R2 → `content_items`.
4. Export options: PNG set (Instagram/LinkedIn carousel), PDF (LinkedIn document post), MP4.

## 7c — Video Studio

1. **Types and prompts:** seed 8 video types from `src/lib/videoTypes.ts` (arc, direction, avoid list, defaults); port the Veo **five-part prompt formula** (cinematography, subject, action, context, style) as one natural-language paragraph per shot (no JSON); hard rules: no in-frame text/logos/numbers, no dialogue, each shot self-contained, no continuity of faces across shots.
2. **Storyboard (free):** `ai.text` job returns post copy + N shots (1–6, 4/6/8 s each) with on-screen text per shot; validator rejects prompts that ask for words in frame, too-short prompts and template placeholders (port the checks from "Video Brief").
3. **Generate (paid, cost-gated):** per-video **ceiling $5** (constant), real-cost confirmation dialog with USD and INR, estimated time shown, shot-by-shot or all-at-once, finished shots reused free. Worker flow per shot: `ai.video` (Veo via the gateway; duration is a **number**; poll with timeout 10 min) → download → store clip → update shot row **as each shot finishes** (UI shows a per-shot progress bar and overall ETA — port the helpers in `videoStudio.ts`).
4. **Audio:** optional voiceover (Gemini TTS, 30 voices with previews generated once and shipped as static assets) and music bed (Lyria, 30 s). Audio is generated **before** shots (cheap fails fast). Veo's native audio cannot be disabled; mix three layers with ducking.
5. **Assembly (`render.video`):** normalize clips (30 fps), concat, burn on-screen text with ffmpeg `drawtext` from the reviewed strings, crop/pad to ratio, logo overlay, mix audio with `apad=whole_dur=…,atrim=duration=…` so audio is **bounded to the video length** (unbounded padding hung a render for hours).
6. **Motion-graphics path** (free of AI cost): reuse the carousel templates → frames → stitched MP4, with optional voiceover.
7. **State rules:** the server row is the source of truth; the UI merges live updates and never writes back a stale copy (lesson #10). If the worker is unreachable the job fails visibly with "nothing was charged" (lesson #3).
8. **Plan gating:** Starter = motion graphics only; Pro+ = AI clips.

## Shared for all sub-phases
- Credits: reserve at job acceptance, settle at actual, release on failure. Show estimate before, actual after.
- Moderation on text and images before they become `ready`.
- All outputs land in R2 under the org/brand path; thumbnails generated.
- "Send to Review" creates/updates the `content_item` and (Starter) allows direct approve, (Pro+) goes through maker-checker (P08).
- Per-feature access levels: `view` sees jobs, `edit` changes prompts/copy, `full` generates (spends).

## Deliverables
7a, 7b and 7c each deployable on their own, with seeds, prompts, worker handlers, screens, and tests.

## Acceptance (per sub-phase)
- [ ] Brief → generate → stamped asset works end to end for a brand with its own logo and colours (7a)
- [ ] Output never contains another brand's name or facts (use two test brands)
- [ ] Carousel renders in the brand's colours and fonts and exports PNG, PDF and MP4 (7b)
- [ ] Render worker survives 20 consecutive renders without exhausting processes (soak test)
- [ ] A 3-shot video completes with a real provider on a tiny budget (use the cheapest engine, 4 s shots) with per-shot progress and correct credit settlement (7c)
- [ ] Killing the worker mid-shot fails the job cleanly and releases credits
- [ ] Cost shown before generation matches the ledger afterwards (± rounding)
- [ ] Per-job ceiling blocks an over-budget video even if the UI is bypassed (server check)

## Traps
- Model outputs vary; always validate JSON and retry once.
- Never ship test spend to a production key; use the staging key with a low cap.
- Keep style thumbnails in the repo (not storage) so a storage cleanup can't remove them.
- Veo cannot spell; do not ask it to render text.
- Docker image: copy directories, not file lists.

## If time is short
Ship 7a (text + AI Studio single image) and 7b basic templates for launch; make 7c motion graphics first and AI clips just after launch.

## Kickoff prompt for Claude Code
> Read `CLAUDE.md`, `phases/P07-content-creation.md`, `docs/video-studio-trd-phase2.md` and the old repo's `src/lib/studio.ts`, `studioStyles.ts`, `videoTypes.ts`, `videoStudio.ts`, plus `carousel-studio/*.js`. Do sub-phase 7a first: migrations, the brief and generate jobs through the AI gateway, brand stamping in the worker, and the AI Studio screens. Show me a generated image for a test brand before starting 7b.
