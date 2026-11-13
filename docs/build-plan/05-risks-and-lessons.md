# Risks and lessons carried over

Every item is something that went wrong, or nearly did, in the current system. The phase column says where it must be handled.

## A. Mistakes not to repeat

| # | What happened | Rule for the SaaS | Phase |
|---|---|---|---|
| 1 | n8n webhooks accept any caller; anyone knowing a URL could start paid jobs | Every inbound webhook verifies a signature over the raw body, checks a timestamp window and stores a receipt to block replays | P04, P05, P10 |
| 2 | A `for all` row policy hid a missing rule for comments | Policies are written per command; the isolation test covers every command | P01 |
| 3 | `continue on error` on the worker-call node made a dead worker look "success"; videos silently never started | Never swallow errors. A failed call writes `failed` + reason on the job row. UI shows it | P04 |
| 4 | Spend was logged when the button was clicked, so five failed clicks logged $10.80 of fake spend | Reserve credits only when the job is accepted; release on failure; settle at the real cost | P04, P10 |
| 5 | "DM sent" was written before the send finished, so failures showed as sent | Write success flags **after** the provider confirms | P09 |
| 6 | A long-lived Instagram token expired silently (60 days); comment-to-DM stopped for 5 days | Monitor connection health daily; alert owners; refresh where the API allows; show "reconnect" in the UI | P05, P09 |
| 7 | Free-plan Supabase hit its storage and egress quotas because videos were stored there | Customer media only in R2 behind a custom domain with caching; Supabase Pro from day one | P00, P04 |
| 8 | The free `r2.dev` address was slow and rate-limited; Buffer timed out fetching four 14-megapixel images | Custom domain + cache; resize images on upload to ≤ 2160 px; retry once on provider media errors | P04, P05 |
| 9 | A cleanup deleted 40 files that code (not the database) still referenced | Before deleting any file or row, search code, workflows and workers for references | all |
| 10 | Local state in the video screen went stale after the worker updated the job; a later "rebuild" re-paid for finished shots | Treat the server row as the source of truth; merge live updates; never write back a stale copy | P07 |
| 11 | Scheduled posts left "publishing" ghost rows when a step failed | Atomic claim, attempt rows, rescue job, visible "stuck" state (03 §3) | P05 |
| 12 | Hand-made schema changes in the dashboard were lost track of | Migrations only, in the repo, reviewed | P00 |
| 13 | Keys typed into workflow nodes; shared server with other clients' workflows | Secrets only in a secret store; dedicated worker | P00, P04 |
| 14 | A workflow was edited by two people at once; one publish pushed half-finished work | Code in git with reviews and CI; no editing live systems | P00 |

## B. Platform facts that shape the design

| Topic | Fact | Consequence |
|---|---|---|
| Meta app mode | Posts and Page features are visible only to app role users while the app is in development; Live mode and Advanced Access need review | The social provider hides this at launch; our own app needs App Review (P13) |
| Instagram tokens | Long-lived tokens last 60 days; refresh needs the token to be at least 24 hours old; a refresh call takes the token as a **query parameter**, not a header | If we ever hold tokens, a daily keeper job pings, refreshes and alerts |
| LinkedIn tokens | Personal tokens last about 60 days, no refresh; Company Pages need the Community Management API (long review) | Prefer the provider; plan for reconnect prompts |
| LinkedIn PDF carousels | A "document post" (PDF) is different from a multi-image post | Support both types in the composer |
| X | Pay per post (more with links) | Pass through as credits |
| YouTube | Default quota is 100 uploads a day for the whole app | Cap Shorts per plan until quota is raised |
| Comment-to-DM | Private replies must be sent within 7 days of the comment; follow status is only readable for someone already in a conversation | Dedupe with a unique comment id; follow gate sends a first DM, then checks |
| Veo | Duration is a number, not a string; clips are generated independently (no face continuity); the model cannot spell, so on-screen text is burned in by ffmpeg; native audio cannot be turned off; Google recommends plain-language prompts, not JSON | Port rules into the prompt builder and render engine |
| ffmpeg | An unbounded audio pad hung a render for hours | Always bound audio to the video's duration |
| Chromium in containers | One Chrome process per frame exhausted process ids and killed the container | Persistent browser, `tini` as PID 1 |
| Docker image | A new source file not listed in `COPY` crashed production | Copy whole directories, not file lists |
| Supabase | Deleting storage rows in SQL does not free space | (R2 for media; if Storage is used, delete via its API) |
| Realtime | Each open subscription counts against connection limits | Subscribe only on screens that show live status |

## C. Risks for the SaaS itself

| Risk | Impact | Mitigation | Phase |
|---|---|---|---|
| Social provider outage or loss of a platform approval | Publishing stops for many customers | Adapter interface; Ayrshare tested as fallback; status page; own apps in background | P05, P13 |
| Cross-tenant data leak | Loss of trust, legal exposure | RLS + test suite on every PR + external penetration test | P01, P11 |
| AI costs outrun credit prices | Margin falls | Credits from measured costs; cost table is data; per-model switch | P04, P10 |
| Abuse or spam by a customer | Platform bans our apps | Acceptable-use policy, post limits per plan, moderation, quick suspend | P11 |
| Timeline slips because P05 or P07 are large | Late launch | Cut scope inside the phase (see "If time is short" in each), never cut security tests | all |
| Single-provider outage (Supabase, OpenAI) | Parts of app stop | Retry queues, status page, second AI text provider | P12 |
| Customer deletes data and asks for it back | Support load | Soft-delete 30 days, then hard delete with confirmation | P11 |
| Currency and tax errors | Compliance issues | GST via Razorpay invoices; tax service for other regions | P10 |
