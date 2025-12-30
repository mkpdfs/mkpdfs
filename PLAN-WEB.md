# mkpdfs-web Implementation Plan

> Last Updated: 2025-12-28

## Overview

Build the mkpdfs-web frontend from scratch - a Next.js 14 dashboard for the PDF generation SaaS platform.

| Config | Value |
|--------|-------|
| Domain | mkpdfs.com |
| Branding | Blue/Purple tech style |
| Languages | English + Spanish (i18n) |
| Pricing | Free, Starter, Pro, Enterprise tiers |

---

## Status Summary

| Phase | Status | Description |
|-------|--------|-------------|
| Phase 1 | ✅ DONE | Project Setup |
| Phase 2 | ✅ DONE | Authentication |
| Phase 3 | ✅ DONE | UI Components |
| Phase 4 | ✅ DONE | API Integration |
| Phase 5 | ✅ DONE | Dashboard Pages |
| Phase 6 | ✅ DONE | Landing Page |
| Phase 7 | ✅ DONE | Internationalization (i18n) |
| Phase 8 | ✅ DONE | Deployment Config |

### Overall Progress: 8/8 Phases Complete ✅

---

## Tech Stack

- Next.js 14.2.35 (App Router)
- TypeScript
- Tailwind CSS + Radix UI + CVA
- AWS Amplify for Cognito authentication
- TanStack React Query
- next-intl 4.6.1 for i18n ✅

---

## What's Remaining

### 🔧 Deployment Tasks (Not Started)

| Task | Status | Description |
|------|--------|-------------|
| Get Cognito credentials | ⏳ | Extract from backend CloudFormation outputs |
| Create `.env.local` | ⏳ | Configure local environment variables |
| Test locally | ⏳ | Run `npm run dev` and verify auth flow |
| Set up AWS Amplify Hosting | ⏳ | Create Amplify app via AWS Console/CLI |
| Configure custom domain | ⏳ | Point mkpdfs.com to Amplify |
| Update Cognito callbacks | ⏳ | Change from templify.com to mkpdfs.com |

### 📝 Optional Enhancements

| Task | Priority | Description |
|------|----------|-------------|
| Add translations to remaining pages | Low | Register, forgot-password, settings, billing, etc. |
| Add language switcher component | Low | UI to switch between EN/ES |
| Add dark mode support | Low | Theme toggle in settings |
| Add more UI components | Low | Dialog, Select, Tabs as needed |

---

## Implementation Phases

### ✅ Phase 1: Project Setup - COMPLETED

- [x] Initialize Next.js 14 with TypeScript and Tailwind
- [x] Install dependencies (aws-amplify, radix-ui, tanstack/react-query, etc.)
- [x] Configure Tailwind with blue/purple theme
- [x] Set up folder structure

**Files created:**
- `mkpdfs-web/package.json`
- `mkpdfs-web/tailwind.config.ts`
- `mkpdfs-web/src/lib/utils.ts`
- `mkpdfs-web/src/types/index.ts`
- `mkpdfs-web/.env.example`

---

### ✅ Phase 2: Authentication - COMPLETED

- [x] Create `src/lib/auth.ts` - AWS Amplify Cognito integration
- [x] Create `src/providers/AuthProvider.tsx` - Auth context with session management
- [x] Create `src/providers/QueryProvider.tsx` - React Query provider
- [x] Implement auth pages

**Files created:**
- `mkpdfs-web/src/lib/auth.ts` - signIn, signUp, signOut, forgotPassword, getIdToken, etc.
- `mkpdfs-web/src/providers/AuthProvider.tsx`
- `mkpdfs-web/src/providers/QueryProvider.tsx`
- `mkpdfs-web/src/providers/index.ts`
- `mkpdfs-web/src/app/providers.tsx`
- `mkpdfs-web/src/app/[locale]/(auth)/login/page.tsx`
- `mkpdfs-web/src/app/[locale]/(auth)/register/page.tsx`
- `mkpdfs-web/src/app/[locale]/(auth)/forgot-password/page.tsx`
- `mkpdfs-web/src/app/[locale]/(auth)/reset-password/page.tsx`

---

### ✅ Phase 3: UI Components - COMPLETED

- [x] Button (with variants: default, destructive, outline, ghost, etc.)
- [x] Input, Label
- [x] Card (Card, CardHeader, CardTitle, CardDescription, CardContent, CardFooter)
- [x] Toast/Toaster
- [x] Spinner, PageLoader

**Files created:**
- `mkpdfs-web/src/components/ui/Button.tsx`
- `mkpdfs-web/src/components/ui/Input.tsx`
- `mkpdfs-web/src/components/ui/Label.tsx`
- `mkpdfs-web/src/components/ui/Card.tsx`
- `mkpdfs-web/src/components/ui/Toast.tsx`
- `mkpdfs-web/src/components/ui/Toaster.tsx`
- `mkpdfs-web/src/components/ui/Spinner.tsx`
- `mkpdfs-web/src/components/ui/index.ts`
- `mkpdfs-web/src/hooks/useToast.tsx`

---

### ✅ Phase 4: API Integration - COMPLETED

- [x] Create `src/lib/api.ts` - Authenticated API client
- [x] Create React Query hooks for all endpoints

**Files created:**
- `mkpdfs-web/src/lib/api.ts`
- `mkpdfs-web/src/hooks/useApi.ts`

**Hooks available:**
- `useProfile()` - GET /user/profile
- `useTemplates()` - GET /templates
- `useUploadTemplate()` - POST /templates/upload
- `useTokens()` - GET /user/tokens
- `useCreateToken()` - POST /user/tokens
- `useUsage()` - GET /user/usage
- `useGeneratePdf()` - POST /pdf/generate

---

### ✅ Phase 5: Dashboard Pages - COMPLETED

- [x] Dashboard Layout (Header + Sidebar)
- [x] `/dashboard` - Usage summary, quick actions
- [x] `/templates` - Template list, upload, delete
- [x] `/generate` - Template selector, JSON data input, generate button
- [x] `/api-keys` - Token list, create/delete tokens
- [x] `/usage` - Usage stats with progress bars
- [x] `/settings` - Profile settings, password change
- [x] `/billing` - Subscription status, plan comparison

**Files created:**
- `mkpdfs-web/src/components/layout/Sidebar.tsx`
- `mkpdfs-web/src/components/layout/Header.tsx`
- `mkpdfs-web/src/components/layout/index.ts`
- `mkpdfs-web/src/app/[locale]/(dashboard)/layout.tsx`
- `mkpdfs-web/src/app/[locale]/(dashboard)/dashboard/page.tsx`
- `mkpdfs-web/src/app/[locale]/(dashboard)/templates/page.tsx`
- `mkpdfs-web/src/app/[locale]/(dashboard)/generate/page.tsx`
- `mkpdfs-web/src/app/[locale]/(dashboard)/api-keys/page.tsx`
- `mkpdfs-web/src/app/[locale]/(dashboard)/usage/page.tsx`
- `mkpdfs-web/src/app/[locale]/(dashboard)/settings/page.tsx`
- `mkpdfs-web/src/app/[locale]/(dashboard)/billing/page.tsx`

---

### ✅ Phase 6: Landing Page - COMPLETED

- [x] Hero section with headline, CTA buttons, code example
- [x] Features section (6 feature cards)
- [x] Pricing section (4 tiers with features)
- [x] CTA section
- [x] Footer

**Files created:**
- `mkpdfs-web/src/app/[locale]/page.tsx`

**Pricing tiers:**
| Plan | PDFs/Month | Templates | Price |
|------|------------|-----------|-------|
| Free | 100 | 5 | $0 |
| Starter | 1,000 | 50 | $29 |
| Professional | 10,000 | 500 | $99 |
| Enterprise | Unlimited | Unlimited | Custom |

---

### ✅ Phase 7: Internationalization - COMPLETED

- [x] Configure next-intl with locales: `['en', 'es']`
- [x] Create message files: `en.json`, `es.json` (~350 strings each)
- [x] Add locale middleware for automatic detection
- [x] Update key pages to use `useTranslations()`
- [x] Wrap routes with `[locale]` segment

**Files created:**
- `mkpdfs-web/src/i18n/config.ts` - Locale configuration
- `mkpdfs-web/src/i18n/routing.ts` - Next-intl routing with Link/useRouter
- `mkpdfs-web/src/i18n/request.ts` - Server-side request config
- `mkpdfs-web/src/i18n/index.ts` - Exports
- `mkpdfs-web/src/i18n/messages/en.json` - English translations
- `mkpdfs-web/src/i18n/messages/es.json` - Spanish translations
- `mkpdfs-web/middleware.ts` - Locale detection middleware

**Pages with full translations:**
| Page | Status | Type |
|------|--------|------|
| Landing page | ✅ Translated | Server component |
| Login page | ✅ Translated | Client component |
| Dashboard page | ✅ Translated | Client component |
| Sidebar | ✅ Translated | Client component |

**Pages with translations available (not yet applied):**
| Page | Status |
|------|--------|
| Register | ⏳ Strings in JSON, page uses hardcoded |
| Forgot Password | ⏳ Strings in JSON, page uses hardcoded |
| Reset Password | ⏳ Strings in JSON, page uses hardcoded |
| Templates | ⏳ Strings in JSON, page uses hardcoded |
| Generate | ⏳ Strings in JSON, page uses hardcoded |
| API Keys | ⏳ Strings in JSON, page uses hardcoded |
| Usage | ⏳ Strings in JSON, page uses hardcoded |
| Settings | ⏳ Strings in JSON, page uses hardcoded |
| Billing | ⏳ Strings in JSON, page uses hardcoded |

**Notes:**
- Uses `localePrefix: 'as-needed'` - English URLs have no prefix, Spanish uses `/es`
- All routes support both `/en` and `/es` prefixes
- Translation strings exist for all pages; just need to update imports

---

### ✅ Phase 8: Deployment Config - COMPLETED

- [x] Create `amplify.yml` for AWS Amplify Hosting
- [x] Configure security headers in `next.config.mjs`
- [x] Update README with deployment instructions

**Files created:**
- `mkpdfs-web/amplify.yml`
- `mkpdfs-web/next.config.mjs`
- `mkpdfs-web/README.md`

---

## Environment Variables

```env
NEXT_PUBLIC_API_URL=https://apis.mkpdfs.com
NEXT_PUBLIC_COGNITO_USER_POOL_ID=<from-cf-export>
NEXT_PUBLIC_COGNITO_CLIENT_ID=<from-cf-export>
NEXT_PUBLIC_COGNITO_IDENTITY_POOL_ID=<from-cf-export>
NEXT_PUBLIC_SITE_URL=https://mkpdfs.com
```

Get values from CloudFormation:
```bash
aws cloudformation describe-stacks --stack-name mkpdfs-api-dev --query "Stacks[0].Outputs" --profile rocketeast
```

---

## Backend Updates Needed

- [x] Update Cognito callback URLs from `app.templify.com` to `mkpdfs.com`
- [x] Verify API domain configuration for `apis.mkpdfs.com`
- [x] Rename CloudFormation stack from `templify-api-*` to `mkpdfs-api-*`

---

## Build Status

```
✓ Compiled successfully
✓ Linting and checking validity of types
✓ Generating static pages (28/28)

Route (app)                              Size     First Load JS
┌ ○ /_not-found                          873 B    88.2 kB
├ ● /[locale]                            178 B    114 kB
├   ├ /en
├   └ /es
├ ● /[locale]/api-keys                   2.8 kB   167 kB
├ ● /[locale]/billing                    4.06 kB  108 kB
├ ● /[locale]/dashboard                  3.77 kB  196 kB
├ ● /[locale]/forgot-password            1.45 kB  165 kB
├ ● /[locale]/generate                   2.5 kB   167 kB
├ ● /[locale]/login                      2.59 kB  191 kB
├ ● /[locale]/register                   6.13 kB  175 kB
├ ● /[locale]/reset-password             2.29 kB  166 kB
├ ● /[locale]/settings                   4.74 kB  165 kB
├ ● /[locale]/templates                  2.53 kB  167 kB
└ ● /[locale]/usage                      2.15 kB  166 kB

+ First Load JS shared by all            87.3 kB
```

---

## Quick Start (for deployment)

```bash
# 1. Clone and install
cd mkpdfs-web
npm install

# 2. Get Cognito credentials from backend
aws cloudformation describe-stacks \
  --stack-name mkpdfs-api-dev \
  --query "Stacks[0].Outputs" \
  --profile rocketeast

# 3. Create .env.local with values from step 2

# 4. Test locally
npm run dev

# 5. Build for production
npm run build
```
