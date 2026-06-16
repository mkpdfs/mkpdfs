# Template Theming + Predefined Params — Design Spec

**Date:** 2026-06-16
**Status:** Approved design, pending implementation plan
**Scope:** mkpdfs-backend + mkpdfs-web

## Problem

The 13 marketplace templates (`mkpdfs-backend/scripts/marketplace-templates/*.hbs`) are
visually static. Colors are hardcoded as CSS variables in `:root` (e.g. `--accent: #8C6CFF`),
fonts are pulled via a Google Fonts `@import`, and the "logo" is just a `.brand-dot` div
showing the first letter of the company name. The Handlebars `{{...}}` payload carries only
**content** (company name, line items, etc.), never **style**. There is no concept of a
theme, brand, or customization anywhere in the data model or UI. Users cannot brand a
template to their own identity, so the marketplace templates are not usable as-is.

## Goal

Let a user brand a template — **primary color, accent color, font, and logo** — **once when
they adopt it** from the marketplace, with the theme **re-editable later** from the dashboard.
Theme is stored as structured data on the user's template row and applied **at render time**.
Content remains Handlebars params (company name/address are content, not theme).

Additionally, provide a small set of **predefined params** (`{{today}}`, `{{now}}`, `{{year}}`)
available in any template without the user passing them.

## Non-goals

- Per-request theme overrides in the generate API (theme is set on the template, not per call).
- Theming arbitrary user-uploaded templates (the render mechanism is generic and will theme
  any template carrying a `theme`, but the adoption/wizard flow targets marketplace templates).
- A free-form font picker. Fonts come from a curated, enumerated list.
- Arbitrary user CSS. Theme is validated data, never raw CSS/HTML.

## Approach (chosen: A — inject overrides)

Each template keeps its own tasteful `:root{}` defaults but uses a **standard themeable
variable contract**. At render time, if the template row carries a `theme`, the service
injects a sanitized `<style>` override (plus the chosen font `@import`) before `</head>` and
resolves the logo to an inline `data:` URI exposed to a Handlebars helper. No theme → the
template renders with its own defaults, so marketplace previews are untouched.

Approaches B (Handlebars inside CSS) and C (post-process/parse CSS) were rejected: B pollutes
templates and breaks on missing theme; C is slow and fragile.

## Theme token contract

Every marketplace template is normalized so its themeable bits read from these variables.
Neutrals (ink/muted/lines/hairlines) stay fixed as part of each template's design.

```css
:root {
  --brand: #8C6CFF;          /* user primary color */
  --brand-soft: #F3EFFF;     /* derived: tinted background */
  --brand-shadow: rgba(140,108,255,0.28); /* derived: soft shadow */
  --accent: #8C6CFF;         /* user accent color */
  --accent-soft: #F3EFFF;    /* derived */
  --font-heading: 'Fraunces', Georgia, serif;
  --font-body: 'Inter', -apple-system, sans-serif;
}
```

- `--brand` / `--accent` come directly from the user's validated hex values.
- `--brand-soft`, `--brand-shadow`, `--accent-soft` are **computed server-side** from the
  validated hex (the invoice template, for example, currently hardcodes an `--accent-soft`
  and a purple RGBA shadow that must become derived tokens).
- Visual defaults are unchanged: the contract values equal today's hardcoded values.

## Data model

Add an optional `theme` attribute to the user's template row (templates table, PK `userId`,
SK `templateId`):

```ts
theme?: {
  brand: string;        // strict hex "#RRGGBB"
  accent: string;       // strict hex "#RRGGBB"
  fontKey: string;      // enum value, e.g. "inter-fraunces"
  logo?: { type: 's3', key: string };  // always resolved to private S3 (see Logos)
}
```

No `theme` → renders with template defaults (marketplace previews and un-themed templates
behave exactly as today).

Also store a **content version** on the row (S3 `VersionId` from the upload, falling back to
`updatedAt`) so the render cache can key on it (see Rendering).

## Logos

Both logo sources converge to a **private S3 object owned by the user**:

- **Uploaded file:** presigned S3 upload (reusing the `/ai/image-upload-url` pattern) to a
  private key like `users/{userId}/logos/{id}.{ext}`. Content-type allowlist (png/jpg/svg/webp),
  byte-size limit, optional dimension limit.
- **URL:** fetched **server-side at save time** with SSRF hardening (HTTPS-only, redirect cap,
  timeout, private/loopback/link-local IP range blocking, content-type allowlist, size cap),
  then stored to the same private S3 key. The template row never holds an arbitrary external URL.

At render, the logo S3 object is fetched server-side and inlined as a **`data:` URI**. This is
the most reliable PDF path (no public bucket exposure, no presigned expiry, no Chromium network
fetch, no per-render SSRF) and the cleanest multi-tenant boundary. Render resolves to
`logoDataUri` or `null`.

Logo rendering uses a registered Handlebars helper rather than repeating a conditional in every
template:

```hbs
{{{mkpdfsLogo companyName}}}
```

The helper owns escaping, sizing, and the fallback initial mark when no logo exists. It also
absorbs the inconsistent name fields across templates (`companyName`, `businessName`,
`applicantName`, etc.) by accepting the relevant field as its argument.

## Rendering pipeline (`pdfService`)

`PdfService.generatePdf()` is changed to own theme + system params (both the sync route and the
SQS job processor call it, so centralizing avoids duplication):

1. **Fetch the template row** from DynamoDB (content version, `s3Key`, `theme`). Today the
   service only reads S3 and has no row access — this is new.
2. **Get the compiled raw template** from cache, keyed by `userId:templateId:<contentVersion>`.
   - The compiled delegate stays **theme-agnostic** — theme is NOT part of the cache key, so
     themes cannot leak across renders.
   - Bonus fix: today the cache is keyed `userId:templateId` and goes stale after a `PUT`
     content edit across warm Lambdas. Versioning the key fixes that too.
3. **Build the render context:** user `data` + reserved system params + reserved theme namespace
   `__mkpdfsTheme` (resolved `logoDataUri`, color/font values for the helper).
4. **Render Handlebars** (per item for batch/array payloads).
5. **Resolve + inject theme CSS** into the rendered HTML before `</head>`: a sanitized
   `<style>:root{ --brand:…; --brand-soft:…; --brand-shadow:…; --accent:…; --accent-soft:…;
   --font-heading:…; --font-body:… }</style>` plus the chosen font's Google Fonts `@import`.
   Injection happens on the **rendered HTML string**, not the source — keeps the compiled
   delegate reusable.
6. **Render with Puppeteer** as today.

## Predefined params

Reserved system params are merged into the render context in `PdfService` immediately before
render — generated **once per PDF request** so all pages of a batch share the same `now`:

- `today` → `YYYY-MM-DD`
- `now` → ISO 8601 string
- `year` → e.g. `2026`

These names are **reserved**: system values override any user-supplied `data` keys of the same
name, and this is documented. The set is small and closed; it can be extended later.

## Fonts

A curated, enumerated font list. A single map `fontKey → { import, headingStack, bodyStack }`
is the source of truth, shared by:

- **backend** (render: produces the `@import` and the `--font-heading` / `--font-body` stacks),
- **web** (the selector UI and live preview).

~6–8 heading/body pairings. The user picks a `fontKey`; raw `@import` / `font-family` strings
are never accepted from user input.

## Security

Theme is treated as data, never as CSS/HTML:

- `brand` / `accent`: strict hex (`#RGB` or `#RRGGBB`) only; reject `url()`, `var()`, `rgb()`,
  comments, semicolons, and any other characters.
- `fontKey`: enum membership only.
- Uploaded logo: same-`userId` private key, content-type allowlist, byte + dimension limits.
- URL logo: SSRF-hardened server-side fetch (HTTPS-only, redirect cap, timeout, private-range
  IP block, content-type allowlist, size cap), then stored to private S3 — never fetched at
  render and never stored as a live external URL.
- Reserved context namespace `__mkpdfsTheme` prevents collisions with user `data`.

## API surface

- `POST /marketplace/{templateId}/use` (existing `useTemplate`) — extended to accept and store
  the captured `theme` (and logo key) on the new template row.
- `POST /templates/logo-upload-url` (new, or reuse `/ai/image-upload-url` pattern) — presigned
  upload for a logo file.
- `PATCH /templates/{templateId}/theme` (new) — theme-only update via `UpdateCommand`. A
  dedicated endpoint avoids the existing `PUT /templates/{id}` requirement that `content` be
  present and avoids clobbering content/metadata.

## Web (mkpdfs-web)

- **Branding wizard** in the "Use template" flow: primary + accent color pickers, font selector
  with live preview, logo input (upload file **or** paste URL). Submits theme to `useTemplate`.
- **Re-openable theme editor** on the template's dashboard page: same component preloaded with
  the saved `theme`; submits to `PATCH /templates/{id}/theme`.

## Template normalization (one-time)

A single pass over all 13 `.hbs` files:

- Replace hardcoded themeable colors with `var(--brand)` / `var(--accent)` and the derived
  `*-soft` / `*-shadow` tokens.
- Replace hardcoded font families with `var(--font-heading)` / `var(--font-body)`.
- Replace the `.brand-dot` letter mark with the `{{{mkpdfsLogo ...}}}` helper.
- Ensure each `:root` declares the full contract with today's values as defaults.

Then re-run `scripts/seed-marketplace.ts` for dev and prod.

## Testing

- Render **with** a theme: overrides applied, derived tokens correct, font `@import` present.
- Render **without** a theme: defaults intact, output byte-identical to pre-change baseline
  (marketplace previews unaffected).
- Logo: uploaded file → inlined `data:` URI; URL → fetched, stored, inlined; absent → letter
  fallback via helper.
- Predefined params: `today`/`now`/`year` present and consistent across batch pages; reserved
  names override user data.
- Cache: edited content (new version) is picked up; switching theme does not leak across
  renders; concurrent renders of different users' themes stay isolated.
- Security: malformed hex rejected; non-enum `fontKey` rejected; SSRF attempts on logo URL
  blocked; oversized/wrong-type logo rejected.
```

## Open items for the plan

- Exact font list (the 6–8 pairings) — to be finalized during planning.
- Whether logo upload reuses `/ai/image-upload-url` verbatim or gets a dedicated endpoint.
