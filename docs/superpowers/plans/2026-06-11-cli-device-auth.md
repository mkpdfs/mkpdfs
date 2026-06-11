# CLI Device-Flow Auth Implementation Plan (Plan A)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Branded device-flow authentication for the upcoming `mkp` CLI: 3 backend endpoints + a `/cli/authorize` approval page in mkpdfs-web, verified end-to-end in dev.

**Architecture:** OAuth-device-flow-shaped, custom implementation. The CLI requests a `deviceCode`/`userCode` pair from the backend, the user approves on a branded Next.js page (logged in via Amplify), the approval page hands the session's Cognito tokens to the backend (stored ≤10 min, deleted on first read), and the CLI polls until it receives them. **Decision recorded here (spec open item #1):** the CUSTOM_AUTH path is dropped — Google-federated users have Cognito status `EXTERNAL_PROVIDER` and cannot authenticate via `AdminInitiateAuth`, so the token-handover fallback is the only path that works uniformly. No Cognito triggers are added.

**Tech Stack:** mkpdfs-backend (TypeScript Lambda + middy + CDK, vitest added for unit tests), mkpdfs-web (Next.js 14 app router, Amplify v6, next-intl, Tailwind + CVA).

**Spec:** `docs/superpowers/specs/2026-06-11-mkpdfs-cli-design.md`

**Working directories:** `mkpdfs-backend/` and `mkpdfs-web/` submodules (each commits to its own repo, branch `dev`).

---

## Shared reference: repo patterns to copy

- Canonical handler: `src/functions/user/createToken/handler.ts` (middyfy + `formatJSONResponse`/`formatErrorResponse` from `@libs/apiGateway`, `iamOnlyMiddleware` from `@libs/middleware/dualAuth`).
- CDK wiring: `cdk/lib/stacks/api-stack.ts` — `makeFn(...)` + `addRoute(path, method, fn, cognitoAuth)`.
- Tables: `cdk/lib/stacks/database-stack.ts` + `tableNames()` in `cdk/lib/config.ts` + env injection in `buildCommonEnv()` (`cdk/lib/service-function.ts`).
- IP rate limiting: `src/functions/contact/enterpriseContact/handler.ts` (rateLimits table, `pk`/`sk`, `ttl`).

DynamoDB item shape for the new `cli-auth` table:

```
{
  deviceCode: "64-hex",          // PK — secret, only the CLI knows it
  userCode:   "KXTM49PF",        // GSI — shown to the human (rendered "KXTM-49PF")
  status:     "pending" | "approved" | "denied",
  createdAt:  1760000000000,
  ttl:        1760000600,        // 10 min from creation (epoch seconds)
  lastPolledAt: 1760000000000,   // server-side interval enforcement
  userId?:    "us-east-1:...",   // set on approval
  tokens?:    { idToken, accessToken, refreshToken }   // set on approval, deleted on first read
}
```

---

### Task 1: `cli-auth` DynamoDB table

**Files:**
- Modify: `mkpdfs-backend/cdk/lib/config.ts` (tableNames)
- Modify: `mkpdfs-backend/cdk/lib/stacks/database-stack.ts`
- Modify: `mkpdfs-backend/cdk/lib/service-function.ts` (buildCommonEnv)
- Modify: `mkpdfs-backend/cdk/lib/stacks/api-stack.ts` (tables interface, if tables are passed via a typed object — follow how `rateLimits` flows through)

- [ ] **Step 1: Add the table name**

In `cdk/lib/config.ts`, inside `tableNames()`:

```typescript
    rateLimits: `${p}-rate-limits`,
    aiJobs: `${p}-ai-jobs`,
    cliAuth: `${p}-cli-auth`,
```

- [ ] **Step 2: Define the table**

In `cdk/lib/stacks/database-stack.ts`, after the `rateLimits` table, following the same `common` spread:

```typescript
    const cliAuth = new dynamodb.Table(this, 'CliAuthTable', {
      ...common,
      tableName: names.cliAuth,
      partitionKey: { name: 'deviceCode', type: dynamodb.AttributeType.STRING },
      timeToLiveAttribute: 'ttl',
    });
    cliAuth.addGlobalSecondaryIndex({
      indexName: 'userCode-index',
      partitionKey: { name: 'userCode', type: dynamodb.AttributeType.STRING },
      projectionType: dynamodb.ProjectionType.ALL,
    });
```

Export it the same way the other tables are exported from the stack (add to the stack's public tables property / interface — mirror `rateLimits` exactly).

- [ ] **Step 3: Inject the env var**

In `cdk/lib/service-function.ts` `buildCommonEnv()`, next to `RATE_LIMIT_TABLE`:

```typescript
  CLI_AUTH_TABLE: names.cliAuth,
```

- [ ] **Step 4: Typecheck and diff**

Run: `cd mkpdfs-backend && npm run typecheck && npm run cdk:diff`
Expected: typecheck clean; diff shows only the new table (+ env var on functions).

- [ ] **Step 5: Commit**

```bash
git add cdk/lib/config.ts cdk/lib/stacks/database-stack.ts cdk/lib/service-function.ts cdk/lib/stacks/api-stack.ts
git commit -m "feat(cli-auth): add cli-auth DynamoDB table (device flow state)"
```

---

### Task 2: vitest setup + code generators (TDD)

The backend has no test framework. Add vitest minimally — it must not touch the deploy path.

**Files:**
- Modify: `mkpdfs-backend/package.json`
- Create: `mkpdfs-backend/src/functions/auth/cli/codes.ts`
- Test: `mkpdfs-backend/src/functions/auth/cli/codes.test.ts`

- [ ] **Step 1: Install vitest and wire the script**

Run: `cd mkpdfs-backend && npm install -D vitest`

In `package.json` replace `"test": "echo \"No tests configured\""` with:

```json
    "test": "vitest run",
```

- [ ] **Step 2: Write the failing test**

`src/functions/auth/cli/codes.test.ts`:

```typescript
import { describe, it, expect } from 'vitest';
import { generateUserCode, generateDeviceCode, formatUserCode, normalizeUserCode } from './codes';

describe('generateUserCode', () => {
  it('produces 8 chars from the unambiguous alphabet', () => {
    for (let i = 0; i < 100; i++) {
      const code = generateUserCode();
      expect(code).toMatch(/^[ABCDEFGHJKLMNPQRSTUVWXYZ23456789]{8}$/);
    }
  });
  it('produces distinct codes', () => {
    expect(new Set(Array.from({ length: 50 }, generateUserCode)).size).toBe(50);
  });
});

describe('generateDeviceCode', () => {
  it('produces 64 hex chars (256 bits)', () => {
    expect(generateDeviceCode()).toMatch(/^[0-9a-f]{64}$/);
  });
});

describe('user code formatting', () => {
  it('formats as XXXX-XXXX for display', () => {
    expect(formatUserCode('KXTM49PF')).toBe('KXTM-49PF');
  });
  it('normalizes user input (case, dashes, spaces)', () => {
    expect(normalizeUserCode(' kxtm-49pf ')).toBe('KXTM49PF');
  });
});
```

- [ ] **Step 3: Run test to verify it fails**

Run: `npx vitest run src/functions/auth/cli/codes.test.ts`
Expected: FAIL — cannot resolve `./codes`.

- [ ] **Step 4: Implement**

`src/functions/auth/cli/codes.ts`:

```typescript
import { randomBytes, randomInt } from 'crypto';

// No 0/O/1/I — unambiguous for humans reading a terminal. 32^8 ≈ 1.1e12 (~40 bits).
const ALPHABET = 'ABCDEFGHJKLMNPQRSTUVWXYZ23456789';

export const generateUserCode = (): string =>
  Array.from({ length: 8 }, () => ALPHABET[randomInt(ALPHABET.length)]).join('');

export const generateDeviceCode = (): string => randomBytes(32).toString('hex');

export const formatUserCode = (code: string): string =>
  `${code.slice(0, 4)}-${code.slice(4)}`;

export const normalizeUserCode = (input: string): string =>
  input.toUpperCase().replace(/[^A-Z0-9]/g, '');
```

- [ ] **Step 5: Run test to verify it passes**

Run: `npx vitest run src/functions/auth/cli/codes.test.ts`
Expected: PASS (6 tests).

- [ ] **Step 6: Commit**

```bash
git add package.json package-lock.json src/functions/auth/cli/
git commit -m "feat(cli-auth): code generators with vitest setup"
```

---

### Task 3: `POST /auth/cli/device` handler

**Files:**
- Create: `mkpdfs-backend/src/functions/auth/cli/device/handler.ts`

- [ ] **Step 1: Write the handler**

```typescript
import { formatJSONResponse, formatErrorResponse } from '@libs/apiGateway';
import { middyfy } from '@libs/lambda';
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient, PutCommand, GetCommand, UpdateCommand } from '@aws-sdk/lib-dynamodb';
import { generateUserCode, generateDeviceCode } from '../codes';

const docClient = DynamoDBDocumentClient.from(new DynamoDBClient({}));

const RATE_LIMIT_MAX = 10;               // device starts per IP
const RATE_LIMIT_WINDOW_MS = 60 * 60 * 1000;
const CODE_TTL_SECONDS = 600;
const POLL_INTERVAL_SECONDS = 5;

async function checkAndBumpRateLimit(ip: string): Promise<boolean> {
  const now = Date.now();
  const key = { pk: `CLI_DEVICE#${ip}`, sk: 'start' };
  const existing = await docClient.send(new GetCommand({
    TableName: process.env.RATE_LIMIT_TABLE!, Key: key,
  }));
  const item = existing.Item;
  const needsReset = !item || item.windowStart < now - RATE_LIMIT_WINDOW_MS;
  if (!needsReset && item!.count >= RATE_LIMIT_MAX) return false;
  await docClient.send(new UpdateCommand({
    TableName: process.env.RATE_LIMIT_TABLE!, Key: key,
    UpdateExpression: needsReset
      ? 'SET #count = :one, windowStart = :now, #ttl = :ttl'
      : 'SET #count = #count + :one, #ttl = :ttl',
    ExpressionAttributeNames: { '#count': 'count', '#ttl': 'ttl' },
    ExpressionAttributeValues: needsReset
      ? { ':one': 1, ':now': now, ':ttl': Math.floor((now + RATE_LIMIT_WINDOW_MS) / 1000) }
      : { ':one': 1, ':ttl': Math.floor((now + RATE_LIMIT_WINDOW_MS) / 1000) },
  }));
  return true;
}

const startDeviceAuth = async (event: any) => {
  try {
    const ip = event.requestContext?.identity?.sourceIp || 'unknown';
    if (!(await checkAndBumpRateLimit(ip))) {
      return formatJSONResponse({ error: 'slow_down', message: 'Too many requests. Try again later.' }, 429);
    }

    const deviceCode = generateDeviceCode();
    const userCode = generateUserCode();   // collision chance ~n/1.1e12 within 10-min TTL: negligible
    const now = Date.now();

    await docClient.send(new PutCommand({
      TableName: process.env.CLI_AUTH_TABLE!,
      Item: {
        deviceCode, userCode,
        status: 'pending',
        createdAt: now,
        lastPolledAt: 0,
        ttl: Math.floor(now / 1000) + CODE_TTL_SECONDS,
      },
      ConditionExpression: 'attribute_not_exists(deviceCode)',
    }));

    const webBase = process.env.STAGE === 'prod' ? 'https://mkpdfs.com' : 'https://dev.mkpdfs.com';
    return formatJSONResponse({
      deviceCode,
      userCode,
      verificationUri: `${webBase}/cli/authorize`,
      expiresIn: CODE_TTL_SECONDS,
      interval: POLL_INTERVAL_SECONDS,
    });
  } catch (error) {
    return formatErrorResponse(error);
  }
};

export const main = middyfy(startDeviceAuth);
```

- [ ] **Step 2: Typecheck**

Run: `npm run typecheck`
Expected: clean.

- [ ] **Step 3: Commit**

```bash
git add src/functions/auth/cli/device/
git commit -m "feat(cli-auth): POST /auth/cli/device handler"
```

---

### Task 4: `POST /auth/cli/approve` handler

**Files:**
- Create: `mkpdfs-backend/src/functions/auth/cli/approve/handler.ts`

- [ ] **Step 1: Write the handler**

JWT-authed (Cognito authorizer at the Gateway + `iamOnlyMiddleware` for `event.userId`). Body: `{ userCode, action: 'approve'|'deny', tokens?: { idToken, accessToken, refreshToken } }`. Approval is an atomic conditional update: only a pending, unexpired code can be bound, exactly once. Failed lookups are capped at 5 per user per 15 min (guessing guard).

```typescript
import { formatJSONResponse, formatErrorResponse } from '@libs/apiGateway';
import { middyfy } from '@libs/lambda';
import { iamOnlyMiddleware } from '@libs/middleware/dualAuth';
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient, QueryCommand, UpdateCommand, GetCommand } from '@aws-sdk/lib-dynamodb';
import { normalizeUserCode } from '../codes';

const docClient = DynamoDBDocumentClient.from(new DynamoDBClient({}));

const MAX_FAILED_LOOKUPS = 5;
const LOCKOUT_WINDOW_MS = 15 * 60 * 1000;

async function bumpFailedLookups(userId: string): Promise<number> {
  const now = Date.now();
  const key = { pk: `CLI_APPROVE#${userId}`, sk: 'failed' };
  const existing = await docClient.send(new GetCommand({
    TableName: process.env.RATE_LIMIT_TABLE!, Key: key,
  }));
  const item = existing.Item;
  const needsReset = !item || item.windowStart < now - LOCKOUT_WINDOW_MS;
  const newCount = needsReset ? 1 : item!.count + 1;
  await docClient.send(new UpdateCommand({
    TableName: process.env.RATE_LIMIT_TABLE!, Key: key,
    UpdateExpression: 'SET #count = :c, windowStart = :w, #ttl = :ttl',
    ExpressionAttributeNames: { '#count': 'count', '#ttl': 'ttl' },
    ExpressionAttributeValues: {
      ':c': newCount,
      ':w': needsReset ? now : item!.windowStart,
      ':ttl': Math.floor((now + LOCKOUT_WINDOW_MS) / 1000),
    },
  }));
  return newCount;
}

async function isLockedOut(userId: string): Promise<boolean> {
  const existing = await docClient.send(new GetCommand({
    TableName: process.env.RATE_LIMIT_TABLE!,
    Key: { pk: `CLI_APPROVE#${userId}`, sk: 'failed' },
  }));
  const item = existing.Item;
  return !!item && item.windowStart >= Date.now() - LOCKOUT_WINDOW_MS && item.count >= MAX_FAILED_LOOKUPS;
}

const approveDevice = async (event: any) => {
  try {
    const userId = event.userId!;
    const { userCode, action, tokens } = event.body ?? {};

    if (!userCode || !['approve', 'deny'].includes(action)) {
      return formatJSONResponse({ message: 'userCode and action (approve|deny) are required' }, 400);
    }
    if (action === 'approve' && !(tokens?.idToken && tokens?.accessToken && tokens?.refreshToken)) {
      return formatJSONResponse({ message: 'tokens {idToken, accessToken, refreshToken} required to approve' }, 400);
    }
    if (await isLockedOut(userId)) {
      return formatJSONResponse({ message: 'Too many failed attempts. Try again later.' }, 429);
    }

    const normalized = normalizeUserCode(userCode);
    const query = await docClient.send(new QueryCommand({
      TableName: process.env.CLI_AUTH_TABLE!,
      IndexName: 'userCode-index',
      KeyConditionExpression: 'userCode = :uc',
      ExpressionAttributeValues: { ':uc': normalized },
    }));

    const nowSec = Math.floor(Date.now() / 1000);
    const item = query.Items?.find((i) => i.status === 'pending' && i.ttl > nowSec);
    if (!item) {
      await bumpFailedLookups(userId);
      return formatJSONResponse({ message: 'Code not found or expired. Check your terminal.' }, 404);
    }

    // Atomic: only transitions a still-pending, unexpired code. Re-approval is rejected.
    await docClient.send(new UpdateCommand({
      TableName: process.env.CLI_AUTH_TABLE!,
      Key: { deviceCode: item.deviceCode },
      UpdateExpression: action === 'approve'
        ? 'SET #status = :new, userId = :uid, tokens = :tok, approvedAt = :now'
        : 'SET #status = :new, userId = :uid, approvedAt = :now',
      ConditionExpression: '#status = :pending AND #ttl > :nowSec',
      ExpressionAttributeNames: { '#status': 'status', '#ttl': 'ttl' },
      ExpressionAttributeValues: {
        ':new': action === 'approve' ? 'approved' : 'denied',
        ':pending': 'pending',
        ':uid': userId,
        ':nowSec': nowSec,
        ':now': Date.now(),
        ...(action === 'approve' ? { ':tok': tokens } : {}),
      },
    }));

    return formatJSONResponse({ success: true, action });
  } catch (error: any) {
    if (error?.name === 'ConditionalCheckFailedException') {
      return formatJSONResponse({ message: 'Code already used or expired.' }, 409);
    }
    return formatErrorResponse(error);
  }
};

export const main = middyfy(approveDevice).use(iamOnlyMiddleware());
```

- [ ] **Step 2: Typecheck**

Run: `npm run typecheck` — Expected: clean.

- [ ] **Step 3: Commit**

```bash
git add src/functions/auth/cli/approve/
git commit -m "feat(cli-auth): POST /auth/cli/approve handler (atomic bind, lockout)"
```

---

### Task 5: `POST /auth/cli/token` handler

**Files:**
- Create: `mkpdfs-backend/src/functions/auth/cli/token/handler.ts`

- [ ] **Step 1: Write the handler**

Public, keyed strictly by `deviceCode`. Enforces the polling interval server-side. One-time read: the approved item is atomically deleted as the tokens are returned (`DeleteCommand` + `ReturnValues: ALL_OLD` + condition).

```typescript
import { formatJSONResponse, formatErrorResponse } from '@libs/apiGateway';
import { middyfy } from '@libs/lambda';
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient, GetCommand, UpdateCommand, DeleteCommand } from '@aws-sdk/lib-dynamodb';

const docClient = DynamoDBDocumentClient.from(new DynamoDBClient({}));
const MIN_POLL_MS = 5000;

const pollToken = async (event: any) => {
  try {
    const { deviceCode } = event.body ?? {};
    if (!deviceCode || typeof deviceCode !== 'string') {
      return formatJSONResponse({ error: 'invalid_request' }, 400);
    }

    const result = await docClient.send(new GetCommand({
      TableName: process.env.CLI_AUTH_TABLE!,
      Key: { deviceCode },
    }));
    const item = result.Item;
    const now = Date.now();

    if (!item || item.ttl <= Math.floor(now / 1000)) {
      return formatJSONResponse({ error: 'expired_token' }, 400);
    }

    if (now - (item.lastPolledAt ?? 0) < MIN_POLL_MS) {
      return formatJSONResponse({ error: 'slow_down' }, 429);
    }
    await docClient.send(new UpdateCommand({
      TableName: process.env.CLI_AUTH_TABLE!,
      Key: { deviceCode },
      UpdateExpression: 'SET lastPolledAt = :now',
      ExpressionAttributeValues: { ':now': now },
    }));

    if (item.status === 'pending') {
      return formatJSONResponse({ error: 'authorization_pending' }, 400);
    }
    if (item.status === 'denied') {
      await docClient.send(new DeleteCommand({
        TableName: process.env.CLI_AUTH_TABLE!, Key: { deviceCode },
      }));
      return formatJSONResponse({ error: 'access_denied' }, 400);
    }

    // approved → one-time atomic read-and-delete
    const deleted = await docClient.send(new DeleteCommand({
      TableName: process.env.CLI_AUTH_TABLE!,
      Key: { deviceCode },
      ConditionExpression: '#status = :approved',
      ExpressionAttributeNames: { '#status': 'status' },
      ReturnValues: 'ALL_OLD',
    }));
    const tokens = deleted.Attributes?.tokens;
    if (!tokens) {
      return formatJSONResponse({ error: 'expired_token' }, 400);
    }

    return formatJSONResponse({
      idToken: tokens.idToken,
      accessToken: tokens.accessToken,
      refreshToken: tokens.refreshToken,
      tokenType: 'Bearer',
    });
  } catch (error: any) {
    if (error?.name === 'ConditionalCheckFailedException') {
      return formatJSONResponse({ error: 'expired_token' }, 400);
    }
    return formatErrorResponse(error);
  }
};

export const main = middyfy(pollToken);
```

- [ ] **Step 2: Typecheck**

Run: `npm run typecheck` — Expected: clean.

- [ ] **Step 3: Commit**

```bash
git add src/functions/auth/cli/token/
git commit -m "feat(cli-auth): POST /auth/cli/token polling handler (one-time read)"
```

---

### Task 6: API stack wiring + dev deploy

**Files:**
- Modify: `mkpdfs-backend/cdk/lib/stacks/api-stack.ts`

- [ ] **Step 1: Wire the three lambdas**

Next to the contactEnterprise block (same `makeFn`/`addRoute` helpers; `tables.cliAuth` from Task 1):

```typescript
    // ---- CLI device-flow auth ----
    const cliDevice = makeFn('CliDeviceFn', {
      entry: 'src/functions/auth/cli/device/handler.ts',
      description: 'CLI device flow: start (public, IP rate-limited)',
    });
    tables.cliAuth.grantWriteData(cliDevice);
    tables.rateLimits.grantReadWriteData(cliDevice);
    addRoute('/auth/cli/device', 'POST', cliDevice, false);

    const cliApprove = makeFn('CliApproveFn', {
      entry: 'src/functions/auth/cli/approve/handler.ts',
      description: 'CLI device flow: approve/deny (Cognito JWT)',
    });
    tables.cliAuth.grantReadWriteData(cliApprove);
    tables.rateLimits.grantReadWriteData(cliApprove);
    addRoute('/auth/cli/approve', 'POST', cliApprove, true);

    const cliToken = makeFn('CliTokenFn', {
      entry: 'src/functions/auth/cli/token/handler.ts',
      description: 'CLI device flow: token polling (public, interval-enforced)',
    });
    tables.cliAuth.grantReadWriteData(cliToken);
    addRoute('/auth/cli/token', 'POST', cliToken, false);
```

- [ ] **Step 2: Typecheck, test, diff**

Run: `npm run typecheck && npm run test && npm run cdk:diff`
Expected: clean; diff shows 3 new lambdas + 3 routes + the table.

- [ ] **Step 3: Deploy to dev**

Run: `npm run cdk:deploy:dev`
Expected: all stacks deploy; **check the very end of the log** — a CFN abort can exit 0 (known gotcha).

- [ ] **Step 4: Commit**

```bash
git add cdk/lib/stacks/api-stack.ts
git commit -m "feat(cli-auth): wire device-flow routes into API stack"
```

---

### Task 7: curl verification of the full flow in dev

No code — evidence gathering. Requires a Cognito JWT from a real dev web session.

- [ ] **Step 1: Start a device flow**

```bash
curl -s -X POST https://dev.apis.mkpdfs.com/auth/cli/device | jq
```
Expected: `{ deviceCode, userCode, verificationUri: "https://dev.mkpdfs.com/cli/authorize", expiresIn: 600, interval: 5 }`.

- [ ] **Step 2: Poll before approval**

```bash
curl -s -X POST https://dev.apis.mkpdfs.com/auth/cli/token \
  -H 'Content-Type: application/json' -d '{"deviceCode":"<from step 1>"}' | jq
```
Expected: `{"error":"authorization_pending"}`. Repeat immediately → `{"error":"slow_down"}`.

- [ ] **Step 3: Approve with a real JWT**

Log into https://dev.mkpdfs.com, grab the idToken/accessToken/refreshToken from devtools → Application → Local Storage (`CognitoIdentityServiceProvider.<clientId>.<user>.{idToken,accessToken,refreshToken}`), then:

```bash
curl -s -X POST https://dev.apis.mkpdfs.com/auth/cli/approve \
  -H "Authorization: Bearer $ID_TOKEN" -H 'Content-Type: application/json' \
  -d '{"userCode":"<userCode>","action":"approve","tokens":{"idToken":"'$ID_TOKEN'","accessToken":"'$ACCESS_TOKEN'","refreshToken":"'$REFRESH_TOKEN'"}}' | jq
```
Expected: `{"success":true,"action":"approve"}`. Re-running the same approve → 409.

- [ ] **Step 4: Collect tokens, confirm one-time read**

Poll again (after ≥5 s): Expected tokens JSON. Poll once more: Expected `{"error":"expired_token"}` (item deleted).

- [ ] **Step 5: Negative checks**

Wrong userCode in approve → 404; 5 wrong attempts → 429. Garbage deviceCode in token → `expired_token`.

---

### Task 8: `/cli/authorize` page in mkpdfs-web

**Files:**
- Create: `mkpdfs-web/src/app/[locale]/cli/authorize/page.tsx`
- Create: `mkpdfs-web/src/app/[locale]/cli/authorize/AuthorizeClient.tsx`
- Create: `mkpdfs-web/src/lib/cliAuth.ts`
- Modify: `mkpdfs-web/src/i18n/messages/en.json`, `mkpdfs-web/src/i18n/messages/es.json`

- [ ] **Step 1: Token-extraction helper**

Amplify v6's `fetchAuthSession()` does **not** expose the refresh token; read it from Amplify's own localStorage keys. `src/lib/cliAuth.ts`:

```typescript
import { fetchAuthSession } from 'aws-amplify/auth'

const CLIENT_ID = process.env.NEXT_PUBLIC_COGNITO_CLIENT_ID!

export interface CliTokens {
  idToken: string
  accessToken: string
  refreshToken: string
}

export async function getCliTokens(): Promise<CliTokens | null> {
  const session = await fetchAuthSession()
  const idToken = session.tokens?.idToken?.toString()
  const accessToken = session.tokens?.accessToken?.toString()
  if (!idToken || !accessToken) return null

  const prefix = `CognitoIdentityServiceProvider.${CLIENT_ID}`
  const lastUser = localStorage.getItem(`${prefix}.LastAuthUser`)
  const refreshToken = lastUser
    ? localStorage.getItem(`${prefix}.${lastUser}.refreshToken`)
    : null
  if (!refreshToken) return null

  return { idToken, accessToken, refreshToken }
}

export async function approveCliDevice(userCode: string, action: 'approve' | 'deny'): Promise<void> {
  const tokens = await getCliTokens()
  if (!tokens) throw new Error('no-session')

  const res = await fetch(`${process.env.NEXT_PUBLIC_API_URL}/auth/cli/approve`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Authorization: `Bearer ${tokens.idToken}`,
    },
    body: JSON.stringify({
      userCode,
      action,
      ...(action === 'approve' ? { tokens } : {}),
    }),
  })
  if (!res.ok) {
    const body = await res.json().catch(() => ({}))
    throw new Error(body.message || `approve-failed-${res.status}`)
  }
}
```

- [ ] **Step 2: Client component**

The user **types the code from their terminal** — no query-param pre-fill (spec anti-phishing requirement). `AuthorizeClient.tsx`:

```tsx
'use client'

import { useState } from 'react'
import { useTranslations } from 'next-intl'
import { useEffect } from 'react'
import { useRouter } from '@/i18n/routing'
import { useAuth } from '@/providers'
import { Button } from '@/components/ui/Button'
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from '@/components/ui/Card'
import { approveCliDevice } from '@/lib/cliAuth'

export default function AuthorizeClient() {
  const t = useTranslations('cli.authorize')
  const router = useRouter()
  const { isAuthenticated, isInitializing, isLoading } = useAuth()
  const [code, setCode] = useState('')
  const [state, setState] = useState<'idle' | 'submitting' | 'done' | 'denied' | 'error'>('idle')
  const [error, setError] = useState('')

  useEffect(() => {
    if (!isInitializing && !isLoading && !isAuthenticated) router.push('/login')
  }, [isAuthenticated, isInitializing, isLoading, router])

  if (isInitializing || isLoading || !isAuthenticated) return null

  const act = async (action: 'approve' | 'deny') => {
    setState('submitting')
    setError('')
    try {
      await approveCliDevice(code, action)
      setState(action === 'approve' ? 'done' : 'denied')
    } catch (e: any) {
      setError(e.message)
      setState('error')
    }
  }

  if (state === 'done' || state === 'denied') {
    return (
      <Card className="max-w-md mx-auto mt-16">
        <CardHeader>
          <CardTitle>{state === 'done' ? t('successTitle') : t('deniedTitle')}</CardTitle>
          <CardDescription>{t('returnToTerminal')}</CardDescription>
        </CardHeader>
      </Card>
    )
  }

  const valid = code.replace(/[^a-zA-Z0-9]/g, '').length === 8
  return (
    <Card className="max-w-md mx-auto mt-16">
      <CardHeader>
        <CardTitle>{t('title')}</CardTitle>
        <CardDescription>{t('subtitle')}</CardDescription>
      </CardHeader>
      <CardContent className="space-y-4">
        <input
          autoFocus
          value={code}
          onChange={(e) => setCode(e.target.value.toUpperCase())}
          placeholder="XXXX-XXXX"
          className="w-full text-center text-2xl font-mono tracking-widest rounded-md border border-border bg-background p-3"
          maxLength={9}
        />
        <p className="text-sm text-muted-foreground">{t('warning')}</p>
        {error && <p className="text-sm text-destructive">{t('error')}</p>}
        <div className="flex gap-3">
          <Button className="flex-1" disabled={!valid || state === 'submitting'}
            isLoading={state === 'submitting'} onClick={() => act('approve')}>
            {t('approve')}
          </Button>
          <Button variant="destructive" className="flex-1" disabled={!valid || state === 'submitting'}
            onClick={() => act('deny')}>
            {t('deny')}
          </Button>
        </div>
      </CardContent>
    </Card>
  )
}
```

- [ ] **Step 3: Page wrapper**

`page.tsx`:

```tsx
import AuthorizeClient from './AuthorizeClient'

export const metadata = { title: 'Authorize CLI — mkpdfs' }

export default function CliAuthorizePage() {
  return <AuthorizeClient />
}
```

- [ ] **Step 4: Translations**

Add to `en.json`:

```json
  "cli": {
    "authorize": {
      "title": "Authorize CLI",
      "subtitle": "Enter the code shown in your terminal",
      "warning": "Only approve if this exact code appears in your terminal. Approving grants that terminal access to your mkpdfs account.",
      "approve": "Approve",
      "deny": "Deny",
      "error": "Code not found or expired. Check your terminal and try again.",
      "successTitle": "CLI authorized",
      "deniedTitle": "Request denied",
      "returnToTerminal": "You can close this tab and return to your terminal."
    }
  }
```

And the Spanish equivalents in `es.json`:

```json
  "cli": {
    "authorize": {
      "title": "Autorizar CLI",
      "subtitle": "Ingresa el código que aparece en tu terminal",
      "warning": "Aprueba solo si este código exacto aparece en tu terminal. Aprobar le da a esa terminal acceso a tu cuenta de mkpdfs.",
      "approve": "Aprobar",
      "deny": "Rechazar",
      "error": "Código no encontrado o expirado. Revisa tu terminal e intenta de nuevo.",
      "successTitle": "CLI autorizado",
      "deniedTitle": "Solicitud rechazada",
      "returnToTerminal": "Puedes cerrar esta pestaña y volver a tu terminal."
    }
  }
```

- [ ] **Step 5: Lint, build, manual end-to-end**

Run: `cd mkpdfs-web && npm run lint && npm run build`
Expected: clean.

Then `npm run dev`, point `NEXT_PUBLIC_API_URL` at `https://dev.apis.mkpdfs.com`, and repeat Task 7 replacing the curl-approve with the real page: start device flow with curl, type the code at `http://localhost:3003/en/cli/authorize`, approve, poll for tokens.
Expected: tokens returned; second poll → `expired_token`.

- [ ] **Step 6: Commit**

```bash
git add src/app/[locale]/cli/ src/lib/cliAuth.ts src/i18n/messages/
git commit -m "feat(cli-auth): branded /cli/authorize device approval page"
```

---

### Task 9: Record decision in spec + push both repos

- [ ] **Step 1: Update the spec**

In `docs/superpowers/specs/2026-06-11-mkpdfs-cli-design.md` (orchestrator repo), replace the "Token minting — primary and fallback" block with the resolved decision: token-handover implemented (CUSTOM_AUTH dropped — `EXTERNAL_PROVIDER` federated users cannot use AdminInitiateAuth; handover works uniformly). Mark open item 1 as resolved.

- [ ] **Step 2: Push dev branches**

```bash
cd mkpdfs-backend && git push origin dev     # triggers CDK deploy dev via GitHub Actions
cd ../mkpdfs-web && git push origin dev      # triggers Amplify dev build
cd .. && git add docs/ mkpdfs-backend mkpdfs-web && git commit -m "CLI device auth live in dev (spec rev 3 + submodule bumps)" 
```

- [ ] **Step 3: Verify the deployed flow once more against dev.mkpdfs.com (not localhost)**

Repeat Task 7 steps 1–4 with the real page at `https://dev.mkpdfs.com/cli/authorize`.
Expected: full flow green. **This is the gate for starting Plan B.**
