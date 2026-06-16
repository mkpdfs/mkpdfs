# Template Theming — Web UI Design Spec

**Date:** 2026-06-16
**Status:** Approved design, pending implementation plan
**Scope:** `mkpdfs-web/` (Next.js 14 App Router dashboard). Consumes the backend theming API shipped 2026-06-16 (see `2026-06-16-template-theming-design.md` and `plans/2026-06-16-template-theming-backend.md`).

## Problem

The backend now stores a per-template `theme` (primary color, accent color, curated font, logo) and applies it at render, exposing three endpoints — but there is no UI. Users cannot brand a marketplace template when they adopt it, nor re-edit a theme later. The marketplace templates therefore still look generic to end users.

## Goal

1. **Adopt with branding:** clicking "Use Template" in the marketplace opens a **branding wizard** (modal) where the user picks primary color, accent color, font, and logo, with a **live preview of the real template**. A clear "Use as-is" path adopts with no theme. On apply, adoption sends the theme in one call.
2. **Re-edit later:** the existing user-template preview modal gains an **"Edit branding"** button that re-opens the same wizard preloaded with the saved theme and saves via `PATCH`.

## Non-goals

- No dedicated `/customize` route — the wizard is a modal (matches the codebase, where everything is a modal).
- No server-side preview-render endpoint — the live preview is client-side (the backend stays the source of truth for actual PDF rendering).
- No new test framework — `mkpdfs-web` has no test runner today; adding one is out of scope (verification = typecheck/build/lint + a node sanity script for the pure functions + manual browser E2E).
- No theming of arbitrary user-uploaded templates in the UI (only marketplace-adopted templates carry a theme); the wizard is reachable only from the marketplace adopt flow and the adopted-template preview modal.

## Backend contract (already shipped — consumed as-is)

- `POST /marketplace/templates/{id}/use` — body now accepts optional `{ theme: ThemeInput }`; returns `{ template }`.
- `POST /templates/logo-upload-url` — body `{ contentType }` → `{ uploadUrl, s3Key, expiresIn }`. Client then `PUT`s the file bytes to `uploadUrl`.
- `PATCH /templates/{id}/theme` — body `ThemeInput` → `{ message, theme }`.
- `GET /marketplace/templates/{id}/preview` — public; returns the marketplace template incl. `content` (the `.hbs`) + `sampleDataJson`.
- `GET /templates/{id}` — returns `TemplateWithContent` (the user's copy, incl. `content`, `theme?`, `sourceMarketplaceId`).

**`ThemeInput`** (what the UI sends):
```ts
{
  brand: string;   // "#RRGGBB"
  accent: string;  // "#RRGGBB"
  fontKey: string; // one of the 8 registry keys
  logo?: { source: 'upload'; s3Key: string } | { source: 'url'; url: string } | null;
}
```
**Stored `Theme`** (what `GET /templates/{id}` returns under `theme`): `{ brand, accent, fontKey, logoKey? }`.

The 8 curated fonts (must match backend `src/libs/theme/fonts.ts`): `inter-fraunces` (default), `inter-inter`, `playfair-lato`, `poppins-poppins`, `montserrat-opensans`, `lora-source-sans`, `space-grotesk-inter`, `dmserif-dmsans`. Backend defaults brand/accent to `#8C6CFF`.

## Architecture

### `src/lib/theme/` (new — client-side, preview-only mirror of the backend)
A small, self-contained module. **It duplicates the backend's theme→CSS math on purpose** (separate repos, can't import); the backend remains the source of truth for real rendering. Each file carries a comment cross-linking the backend file it mirrors.

- `fonts.ts` — `FONTS` registry (`fontKey → { label, linkHref, headingStack, bodyStack }`), `DEFAULT_FONT_KEY`, `isFontKey`. Mirrors `mkpdfs-backend/src/libs/theme/fonts.ts`.
- `themePreview.ts` —
  - `softTint(hex)`, `shadowRgba(hex, alpha)` (mirror `colorDerive.ts`).
  - `buildSystemParams(now)` → `{ today, now, year }` (mirror `systemParams.ts`) so preview fidelity matches the backend (templates may use `{{today}}` etc.).
  - `buildThemeHead(theme)` → the `<link>` + `<style id="mkpdfs-theme">:root{…}</style>` string (mirror `buildThemeStyle.ts`).
  - `injectIntoHead(html, fragment)` (mirror `injectTheme.ts`).
  - `composeThemedHtml(templateContent, sampleData, theme?, logoPreviewUrl?)` → registers the **same helpers as the backend** (`ifEq`; boolean `gt`; `formatDate`; `formatCurrency`; and `mkpdfsLogo` reading `options.data.mkpdfsTheme.logoDataUri`, emitting `<img class="brand-logo">` when set else the `.brand-dot` initial). Builds a resolved theme `{ brand, accent, fontKey, logoDataUri: logoPreviewUrl ?? null }`, renders with `compiled({ ...sampleData, ...buildSystemParams(new Date()) }, { data: { mkpdfsTheme } })`, and (if `theme`) injects the head fragment. Returns a full HTML string for an iframe.

### Components (`src/components/theme/`)
- `BrandingWizardModal.tsx` — Radix `Dialog`, two-pane. **Left:** controls. **Right:** `ThemedPreview`. Props: `{ mode: 'adopt' | 'edit', templateContent, sampleData, initialTheme?, existingLogoKey?, onApply(themeInput | null), onClose, isSubmitting }`. Holds form state `{ brand, accent, fontKey, logo }` where `logo` is `null | { kind:'upload', file, previewUrl } | { kind:'url', url } | { kind:'existing', s3Key }`. Footer: **Use as-is** (adopt mode → `onApply(null)`) and **Apply** (→ `onApply(themeInput)`). The modal does NOT call the API itself — the parent owns mutations (keeps the modal reusable for both flows).
- `ColorField.tsx` — label + swatch button → `react-colorful` `HexColorPicker` popover + a hex `<input>` (normalizes/validates `#RRGGBB`).
- `FontSelect.tsx` — dropdown of `FONTS`; each option label rendered in its own font (loads the font links). Emits `fontKey`.
- `LogoInput.tsx` — tabbed **Upload | URL**. Upload reuses the existing drag/drop pattern (produces a `File` + `URL.createObjectURL` preview). URL is a text field, **https-only** (matches the backend SSRF guard; an `http:` URL also fails as mixed content in the https app). Emits the `logo` state above, plus a "remove" affordance. In edit mode it starts in an `existing` state ("Current logo will be kept") until the user replaces or removes it.
- `ThemedPreview.tsx` (new, dedicated) — wraps an iframe; on `{templateContent, sampleData, theme, logoPreviewUrl}` change (debounced ~300 ms) computes `composeThemedHtml(...)` and writes it via `contentDocument.open/write/close`, `sandbox="allow-same-origin"`. Built fresh rather than extending `LivePreview`/`FullScreenPreview` (those are an unused/AI-only pair and register subtly different helpers — keeping a separate component avoids drift and avoids touching the AI flow).

### Data / API (`src/lib/api.ts` + `src/hooks/useApi.ts`)
- Extend `useMarketplaceTemplate(templateId, theme?: ThemeInput)` → posts `{ theme }` when provided.
- Add `getLogoUploadUrl(contentType)` and `uploadLogoFile(file): Promise<string /*s3Key*/>` (presigned `POST /templates/logo-upload-url`, then `PUT` bytes to `uploadUrl` with the right `Content-Type`).
- Add `updateTemplateTheme(templateId, theme: ThemeInput)` → `PATCH /templates/{id}/theme`.
- Hooks: extend `useCopyMarketplaceTemplate` to pass a theme; add `useUpdateTemplateTheme` (invalidates `queryKeys.templates` + `queryKeys.template(id)`).
- Types (`src/types/index.ts`): add `ThemeInput`, `Theme`, `LogoInput`; add `theme?: Theme` and `sourceMarketplaceId?: string` to the `Template`/`TemplateWithContent` types if missing.

### Flows
**Adopt (marketplace):** "Use Template" → fetch that template's preview (`getMarketplaceTemplatePreview`, already used by the preview modal — reuse its `content` + parsed `sampleDataJson`) → open `BrandingWizardModal` (`adopt`). On **Apply**: if logo is an upload, `uploadLogoFile(file)` → `s3Key`, build `ThemeInput` (`logo:{source:'upload',s3Key}` or `{source:'url',url}` or null) → `useCopyMarketplaceTemplate.mutateAsync({ templateId, theme })`. On **Use as-is**: same mutation with no theme. Toast + invalidate (preserves current behavior).

**Edit (adopted template):** `UserTemplatePreviewModal` gets an **"Edit branding"** button → fetch `getTemplate(id)` (content + `theme` + `sourceMarketplaceId`) and the sample data via `getMarketplaceTemplatePreview(sourceMarketplaceId)`'s `sampleDataJson` (safe-parse; fallback to a minimal generic object if absent) → open `BrandingWizardModal` (`edit`, `initialTheme` from the stored `theme`, `existingLogoKey` from `theme.logoKey`). On **Apply**: resolve logo (see below) → `useUpdateTemplateTheme.mutateAsync({ templateId, theme })`. No "Use as-is" in edit mode.

### Logo handling
- Preview uses the local `File` object-URL (upload) or the pasted URL (url) directly in the iframe `<img>` — no backend round-trip while editing.
- The real S3 upload happens only on **Apply** (upload case), via the presigned flow. URL logos are sent as `{source:'url',url}`; the backend ingests them (SSRF-guarded) at save time.
- **Edit-mode logo preservation (critical):** `PATCH` overwrites the whole `theme`, and the backend only sets `logoKey` when `input.logo` is present — so omitting `logo` would silently drop an existing logo. Therefore the wizard's logo state in edit mode starts as `{ kind:'existing', s3Key: theme.logoKey }`, and on Apply:
  - unchanged → send `logo: { source:'upload', s3Key: existingLogoKey }` (re-references the stored key; the backend ownership check is prefix-based on `users/{userId}/logos/`, so the stored key passes — no re-upload),
  - replaced (upload) → upload the new file → `{ source:'upload', s3Key:<new> }`,
  - replaced (url) → `{ source:'url', url }`,
  - removed → `logo: null`,
  - never had one → omit `logo`.
- **Orphaned uploads:** if the logo `PUT` succeeds but the subsequent `use`/`PATCH` fails, the S3 object is left under `users/{userId}/logos/` (the bucket lifecycle only expires `pdfs/`). This is a known backend gap; the UI does not attempt cleanup and must not claim to.

### Dependencies & performance
- Add `react-colorful` (~2.8 KB). `handlebars` is already a dependency.
- **Lazy-load** `BrandingWizardModal` via `next/dynamic` so Handlebars + the picker don't enter the main bundle.
- Preview is debounced (reuse `LivePreview`'s existing 300 ms debounce).

### i18n
Add a `branding.*` namespace to `src/i18n/messages/en.json` and `es.json` (wizard title, field labels, Upload/URL tabs, Use-as-is, Apply, Edit branding, errors). All user-facing strings via `useTranslations`.

## Error handling
- Invalid hex in a `ColorField` → inline validation, Apply disabled until valid (mirrors backend's strict `#RRGGBB`).
- Logo upload failure (presigned/PUT) → toast, keep the wizard open.
- `use`/`PATCH` failure → toast with the API message, keep the wizard open.
- Template/sample fetch failure for the preview → show the wizard with an empty/error preview but still allow choosing a theme (Apply still works).

## Testing / verification
- **Pure functions** (`lib/theme/fonts.ts`, `themePreview.ts`): a node sanity script (no framework) asserting `composeThemedHtml` injects `--brand`, the font `<link>`, and the logo `<img>` vs `.brand-dot` fallback — same spirit as the backend unit tests.
- **Typecheck/build:** `npm run build` (or `tsc --noEmit`) + `npm run lint` must pass.
- **Manual E2E (browser):** adopt a marketplace template with a theme → live preview updates as colors/font/logo change → adopt → generate from the dashboard → themed PDF; "Use as-is" → default render. Edit-branding cases: change colors only with an **existing logo kept** (logo must survive); **existing logo removed**; **upload succeeds but apply fails** (toast, wizard stays open, no partial save); **URL logo previews but backend rejects it** (e.g. non-image) → toast surfaces the backend message.

## Implementation sequence (for the plan)
1. Types + API functions + hooks (`ThemeInput`/`Theme`/`LogoInput`; `uploadLogoFile`, `updateTemplateTheme`; extend `useMarketplaceTemplate`/`useCopyMarketplaceTemplate`; `useUpdateTemplateTheme`).
2. Pure preview mirror (`lib/theme/fonts.ts`, `lib/theme/themePreview.ts`) + node sanity script.
3. Wizard UI primitives (`ColorField`, `FontSelect`, `LogoInput`, `ThemedPreview`) + `BrandingWizardModal`.
4. Marketplace adopt flow wiring (lazy-load the wizard).
5. Edit-branding flow wiring (`UserTemplatePreviewModal` + preservation logic).
6. i18n strings + build/lint + manual E2E.

## File summary
- New: `src/lib/theme/fonts.ts`, `src/lib/theme/themePreview.ts`; `src/components/theme/{BrandingWizardModal,ColorField,FontSelect,LogoInput,ThemedPreview}.tsx`; `scripts/theme-preview-sanity.mjs`.
- Modified: `src/lib/api.ts`, `src/hooks/useApi.ts`, `src/types/index.ts`, the marketplace page / `TemplateCard` / `TemplatePreviewModal` (open wizard on Use), `src/components/templates/UserTemplatePreviewModal.tsx` ("Edit branding"), `src/i18n/messages/{en,es}.json`, `package.json` (`react-colorful`). **Not** modified: `LivePreview.tsx` / `FullScreenPreview.tsx` (the new `ThemedPreview` is standalone).
