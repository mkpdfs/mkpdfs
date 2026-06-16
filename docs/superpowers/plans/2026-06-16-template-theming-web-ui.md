# Template Theming — Web UI Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a branding wizard (color, accent, font, logo) to the marketplace "Use Template" flow with a live preview of the real template, plus an "Edit branding" entry on adopted templates — consuming the already-shipped backend theming API.

**Architecture:** A client-side, preview-only mirror of the backend's theme→CSS math (`src/lib/theme/`) feeds a sandboxed iframe (`ThemedPreview`). A reusable `BrandingWizardModal` (parent owns the mutations) is opened from the marketplace adopt flow (lazy-loaded) and from the adopted-template preview modal. Logos upload to S3 via the presigned flow only on Apply; URL logos pass through to the backend's SSRF-guarded ingest. The backend stays the source of truth for real rendering.

**Tech Stack:** Next.js 14 (App Router), TypeScript, React Query (TanStack), Radix Dialog, Tailwind, next-intl, `handlebars` (already a dep), `react-colorful` (added here). No test runner exists in `mkpdfs-web` — verification is a node sanity script for the pure modules + `npm run build` + `npm run lint` + manual browser E2E.

**Working dir:** `/Users/sim4r4/sim4r4/repos/mkpdfs/mkpdfs-web` (git submodule). Create a feature branch off `dev` before starting. Backend contract recap: `POST /marketplace/templates/{id}/use` accepts `{theme?}`; `POST /templates/logo-upload-url` → `{uploadUrl,s3Key}`; `PATCH /templates/{id}/theme`; `GET /marketplace/templates/{id}/preview` (public, returns `content`+`sampleDataJson`); `GET /templates/{id}` (returns `content`+`theme?`+`sourceMarketplaceId`).

---

## File Structure

**New:**
- `src/lib/theme/fonts.ts` — `FONTS` registry (8) + `DEFAULT_FONT_KEY` + `isFontKey` (mirror of backend).
- `src/lib/theme/themePreview.ts` — `softTint`, `shadowRgba`, `buildSystemParams`, `buildThemeHead`, `injectIntoHead`, `composeThemedHtml`.
- `src/components/theme/ThemedPreview.tsx` — debounced iframe preview.
- `src/components/theme/ColorField.tsx`, `FontSelect.tsx`, `LogoInput.tsx` — wizard controls.
- `src/components/theme/BrandingWizardModal.tsx` — the modal (parent owns mutations).
- `scripts/theme-preview-sanity.mjs` — node assertions for the pure modules.

**Modified:**
- `src/types/index.ts` — `ThemeInput`, `Theme`, `LogoInput`; add `theme?`/`sourceMarketplaceId?` to template types.
- `src/lib/api.ts` — `getLogoUploadUrl`, `uploadLogoFile`, `updateTemplateTheme`; extend `useMarketplaceTemplate`.
- `src/hooks/useApi.ts` — extend `useCopyMarketplaceTemplate`; add `useUpdateTemplateTheme`.
- marketplace page + `TemplateCard`/`TemplatePreviewModal` — open wizard on Use.
- `src/components/templates/UserTemplatePreviewModal.tsx` — "Edit branding".
- `src/i18n/messages/{en,es}.json` — `branding.*`.
- `package.json` — `react-colorful`.

---

## Task 1: Types, API functions, hooks

**Files:** Modify `src/types/index.ts`, `src/lib/api.ts`, `src/hooks/useApi.ts`.

- [ ] **Step 1: Add types** — in `src/types/index.ts` add:

```ts
export type LogoInput =
  | { source: 'upload'; s3Key: string }
  | { source: 'url'; url: string }
  | null

export interface ThemeInput {
  brand: string
  accent: string
  fontKey: string
  logo?: LogoInput
}

export interface Theme {
  brand: string
  accent: string
  fontKey: string
  logoKey?: string
}
```

Find the `Template` and `TemplateWithContent` interfaces and add (if not present): `theme?: Theme` and `sourceMarketplaceId?: string`.

- [ ] **Step 2: Add API functions** — in `src/lib/api.ts`. Extend `useMarketplaceTemplate` to accept an optional theme, and add the logo-upload + theme-patch functions. Note: the presigned S3 `PUT` must use plain `fetch` (NOT `authFetch`, which injects the Authorization header, a JSON content-type, and the API base URL).

**Rename** the existing `useMarketplaceTemplate` (it reads like a hook and collides with the React Query hook of the same name) to `copyMarketplaceTemplate`, and add the theme arg:

```ts
export async function copyMarketplaceTemplate(templateId: string, theme?: ThemeInput): Promise<Template> {
  const response = await authFetch<{ message: string; template: Template }>(
    `/marketplace/templates/${templateId}/use`,
    { method: 'POST', body: JSON.stringify(theme ? { theme } : {}) }
  )
  return response.template
}
```

Update the hook's import and any `useMarketplaceTemplate as copyMarketplaceTemplate` alias accordingly. Grep to find everything to update:
```bash
rg "useMarketplaceTemplate|copyMarketplaceTemplate|copyTemplate\.(mutate|mutateAsync)" src
```

Add the logo-upload + theme-patch functions in the same file:

```ts
const LOGO_CONTENT_TYPES: Record<string, string> = {
  'image/png': 'png', 'image/jpeg': 'jpg', 'image/webp': 'webp', 'image/svg+xml': 'svg',
}

export async function uploadLogoFile(file: File): Promise<string> {
  if (!LOGO_CONTENT_TYPES[file.type]) {
    throw new Error('Logo must be PNG, JPEG, WebP or SVG')
  }
  const { uploadUrl, s3Key } = await authFetch<{ uploadUrl: string; s3Key: string }>(
    '/templates/logo-upload-url',
    { method: 'POST', body: JSON.stringify({ contentType: file.type }) }
  )
  const put = await fetch(uploadUrl, { method: 'PUT', body: file, headers: { 'Content-Type': file.type } })
  if (!put.ok) throw new Error(`Logo upload failed: ${put.status}`)
  return s3Key
}

export async function updateTemplateTheme(templateId: string, theme: ThemeInput): Promise<Theme> {
  const response = await authFetch<{ message: string; theme: Theme }>(
    `/templates/${templateId}/theme`,
    { method: 'PATCH', body: JSON.stringify(theme) }
  )
  return response.theme
}
```

Import the new types at the top of `api.ts` (match the file's existing import style).

- [ ] **Step 3: Update hooks** — in `src/hooks/useApi.ts`. Extend the copy mutation to pass a theme and add the theme-update mutation:

```ts
export function useCopyMarketplaceTemplate() {
  const queryClient = useQueryClient()
  return useMutation({
    mutationFn: ({ templateId, theme }: { templateId: string; theme?: ThemeInput }) =>
      copyMarketplaceTemplate(templateId, theme),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: queryKeys.templates })
      queryClient.invalidateQueries({ queryKey: queryKeys.usage })
    },
  })
}

export function useUpdateTemplateTheme() {
  const queryClient = useQueryClient()
  return useMutation({
    mutationFn: ({ templateId, theme }: { templateId: string; theme: ThemeInput }) =>
      updateTemplateTheme(templateId, theme),
    onSuccess: (_data, { templateId }) => {
      queryClient.invalidateQueries({ queryKey: queryKeys.templates })
      queryClient.invalidateQueries({ queryKey: queryKeys.template(templateId) })
    },
  })
}
```

Update the import in `useApi.ts` to pull `copyMarketplaceTemplate` (renamed in Step 2) + `updateTemplateTheme` + the `ThemeInput` type from `@/lib/api`/`@/types`. **The signature of `useCopyMarketplaceTemplate` changed from `(templateId)` to `({templateId, theme})`** — its only current call site is `copyTemplate.mutateAsync(template.templateId)` in the marketplace page, updated in Task 4.

- [ ] **Step 4: Typecheck**

Run: `npm run build` (Next type-checks during build). It will FAIL at the old `useCopyMarketplaceTemplate` call site in the marketplace page (expected — fixed in Task 4). To check just this task in isolation instead, run `npx tsc --noEmit` and confirm the ONLY errors are the changed-signature call sites you will fix in Task 4.

- [ ] **Step 5: Commit**

```bash
git add src/types/index.ts src/lib/api.ts src/hooks/useApi.ts
git commit -m "feat(theme-ui): types, logo-upload/theme API fns, hooks"
```

---

## Task 2: Pure preview mirror + sanity script

**Files:** Create `src/lib/theme/fonts.ts`, `src/lib/theme/themePreview.ts`, `scripts/theme-preview-sanity.mjs`.

- [ ] **Step 1: Write the sanity script first (it will fail until the modules exist)** — `scripts/theme-preview-sanity.mjs`:

```js
// Sanity checks for the client-side theme preview mirror (no test runner in this repo).
// Run: node scripts/theme-preview-sanity.mjs   → prints THEME_PREVIEW_OK or throws.
import { FONTS, DEFAULT_FONT_KEY, isFontKey } from '../src/lib/theme/fonts.ts'
import { composeThemedHtml, softTint, shadowRgba } from '../src/lib/theme/themePreview.ts'

const assert = (c, m) => { if (!c) throw new Error('FAIL: ' + m) }

assert(Object.keys(FONTS).length === 8, '8 fonts')
assert(isFontKey(DEFAULT_FONT_KEY) && !isFontKey('nope'), 'isFontKey')
assert(softTint('#000000') === '#ebebeb', 'softTint black->#ebebeb')
assert(shadowRgba('#8c6cff', 0.28) === 'rgba(140, 108, 255, 0.28)', 'shadowRgba')

const tpl = '<html><head><style>:root{--brand:#000}</style></head><body>{{mkpdfsLogo companyName}}|{{today}}|{{companyName}}</body></html>'

const plain = composeThemedHtml(tpl, { companyName: 'Acme' })
assert(!plain.includes('id="mkpdfs-theme"'), 'no theme style when no theme')
assert(plain.includes('brand-dot') && plain.includes('>A<'), 'logo fallback initial')
assert(/\|\d{4}-\d{2}-\d{2}\|Acme/.test(plain), 'system params + content')

const themed = composeThemedHtml(tpl, { companyName: 'Acme' },
  { brand: '#0f62fe', accent: '#ff6b35', fontKey: 'poppins-poppins' }, 'blob:fake')
assert(themed.includes('id="mkpdfs-theme"'), 'theme style injected')
assert(themed.includes('--brand: #0f62fe;'), 'brand var')
assert(themed.includes('fonts.googleapis.com'), 'font link')
assert(themed.includes('<img class="brand-logo" src="blob:fake"'), 'logo img')
assert(!themed.includes('brand-dot'), 'no fallback when logo set')

console.log('THEME_PREVIEW_OK')
```

**This script is OPTIONAL / non-gating.** The repo's Node is v20 (`.nvmrc`) — no `--experimental-strip-types` — and `tsx` is not installed, so it will NOT run out of the box (extensionless TS imports + the `@/` alias). Keep it as executable documentation. The **authoritative** verification for these pure modules is `npm run build` (full typecheck, Step 4) + the live preview in Task 4. Run the script only if you choose to add `tsx` first (`npm i -D tsx && npx tsx scripts/theme-preview-sanity.mjs` → `THEME_PREVIEW_OK`).

- [ ] **Step 2: Write `src/lib/theme/fonts.ts`** — copy the backend registry verbatim (must match `mkpdfs-backend/src/libs/theme/fonts.ts`):

```ts
// MIRROR of mkpdfs-backend/src/libs/theme/fonts.ts — keep in sync. Preview-only.
export interface FontDef { label: string; linkHref: string; headingStack: string; bodyStack: string }

const SANS = `-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif`
const SERIF = `Georgia, 'Times New Roman', serif`

export const FONTS: Record<string, FontDef> = {
  'inter-fraunces': { label: 'Inter + Fraunces', linkHref: 'https://fonts.googleapis.com/css2?family=Fraunces:opsz,wght@9..144,400;9..144,500;9..144,600&family=Inter:wght@400;500;600;700&display=swap', headingStack: `'Fraunces', ${SERIF}`, bodyStack: `'Inter', ${SANS}` },
  'inter-inter': { label: 'Inter', linkHref: 'https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700;800&display=swap', headingStack: `'Inter', ${SANS}`, bodyStack: `'Inter', ${SANS}` },
  'playfair-lato': { label: 'Playfair Display + Lato', linkHref: 'https://fonts.googleapis.com/css2?family=Playfair+Display:wght@500;600;700&family=Lato:wght@400;700&display=swap', headingStack: `'Playfair Display', ${SERIF}`, bodyStack: `'Lato', ${SANS}` },
  'poppins-poppins': { label: 'Poppins', linkHref: 'https://fonts.googleapis.com/css2?family=Poppins:wght@400;500;600;700&display=swap', headingStack: `'Poppins', ${SANS}`, bodyStack: `'Poppins', ${SANS}` },
  'montserrat-opensans': { label: 'Montserrat + Open Sans', linkHref: 'https://fonts.googleapis.com/css2?family=Montserrat:wght@500;600;700&family=Open+Sans:wght@400;600&display=swap', headingStack: `'Montserrat', ${SANS}`, bodyStack: `'Open Sans', ${SANS}` },
  'lora-source-sans': { label: 'Lora + Source Sans 3', linkHref: 'https://fonts.googleapis.com/css2?family=Lora:wght@500;600;700&family=Source+Sans+3:wght@400;600&display=swap', headingStack: `'Lora', ${SERIF}`, bodyStack: `'Source Sans 3', ${SANS}` },
  'space-grotesk-inter': { label: 'Space Grotesk + Inter', linkHref: 'https://fonts.googleapis.com/css2?family=Space+Grotesk:wght@500;600;700&family=Inter:wght@400;500;600&display=swap', headingStack: `'Space Grotesk', ${SANS}`, bodyStack: `'Inter', ${SANS}` },
  'dmserif-dmsans': { label: 'DM Serif Display + DM Sans', linkHref: 'https://fonts.googleapis.com/css2?family=DM+Serif+Display&family=DM+Sans:wght@400;500;700&display=swap', headingStack: `'DM Serif Display', ${SERIF}`, bodyStack: `'DM Sans', ${SANS}` },
}
export const DEFAULT_FONT_KEY = 'inter-fraunces'
export function isFontKey(key: unknown): key is string {
  return typeof key === 'string' && Object.prototype.hasOwnProperty.call(FONTS, key)
}
```

- [ ] **Step 3: Write `src/lib/theme/themePreview.ts`**:

```ts
// Preview-only MIRROR of the backend theme→CSS math
// (mkpdfs-backend/src/libs/theme/{colorDerive,buildThemeStyle,injectTheme}.ts + systemParams.ts
//  + the mkpdfsLogo helper in services/pdfService.ts). Keep in sync. The backend is the
//  source of truth for actual PDF rendering; this only drives the wizard's live preview.
import Handlebars from 'handlebars'
import { FONTS } from './fonts'
import type { ThemeInput } from '@/types'

function hexToRgb(hex: string): [number, number, number] {
  let h = hex.replace('#', '')
  if (h.length === 3) h = h.split('').map((c) => c + c).join('')
  return [parseInt(h.slice(0, 2), 16), parseInt(h.slice(2, 4), 16), parseInt(h.slice(4, 6), 16)]
}
const toHex = (n: number) => Math.round(n).toString(16).padStart(2, '0')
export function softTint(hex: string, weight = 0.08): string {
  const [r, g, b] = hexToRgb(hex)
  const mix = (c: number) => c * weight + 255 * (1 - weight)
  return `#${toHex(mix(r))}${toHex(mix(g))}${toHex(mix(b))}`
}
export function shadowRgba(hex: string, alpha: number): string {
  const [r, g, b] = hexToRgb(hex)
  return `rgba(${r}, ${g}, ${b}, ${alpha})`
}

function buildSystemParams(now: Date) {
  return { today: now.toISOString().slice(0, 10), now: now.toISOString(), year: now.getUTCFullYear() }
}

function buildThemeHead(theme: ThemeInput): string {
  const font = FONTS[theme.fontKey] ?? FONTS['inter-fraunces']
  const vars = [
    `--brand: ${theme.brand};`, `--brand-soft: ${softTint(theme.brand)};`,
    `--brand-shadow: ${shadowRgba(theme.brand, 0.28)};`,
    `--accent: ${theme.accent};`, `--accent-soft: ${softTint(theme.accent)};`,
    `--font-heading: ${font.headingStack};`, `--font-body: ${font.bodyStack};`,
  ].join(' ')
  return `<link rel="stylesheet" href="${font.linkHref}"><style id="mkpdfs-theme">:root { ${vars} }</style>`
}
function injectIntoHead(html: string, fragment: string): string {
  const i = html.search(/<\/head>/i)
  return i === -1 ? fragment + html : html.slice(0, i) + fragment + html.slice(i)
}

let registered = false
function registerHelpers() {
  if (registered) return
  registered = true
  Handlebars.registerHelper('ifEq', function (this: unknown, a: unknown, b: unknown, o: Handlebars.HelperOptions) {
    return a == b ? o.fn(this) : o.inverse(this)
  })
  Handlebars.registerHelper('gt', (a: unknown, b: unknown) => Number(a) > Number(b))
  Handlebars.registerHelper('formatDate', (d: unknown) => { try { return new Date(d as any).toLocaleDateString() } catch { return String(d) } })
  Handlebars.registerHelper('formatCurrency', (a: unknown) => new Intl.NumberFormat('en-US', { style: 'currency', currency: 'USD' }).format(Number(a)))
  Handlebars.registerHelper('mkpdfsLogo', function (name: unknown, o: Handlebars.HelperOptions) {
    const theme = (o?.data as any)?.mkpdfsTheme as { logoDataUri: string | null } | undefined
    if (theme && theme.logoDataUri) {
      return new Handlebars.SafeString(`<img class="brand-logo" src="${theme.logoDataUri}" alt="">`)
    }
    const initial = (typeof name === 'string' && name.trim() ? name.trim()[0] : '') || ''
    return new Handlebars.SafeString(`<div class="brand-dot">${Handlebars.escapeExpression(initial)}</div>`)
  })
}

/** Render the template with sample data + (optional) theme, returning HTML for an iframe. */
export function composeThemedHtml(
  templateContent: string,
  sampleData: Record<string, unknown>,
  theme?: ThemeInput,
  logoPreviewUrl?: string | null,
): string {
  registerHelpers()
  const ctx = { ...(sampleData || {}), ...buildSystemParams(new Date()) }
  const resolved = theme ? { ...theme, logoDataUri: logoPreviewUrl ?? null } : undefined
  const compiled = Handlebars.compile(templateContent)
  let html = compiled(ctx, resolved ? { data: { mkpdfsTheme: resolved } } : undefined)
  if (theme) html = injectIntoHead(html, buildThemeHead(theme))
  return html
}
```

- [ ] **Step 4: Verify the modules typecheck (gating)** — `cd mkpdfs-web && npx tsc --noEmit` → no new errors in `src/lib/theme/*`. (The Step 1 script is optional per its note.)

- [ ] **Step 5: Commit**

```bash
git add src/lib/theme/ scripts/theme-preview-sanity.mjs
git commit -m "feat(theme-ui): client-side theme preview mirror"
```

---

## Task 3: Wizard controls + ThemedPreview + BrandingWizardModal

**Files:** Create `src/components/theme/{ThemedPreview,ColorField,FontSelect,LogoInput,BrandingWizardModal}.tsx`. Add `react-colorful`.

- [ ] **Step 1: Add the dependency** — `cd mkpdfs-web && npm install react-colorful` (commit the lockfile change with this task).

- [ ] **Step 2: `ThemedPreview.tsx`** — debounced iframe; mirrors how `src/components/templates/LivePreview.tsx` writes to the iframe (`contentDocument.open/write/close`, `sandbox="allow-same-origin"`). Read `LivePreview.tsx` first to match the iframe wiring and styling conventions.

```tsx
'use client'
import { useEffect, useRef, useState } from 'react'
import { composeThemedHtml } from '@/lib/theme/themePreview'
import type { ThemeInput } from '@/types'

interface Props {
  templateContent: string
  sampleData: Record<string, unknown>
  theme?: ThemeInput
  logoPreviewUrl?: string | null
  className?: string
}

export function ThemedPreview({ templateContent, sampleData, theme, logoPreviewUrl, className }: Props) {
  const iframeRef = useRef<HTMLIFrameElement>(null)
  const [html, setHtml] = useState('')
  const debounce = useRef<ReturnType<typeof setTimeout> | null>(null)

  useEffect(() => {
    if (debounce.current) clearTimeout(debounce.current)
    debounce.current = setTimeout(() => {
      try { setHtml(templateContent ? composeThemedHtml(templateContent, sampleData, theme, logoPreviewUrl) : '') }
      catch { setHtml('<p style="font-family:sans-serif;padding:16px;color:#b00">Preview error</p>') }
    }, 300)
    return () => { if (debounce.current) clearTimeout(debounce.current) }
  }, [templateContent, sampleData, theme, logoPreviewUrl])

  useEffect(() => {
    const doc = iframeRef.current?.contentDocument
    if (doc) { doc.open(); doc.write(html); doc.close() }
  }, [html])

  return <iframe ref={iframeRef} title="Theme preview" className={className} sandbox="allow-same-origin" />
}
```

- [ ] **Step 3: `ColorField.tsx`** — a labeled swatch that opens a `react-colorful` `HexColorPicker` popover, plus a `#RRGGBB` text input. Validate with `/^#[0-9a-fA-F]{6}$/`; expose `value`/`onChange(hex)` and an `isValid` state the parent can read (or call `onChange` only with valid values and keep local text state). Use the project's Tailwind classes (match an existing input in `src/components/ui/`). `import { HexColorPicker } from 'react-colorful'`.

- [ ] **Step 4: `FontSelect.tsx`** — a `<select>` (or styled dropdown matching the codebase) over `Object.entries(FONTS)`; render each option's `label` styled with its `headingStack` (inject the font `<link>`s once via a hidden `<link>` per font or a single combined link). Props: `value: string`, `onChange(fontKey)`.

- [ ] **Step 5: `LogoInput.tsx`** — tabs **Upload | URL**. Upload: reuse the drag/drop pattern from `src/components/ui/DropZone.tsx` (or the templates upload zone); accept png/jpeg/webp/svg; on file → emit `{ kind:'upload', file, previewUrl: URL.createObjectURL(file) }` (revoke the object URL on unmount/replace). URL: text input requiring `https://` (block other schemes with inline validation) → emit `{ kind:'url', url }` with `previewUrl = url`. Show a "remove" button → emit `null`. In edit mode the initial state is `{ kind:'existing', s3Key }` rendered as "Current logo will be kept" with no preview image (the private S3 object isn't web-fetchable). Props: `value`, `onChange(logoState)`, `mode`.

- [ ] **Step 6: `BrandingWizardModal.tsx`** — Radix `Dialog` (match `src/components/ui/Dialog.tsx` usage). Two-pane layout (controls left, `ThemedPreview` right). Holds state `{ brand, accent, fontKey, logo }` initialized from `initialTheme`/`existingLogoKey` (default brand/accent `#8C6CFF`, fontKey `DEFAULT_FONT_KEY`). The `logo` state is `null | { kind:'upload', file, previewUrl } | { kind:'url', url } | { kind:'existing', s3Key }`. Compute the preview `theme` as `{ brand, accent, fontKey }` and `logoPreviewUrl` from the logo state's `previewUrl` (null for `existing`/none).

  **The modal is presentational — it does NOT call the API or upload anything.** It owns only form state. Footer buttons (affirmative on the right): **Use as-is** (only `mode==='adopt'` → `onApply(null)`) and **Apply** (→ `onApply(draft)`); both disabled while colors are invalid or `isSubmitting` (the single flag the parent sets to cover upload **and** mutation). On Apply it passes the raw draft up:
  ```ts
  type LogoDraft =
    | { kind: 'upload'; file: File }
    | { kind: 'url'; url: string }
    | { kind: 'existing'; s3Key: string }
    | null            // explicit remove
    | undefined       // never set (adopt, no logo)
  type WizardDraft = { brand: string; accent: string; fontKey: string; logo: LogoDraft }
  ```
  Props: `{ open, mode, templateContent, sampleData, initialTheme?, existingLogoKey?, isSubmitting, onApply: (draft: WizardDraft | null) => void, onClose }` (adopt's "Use as-is" calls `onApply(null)` meaning "no theme"; in adopt the absence of a chosen logo is `logo: undefined`).

  The **parent** (Tasks 4/5) converts a `WizardDraft` into a `ThemeInput` — this is where the upload happens, so a single `isApplying` flag covers both the `uploadLogoFile` call and the mutation. Shared helper to add in `src/lib/theme/` or inline in each parent:
  ```ts
  async function draftToThemeInput(d: WizardDraft): Promise<ThemeInput> {
    let logo: ThemeInput['logo']
    if (d.logo === undefined) logo = undefined
    else if (d.logo === null) logo = null
    else if (d.logo.kind === 'url') logo = { source: 'url', url: d.logo.url }
    else if (d.logo.kind === 'existing') logo = { source: 'upload', s3Key: d.logo.s3Key } // preserve
    else logo = { source: 'upload', s3Key: await uploadLogoFile(d.logo.file) }            // upload now
    return { brand: d.brand, accent: d.accent, fontKey: d.fontKey, ...(logo !== undefined ? { logo } : {}) }
  }
  ```
  All strings via `useTranslations('branding')`.

- [ ] **Step 7: Verify build/lint** — `npm run build && npm run lint`. Expected: passes (call sites wired in Tasks 4–5; if build fails only on the not-yet-wired marketplace/user-modal imports, that's expected and fixed next).

- [ ] **Step 8: Commit**

```bash
git add src/components/theme/ package.json package-lock.json
git commit -m "feat(theme-ui): branding wizard modal + controls + themed preview"
```

---

## Task 4: Marketplace adopt flow

**Files:** Modify the marketplace page (`src/app/[locale]/(dashboard)/marketplace/page.tsx`) and/or `TemplateCard.tsx` / `TemplatePreviewModal.tsx`.

- [ ] **Step 1: Imports** — at the top of the marketplace page add `import dynamic from 'next/dynamic'`, `import { getMarketplaceTemplatePreview, uploadLogoFile } from '@/lib/api'`, and lazy-load the wizard: `const BrandingWizardModal = dynamic(() => import('@/components/theme/BrandingWizardModal').then(m => m.BrandingWizardModal), { ssr: false })`. (`getMarketplaceTemplatePreview` is a plain async fn — call it directly in the handler; do NOT use the `useMarketplaceTemplatePreview` hook, which can't be called inside an event handler.)

- [ ] **Step 2: Wire "Use Template"** — read the current `handleUseTemplate`. Replace the immediate `copyTemplate.mutateAsync(template.templateId)` with: `const preview = await getMarketplaceTemplatePreview(template.templateId)`, safe-parse `preview.sampleDataJson` (`try/catch` → `{}`), and open the wizard holding `{ open:true, templateId, content: preview.content, sampleData }` in state. Add an `isApplying` boolean state.

- [ ] **Step 3: Wizard `onApply`** — `mode='adopt'`. The handler converts the draft and runs the upload+mutation under one flag:
```ts
const onApply = async (draft: WizardDraft | null) => {
  setIsApplying(true)
  try {
    const theme = draft ? await draftToThemeInput(draft) : undefined   // draft===null => "use as-is"
    await copyTemplate.mutateAsync({ templateId, theme })
    toast.success(...); closeWizard()
  } catch (e) { toast.error((e as Error).message) }   // upload OR mutation failure — keep wizard open
  finally { setIsApplying(false) }
}
```
Pass `isSubmitting={isApplying}` to the modal. `draftToThemeInput` is the helper from Task 3 Step 6 (uploads the logo file here). "Use as-is" (`draft === null`) sends no theme → backend renders the template's built-in defaults. Keep the existing success/error toast + list refresh.

- [ ] **Step 4: Verify** — `npm run build && npm run lint` → pass.

- [ ] **Step 5: Commit**

```bash
git add "src/app/[locale]/(dashboard)/marketplace/" src/components/marketplace/
git commit -m "feat(theme-ui): branding wizard in marketplace adopt flow"
```

---

## Task 5: Edit-branding flow

**Files:** Modify `src/components/templates/UserTemplatePreviewModal.tsx` (and its opener if needed).

- [ ] **Step 1: Add "Edit branding"** — read `UserTemplatePreviewModal.tsx`. Add an "Edit branding" button to the footer. On click, ensure the template's content + theme are available: use `useTemplate(templateId)` (returns content+theme+sourceMarketplaceId) and, for sample data, `getMarketplaceTemplatePreview(sourceMarketplaceId)` parsed `sampleDataJson` (fallback `{}`). Lazy-load `BrandingWizardModal` the same way as Task 4.

- [ ] **Step 2: Open wizard in edit mode** — pass `mode='edit'`, `initialTheme={theme}` (the stored `{brand,accent,fontKey}`), `existingLogoKey={theme?.logoKey}`, `templateContent`, `sampleData`.

- [ ] **Step 3: `onApply`** — `const updateTheme = useUpdateTemplateTheme()` + an `isApplying` flag. On `onApply(draft)` (edit mode never passes `null`): `const theme = await draftToThemeInput(draft!)` then `await updateTheme.mutateAsync({ templateId, theme })`, under one `try/finally` setting `isApplying` (covers the logo upload + the PATCH); pass `isSubmitting={isApplying}` to the modal. Toast on success/error; close on success. The logo **preservation** is handled by `draftToThemeInput`: an unchanged existing logo arrives as `{ kind:'existing', s3Key }` → `{ source:'upload', s3Key }` (re-references the stored key so the PATCH doesn't drop it); explicit remove → `null`; replacement → upload.

- [ ] **Step 4: Verify** — `npm run build && npm run lint` → pass.

- [ ] **Step 5: Commit**

```bash
git add src/components/templates/UserTemplatePreviewModal.tsx
git commit -m "feat(theme-ui): edit-branding entry on adopted templates"
```

---

## Task 6: i18n + final verification

**Files:** `src/i18n/messages/en.json`, `src/i18n/messages/es.json`.

- [ ] **Step 1: Add `branding` namespace** — add a `branding` object to both files with every key the components reference (`title`, `editTitle`, `primaryColor`, `accentColor`, `font`, `logo`, `logoUpload`, `logoUrl`, `logoUrlHttpsOnly`, `logoKept`, `remove`, `useAsIs`, `apply`, `editBranding`, `uploadError`, `urlInvalid`, …). Mirror the structure/locale style of the existing `marketplace`/`ai` namespaces. English + Spanish translations (Spanish matches the app's existing es.json tone).

- [ ] **Step 2: Lint + build (gating)** — both must pass:

```bash
cd mkpdfs-web && npm run lint && npm run build
```

- [ ] **Step 3: Manual E2E (browser)** — against dev (after merge/deploy, or `npm run dev` pointed at dev API):
  - Marketplace → Use Template → wizard opens, live preview shows the real template; change brand/accent (preview recolors), font (preview re-types), upload a logo (preview shows it) → Apply → template added; generate from dashboard → themed PDF.
  - "Use as-is" → adopts with no theme → generated PDF uses the template's built-in defaults (purple/Fraunces).
  - Edit branding on that template → colors change, **logo preserved** when untouched; save → regenerate shows new theme + same logo.
  - Edit branding → remove logo → save → regenerate shows the initial-letter mark.
  - URL logo with a non-image/invalid URL → backend rejects → toast shows the message, wizard stays open.
  - Upload that succeeds but force an apply failure (e.g. offline) → toast, no partial save.

- [ ] **Step 4: Commit**

```bash
git add src/i18n/messages/en.json src/i18n/messages/es.json
git commit -m "feat(theme-ui): branding i18n strings (en/es)"
```

---

## Self-Review

**Spec coverage:** wizard on adopt (Tasks 3–4), live real-template preview (Tasks 2–3), Edit-branding entry (Task 5), logo upload-or-URL with https rule + edit-mode preservation (Tasks 1,3,5), font registry mirror + system params + helpers matching backend (Task 2), react-colorful + lazy-load (Tasks 3–4), i18n (Task 6), verification without a test runner = `npm run build` + `npm run lint` + manual browser E2E (Tasks 2,3,6; the sanity script is optional/non-gating). Orphaned-logo gap documented (not promised). All spec sections map to a task.

**Placeholder scan:** The React component tasks (3–5) specify props, state shape, exact `WizardDraft`→`ThemeInput` construction (`draftToThemeInput`), and the files/patterns to match, but defer pixel-level JSX to the implementer (matching existing Tailwind/Radix conventions) rather than inventing markup that may clash with the design system — the contracts and the wiring code are concrete. No `TBD`/`TODO`.

**Type consistency:** `ThemeInput {brand,accent,fontKey,logo?}` and `Theme {…,logoKey?}` (Task 1) are used identically in api/hooks (Task 1), `composeThemedHtml` (Task 2), and `draftToThemeInput` (Task 3). API fn renamed to `copyMarketplaceTemplate(templateId, theme?)` (Task 1); `useCopyMarketplaceTemplate` takes `{templateId, theme?}` (Task 1) and is called that way (Task 4). `useUpdateTemplateTheme` takes `{templateId, theme}` (Task 1), called in Task 5. The modal emits a `WizardDraft`; the parents (Tasks 4/5) convert it via `draftToThemeInput`, which is the sole caller of `uploadLogoFile(file): Promise<string>`.

## Note for executor
`mkpdfs-web` has **no unit-test runner** and its Node is **v20** (no `--experimental-strip-types`); do NOT add vitest/jest, and treat the Task 2 sanity script as optional reference only. The gating verification is `npm run lint && npm run build` (full typecheck) + the manual browser E2E. Verify against the dev API (the theming backend is already deployed there) — `npm run dev` reads `NEXT_PUBLIC_API_URL`; point it at `https://dev.apis.mkpdfs.com`. Create a feature branch off `dev` first; finish via the finishing-a-development-branch flow (merge to `dev` → Amplify deploys the dashboard).
