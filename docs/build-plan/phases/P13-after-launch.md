# P13 — After launch

Triggered by real usage, not dates. Each item has a **trigger** and an **owner decision**.

## 1. First 30 days
| Item | Detail |
|---|---|
| External penetration test | Book before 50 paying customers; fix all high/critical; retest |
| Customer feedback loop | Weekly review of support tickets and feedback form; top 5 fixes shipped weekly |
| Own platform apps (Option A) | Start **now** (reviews take weeks): fresh Meta app owned by the business (separate from internal `ScalePods_Lead_Sync`) with use-case texts and 1080p screen recordings per permission; LinkedIn Community Management API (Development → Standard tier); Google/YouTube sensitive-scope verification and quota increase; X native API (pay per post) |
| Cost review | Real AI cost per credit vs price; adjust the cost table and credit packs from data |

## 2. Months 2–3
| Item | Trigger |
|---|---|
| **Agency tier**: many brands, client approval links (public tokenized review page), white-label (custom logo/domain on approval page), client roles | First Agency customer or 10 requests |
| Queue workers scale-out (more worker instances, separate AI and render workers) | AI jobs wait > 60 s regularly |
| Second render container | Renders wait > 2 min |
| Board (tickets) module | Requested by teams > 10 |
| WordPress / Webflow / Shopify blog publishing (replaces the internal-only Blog module) | Demand from content teams |

## 3. Months 3–6 (toward 500–2,000 users)
| Item | Trigger |
|---|---|
| Supabase compute Medium or Team plan; connection pooling review | DB CPU > 60% at peak, 150+ concurrent users, 400+ live connections |
| Read replica for analytics | Analytics queries slow the app |
| Scheduler at one-minute precision with sharding | > 2,000 posts per hour |
| Move customers to **our own** approved apps and drop per-account provider fees; keep the provider as fallback | Platform approvals granted; provider bill > cost of own apps |
| Autoscaling renders (Fly.io or ECS Fargate) | Renders > 20 concurrent |
| Stripe live; more currencies; GST/tax automation review | International customers |
| Hindi and other languages | Demand |

## 4. Later
SOC 2 Type II (about a year of evidence), ISO 27001, SSO/SAML for Enterprise, data-region choice (separate Supabase project per region or per customer), public API and webhooks for customers, mobile app, dedicated private deployments (AWS) only for enterprise deals that require it.

## Review cadence
Monthly: review this list against metrics (signups, activation within 15 minutes, publish success rate, AI margin, churn, support load), reprioritize, record decisions as ADRs.
