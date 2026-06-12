# CLAUDE.md

This file provides guidance to Claude Code when working with this repository.

## Project Overview

mkpdfs (formerly Templify — the `tlfy_` token prefix and the `github-actions-templify-*` IAM roles survive from that era) is a multi-tenant PDF generation SaaS. Users upload Handlebars templates, manage API keys, and generate PDFs via API or web dashboard.

## Repository Structure

Orchestrator repo with git submodules (each an independent repo with its own CI/CD):

- `mkpdfs-backend/` — AWS CDK app (Lambda, DynamoDB, S3, Cognito, SQS) + TypeScript handlers
- `mkpdfs-web/` — Next.js frontend (Amplify app `d1cfnbzyl1wf46`; mkpdfs.com / dev.mkpdfs.com)
- `mkpdfs-cli/` — Go + Cobra CLI (`mkp`; dev binary `mkp-cli` via `make build`/`make dev-link`). Developer workflow: branded device-flow login (`mkp auth login` → mkpdfs.com/cli/authorize), templates pull/push with `.mkpdfs.json` mapping + conflict/env guards, `pdf generate` (JWT or `--api-key`), tokens/usage/config. Per-env credentials in `~/Library/Application Support/mkpdfs/config.json`. Smoke: `scripts/smoke.sh` vs dev. CLI auth backend: `/auth/cli/{device,approve,token}` (device flow, token handover, one-time read).
- `legacy-paper-api/`, `templify-backend/` — legacy, retired

## Infrastructure (CDK — migrated 2026-06-11, greenfield)

**Serverless Framework was RETIRED.** `mkpdfs-backend/cdk/` is the only deploy path. 5 stacks per env, `-c environment=dev|prod`:

| Stack | Contents |
|---|---|
| `Mkpdfs-Database-{env}` | 10 DynamoDB tables `mkpdfs-{env}-*` (incl. `credit-ledger`) |
| `Mkpdfs-Storage-{env}` | S3 bucket `mkpdfs-{env}-bucket` (versioned; `lambda-layers/` holds the Chromium artifact) |
| `Mkpdfs-Auth-{env}` | Cognito pool + client + identity pool + Hosted UI `auth-mkpdfs-{env}` + Google IdP + native lambda triggers |
| `Mkpdfs-Jobs-{env}` | 4 SQS queues + event source mappings |
| `Mkpdfs-Api-{env}` | RestApi (~28 lambdas, real OPTIONS preflights), custom domain EDGE + Route53 |

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
- Biggest consumer: Academia Connects service account `platform@academiaconnects.com` (enterprise, manual "Contact Sales" row). Provisioning is idempotent via `provision-mkpdfs.mjs` in the democonnect-api repo; its secrets live in SSM `/democonnect/labs/mkpdfs[-dev]/*` (SecureString — read with `--with-decryption`, never print).

## Billing (prepaid credits — replaced monthly subscriptions 2026-06-12)

- **$10 = 1,000 credits; 1 credit = 1 PDF page** (same unit as `pageCount`). Credits never expire. New accounts get **10 welcome credits** (`WELCOME_CREDITS` in `src/libs/creditConstants.ts`; the landing promises this). Plans: `credits` (default, flat limits: 500 templates / 10 tokens / 50MB / 15 AI gens-month) | `enterprise` (unlimited, manual).
- Balance lives on the `subscriptions` table (`creditBalance`, atomic `ADD`); `mkpdfs-{env}-credit-ledger` (PK userId, SK entryId) is audit trail AND webhook idempotency (`entryId = stripe#<paymentIntentId>`, conditional put inside a TransactWrite).
- Gate: `checkCreditsMiddleware` → **402 INSUFFICIENT_CREDITS**; debit in after-hook on HTTP 200 (`debitCreditsMiddleware`); async jobs debit in the SQS processor. Brief overdraw under concurrency is accepted by design.
- **Auto-recharge** (opt-in, threshold 1–1,000, default 100): post-debit trigger creates an off-session PaymentIntent with the saved card; `rechargeInProgress` lock with 15-min stale takeover; card decline → webhook disables it + sets `autoRechargeError` (UI banner). Routes: `PUT /billing/auto-recharge`, `GET /billing/ledger`.
- Stripe: one-time Checkout (`setup_future_usage: off_session` saves the card), webhook events `checkout.session.completed` + `payment_intent.succeeded/payment_failed` (recharge PIs carry `metadata.kind=auto_recharge`; purchase PIs are credited ONLY via the checkout event). Price id via SSM String `/mkpdfs/{env}/stripe-price-credits-1000`. Customer Portal = payment-method management only.
- AI generation requires `creditBalance > 0` (fixed 15/month quota, does NOT consume credits).
- Monthly `usage` table is stats-only now — it never blocks.

## AI Template Generation

Requires positive credit balance (see Billing). Async SQS + Bedrock (Claude) job generates a template; poll `/ai/jobs/{jobId}`. Images >500KB go via presigned S3 (`/ai/image-upload-url`). See `mkpdfs-backend/CLAUDE.md` for details.

## Marketplace

Pre-built templates with public thumbnails at `marketplace/thumbnails[-full]/{templateId}.png` in the env bucket (public-read via bucket policy). Handlers convert `thumbnailKey` → `thumbnailUrl` via `ASSETS_BUCKET_URL`.

## Domains & Account

- AWS account `197837191835`, profile `rocketeast`, region us-east-1.
- `mkpdfs.com` zone `Z0217803KO361QOLBIHN` (app: mkpdfs.com/dev.mkpdfs.com via Amplify; API: apis.mkpdfs.com/dev.apis.mkpdfs.com via API GW custom domains, owned by CDK).
- Gotcha: API GW custom domains created by the old serverless plugin lived OUTSIDE CFN — if a stray domain/A/AAAA record ever blocks a deploy, delete domain AND records together.
- Gotcha: a CDK deploy can add API GW resources while the stage keeps serving the OLD deployment snapshot (routes exist but return `{"message":"Missing Authentication Token"}`). Fix: `aws apigateway create-deployment --rest-api-id <id> --stage-name <env>` (happened on prod 2026-06-12 after a partially-failed→rerun deploy).

## Known backlog

- Tokens with expiration compare `expiresAt` (ISO) against `Date.now()` (epoch) — expired tokens never expire.
- No serverless-offline replacement documented (use dev env + `aws logs tail`).
- `POST /jobs/submit` has the Cognito Gateway authorizer, so its advertised dual-auth (API key) path never worked — tokens get 401 at the Gateway. Found during credits E2E 2026-06-12; either remove the authorizer (in-lambda dual auth like `/v1/pdf/generate`) or add a `/v1/jobs/submit`.
- Stripe test mode has a duplicate archived-candidate product `prod_Ugv5Pj3JdUg7Az` ("1,000 PDF credits", manually created); the canonical one is `prod_UgjDRUuFAj4lUm` (its price is in SSM dev).
