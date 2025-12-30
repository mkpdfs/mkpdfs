# Stripe Integration - What Was Done

## Summary
This document describes all Stripe integration work completed for mkpdfs.

---

## BACKEND (mkpdfs-backend) - COMPLETED & DEPLOYED

### 1. Installed stripe package
```bash
npm install stripe
```

### 2. Created SSM Parameters (stored in AWS)
```
/mkpdfs/dev/stripe-secret-key = sk_test_XXXX (stored in AWS SSM)
/mkpdfs/dev/stripe-webhook-secret = whsec_XXXX (stored in AWS SSM)
/mkpdfs/dev/stripe-price-basic = price_1SjpmjHj5mtJUc59PToqCsTX
/mkpdfs/dev/stripe-price-professional = price_1SjpmvHj5mtJUc59B5AP8XLh
```

### 3. Updated serverless.ts
- Added Stripe env vars referencing SSM parameters
- Added IAM permission for SSM:GetParameter
- Added frontendUrls config for checkout redirects

### 4. Created Files

**src/libs/services/stripeService.ts**
- Stripe client initialization
- createCheckoutSession() - creates Stripe Checkout session
- createPortalSession() - creates Customer Portal session
- constructWebhookEvent() - verifies webhook signatures
- PRICE_TO_PLAN / PLAN_TO_PRICE mappings

**src/functions/stripe/createCheckoutSession/handler.ts & index.ts**
- POST /stripe/create-checkout-session
- Auth: AWS_IAM (Cognito)
- Creates checkout session, saves Stripe customer ID

**src/functions/stripe/webhook/handler.ts & index.ts**
- POST /stripe/webhook
- Auth: None (verified by Stripe signature)
- Handles: checkout.session.completed, customer.subscription.updated, customer.subscription.deleted, invoice.payment_failed

**src/functions/stripe/createPortalSession/handler.ts & index.ts**
- POST /stripe/create-portal-session
- Auth: AWS_IAM
- Creates Stripe Customer Portal session

### 5. Updated src/functions/index.ts
- Added exports for 3 new Stripe functions

### 6. Deployed to dev
- Backend deployed successfully
- Endpoints live at https://dev.apis.mkpdfs.com/stripe/*

### 7. Created Stripe Webhook
- URL: https://dev.apis.mkpdfs.com/stripe/webhook
- Events: checkout.session.completed, customer.subscription.updated, customer.subscription.deleted, invoice.payment_failed
- Webhook ID: we_1SjqtnHj5mtJUc59NZOQtih7

---

## FRONTEND (mkpdfs-web) - NOT COMPLETED

### What needs to be done:

**1. Install @stripe/stripe-js**
```bash
npm install @stripe/stripe-js
```

**2. Update src/lib/api.ts** - Add these functions:
```typescript
export async function createCheckoutSession(plan: string): Promise<{ url: string; sessionId: string }> {
  const response = await authFetch<{ success: boolean; url: string; sessionId: string }>(
    '/stripe/create-checkout-session',
    {
      method: 'POST',
      body: JSON.stringify({ plan }),
    }
  )
  return { url: response.url, sessionId: response.sessionId }
}

export async function createPortalSession(): Promise<{ url: string }> {
  const response = await authFetch<{ success: boolean; url: string }>(
    '/stripe/create-portal-session',
    {
      method: 'POST',
    }
  )
  return { url: response.url }
}
```

**3. Update src/types/index.ts** - Add to MkpdfsUser and Subscription:
```typescript
// Add to MkpdfsUser interface:
subscription?: Subscription
subscriptionLimits?: SubscriptionLimits
currentUsage?: CurrentUsage

// Add these interfaces:
export interface SubscriptionLimits {
  pdfGenerationsPerMonth: number
  templatesAllowed: number
  apiTokensAllowed: number
  maxPdfSizeMB: number
}

export interface CurrentUsage {
  userId: string
  yearMonth: string
  pdfCount: number
  totalSizeMB: number
}

// Update Subscription interface:
export interface Subscription {
  plan: SubscriptionPlan
  status: 'active' | 'cancelled' | 'past_due'
  currentPeriodEnd?: string
  stripeCustomerId?: string
  stripeSubscriptionId?: string
  stripePriceId?: string
}
```

**4. Update src/app/[locale]/(dashboard)/billing/page.tsx**
- Import useProfile hook
- Import createCheckoutSession, createPortalSession from api
- Add handleUpgrade function that calls createCheckoutSession and redirects
- Add handleManageSubscription function that calls createPortalSession and redirects
- Show actual current plan from profile.subscription.plan
- Show "Manage Subscription" button for users with stripeCustomerId

**5. Create src/app/[locale]/(dashboard)/billing/success/page.tsx**
- Success page after checkout
- Invalidate profile query to refresh subscription

**6. Create src/app/[locale]/(dashboard)/billing/cancel/page.tsx**
- Cancel page if user abandons checkout

---

## Stripe Dashboard Setup (Already Done)

- Products created: Basic ($29.99/mo), Professional ($99.99/mo)
- Price IDs: price_1SjpmjHj5mtJUc59PToqCsTX, price_1SjpmvHj5mtJUc59B5AP8XLh
- Webhook configured
- Test mode enabled

---

## Implementation Order (for completing frontend)

1. Fix current 404 error on frontend first
2. Install @stripe/stripe-js
3. Add API functions to api.ts
4. Update types
5. Update billing page
6. Create success/cancel pages
7. Test full flow in dev environment
