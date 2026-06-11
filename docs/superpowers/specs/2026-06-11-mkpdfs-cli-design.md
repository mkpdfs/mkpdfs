# mkpdfs CLI (`mkp`) — v1 Design

**Date:** 2026-06-11
**Status:** Approved
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
│   ├── push <file>        # create OR update (see Local mapping)
│   └── delete <id>        # confirm prompt; --force skips
├── pdf
│   └── generate -t <id|file> -d data.json [-o out.pdf] [--open] [--api-key]
├── tokens
│   ├── list
│   ├── create [--name N] [--expires-days D]   # prints tlfy_ token once
│   └── revoke <id>
├── usage                  # pages/templates/tokens vs plan limits
├── config                 # get | set | list | path
└── version
```

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
- `--api-key`: uses `POST /v1/pdf/generate` with the configured `tlfy_` token — exercises exactly the server-to-server path an integration (e.g. Academia Connects) sees.

## Local mapping: `.mkpdfs.json`

Per-directory, committable:

```json
{
  "environment": "prod",
  "templates": {
    "invoice.hbs": { "templateId": "8f3a...", "name": "Invoice" }
  }
}
```

- `pull` creates/updates entries; `push` of a known file → `PUT /templates/{templateId}`; unknown file → prompt "Create new template 'invoice'? [Y/n]" → `POST /templates/upload`.
- Overrides: `push --id <id>` forces update to that ID; `push --new` forces creation.
- Handlebars is validated **locally before upload** (error with line/column, no round-trip).

## Auth, config, environments

- Config at `~/Library/Application Support/mkpdfs/config.json` (macOS), `~/.config/mkpdfs/config.json` (Linux), `%APPDATA%/mkpdfs/config.json` (Windows). File mode 0600. Stores JWT/refresh tokens, optional `tlfy_` API key, selected environment.
- Tokens are stored **per environment** — logging into dev does not clobber prod.
- `mkp config set environment dev` switches to `dev.apis.mkpdfs.com` / `dev.mkpdfs.com`; global `--env dev|prod` flag for one-off overrides.
- JWT auto-refresh before each call when expired (refresh token via Cognito client).

### New backend (mkpdfs-backend)

| Endpoint | Auth | Behavior |
|---|---|---|
| `POST /auth/cli/device` | public, rate-limited | Creates `{deviceCode, userCode (XXXX-XXXX), verificationUri, expiresIn: 600, interval: 5}`; DynamoDB item with TTL. |
| `POST /auth/cli/approve` | Cognito JWT | Called by the web approval page; binds the authenticated userId to the userCode. |
| `POST /auth/cli/token` | public | CLI polling endpoint. Returns `authorization_pending`, `slow_down`, `expired`, or — once approved — Cognito tokens minted **for the CLI** via CUSTOM_AUTH flow (define/create/verify auth challenge lambda triggers validate the deviceCode). Tokens are never stored in DynamoDB. |

### New web (mkpdfs-web)

Page `/cli/authorize`: Amplify login if no session (product branding, Google included), shows the pending code for the user to confirm, Approve/Deny buttons, success state telling the user to return to the terminal.

## Output, errors, scripting

- Human output by default: tablewriter tables, spinners, color (respects `NO_COLOR`). `--json` on all read commands for `jq` piping.
- Errors map to actions: `401` → "Session expired. Run: mkp auth login"; `402` → "Your plan doesn't include this. See: mkpdfs.com/pricing"; `403` limit errors show the plan limit and current usage.
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

## Out of scope (v1)

AI template generation, marketplace browse/use, async jobs + webhooks, Stripe/plan upgrade from the CLI, interactive TUI (`-i`), clipboard/QR helpers.
