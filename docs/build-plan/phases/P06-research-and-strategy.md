# P06 — Research and strategy

**Goal:** the "think" half of the loop: AI business report, competitor discovery, trend scans, Trend Alerts, and AI marketing strategies that produce a content calendar. **Est:** 8–10 working days. **Depends on:** P03 (brand context), P04 (jobs, gateway). Can run in parallel with P07 if two people build.

## Steps

### 1. Migrations
`intelligence_reports`, `trend_runs`, `trend_signals`, `trend_watches`, `trend_events`, `trend_rules`, `trend_watch_runs`, `strategies`, `strategy_signals`, `competitor_candidates`. All with `org_id`, `brand_id`, RLS, indexes, isolation catalogue entries. Plan-limit trigger on `trend_watches`.

### 2. Intelligence report
Port the **7 sub-analyses** prompt from the old n8n workflow "AI Analysis" (read its Code and HTTP nodes) into `packages/shared/prompts/intelligence.ts`, taking `brandContext`. Job kind `intelligence.generate` (`ai.text`). Output stored as sections JSON; UI renders each section with markdown (port the safe Markdown renderer `src/components/Markdown.tsx`: it never injects HTML). History list and "Regenerate".

### 3. Competitors
Manual add/edit (P03) plus **AI discovery**: port the two-pass algorithm from "Competitor Search" (`jkyRS1nbiGOJ9BQn`): search the web for the brand's landscape → LLM proposes only names it can point to an exact URL for → second search verifies each candidate → store candidates for the customer to accept or reject. Web search through **Serper** (or similar) behind a `packages/shared/search.ts` interface; Apify only where scraping is needed. Costs go through the gateway as credits.

### 4. Trends
Port source handling from "Trend Intelligence" and `trend-scraper/app.py`: **Reddit, Instagram, YouTube, Google Search, Google Trends** (current `SCAN_PLATFORMS`), with the customer choosing sources and result counts. Job `trends.scan` collects → LLM ranks relevance against `brandContext` (relevance score + reason) → stores `trend_signals`. Daily scheduled scan per brand (opt-in, plan-limited) anchored to a real local time, not a rolling timer (a past bug). UI: run history, signals grouped by source with source-colour dots, date range filter, multi-select of signals to feed a strategy. Scraped text is **data**, never instructions.

### 5. Trend Alerts
Read `docs/radar-prd.md` fully before building. Watch item = keywords + excludes + sources (Google News, Reddit, web, YouTube, RSS) + region + lookback + relevance threshold + budget + frequency. Pipeline per run: collect → embed → filter and cluster duplicates → classify (cheap model) → spike detection (today vs 7-day mean/stddev) → rules → explanation (stronger model) → delivery (in-app, Slack webhook, generic webhook; daily/weekly digests). Port the SQL/cron design but run it as a worker job (`alerts.run`) scheduled by the queue. Per-watch monthly budget enforced through credits; stop and notify when exhausted. Plan limit on number of watches.

### 6. Strategy
Port the prompts and JSON shapes from "Marketing Strategy", "Strategy Generation" and "Strategy Section Regenerate": 7 components (campaign planning, weekly content strategy, content pillars, platform strategy, lead-generation strategy, CTA strategy, content calendar) plus `header_insights` and `pillar_balance` computed from real analytics once P09 exists. Scoped strategies: pick signals, scope (day/week/month), platform, content type → a standalone strategy. **Only one approved strategy is "active"** per brand. UI: list ↔ detail with section tabs, per-section regenerate, calendar view, approve gate. Port components from `src/components/strategy/*`. Pillars come from the brand (P03), not fixed HR/Sales/Ops/Marketing.

### 7. From calendar to content
"Create from calendar item" sends the item to the creation modules in P07 with brand context and source ids attached (traceability from signal → strategy → content).

### 8. Cost transparency
Every action shows an estimated credit cost before running and the actual after. Heavy actions (full strategy, big scans) ask for confirmation using the shared `ConfirmDialog` cost line.

## Deliverables
Five screens (Intelligence, Competitors, Trends, Trend Alerts, Strategy) with jobs, history, credits, plan limits.

## Acceptance
- [ ] Brand with no data gets a useful empty state and a "complete your brand first" nudge
- [ ] Each job shows progress, a cost, and fails with a reason (test by breaking the provider key)
- [ ] Facts in outputs come only from approved facts when the toggle is on (test with a brand that has none: no invented statistics)
- [ ] Trend scan respects the plan's source list and credit balance
- [ ] A Trend Alert watch stops at its monthly budget and notifies
- [ ] Only one strategy per brand can be approved; approving another demotes the first
- [ ] Isolation suite covers every new table

## Traps
- JSON replies from models sometimes arrive in code fences; strip before parsing and retry once on failure.
- Don't let a watch run unattended without a budget cap.
- Scraping sources break; show per-source status ("Reddit failed, 4 of 5 sources used") instead of failing the whole run.
- The old strategy page had a race where a placeholder row did not exist yet; create the row in the **same transaction** as the job.

## If time is short
v1 minimum: Intelligence, Trends, Strategy. Postpone Trend Alerts to after launch (it is the most expensive and complex).

## Kickoff prompt for Claude Code
> Read `CLAUDE.md`, `phases/P06-research-and-strategy.md`, `docs/radar-prd.md` and `04-existing-system-inventory.md`. Implement, one at a time with tests: migrations; the intelligence job and screen (port the prompts from the old workflow into `packages/shared/prompts` using `brandContext`); competitor discovery; trend scans; strategy generation and calendar. Use the gateway for every model call. Leave Trend Alerts for last and ask me before starting it.
