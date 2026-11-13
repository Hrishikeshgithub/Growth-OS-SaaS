# Product scope — version 1

## What we are replicating

The internal Growth OS is a complete marketing loop for one business: **research → plan → create → review → publish → measure → learn**. The SaaS is the same loop for many customers. The structural spec of the current screens is in `docs/reference/growth-os-structural-spec.md` (outside this folder).

## Modules and where they land

| Module | In v1? | New phase | Notes for the SaaS |
|---|---|---|---|
| Sign in, accounts, roles | Yes | P01 | Email + Google, 2FA for owners/admins, org memberships |
| Organizations and brands | Yes | P01, P03 | Today's "business profile" becomes a **brand** |
| Brand kit and voice (logo, colours, fonts, tone, banned words, approved facts, CTAs) | Yes | P03 | Replaces everything hard-coded for ScalePods |
| Onboarding wizard (website → AI-drafted brand → connect channels → first post) | Yes | P03, P05 | Target: first scheduled post in under 15 minutes |
| Intelligence (AI business report) | Yes | P06 | |
| Competitors (add, edit, AI discovery) | Yes | P06 | |
| Trends (multi-source scan, signals) | Yes | P06 | Sources limited by plan |
| Trend Alerts (watch items, AI-filtered alerts) | Yes, low limits | P06 | |
| Strategy (7-part plan, scoped strategies, calendar, approve) | Yes | P06 | |
| Content engine (text, image, carousel from the calendar) | Yes | P07 | Brand voice injected per brand |
| AI Studio (styles, models, reference image, carousels) | Yes | P07 | |
| Carousel Studio (branded HTML-rendered slides) | Yes | P07 | Needs per-brand templates |
| Video Studio (motion graphics, Veo clips, voice, music) | Yes (AI clips: Pro+) | P07 | Cost-gated per job |
| Creative Review (maker-checker, send-to-reviewer, attachments) | Yes (Pro+) | P08 | Starter has simple approve |
| Board (tickets) | Later (P13) | | Not core for strangers; keep internal tool's version |
| Notifications (in-app, email) | Yes | P08 | Queue-driven, per-member email prefs |
| Calendar and Publishing | Yes | P05 | Via the social provider adapter |
| Edit live post | Where the provider allows it | P05 | LinkedIn and YouTube only today |
| Blog (publish to scalepods.co) | **No** | — | Internal use only |
| Analytics and AI insights | Yes | P09 | Fed back into scheduling and copy |
| Instagram comment-to-DM, follow gate, lead list | Yes | P09 | Provider support must be confirmed in the P05 test |
| Billing, plans, credits, usage | Yes | P10 | |
| Team and permissions (per-feature levels) | Yes | P01, P08 | Roles per org |
| Spend caps per member | Yes | P10 | Built on the credit ledger |
| User manual | Yes | P12 | In-app help plus docs site |
| Support AI link | Optional | P12 | |
| Admin console for us (customers, usage, suspend) | Yes (small) | P11 | Needed to run the business |

## Plans (proposal from the approved solution doc)

|  | Starter | Pro | Agency | Enterprise |
|---|---|---|---|---|
| Price / month (USD) | 49 | 149 | 399 | Custom (from 1,500) |
| Brands | 1 | 3 | 15 | Unlimited |
| Connected channels | 5 | 15 | 75 | Unlimited |
| Seats | 2 | 5 | 15 | Unlimited |
| AI credits / month | 1,000 | 4,000 | 15,000 | Pooled |
| Video Studio | Motion graphics only | Yes | Yes | Yes |
| Approval workflow | Simple approve | Maker-checker | Maker-checker | Custom stages |
| Comment-to-DM automations | 1 | 10 | Unlimited | Unlimited |
| Trend Alerts watch items | 2 | 10 | 40 | Custom |
| Analytics history | 90 days | 1 year | 2 years | Unlimited |
| Client approval links, white-label | No | No | Yes (P13) | Yes |
| SSO, audit export, own AI keys, data region | No | No | No | Yes |

Trial: 14 days on Pro, 200 credits, no card. Limits are stored as **data** (a `plan_limits` table, see 03-data-model) so changing a number never needs a deploy.

## Out of scope for v1 (decided)

White-label and client portals, SSO/SAML, SOC 2, data-region choice, read replicas, public API for customers, mobile app, languages other than English, WordPress/Webflow publishing, TikTok and Google Business Profile (not supported today either).

## Definition of "version 1 is done"

1. A stranger signs up, creates a brand, connects at least Instagram, LinkedIn and Facebook, and schedules a post without our help, in under 15 minutes.
2. Posts publish on time and failures are visible and retryable.
3. AI generation works with credits, and running out of credits pauses only AI.
4. Two customers cannot see or change each other's data (proven by the automated suite).
5. A paying customer can upgrade, downgrade, buy a credit pack and cancel from the app.
6. Backups, monitoring, status page, legal pages and data export/deletion exist.
