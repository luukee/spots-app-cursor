---
name: Mobile Header User Switching
overview: Update the mobile header so parent users see the child selector by default, and add a gated `Switch User` action that routes eligible accounts to `/auth/reporting-as`. Eligibility will follow reporting-as semantics (same Cognito email mapped to multiple REDCap records).
todos:
  - id: header-mobile-child-selector
    content: Move mobile child selector out of Profile dropdown and render it by default for parent/admin users.
    status: completed
  - id: header-switch-user-gating
    content: Add reporting-as eligibility boolean in Header based on current user email and children email match.
    status: completed
  - id: header-switch-user-action
    content: Add mobile `Switch User` button linking to `/auth/reporting-as` with proper menu state cleanup.
    status: completed
  - id: header-regression-checks
    content: Verify mobile/desktop menu behavior for parent eligible, parent non-eligible, and child users.
    status: completed
isProject: false
---

# Mobile Header: Default Child Selector + Switch User

## Scope

- Update only the mobile menu behavior in `[/Users/lukey/Sites/UT/SPOTS/spots-app/app/_components/layout/Header.tsx](/Users/lukey/Sites/UT/SPOTS/spots-app/app/_components/layout/Header.tsx)`.
- Reuse existing reporting-as route at `[/Users/lukey/Sites/UT/SPOTS/spots-app/app/auth/reporting-as/page.tsx](/Users/lukey/Sites/UT/SPOTS/spots-app/app/auth/reporting-as/page.tsx)` without changing its core selection flow.

## Implementation Plan

- Refactor the mobile user section in `Header` so the child selector block is rendered directly in the main mobile menu for parent/admin users (not nested behind `Profile` dropdown state).
- Keep desktop menu behavior unchanged.
- Add a computed eligibility flag in `Header` for showing `Switch User` only when account matches reporting-as conditions.
  - Proposed rule in UI layer: parent/admin with family context and at least one child record sharing the same `email_primary` as the logged-in user.
  - This mirrors backend reporting-as trigger (`same email -> multiple REDCap records`) without calling login endpoints that create sessions.
- Add a `Switch User` button/link in mobile user actions that navigates to `/auth/reporting-as` and closes mobile/menu states cleanly.
- Ensure menu state cleanup is consistent (`setUserMenuOpen(false)` + `setMobileMenuOpen(false)` where applicable) to avoid stale open dropdowns.

## Key Logic To Introduce

```ts
const normalizedUserEmail = (userBusiness?.email_primary || "").trim().toLowerCase();
const isReportingAsEligible =
  isParent &&
  hasFamilyId &&
  normalizedUserEmail.length > 0 &&
  children.some((child) => (child.email_primary || "").trim().toLowerCase() === normalizedUserEmail);
```

## Validation

- Parent/admin with eligible shared-email child records:
  - Open mobile menu -> child selector is visible immediately.
  - `Switch User` is visible and routes to `/auth/reporting-as`.
- Parent/admin not eligible:
  - Child selector still visible by default.
  - `Switch User` hidden.
- Child user:
  - No child selector and no `Switch User` button.
- Confirm desktop avatar menu still works as before.

## Notes

- This avoids using `/api/auth/login` for eligibility checks, since that endpoint can create login records as a side effect on success.
- If you later want exact eligibility from backend (instead of email-match heuristic), we can add a dedicated side-effect-free eligibility endpoint in a separate change.

