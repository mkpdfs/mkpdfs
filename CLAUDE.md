# CLAUDE.md

This file provides guidance to Claude Code when working with this repository.

## Project Overview

mkpdfs (formerly Templify — the `tlfy_` token prefix and the `github-actions-templify-*` IAM roles survive from that era) is a multi-tenant PDF generation SaaS. Users upload Handlebars templates, manage API keys, and generate PDFs via API or web dashboard.

## Repository Structure

Orchestrator repo with git submodules (each an independent repo with its own CI/CD):

- `mkpdfs-backend/` — AWS CDK app (Lambda, DynamoDB, S3, Cognito, SQS) + TypeScript handlers
- `mkpdfs-web/` — Next.js frontend (Amplify app `d1cfnbzyl1wf46`; mkpdfs.com / dev.mkpdfs.com)
- `mkpdfs-cli/` — Go + Cobra CLI (`mkp`; dev binary `mkp-cli` via `make build`/`make dev-link`). Developer workflow: branded device-flow login (`mkp auth login` → mkpdfs.com/cli/authorize), templates pull/push/list/get/delete with `.mkpdfs.json` mapping + conflict/env guards (JWT, or headless `--api-key` → `/v1/templates/*` since v0.3.0), `pdf generate` (JWT or `--api-key`), credits (balance/ledger/auto-recharge/buy), tokens/usage/config, `instructions [--agent]` (offline embedded markdown walkthrough an AI coding agent reads to author+push a template end-to-end; helper signatures sourced from `pdfService.ts`). Per-env credentials in `~/Library/Application Support/mkpdfs/config.json`. Smoke: `scripts/smoke.sh` vs dev. CLI auth backend: `/auth/cli/{device,approve,token}` (device flow, token handover, one-time read).
- `legacy-paper-api/`, `templify-backend/` — legacy, retired

## Infrastructure (CDK — migrated 2026-06-11, greenfield)

**Serverless Framework was RETIRED.** `mkpdfs-backend/cdk/` is the only deploy path. 6 stacks per env, `-c environment=dev|prod`:

| Stack | Contents |
|---|---|
| `Mkpdfs-Database-{env}` | 10 DynamoDB tables `mkpdfs-{env}-*` (incl. `credit-ledger`) |
| `Mkpdfs-Storage-{env}` | S3 bucket `mkpdfs-{env}-bucket` (versioned; `lambda-layers/` holds the Chromium artifact) |
| `Mkpdfs-Auth-{env}` | Cognito pool + client + identity pool + Hosted UI `auth-mkpdfs-{env}` + Google IdP + native lambda triggers |
| `Mkpdfs-Jobs-{env}` | 4 SQS queues + event source mappings |
| `Mkpdfs-Api-{env}` | RestApi (~28 lambdas, real OPTIONS preflights), custom domain EDGE + Route53 |
| `Mkpdfs-Monitoring-{env}` | 15 CW alarms (billing-focused: webhook errors/signatures, recharge declines, debit failures + API/DDB/DLQ), dashboard `mkpdfs-operations-{env}`, SNS `mkpdfs-alerts-{env}`, CloudWatch RUM app monitor `mkpdfs-web-{env}` + dedicated guest identity pool `mkpdfs-rum-{env}` (added 2026-07-11; frontend real-user monitoring — see mkpdfs-web notes below). Gotcha: its log metric filters key off literal handler log strings AND require the lambda log groups to EXIST (pre-create `/aws/lambda/<fn>` for never-invoked fns or the stack rolls back — bit us on first prod deploy) |

Key facts:
- **Live IDs (post-greenfield)**: dev pool `us-east-1_en1MuJD0a` / client `vis091qbpsj164csp32jketbd`; prod pool `us-east-1_IijpRQ3FN` / client `3mgah7n76j694e5sb0092fl6hn`. Old serverless-era pools/IDs are dead.
- **Chromium layer by ARN**: `arn:aws:lambda:us-east-1:197837191835:layer:mkpdfs-chromium:1` (official Sparticuz v143 x64, artifact in `s3://mkpdfs-prod-bucket/lambda-layers/`). Functions reference the ARN; `puppeteer-core` is bundled by esbuild. Never rebuild/package a local 114MB layer. Updating Chromium = publish a new layer version, bump the ARN.
- NodejsFunction esbuild local bundling (`forceDockerBundling: false`), per-function IAM grants, RemovalPolicy DESTROY dev / RETAIN prod.
- Stripe secrets are read at **runtime from SSM** (`src/libs/ssmParams.ts`); price IDs are deploy-time SSM refs and MUST be `String` type — **CFN does not accept SecureString as template Parameter and the deploy aborts with exit 0** (check the end of the log).
- Migration history, live-verified inventory and runbook lessons: `mkpdfs-backend/docs/cdk-migration-plan.md`.

### Deploy

```bash
cd mkpdfs-backend
npm run cdk:diff          # dev
npm run cdk:deploy:dev    # has --require-approval never (no TTY in CI/agents)
npm run cdk:deploy:prod
```

CI: `.github/workflows/deploy.yml` — push to `dev` → CDK deploy dev; push to `main` → CDK deploy prod (OIDC roles `github-actions-templify-{dev,prod}`, secrets `AWS_ROLE_ARN_{DEV,PROD}`).

## Branch Strategy

| Branch | Environment |
|--------|-------------|
| main | Production (mkpdfs.com, apis.mkpdfs.com) |
| dev | Development (dev.mkpdfs.com, dev.apis.mkpdfs.com) |

All changes go through dev first; merge to main only after dev verification. `stage` is legacy.

## API & Auth

- **Dual auth** (`src/libs/middleware/dualAuth.ts`): Cognito JWT (validated by the API Gateway authorizer) OR API token `x-api-key: tlfy_*` (SHA256 vs tokens table).
- **`POST /v1/pdf/generate`** — server-to-server route, **API-key ONLY** (`apiKeyOnlyMiddleware`, no Gateway authorizer; JWT deliberately rejected there because without the authorizer a forged JWT could impersonate).
- **`PUT /templates/{templateId}`** — update template content in place (multipart or JSON base64, Handlebars validated, ownership-checked; added 2026-06-11).
- **`/v1/templates/*`** — headless template CRUD, **API-key ONLY** (no Gateway authorizer, `apiKeyOnlyMiddleware`; mirrors `/v1/pdf/generate`), added 2026-06-18 (#2): `GET /v1/templates`, `GET|PUT|DELETE /v1/templates/{templateId}`, `POST /v1/templates/upload`. Consumed by the CLI `mkp templates … --api-key`. Token mint/revoke stays JWT-only.
- **`POST /v1/mcp`** — MCP (Model Context Protocol) server, **API-key ONLY** (mirrors `/v1/pdf/generate` and `/v1/templates/*`), added 2026-07-03. Exposes `generate_pdf` + template CRUD as MCP tools by invoking the same `*ApiKey` handlers in-process (synthetic API Gateway event, no network hop) — guarantees auth/credits/subscription parity with REST. Stateless Streamable HTTP transport (fresh `McpServer` + transport per invocation, no session state). Details: `mkpdfs-backend/CLAUDE.md`.
- Biggest consumer: Academia Connects service account `platform@academiaconnects.com` (enterprise, manual "Contact Sales" row). Provisioning is idempotent via `provision-mkpdfs.mjs` in the democonnect-api repo; its secrets live in SSM `/democonnect/labs/mkpdfs[-dev]/*` (SecureString — read with `--with-decryption`, never print).

## Billing (prepaid credits — replaced monthly subscriptions 2026-06-12)

- **$10 = 1,000 credits; 1 credit = 1 PDF page** (same unit as `pageCount`). Credits never expire. New accounts get **10 welcome credits** (`WELCOME_CREDITS` in `src/libs/creditConstants.ts`; the landing promises this). Plans: `credits` (default, flat limits: 500 templates / 10 tokens / 50MB / 15 AI gens-month) | `enterprise` (unlimited, manual).
- Balance lives on the `subscriptions` table (`creditBalance`, atomic `ADD`); `mkpdfs-{env}-credit-ledger` (PK userId, SK entryId) is audit trail AND webhook idempotency (`entryId = stripe#<paymentIntentId>`, conditional put inside a TransactWrite).
- Gate: `checkCreditsMiddleware` → **402 INSUFFICIENT_CREDITS**; debit in after-hook on HTTP 200 (`debitCreditsMiddleware`); async jobs debit in the SQS processor. Brief overdraw under concurrency is accepted by design.
- **Auto-recharge** (opt-in, threshold 1–1,000, default 100): post-debit trigger creates an off-session PaymentIntent with the saved card; `rechargeInProgress` lock with 15-min stale takeover; card decline → webhook disables it + sets `autoRechargeError` (UI banner). Routes: `PUT /billing/auto-recharge`, `GET /billing/ledger`.
- Stripe: one-time Checkout (`setup_future_usage: off_session` saves the card), webhook events `checkout.session.completed` + `payment_intent.succeeded/payment_failed` (recharge PIs carry `metadata.kind=auto_recharge`; purchase PIs are credited ONLY via the checkout event). Price id via SSM String `/mkpdfs/{env}/stripe-price-credits-1000`. Customer Portal = payment-method management only.
- **Refunds**: `charge.refunded` claws back the refunded credits (per-Refund idempotency `stripe#<re_…>`, prorated for partials). Balance may go NEGATIVE (user already spent them) — the 402 gate blocks until they repurchase.
- AI generation requires `creditBalance > 0` (fixed 15/month quota, does NOT consume credits).
- Monthly `usage` table is stats-only now — it never blocks.

## AI Template Generation

Requires positive credit balance (see Billing). Async SQS + Bedrock (Claude) job generates a template; poll `/ai/jobs/{jobId}`. Images >500KB go via presigned S3 (`/ai/image-upload-url`). See `mkpdfs-backend/CLAUDE.md` for details.

## Marketplace

Pre-built templates with public thumbnails at `marketplace/thumbnails[-full]/{templateId}.png` in the env bucket (public-read via bucket policy). Handlers convert `thumbnailKey` → `thumbnailUrl` via `ASSETS_BUCKET_URL`.

## Frontend Observability (CloudWatch RUM — added 2026-07-11)

- App monitor `mkpdfs-web-{env}` lives in the Monitoring stack (dev id `b4e5db41-0e9e-4fc1-9321-e83ba60c9f22`, prod id `4cf8e09e-a922-4bea-b199-1f25adbd6c05`); guest creds via dedicated identity pool `mkpdfs-rum-{env}` (enhanced auth flow — no role ARN needed client-side). domainList dev: `dev.mkpdfs.com` + `localhost`; prod: `mkpdfs.com` + `www.mkpdfs.com` (www is a live Amplify subdomain). 100% session sampling, telemetries errors/performance/http, `cwLogEnabled` (queryable in Logs Insights), plus a `mkpdfs-rum-ingest-spike-{env}` alarm (>10MB/h RumEventPayloadSize) covering abuse of the public guest pool.
- **aws-rum-web does NOT capture console output.** mkpdfs-web logs through `src/lib/rum-logger.ts` (`rum.info/warn/error('<Area>', …)` — Areas: App/Auth/Login/Register/Password/Callback/API/Upload/Billing/AIGenerate): console + forward to RUM (info/warn as `mkpdfs.log` custom events with `{level, area, message}`; errors also `recordError`). Init: `<RumInit />` in the root layout (`src/lib/rum.ts`); off when `NEXT_PUBLIC_RUM_*` unset. Never pass PII/secrets to rum-logger — events ship to CloudWatch (the OAuth callback logs userId only, not email).
- Amplify branch env vars `NEXT_PUBLIC_RUM_APP_MONITOR_ID` + `NEXT_PUBLIC_RUM_IDENTITY_POOL_ID` set on BOTH branches (dev + main, 2026-07-11; values come from the Monitoring stack outputs and are inlined at build time — changing them requires an Amplify rebuild). Live-verified both envs (PutRumEvents 200 from dev.mkpdfs.com and mkpdfs.com).

## Domains & Account

- AWS account `197837191835`, profile `rocketeast`, region us-east-1.
- `mkpdfs.com` zone `Z0217803KO361QOLBIHN` (app: mkpdfs.com/dev.mkpdfs.com via Amplify; API: apis.mkpdfs.com/dev.apis.mkpdfs.com via API GW custom domains, owned by CDK).
- Gotcha: API GW custom domains created by the old serverless plugin lived OUTSIDE CFN — if a stray domain/A/AAAA record ever blocks a deploy, delete domain AND records together.
- Gotcha: a CDK deploy can add API GW resources while the stage keeps serving the OLD deployment snapshot (routes exist but return `{"message":"Missing Authentication Token"}`). Fix: `aws apigateway create-deployment --rest-api-id <id> --stage-name <env>` (happened on prod 2026-06-12 after a partially-failed→rerun deploy).

## PDF Generation Performance (optimized 2026-06-18)

Render speed work, live in dev + prod. Engine stays headless Chromium (a free WeasyPrint swap was evaluated + rejected: no `box-shadow`, partial grid/flex, breaks the arbitrary-HTML/CSS promise). Done: warm browser reuse + concurrent-launch guard; `waitUntil: 'load'` + bounded font wait (replaced slow `networkidle0`); **self-hosted webfonts** (woff2 inlined as `@font-face` data: URIs, no render-time network — `scripts/fetch-fonts.mjs` → `src/libs/theme/generated/fontFaces.ts`, theme path + `{{{mkpdfsFontFaces}}}` helper for marketplace); per-`logoKey` logo cache; PDF lambdas at 4096 MB. Editing fonts → re-run `fetch-fonts.mjs`; editing a marketplace `.hbs` → re-run `seed-marketplace.ts <env>`. Full detail + deferred improvements (output cache, cold-start/provisioned concurrency, persistent Chromium, async email, non-Latin font subsets): `mkpdfs-backend/CLAUDE.md` → "PDF Generation Performance".

## Known backlog

- Tokens with expiration compare `expiresAt` (ISO) against `Date.now()` (epoch) — expired tokens never expire.
- No serverless-offline replacement documented (use dev env + `aws logs tail`).
- `POST /jobs/submit` has the Cognito Gateway authorizer, so its advertised dual-auth (API key) path never worked — tokens get 401 at the Gateway. Found during credits E2E 2026-06-12; either remove the authorizer (in-lambda dual auth like `/v1/pdf/generate`) or add a `/v1/jobs/submit`.
- Stripe test mode has a duplicate archived-candidate product `prod_Ugv5Pj3JdUg7Az` ("1,000 PDF credits", manually created); the canonical one is `prod_UgjDRUuFAj4lUm` (its price is in SSM dev).
- `scripts/generate-thumbnails.ts` does not register the `mkpdfsLogo` helper (despite its "MUST stay identical to pdfService.ts" comment) — regenerating thumbnails for logo-using marketplace templates throws `Missing helper: "mkpdfsLogo"`. Add it (mirror `pdfService.ts`) before next thumbnail regen.
- PDF perf improvements not yet done (deferred until scale): output cache (hash→S3), cold-start work (provisioned concurrency / smaller bundle), persistent Chromium, async email. See `mkpdfs-backend/CLAUDE.md`.
