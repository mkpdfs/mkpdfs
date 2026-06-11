# mkpdfs CLI (`mkp`) — v1 Design

**Date:** 2026-06-11
**Status:** Approved — rev 2 after external (Codex) design review
**Repos affected:** `mkpdfs-cli/` (new code), `mkpdfs-backend/` (3 new endpoints + Cognito custom auth triggers), `mkpdfs-web/` (1 new page)

## Goal

A command-line tool covering the daily developer workflow for mkpdfs: edit a Handlebars template locally, upload it, generate a test PDF, iterate. It replaces clicking through the web dashboard for template work and gives API consumers a way to exercise the same server-to-server path their integration uses.

## Decisions (settled with the user)

| Decision | Choice |
|---|---|
| Scope v1 | Developer workflow: auth, templates CRUD (push/pull), PDF generation, tokens, usage. AI generation and marketplace are v2. |
| Stack | Go + Cobra, reusing nikte-cli patterns (spinners, tablewriter tables, `--json`, cross-platform config). |
| Binary name | `mkp` — `mk` collides with an existing Homebrew formula and the Plan 9 build tool. |
| Auth UX | Custom device flow with mkpdfs branding. **No Cognito Hosted UI** (explicit user requirement). The browser leg runs in mkpdfs-web (Amplify) so login, including Google, uses the product's own branded experience. |
| Command style | Consistent `mkp <noun> <verb>` (e.g. `mkp auth login`), like `nk auth login`. |
| Output language | English (public product). |

## Command surface

```
mkp
├── auth
│   ├── login              # device flow, branded web approval
│   ├── logout             # clear tokens for current environment
│   └── whoami             # email, plan, current month usage summary
├── templates  (alias: tpl)
│   ├── list   (ls)        # table: ID, name, size, updated; --json
│   ├── get <id>           # metadata + detected {{variables}}
│   ├── pull <id> [-o file]   # download .hbs, record local mapping
│   ├── push <file> [--dry-run]   # create OR update (see Local mapping)
│   └── delete <id>        # confirm prompt; --force skips
├── pdf
│   └── generate -t <id|file> -d data.json [-o out.pdf] [--open] [--api-key]
├── tokens
│   ├── list
│   ├── create [--name N] [--expires-days D] [--save]   # prints tlfy_ once; --save also stores it as the CLI's API key
│   └── revoke <id>
├── usage                  # pages/templates/tokens vs plan limits
├── config                 # get | set | list | path
└── version
```

Global flags: `--env dev|prod`, `--json` (stable, script-safe output on all read commands), `--yes` (assume-yes for all prompts; required for deterministic CI), `--verbose`.

**Stretch (cuttable without affecting the rest):** `mkp dev <file> -d data.json` — watch mode: on every save, re-push + re-generate + reopen the PDF.

## Key experiences

### Login

```
$ mkp auth login

  Opening https://mkpdfs.com/cli/authorize

  Verification code: KXTM-49PF

  If the browser doesn't open, visit the URL
  and enter the code manually.

  ⠋ Waiting for authorization...

  ✓ Logged in as aramis002@gmail.com

  Ready. Try: mkp templates list
```

Headless-friendly: always prints the URL + code, so it works over SSH.

### Edit loop

```
$ mkp templates pull 8f3a... -o invoice.hbs    # downloads + records mapping
$ vim invoice.hbs
$ mkp templates push invoice.hbs               # knows it's an update
  ✓ Updated "Invoice" (8f3a…) — 4.2 KB
$ mkp pdf generate -t invoice.hbs -d sample.json --open
  ⠋ Generating PDF...
  ✓ invoice-2026-06-11.pdf (45 KB, 1 page) — opening...
```

`generate -t` accepts a templateId **or** a local file path; a file path resolves to its templateId via the local mapping.

### PDF generation — two paths

- Default: Cognito-JWT route `POST /pdf/generate`. No API key needed (free tier allows only 1 token, so key creation stays optional).
- `--api-key`: uses `POST /v1/pdf/generate` with the configured `tlfy_` token — exercises exactly the server-to-server path an integration (e.g. Academia Connects) sees. **Hard error if no key is configured** (no silent fallback to JWT).
- The result line (and `--json` output) always states which auth path was used.
- Batch data (array in the JSON file) is validated client-side against the 50-item backend limit before the request.

## Local mapping: `.mkpdfs.json`

Per-directory (current directory only — no parent search), committable:

```json
{
  "environment": "prod",
  "userId": "us-east-1:abc...",
  "templates": {
    "invoice.hbs": {
      "templateId": "8f3a...",
      "name": "Invoice",
      "remoteUpdatedAt": "2026-06-11T18:02:33Z"
    }
  }
}
```

- `pull` creates/updates entries; `push` of a known file → `PUT /templates/{templateId}`; unknown file → prompt "Create new template 'invoice'? [Y/n]" → `POST /templates/upload`.
- Overrides: `push --id <id>` forces update to that ID; `push --new` forces creation. `push --dry-run` shows what would happen (create vs update, target env, remote diff summary) without writing.
- Handlebars is validated locally before upload (error with line/column, no round-trip) — but local validation is **advisory**: the Go validator may not match the backend's JS `Handlebars.compile` exactly, so backend validation remains authoritative and its errors are surfaced verbatim.

### Safety checks (all hard errors unless overridden)

- **Environment mismatch:** if the mapping's `environment` differs from the active config/`--env`, push/pull/generate abort with a clear message. No cross-env writes, ever.
- **Account mismatch:** the mapping records the `userId` that created it; if the logged-in user differs (a collaborator with another account), the CLI warns and refuses to push without `--force` — their account won't own those templateIds anyway.
- **Conflict detection:** before `PUT`, the CLI fetches the remote template's `updatedAt` and compares it to the stored `remoteUpdatedAt`. If the remote changed since the last pull/push, abort with "Remote template changed since your last sync (remote: …, local record: …). Pull first or push --force." The backend `PUT` is currently a blind overwrite, so this client-side check is the v1 guard; a backend `If-Match`/expected-version parameter is listed as a follow-up.
- **Prod confirmation:** pushes to `prod` prompt for confirmation (skipped by `--yes`).

## Auth, config, environments

- Config at `~/Library/Application Support/mkpdfs/config.json` (macOS), `~/.config/mkpdfs/config.json` (Linux), `%APPDATA%/mkpdfs/config.json` (Windows). File mode 0600 on Unix; on Windows 0600 has no real equivalent — documented caveat in v1, OS keychain (`go-keyring`) listed as v2 hardening. Stores JWT/refresh tokens, optional `tlfy_` API key, selected environment.
- `mkp config list` and logs **redact** tokens and API keys (show name/prefix only).
- Tokens are stored **per environment** — logging into dev does not clobber prod.
- `mkp config set environment dev` switches to `dev.apis.mkpdfs.com` / `dev.mkpdfs.com`; global `--env dev|prod` flag for one-off overrides.
- JWT auto-refresh before each call when expired (refresh token via Cognito client).
- **CI / non-browser usage:** `MKPDFS_API_KEY` env var overrides any configured key — `mkp pdf generate --api-key` works in CI with no login. JWT-only commands (templates, tokens, usage) are explicitly unavailable without a browser login in v1; widening those routes to API-key auth is a deliberate backend decision deferred to v2.

### New backend (mkpdfs-backend)

| Endpoint | Auth | Behavior |
|---|---|---|
| `POST /auth/cli/device` | public, rate-limited | Creates `{deviceCode, userCode (XXXX-XXXX), verificationUri, expiresIn: 600, interval: 5}`; DynamoDB item with TTL. |
| `POST /auth/cli/approve` | Cognito JWT | Called by the web approval page; atomically binds the authenticated userId to a **pending, unexpired, unapproved** code (DynamoDB conditional write; reuse rejected). |
| `POST /auth/cli/token` | public | CLI polling endpoint keyed by `deviceCode` (never by `userCode`). Returns `authorization_pending`, `slow_down`, `expired`, or — once approved — Cognito tokens for the CLI. The `deviceCode` is invalidated after the first successful token response (one-time use). |

**Device-flow hardening (requirements, not suggestions):**

- `userCode`: 8 chars from an unambiguous alphabet (no 0/O/1/I; ~40 bits), collision-checked on insert, 10-minute TTL. `deviceCode`: 256-bit random, returned only to the CLI, never displayed.
- Rate limits: per-IP on `/device` and `/token`; `/token` enforces the polling `interval` server-side (`slow_down` on violation); `/approve` allows max 5 failed code lookups per user before temporary lockout — `userCode` guessing must be infeasible within the TTL.
- Approval binds the code to the approving user only; a guessed/wrong code can never bind someone else's pending device.

**Token minting — primary and fallback (verify during planning):**

- *Primary:* CUSTOM_AUTH flow — `/token` lambda calls `AdminInitiateAuth(CUSTOM_AUTH)` for the bound user; define/create/verify challenge triggers validate the `deviceCode` against the DynamoDB item. Tokens are never stored at rest. **Open risk:** Google-federated users have Cognito status `EXTERNAL_PROVIDER` and may not be eligible for this flow — this MUST be verified with a real federated user in dev before committing to this path.
- *Fallback (if federated users fail):* the approval page submits its session tokens to `/approve`; they are stored KMS-encrypted with a ≤5-minute TTL and deleted on first read by `/token`. Less elegant (tokens transit DynamoDB briefly) but works uniformly for all identity types.

### New web (mkpdfs-web)

Page `/cli/authorize`: Amplify login if no session (product branding, Google included). The user must **type or explicitly confirm the exact code shown in their terminal** — no auto-approve from a query param — with copy stating "Only approve if this exact code appears in your terminal" (phishing/code-substitution guard). Approve/Deny buttons; success state tells the user to return to the terminal.

## Output, errors, scripting

- Human output by default: tablewriter tables, spinners, color (respects `NO_COLOR`). `--json` on all read commands for `jq` piping; its shape is a stable contract for scripts.
- Error handling keys off the **structured error body** (message/code fields), not HTTP status alone — the backend is inconsistent today (limit errors surface as `403` in some paths and `429` in others). Mapped actions: auth expired → "Session expired. Run: mkp auth login"; subscription inactive → "Your plan doesn't include this. See: mkpdfs.com/pricing"; plan-limit errors show the limit and current usage.
- Exit codes: `0` success, `1` API error, `2` usage/local-validation error.

## Repo structure & testing

```
mkpdfs-cli/
├── cmd/mkp/main.go
├── internal/
│   ├── cli/          # one cobra command per file
│   ├── api/          # HTTP client, auto-refresh middleware
│   ├── auth/         # device flow
│   ├── config/       # cross-platform config
│   ├── localmap/     # .mkpdfs.json read/write
│   └── util/         # bytes/dates formatting, spinner, tables
├── Makefile          # build, build-all, dev-link, test, lint (nikte pattern)
└── test/integration/ # build-tag guarded, runs against dev env
```

- Unit tests: localmap, config, local Handlebars validation, response parsing.
- Integration tests: `//go:build integration`, against dev environment, excluded from `make test`.
- Version injected via ldflags; distribution via GitHub releases + Homebrew tap.

## Open items to verify during planning

1. **CUSTOM_AUTH with Google-federated users** (`EXTERNAL_PROVIDER` status) — test in dev before building; switch to the documented fallback if it fails.
2. **Backend `If-Match`/expected-version on `PUT /templates/{id}`** — follow-up to replace the client-side conflict check with a real server-side guard.
3. **Pagination on `GET /templates`** — the handler currently returns everything with no `Limit`/`LastEvaluatedKey`; fine at today's plan limits, but the CLI's `--json` contract should anticipate a paginated response before it ossifies.
4. Size limits surfaced by the CLI: max template upload size, max data JSON size, and PDF download streaming behavior — confirm backend numbers and bake them into client-side pre-validation messages.

## Out of scope (v1)

AI template generation, marketplace browse/use, async jobs + webhooks, Stripe/plan upgrade from the CLI, interactive TUI (`-i`), clipboard/QR helpers, OS-keychain secret storage (v2 hardening), API-key auth for template/token routes (deliberate backend decision, v2).
