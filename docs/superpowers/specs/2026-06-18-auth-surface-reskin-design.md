# Auth Surface Re-skin — Design

**Date:** 2026-06-18
**Repo:** `mkpdfs-web` (submodule)
**Status:** Design — pending implementation plan

## Problem

Running `mkp auth login` opens `dev.mkpdfs.com/cli/authorize`; when the browser
has no session it redirects to `/login?redirect=/cli/authorize`. The landing page
was recently redesigned with a distinctive design system (new `ink/surface/fg/brand`
token set, purple brand gradient `#8C6CFF → #5B3FE0`, Geist fonts, ambient violet
glow + faint grid). The auth pages were never migrated — they still render the old
generic theme (a floating shadcn `Card`, a **blue** `bg-primary` button, Inter
font, pale-blue gradient background). So the first screen a CLI user lands on looks
off-brand relative to the marketing site.

Confirmed by driving the real flow: `/cli/authorize` → `/login`, screenshotted
side-by-side with the landing. The mismatch is the in-app `/login` (and its sibling
auth pages), **not** the Cognito Hosted UI. (The washed-out button in the screenshot
is just `disabled:opacity-50` on an empty form.)

## Goal

Re-skin the **entire auth surface** to match the landing design system, with the
change **isolated to auth** — no global component or token mutation. Preserve all
auth logic exactly.

### In scope (5 surfaces)
- `src/app/[locale]/(auth)/login/LoginClient.tsx`
- `src/app/[locale]/(auth)/register/RegisterClient.tsx`
- `src/app/[locale]/(auth)/forgot-password/ForgotPasswordClient.tsx`
- `src/app/[locale]/(auth)/reset-password/ResetPasswordClient.tsx`
- `src/app/[locale]/cli/authorize/AuthorizeClient.tsx` (approve/deny card)

### Explicitly out of scope
- **Token consolidation / dual-system debt** — see "Known debt" below. File a
  separate ticket; do not normalize `primary` here.
- **Cognito Hosted UI for Google OAuth** — the "Continue with Google" button still
  redirects to `auth-mkpdfs-{env}.auth...amazoncognito.com`. Restyle the button
  only; rewiring OAuth to an in-app flow is a separate backend ticket. (Aligns with
  the standing rule: never send users to Hosted UI — tracked, not fixed here.)
- **Shared `ui/{Button,Card,Input}` components** — used across the whole dashboard.
  Do not touch.

## Why this approach (decision)

The shared `ui/Button`, `ui/Card`, `ui/Input` are bound to the **old** token system
(`bg-primary` → tailwind hardcoded blue `#3B82F6`; `bg-card`; `border-input`) and are
consumed all over the dashboard. Two options were considered:

- **A. Auth-isolated primitives + shell (chosen).** New auth-scoped presentational
  components using the new tokens, plus a shell that recreates the landing context.
  Dashboard untouched. Lowest risk, clearest boundary.
- **B. Add brand variants to shared `ui/*` via cva.** Reuses infra but invites a
  half-migrated component library and risks visual drift in the dashboard. Only
  justified as the start of a deliberate design-system migration — which this is not.

Codex review concurred with A and sharpened the scope rules below.

## Architecture

### 1. `AuthShell` — server component (new)
`src/components/auth/AuthShell.tsx`. Recreates the landing wrapper context (mirrors
`src/app/[locale]/page.tsx:63`):
- Applies `${GeistSans.variable} ${GeistMono.variable} mk-landing relative min-h-screen bg-surface font-geist text-fg`.
- Owns the **viewport, centering, padding, and ambient layers** (violet glow + faint
  grid). Children render centered inside.
- Server component — imported by the `(auth)` layout and the CLI authorize page, not
  by client components.
- The card/content sits at `relative z-10` above the ambient `absolute` layers.

**Ambient layers — token-aware.** The landing grid is hardcoded `rgba(255,255,255,…)`
(`page.tsx:73`), tuned for the landing's dark background. On light `bg-surface` it
reads wrong. Make the grid lines `rgb(var(--ink) / <low-alpha>)` (or paired light/dark
classes) so it works in both themes. The violet glow can be reused as-is.

### 2. Auth-scoped primitives (new)
`src/components/auth/` — presentational only, new tokens:
- `AuthCard` — raised surface card (`bg-surface-raised`, `border-ink/[0.07]`, brand-ish
  shadow), replaces `ui/Card`.
- `AuthButton` — primary action styled like the landing nav "Get Started" (dark `fg`
  pill or purple brand gradient), with `isLoading` spinner. Replaces `ui/Button`
  default + destructive (deny) variants.
- `AuthInput` + `AuthLabel` (or `ui/Label` with class overrides) — new-token field
  styling, focus ring on `brand`, error on `danger`.
- `AuthDivider` — the "or continue with" rule (replaces the `bg-card`/`text-muted-foreground`
  divider).
- `AuthLoader` — branded "checking session" loader (replaces `PageLoader`/`Spinner`,
  which use old tokens).
- `AuthBrandMark` — the landing's purple gradient `FileText` logo tile (from
  `LandingNav`), reused at the top of each card.

### 3. Layout + CLI page wiring
- New `src/app/[locale]/(auth)/layout.tsx` wraps the route group's children in
  `AuthShell`.
- `cli/authorize` is **outside** the `(auth)` group, so wrap it directly in its
  `page.tsx` (`src/app/[locale]/cli/authorize/page.tsx`) with `AuthShell` — the layout
  will not cover it.

### 4. Refactor the 5 clients
- **Remove the per-client full-screen wrappers** (`min-h-screen flex items-center
  justify-center bg-gradient-to-br from-primary-50 …` etc.) — the shell now owns that.
  Nested backgrounds otherwise.
- Swap `ui/{Card,Button,Input,Label}` usage for the auth primitives.
- **Hunt every old-token residue**, not just the obvious imports:
  - links `text-primary` → `text-brand-text`
  - dividers `bg-card` / `text-muted-foreground`
  - error/success states `destructive` / `success` → `danger` / `ok`
  - register password-requirement indicators
  - loaders (`PageLoader`/`isInitializing` branches) → `AuthLoader`
  - the Google button restyle (functionality unchanged)
- **Preserve all logic byte-for-byte:** `signIn`, `signInWithGoogle(customState)`,
  `getRedirectTarget()`/`sanitizeRedirectPath`, validation, `isSubmitting`/`isLoading`/
  `isInitializing`/`error` states, the `useAuth` wiring, the deny-left/approve-right
  button order (per the primary-action-right convention).

### 5. CLI authorize specifics
- Replace the hardcoded `router.push('/login?redirect=/cli/authorize')`
  (`AuthorizeClient.tsx:28`) with a redirect that preserves the current
  `pathname + search`, so any future CLI query params survive the login round-trip.
- Replace the `return null` during the redirect/initializing window
  (`AuthorizeClient.tsx:31`) with `AuthLoader`.
- Keep the 8-char code input, approve/deny, and the done/denied/error states; re-skin
  only.

## Data flow

Unchanged. UI is presentational; `useAuth`, Amplify, and the backend device-flow
endpoints (`/auth/cli/{device,approve,token}`) are untouched. `AuthShell` is a pure
layout wrapper.

## Error handling

No new error paths. Existing error/loading/success states are re-skinned to new
tokens (`danger`/`ok`/`brand`). The Google → Hosted UI path is unchanged and still
out of scope.

## Theming / dark mode

New tokens are already theme-aware via `.dark` in `globals.css`. Verify both themes
(there is a `ThemeToggle`). The only theme risk is the ambient grid (addressed:
make it token-aware) and ensuring card/content `z-10` sits above ambient layers in
both modes.

## Testing / verification

Drive the real browser (chrome-devtools / cdp fallback) against dev and screenshot,
**light and dark**:
1. `mkp auth login` redirect: `/cli/authorize` (unauth) → `/login` renders branded.
2. `/login`, `/register`, `/forgot-password`, `/reset-password` — branded, matches landing.
3. CLI authorize authenticated: code input + approve/deny + done/denied/error cards.
4. Confirm `signIn` and the `redirect` round-trip still land on the right page.
5. No old-theme fragments left (links, dividers, errors, loaders).

## Known debt (out of scope — file separate tickets)

1. **Dual token system.** `globals.css` defines `--primary` as purple, but
   `tailwind.config.ts` hardcodes `primary.DEFAULT = #3B82F6` (blue), so
   `bg-primary`/`text-primary`/`from-primary` are blue, not brand. Normalizing this
   would visually shift the dashboard, modals, dropzones, settings — needs its own
   migration ticket.
2. **Google OAuth → Cognito Hosted UI.** Replace the Hosted-UI redirect with an
   in-app branded flow (backend/OAuth change).
