# Credits-Based Billing Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace monthly Stripe subscriptions with prepaid credits ($10 = 1,000 PDF pages, never expire) plus opt-in auto-recharge, rolled out to dev first then prod.

**Architecture:** Credit balance lives in DynamoDB (`subscriptions` table, atomic `ADD`); a new `credit-ledger` table is both audit trail and webhook idempotency key. Stripe is only touched at purchase time (one-time Checkout that saves the card) and recharge time (off-session PaymentIntent). PDF gating returns 402 when balance is insufficient; debit happens in the `after` hook on HTTP 200. Enterprise plan bypasses everything.

**Tech Stack:** AWS CDK (TypeScript), Lambda + Middy, DynamoDB, Stripe SDK v20, vitest + aws-sdk-client-mock, Next.js 15 + next-intl + react-query.

**Spec:** `docs/superpowers/specs/2026-06-11-credits-billing-design.md`

**Repos/branches:** Work happens in two submodules, each on its own `dev` branch: `mkpdfs-backend/` (Phases 1–2) and `mkpdfs-web/` (Phase 3). The orchestrator repo gets submodule-bump commits. Push to `dev` triggers CI deploy to the dev env; merge to `main` deploys prod.

**Environments:** Phase 0 + 2 + 4 target **dev** (Stripe test mode). Phase 5 is the **prod** cutover (Stripe live mode). SSM params must exist in BOTH envs before their respective deploys — `valueForStringParameter` resolves at deploy time and a missing/SecureString param aborts the deploy (exit 0 trap, check end of log).

---

## Phase 0 — Stripe test mode + SSM (dev prerequisites)

### Task 1: Create the Stripe test-mode price and dev SSM param

Manual steps in the Stripe **test mode** dashboard (user does these; the engineer verifies):

- [ ] **Step 1: Create Product/Price.** Stripe Dashboard (test mode) → Product catalog → Add product: Name `1,000 PDF credits`, description `1,000 PDF pages for mkpdfs. Credits never expire.` One-time price, $10.00 USD. Copy the price id (`price_...`).

- [ ] **Step 2: Write the dev SSM param (type String, NEVER SecureString):**

```bash
aws ssm put-parameter --profile rocketeast --region us-east-1 \
  --name /mkpdfs/dev/stripe-price-credits-1000 \
  --type String --value "price_XXXXXXXXXXXX"
```

- [ ] **Step 3: Update webhook events.** Dashboard → Developers → Webhooks → endpoint `https://dev.apis.mkpdfs.com/stripe/webhook`: enabled events must include `checkout.session.completed`, `payment_intent.succeeded`, `payment_intent.payment_failed`. Remove the `customer.subscription.*` and `invoice.payment_failed` events. The signing secret does not change.

- [ ] **Step 4: Configure the Customer Portal** (test mode). Settings → Billing → Customer portal: enable **Payment methods** (update allowed, sets default), disable the Subscriptions/Plans sections (no more subscriptions to manage).

- [ ] **Step 5: Verify the param exists:**

```bash
aws ssm get-parameter --profile rocketeast --name /mkpdfs/dev/stripe-price-credits-1000 --query Parameter.Type
```
Expected: `"String"`

---

## Phase 1 — Backend (`mkpdfs-backend/`, branch `dev`)

File structure for this phase:

| File | Action | Responsibility |
|---|---|---|
| `src/libs/creditConstants.ts` | Create | Pack size/price/threshold constants (separate file: creditService and stripeService both need them, avoid a circular import) |
| `src/libs/services/creditService.ts` | Create | All DynamoDB credit ops: idempotent credit, atomic debit, auto-recharge trigger, ledger query |
| `src/libs/services/creditService.test.ts` | Create | Unit tests (aws-sdk-client-mock) |
| `src/libs/middleware/credits.ts` | Create | `checkCreditsMiddleware` (402 gate) + `debitCreditsMiddleware` (after-hook debit) |
| `src/libs/middleware/credits.test.ts` | Create | Unit tests |
| `src/libs/middleware/subscription.ts` | Modify | Collapse plans to `credits`/`enterprise`, auto-create with credit fields |
| `src/libs/middleware/usageTracking.ts` | Modify | Delete `checkLimitsMiddleware` (replaced by credits gate); keep tracking |
| `src/libs/middleware/aiLimits.ts` | Modify | Gate AI on `creditBalance > 0` instead of paid plan |
| `src/libs/services/stripeService.ts` | Modify | One-time checkout + off-session recharge PI; drop subscription helpers |
| `src/functions/stripe/createCheckoutSession/handler.ts` | Modify | No `plan` param |
| `src/functions/stripe/webhook/handler.ts` | Rewrite | New event set |
| `src/functions/billing/updateAutoRecharge/handler.ts` | Create | `PUT /billing/auto-recharge` |
| `src/functions/billing/getLedger/handler.ts` | Create | `GET /billing/ledger` |
| `src/functions/pdf/generate/handler.ts`, `src/functions/pdf/generateApiKey/handler.ts`, `src/functions/jobs/submit/handler.ts` | Modify | Swap middleware chains |
| `src/functions/jobs/process/handler.ts` | Modify | Debit on async job success |
| `cdk/lib/config.ts`, `cdk/lib/service-function.ts`, `cdk/lib/stacks/database-stack.ts`, `cdk/lib/stacks/api-stack.ts`, `cdk/lib/stacks/jobs-stack.ts` | Modify | Ledger table, env vars, routes, grants |
| `vitest.config.ts` | Create | `@libs/*` alias so tests can import aliased modules |

### Task 2: Test infrastructure

**Files:**
- Modify: `mkpdfs-backend/package.json` (devDependency)
- Create: `mkpdfs-backend/vitest.config.ts`

- [ ] **Step 1: Install the DynamoDB client mock:**

```bash
cd mkpdfs-backend && npm install -D aws-sdk-client-mock
```

- [ ] **Step 2: Create `vitest.config.ts`** (aliases mirror `tsconfig.json` paths, baseUrl `src`):

```ts
import * as path from 'path';
import { defineConfig } from 'vitest/config';

export default defineConfig({
  resolve: {
    alias: {
      '@libs': path.resolve(__dirname, 'src/libs'),
      '@functions': path.resolve(__dirname, 'src/functions'),
      '@resources': path.resolve(__dirname, 'src/resources'),
    },
  },
});
```

- [ ] **Step 3: Verify the existing suite still runs:**

Run: `npm test`
Expected: the existing `codes.test.ts` passes.

- [ ] **Step 4: Commit**

```bash
git add package.json package-lock.json vitest.config.ts
git commit -m "test: add aws-sdk-client-mock and vitest aliases for @libs imports"
```

### Task 3: Credit constants + creditService (TDD)

**Files:**
- Create: `mkpdfs-backend/src/libs/creditConstants.ts`
- Create: `mkpdfs-backend/src/libs/services/creditService.ts`
- Test: `mkpdfs-backend/src/libs/services/creditService.test.ts`

- [ ] **Step 1: Create `src/libs/creditConstants.ts`:**

```ts
/** $10 buys 1,000 credits; 1 credit = 1 PDF page. Credits never expire. */
export const CREDITS_PER_PACK = 1000;
export const PACK_PRICE_CENTS = 1000; // $10.00 USD
export const DEFAULT_RECHARGE_THRESHOLD = 100;
```

- [ ] **Step 2: Write the failing tests** in `src/libs/services/creditService.test.ts`:

```ts
import { beforeEach, describe, expect, it, vi } from 'vitest';
import { mockClient } from 'aws-sdk-client-mock';
import {
  DynamoDBDocumentClient,
  PutCommand,
  TransactWriteCommand,
  UpdateCommand,
} from '@aws-sdk/lib-dynamodb';
import {
  creditFromStripePayment,
  debitCredits,
  maybeTriggerAutoRecharge,
} from './creditService';

vi.mock('./stripeService', () => ({
  createRechargePaymentIntent: vi.fn().mockResolvedValue({ id: 'pi_test' }),
}));
import { createRechargePaymentIntent } from './stripeService';

const ddbMock = mockClient(DynamoDBDocumentClient);

beforeEach(() => {
  ddbMock.reset();
  vi.clearAllMocks();
  process.env.SUBSCRIPTIONS_TABLE = 'subs';
  process.env.CREDIT_LEDGER_TABLE = 'ledger';
});

describe('creditFromStripePayment', () => {
  it('writes ledger + balance in one transaction and reports credited', async () => {
    ddbMock.on(TransactWriteCommand).resolves({});
    const res = await creditFromStripePayment({
      userId: 'u1', paymentIntentId: 'pi_1', type: 'purchase',
    });
    expect(res.credited).toBe(true);
    const tx = ddbMock.commandCalls(TransactWriteCommand)[0].args[0].input;
    expect(tx.TransactItems).toHaveLength(2);
    expect(tx.TransactItems![0].Put!.Item!.entryId).toBe('stripe#pi_1');
    expect(tx.TransactItems![0].Put!.ConditionExpression).toContain('attribute_not_exists');
    expect(tx.TransactItems![1].Update!.ExpressionAttributeValues![':amount']).toBe(1000);
  });

  it('is idempotent: a duplicate webhook delivery is a no-op', async () => {
    const err: any = new Error('cancelled');
    err.name = 'TransactionCanceledException';
    err.CancellationReasons = [{ Code: 'ConditionalCheckFailed' }, { Code: 'None' }];
    ddbMock.on(TransactWriteCommand).rejects(err);
    const res = await creditFromStripePayment({
      userId: 'u1', paymentIntentId: 'pi_1', type: 'purchase',
    });
    expect(res.credited).toBe(false);
  });
});

describe('debitCredits', () => {
  it('decrements atomically and writes a ledger entry with balanceAfter', async () => {
    ddbMock.on(UpdateCommand).resolves({ Attributes: { creditBalance: 990 } });
    ddbMock.on(PutCommand).resolves({});
    const after = await debitCredits({ userId: 'u1', amount: 10, description: 'pdf_generation' });
    expect(after).toBe(990);
    const update = ddbMock.commandCalls(UpdateCommand)[0].args[0].input;
    expect(update.ExpressionAttributeValues![':neg']).toBe(-10);
    const put = ddbMock.commandCalls(PutCommand)[0].args[0].input;
    expect(put.Item!.type).toBe('debit');
    expect(put.Item!.balanceAfter).toBe(990);
  });
});

describe('maybeTriggerAutoRecharge', () => {
  const billing = {
    userId: 'u1', plan: 'credits', status: 'active',
    autoRecharge: true, rechargeThreshold: 100,
    stripeCustomerId: 'cus_1', stripePaymentMethodId: 'pm_1',
  };

  it('does nothing when balance is above threshold', async () => {
    await maybeTriggerAutoRecharge({ billing, balanceAfter: 100 });
    expect(ddbMock.commandCalls(UpdateCommand)).toHaveLength(0);
  });

  it('does nothing when autoRecharge is off', async () => {
    await maybeTriggerAutoRecharge({ billing: { ...billing, autoRecharge: false }, balanceAfter: 5 });
    expect(ddbMock.commandCalls(UpdateCommand)).toHaveLength(0);
  });

  it('takes the lock and creates the off-session PaymentIntent', async () => {
    ddbMock.on(UpdateCommand).resolves({});
    await maybeTriggerAutoRecharge({ billing, balanceAfter: 50 });
    const lock = ddbMock.commandCalls(UpdateCommand)[0].args[0].input;
    expect(lock.ConditionExpression).toContain('rechargeInProgress');
    expect(createRechargePaymentIntent).toHaveBeenCalledWith({
      userId: 'u1', customerId: 'cus_1', paymentMethodId: 'pm_1',
    });
  });

  it('is silent when another request already holds the lock', async () => {
    const err: any = new Error('cond');
    err.name = 'ConditionalCheckFailedException';
    ddbMock.on(UpdateCommand).rejects(err);
    await maybeTriggerAutoRecharge({ billing, balanceAfter: 50 });
    expect(createRechargePaymentIntent).not.toHaveBeenCalled();
  });

  it('clears the lock if the Stripe call throws', async () => {
    ddbMock.on(UpdateCommand).resolves({});
    (createRechargePaymentIntent as any).mockRejectedValueOnce(new Error('stripe down'));
    await maybeTriggerAutoRecharge({ billing, balanceAfter: 50 });
    const updates = ddbMock.commandCalls(UpdateCommand);
    expect(updates).toHaveLength(2); // lock + clear
    expect(updates[1].args[0].input.ExpressionAttributeValues![':false']).toBe(false);
  });
});
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `npx vitest run src/libs/services/creditService.test.ts`
Expected: FAIL — `creditService` module not found.

- [ ] **Step 4: Implement `src/libs/services/creditService.ts`:**

```ts
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import {
  DynamoDBDocumentClient,
  PutCommand,
  QueryCommand,
  TransactWriteCommand,
  UpdateCommand,
} from '@aws-sdk/lib-dynamodb';
import { v4 as uuidv4 } from 'uuid';
import { CREDITS_PER_PACK, DEFAULT_RECHARGE_THRESHOLD } from '../creditConstants';
import { createRechargePaymentIntent } from './stripeService';

const dynamoClient = new DynamoDBClient({});
const docClient = DynamoDBDocumentClient.from(dynamoClient);

export type CreditType = 'purchase' | 'auto_recharge';

export interface BillingRecord {
  userId: string;
  plan: string;
  status: string;
  creditBalance?: number;
  autoRecharge?: boolean;
  rechargeThreshold?: number;
  rechargeInProgress?: boolean;
  stripeCustomerId?: string;
  stripePaymentMethodId?: string;
  autoRechargeError?: string;
}

/**
 * Idempotently credit a Stripe payment. A single transaction writes the
 * ledger entry (conditional — the dedup key is `stripe#<paymentIntentId>`)
 * and increments the balance, so a redelivered webhook credits exactly once
 * and a crash can never apply one half without the other.
 */
export async function creditFromStripePayment(params: {
  userId: string;
  paymentIntentId: string;
  type: CreditType;
  amount?: number;
}): Promise<{ credited: boolean }> {
  const amount = params.amount ?? CREDITS_PER_PACK;
  const now = new Date().toISOString();
  try {
    await docClient.send(new TransactWriteCommand({
      TransactItems: [
        {
          Put: {
            TableName: process.env.CREDIT_LEDGER_TABLE!,
            Item: {
              userId: params.userId,
              entryId: `stripe#${params.paymentIntentId}`,
              type: params.type,
              amount,
              stripePaymentIntentId: params.paymentIntentId,
              createdAt: now,
            },
            ConditionExpression: 'attribute_not_exists(userId)',
          },
        },
        {
          Update: {
            TableName: process.env.SUBSCRIPTIONS_TABLE!,
            Key: { userId: params.userId },
            UpdateExpression: 'SET updatedAt = :now ADD creditBalance :amount',
            ExpressionAttributeValues: { ':now': now, ':amount': amount },
          },
        },
      ],
    }));
    return { credited: true };
  } catch (err: any) {
    if (
      err.name === 'TransactionCanceledException' &&
      err.CancellationReasons?.some((r: any) => r.Code === 'ConditionalCheckFailed')
    ) {
      console.log(`[credits] PaymentIntent ${params.paymentIntentId} already credited — skipping`);
      return { credited: false };
    }
    throw err;
  }
}

/** Atomic debit + ledger entry. Returns the balance after the debit. */
export async function debitCredits(params: {
  userId: string;
  amount: number;
  description: string;
}): Promise<number> {
  const now = new Date().toISOString();
  const result = await docClient.send(new UpdateCommand({
    TableName: process.env.SUBSCRIPTIONS_TABLE!,
    Key: { userId: params.userId },
    UpdateExpression: 'SET updatedAt = :now ADD creditBalance :neg',
    ExpressionAttributeValues: { ':now': now, ':neg': -params.amount },
    ReturnValues: 'UPDATED_NEW',
  }));
  const balanceAfter = (result.Attributes?.creditBalance as number) ?? 0;
  await docClient.send(new PutCommand({
    TableName: process.env.CREDIT_LEDGER_TABLE!,
    Item: {
      userId: params.userId,
      entryId: `${now}#${uuidv4()}`,
      type: 'debit',
      amount: -params.amount,
      balanceAfter,
      description: params.description,
      createdAt: now,
    },
  }));
  return balanceAfter;
}

/**
 * Called after a debit. The rechargeInProgress conditional lock guarantees a
 * single in-flight PaymentIntent per user under concurrent debits; the
 * webhook (payment_intent.succeeded / payment_failed) clears the lock.
 */
export async function maybeTriggerAutoRecharge(params: {
  billing: BillingRecord;
  balanceAfter: number;
}): Promise<void> {
  const { billing, balanceAfter } = params;
  if (!billing.autoRecharge || !billing.stripeCustomerId || !billing.stripePaymentMethodId) {
    return;
  }
  const threshold = billing.rechargeThreshold ?? DEFAULT_RECHARGE_THRESHOLD;
  if (balanceAfter >= threshold) return;

  try {
    await docClient.send(new UpdateCommand({
      TableName: process.env.SUBSCRIPTIONS_TABLE!,
      Key: { userId: billing.userId },
      UpdateExpression: 'SET rechargeInProgress = :true, updatedAt = :now',
      ConditionExpression:
        'attribute_not_exists(rechargeInProgress) OR rechargeInProgress = :false',
      ExpressionAttributeValues: {
        ':true': true,
        ':false': false,
        ':now': new Date().toISOString(),
      },
    }));
  } catch (err: any) {
    if (err.name === 'ConditionalCheckFailedException') return; // already in flight
    throw err;
  }

  try {
    await createRechargePaymentIntent({
      userId: billing.userId,
      customerId: billing.stripeCustomerId,
      paymentMethodId: billing.stripePaymentMethodId,
    });
  } catch (err) {
    // Card declines also land here (off-session confirm throws). The
    // payment_intent.payment_failed webhook disables autoRecharge; clearing
    // the lock here covers pure API failures where no PI was created.
    console.error('[credits] auto-recharge PaymentIntent failed:', err);
    await docClient.send(new UpdateCommand({
      TableName: process.env.SUBSCRIPTIONS_TABLE!,
      Key: { userId: billing.userId },
      UpdateExpression: 'SET rechargeInProgress = :false',
      ExpressionAttributeValues: { ':false': false },
    })).catch((e) => console.error('[credits] failed to clear recharge lock:', e));
  }
}

/** Ledger entries, newest first. SK order is mixed (stripe#… vs ISO#…), so sort by createdAt. */
export async function listLedgerEntries(userId: string, limit = 50) {
  const result = await docClient.send(new QueryCommand({
    TableName: process.env.CREDIT_LEDGER_TABLE!,
    KeyConditionExpression: 'userId = :userId',
    ExpressionAttributeValues: { ':userId': userId },
  }));
  const items = result.Items ?? [];
  items.sort((a, b) => ((a.createdAt as string) < (b.createdAt as string) ? 1 : -1));
  return items.slice(0, limit);
}
```

NOTE: `createRechargePaymentIntent` doesn't exist in `stripeService.ts` yet — that's Task 7. To keep this task green now, add this temporary export at the bottom of `src/libs/services/stripeService.ts` (Task 7 replaces it with the real one):

```ts
// TEMP — implemented for real in the credits rework (see plan Task 7)
export async function createRechargePaymentIntent(_params: {
  userId: string; customerId: string; paymentMethodId: string;
}): Promise<{ id: string }> {
  throw new Error('not implemented yet');
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `npx vitest run src/libs/services/creditService.test.ts`
Expected: PASS (8 tests).

- [ ] **Step 6: Commit**

```bash
git add src/libs/creditConstants.ts src/libs/services/creditService.ts src/libs/services/creditService.test.ts src/libs/services/stripeService.ts
git commit -m "feat(credits): creditService — idempotent Stripe credits, atomic debits, auto-recharge trigger"
```

### Task 4: Rework subscription middleware (plans collapse to credits/enterprise)

**Files:**
- Modify: `mkpdfs-backend/src/libs/middleware/subscription.ts`

- [ ] **Step 1: Replace the limits interface and plan table** (lines 7–44). New content:

```ts
export interface SubscriptionLimits {
  templatesAllowed: number;
  apiTokensAllowed: number;
  maxPdfSizeMB: number;
  aiGenerationsPerMonth: number; // fixed monthly quota, requires creditBalance > 0
}

export const SUBSCRIPTION_PLANS: Record<string, SubscriptionLimits> = {
  credits: {
    templatesAllowed: 500,
    apiTokensAllowed: 10,
    maxPdfSizeMB: 50,
    aiGenerationsPerMonth: 15,
  },
  enterprise: {
    templatesAllowed: -1, // unlimited
    apiTokensAllowed: -1,
    maxPdfSizeMB: 100,
    aiGenerationsPerMonth: -1,
  },
};
```

- [ ] **Step 2: Update the auto-create block** (the `if (!subscription)` branch, currently lines 64–89) to create a credits record:

```ts
        // If no record, create the default credits billing record (balance 0)
        if (!subscription) {
          const now = new Date().toISOString();
          subscription = {
            userId,
            plan: 'credits',
            status: 'active',
            creditBalance: 0,
            autoRecharge: false,
            rechargeThreshold: 100,
            createdAt: now,
            updatedAt: now
          };

          await docClient.send(new UpdateCommand({
            TableName: process.env.SUBSCRIPTIONS_TABLE!,
            Key: { userId },
            UpdateExpression:
              'SET #plan = :plan, #status = :status, creditBalance = :balance, ' +
              'autoRecharge = :autoRecharge, rechargeThreshold = :threshold, ' +
              'createdAt = :createdAt, updatedAt = :updatedAt',
            ExpressionAttributeNames: {
              '#plan': 'plan',
              '#status': 'status'
            },
            ExpressionAttributeValues: {
              ':plan': 'credits',
              ':status': 'active',
              ':balance': 0,
              ':autoRecharge': false,
              ':threshold': 100,
              ':createdAt': now,
              ':updatedAt': now
            }
          }));
        }
```

- [ ] **Step 3: Update the limits lookup** (line 126). Any plan that isn't `enterprise` gets credits limits (this also neutralizes stale `free`/`starter` rows in the dev table):

```ts
        handler.event.subscription = subscription;
        handler.event.subscriptionLimits =
          subscription.plan === 'enterprise'
            ? SUBSCRIPTION_PLANS.enterprise
            : SUBSCRIPTION_PLANS.credits;
        handler.event.currentUsage = usage;
```

- [ ] **Step 4: Update the catch fallback** (lines 129–135). Fail closed — balance 0 blocks generation but other routes keep working:

```ts
      } catch (error) {
        console.error('Error checking subscription:', error);
        // Fail closed for credits: a DDB read error must not grant free PDFs
        handler.event.subscription = { plan: 'credits', status: 'active', creditBalance: 0 };
        handler.event.subscriptionLimits = SUBSCRIPTION_PLANS.credits;
        handler.event.currentUsage = { pdfCount: 0, totalSizeMB: 0 };
      }
```

- [ ] **Step 5: Find and fix every `pagesPerMonth` consumer:**

Run: `grep -rn "pagesPerMonth\|SUBSCRIPTION_PLANS\.\(free\|starter\|professional\)" src/`
Fix each hit (besides the middleware itself). Known consumer: `checkLimitsMiddleware` in `usageTracking.ts` — handled in Task 5. Any other hit (e.g. a profile/usage handler echoing limits) just passes the limits object through and needs no change; only code that *reads* `pagesPerMonth` must be updated to stop doing so.

- [ ] **Step 6: Typecheck** — `npm run typecheck`. Expected: errors ONLY in files this plan still rewrites (`usageTracking.ts` if it references removed fields). Anything else, fix now.

- [ ] **Step 7: Commit**

```bash
git add src/libs/middleware/subscription.ts
git commit -m "feat(credits): collapse plans to credits/enterprise, auto-create with credit fields"
```

### Task 5: Credits middleware (TDD) + delete the old pages gate

**Files:**
- Create: `mkpdfs-backend/src/libs/middleware/credits.ts`
- Test: `mkpdfs-backend/src/libs/middleware/credits.test.ts`
- Modify: `mkpdfs-backend/src/libs/middleware/usageTracking.ts`

- [ ] **Step 1: Write the failing tests** in `src/libs/middleware/credits.test.ts`:

```ts
import { beforeEach, describe, expect, it, vi } from 'vitest';

vi.mock('@libs/services/creditService', () => ({
  debitCredits: vi.fn().mockResolvedValue(990),
  maybeTriggerAutoRecharge: vi.fn().mockResolvedValue(undefined),
}));
import { debitCredits, maybeTriggerAutoRecharge } from '@libs/services/creditService';
import { checkCreditsMiddleware, debitCreditsMiddleware } from './credits';

beforeEach(() => vi.clearAllMocks());

const makeHandler = (over: any = {}) => ({
  event: {
    userId: 'u1',
    subscription: { userId: 'u1', plan: 'credits', status: 'active', creditBalance: 5 },
    body: { data: [{}, {}, {}] }, // 3 pages
    ...over.event,
  },
  response: over.response,
});

describe('checkCreditsMiddleware', () => {
  it('lets the request through when balance covers the pages', async () => {
    const h = makeHandler();
    expect(await checkCreditsMiddleware().before(h)).toBeUndefined();
  });

  it('returns 402 with creditsRemaining when balance is insufficient', async () => {
    const h = makeHandler({ event: { userId: 'u1', subscription: { plan: 'credits', creditBalance: 2 }, body: { data: [{}, {}, {}] } } });
    const res: any = await checkCreditsMiddleware().before(h);
    expect(res.statusCode).toBe(402);
    const body = JSON.parse(res.body);
    expect(body.error).toBe('INSUFFICIENT_CREDITS');
    expect(body.creditsRemaining).toBe(2);
    expect(body.creditsRequested).toBe(3);
  });

  it('bypasses enterprise', async () => {
    const h = makeHandler({ event: { userId: 'u1', subscription: { plan: 'enterprise' }, body: { data: new Array(99).fill({}) } } });
    expect(await checkCreditsMiddleware().before(h)).toBeUndefined();
  });
});

describe('debitCreditsMiddleware', () => {
  it('debits pageCount and triggers auto-recharge check on 200', async () => {
    const h = makeHandler({ response: { statusCode: 200 } });
    h.event.pageCount = 3;
    await debitCreditsMiddleware().after(h);
    expect(debitCredits).toHaveBeenCalledWith({ userId: 'u1', amount: 3, description: 'pdf_generation' });
    expect(maybeTriggerAutoRecharge).toHaveBeenCalledWith({ billing: h.event.subscription, balanceAfter: 990 });
  });

  it('does not debit on non-200', async () => {
    const h = makeHandler({ response: { statusCode: 500 } });
    await debitCreditsMiddleware().after(h);
    expect(debitCredits).not.toHaveBeenCalled();
  });

  it('does not debit enterprise', async () => {
    const h = makeHandler({ response: { statusCode: 200 } });
    h.event.subscription.plan = 'enterprise';
    await debitCreditsMiddleware().after(h);
    expect(debitCredits).not.toHaveBeenCalled();
  });

  it('never fails the request if the debit throws', async () => {
    (debitCredits as any).mockRejectedValueOnce(new Error('ddb down'));
    const h = makeHandler({ response: { statusCode: 200 } });
    await expect(debitCreditsMiddleware().after(h)).resolves.toBeUndefined();
  });
});
```

- [ ] **Step 2: Run to verify failure** — `npx vitest run src/libs/middleware/credits.test.ts`. Expected: FAIL (module not found).

- [ ] **Step 3: Implement `src/libs/middleware/credits.ts`:**

```ts
import { debitCredits, maybeTriggerAutoRecharge } from '@libs/services/creditService';

const corsHeaders = {
  'Content-Type': 'application/json',
  'Access-Control-Allow-Origin': '*',
  'Access-Control-Allow-Credentials': true,
};

const pagesIn = (body: any): number =>
  body?.data && Array.isArray(body.data) ? body.data.length : 1;

/** 402 gate: requires subscriptionMiddleware to have run (event.subscription). */
export const checkCreditsMiddleware = () => ({
  before: async (handler: any): Promise<any> => {
    const sub = handler.event.subscription;
    if (!sub || sub.plan === 'enterprise') return;

    const creditsRequested = pagesIn(handler.event.body);
    const creditsRemaining = sub.creditBalance ?? 0;

    if (creditsRequested > creditsRemaining) {
      return {
        statusCode: 402,
        headers: corsHeaders,
        body: JSON.stringify({
          error: 'INSUFFICIENT_CREDITS',
          message:
            creditsRemaining <= 0
              ? 'You have no PDF credits. Buy a credit pack to continue ($10 = 1,000 PDFs).'
              : `Insufficient credits: ${creditsRemaining} remaining, ${creditsRequested} requested.`,
          creditsRemaining: Math.max(0, creditsRemaining),
          creditsRequested,
        }),
      };
    }
  },
});

/** Debit on HTTP 200 only (mirrors usageTrackingMiddleware), then check auto-recharge. */
export const debitCreditsMiddleware = () => ({
  after: async (handler: any): Promise<void> => {
    if (handler.response?.statusCode !== 200) return;
    const sub = handler.event.subscription;
    const userId = handler.event.userId;
    if (!userId || !sub || sub.plan === 'enterprise') return;

    const pageCount = handler.event.pageCount || 1;
    try {
      const balanceAfter = await debitCredits({
        userId,
        amount: pageCount,
        description: 'pdf_generation',
      });
      await maybeTriggerAutoRecharge({ billing: sub, balanceAfter });
    } catch (error) {
      console.error('[credits] debit failed (request NOT failed):', error);
    }
  },
});
```

- [ ] **Step 4: Run to verify pass** — `npx vitest run src/libs/middleware/credits.test.ts`. Expected: PASS (7 tests).

- [ ] **Step 5: Delete `checkLimitsMiddleware` and `calculatePageCount`** from `src/libs/middleware/usageTracking.ts` (lines 115–183). Keep `usageTrackingMiddleware` untouched — monthly stats stay.

- [ ] **Step 6: Commit**

```bash
git add src/libs/middleware/credits.ts src/libs/middleware/credits.test.ts src/libs/middleware/usageTracking.ts
git commit -m "feat(credits): 402 credit gate + after-hook debit middleware; drop monthly pages gate"
```

### Task 6: AI gate requires positive balance

**Files:**
- Modify: `mkpdfs-backend/src/libs/middleware/aiLimits.ts`

- [ ] **Step 1: Replace the free-tier check** (lines 10–29) with a balance check; the monthly-quota block below it stays as is:

```ts
      const subscription = handler.event.subscription;
      const plan = subscription?.plan || 'credits';

      // AI is included for anyone with a positive credit balance (enterprise always)
      if (plan !== 'enterprise' && (subscription?.creditBalance ?? 0) <= 0) {
        return {
          statusCode: 403,
          headers: {
            'Content-Type': 'application/json',
            'Access-Control-Allow-Origin': '*',
            'Access-Control-Allow-Credentials': true,
          },
          body: JSON.stringify({
            success: false,
            error: 'INSUFFICIENT_CREDITS',
            message: 'AI Template Generation requires a positive credit balance. Buy a credit pack ($10 = 1,000 PDFs) to unlock it.',
            purchaseRequired: true
          })
        };
      }
```

Delete the now-unused `const limits = handler.event.subscriptionLimits;` ONLY if it's no longer referenced — it IS still used by the monthly-limit block, so keep it.

- [ ] **Step 2: Run full suite + typecheck** — `npm test && npm run typecheck`. Expected: PASS / clean except files still pending rewrite.

- [ ] **Step 3: Commit**

```bash
git add src/libs/middleware/aiLimits.ts
git commit -m "feat(credits): AI generation gated on positive credit balance (15/month quota unchanged)"
```

### Task 7: Rework stripeService for one-time purchases + off-session recharges

**Files:**
- Modify: `mkpdfs-backend/src/libs/services/stripeService.ts`

- [ ] **Step 1: Delete the subscription-era exports**: `PRICE_TO_PLAN`, `PLAN_TO_PRICE` (lines 45–56), `getSubscription` (135–140), and the TEMP `createRechargePaymentIntent` stub from Task 3. Keep `getStripe`, `getWebhookSecret`, `createPortalSession`, `constructWebhookEvent`, `getCustomer`.

- [ ] **Step 2: Replace `createCheckoutSession`** (its params interface and function, lines 58–111):

```ts
import { CREDITS_PER_PACK, PACK_PRICE_CENTS } from '../creditConstants';

export interface CreateCheckoutSessionParams {
  userId: string;
  userEmail: string;
  stripeCustomerId?: string;
}

/**
 * One-time payment for a credit pack. setup_future_usage saves the card so
 * auto-recharge can charge off-session later without a separate card flow.
 */
export async function createCheckoutSession({
  userId,
  userEmail,
  stripeCustomerId,
}: CreateCheckoutSessionParams): Promise<Stripe.Checkout.Session> {
  const frontendUrl = process.env.FRONTEND_URL!;
  const stripe = await getStripe();

  let customerId = stripeCustomerId;
  if (!customerId) {
    const customer = await stripe.customers.create({
      email: userEmail,
      metadata: { userId },
    });
    customerId = customer.id;
  }

  const session = await stripe.checkout.sessions.create({
    customer: customerId,
    mode: 'payment',
    payment_method_types: ['card'],
    allow_promotion_codes: true,
    line_items: [
      {
        price: process.env.STRIPE_PRICE_CREDITS_1000!,
        quantity: 1,
      },
    ],
    payment_intent_data: {
      setup_future_usage: 'off_session',
      metadata: { userId, kind: 'purchase' },
    },
    success_url: `${frontendUrl}/en/billing/success?session_id={CHECKOUT_SESSION_ID}`,
    cancel_url: `${frontendUrl}/en/billing/cancel`,
    metadata: { userId, kind: 'purchase' },
  });

  return session;
}

/**
 * Off-session $10 charge for auto-recharge. Prefers the customer's default
 * payment method (the Stripe portal updates that) and falls back to the card
 * saved at first purchase. Card declines throw — the payment_failed webhook
 * handles disabling autoRecharge.
 */
export async function createRechargePaymentIntent(params: {
  userId: string;
  customerId: string;
  paymentMethodId: string;
}): Promise<Stripe.PaymentIntent> {
  const stripe = await getStripe();
  const customer = await stripe.customers.retrieve(params.customerId);
  const defaultPm =
    !('deleted' in customer && customer.deleted) &&
    typeof customer.invoice_settings?.default_payment_method === 'string'
      ? customer.invoice_settings.default_payment_method
      : undefined;

  return stripe.paymentIntents.create({
    amount: PACK_PRICE_CENTS,
    currency: 'usd',
    customer: params.customerId,
    payment_method: defaultPm ?? params.paymentMethodId,
    off_session: true,
    confirm: true,
    description: `mkpdfs auto-recharge: ${CREDITS_PER_PACK} PDF credits`,
    metadata: { userId: params.userId, kind: 'auto_recharge' },
  });
}
```

- [ ] **Step 3: Typecheck** — `npm run typecheck`. Expected: errors only in `createCheckoutSession/handler.ts` and `webhook/handler.ts` (they import the deleted maps) — fixed in Tasks 8–9.

- [ ] **Step 4: Commit**

```bash
git add src/libs/services/stripeService.ts
git commit -m "feat(credits): one-time checkout with saved card + off-session recharge PaymentIntent"
```

### Task 8: Checkout handler — no plan parameter

**Files:**
- Modify: `mkpdfs-backend/src/functions/stripe/createCheckoutSession/handler.ts`

- [ ] **Step 1: Drop the plan/price resolution.** Remove the `CheckoutRequest` interface, the `plan` destructure/validation (lines 11–27) and the `PLAN_TO_PRICE` import. The handler body becomes:

```ts
import { ValidatedEventAPIGatewayProxyEvent, formatJSONResponse, formatErrorResponse } from '@libs/apiGateway';
import { middyfy } from '@libs/lambda';
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient, GetCommand, UpdateCommand } from '@aws-sdk/lib-dynamodb';
import { iamOnlyMiddleware } from '@libs/middleware/dualAuth';
import { createCheckoutSession } from '@libs/services/stripeService';

const dynamoClient = new DynamoDBClient({});
const docClient = DynamoDBDocumentClient.from(dynamoClient);

const handler: ValidatedEventAPIGatewayProxyEvent<null> = async (event: any) => {
  try {
    const userId = event.userId!;

    const userData = await docClient.send(new GetCommand({
      TableName: process.env.USERS_TABLE!,
      Key: { userId }
    }));

    const user = userData.Item;
    if (!user) {
      return formatErrorResponse(new Error('User not found'), 404);
    }

    const subscriptionData = await docClient.send(new GetCommand({
      TableName: process.env.SUBSCRIPTIONS_TABLE!,
      Key: { userId }
    }));

    const stripeCustomerId = subscriptionData.Item?.stripeCustomerId;

    const session = await createCheckoutSession({
      userId,
      userEmail: user.email,
      stripeCustomerId,
    });

    if (!stripeCustomerId && session.customer) {
      await docClient.send(new UpdateCommand({
        TableName: process.env.SUBSCRIPTIONS_TABLE!,
        Key: { userId },
        UpdateExpression: 'SET stripeCustomerId = :customerId, updatedAt = :updatedAt',
        ExpressionAttributeValues: {
          ':customerId': session.customer,
          ':updatedAt': new Date().toISOString(),
        }
      }));
    }

    return formatJSONResponse({
      success: true,
      url: session.url,
      sessionId: session.id,
    });

  } catch (error) {
    console.error('Error creating checkout session:', error);
    return formatErrorResponse(error);
  }
};

export const main = middyfy(handler)
  .use(iamOnlyMiddleware());
```

- [ ] **Step 2: Commit**

```bash
git add src/functions/stripe/createCheckoutSession/handler.ts
git commit -m "feat(credits): checkout handler sells the single credit pack (no plan param)"
```

### Task 9: Webhook rewrite — purchase, recharge success, recharge failure

**Files:**
- Rewrite: `mkpdfs-backend/src/functions/stripe/webhook/handler.ts`

- [ ] **Step 1: Replace the whole file.** Keep the signature-verification block (lines 10–45) verbatim; replace the switch and all `handle*` functions:

```ts
import type { APIGatewayProxyHandler, APIGatewayProxyResult } from 'aws-lambda';
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient, UpdateCommand } from '@aws-sdk/lib-dynamodb';
import { constructWebhookEvent, getStripe } from '@libs/services/stripeService';
import { creditFromStripePayment } from '@libs/services/creditService';
import type Stripe from 'stripe';

const dynamoClient = new DynamoDBClient({});
const docClient = DynamoDBDocumentClient.from(dynamoClient);

const handler: APIGatewayProxyHandler = async (event): Promise<APIGatewayProxyResult> => {
  const signature = event.headers['Stripe-Signature'] || event.headers['stripe-signature'];

  if (!signature) {
    return {
      statusCode: 400,
      headers: { 'Content-Type': 'application/json', 'Access-Control-Allow-Origin': '*' },
      body: JSON.stringify({ error: 'Missing Stripe signature' }),
    };
  }

  let stripeEvent: Stripe.Event;

  try {
    const rawBody = event.isBase64Encoded
      ? Buffer.from(event.body!, 'base64').toString('utf8')
      : event.body!;
    stripeEvent = await constructWebhookEvent(rawBody, signature);
  } catch (error: any) {
    console.error('Webhook signature verification failed:', error.message);
    return {
      statusCode: 400,
      headers: { 'Content-Type': 'application/json', 'Access-Control-Allow-Origin': '*' },
      body: JSON.stringify({ error: `Webhook Error: ${error.message}` }),
    };
  }

  console.log('Received Stripe event:', stripeEvent.type);

  try {
    switch (stripeEvent.type) {
      case 'checkout.session.completed': {
        const session = stripeEvent.data.object as Stripe.Checkout.Session;
        if (session.mode === 'payment') {
          await handlePurchaseCompleted(session);
        }
        break;
      }

      // Auto-recharge PIs only: purchase PIs are credited via the checkout
      // event (same idempotency key — paymentIntentId — so even event overlap
      // can't double-credit).
      case 'payment_intent.succeeded': {
        const pi = stripeEvent.data.object as Stripe.PaymentIntent;
        if (pi.metadata?.kind === 'auto_recharge') {
          await handleRechargeSucceeded(pi);
        }
        break;
      }

      case 'payment_intent.payment_failed': {
        const pi = stripeEvent.data.object as Stripe.PaymentIntent;
        if (pi.metadata?.kind === 'auto_recharge') {
          await handleRechargeFailed(pi);
        }
        break;
      }

      default:
        console.log(`Unhandled event type: ${stripeEvent.type}`);
    }

    return {
      statusCode: 200,
      headers: { 'Content-Type': 'application/json', 'Access-Control-Allow-Origin': '*' },
      body: JSON.stringify({ received: true }),
    };
  } catch (error: any) {
    console.error('Error processing webhook:', error);
    return {
      statusCode: 500,
      headers: { 'Content-Type': 'application/json', 'Access-Control-Allow-Origin': '*' },
      body: JSON.stringify({ error: 'Internal server error' }),
    };
  }
};

async function handlePurchaseCompleted(session: Stripe.Checkout.Session) {
  const userId = session.metadata?.userId;
  if (!userId) {
    console.error('No userId in checkout session metadata');
    return;
  }
  const paymentIntentId = session.payment_intent as string;
  const customerId = session.customer as string;

  const { credited } = await creditFromStripePayment({
    userId,
    paymentIntentId,
    type: 'purchase',
  });
  console.log(`Purchase for user ${userId}: pi=${paymentIntentId} credited=${credited}`);

  // Save the card for future off-session auto-recharges
  const stripe = await getStripe();
  const pi = await stripe.paymentIntents.retrieve(paymentIntentId);
  const paymentMethodId = typeof pi.payment_method === 'string' ? pi.payment_method : pi.payment_method?.id;

  await docClient.send(new UpdateCommand({
    TableName: process.env.SUBSCRIPTIONS_TABLE!,
    Key: { userId },
    UpdateExpression:
      'SET #plan = :plan, #status = :status, stripeCustomerId = :customerId, updatedAt = :now' +
      (paymentMethodId ? ', stripePaymentMethodId = :pm' : ''),
    ExpressionAttributeNames: { '#plan': 'plan', '#status': 'status' },
    ExpressionAttributeValues: {
      ':plan': 'credits',
      ':status': 'active',
      ':customerId': customerId,
      ':now': new Date().toISOString(),
      ...(paymentMethodId ? { ':pm': paymentMethodId } : {}),
    },
  }));
}

async function handleRechargeSucceeded(pi: Stripe.PaymentIntent) {
  const userId = pi.metadata.userId;
  if (!userId) {
    console.error('No userId in recharge PaymentIntent metadata');
    return;
  }
  const { credited } = await creditFromStripePayment({
    userId,
    paymentIntentId: pi.id,
    type: 'auto_recharge',
  });
  console.log(`Auto-recharge for user ${userId}: pi=${pi.id} credited=${credited}`);

  await docClient.send(new UpdateCommand({
    TableName: process.env.SUBSCRIPTIONS_TABLE!,
    Key: { userId },
    UpdateExpression: 'SET rechargeInProgress = :false, updatedAt = :now REMOVE autoRechargeError',
    ExpressionAttributeValues: { ':false': false, ':now': new Date().toISOString() },
  }));
}

async function handleRechargeFailed(pi: Stripe.PaymentIntent) {
  const userId = pi.metadata.userId;
  if (!userId) {
    console.error('No userId in failed recharge PaymentIntent metadata');
    return;
  }
  const message = pi.last_payment_error?.message || 'Payment failed';
  console.log(`Auto-recharge FAILED for user ${userId}: ${message}`);

  // Disable auto-recharge so we don't retry-charge a bad card; the UI shows
  // a banner and the user re-enables after updating the card.
  await docClient.send(new UpdateCommand({
    TableName: process.env.SUBSCRIPTIONS_TABLE!,
    Key: { userId },
    UpdateExpression:
      'SET rechargeInProgress = :false, autoRecharge = :false, autoRechargeError = :msg, updatedAt = :now',
    ExpressionAttributeValues: {
      ':false': false,
      ':msg': message,
      ':now': new Date().toISOString(),
    },
  }));
}

export const main = handler;
```

- [ ] **Step 2: Typecheck** — `npm run typecheck`. Expected: clean (the deleted stripeService exports have no remaining importers).

- [ ] **Step 3: Commit**

```bash
git add src/functions/stripe/webhook/handler.ts
git commit -m "feat(credits): webhook credits purchases idempotently, handles auto-recharge success/failure"
```

### Task 10: New billing endpoints (auto-recharge toggle + ledger)

**Files:**
- Create: `mkpdfs-backend/src/functions/billing/updateAutoRecharge/handler.ts`
- Create: `mkpdfs-backend/src/functions/billing/getLedger/handler.ts`

- [ ] **Step 1: Create `src/functions/billing/updateAutoRecharge/handler.ts`:**

```ts
import { ValidatedEventAPIGatewayProxyEvent, formatJSONResponse, formatErrorResponse } from '@libs/apiGateway';
import { middyfy } from '@libs/lambda';
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient, GetCommand, UpdateCommand } from '@aws-sdk/lib-dynamodb';
import { iamOnlyMiddleware } from '@libs/middleware/dualAuth';
import { DEFAULT_RECHARGE_THRESHOLD } from '@libs/creditConstants';

const dynamoClient = new DynamoDBClient({});
const docClient = DynamoDBDocumentClient.from(dynamoClient);

interface UpdateAutoRechargeRequest {
  enabled: boolean;
  threshold?: number;
}

const handler: ValidatedEventAPIGatewayProxyEvent<UpdateAutoRechargeRequest> = async (event: any) => {
  try {
    const userId = event.userId!;
    const { enabled, threshold } = event.body || {};

    if (typeof enabled !== 'boolean') {
      return formatErrorResponse(new Error('enabled (boolean) is required'), 400);
    }
    if (threshold !== undefined && (!Number.isInteger(threshold) || threshold < 1)) {
      return formatErrorResponse(new Error('threshold must be a positive integer'), 400);
    }

    const data = await docClient.send(new GetCommand({
      TableName: process.env.SUBSCRIPTIONS_TABLE!,
      Key: { userId },
    }));
    const billing = data.Item;

    if (enabled && !billing?.stripePaymentMethodId) {
      return formatJSONResponse({
        success: false,
        error: 'NO_PAYMENT_METHOD',
        message: 'Buy a credit pack first so there is a saved card for auto-recharge.',
      }, 400);
    }

    const newThreshold =
      threshold ?? billing?.rechargeThreshold ?? DEFAULT_RECHARGE_THRESHOLD;

    await docClient.send(new UpdateCommand({
      TableName: process.env.SUBSCRIPTIONS_TABLE!,
      Key: { userId },
      UpdateExpression:
        'SET autoRecharge = :enabled, rechargeThreshold = :threshold, updatedAt = :now' +
        (enabled ? ' REMOVE autoRechargeError' : ''),
      ExpressionAttributeValues: {
        ':enabled': enabled,
        ':threshold': newThreshold,
        ':now': new Date().toISOString(),
      },
    }));

    return formatJSONResponse({
      success: true,
      autoRecharge: enabled,
      rechargeThreshold: newThreshold,
    });
  } catch (error) {
    console.error('Error updating auto-recharge:', error);
    return formatErrorResponse(error);
  }
};

export const main = middyfy(handler)
  .use(iamOnlyMiddleware());
```

- [ ] **Step 2: Create `src/functions/billing/getLedger/handler.ts`:**

```ts
import { ValidatedEventAPIGatewayProxyEvent, formatJSONResponse, formatErrorResponse } from '@libs/apiGateway';
import { middyfy } from '@libs/lambda';
import { iamOnlyMiddleware } from '@libs/middleware/dualAuth';
import { listLedgerEntries } from '@libs/services/creditService';

const handler: ValidatedEventAPIGatewayProxyEvent<null> = async (event: any) => {
  try {
    const userId = event.userId!;
    const entries = await listLedgerEntries(userId, 50);
    return formatJSONResponse({
      success: true,
      entries: entries.map((e) => ({
        entryId: e.entryId,
        type: e.type,
        amount: e.amount,
        balanceAfter: e.balanceAfter,
        description: e.description,
        createdAt: e.createdAt,
      })),
    });
  } catch (error) {
    console.error('Error listing ledger:', error);
    return formatErrorResponse(error);
  }
};

export const main = middyfy(handler)
  .use(iamOnlyMiddleware());
```

- [ ] **Step 3: Typecheck + commit**

```bash
npm run typecheck
git add src/functions/billing/
git commit -m "feat(credits): PUT /billing/auto-recharge and GET /billing/ledger handlers"
```

### Task 11: Wire credits gate/debit into the PDF routes

**Files:**
- Modify: `mkpdfs-backend/src/functions/pdf/generate/handler.ts:102-106`
- Modify: `mkpdfs-backend/src/functions/pdf/generateApiKey/handler.ts:19-23`
- Modify: `mkpdfs-backend/src/functions/jobs/submit/handler.ts:5,150-153`

- [ ] **Step 1: `pdf/generate/handler.ts`** — replace the import on line 5 and the middleware chain:

```ts
import { usageTrackingMiddleware } from '@libs/middleware/usageTracking';
import { checkCreditsMiddleware, debitCreditsMiddleware } from '@libs/middleware/credits';
```

```ts
export const main = middyfy(generatePdf)
  .use(dualAuthMiddleware())
  .use(subscriptionMiddleware())
  .use(checkCreditsMiddleware())
  .use(usageTrackingMiddleware({ actionType: 'pdf_generation' }))
  .use(debitCreditsMiddleware());
```

- [ ] **Step 2: `pdf/generateApiKey/handler.ts`** — same swap:

```ts
import { middyfy } from '@libs/lambda';
import { apiKeyOnlyMiddleware } from '@libs/middleware/dualAuth';
import { subscriptionMiddleware } from '@libs/middleware/subscription';
import { usageTrackingMiddleware } from '@libs/middleware/usageTracking';
import { checkCreditsMiddleware, debitCreditsMiddleware } from '@libs/middleware/credits';
import { generatePdf } from '../generate/handler';

export const main = middyfy(generatePdf)
  .use(apiKeyOnlyMiddleware())
  .use(subscriptionMiddleware())
  .use(checkCreditsMiddleware())
  .use(usageTrackingMiddleware({ actionType: 'pdf_generation' }))
  .use(debitCreditsMiddleware());
```

(Keep the explanatory comment block in that file; update its mention of "subscription/limits/usage middleware" to "subscription/credits/usage middleware".)

- [ ] **Step 3: `jobs/submit/handler.ts`** — gate only (debit happens in the processor): replace the `checkLimitsMiddleware` import (line 5) with `checkCreditsMiddleware` and the chain:

```ts
import { checkCreditsMiddleware } from '@libs/middleware/credits';
```

```ts
// Note: No debit here — credits are debited when the job completes (process handler)
export const main = middyfy(submitJob)
  .use(dualAuthMiddleware())
  .use(subscriptionMiddleware())
  .use(checkCreditsMiddleware());
```

- [ ] **Step 4: Typecheck + commit**

```bash
npm run typecheck
git add src/functions/pdf/ src/functions/jobs/submit/
git commit -m "feat(credits): PDF routes gate on credit balance and debit on success"
```

### Task 12: Debit credits in the async job processor

**Files:**
- Modify: `mkpdfs-backend/src/functions/jobs/process/handler.ts`

- [ ] **Step 1: Add imports** (after line 5):

```ts
import { debitCredits, maybeTriggerAutoRecharge, BillingRecord } from '@libs/services/creditService';
```

- [ ] **Step 2: Debit after the success path.** In `processRecord`, right after `await trackUsage(userId, pageCount, result.sizeBytes);` (line 101), add:

```ts
    // Debit credits (job was gated at submit; balance can briefly go negative
    // if it was spent between submit and process — accepted for v1)
    await debitForJob(userId, pageCount);
```

- [ ] **Step 3: Add the helper** (next to `trackUsage`, after line 176):

```ts
const debitForJob = async (userId: string, pageCount: number): Promise<void> => {
  try {
    const billingData = await docClient.send(new GetCommand({
      TableName: process.env.SUBSCRIPTIONS_TABLE,
      Key: { userId }
    }));
    const billing = billingData.Item as BillingRecord | undefined;
    if (!billing || billing.plan === 'enterprise') return;

    const balanceAfter = await debitCredits({
      userId,
      amount: pageCount,
      description: 'pdf_generation_async'
    });
    await maybeTriggerAutoRecharge({ billing, balanceAfter });
  } catch (error) {
    console.error('Failed to debit credits for job (job NOT failed):', error);
  }
};
```

- [ ] **Step 4: Typecheck + commit**

```bash
npm run typecheck
git add src/functions/jobs/process/handler.ts
git commit -m "feat(credits): async job processor debits credits on completion"
```

### Task 13: CDK — ledger table, env vars, routes, grants

**Files:**
- Modify: `mkpdfs-backend/cdk/lib/config.ts:82-96,113-121`
- Modify: `mkpdfs-backend/cdk/lib/stacks/database-stack.ts`
- Modify: `mkpdfs-backend/cdk/lib/service-function.ts:137-146`
- Modify: `mkpdfs-backend/cdk/lib/stacks/api-stack.ts`
- Modify: `mkpdfs-backend/cdk/lib/stacks/jobs-stack.ts`

- [ ] **Step 1: `config.ts`** — add the ledger to `tableNames` and swap the price params in `ssmParamNames`:

```ts
    cliAuth: `${p}-cli-auth`,
    creditLedger: `${p}-credit-ledger`,
```

```ts
export function ssmParamNames(environment: EnvironmentName) {
  const p = `/mkpdfs/${environment}`;
  return {
    stripeSecretKey: `${p}/stripe-secret-key`,
    stripeWebhookSecret: `${p}/stripe-webhook-secret`,
    stripePriceCredits1000: `${p}/stripe-price-credits-1000`,
  };
}
```

- [ ] **Step 2: `database-stack.ts`** — add to the `MkpdfsTables` interface (`creditLedger: dynamodb.Table;`), create the table after `cliAuth` (lines 139–149), and add `creditLedger` to `this.tables`:

```ts
    // Credit ledger: audit trail + webhook idempotency (entryId is
    // `stripe#<paymentIntentId>` for credits, `<ISO>#<uuid>` for debits)
    const creditLedger = new dynamodb.Table(this, 'CreditLedgerTable', {
      ...common,
      tableName: names.creditLedger,
      partitionKey: { name: 'userId', type: dynamodb.AttributeType.STRING },
      sortKey: { name: 'entryId', type: dynamodb.AttributeType.STRING },
    });
```

- [ ] **Step 3: `service-function.ts` `buildCommonEnv`** — replace the two price env vars (lines 139–146) and add the table:

```ts
    CREDIT_LEDGER_TABLE: tables.creditLedger,
```
(goes with the other `*_TABLE` entries around line 128)

```ts
    STRIPE_SECRET_KEY_PARAM: params.stripeSecretKey,
    STRIPE_WEBHOOK_SECRET_PARAM: params.stripeWebhookSecret,
    STRIPE_PRICE_CREDITS_1000: ssm.StringParameter.valueForStringParameter(
      scope,
      params.stripePriceCredits1000,
    ),
```

- [ ] **Step 4: `api-stack.ts`** — grants and routes:

In the PDF section: `generatePdf` and `generatePdfApiKey` now debit credits (subscriptions write is already granted via `grantSubscriptionMw`) and call Stripe for auto-recharge. After line 287 (`grantSes(generatePdf)`) add:

```ts
    tables.creditLedger.grantWriteData(generatePdf); // debit ledger entries
    grantSsmParams(generatePdf, env); // stripe-secret-key for auto-recharge
```

and after line 308 (`grantSes(generatePdfApiKey)`):

```ts
    tables.creditLedger.grantWriteData(generatePdfApiKey);
    grantSsmParams(generatePdfApiKey, env);
```

In the STRIPE section, after line 361 (`tables.subscriptions.grantReadWriteData(stripeWebhook)`):

```ts
    tables.creditLedger.grantWriteData(stripeWebhook); // idempotent credit txn
```

New BILLING section (after the Stripe block, before MARKETPLACE):

```ts
    // =================================================================
    // BILLING (credits)
    // =================================================================
    const billingUpdateAutoRecharge = makeFn('BillingUpdateAutoRechargeFn', {
      entry: 'src/functions/billing/updateAutoRecharge/handler.ts',
    });
    tables.subscriptions.grantReadWriteData(billingUpdateAutoRecharge);
    addRoute('/billing/auto-recharge', 'PUT', billingUpdateAutoRecharge, true);

    const billingGetLedger = makeFn('BillingGetLedgerFn', {
      entry: 'src/functions/billing/getLedger/handler.ts',
    });
    tables.creditLedger.grantReadData(billingGetLedger);
    addRoute('/billing/ledger', 'GET', billingGetLedger, true);
```

- [ ] **Step 5: `jobs-stack.ts`** — the processor debits + may trigger recharge. Add `grantSsmParams` to the import from `../service-function` and, next to the existing `processJobFn` grants (lines 125–129):

```ts
    tables.subscriptions.grantReadWriteData(this.processJobFn); // credit debit
    tables.creditLedger.grantWriteData(this.processJobFn);
    grantSsmParams(this.processJobFn, cfg.environment); // stripe key for auto-recharge
```

(Use the same `cfg` variable name the file already uses; if it destructures `props` differently, match it.)

- [ ] **Step 6: Synth to verify both envs:**

```bash
npm run typecheck && npm run cdk:synth && npm run cdk:synth:prod
```
Expected: both synths succeed. (Prod synth resolves prod SSM at deploy time only, so a missing prod param does NOT fail synth — that's why Phase 5 creates it before deploying.)

- [ ] **Step 7: Commit**

```bash
git add cdk/
git commit -m "feat(credits): credit-ledger table, billing routes, debit/recharge grants, single price param"
```

### Task 14: Backend gate — full verification

- [ ] **Step 1:** `npm test` — Expected: all suites pass (codes, creditService, credits middleware).
- [ ] **Step 2:** `npm run typecheck` — Expected: clean.
- [ ] **Step 3:** `grep -rn "pagesPerMonth\|PRICE_TO_PLAN\|PLAN_TO_PRICE\|checkLimitsMiddleware\|STRIPE_PRICE_BASIC\|STRIPE_PRICE_PROFESSIONAL" src/ cdk/lib/` — Expected: no hits. Fix any stragglers.
- [ ] **Step 4:** Commit anything outstanding; push:

```bash
git push origin dev
```
This triggers the CI deploy to dev (Phase 2 verifies it). Then bump the submodule in the orchestrator repo:

```bash
cd .. && git add mkpdfs-backend && git commit -m "Bump mkpdfs-backend: credits billing backend"
```

---

## Phase 2 — Deploy & verify backend in dev

### Task 15: Dev deploy + smoke

- [ ] **Step 1: Watch CI** (`gh run watch --repo <backend repo>` or deploy locally with `npm run cdk:deploy:dev`). **Check the END of the log** — the SecureString/param trap aborts with exit 0.
- [ ] **Step 2: Verify the new table exists:**

```bash
aws dynamodb describe-table --profile rocketeast --table-name mkpdfs-dev-credit-ledger --query Table.KeySchema
```
Expected: `userId` HASH, `entryId` RANGE.

- [ ] **Step 3: Smoke the 402 gate.** With a dev API token whose owner has 0 credits:

```bash
curl -s -o /dev/null -w "%{http_code}" -X POST https://dev.apis.mkpdfs.com/v1/pdf/generate \
  -H "x-api-key: tlfy_..." -H "Content-Type: application/json" \
  -d '{"templateId":"<existing>","data":{}}'
```
Expected: `402`. If the dev table still has a row for this user with an old plan and no `creditBalance`, that's correct behavior (treated as 0).

- [ ] **Step 4: Tail webhook logs while doing a test checkout from Stripe dashboard later (Phase 4); for now confirm the lambda updated:**

```bash
aws lambda get-function-configuration --profile rocketeast \
  --function-name $(aws lambda list-functions --profile rocketeast --query "Functions[?contains(FunctionName,'StripeWebhookFn')].FunctionName | [0]" --output text) \
  --query "Environment.Variables.CREDIT_LEDGER_TABLE"
```
Expected: `"mkpdfs-dev-credit-ledger"`.

---

## Phase 3 — Frontend (`mkpdfs-web/`, branch `dev`)

### Task 16: Types + API client

**Files:**
- Modify: `mkpdfs-web/src/types/index.ts:13-19,84-93`
- Modify: `mkpdfs-web/src/lib/api.ts` (Stripe/Billing section)

- [ ] **Step 1: types** — replace `SubscriptionLimits` (13–19), `SubscriptionPlan`/`Subscription` (84–93) and add `LedgerEntry`:

```ts
export interface SubscriptionLimits {
  templatesAllowed: number
  apiTokensAllowed: number
  maxPdfSizeMB: number
  aiGenerationsPerMonth: number
}

export type SubscriptionPlan = 'credits' | 'enterprise'

export interface Subscription {
  plan: SubscriptionPlan
  status: 'active' | 'cancelled' | 'past_due'
  creditBalance?: number
  autoRecharge?: boolean
  rechargeThreshold?: number
  autoRechargeError?: string
  stripeCustomerId?: string
  stripePaymentMethodId?: string
}

export interface LedgerEntry {
  entryId: string
  type: 'purchase' | 'auto_recharge' | 'debit'
  amount: number
  balanceAfter?: number
  description?: string
  createdAt: string
}
```

- [ ] **Step 2: api.ts** — in the `Stripe / Billing` section, make `createCheckoutSession` parameterless and add the two new calls:

```ts
export async function createCheckoutSession(): Promise<{ url: string; sessionId: string }> {
  const response = await authFetch<{ success: boolean; url: string; sessionId: string }>(
    '/stripe/create-checkout-session',
    { method: 'POST', body: JSON.stringify({}) }
  )
  return { url: response.url, sessionId: response.sessionId }
}

export async function updateAutoRecharge(
  enabled: boolean,
  threshold?: number
): Promise<{ success: boolean; autoRecharge: boolean; rechargeThreshold: number }> {
  return authFetch('/billing/auto-recharge', {
    method: 'PUT',
    body: JSON.stringify({ enabled, threshold }),
  })
}

export async function getCreditLedger(): Promise<{ success: boolean; entries: import('@/types').LedgerEntry[] }> {
  return authFetch('/billing/ledger', { method: 'GET' })
}
```

- [ ] **Step 3: Build check** — `npm run build` (or `npx tsc --noEmit`). Expected: errors only in `billing/page.tsx` and any pagesPerMonth consumers — next tasks.

### Task 17: i18n — credits strings (en + es)

**Files:**
- Modify: `mkpdfs-web/src/i18n/messages/en.json`
- Modify: `mkpdfs-web/src/i18n/messages/es.json`

- [ ] **Step 1: en.json** — replace `landing.pricing` with:

```json
"pricing": {
  "title": "Simple, prepaid pricing",
  "subtitle": "No subscriptions. Buy credits, use them whenever. 1 credit = 1 PDF page.",
  "mostPopular": "Most Popular",
  "contactSales": "Contact Sales",
  "credits": {
    "name": "PDF Credits",
    "price": "$10",
    "unit": "per 1,000 PDFs",
    "description": "Pay once, generate whenever",
    "features": {
      "pages": "1,000 PDF pages per pack",
      "expiry": "Credits never expire",
      "templates": "500 templates",
      "keys": "10 API keys",
      "fileSize": "50MB max file size",
      "ai": "15 AI template generations / month",
      "autoRecharge": "Optional auto-recharge"
    }
  },
  "enterprise": {
    "name": "Enterprise",
    "description": "Unlimited everything",
    "features": {
      "pages": "Unlimited pages",
      "templates": "Unlimited templates",
      "keys": "Unlimited API keys",
      "fileSize": "100MB max file size",
      "support": "Dedicated support",
      "integrations": "Custom integrations",
      "sla": "SLA guarantee"
    }
  }
}
```

and replace `billing` with:

```json
"billing": {
  "title": "Billing",
  "subtitle": "Buy PDF credits and manage auto-recharge.",
  "balance": {
    "title": "Credit Balance",
    "credits": "{count} credits",
    "hint": "1 credit = 1 PDF page. Credits never expire.",
    "buy": "Buy 1,000 PDFs — $10",
    "enterprise": "Enterprise — unlimited",
    "updateCard": "Update card"
  },
  "autoRecharge": {
    "title": "Auto-recharge",
    "description": "Automatically buy 1,000 more credits ($10) when your balance drops below the threshold.",
    "threshold": "Recharge when balance falls below",
    "enable": "Enable auto-recharge",
    "disable": "Disable auto-recharge",
    "needsPurchase": "Buy a credit pack first so we have a card on file.",
    "failedBanner": "Your last auto-recharge failed: {reason}. Update your card and re-enable auto-recharge.",
    "saved": "Auto-recharge settings saved"
  },
  "history": {
    "title": "History",
    "empty": "No transactions yet",
    "purchase": "Credit pack purchase",
    "auto_recharge": "Auto-recharge",
    "debit": "PDF generation",
    "balanceAfter": "Balance: {count}"
  }
}
```

- [ ] **Step 2: es.json** — same structure in Spanish:

```json
"pricing": {
  "title": "Precios simples, prepagados",
  "subtitle": "Sin suscripciones. Compra créditos y úsalos cuando quieras. 1 crédito = 1 página PDF.",
  "mostPopular": "Más Popular",
  "contactSales": "Contactar Ventas",
  "credits": {
    "name": "Créditos PDF",
    "price": "$10",
    "unit": "por 1,000 PDFs",
    "description": "Paga una vez, genera cuando quieras",
    "features": {
      "pages": "1,000 páginas PDF por paquete",
      "expiry": "Los créditos nunca expiran",
      "templates": "500 plantillas",
      "keys": "10 llaves API",
      "fileSize": "Archivos de hasta 50MB",
      "ai": "15 generaciones de plantillas con IA / mes",
      "autoRecharge": "Auto-recarga opcional"
    }
  },
  "enterprise": {
    "name": "Enterprise",
    "description": "Todo ilimitado",
    "features": {
      "pages": "Páginas ilimitadas",
      "templates": "Plantillas ilimitadas",
      "keys": "Llaves API ilimitadas",
      "fileSize": "Archivos de hasta 100MB",
      "support": "Soporte dedicado",
      "integrations": "Integraciones a medida",
      "sla": "Garantía SLA"
    }
  }
}
```

```json
"billing": {
  "title": "Facturación",
  "subtitle": "Compra créditos PDF y gestiona la auto-recarga.",
  "balance": {
    "title": "Saldo de Créditos",
    "credits": "{count} créditos",
    "hint": "1 crédito = 1 página PDF. Los créditos nunca expiran.",
    "buy": "Comprar 1,000 PDFs — $10",
    "enterprise": "Enterprise — ilimitado",
    "updateCard": "Actualizar tarjeta"
  },
  "autoRecharge": {
    "title": "Auto-recarga",
    "description": "Compra automáticamente 1,000 créditos más ($10) cuando tu saldo baje del umbral.",
    "threshold": "Recargar cuando el saldo baje de",
    "enable": "Activar auto-recarga",
    "disable": "Desactivar auto-recarga",
    "needsPurchase": "Primero compra un paquete de créditos para tener una tarjeta guardada.",
    "failedBanner": "Tu última auto-recarga falló: {reason}. Actualiza tu tarjeta y reactiva la auto-recarga.",
    "saved": "Configuración de auto-recarga guardada"
  },
  "history": {
    "title": "Historial",
    "empty": "Aún no hay transacciones",
    "purchase": "Compra de paquete de créditos",
    "auto_recharge": "Auto-recarga",
    "debit": "Generación de PDF",
    "balanceAfter": "Saldo: {count}"
  }
}
```

If there are more locale files in `src/i18n/messages/`, mirror the structure (machine-translate consistently). Keys used elsewhere that vanished (`landing.pricing.free.*`, `starter.*`, `professional.*`, `billing.currentPlan.*`, `billing.plans.*`, `billing.invoices.*`, `billing.paymentMethod.*`) will surface as build/runtime errors in Tasks 18–20 — that's expected; those consumers get rewritten.

### Task 18: Billing page rewrite

**Files:**
- Rewrite: `mkpdfs-web/src/app/[locale]/(dashboard)/billing/page.tsx`

- [ ] **Step 1: Replace the page** with the credits UI (keeps the post-checkout polling pattern):

```tsx
'use client'

import { useState, useEffect } from 'react'
import { Card, CardHeader, CardTitle, CardContent } from '@/components/ui'
import { Button } from '@/components/ui'
import { CreditCard, Zap, Loader2, ExternalLink, AlertTriangle, History } from 'lucide-react'
import { useProfile } from '@/hooks/useApi'
import { createCheckoutSession, createPortalSession, updateAutoRecharge, getCreditLedger } from '@/lib/api'
import type { LedgerEntry } from '@/types'
import { useTranslations } from 'next-intl'
import { useQueryClient, useQuery } from '@tanstack/react-query'

export default function BillingPage() {
  const queryClient = useQueryClient()
  const { data: profile, isLoading, refetch } = useProfile()
  const [loadingBuy, setLoadingBuy] = useState(false)
  const [loadingPortal, setLoadingPortal] = useState(false)
  const [savingAutoRecharge, setSavingAutoRecharge] = useState(false)
  const [threshold, setThreshold] = useState<number | null>(null)
  const t = useTranslations('billing')
  const errors = useTranslations('errors')

  const { data: ledger } = useQuery({
    queryKey: ['creditLedger'],
    queryFn: getCreditLedger,
  })

  const sub = profile?.subscription
  const isEnterprise = sub?.plan === 'enterprise'
  const balance = sub?.creditBalance ?? 0
  const autoRechargeOn = !!sub?.autoRecharge
  const effectiveThreshold = threshold ?? sub?.rechargeThreshold ?? 100
  const hasCard = !!sub?.stripePaymentMethodId

  // Poll after returning from Stripe checkout so the webhook-credited
  // balance shows up without a manual refresh
  useEffect(() => {
    queryClient.invalidateQueries({ queryKey: ['profile'] })
    let pollCount = 0
    const interval = setInterval(() => {
      pollCount++
      refetch()
      queryClient.invalidateQueries({ queryKey: ['creditLedger'] })
      if (pollCount >= 6) clearInterval(interval)
    }, 5000)
    return () => clearInterval(interval)
  }, [queryClient, refetch])

  const handleBuy = async () => {
    try {
      setLoadingBuy(true)
      const { url } = await createCheckoutSession()
      window.location.href = url
    } catch (error) {
      console.error('Failed to create checkout session:', error)
      alert(errors('generic'))
      setLoadingBuy(false)
    }
  }

  const handlePortal = async () => {
    try {
      setLoadingPortal(true)
      const { url } = await createPortalSession()
      window.location.href = url
    } catch (error) {
      console.error('Failed to create portal session:', error)
      alert(errors('generic'))
      setLoadingPortal(false)
    }
  }

  const handleAutoRechargeToggle = async (enabled: boolean) => {
    try {
      setSavingAutoRecharge(true)
      await updateAutoRecharge(enabled, effectiveThreshold)
      await refetch()
    } catch (error) {
      console.error('Failed to update auto-recharge:', error)
      alert(errors('generic'))
    } finally {
      setSavingAutoRecharge(false)
    }
  }

  if (isLoading) {
    return (
      <div className="flex items-center justify-center py-12">
        <Loader2 className="h-8 w-8 animate-spin text-primary" />
      </div>
    )
  }

  return (
    <div className="space-y-6">
      <div>
        <h1 className="text-2xl font-bold text-foreground-dark">{t('title')}</h1>
        <p className="mt-1 text-sm text-foreground-light">{t('subtitle')}</p>
      </div>

      {sub?.autoRechargeError && (
        <div className="flex items-start gap-3 rounded-md border border-amber-300 bg-amber-50 p-4 text-sm text-amber-900">
          <AlertTriangle className="mt-0.5 h-5 w-5 shrink-0" />
          <span>{t('autoRecharge.failedBanner', { reason: sub.autoRechargeError })}</span>
        </div>
      )}

      {/* Balance */}
      <Card>
        <CardHeader>
          <CardTitle className="flex items-center gap-2 text-lg">
            <CreditCard className="h-5 w-5" />
            {t('balance.title')}
          </CardTitle>
        </CardHeader>
        <CardContent>
          <div className="flex items-center justify-between">
            <div>
              <p className="text-4xl font-bold text-foreground-dark">
                {isEnterprise ? t('balance.enterprise') : t('balance.credits', { count: balance })}
              </p>
              {!isEnterprise && (
                <p className="mt-1 text-sm text-foreground-light">{t('balance.hint')}</p>
              )}
            </div>
            {!isEnterprise && (
              <div className="flex gap-2">
                {hasCard && (
                  <Button variant="outline" onClick={handlePortal} disabled={loadingPortal}>
                    {loadingPortal ? (
                      <Loader2 className="mr-2 h-4 w-4 animate-spin" />
                    ) : (
                      <ExternalLink className="mr-2 h-4 w-4" />
                    )}
                    {t('balance.updateCard')}
                  </Button>
                )}
                <Button onClick={handleBuy} disabled={loadingBuy}>
                  {loadingBuy ? (
                    <Loader2 className="mr-2 h-4 w-4 animate-spin" />
                  ) : (
                    <Zap className="mr-2 h-4 w-4" />
                  )}
                  {t('balance.buy')}
                </Button>
              </div>
            )}
          </div>
        </CardContent>
      </Card>

      {/* Auto-recharge */}
      {!isEnterprise && (
        <Card>
          <CardHeader>
            <CardTitle className="text-lg">{t('autoRecharge.title')}</CardTitle>
          </CardHeader>
          <CardContent className="space-y-4">
            <p className="text-sm text-foreground-light">{t('autoRecharge.description')}</p>
            {!hasCard ? (
              <p className="text-sm text-foreground-light">{t('autoRecharge.needsPurchase')}</p>
            ) : (
              <div className="flex flex-wrap items-center gap-4">
                <label className="flex items-center gap-2 text-sm text-foreground-dark">
                  {t('autoRecharge.threshold')}
                  <input
                    type="number"
                    min={1}
                    className="w-24 rounded-md border border-input bg-background px-2 py-1"
                    value={effectiveThreshold}
                    disabled={autoRechargeOn}
                    onChange={(e) => setThreshold(Math.max(1, parseInt(e.target.value || '1', 10)))}
                  />
                </label>
                <Button
                  variant={autoRechargeOn ? 'outline' : 'default'}
                  onClick={() => handleAutoRechargeToggle(!autoRechargeOn)}
                  disabled={savingAutoRecharge}
                >
                  {savingAutoRecharge && <Loader2 className="mr-2 h-4 w-4 animate-spin" />}
                  {autoRechargeOn ? t('autoRecharge.disable') : t('autoRecharge.enable')}
                </Button>
              </div>
            )}
          </CardContent>
        </Card>
      )}

      {/* History */}
      <Card>
        <CardHeader>
          <CardTitle className="flex items-center gap-2 text-lg">
            <History className="h-5 w-5" />
            {t('history.title')}
          </CardTitle>
        </CardHeader>
        <CardContent>
          {!ledger?.entries?.length ? (
            <p className="text-sm text-foreground-light">{t('history.empty')}</p>
          ) : (
            <ul className="divide-y divide-border">
              {ledger.entries.map((entry: LedgerEntry) => (
                <li key={entry.entryId} className="flex items-center justify-between py-2 text-sm">
                  <div>
                    <p className="text-foreground-dark">{t(`history.${entry.type}`)}</p>
                    <p className="text-xs text-foreground-light">
                      {new Date(entry.createdAt).toLocaleString()}
                    </p>
                  </div>
                  <div className="text-right">
                    <p className={entry.amount > 0 ? 'font-medium text-success' : 'text-foreground-dark'}>
                      {entry.amount > 0 ? `+${entry.amount}` : entry.amount}
                    </p>
                    {entry.balanceAfter !== undefined && (
                      <p className="text-xs text-foreground-light">
                        {t('history.balanceAfter', { count: entry.balanceAfter })}
                      </p>
                    )}
                  </div>
                </li>
              ))}
            </ul>
          )}
        </CardContent>
      </Card>
    </div>
  )
}
```

If `Button`/`Card` are exported from one barrel (`@/components/ui`), merge the two imports at the top into one (match the old file's import line).

- [ ] **Step 2: Build** — `npm run build`. Fix import/UI-primitive mismatches against the actual `@/components/ui` exports (the old page is the reference).

- [ ] **Step 3: Commit**

```bash
git add src/types/index.ts src/lib/api.ts src/i18n/messages/ src/app/
git commit -m "feat(credits): billing page — balance, buy pack, auto-recharge, ledger history"
```

### Task 19: Landing pricing — two cards

**Files:**
- Modify: `mkpdfs-web/src/app/[locale]/page.tsx` (pricing section, ~lines 60–113)

- [ ] **Step 1:** Open the pricing section. Replace the 4-entry plans array with 2 entries using the new keys, keeping the existing card markup/styling:

```tsx
const plans = [
  {
    id: 'credits',
    name: pricing('credits.name'),
    price: pricing('credits.price'),
    unit: pricing('credits.unit'),
    description: pricing('credits.description'),
    features: [
      pricing('credits.features.pages'),
      pricing('credits.features.expiry'),
      pricing('credits.features.templates'),
      pricing('credits.features.keys'),
      pricing('credits.features.fileSize'),
      pricing('credits.features.ai'),
      pricing('credits.features.autoRecharge'),
    ],
    popular: true,
  },
  {
    id: 'enterprise',
    name: pricing('enterprise.name'),
    price: pricing('contactSales'),
    unit: '',
    description: pricing('enterprise.description'),
    features: [
      pricing('enterprise.features.pages'),
      pricing('enterprise.features.templates'),
      pricing('enterprise.features.keys'),
      pricing('enterprise.features.fileSize'),
      pricing('enterprise.features.support'),
      pricing('enterprise.features.integrations'),
      pricing('enterprise.features.sla'),
    ],
  },
]
```

Where the card rendered `{common('perMonth')}` next to the price, render `{plan.unit}` instead. Adjust the grid class from 4 columns to 2 (e.g. `md:grid-cols-2 max-w-3xl mx-auto`). Remove now-unused references.

- [ ] **Step 2: Sweep for leftover plan references:**

Run: `grep -rn "pricing('free\|pricing('starter\|pricing('professional\|perMonth\|pagesPerMonth\|pagesLimit" src/`
Update every hit: usage widgets that showed "X / Y pages this month" now show `creditBalance` from `profile.subscription.creditBalance` ("{n} credits remaining") with the monthly `pdfCount` kept as a plain stat where useful. Components referencing removed `Subscription` fields (`stripeSubscriptionId`, `stripePriceId`, `currentPeriodEnd`) drop them.

- [ ] **Step 3: Build + lint** — `npm run build`. Expected: clean.

- [ ] **Step 4: Commit + push + bump**

```bash
git add -A && git commit -m "feat(credits): landing pricing and dashboard widgets for credit model"
git push origin dev
cd .. && git add mkpdfs-web && git commit -m "Bump mkpdfs-web: credits billing UI"
```
Amplify deploys dev.mkpdfs.com from the `dev` branch automatically.

---

## Phase 4 — E2E verification in dev (Stripe test mode)

### Task 20: End-to-end flows

Use the `e2e-browser` skill against `https://dev.mkpdfs.com` with a real test login. Stripe test cards: success `4242 4242 4242 4242`, attaches-but-charges-fail `4000 0000 0000 0341`. Tail logs in parallel:

```bash
aws logs tail /aws/lambda/<StripeWebhookFn-name> --follow --profile rocketeast
```

- [ ] **Flow 1 — Purchase:** Billing → "Buy 1,000 PDFs — $10" → Stripe Checkout with 4242 card → redirected back → within the 30s polling window the balance shows **1,000 credits** and History shows a `purchase +1000`. Verify in DDB: `aws dynamodb get-item --profile rocketeast --table-name mkpdfs-dev-subscriptions --key '{"userId":{"S":"<id>"}}'` → `creditBalance: 1000`, `stripePaymentMethodId` present.
- [ ] **Flow 2 — Webhook idempotency:** Stripe dashboard → the webhook delivery → "Resend". Balance must stay 1,000 (log shows `already credited — skipping`).
- [ ] **Flow 3 — Generate & debit:** Generate a 1-page PDF from the dashboard → balance 999, ledger `debit -1, balance 999`.
- [ ] **Flow 4 — 402:** With a second user (0 credits), `POST /v1/pdf/generate` with their API token → 402 `INSUFFICIENT_CREDITS`.
- [ ] **Flow 5 — Auto-recharge success:** Enable auto-recharge with threshold 1000 (any debit crosses it). Generate 1 page → balance 998... then webhook lands → balance jumps +1000, ledger shows `auto_recharge +1000`, `rechargeInProgress` cleared. Reset threshold to something sane (e.g. 100) afterwards.
- [ ] **Flow 6 — Auto-recharge failure:** "Update card" → portal → set default card to `4000 0000 0000 0341` → enable auto-recharge (threshold 1000+) → generate 1 page → off-session charge fails → billing page shows the amber failure banner and auto-recharge is OFF in DDB (`autoRecharge: false`, `autoRechargeError` set). Re-set card to 4242 afterwards.
- [ ] **Flow 7 — Async jobs:** `POST /jobs/submit` with 2 pages → on completion balance dropped by 2 and ledger has `pdf_generation_async`.
- [ ] **Flow 8 — AI gate:** With the 0-credit user, AI template generation returns 403 `INSUFFICIENT_CREDITS`; with the funded user it works.
- [ ] **Fix-forward:** Any failure: diagnose with `aws logs tail`, fix in the matching task's file, commit, redeploy dev, re-run the flow.

---

## Phase 5 — Production cutover

### Task 21: Stripe live mode + prod SSM

- [ ] **Step 1:** Stripe **live mode**: create the same Product/Price ($10 / 1,000 credits). Copy `price_...`.
- [ ] **Step 2:**

```bash
aws ssm put-parameter --profile rocketeast --region us-east-1 \
  --name /mkpdfs/prod/stripe-price-credits-1000 \
  --type String --value "price_LIVE_XXXX"
```
- [ ] **Step 3:** Live webhook endpoint `https://apis.mkpdfs.com/stripe/webhook`: events `checkout.session.completed`, `payment_intent.succeeded`, `payment_intent.payment_failed`; remove subscription events. Live Customer Portal: payment-methods only.
- [ ] **Step 4:** Verify: `aws ssm get-parameter --profile rocketeast --name /mkpdfs/prod/stripe-price-credits-1000 --query Parameter.Type` → `"String"`.

### Task 22: Merge dev → main (both submodules + orchestrator)

- [ ] **Step 1:** In `mkpdfs-backend/`: `git checkout main && git merge dev && git push origin main` → CI deploys prod CDK. **Watch the end of the log** for the silent param abort.
- [ ] **Step 2:** In `mkpdfs-web/`: same merge/push → Amplify deploys mkpdfs.com.
- [ ] **Step 3:** Orchestrator: merge `dev` → `main` (carries the submodule bumps + spec + this plan), push.
- [ ] **Step 4:** Verify prod table: `aws dynamodb describe-table --profile rocketeast --table-name mkpdfs-prod-credit-ledger --query Table.KeySchema` → userId/entryId. RemovalPolicy is RETAIN in prod (inherited from `common`).

### Task 23: Prod smoke + cleanup + docs

- [ ] **Step 1 — Smoke (real $10 charge, owner's card):** buy a pack on mkpdfs.com, verify balance 1,000 and ledger entry, generate 1 PDF → 999. Optionally refund the charge in the Stripe dashboard (credits stay — it's the owner's account; note the ledger intentionally does NOT reverse refunds in v1).
- [ ] **Step 2 — 402 gate in prod:** fresh/0-credit account → `POST /v1/pdf/generate` → 402.
- [ ] **Step 3 — Enterprise unaffected:** the Academia Connects token still generates with no debit (check `mkpdfs-prod-credit-ledger` has no entries for that userId and the subscriptions row still says `plan: enterprise`).
- [ ] **Step 4 — Archive the old monthly Products** in Stripe (test AND live modes).
- [ ] **Step 5 — Delete the dead SSM params** (both envs, AFTER both deploys are green):

```bash
for env in dev prod; do
  aws ssm delete-parameter --profile rocketeast --name /mkpdfs/$env/stripe-price-basic
  aws ssm delete-parameter --profile rocketeast --name /mkpdfs/$env/stripe-price-professional
done
```
- [ ] **Step 6 — Docs:** update root `CLAUDE.md` (billing model: credits $10=1,000, ledger table now makes it 10 DynamoDB tables, new `/billing/*` routes, new SSM param name) and `mkpdfs-backend/CLAUDE.md` (subscription-tiers sections → credits model, middleware chain example). Commit in the respective repos.

---

## Acceptance criteria (whole feature)

1. A user with 0 credits gets 402 on every PDF path (sync, API-key, async submit) and 403 on AI.
2. $10 checkout credits exactly 1,000, exactly once, even with duplicate webhook deliveries.
3. Auto-recharge: opt-in, threshold-driven, single in-flight charge under concurrency, disables itself + shows a banner on card failure.
4. Enterprise generates without debits; nothing else about its flow changed.
5. Monthly usage stats still accumulate (dashboard), but never block.
6. `npm test` + `npm run typecheck` green in backend; `npm run build` green in web; both envs deployed and e2e-verified.
