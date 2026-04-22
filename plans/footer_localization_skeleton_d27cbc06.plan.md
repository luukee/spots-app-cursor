---
name: Footer localization skeleton
overview: Add a dedicated footer skeleton shown on every route (except where the footer is already hidden) while localized strings are not ready, so raw `Footer_*` keys never flash. Skeleton mirrors the real footer grid at all breakpoints, replaces theme/language controls with pulse placeholders, and exposes accessible busy state.
todos:
  - id: add-footer-skeleton
    content: Create FooterSkeleton.tsx mirroring Footer grid, pulse columns, logo block, toggle row (1 vs 2 pills), copyright bar, DecorativeBubbles; add a11y attrs
    status: completed
  - id: gate-footer
    content: "Update Footer.tsx: use hasLoadedStrings from useLocalization; return FooterSkeleton before QUICK_LINKS when false; keep /auth/reporting-as null"
    status: completed
  - id: tests
    content: Add minimal test(s) for skeleton vs loaded footer (mock localization or test FooterSkeleton + Footer gate)
    status: completed
isProject: false
---

# Footer skeleton until localization loads

## Problem

[`Footer.tsx`](spots-app/app/_components/layout/Footer.tsx) calls `getLocSetting` without fallbacks. Until `hasLoadedStrings` is true, [`getLocSetting`](spots-app/app/_lib/hooks/loc/useLocalizedStrings.ts) returns the **key** (`Footer_ProjectContact_Text`, etc.). The footer sits in the [root layout](spots-app/app/layout.tsx) next to `<main>`, so it paints **before** the login card’s `LoginPageLayout` localization gate—especially visible on slow networks.

## Approach

1. **Gate the real footer** on `hasLoadedStrings` from [`useLocalization()`](spots-app/app/_lib/hooks/loc/useLocalization.ts) (already exposes `hasLoadedStrings`).
2. **New presentational component** `FooterSkeleton` in the same feature area as the footer (e.g. [`spots-app/app/_components/layout/FooterSkeleton.tsx`](spots-app/app/_components/layout/FooterSkeleton.tsx)).
3. **Wire in [`Footer.tsx`](spots-app/app/_components/layout/Footer.tsx)** after existing pathname checks:
   - Keep early `return null` for `/auth/reporting-as` unchanged.
   - If `!hasLoadedStrings`, `return <FooterSkeleton isLoginPage={pathname === "/"} />` (or equivalent) so **`QUICK_LINKS` is never built** while keys would leak.
   - Otherwise keep current footer markup unchanged.

## `FooterSkeleton` layout (match real footer, all breakpoints)

Mirror the structure in [`Footer.tsx`](spots-app/app/_components/layout/Footer.tsx) lines 50–153:

- Outer `<footer>`: **same** `className` as today (`relative border-t border-border md:mt-12 py-12`) so swap is layout-stable.
- Inner `max-w-6xl mx-auto px-6`.
- **Grid**: `grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-8 mb-6` with three columns:
  - **Left**: one short bar (h4), three `text-sm`-height bars, gap, two lines (phone + email row widths).
  - **Center**: one short bar (h4), four link-height bars with `gap-2` / `flex-col`.
  - **Right**: centered `md:justify-end`, `md:col-span-2 lg:col-span-1` — rounded block ~logo footprint (`h-24` min width) instead of `<Image>`.
- **Bottom block** (`space-y-3 mt-12 text-center`):
  - **Toggles row**: `flex justify-center gap-4 mb-4` — **pulse pill(s)** only (no `ThemeToggle` / `LanguageToggle`): one pill when `isLoginPage`, two when not (matches real footer’s `{!isLoginPage && <LanguageToggle />}`).
  - **Copyright**: single centered narrow/medium-wide pulse bar (`text-sm` height).
- **DecorativeBubbles**: reuse the same [`DecorativeBubbles`](spots-app/app/_components/ui/DecorativeBubbles.tsx) block as the real footer (`variant="02"`, same `className` / `width` / `height`) so visual continuity matches production.

Use a shared inner constant for pulse bar classes (same pattern as [`LoginSkeleton`](spots-app/app/loading.tsx): `animate-pulse rounded-lg bg-muted` or existing gray tokens used elsewhere).

## Accessibility

On the skeleton `<footer>`:

- `role="contentinfo"` (redundant but fine on `footer`).
- `aria-busy="true"`, `aria-live="polite"`.
- `aria-label="Loading footer"` (literal English is OK—strings are not loaded yet).

Ensure skeleton bars use `aria-hidden="true"` so screen readers announce the region label, not a list of meaningless shapes.

## Tests (lightweight)

Add a small unit test (e.g. [`spots-app/app/_components/layout/__tests__/Footer.test.tsx`](spots-app/app/_components/layout/__tests__/Footer.test.tsx) or colocated) that wraps `Footer` in the same providers as other loc tests (see [`useLocalizedStrings.test.tsx`](spots-app/app/_lib/hooks/loc/__tests__/useLocalizedStrings.test.tsx) patterns): mock `hasLoadedStrings: false` and assert skeleton `aria-busy` / absence of raw `Footer_` key text; mock `hasLoadedStrings: true` and assert one real link label appears. If provider setup is heavy, testing `FooterSkeleton` in isolation plus a single integration test for `Footer` is acceptable.

## Out of scope

- Changing `getLocSetting` global behavior for all keys (would affect every consumer).
- `/auth/callback` special-case unless you later want the footer hidden there too (currently footer renders on callback).

## Files to touch

| File | Action |
|------|--------|
| [`spots-app/app/_components/layout/FooterSkeleton.tsx`](spots-app/app/_components/layout/FooterSkeleton.tsx) | **Create** |
| [`spots-app/app/_components/layout/Footer.tsx`](spots-app/app/_components/layout/Footer.tsx) | **Edit** — `hasLoadedStrings` gate + pass `isLoginPage` |
| Optional test under `app/_components/layout/__tests__/` | **Create** |
