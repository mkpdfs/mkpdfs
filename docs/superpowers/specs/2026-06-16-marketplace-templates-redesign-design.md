# Marketplace Templates Redesign — Design

**Date:** 2026-06-16
**Status:** Approved (pending user spec review)
**Owner:** sim4r4

## Problem

The mkpdfs marketplace is empty in both `dev` and `prod` (DynamoDB tables `mkpdfs-{env}-marketplace` have 0 items). There ARE 13 Handlebars templates defined in `mkpdfs-backend/scripts/seed-marketplace.ts` with `.hbs` files in `scripts/marketplace-templates/`, but:

1. The seed was never run, so nothing is listed.
2. The seed does **not** set a `thumbnailKey` nor upload preview images — so even after seeding, cards fall back to a generic placeholder (handlers build `thumbnailUrl` from `thumbnailKey` via `src/libs/thumbnailUrl.ts`).
3. The existing `.hbs` are functional but generic ("default template" look: blue `#2563eb`, system fonts, boxy layout) — below the bar for a public-facing showcase.

The marketplace is our calling card ("carta de presentación"): every template must be very well polished and feel like one cohesive premium product.

## Goals

- Redesign all **13** templates to a premium-minimalist standard, visually cohesive across the catalog.
- Generate a high-quality preview PNG per template and wire it through `thumbnailKey`.
- Seed `dev` end-to-end so the marketplace renders live. **Prod stays paused until user OK.**

## Non-Goals

- No new templates — exactly the existing 13, polished.
- No `thumbnails-full` variant (the web never reads it — YAGNI). One thumbnail per template serves both the card crop and the modal.
- No changes to marketplace handlers, web components, or the `useTemplate` flow.
- No prod seeding in this pass.

## The 13 templates (unchanged lineup)

| Category | templateId | Name |
|---|---|---|
| business | mp-business-invoice | Professional Invoice |
| business | mp-business-quote | Service Quote |
| business | mp-business-receipt | Payment Receipt |
| certificates | mp-cert-completion | Course Completion Certificate |
| certificates | mp-cert-achievement | Achievement Award |
| certificates | mp-cert-participation | Participation Certificate |
| marketing | mp-marketing-brochure | Product Brochure |
| marketing | mp-marketing-flyer | Event Flyer |
| marketing | mp-marketing-newsletter | Email Newsletter |
| personal | mp-personal-resume | Modern Resume |
| personal | mp-personal-letter | Formal Letter |
| personal | mp-personal-invitation | Event Invitation |
| personal | mp-personal-cover-letter | Cover Letter |

Each keeps its existing `templateId`, `category`, `name`, `description`, `tags`, and the **same `sampleDataJson` field shape** (so the redesigned `.hbs` stays compatible with the data already in the seed). Sample data wording may be lightly refined but field names/structure must not change.

## Design system (shared across all 13)

**Typography:** premium serif display + geometric sans pairing.
- Display/headers & certificates: a serif (Fraunces or Playfair Display).
- Body/UI/numbers: Inter; tabular/mono for figures where it reads better.
- Delivery: Google Fonts via `@import`/`<link>`, with a robust system-font fallback stack.

**Palette:**
- Ink (near-black): `#16151A`
- Neutral text/lines: `#6B6B76`, hairlines `#E6E6EA`
- Surface: `#FFFFFF`
- **Brand accent: `#8C6CFF`** (matches the dashboard), used with restraint — rules, totals, seals, key emphasis. No heavy color fills as the default.

**Layout principles:** generous margins and whitespace; hierarchy through weight/size/space rather than colored boxes; thin hairline rules; consistent vertical rhythm.

**Per-category character (same base, distinct voice):**
- **Business** — trustworthy & precise: thin tables (no heavy zebra), totals with an accent rule, tabular figures.
- **Certificates** — elegant & ceremonial: landscape, fine border frame, large serif name, an SVG seal/medal in the accent, signature line. Most ornamented but restrained.
- **Marketing** — editorial & confident: a touch more color/scale, hero accent block, feature grid, clear CTA.
- **Personal** — warm & human: resume as an ATS-friendly two-column; letters with a clean letterhead; invitation more centered and warm.

## Font-rendering risk

The deployed PDF path uses the Sparticuz Chromium layer in Lambda; external Google Fonts may not load reliably there.
- Thumbnails are generated **locally** (network + system fonts available) → no risk for the previews.
- For the deployed templates, verify the fonts actually render through the real generation path. If they don't load, **embed the WOFF2 fonts as base64 data URIs** in the `.hbs`. Always ship a system-font fallback so output is never broken.

## Thumbnail pipeline

New script: `mkpdfs-backend/scripts/generate-thumbnails.ts`.

1. For each template: read its `.hbs`, compile with Handlebars using the template's `sampleDataJson`.
2. Render the HTML in headless Chrome via `puppeteer-core` pointed at the locally installed Chrome (fallback: `@sparticuz/chromium` or the e2e-browser skill's bundled Chrome).
3. Set viewport to the template's natural page size (portrait A4/Letter for docs; landscape for certificates and the flyer), `deviceScaleFactor: 2`.
4. Full-page screenshot → PNG written to `scripts/marketplace-thumbnails/{templateId}.png` (gitignored or kept — TBD trivial).
5. Upload each PNG to `s3://mkpdfs-{env}-bucket/marketplace/thumbnails/{templateId}.png`, `ContentType: image/png` (public-read is already granted by the bucket policy for `marketplace/thumbnails`).

The card (`object-cover`, `h-32` band) crops to the document's top; the modal (`object-contain`) shows the whole page. One full-page PNG serves both.

## Seed changes

In `seed-marketplace.ts`:
- Add `thumbnailKey: \`marketplace/thumbnails/${templateId}.png\`` to the `MarketplaceTemplate` interface and to each seeded item.
- No other behavioral change. `clearExistingData` + `seedTemplates` already handle DynamoDB + `.hbs` S3 upload.

Order of operations for `dev`:
1. Redesign the 13 `.hbs`.
2. `generate-thumbnails.ts dev` → render PNGs, **show the 13 PNGs to the user for sign-off**.
3. Upload PNGs to S3 dev.
4. `seed-marketplace.ts dev` → populate DynamoDB + upload `.hbs`.

## Verification

- Render and visually review all 13 PNGs before seeding.
- After seeding dev: load `dev.mkpdfs.com/marketplace`, confirm the grid shows polished thumbnails across all 4 category filters, and the preview modal shows the full image.
- Spot-check that `useTemplate` still copies a redesigned template into a user account (no schema/field drift).
- Prod: only after explicit user approval, re-run `generate-thumbnails.ts prod` + `seed-marketplace.ts prod`.

## Out-of-scope follow-ups (noted, not done here)

- The preview modal's "preview" tab still shows raw template code rather than a rendered PDF — a separate UX improvement.
- Optional: align sample-data wording/branding across templates for a more consistent demo story.
