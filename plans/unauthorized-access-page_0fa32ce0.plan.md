---
name: unauthorized-access-page
overview: Add a dedicated Unauthorized Access page and route users there when role checks fail (e.g., child opening `/dashboard`). Reuse localization key `UnauthorizedText` for the main message and enforce admin-only access on dashboard at route level.
todos:
  - id: add-unauthorized-page
    content: Add `app/(pages)/unauthorized/page.tsx` with localized `UnauthorizedText` main message.
    status: completed
  - id: wire-protected-route-redirect
    content: Change role-denied behavior in `ProtectedRoute` from inline access-denied UI to redirect `/unauthorized`.
    status: completed
  - id: enforce-dashboard-admin-role
    content: Require `UserRole.ADMIN` in dashboard layout ProtectedRoute wrapper.
    status: completed
  - id: verify-role-flow
    content: Validate admin/non-admin behavior for `/dashboard` and run lints for changed files.
    status: completed
isProject: false
---

# Unauthorized Access Route Plan

## What I found

- `UnauthorizedText` already exists and is used in `[app/_components/auth/ErrorDisplay.tsx](/Users/lukey/Sites/UT/SPOTS/spots-app/app/_components/auth/ErrorDisplay.tsx)`.
- `/dashboard` is wrapped by `[app/(pages)/dashboard/layout.tsx](/Users/lukey/Sites/UT/SPOTS/spots-app/app/(pages)`/dashboard/layout.tsx) but currently does not pass a `requiredRole`.
- Authorization fallback in `[app/_components/auth/ProtectedRoute.tsx](/Users/lukey/Sites/UT/SPOTS/spots-app/app/_components/auth/ProtectedRoute.tsx)` currently renders inline “Access denied” text instead of navigating to a dedicated page.

## Implementation

- Create a dedicated unauthorized page at `[app/(pages)/unauthorized/page.tsx](/Users/lukey/Sites/UT/SPOTS/spots-app/app/(pages)`/unauthorized/page.tsx).
- Render the page as a client component using `useLocalization()` and set the main body copy to `getLocSetting('UnauthorizedText')`.
- Include a concise heading and a clear navigation action (back to home/sign-in) consistent with existing auth UI patterns.
- Update `[app/_components/auth/ProtectedRoute.tsx](/Users/lukey/Sites/UT/SPOTS/spots-app/app/_components/auth/ProtectedRoute.tsx)`:
  - Keep unauthenticated users redirecting to `/`.
  - On failed `requiredRole` check, redirect with `router.replace('/unauthorized')` instead of inline error box.
- Update `[app/(pages)/dashboard/layout.tsx](/Users/lukey/Sites/UT/SPOTS/spots-app/app/(pages)`/dashboard/layout.tsx) to enforce admin role by passing `requiredRole={UserRole.ADMIN}` to `ProtectedRoute`.

## Validation

- Manually verify:
  - Admin user can open `/dashboard`.
  - Child/non-admin user navigating to `/dashboard` is redirected to `/unauthorized`.
  - Unauthorized page main text resolves from `UnauthorizedText`.
- Run lint diagnostics on edited files and address any new issues.

