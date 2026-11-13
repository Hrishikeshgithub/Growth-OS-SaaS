# P10 — Billing and entitlements

**Goal:** customers can start a trial, pick a plan, pay, upgrade, downgrade, buy extra credits and cancel — with plan limits and credits enforced by the database, and plan state changed **only** by verified payment webhooks. **Est:** 6–8 working days. **Depends on:** P01, P04. Start Razorpay KYC early (inputs #9).

## Steps

### 1. Catalogue
Tables `plans`, `plan_limits`, `credit_packs`, `subscriptions`, `billing_events`, `invoices` (mirror). Seed plans from `01-product-scope.md` in USD **and** INR (decide INR prices: input #13). Limits keys: `brands, channels, seats, credits_month, automations, watches, history_days, video_clips, maker_checker, client_links`. `private.limit_for(org, key)` and the enforcement triggers from `03-data-model.md` §5.

### 2. Razorpay integration (`packages/billing`)
1. Create the plans in Razorpay (monthly and yearly) and store their ids on `plans`.
2. **Checkout:** API route creates a Razorpay Subscription for the org (customer created once, notes include `org_id`), returns the checkout parameters; the browser opens Razorpay Checkout. UPI mandates, cards, GST invoices handled by Razorpay.
3. **Webhook** `/api/webhooks/razorpay`: verify `X-Razorpay-Signature` (HMAC-SHA256 of the raw body with the webhook secret), store in `billing_events` (unique event id), then in a transaction update `subscriptions`, set `organizations.plan_id`, grant monthly credits (idempotent by event id). Events to handle: `subscription.activated`, `subscription.charged`, `subscription.pending`, `subscription.halted`, `subscription.cancelled`, `subscription.completed`, `payment.failed`, `invoice.paid`.
4. **Never** change plan or grant credits from a browser call or the redirect after checkout; the redirect only shows "processing…" until the webhook arrives.
5. Dunning: on `payment.failed`/`halted` show a banner, email owners, keep the account in a **grace period** (e.g. 7 days) with AI paused but data safe; after grace, downgrade to a read-only/free state. Never delete data for non-payment.

### 3. Plan changes
Upgrade: immediate with proration (provider handles); credits for the new plan granted on charge. Downgrade: effective at the period end; if usage exceeds the lower plan's limits (brands, channels, seats), show exactly what must be removed first and block the downgrade until fixed. Cancel: access until period end; then read-only. Trial: 14 days on Pro, 200 credits, no card; at the end, require a plan or fall to read-only.

### 4. Credits
- Monthly grant on renewal (`grant` ledger row; unused monthly credits **do not roll over**: an `expire` row at period end).
- **Credit packs**: one-off Razorpay orders; `purchase` rows valid 12 months (track expiry with a `credit_lots` table or expiry date on the ledger rows).
- Low-balance warnings at 80% and 100% (in-app + email). At zero only **AI generation pauses**; publishing, analytics and review keep working.
- Per-member monthly credit caps (from `memberships.monthly_credit_cap`) enforced in `reserve_credits`.

### 5. Usage and billing screens
Settings → Billing: current plan, renewal date, credits balance and recent usage by feature (from `ai_usage`), invoices (links), change plan, buy credits, update payment method (Razorpay portal link), cancel. Show limits with bars (brands 2/3, channels 9/15, seats 4/5).

### 6. Stripe (P10b, after launch if India-first)
Same adapter interface: Checkout Sessions + customer portal; webhooks `checkout.session.completed`, `invoice.paid`, `invoice.payment_failed`, `customer.subscription.updated`, `customer.subscription.deleted`; tax via Stripe Tax. Same idempotent event table.

### 7. Reconciliation
Nightly `billing.reconcile`: compare provider subscription status with ours and fix drift; alert on mismatches; check ledger invariants (balance ≥ 0, no duplicate event grants).

### 8. Admin console basics (internal)
A small internal page (admin role only, separate from customer roles): search orgs, see plan/status/credits/usage, grant credits with a reason, suspend/unsuspend an org. All actions audited.

## Deliverables
Plans, Razorpay checkout and webhooks, entitlements, credits and packs, billing screens, dunning, reconciliation, admin basics.

## Acceptance
- [ ] A test-mode card upgrades Starter → Pro; limits and credits change within seconds of the webhook
- [ ] Replaying the same webhook changes nothing (no double credits)
- [ ] A forged webhook (bad signature) is rejected
- [ ] Setting the plan from the browser console does nothing (calls are refused)
- [ ] A Starter org cannot exceed 2 seats or 1 brand even with a direct database insert
- [ ] Downgrade is blocked while limits are exceeded, with a clear list of what to remove
- [ ] Failed payment → banner, email, grace period, then read-only; no data deleted
- [ ] Credit pack purchase adds credits once; unused monthly credits expire at renewal
- [ ] Reconciliation catches a manually drifted subscription in staging

## Traps
- Redirect URLs are not proof of payment.
- Currency and tax: keep amounts in the smallest unit (paise/cents) as integers.
- Webhooks can arrive out of order; handlers must be order-tolerant (use event timestamps, set state from the latest known provider status).
- Don't ship real pricing numbers hard-coded in the UI; read `plans`.

## If time is short
Launch with Razorpay subscriptions + monthly credit grants + limit enforcement; credit packs and yearly plans right after.

## Kickoff prompt for Claude Code
> Read `CLAUDE.md`, `phases/P10-billing-and-entitlements.md`, `03-data-model.md` §4–5. Build the catalogue tables and enforcement triggers with tests, then the Razorpay adapter and the webhook handler with signature, replay and out-of-order tests, then the billing screens. Use test-mode keys only and list what I must configure in the Razorpay dashboard (plans, webhook URL and events).
