# P02 — App shell

**Goal:** the signed-in frame every module lives in: navigation, org and brand switchers, theming, permissions-aware UI, notifications bell placeholder, empty states. **Est:** 3–4 working days. **Depends on:** P01.

## Reference
Layout, proportions and nav grouping come from the current app (`docs/reference/growth-os-structural-spec.md` §3). Keep what worked, fix the gaps it lists.

## Steps

### 1. Layout
1. `app/(app)/layout.tsx`: sidebar + top bar + main region. Sidebar **256 px pinned / 72 px icon rail**, collapses to a **mobile drawer with backdrop** below `md` (the current app lacks this). Top bar ≈ 69 px. Content max width 1680 px, padding 32 px (16 px on mobile). Pin state in `localStorage`.
2. Sidebar top: logo, **organization switcher** (only if the user is in more than one), **brand switcher** (list brands, "New brand" if the plan allows). The active brand id lives in a cookie and the URL for deep links (`/b/[brandSlug]/…`).
3. Navigation groups (flat, grouped by pipeline stage): Overview (Dashboard) · Strategy (Brand, Trends, Trend Alerts, Strategy) · Create (AI Studio, Carousel Studio, Video Studio, Review) · Publish (Calendar, Publishing) · Insight (Analytics, Intelligence, Settings, Team & access, Help). Items appear only if the member has `view` on that feature.
4. Top bar: notification bell (empty list for now), theme toggle, user menu (profile, security, sign out). Show **credits remaining** as a small pill (reads the balance later in P10; stub it now).

### 2. Theming
Dark default + light, via CSS variables on `data-theme`. Persist choice in a cookie so server rendering matches (no flash). Define the brand-accent token slots so a brand's colours can theme *previews* without touching the product chrome.

### 3. Permissions in the UI
`useCan(feature, level)` reads the member's levels (loaded once, cached). Disabled controls show a hover reason ("You need Edit access to Studio. An admin can grant it in Team & access"). Hiding vs disabling: hide navigation, disable actions. This is a courtesy; the server and database decide.

### 4. Shared building blocks
`PageHeader`, `MetricTile`, `EmptyState`, `Modal` (sizes md/lg/xl; 16:9 variant), `ConfirmDialog` with a **cost line** slot for spend confirmations, `StatusBadge` (green published/blue scheduled/orange failed), `PlatformBadge`, `PostTile`, `Toast`, `DataTable`. Reuse the pattern list from the spec §5 and port visuals from `src/components/ui.tsx`, `postPreview.tsx`, `mediaUi.tsx`.

### 5. Dashboard (v0)
Four tiles (Approved & ready, Published this month, Engagement, Pending review), "This week" strip, Recent activity, Connected channels, and pipeline shortcut cards. Wire with empty states now; data arrives in later phases.

### 6. Errors, loading, not-found
Route-level `loading.tsx`, `error.tsx` (reports to Sentry with request id), custom 404, and a global "service degraded" banner driven by a feature flag.

### 7. Accessibility and responsiveness
Keyboard-navigable sidebar and modals, focus trap, visible focus ring, colour contrast AA on both themes, no horizontal scroll at 360 px.

## Deliverables
Shell, switchers, theming, permission hooks, shared components, dashboard v0, responsive behaviour.

## Acceptance
- [ ] Works at 360 px, 768 px and 1440 px; mobile drawer opens and closes with Esc and backdrop
- [ ] Theme persists across reload with no flash
- [ ] A `viewer` sees no actions that write; a user without `view` on a feature does not see its nav item and gets a clear "No access" page on a typed URL
- [ ] Switching brand changes data on every page without a full reload
- [ ] Lighthouse accessibility score ≥ 95 on the dashboard
- [ ] Sentry captures a thrown error from a client component with the request id

## Traps
- Do not hide the user manual/help from users with narrow access; they need it most.
- Don't render user-provided brand names as HTML.
- Keep the shell server-rendered where possible; avoid putting the whole tree behind a client provider.

## If time is short
Skip the light theme polish and the dashboard "This week" strip; keep switchers, nav, permissions.

## Kickoff prompt for Claude Code
> Read `CLAUDE.md`, `phases/P02-app-shell.md` and `docs/reference/growth-os-structural-spec.md` §3 and §5. Build the signed-in shell in `apps/web/app/(app)` with the sidebar (pinned/rail/mobile drawer), org and brand switchers, grouped navigation filtered by feature access, top bar, theming without flash, and the shared components in `packages/ui`. Add the dashboard v0 with empty states. Show me screenshots at 360, 768 and 1440 px when done.
