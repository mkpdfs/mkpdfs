# Credits-Based Billing — Design Spec

**Date:** 2026-06-11
**Status:** Approved (brainstorming session)
**Replaces:** Monthly Stripe subscriptions (Starter $29.99 / Professional $99.99)

## Summary

Replace the monthly subscription model with prepaid credits: **$10 USD buys 1,000 credits; 1 credit = 1 PDF page** (same unit as today's `pageCount`). Credits never expire. Optional opt-in auto-recharge tops the balance up automatically via a saved card. Enterprise (`plan='enterprise'`, e.g. Academia Connects) is untouched: unlimited, no credits, manually provisioned.

Decisions locked in with the user:

- Full replacement — Starter/Professional tiers are removed; no free tier. New users start at 0 credits and must purchase before generating PDFs (they can still create templates and API tokens).
- Auto-recharge is opt-in and configurable (threshold, default 100 credits; recharge amount fixed at one $10 pack).
- Credits never expire (no FIFO expiry ledger needed).
- Non-PDF limits become flat for everyone: 500 templates, 10 API tokens, 50 MB max PDF size. AI generation keeps a fixed monthly quota of 15/month, available to any user with `creditBalance > 0` (enterprise unlimited).

There are no existing paying users (greenfield rebuild), so no data migration of live subscribers is required.

## Architecture choice

**Stripe one-time Checkout + off-session PaymentIntents + balance in DynamoDB.** The credit balance lives in DynamoDB and PDF gating never calls Stripe at request time. Stripe is only involved at purchase time (Checkout) and recharge time (off-session PaymentIntent). Rejected alternatives: Stripe Billing credit grants (couples runtime gating to Stripe; immature) and $10/month metered subscription (not prepaid).

## Data model

### `mkpdfs-{env}-subscriptions` table (reused as per-user billing record)

Keep PK `userId`. Attribute changes:

| Attribute | Change | Notes |
|---|---|---|
| `plan` | now `'credits' \| 'enterprise'` | `'credits'` is the default for new users (replaces `'free'`) |
| `status` | keep | `'active'` for everyone except manual suspension |
| `creditBalance` | **new**, number | authoritative balance, default 0 |
| `autoRecharge` | **new**, boolean | default `false` |
| `rechargeThreshold` | **new**, number | default 100; recharge fires when balance drops below it |
| `rechargeInProgress` | **new**, boolean/absent | concurrency lock for auto-recharge (see below) |
| `stripeCustomerId` | keep | |
| `stripePaymentMethodId` | **new**, string | saved card for off-session recharge |
| `stripeSubscriptionId`, `stripePriceId` | **removed** | subscription-era fields |

### `mkpdfs-{env}-credit-ledger` table (new, Database stack)

- PK `userId`, SK `entryId` = `{ISO timestamp}#{uuid}` (chronological queries).
- Attributes: `type` (`'purchase' | 'debit' | 'auto_recharge'`), `amount` (signed: +1000 purchase, −N debit), `balanceAfter`, `stripePaymentIntentId` (purchases/recharges), `description`, `createdAt`.
- Doubles as the **webhook idempotency key**: before crediting, write a ledger item with a deterministic `entryId` derived from the Stripe event/PaymentIntent id under a `attribute_not_exists` condition; if the conditional write fails, the event was already processed — skip.
- Billing mode PAY_PER_REQUEST, RemovalPolicy DESTROY dev / RETAIN prod (match existing tables).

### `mkpdfs-{env}-usage` table

Unchanged schema. No longer used for PDF enforcement — kept as monthly stats for the dashboard and as the counter for the AI generation quota (15/month).

## Backend behavior

### Gating & debit (PDF generation, both `/pdf/generate` and `/v1/pdf/generate`)

- `subscriptionMiddleware` injects `creditBalance` (and flat limits) instead of plan-tier limits. It keeps auto-creating the billing record (`plan='credits'`, `creditBalance: 0`) on first request. The `status==='active'` 402 check stays.
- `checkLimitsMiddleware('pdf_generation')`: enterprise bypasses; otherwise if `pagesRequested > creditBalance` return **402** with `{ creditsRemaining, creditsRequested, message }` (402 fits "payment required" better than the current 429).
- Debit happens in the `after` hook on HTTP 200 only (same pattern as today's `usageTrackingMiddleware`): atomic `ADD creditBalance -:n` + ledger `debit` entry. The check-then-debit race across concurrent requests is accepted for v1 (worst case a user briefly goes slightly negative; `ADD` keeps the math consistent).
- Monthly usage tracking (`usageTrackingMiddleware`) keeps running for stats.

### Auto-recharge

Trigger: after a debit, if `autoRecharge === true` and new balance `< rechargeThreshold`:

1. Conditional update: set `rechargeInProgress = true` with `ConditionExpression: attribute_not_exists(rechargeInProgress) OR rechargeInProgress = false`. If the condition fails, another request already triggered it — do nothing.
2. Create an off-session PaymentIntent: $10, `customer = stripeCustomerId`, `payment_method = stripePaymentMethodId`, `off_session: true`, `confirm: true`, metadata `{ userId, kind: 'auto_recharge' }`.
3. `payment_intent.succeeded` webhook → idempotent ledger credit of 1,000 (`type: 'auto_recharge'`), `ADD creditBalance 1000`, clear `rechargeInProgress`.
4. `payment_intent.payment_failed` webhook → clear `rechargeInProgress`, set `autoRecharge = false`, ledger is not written. The UI surfaces this state ("auto-recharge failed — update your card and re-enable").

The recharge fires from the debit path (post-response, inside the `after` hook) — no new queue. If the Stripe call itself throws, clear the flag and log; next debit retries.

### Purchase flow (manual)

- `POST /stripe/create-checkout-session` (existing route, reworked): no `plan` body param; creates Checkout Session `mode: 'payment'` with the single credits price, `payment_intent_data: { setup_future_usage: 'off_session' }`, metadata `{ userId, kind: 'purchase' }`. Creates the Stripe Customer first if needed (unchanged logic).
- `checkout.session.completed` webhook (mode `payment`) → idempotent ledger credit of 1,000 (`type: 'purchase'`), `ADD creditBalance 1000`, store `stripePaymentMethodId` from the PaymentIntent's payment method (so auto-recharge can be enabled later without a separate card-entry flow).
- All `customer.subscription.*` webhook handling is deleted.

### New/changed routes

| Route | Change |
|---|---|
| `POST /stripe/create-checkout-session` | reworked: one-time payment, no plan param |
| `POST /stripe/create-portal-session` | kept; Stripe portal reconfigured (dashboard side) to payment-method management only |
| `POST /stripe/webhook` | reworked event set: `checkout.session.completed`, `payment_intent.succeeded`, `payment_intent.payment_failed` |
| `PUT /billing/auto-recharge` | **new** (Cognito only): body `{ enabled: boolean, threshold?: number }`; validates `threshold >= 1`; requires `stripePaymentMethodId` present to enable |
| `GET /billing/ledger` | **new** (Cognito only): paginated ledger entries for purchase history UI |
| `GET /user/profile` | response now includes `creditBalance`, `autoRecharge`, `rechargeThreshold`, flat limits |

### Limits cleanup

`SUBSCRIPTION_PLANS` in `src/libs/middleware/subscription.ts` collapses to two rows:

| plan | templates | API tokens | max PDF MB | AI generations/month |
|---|---|---|---|---|
| `credits` | 500 | 10 | 50 | 15 (requires `creditBalance > 0`) |
| `enterprise` | ∞ | ∞ | 100 | ∞ |

`pagesPerMonth` disappears as a concept; the PDF gate is the credit balance.

## Stripe configuration

- New Product/Price in Stripe (user creates in dashboard, both envs): "1,000 PDF credits", one-time, $10 USD.
- New SSM param `/mkpdfs/{env}/stripe-price-credits-1000` — **type String, never SecureString** (CFN template-parameter gotcha: SecureString aborts the deploy with exit 0).
- Old params `/mkpdfs/{env}/stripe-price-basic` and `stripe-price-professional` removed from config/CDK after cutover.
- `stripe-secret-key` / `stripe-webhook-secret` runtime-SSM pattern unchanged.
- User archives the old monthly Products in the Stripe dashboard (manual step).

## CDK changes

- **Database stack:** add `CreditLedgerTable`; grant read/write to webhook, PDF-generation, billing lambdas.
- **Api stack:** wire new routes (`PUT /billing/auto-recharge`, `GET /billing/ledger`); update env vars (drop old price IDs, add `STRIPE_PRICE_CREDITS_1000`, add `CREDIT_LEDGER_TABLE`); PDF lambdas get write access to subscriptions table (debit) and ledger.

## Frontend (mkpdfs-web)

- **Billing page** redesigned: big current-balance display, "Buy 1,000 PDFs — $10" button (existing checkout redirect + existing post-checkout polling), auto-recharge toggle + threshold input (calls `PUT /billing/auto-recharge`), purchase history from `GET /billing/ledger`, "Update card" → portal session. Banner when auto-recharge was disabled by a failed payment.
- **Landing pricing section:** the 4 plan cards collapse to a single credits card ($10 / 1,000 PDFs, credits never expire) + "Contact Sales" enterprise card. i18n strings updated (all locales).
- **Types:** `Subscription` type gains `creditBalance`/`autoRecharge`/`rechargeThreshold`, drops subscription-era fields; `SubscriptionLimits` drops `pagesPerMonth`/`aiGenerationsPerMonth` adjustments accordingly.
- Dashboard usage widgets switch from "X / 1,000 pages this month" to "X credits remaining" (+ monthly generated count as a stat).

## Error handling summary

| Failure | Behavior |
|---|---|
| Balance insufficient | 402 with `creditsRemaining`; UI links to billing page |
| Webhook delivered twice | Ledger conditional write makes credit idempotent |
| Concurrent debits trigger recharge twice | `rechargeInProgress` conditional lock |
| Off-session charge fails (card declined) | `autoRecharge` auto-disabled, flag cleared, UI banner |
| Stripe API error during recharge trigger | Flag cleared, logged; retried on next debit |
| PDF handler fails after gate passed | No debit (debit only on HTTP 200) |

## Testing

- Unit tests: middleware gating (402 paths, enterprise bypass), debit atomicity/ledger writes, webhook idempotency (duplicate event), auto-recharge trigger conditions and lock, payment-failure handling.
- Dev e2e: real Stripe test-mode purchase → balance credited → generate PDFs down past threshold → auto-recharge fires → balance tops up; declined-card test (`4000 0000 0000 9995`) disables auto-recharge.

## Out of scope (v1)

- Multiple pack sizes / volume discounts (single $10 pack only).
- Credit refunds on failed PDF jobs beyond "no debit unless 200".
- Email notifications for low balance / failed recharge (UI banner only).
- Migrating any existing subscriber data (none exist).
