# Template Theming + Predefined Params — Backend Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Let a user brand a marketplace template (primary color, accent color, font, logo) at adoption time and re-edit it later, applied at PDF render time, plus expose predefined params (`{{today}}`, `{{now}}`, `{{year}}`) in every template.

**Architecture:** Theme is validated structured data stored on the user's template row. The compiled Handlebars template stays theme-agnostic and is cached by S3 content version. At render, `PdfService` reads the row, passes the resolved logo via a Handlebars runtime `data` frame, merges reserved system params into the context, then injects a sanitized `:root` CSS override + font `<link>` before `</head>`. Logos converge to a private S3 object (URL logos are fetched server-side with SSRF guards at save time) and are inlined as `data:` URIs at render. No theme on a row ⇒ the template renders exactly as today (marketplace previews unaffected).

**Tech Stack:** TypeScript, AWS Lambda (Middy), DynamoDB, S3 (versioning enabled), Handlebars, puppeteer-core + Sparticuz Chromium layer, AWS CDK, Vitest + `aws-sdk-client-mock`.

**Scope:** Backend only (`mkpdfs-backend/`). The web UI (branding wizard + theme editor) is a separate follow-up plan that consumes these APIs.

---

## File Structure

**New (pure logic — `mkpdfs-backend/src/libs/theme/`):**
- `themeTypes.ts` — `Theme` (stored), `ThemeInput` (from client), `LogoInput` types.
- `fonts.ts` — curated font registry `FONTS` (`fontKey → { label, linkHref, headingStack, bodyStack }`) + `isFontKey()`. Single source of truth for render; mirrored in the web plan.
- `colorDerive.ts` — `softTint(hex)`, `shadowRgba(hex, alpha)` derived tokens.
- `validateTheme.ts` — `validateThemeFields(input)` → sanitized `{brand, accent, fontKey}` or throws `ThemeValidationError`.
- `buildThemeStyle.ts` — `buildThemeHead(theme)` → injected `<link>` + `<style>:root{…}</style>` string.
- `injectTheme.ts` — `injectIntoHead(html, fragment)` inserts before `</head>`.
- `logoIngest.ts` — `fetchLogoFromUrl(url)` SSRF-guarded fetch → `{buffer, contentType, ext}`; `MAX_LOGO_BYTES`, `LOGO_CONTENT_TYPES`.

**New (other):**
- `mkpdfs-backend/src/libs/systemParams.ts` — `buildSystemParams(now: Date)` → `{today, now, year}`.
- `mkpdfs-backend/src/functions/templates/logoUploadUrl/handler.ts` — presigned PUT for a logo file. Route `POST /templates/logo-upload-url`.
- `mkpdfs-backend/src/functions/templates/updateTheme/handler.ts` — `PATCH /templates/{templateId}/theme`.

**Modified:**
- `src/libs/services/pdfService.ts` — read row, version-keyed + LRU cache, `mkpdfsLogo` helper, theme `data` frame, system params, logo→data-URI, head injection, `document.fonts.ready`.
- `src/functions/marketplace/useTemplate/handler.ts` — accept/validate/persist `theme`; store `contentVersion`.
- `src/functions/templates/updateTemplate/handler.ts` — capture S3 `VersionId` into `contentVersion`.
- `src/functions/templates/uploadTemplate/handler.ts` — capture S3 `VersionId` into `contentVersion`.
- `cdk/lib/stacks/api-stack.ts` — grants + 2 new routes.
- `cdk/lib/stacks/jobs-stack.ts` — `templates.grantReadData(processJobFn)`.
- `scripts/marketplace-templates/*.hbs` (all 13) — normalize to the theme-token contract + `{{mkpdfsLogo}}` slot.

---

## Task 1: Font registry + theme types

**Files:**
- Create: `src/libs/theme/themeTypes.ts`
- Create: `src/libs/theme/fonts.ts`
- Test: `src/libs/theme/fonts.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
// src/libs/theme/fonts.test.ts
import { describe, expect, it } from 'vitest';
import { FONTS, isFontKey, DEFAULT_FONT_KEY } from './fonts';

describe('font registry', () => {
  it('every font has a label, https google-fonts link, and non-empty stacks', () => {
    for (const [key, f] of Object.entries(FONTS)) {
      expect(f.label, key).toBeTruthy();
      expect(f.linkHref, key).toMatch(/^https:\/\/fonts\.googleapis\.com\/css2\?/);
      expect(f.headingStack, key).toContain(',');
      expect(f.bodyStack, key).toContain(',');
    }
  });

  it('isFontKey accepts registry keys and rejects others', () => {
    expect(isFontKey(DEFAULT_FONT_KEY)).toBe(true);
    expect(isFontKey('inter-fraunces')).toBe(true);
    expect(isFontKey('not-a-font')).toBe(false);
    expect(isFontKey('')).toBe(false);
    expect(isFontKey(undefined as any)).toBe(false);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd mkpdfs-backend && npx vitest run src/libs/theme/fonts.test.ts`
Expected: FAIL — cannot find module `./fonts`.

- [ ] **Step 3: Write the implementation**

```ts
// src/libs/theme/themeTypes.ts

/** Theme as stored on a template row. Logo is always a private S3 key. */
export interface Theme {
  brand: string;     // "#RRGGBB"
  accent: string;    // "#RRGGBB"
  fontKey: string;   // a key of FONTS
  logoKey?: string;  // e.g. "users/{userId}/logos/{id}.png"
}

/** Logo as supplied by the client (before server-side resolution to S3). */
export type LogoInput =
  | { source: 'upload'; s3Key: string }  // already uploaded via presigned PUT
  | { source: 'url'; url: string }       // remote URL, ingested server-side
  | null;

/** Theme payload accepted from the client. */
export interface ThemeInput {
  brand: string;
  accent: string;
  fontKey: string;
  logo?: LogoInput;
}
```

```ts
// src/libs/theme/fonts.ts

export interface FontDef {
  label: string;
  linkHref: string;     // Google Fonts CSS2 stylesheet URL
  headingStack: string; // CSS font-family value for headings
  bodyStack: string;    // CSS font-family value for body
}

const SANS = `-apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif`;
const SERIF = `Georgia, 'Times New Roman', serif`;

export const FONTS: Record<string, FontDef> = {
  'inter-fraunces': {
    label: 'Inter + Fraunces',
    linkHref:
      'https://fonts.googleapis.com/css2?family=Fraunces:opsz,wght@9..144,400;9..144,500;9..144,600&family=Inter:wght@400;500;600;700&display=swap',
    headingStack: `'Fraunces', ${SERIF}`,
    bodyStack: `'Inter', ${SANS}`,
  },
  'inter-inter': {
    label: 'Inter',
    linkHref: 'https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700;800&display=swap',
    headingStack: `'Inter', ${SANS}`,
    bodyStack: `'Inter', ${SANS}`,
  },
  'playfair-lato': {
    label: 'Playfair Display + Lato',
    linkHref:
      'https://fonts.googleapis.com/css2?family=Playfair+Display:wght@500;600;700&family=Lato:wght@400;700&display=swap',
    headingStack: `'Playfair Display', ${SERIF}`,
    bodyStack: `'Lato', ${SANS}`,
  },
  'poppins-poppins': {
    label: 'Poppins',
    linkHref: 'https://fonts.googleapis.com/css2?family=Poppins:wght@400;500;600;700&display=swap',
    headingStack: `'Poppins', ${SANS}`,
    bodyStack: `'Poppins', ${SANS}`,
  },
  'montserrat-opensans': {
    label: 'Montserrat + Open Sans',
    linkHref:
      'https://fonts.googleapis.com/css2?family=Montserrat:wght@500;600;700&family=Open+Sans:wght@400;600&display=swap',
    headingStack: `'Montserrat', ${SANS}`,
    bodyStack: `'Open Sans', ${SANS}`,
  },
  'lora-source-sans': {
    label: 'Lora + Source Sans 3',
    linkHref:
      'https://fonts.googleapis.com/css2?family=Lora:wght@500;600;700&family=Source+Sans+3:wght@400;600&display=swap',
    headingStack: `'Lora', ${SERIF}`,
    bodyStack: `'Source Sans 3', ${SANS}`,
  },
  'space-grotesk-inter': {
    label: 'Space Grotesk + Inter',
    linkHref:
      'https://fonts.googleapis.com/css2?family=Space+Grotesk:wght@500;600;700&family=Inter:wght@400;500;600&display=swap',
    headingStack: `'Space Grotesk', ${SANS}`,
    bodyStack: `'Inter', ${SANS}`,
  },
  'dmserif-dmsans': {
    label: 'DM Serif Display + DM Sans',
    linkHref:
      'https://fonts.googleapis.com/css2?family=DM+Serif+Display&family=DM+Sans:wght@400;500;700&display=swap',
    headingStack: `'DM Serif Display', ${SERIF}`,
    bodyStack: `'DM Sans', ${SANS}`,
  },
};

export const DEFAULT_FONT_KEY = 'inter-fraunces';

export function isFontKey(key: unknown): key is string {
  return typeof key === 'string' && Object.prototype.hasOwnProperty.call(FONTS, key);
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd mkpdfs-backend && npx vitest run src/libs/theme/fonts.test.ts`
Expected: PASS (2 tests).

- [ ] **Step 5: Commit**

```bash
git add src/libs/theme/themeTypes.ts src/libs/theme/fonts.ts src/libs/theme/fonts.test.ts
git commit -m "feat(theme): curated font registry + theme types"
```

---

## Task 2: Derived color tokens

**Files:**
- Create: `src/libs/theme/colorDerive.ts`
- Test: `src/libs/theme/colorDerive.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
// src/libs/theme/colorDerive.test.ts
import { describe, expect, it } from 'vitest';
import { softTint, shadowRgba } from './colorDerive';

describe('colorDerive', () => {
  it('softTint blends the color 8% over white → near-white hex', () => {
    // pure black at 8% over white = #ebebeb (0.92*255 ≈ 235)
    expect(softTint('#000000')).toBe('#ebebeb');
    // white stays white
    expect(softTint('#ffffff')).toBe('#ffffff');
  });

  it('softTint accepts 3-digit hex', () => {
    expect(softTint('#000')).toBe('#ebebeb');
  });

  it('shadowRgba produces an rgba() string from a hex + alpha', () => {
    expect(shadowRgba('#8C6CFF', 0.28)).toBe('rgba(140, 108, 255, 0.28)');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd mkpdfs-backend && npx vitest run src/libs/theme/colorDerive.test.ts`
Expected: FAIL — cannot find module `./colorDerive`.

- [ ] **Step 3: Write the implementation**

```ts
// src/libs/theme/colorDerive.ts

/** Parse "#RGB" or "#RRGGBB" into [r,g,b] (0-255). Assumes already validated. */
function hexToRgb(hex: string): [number, number, number] {
  let h = hex.replace('#', '');
  if (h.length === 3) h = h.split('').map((c) => c + c).join('');
  return [
    parseInt(h.slice(0, 2), 16),
    parseInt(h.slice(2, 4), 16),
    parseInt(h.slice(4, 6), 16),
  ];
}

function toHex(n: number): string {
  return Math.round(n).toString(16).padStart(2, '0');
}

/** Blend `hex` `weight` (default 0.08) over white → a soft tint hex. */
export function softTint(hex: string, weight = 0.08): string {
  const [r, g, b] = hexToRgb(hex);
  const mix = (c: number) => c * weight + 255 * (1 - weight);
  return `#${toHex(mix(r))}${toHex(mix(g))}${toHex(mix(b))}`;
}

/** rgba() string for soft shadows from a hex color. */
export function shadowRgba(hex: string, alpha: number): string {
  const [r, g, b] = hexToRgb(hex);
  return `rgba(${r}, ${g}, ${b}, ${alpha})`;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd mkpdfs-backend && npx vitest run src/libs/theme/colorDerive.test.ts`
Expected: PASS (3 tests).

- [ ] **Step 5: Commit**

```bash
git add src/libs/theme/colorDerive.ts src/libs/theme/colorDerive.test.ts
git commit -m "feat(theme): derived soft-tint and shadow color tokens"
```

---

## Task 3: Theme field validation

**Files:**
- Create: `src/libs/theme/validateTheme.ts`
- Test: `src/libs/theme/validateTheme.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
// src/libs/theme/validateTheme.test.ts
import { describe, expect, it } from 'vitest';
import { validateThemeFields, ThemeValidationError } from './validateTheme';

describe('validateThemeFields', () => {
  it('accepts valid hex + fontKey and normalizes 3-digit hex to 6-digit lowercase', () => {
    const r = validateThemeFields({ brand: '#FFF', accent: '#8C6CFF', fontKey: 'inter-inter' });
    expect(r).toEqual({ brand: '#ffffff', accent: '#8c6cff', fontKey: 'inter-inter' });
  });

  it('rejects non-hex colors (css-injection guards)', () => {
    for (const bad of ['red', 'rgb(0,0,0)', 'var(--x)', '#12', '#1234567', 'url(x)', '#ff;}', '']) {
      expect(() => validateThemeFields({ brand: bad, accent: '#000000', fontKey: 'inter-inter' }),
        bad).toThrow(ThemeValidationError);
    }
  });

  it('rejects an unknown fontKey', () => {
    expect(() => validateThemeFields({ brand: '#000000', accent: '#000000', fontKey: 'comic' }))
      .toThrow(ThemeValidationError);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd mkpdfs-backend && npx vitest run src/libs/theme/validateTheme.test.ts`
Expected: FAIL — cannot find module `./validateTheme`.

- [ ] **Step 3: Write the implementation**

```ts
// src/libs/theme/validateTheme.ts
import { isFontKey } from './fonts';

export class ThemeValidationError extends Error {
  constructor(message: string) {
    super(message);
    this.name = 'ThemeValidationError';
  }
}

const HEX_RE = /^#(?:[0-9a-fA-F]{3}|[0-9a-fA-F]{6})$/;

function normalizeHex(value: unknown, field: string): string {
  if (typeof value !== 'string' || !HEX_RE.test(value)) {
    throw new ThemeValidationError(`Invalid ${field}: must be a hex color like #RRGGBB`);
  }
  let h = value.slice(1).toLowerCase();
  if (h.length === 3) h = h.split('').map((c) => c + c).join('');
  return `#${h}`;
}

/** Validate + normalize the color/font fields. Logo is handled separately. */
export function validateThemeFields(input: {
  brand: unknown;
  accent: unknown;
  fontKey: unknown;
}): { brand: string; accent: string; fontKey: string } {
  const brand = normalizeHex(input.brand, 'brand');
  const accent = normalizeHex(input.accent, 'accent');
  if (!isFontKey(input.fontKey)) {
    throw new ThemeValidationError('Invalid fontKey: not a recognized font');
  }
  return { brand, accent, fontKey: input.fontKey };
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd mkpdfs-backend && npx vitest run src/libs/theme/validateTheme.test.ts`
Expected: PASS (3 tests).

- [ ] **Step 5: Commit**

```bash
git add src/libs/theme/validateTheme.ts src/libs/theme/validateTheme.test.ts
git commit -m "feat(theme): strict theme field validation"
```

---

## Task 4: Build the injected theme `<head>` fragment

**Files:**
- Create: `src/libs/theme/buildThemeStyle.ts`
- Test: `src/libs/theme/buildThemeStyle.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
// src/libs/theme/buildThemeStyle.test.ts
import { describe, expect, it } from 'vitest';
import { buildThemeHead } from './buildThemeStyle';

describe('buildThemeHead', () => {
  const head = buildThemeHead({ brand: '#8c6cff', accent: '#ff6b35', fontKey: 'inter-fraunces' });

  it('emits a google-fonts stylesheet link for the chosen font', () => {
    expect(head).toContain('<link rel="stylesheet" href="https://fonts.googleapis.com/css2?');
    expect(head).toContain('Fraunces');
  });

  it('emits a :root override with brand/accent + derived tokens + font stacks', () => {
    expect(head).toContain('--brand: #8c6cff;');
    expect(head).toContain('--accent: #ff6b35;');
    expect(head).toContain('--brand-soft:');
    expect(head).toContain('--brand-shadow: rgba(140, 108, 255, 0.28);');
    expect(head).toContain('--accent-soft:');
    expect(head).toContain("--font-heading: 'Fraunces'");
    expect(head).toContain("--font-body: 'Inter'");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd mkpdfs-backend && npx vitest run src/libs/theme/buildThemeStyle.test.ts`
Expected: FAIL — cannot find module `./buildThemeStyle`.

- [ ] **Step 3: Write the implementation**

```ts
// src/libs/theme/buildThemeStyle.ts
import { FONTS } from './fonts';
import { softTint, shadowRgba } from './colorDerive';

/**
 * Build the HTML to inject before </head>: a Google Fonts <link> for the chosen
 * font and a :root override block. Inputs are already validated/normalized
 * (validateThemeFields), so they are safe to interpolate into CSS.
 */
export function buildThemeHead(theme: { brand: string; accent: string; fontKey: string }): string {
  const font = FONTS[theme.fontKey] ?? FONTS['inter-fraunces'];
  const vars = [
    `--brand: ${theme.brand};`,
    `--brand-soft: ${softTint(theme.brand)};`,
    `--brand-shadow: ${shadowRgba(theme.brand, 0.28)};`,
    `--accent: ${theme.accent};`,
    `--accent-soft: ${softTint(theme.accent)};`,
    `--font-heading: ${font.headingStack};`,
    `--font-body: ${font.bodyStack};`,
  ].join(' ');
  return (
    `<link rel="stylesheet" href="${font.linkHref}">` +
    `<style id="mkpdfs-theme">:root { ${vars} }</style>`
  );
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd mkpdfs-backend && npx vitest run src/libs/theme/buildThemeStyle.test.ts`
Expected: PASS (2 tests).

- [ ] **Step 5: Commit**

```bash
git add src/libs/theme/buildThemeStyle.ts src/libs/theme/buildThemeStyle.test.ts
git commit -m "feat(theme): build injected theme head fragment"
```

---

## Task 5: Inject the fragment into rendered HTML

**Files:**
- Create: `src/libs/theme/injectTheme.ts`
- Test: `src/libs/theme/injectTheme.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
// src/libs/theme/injectTheme.test.ts
import { describe, expect, it } from 'vitest';
import { injectIntoHead } from './injectTheme';

describe('injectIntoHead', () => {
  it('inserts the fragment immediately before the first </head>', () => {
    const html = '<html><head><style>:root{--brand:#000}</style></head><body>x</body></html>';
    const out = injectIntoHead(html, '<style id="t">Z</style>');
    expect(out).toContain('</style><style id="t">Z</style></head>');
    expect(out.indexOf('id="t"')).toBeLessThan(out.indexOf('</head>'));
  });

  it('is case-insensitive on the </head> tag', () => {
    const out = injectIntoHead('<HEAD></HEAD>', 'FRAG');
    expect(out).toContain('FRAG</HEAD>');
  });

  it('falls back to prepending when there is no </head>', () => {
    const out = injectIntoHead('<body>no head</body>', 'FRAG');
    expect(out.startsWith('FRAG')).toBe(true);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd mkpdfs-backend && npx vitest run src/libs/theme/injectTheme.test.ts`
Expected: FAIL — cannot find module `./injectTheme`.

- [ ] **Step 3: Write the implementation**

```ts
// src/libs/theme/injectTheme.ts

/** Insert `fragment` immediately before the first </head> (case-insensitive). */
export function injectIntoHead(html: string, fragment: string): string {
  const idx = html.search(/<\/head>/i);
  if (idx === -1) return fragment + html;
  return html.slice(0, idx) + fragment + html.slice(idx);
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd mkpdfs-backend && npx vitest run src/libs/theme/injectTheme.test.ts`
Expected: PASS (3 tests).

- [ ] **Step 5: Commit**

```bash
git add src/libs/theme/injectTheme.ts src/libs/theme/injectTheme.test.ts
git commit -m "feat(theme): inject theme fragment into rendered head"
```

---

## Task 6: Predefined system params

**Files:**
- Create: `src/libs/systemParams.ts`
- Test: `src/libs/systemParams.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
// src/libs/systemParams.test.ts
import { describe, expect, it } from 'vitest';
import { buildSystemParams } from './systemParams';

describe('buildSystemParams', () => {
  it('returns today (YYYY-MM-DD), now (ISO) and year from the given clock', () => {
    const d = new Date('2026-06-16T13:45:00.000Z');
    expect(buildSystemParams(d)).toEqual({
      today: '2026-06-16',
      now: '2026-06-16T13:45:00.000Z',
      year: 2026,
    });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd mkpdfs-backend && npx vitest run src/libs/systemParams.test.ts`
Expected: FAIL — cannot find module `./systemParams`.

- [ ] **Step 3: Write the implementation**

```ts
// src/libs/systemParams.ts

export interface SystemParams {
  today: string; // YYYY-MM-DD (UTC)
  now: string;   // ISO 8601
  year: number;
}

/** Reserved params merged into every render context. Generated once per request. */
export function buildSystemParams(now: Date): SystemParams {
  return {
    today: now.toISOString().slice(0, 10),
    now: now.toISOString(),
    year: now.getUTCFullYear(),
  };
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd mkpdfs-backend && npx vitest run src/libs/systemParams.test.ts`
Expected: PASS (1 test).

- [ ] **Step 5: Commit**

```bash
git add src/libs/systemParams.ts src/libs/systemParams.test.ts
git commit -m "feat: predefined system params (today/now/year)"
```

---

## Task 7: SSRF-guarded logo URL ingest

**Files:**
- Create: `src/libs/theme/logoIngest.ts`
- Test: `src/libs/theme/logoIngest.test.ts`

This module fetches a remote logo URL safely (at save time only). It blocks non-HTTPS, private/loopback/link-local hosts, oversized bodies, and non-image content types.

- [ ] **Step 1: Write the failing test**

```ts
// src/libs/theme/logoIngest.test.ts
import { describe, expect, it } from 'vitest';
import { assertSafeLogoUrl, LogoIngestError, extFromContentType } from './logoIngest';

describe('assertSafeLogoUrl', () => {
  it('accepts an https URL with a public host', () => {
    expect(() => assertSafeLogoUrl('https://cdn.example.com/logo.png')).not.toThrow();
  });

  it('rejects non-https schemes', () => {
    expect(() => assertSafeLogoUrl('http://example.com/x.png')).toThrow(LogoIngestError);
    expect(() => assertSafeLogoUrl('file:///etc/passwd')).toThrow(LogoIngestError);
    expect(() => assertSafeLogoUrl('data:image/png;base64,AAAA')).toThrow(LogoIngestError);
  });

  it('rejects localhost / private / link-local / metadata hosts', () => {
    for (const u of [
      'https://localhost/a.png',
      'https://127.0.0.1/a.png',
      'https://10.0.0.5/a.png',
      'https://192.168.1.1/a.png',
      'https://169.254.169.254/latest/meta-data',
      'https://[::1]/a.png',
    ]) {
      expect(() => assertSafeLogoUrl(u), u).toThrow(LogoIngestError);
    }
  });
});

describe('extFromContentType', () => {
  it('maps allowed image content types to extensions', () => {
    expect(extFromContentType('image/png')).toBe('png');
    expect(extFromContentType('image/jpeg')).toBe('jpg');
    expect(extFromContentType('image/svg+xml')).toBe('svg');
    expect(extFromContentType('image/webp')).toBe('webp');
  });
  it('throws on a disallowed content type', () => {
    expect(() => extFromContentType('text/html')).toThrow(LogoIngestError);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd mkpdfs-backend && npx vitest run src/libs/theme/logoIngest.test.ts`
Expected: FAIL — cannot find module `./logoIngest`.

- [ ] **Step 3: Write the implementation**

```ts
// src/libs/theme/logoIngest.ts

export class LogoIngestError extends Error {
  constructor(message: string) {
    super(message);
    this.name = 'LogoIngestError';
  }
}

export const MAX_LOGO_BYTES = 512 * 1024; // 512 KB cap (data-URI inlined at render)

const CONTENT_TYPE_EXT: Record<string, string> = {
  'image/png': 'png',
  'image/jpeg': 'jpg',
  'image/webp': 'webp',
  'image/svg+xml': 'svg',
};

export function extFromContentType(contentType: string): string {
  const ct = contentType.split(';')[0].trim().toLowerCase();
  const ext = CONTENT_TYPE_EXT[ct];
  if (!ext) throw new LogoIngestError(`Unsupported logo content type: ${ct}`);
  return ext;
}

/** Block obviously-internal hosts. Defense in depth (Lambda has no VPC route to metadata, but be strict). */
export function assertSafeLogoUrl(raw: string): URL {
  let url: URL;
  try {
    url = new URL(raw);
  } catch {
    throw new LogoIngestError('Invalid logo URL');
  }
  if (url.protocol !== 'https:') throw new LogoIngestError('Logo URL must be https');
  const host = url.hostname.toLowerCase().replace(/^\[|\]$/g, '');
  if (
    host === 'localhost' ||
    host === '::1' ||
    host.endsWith('.localhost') ||
    /^127\./.test(host) ||
    /^10\./.test(host) ||
    /^192\.168\./.test(host) ||
    /^169\.254\./.test(host) ||
    /^fe80:/i.test(host) ||
    /^fc00:/i.test(host) ||
    /^fd[0-9a-f]{2}:/i.test(host) ||
    /^172\.(1[6-9]|2[0-9]|3[0-1])\./.test(host)
  ) {
    throw new LogoIngestError('Logo URL host is not allowed');
  }
  return url;
}

export interface IngestedLogo {
  buffer: Buffer;
  contentType: string;
  ext: string;
}

/** Fetch + validate a remote logo. HTTPS only, public host, size & type capped. */
export async function fetchLogoFromUrl(raw: string): Promise<IngestedLogo> {
  assertSafeLogoUrl(raw);
  const controller = new AbortController();
  const timeout = setTimeout(() => controller.abort(), 5000);
  try {
    const res = await fetch(raw, { redirect: 'error', signal: controller.signal });
    if (!res.ok) throw new LogoIngestError(`Logo fetch failed: HTTP ${res.status}`);
    const contentType = res.headers.get('content-type') || '';
    const ext = extFromContentType(contentType);
    const buf = Buffer.from(await res.arrayBuffer());
    if (buf.length === 0) throw new LogoIngestError('Logo is empty');
    if (buf.length > MAX_LOGO_BYTES) {
      throw new LogoIngestError(`Logo exceeds ${MAX_LOGO_BYTES} bytes`);
    }
    return { buffer: buf, contentType: contentType.split(';')[0].trim().toLowerCase(), ext };
  } catch (err) {
    if (err instanceof LogoIngestError) throw err;
    throw new LogoIngestError(`Logo fetch failed: ${(err as Error).message}`);
  } finally {
    clearTimeout(timeout);
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd mkpdfs-backend && npx vitest run src/libs/theme/logoIngest.test.ts`
Expected: PASS (5 tests).

- [ ] **Step 5: Commit**

```bash
git add src/libs/theme/logoIngest.ts src/libs/theme/logoIngest.test.ts
git commit -m "feat(theme): SSRF-guarded logo URL ingest"
```

---

## Task 8: Apply theme + system params in PdfService

**Files:**
- Modify: `src/libs/services/pdfService.ts`
- Test: `src/libs/services/pdfService.theme.test.ts`

This is the core render change. The compiled-template cache is keyed on the row's `contentVersion` (S3 VersionId; fallback `updatedAt`), bounded by an LRU cap, and never includes theme. Theme + logo flow through a Handlebars runtime `data` frame; system params merge into the context.

- [ ] **Step 1: Write the failing test**

We test the new pure pieces of the render path without launching Chromium by extracting them into a testable static method `composeHtml`. Add the test:

```ts
// src/libs/services/pdfService.theme.test.ts
import { describe, expect, it } from 'vitest';
import Handlebars from 'handlebars';
import { PdfService } from './pdfService';

describe('PdfService.composeHtml', () => {
  const tpl = Handlebars.compile(
    '<html><head><style>:root{--brand:#000}</style></head>' +
    '<body>{{mkpdfsLogo companyName}}|{{today}}|{{companyName}}</body></html>',
  );

  it('with no theme: renders defaults, no injected theme style, system params present', () => {
    const html = PdfService.composeHtml(tpl, { companyName: 'Acme' }, undefined,
      { today: '2026-06-16', now: 'x', year: 2026 });
    expect(html).not.toContain('id="mkpdfs-theme"');
    expect(html).toContain('<div class="brand-dot">A</div>'); // fallback mark
    expect(html).toContain('|2026-06-16|Acme');
  });

  it('with theme: injects :root override + font link, logo becomes an <img> data-uri', () => {
    const html = PdfService.composeHtml(
      tpl,
      { companyName: 'Acme' },
      { brand: '#8c6cff', accent: '#ff6b35', fontKey: 'inter-inter',
        logoDataUri: 'data:image/png;base64,AAAA' },
      { today: '2026-06-16', now: 'x', year: 2026 },
    );
    expect(html).toContain('id="mkpdfs-theme"');
    expect(html).toContain('--brand: #8c6cff;');
    expect(html).toContain('<link rel="stylesheet" href="https://fonts.googleapis.com');
    expect(html).toContain('<img class="brand-logo" src="data:image/png;base64,AAAA"');
    expect(html).not.toContain('brand-dot');
  });

  it('batch arrays: each page gets system params and the theme', () => {
    const html = PdfService.composeHtml(
      tpl,
      [{ companyName: 'A' }, { companyName: 'B' }],
      { brand: '#000000', accent: '#000000', fontKey: 'inter-inter', logoDataUri: null },
      { today: '2026-06-16', now: 'x', year: 2026 },
    );
    expect(html).toContain('|2026-06-16|A');
    expect(html).toContain('|2026-06-16|B');
    expect(html).toContain('page-break-after');
    expect(html.match(/id="mkpdfs-theme"/g)).toHaveLength(1); // injected once
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd mkpdfs-backend && npx vitest run src/libs/services/pdfService.theme.test.ts`
Expected: FAIL — `PdfService.composeHtml is not a function` and the `mkpdfsLogo` helper is unregistered.

- [ ] **Step 3: Implement the changes**

In `src/libs/services/pdfService.ts`:

(a) Add imports near the top (after existing imports):

```ts
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient, GetCommand } from '@aws-sdk/lib-dynamodb';
import { Theme } from '../theme/themeTypes';
import { buildThemeHead } from '../theme/buildThemeStyle';
import { injectIntoHead } from '../theme/injectTheme';
import { buildSystemParams, SystemParams } from '../systemParams';
```

(b) Add a DDB client + cache bound near the other module-level constants:

```ts
const ddbDocClient = DynamoDBDocumentClient.from(new DynamoDBClient({}));
const MAX_CACHED_TEMPLATES = 100;

interface ResolvedTheme {
  brand: string;
  accent: string;
  fontKey: string;
  logoDataUri: string | null;
}
```

(c) Replace the constructor's helper registration to also register `mkpdfsLogo`. Inside `registerHandlebarsHelpers()`, add:

```ts
    Handlebars.registerHelper('mkpdfsLogo', function (this: any, name: any, options: any) {
      const theme = options?.data?.mkpdfsTheme as ResolvedTheme | undefined;
      if (theme && theme.logoDataUri) {
        return new Handlebars.SafeString(
          `<img class="brand-logo" src="${theme.logoDataUri}" alt="">`,
        );
      }
      const initial = (typeof name === 'string' && name.trim() ? name.trim()[0] : '') || '';
      return new Handlebars.SafeString(
        `<div class="brand-dot">${Handlebars.escapeExpression(initial)}</div>`,
      );
    });
```

(d) Add the static `composeHtml` method (pure; used by `generatePdf` and by tests):

```ts
  /**
   * Render the compiled template with system params + theme, then inject the
   * theme <head> fragment. Pure string work — no browser, no I/O.
   */
  static composeHtml(
    compiled: TemplateDelegate,
    data: any,
    resolvedTheme: ResolvedTheme | undefined,
    systemParams: SystemParams,
  ): string {
    const runtime = resolvedTheme ? { data: { mkpdfsTheme: resolvedTheme } } : undefined;
    const renderOne = (item: any) =>
      compiled(typeof item === 'object' && item !== null ? { ...item, ...systemParams } : item, runtime);

    let html: string;
    if (Array.isArray(data)) {
      html = data.map(renderOne).join('<div style="page-break-after: always;"></div>');
    } else {
      html = renderOne(data);
    }

    if (resolvedTheme) {
      html = injectIntoHead(html, buildThemeHead(resolvedTheme));
    }
    return html;
  }
```

(e) Rewrite `generatePdf` to read the row, resolve theme, and use `composeHtml`. Replace the existing top of `generatePdf` (the compiled-template + html block, lines ~62-74) with:

```ts
    const { userId, templateId, data, sendEmail } = options;

    // Read the template row (theme + content version) — falls through to
    // S3-only behaviour if the row is missing (legacy/direct uploads).
    const row = await this.getTemplateRow(userId, templateId);
    const contentVersion = row?.contentVersion || row?.updatedAt || 'v0';

    const compiledTemplate = await this.getCompiledTemplate(userId, templateId, contentVersion);
    const resolvedTheme = await this.resolveTheme(row?.theme as Theme | undefined);
    const systemParams = buildSystemParams(new Date());

    const html = PdfService.composeHtml(compiledTemplate, data, resolvedTheme, systemParams);
```

(f) Add the row read, theme resolution, and update the cache signature. Add these private methods and change `getCompiledTemplate`:

```ts
  private async getTemplateRow(userId: string, templateId: string): Promise<any | undefined> {
    try {
      const res = await ddbDocClient.send(new GetCommand({
        TableName: process.env.TEMPLATES_TABLE!,
        Key: { userId, templateId },
      }));
      return res.Item;
    } catch (err) {
      console.error('[pdfService] failed to read template row:', err);
      return undefined; // render with defaults rather than fail the PDF
    }
  }

  /** Resolve a stored Theme into render-ready values (logo → data URI). */
  private async resolveTheme(theme?: Theme): Promise<ResolvedTheme | undefined> {
    if (!theme) return undefined;
    let logoDataUri: string | null = null;
    if (theme.logoKey) {
      try {
        const obj = await s3Client.send(new GetObjectCommand({
          Bucket: process.env.ASSETS_BUCKET!,
          Key: theme.logoKey,
        }));
        const bytes = await obj.Body!.transformToByteArray();
        const contentType = obj.ContentType || 'image/png';
        logoDataUri = `data:${contentType};base64,${Buffer.from(bytes).toString('base64')}`;
      } catch (err) {
        console.error('[pdfService] failed to load logo, falling back to mark:', err);
      }
    }
    return { brand: theme.brand, accent: theme.accent, fontKey: theme.fontKey, logoDataUri };
  }
```

Change `getCompiledTemplate` to accept the version, drop the TTL, and bound the cache:

```ts
  private async getCompiledTemplate(
    userId: string,
    templateId: string,
    contentVersion: string,
  ): Promise<TemplateDelegate> {
    const cacheKey = `${userId}:${templateId}:${contentVersion}`;
    const cached = templateCache.get(cacheKey);
    if (cached) return cached.compiled;

    const templateKey = `${userId}/templates/${templateId}.hbs`;
    let templateContent: string;
    try {
      const templateResponse = await s3Client.send(new GetObjectCommand({
        Bucket: process.env.ASSETS_BUCKET!,
        Key: templateKey,
      }));
      templateContent = await this.streamToString(templateResponse.Body as Readable);
    } catch (error: any) {
      if (error.name === 'NoSuchKey') throw new Error(`Template not found: ${templateId}`);
      throw error;
    }

    const compiled = Handlebars.compile(templateContent);
    templateCache.set(cacheKey, { compiled, timestamp: Date.now() });
    // Bound the cache (LRU-ish: Map preserves insertion order)
    if (templateCache.size > MAX_CACHED_TEMPLATES) {
      const oldest = templateCache.keys().next().value;
      if (oldest) templateCache.delete(oldest);
    }
    return compiled;
  }
```

(g) Add `'document.fonts.ready'` wait in `generatePdfFromHtml`, right after `emulateMediaType`:

```ts
      await page.emulateMediaType('screen');
      await page.evaluate(() => (document as any).fonts?.ready).catch(() => undefined);
```

(h) The `TEMPLATE_CACHE_TTL` constant is now unused — delete its declaration (line ~17).

- [ ] **Step 4: Run the theme test + full suite**

Run: `cd mkpdfs-backend && npx vitest run src/libs/services/pdfService.theme.test.ts && npm run typecheck`
Expected: theme test PASS (3 tests); typecheck PASS.

- [ ] **Step 5: Commit**

```bash
git add src/libs/services/pdfService.ts src/libs/services/pdfService.theme.test.ts
git commit -m "feat(theme): apply theme + system params at render in PdfService"
```

---

## Task 9: Logo upload presigned-URL endpoint

**Files:**
- Create: `src/functions/templates/logoUploadUrl/handler.ts`
- Create: `src/functions/templates/logoUploadUrl/index.ts`
- Test: `src/functions/templates/logoUploadUrl/handler.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
// src/functions/templates/logoUploadUrl/handler.test.ts
import { beforeEach, describe, expect, it, vi } from 'vitest';

vi.mock('@aws-sdk/s3-request-presigner', () => ({
  getSignedUrl: vi.fn().mockResolvedValue('https://signed.example/put'),
}));
vi.mock('@libs/middleware/dualAuth', () => ({ iamOnlyMiddleware: () => ({}) }));
vi.mock('@libs/middleware/subscription', () => ({ subscriptionMiddleware: () => ({}) }));
vi.mock('@libs/lambda', () => ({ middyfy: (h: any) => Object.assign(h, { use: () => h }) }));

import { logoUploadUrl } from './handler';

beforeEach(() => { process.env.ASSETS_BUCKET = 'bucket'; });

it('returns a presigned URL + an s3Key under the user logos prefix', async () => {
  const res: any = await logoUploadUrl(
    { userId: 'u1', body: { contentType: 'image/png' } } as any, {} as any, {} as any);
  const body = JSON.parse(res.body);
  expect(body.uploadUrl).toBe('https://signed.example/put');
  expect(body.s3Key).toMatch(/^users\/u1\/logos\/[0-9a-f-]+\.png$/);
});

it('rejects a disallowed content type', async () => {
  const res: any = await logoUploadUrl(
    { userId: 'u1', body: { contentType: 'image/gif' } } as any, {} as any, {} as any);
  expect(res.statusCode).toBe(400);
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd mkpdfs-backend && npx vitest run src/functions/templates/logoUploadUrl/handler.test.ts`
Expected: FAIL — cannot find module `./handler`.

- [ ] **Step 3: Write the implementation**

```ts
// src/functions/templates/logoUploadUrl/handler.ts
import { ValidatedEventAPIGatewayProxyEvent, formatJSONResponse, formatErrorResponse } from '@libs/apiGateway';
import { middyfy } from '@libs/lambda';
import { iamOnlyMiddleware } from '@libs/middleware/dualAuth';
import { subscriptionMiddleware } from '@libs/middleware/subscription';
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';
import { v4 as uuidv4 } from 'uuid';

const s3Client = new S3Client({});

const ALLOWED: Record<string, string> = {
  'image/png': 'png',
  'image/jpeg': 'jpg',
  'image/webp': 'webp',
  'image/svg+xml': 'svg',
};

interface Body { contentType: string }

export const logoUploadUrl: ValidatedEventAPIGatewayProxyEvent<Body> = async (event: any) => {
  try {
    const userId = event.userId!;
    const body = typeof event.body === 'string' ? JSON.parse(event.body) : event.body;
    const ext = ALLOWED[body?.contentType];
    if (!ext) {
      return formatJSONResponse(
        { message: 'contentType must be image/png, image/jpeg, image/webp or image/svg+xml' }, 400);
    }
    const s3Key = `users/${userId}/logos/${uuidv4()}.${ext}`;
    const uploadUrl = await getSignedUrl(s3Client, new PutObjectCommand({
      Bucket: process.env.ASSETS_BUCKET!,
      Key: s3Key,
      ContentType: body.contentType,
      Metadata: { 'user-id': userId, 'upload-purpose': 'template-logo' },
    }), { expiresIn: 300 });
    return formatJSONResponse({ uploadUrl, s3Key, expiresIn: 300 });
  } catch (error) {
    return formatErrorResponse(error as Error);
  }
};

export const main = middyfy(logoUploadUrl)
  .use(iamOnlyMiddleware())
  .use(subscriptionMiddleware());
```

```ts
// src/functions/templates/logoUploadUrl/index.ts
export { main } from './handler';
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd mkpdfs-backend && npx vitest run src/functions/templates/logoUploadUrl/handler.test.ts`
Expected: PASS (2 tests).

- [ ] **Step 5: Commit**

```bash
git add src/functions/templates/logoUploadUrl/
git commit -m "feat(theme): presigned logo upload endpoint"
```

---

## Task 10: Shared theme-resolution helper for handlers

**Files:**
- Create: `src/libs/theme/resolveLogoInput.ts`
- Test: `src/libs/theme/resolveLogoInput.test.ts`

Both `useTemplate` and `updateTheme` need the same logic: validate fields, resolve a `LogoInput` to a private S3 `logoKey`. Extract it once (DRY).

- [ ] **Step 1: Write the failing test**

```ts
// src/libs/theme/resolveLogoInput.test.ts
import { beforeEach, describe, expect, it, vi } from 'vitest';
import { mockClient } from 'aws-sdk-client-mock';
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';

const s3 = mockClient(S3Client);

vi.mock('./logoIngest', async (orig) => {
  const actual = await orig<typeof import('./logoIngest')>();
  return { ...actual, fetchLogoFromUrl: vi.fn() };
});
import { fetchLogoFromUrl } from './logoIngest';
import { resolveThemeInput } from './resolveLogoInput';

beforeEach(() => { s3.reset(); vi.clearAllMocks(); process.env.ASSETS_BUCKET = 'b'; });

it('keeps a validated upload s3Key owned by the user', async () => {
  const t = await resolveThemeInput('u1',
    { brand: '#000', accent: '#fff', fontKey: 'inter-inter',
      logo: { source: 'upload', s3Key: 'users/u1/logos/abc.png' } });
  expect(t).toEqual({ brand: '#000000', accent: '#ffffff', fontKey: 'inter-inter',
    logoKey: 'users/u1/logos/abc.png' });
});

it('rejects an upload s3Key that belongs to another user', async () => {
  await expect(resolveThemeInput('u1',
    { brand: '#000', accent: '#fff', fontKey: 'inter-inter',
      logo: { source: 'upload', s3Key: 'users/u2/logos/abc.png' } })).rejects.toThrow();
});

it('ingests a url logo to private S3 and returns its key', async () => {
  (fetchLogoFromUrl as any).mockResolvedValue({
    buffer: Buffer.from('x'), contentType: 'image/png', ext: 'png' });
  s3.on(PutObjectCommand).resolves({});
  const t = await resolveThemeInput('u1',
    { brand: '#000', accent: '#fff', fontKey: 'inter-inter',
      logo: { source: 'url', url: 'https://cdn.example.com/l.png' } });
  expect(t.logoKey).toMatch(/^users\/u1\/logos\/[0-9a-f-]+\.png$/);
  expect(s3.commandCalls(PutObjectCommand)).toHaveLength(1);
});

it('omits logoKey when no logo is provided', async () => {
  const t = await resolveThemeInput('u1',
    { brand: '#000', accent: '#fff', fontKey: 'inter-inter' });
  expect(t.logoKey).toBeUndefined();
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd mkpdfs-backend && npx vitest run src/libs/theme/resolveLogoInput.test.ts`
Expected: FAIL — cannot find module `./resolveLogoInput`.

- [ ] **Step 3: Write the implementation**

```ts
// src/libs/theme/resolveLogoInput.ts
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';
import { v4 as uuidv4 } from 'uuid';
import { Theme, ThemeInput } from './themeTypes';
import { validateThemeFields, ThemeValidationError } from './validateTheme';
import { fetchLogoFromUrl } from './logoIngest';

const s3Client = new S3Client({});

/**
 * Validate a ThemeInput and resolve its logo to a private S3 key owned by the
 * user. URL logos are fetched (SSRF-guarded) and stored; upload keys are
 * ownership-checked. Returns a storable Theme.
 */
export async function resolveThemeInput(userId: string, input: ThemeInput): Promise<Theme> {
  const fields = validateThemeFields(input);
  const theme: Theme = { ...fields };

  const logo = input.logo;
  if (!logo) return theme;

  const prefix = `users/${userId}/logos/`;
  if (logo.source === 'upload') {
    if (typeof logo.s3Key !== 'string' || !logo.s3Key.startsWith(prefix)) {
      throw new ThemeValidationError('logo.s3Key must be an uploaded key under your own prefix');
    }
    theme.logoKey = logo.s3Key;
  } else if (logo.source === 'url') {
    const { buffer, contentType, ext } = await fetchLogoFromUrl(logo.url);
    const key = `${prefix}${uuidv4()}.${ext}`;
    await s3Client.send(new PutObjectCommand({
      Bucket: process.env.ASSETS_BUCKET!,
      Key: key,
      Body: buffer,
      ContentType: contentType,
      Metadata: { 'user-id': userId, 'upload-purpose': 'template-logo' },
    }));
    theme.logoKey = key;
  }
  return theme;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd mkpdfs-backend && npx vitest run src/libs/theme/resolveLogoInput.test.ts`
Expected: PASS (4 tests).

- [ ] **Step 5: Commit**

```bash
git add src/libs/theme/resolveLogoInput.ts src/libs/theme/resolveLogoInput.test.ts
git commit -m "feat(theme): shared theme-input resolution (validate + logo→S3)"
```

---

## Task 11: PATCH /templates/{templateId}/theme

**Files:**
- Create: `src/functions/templates/updateTheme/handler.ts`
- Create: `src/functions/templates/updateTheme/index.ts`
- Test: `src/functions/templates/updateTheme/handler.test.ts`

- [ ] **Step 1: Write the failing test**

```ts
// src/functions/templates/updateTheme/handler.test.ts
import { beforeEach, describe, expect, it, vi } from 'vitest';
import { mockClient } from 'aws-sdk-client-mock';
import { DynamoDBDocumentClient, GetCommand, UpdateCommand } from '@aws-sdk/lib-dynamodb';

const ddb = mockClient(DynamoDBDocumentClient);

vi.mock('@libs/middleware/dualAuth', () => ({ iamOnlyMiddleware: () => ({}) }));
vi.mock('@libs/middleware/subscription', () => ({ subscriptionMiddleware: () => ({}) }));
vi.mock('@libs/lambda', () => ({ middyfy: (h: any) => Object.assign(h, { use: () => h }) }));
vi.mock('@libs/theme/resolveLogoInput', () => ({
  resolveThemeInput: vi.fn().mockResolvedValue(
    { brand: '#000000', accent: '#ffffff', fontKey: 'inter-inter' }),
}));
import { resolveThemeInput } from '@libs/theme/resolveLogoInput';
import { updateTheme } from './handler';

beforeEach(() => { ddb.reset(); vi.clearAllMocks(); process.env.TEMPLATES_TABLE = 't'; });

it('404s when the template does not belong to the user', async () => {
  ddb.on(GetCommand).resolves({ Item: undefined });
  const res: any = await updateTheme({ userId: 'u1', pathParameters: { templateId: 'x' },
    body: { brand: '#000', accent: '#fff', fontKey: 'inter-inter' } } as any, {} as any, {} as any);
  expect(res.statusCode).toBe(404);
});

it('writes the resolved theme via UpdateCommand (no content clobber)', async () => {
  ddb.on(GetCommand).resolves({ Item: { userId: 'u1', templateId: 'x', name: 'N' } });
  ddb.on(UpdateCommand).resolves({});
  const res: any = await updateTheme({ userId: 'u1', pathParameters: { templateId: 'x' },
    body: { brand: '#000', accent: '#fff', fontKey: 'inter-inter' } } as any, {} as any, {} as any);
  expect(res.statusCode).toBe(200);
  expect(resolveThemeInput).toHaveBeenCalledWith('u1', expect.objectContaining({ fontKey: 'inter-inter' }));
  const call = ddb.commandCalls(UpdateCommand)[0].args[0].input;
  expect(call.UpdateExpression).toContain('#theme');
  expect(call.ExpressionAttributeValues![':theme']).toEqual(
    { brand: '#000000', accent: '#ffffff', fontKey: 'inter-inter' });
});

it('400s on invalid theme fields', async () => {
  ddb.on(GetCommand).resolves({ Item: { userId: 'u1', templateId: 'x' } });
  (resolveThemeInput as any).mockRejectedValueOnce(
    Object.assign(new Error('bad'), { name: 'ThemeValidationError' }));
  const res: any = await updateTheme({ userId: 'u1', pathParameters: { templateId: 'x' },
    body: { brand: 'red', accent: '#fff', fontKey: 'inter-inter' } } as any, {} as any, {} as any);
  expect(res.statusCode).toBe(400);
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd mkpdfs-backend && npx vitest run src/functions/templates/updateTheme/handler.test.ts`
Expected: FAIL — cannot find module `./handler`.

- [ ] **Step 3: Write the implementation**

```ts
// src/functions/templates/updateTheme/handler.ts
import { ValidatedEventAPIGatewayProxyEvent, formatJSONResponse, formatErrorResponse } from '@libs/apiGateway';
import { middyfy } from '@libs/lambda';
import { iamOnlyMiddleware } from '@libs/middleware/dualAuth';
import { subscriptionMiddleware } from '@libs/middleware/subscription';
import { DynamoDBClient } from '@aws-sdk/client-dynamodb';
import { DynamoDBDocumentClient, GetCommand, UpdateCommand } from '@aws-sdk/lib-dynamodb';
import { resolveThemeInput } from '@libs/theme/resolveLogoInput';
import { ThemeInput } from '@libs/theme/themeTypes';

const docClient = DynamoDBDocumentClient.from(new DynamoDBClient({}));

export const updateTheme: ValidatedEventAPIGatewayProxyEvent<ThemeInput> = async (event: any) => {
  try {
    const userId = event.userId!;
    const templateId = event.pathParameters?.templateId;
    if (!templateId) return formatJSONResponse({ message: 'Template ID is required' }, 400);

    const existing = await docClient.send(new GetCommand({
      TableName: process.env.TEMPLATES_TABLE!,
      Key: { userId, templateId },
    }));
    if (!existing.Item) return formatJSONResponse({ message: 'Template not found' }, 404);

    const input = typeof event.body === 'string' ? JSON.parse(event.body) : event.body;

    let theme;
    try {
      theme = await resolveThemeInput(userId, input as ThemeInput);
    } catch (err: any) {
      if (err?.name === 'ThemeValidationError' || err?.name === 'LogoIngestError') {
        return formatJSONResponse({ message: err.message }, 400);
      }
      throw err;
    }

    await docClient.send(new UpdateCommand({
      TableName: process.env.TEMPLATES_TABLE!,
      Key: { userId, templateId },
      UpdateExpression: 'SET #theme = :theme, updatedAt = :now',
      ExpressionAttributeNames: { '#theme': 'theme' },
      ExpressionAttributeValues: { ':theme': theme, ':now': new Date().toISOString() },
    }));

    return formatJSONResponse({ message: 'Theme updated', theme });
  } catch (error) {
    return formatErrorResponse(error as Error);
  }
};

export const main = middyfy(updateTheme)
  .use(iamOnlyMiddleware())
  .use(subscriptionMiddleware());
```

```ts
// src/functions/templates/updateTheme/index.ts
export { main } from './handler';
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd mkpdfs-backend && npx vitest run src/functions/templates/updateTheme/handler.test.ts`
Expected: PASS (3 tests).

- [ ] **Step 5: Commit**

```bash
git add src/functions/templates/updateTheme/
git commit -m "feat(theme): PATCH /templates/{id}/theme endpoint"
```

---

## Task 12: Capture theme at adoption + content version on writes

**Files:**
- Modify: `src/functions/marketplace/useTemplate/handler.ts`
- Modify: `src/functions/templates/updateTemplate/handler.ts:117-142`
- Modify: `src/functions/templates/uploadTemplate/handler.ts`
- Test: `src/functions/marketplace/useTemplate/handler.test.ts`

- [ ] **Step 1: Write the failing test (useTemplate stores theme + contentVersion)**

```ts
// src/functions/marketplace/useTemplate/handler.test.ts
import { beforeEach, describe, expect, it, vi } from 'vitest';
import { mockClient } from 'aws-sdk-client-mock';
import { DynamoDBDocumentClient, GetCommand, PutCommand, QueryCommand, UpdateCommand } from '@aws-sdk/lib-dynamodb';
import { S3Client, CopyObjectCommand } from '@aws-sdk/client-s3';

const ddb = mockClient(DynamoDBDocumentClient);
const s3 = mockClient(S3Client);

vi.mock('@libs/middleware/dualAuth', () => ({ iamOnlyMiddleware: () => ({}) }));
vi.mock('@libs/middleware/subscription', () => ({ subscriptionMiddleware: () => ({}) }));
vi.mock('@libs/lambda', () => ({ middyfy: (h: any) => Object.assign(h, { use: () => h }) }));
vi.mock('@libs/theme/resolveLogoInput', () => ({
  resolveThemeInput: vi.fn().mockResolvedValue(
    { brand: '#000000', accent: '#ffffff', fontKey: 'inter-inter' }),
}));
import { main as useTemplate } from './handler';

beforeEach(() => {
  ddb.reset(); s3.reset(); vi.clearAllMocks();
  process.env.MARKETPLACE_TABLE = 'mp'; process.env.TEMPLATES_TABLE = 't'; process.env.ASSETS_BUCKET = 'b';
  ddb.on(GetCommand).resolves({ Item: { templateId: 'mp-x', name: 'X', s3Key: 'marketplace/x.hbs' } });
  ddb.on(QueryCommand).resolves({ Count: 0 });
  ddb.on(PutCommand).resolves({});
  ddb.on(UpdateCommand).resolves({});
  s3.on(CopyObjectCommand).resolves({ VersionId: 'ver-123' });
});

it('persists the resolved theme and the S3 content version on the new row', async () => {
  const res: any = await useTemplate({ userId: 'u1', pathParameters: { templateId: 'mp-x' },
    subscriptionLimits: { templatesAllowed: -1 },
    body: { theme: { brand: '#000', accent: '#fff', fontKey: 'inter-inter' } } } as any, {} as any, {} as any);
  expect(res.statusCode).toBe(201);
  const item = ddb.commandCalls(PutCommand)[0].args[0].input.Item as any;
  expect(item.theme).toEqual({ brand: '#000000', accent: '#ffffff', fontKey: 'inter-inter' });
  expect(item.contentVersion).toBe('ver-123');
});

it('works with no theme (theme omitted from the row)', async () => {
  const res: any = await useTemplate({ userId: 'u1', pathParameters: { templateId: 'mp-x' },
    subscriptionLimits: { templatesAllowed: -1 }, body: {} } as any, {} as any, {} as any);
  expect(res.statusCode).toBe(201);
  const item = ddb.commandCalls(PutCommand)[0].args[0].input.Item as any;
  expect(item.theme).toBeUndefined();
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd mkpdfs-backend && npx vitest run src/functions/marketplace/useTemplate/handler.test.ts`
Expected: FAIL — current handler ignores `body.theme` and doesn't set `contentVersion`.

- [ ] **Step 3: Edit `useTemplate/handler.ts`**

Add import at the top with the other imports:

```ts
import { resolveThemeInput } from '@libs/theme/resolveLogoInput';
import { ThemeInput } from '@libs/theme/themeTypes';
```

After `const templateId = event.pathParameters?.templateId;` and its guard, parse + resolve the theme:

```ts
    const input = typeof event.body === 'string' ? JSON.parse(event.body || '{}') : (event.body || {});
    let theme;
    if (input.theme) {
      try {
        theme = await resolveThemeInput(userId, input.theme as ThemeInput);
      } catch (err: any) {
        if (err?.name === 'ThemeValidationError' || err?.name === 'LogoIngestError') {
          return formatJSONResponse({ message: err.message }, 400);
        }
        throw err;
      }
    }
```

Capture the CopyObject response so we can store the new version. Change:

```ts
    await s3Client.send(new CopyObjectCommand({
```

to:

```ts
    const copyResult = await s3Client.send(new CopyObjectCommand({
```

In the `userTemplate` object literal, add `theme` (only when set) and `contentVersion`:

```ts
    const userTemplate = {
      userId,
      templateId: newTemplateId,
      id: newTemplateId,
      name: mpTemplate.name,
      description: mpTemplate.description || '',
      s3Key: newS3Key,
      sourceMarketplaceId: templateId,
      contentVersion: copyResult.VersionId || now,
      ...(theme ? { theme } : {}),
      createdAt: now,
      updatedAt: now
    };
```

- [ ] **Step 4: Edit `updateTemplate/handler.ts` to capture the version**

Change the `PutObjectCommand` send (line ~117) to capture the result:

```ts
    const putResult = await s3Client.send(new PutObjectCommand({
```

In `updatedTemplate`, add the version (right after `s3Key,`):

```ts
      s3Key,
      contentVersion: putResult.VersionId || now,
      fileSize: Buffer.byteLength(templateContent, 'utf-8'),
```

- [ ] **Step 5: Edit `uploadTemplate/handler.ts` to capture the version**

Read the file, find the `PutObjectCommand` send that writes the `.hbs` and the DynamoDB `PutCommand` that creates the row. Capture the put result (`const putResult = await s3Client.send(new PutObjectCommand({...}))`) and add `contentVersion: putResult.VersionId || <the createdAt/now value already in scope>` to the template item written to DynamoDB. (Match the existing variable name used for the timestamp.)

- [ ] **Step 6: Run tests + typecheck**

Run: `cd mkpdfs-backend && npx vitest run src/functions/marketplace/useTemplate/handler.test.ts && npm run typecheck`
Expected: useTemplate tests PASS (2); typecheck PASS.

- [ ] **Step 7: Commit**

```bash
git add src/functions/marketplace/useTemplate/ src/functions/templates/updateTemplate/handler.ts src/functions/templates/uploadTemplate/handler.ts
git commit -m "feat(theme): capture theme at adoption + S3 content version on writes"
```

---

## Task 13: CDK — grants + new routes

**Files:**
- Modify: `cdk/lib/stacks/api-stack.ts`
- Modify: `cdk/lib/stacks/jobs-stack.ts:130` (after the existing `bucket.grantRead(this.processJobFn)`)

- [ ] **Step 1: Grant the render lambdas read access to the templates table**

In `api-stack.ts`, in the `generatePdf` block (after `bucket.grantPut(generatePdf);`, ~line 290) add:

```ts
    tables.templates.grantReadData(generatePdf); // pdfService reads the theme + contentVersion
```

In the `generatePdfApiKey` block (after `bucket.grantPut(generatePdfApiKey);`, ~line 313) add:

```ts
    tables.templates.grantReadData(generatePdfApiKey);
```

- [ ] **Step 2: Grant the SQS processor read access**

In `jobs-stack.ts`, after `bucket.grantRead(this.processJobFn);` (line ~130) add:

```ts
    tables.templates.grantReadData(this.processJobFn); // pdfService reads the theme
```

- [ ] **Step 3: Add the two new routes in `api-stack.ts`**

In the TEMPLATES section (after the `updateTemplate` block, ~line 258) add:

```ts
    const logoUploadUrl = makeFn('LogoUploadUrlFn', {
      entry: 'src/functions/templates/logoUploadUrl/handler.ts',
      timeoutSeconds: 10,
      memorySize: 256,
    });
    bucket.grantPut(logoUploadUrl); // presigned PUT for the logo file
    grantSubscriptionMw(logoUploadUrl);
    addRoute('/templates/logo-upload-url', 'POST', logoUploadUrl, true);

    const updateTemplateTheme = makeFn('UpdateTemplateThemeFn', {
      entry: 'src/functions/templates/updateTheme/handler.ts',
    });
    tables.templates.grantReadWriteData(updateTemplateTheme);
    bucket.grantPut(updateTemplateTheme); // store URL-ingested logos
    grantSubscriptionMw(updateTemplateTheme);
    addRoute('/templates/{templateId}/theme', 'PATCH', updateTemplateTheme, true);
```

- [ ] **Step 4: Grant `useTemplate` bucket PUT for URL-logo ingestion**

In the MARKETPLACE section, the `marketplaceUseTemplate` block already has `bucket.grantPut` (line ~427) for the CopyObject — no change needed. Confirm it is present; it covers the logo PutObject too.

- [ ] **Step 5: Typecheck + synth**

Run: `cd mkpdfs-backend && npm run typecheck && npx cdk synth -c environment=dev Mkpdfs-Api-dev > /dev/null && echo SYNTH_OK`
Expected: typecheck PASS; `SYNTH_OK` printed (CloudFormation synthesizes without error).

- [ ] **Step 6: Commit**

```bash
git add cdk/lib/stacks/api-stack.ts cdk/lib/stacks/jobs-stack.ts
git commit -m "feat(theme): CDK grants + logo-upload-url and theme PATCH routes"
```

---

## Task 14: Normalize the 13 marketplace templates to the token contract

**Files:**
- Modify: all of `scripts/marketplace-templates/*.hbs`

Each template must: (a) declare the full themeable contract in `:root` with TODAY's values as defaults; (b) use `var(--brand)`, `var(--accent)`, `var(--brand-soft)`, `var(--brand-shadow)`, `var(--accent-soft)`, `var(--font-heading)`, `var(--font-body)` everywhere those concepts appear (replacing hardcoded hex / rgba / font-family); (c) replace the `.brand-dot` letter mark with the `{{mkpdfsLogo <nameField>}}` helper and add a `.brand-logo` style.

- [ ] **Step 1: Worked example — `mp-business-invoice.hbs`**

Replace the `:root` block (current lines ~7-16) with the full contract, keeping today's values as defaults:

```css
    :root {
      --ink: #16151A;
      --muted: #6B6B76;
      --faint: #9A9AA4;
      --line: #E6E6EA;
      --hair: #F1F1F4;
      --brand: #8C6CFF;
      --brand-soft: #F3EFFF;
      --brand-shadow: rgba(140, 108, 255, 0.28);
      --accent: #8C6CFF;
      --accent-soft: #F3EFFF;
      --font-heading: 'Fraunces', Georgia, serif;
      --font-body: 'Inter', -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif;
    }
```

Then apply these substitutions in the same file:
- `font-family: 'Inter', -apple-system, …;` on `html, body` → `font-family: var(--font-body);`
- every `font-family: 'Fraunces', Georgia, serif;` → `font-family: var(--font-heading);`
- `.brand-dot { … background: var(--accent); … box-shadow: 0 4px 12px rgba(140, 108, 255, 0.28); … }` → `background: var(--brand);` and `box-shadow: 0 4px 12px var(--brand-shadow);`
- `.doc-num { … color: var(--accent); }` (keep) — `--accent` now resolves to the user accent.
- `.totals-grand { … background: var(--accent-soft); }` (was already `var(--accent-soft)`; ensure the var exists — it now does).
- `.totals-grand .lbl { color: var(--accent); }` (keep).
- `.terms .tick { background: var(--accent); }` (keep).

Add a `.brand-logo` rule near `.brand-dot`:

```css
    .brand-logo { height: 30px; width: auto; max-width: 160px; object-fit: contain; display: block; }
```

Replace the logo markup in `.brand-mark`:

```hbs
        <div class="brand-mark">
          {{mkpdfsLogo companyName}}
          <div class="company-name">{{companyName}}</div>
        </div>
```

- [ ] **Step 2: Apply the same contract to the other 12 templates**

For each of `mp-business-quote`, `mp-business-receipt`, `mp-cert-achievement`, `mp-cert-completion`, `mp-cert-participation`, `mp-marketing-brochure`, `mp-marketing-flyer`, `mp-marketing-newsletter`, `mp-personal-cover-letter`, `mp-personal-invitation`, `mp-personal-letter`, `mp-personal-resume`:

1. Open the file. Insert the full `:root` contract: keep its existing neutral vars; map its current primary brand color → `--brand`, its secondary/highlight → `--accent` (if it only has one, set both `--brand` and `--accent` to it); set `--brand-soft`/`--accent-soft` to its existing soft tint (or the same color); set `--brand-shadow` to its existing shadow rgba (or `rgba` of its brand at 0.28); set `--font-heading`/`--font-body` from its current `@import` fonts.
2. Replace hardcoded color hex/rgba that represent the brand/accent with the matching `var(--…)`. Leave true neutrals (ink/grey/lines) as-is or mapped to the neutral vars.
3. Replace hardcoded `font-family` declarations with `var(--font-heading)` / `var(--font-body)`.
4. If the template has a letter/initial logo mark, replace it with `{{mkpdfsLogo <nameField>}}` (use whichever field that template uses — `companyName`, `businessName`, `organizationName`, `recipientName`, etc.) and add the `.brand-logo` rule. If a template has no brand mark, skip the logo step for it.
5. Keep each template's own `@import` line — it is the default-font fallback when no theme is applied.

- [ ] **Step 3: Verify defaults render unchanged (no theme)**

Sanity-check one template locally without Chromium by compiling it and confirming the contract resolves. Run:

```bash
cd mkpdfs-backend && node -e "
const Handlebars=require('handlebars');
const fs=require('fs');
const t=fs.readFileSync('scripts/marketplace-templates/mp-business-invoice.hbs','utf8');
Handlebars.registerHelper('mkpdfsLogo',(n)=>new Handlebars.SafeString('<div class=\"brand-dot\">'+(n?String(n)[0]:'')+'</div>'));
Handlebars.registerHelper('formatCurrency',(a)=>'\$'+a);
const html=Handlebars.compile(t)({companyName:'Acme',items:[],invoiceNumber:'1'});
if(!html.includes('--brand: #8C6CFF')) throw new Error('contract var missing');
if(!html.includes('brand-dot')) throw new Error('logo slot missing');
console.log('TEMPLATE_OK');
"
```

Expected: `TEMPLATE_OK`.

- [ ] **Step 4: Commit**

```bash
git add scripts/marketplace-templates/
git commit -m "feat(theme): normalize 13 marketplace templates to theme-token contract"
```

---

## Task 15: Deploy to dev, re-seed, and verify end-to-end

**Files:** none (operational)

- [ ] **Step 1: Full test suite + typecheck**

Run: `cd mkpdfs-backend && npm test && npm run typecheck`
Expected: all tests PASS; typecheck clean.

- [ ] **Step 2: Deploy backend to dev**

Run: `cd mkpdfs-backend && npm run cdk:deploy:dev`
Expected: `Mkpdfs-Api-dev` and `Mkpdfs-Jobs-dev` update succeed.

- [ ] **Step 3: Re-seed the dev marketplace with the normalized templates**

Run: `cd mkpdfs-backend && AWS_PROFILE=rocketeast npx ts-node scripts/seed-marketplace.ts dev`
Expected: 13 templates uploaded/updated.

- [ ] **Step 4: Verify themed render via API (manual, dev)**

Adopt a template, set a theme, generate a PDF, and confirm it renders. Using a Cognito JWT for a dev test user (`$JWT`) and `dev.apis.mkpdfs.com`:

```bash
# 1) adopt the invoice with a theme (no logo)
curl -s -X POST https://dev.apis.mkpdfs.com/marketplace/templates/mp-business-invoice/use \
  -H "Authorization: Bearer $JWT" -H 'Content-Type: application/json' \
  -d '{"theme":{"brand":"#0F62FE","accent":"#FF6B35","fontKey":"poppins-poppins"}}'
# → note the returned template.templateId as $TID

# 2) generate a PDF from the adopted (themed) template
curl -s -X POST https://dev.apis.mkpdfs.com/pdf/generate \
  -H "Authorization: Bearer $JWT" -H 'Content-Type: application/json' \
  -d '{"templateId":"'$TID'","data":{"companyName":"Acme","companyAddress":"1 St","companyEmail":"a@b.com","clientName":"C","clientAddress":"2 St","invoiceNumber":"INV-1","invoiceDate":"Jan 1","dueDate":"Feb 1","paymentTerms":"Net 30","items":[{"description":"Work","quantity":1,"rate":100,"amount":100}],"subtotal":100,"taxRate":8,"taxAmount":8,"total":108}}'
# → open pdfUrl; confirm blue brand + orange accent + Poppins, and {{today}} usable
```

Expected: a 201 from `use` (theme echoed), a 200 from `generate` with a `pdfUrl`; opening it shows the blue/orange Poppins theme.

- [ ] **Step 5: Verify re-editing the theme**

```bash
curl -s -X PATCH https://dev.apis.mkpdfs.com/templates/$TID/theme \
  -H "Authorization: Bearer $JWT" -H 'Content-Type: application/json' \
  -d '{"brand":"#16A34A","accent":"#16A34A","fontKey":"lora-source-sans"}'
# regenerate (step 4 part 2) → now green + Lora
```

Expected: PATCH returns `{ "message": "Theme updated", ... }`; regenerated PDF is green/Lora.

- [ ] **Step 6: Verify an un-themed template is unchanged**

Adopt a second template WITHOUT a theme and generate — output must match pre-change rendering (defaults intact, marketplace previews unaffected).

- [ ] **Step 7: Commit any seed/log artifacts if applicable, then merge dev → main per branch strategy after verification.**

```bash
git commit --allow-empty -m "chore(theme): backend theming verified on dev"
```

---

## Self-Review

**Spec coverage:**
- Theme contract (`--brand/--accent/--brand-soft/--brand-shadow/--accent-soft/--font-heading/--font-body`) → Tasks 4, 14.
- `theme` on the template row → Tasks 12 (write), 8 (read).
- Capture at adoption, re-editable → Tasks 12 (useTemplate), 11 (PATCH).
- Stored as data, applied at render → Task 8.
- Logo URL or upload, converge to private S3, inline data-URI → Tasks 7, 9, 10, 8.
- `mkpdfsLogo` helper / inconsistent name fields → Tasks 8, 14.
- Derived tokens computed server-side → Tasks 2, 4.
- Cache keyed on content version, LRU bound, TTL dropped → Tasks 8, 12.
- Predefined params (today/now/year), reserved/override, once per request → Tasks 6, 8.
- Curated fonts shared registry → Task 1.
- Security: strict hex, fontKey enum, SSRF, size/type caps, reserved `mkpdfsTheme` data frame → Tasks 3, 7, 8, 10.
- Re-seed + previews unchanged → Tasks 14, 15.

**Placeholder scan:** Task 14 Step 2 and Task 12 Step 5 describe per-file edits rather than full literals because the 12 templates and the upload handler differ in their existing markup; the contract values, substitution rules, and the variable name to match are all specified explicitly. No `TBD`/`TODO` remain.

**Type consistency:** `Theme` (`brand/accent/fontKey/logoKey?`) is defined in Task 1 and used identically in Tasks 8/10/11/12. `ResolvedTheme` (`brand/accent/fontKey/logoDataUri`) is the render-time shape in Task 8 and matched by the `composeHtml` test. `resolveThemeInput(userId, ThemeInput): Theme` (Task 10) is called with the same signature in Tasks 11 and 12. `buildSystemParams(Date): SystemParams` (Task 6) matches its use in Task 8.

## Open follow-up (separate plan)
- Web UI: branding wizard in the "Use template" flow + re-openable theme editor (mirrors the `FONTS` registry, calls `POST /templates/logo-upload-url`, `POST /marketplace/.../use` with `theme`, and `PATCH /templates/{id}/theme`).
