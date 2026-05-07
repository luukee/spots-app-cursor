---
name: timeout relogin reliability
overview: Improve reliability of the first login attempt after idle timeout by removing token-settle races and tightening the timeout-specific validation path without broad auth-flow changes.
todos:
  - id: align-settle-delay
    content: Add post-retry sign-in settle delay in useLogin already-authenticated recovery path
    status: completed
  - id: bounded-token-retry
    content: Implement bounded second-chance token acquisition in validateAndRedirect before showing login timeout error
    status: completed
  - id: validation-guard-hardening
    content: Review and tighten useAuthValidation trigger paths to avoid overlapping validateAndRedirect calls after timeout
    status: completed
  - id: auth-qa-pass
    content: Run timeout re-login, normal login, and reporting-as regression checks
    status: completed
isProject: false
---

# Fix Plan: Post-Timeout Re-Login Reliability

## Goal
Ensure users can log in successfully on the first attempt after idle session expiration, while preserving current behavior for normal login, Google OAuth, and reporting-as routing.

## Locked Defaults
- Conservative extra retry window in `validateAndRedirect` (target total added wait about 500-800ms max).
- Temporary `devLog` instrumentation is included for QA verification, then removed before merge unless explicitly requested to keep.

## Root Cause To Address
The login flow has a timing race after idle timeout/logout cleanup:
- `useLogin` has an `already authenticated` recovery path that signs out, retries `signIn`, then immediately calls validation.
- `validateAndRedirect` uses a short token availability budget and can fail before Amplify token state is fully settled.
- Users then retry manually, and later attempts succeed once Cognito session state stabilizes.

## Scope (Targeted)
- [app/_lib/hooks/auth/useLogin.ts](/Users/lukey/Sites/UT/SPOTS/spots-app/app/_lib/hooks/auth/useLogin.ts)
- [app/_lib/services/auth/login-service.ts](/Users/lukey/Sites/UT/SPOTS/spots-app/app/_lib/services/auth/login-service.ts)
- [app/_lib/hooks/auth/useAuthValidation.ts](/Users/lukey/Sites/UT/SPOTS/spots-app/app/_lib/hooks/auth/useAuthValidation.ts)

No changes to backend auth proxy route unless diagnostics show server-side errors are a co-factor.

## Implementation Steps
1. Standardize post-sign-in settle behavior in `useLogin`
- In the `isAlreadyAuthenticatedError` branch, after retry `signIn` success, add the same short settle delay used in the primary success path before calling `onLoginSuccess`.
- Keep existing `isValidationInProgress` protections and error handling.
- Add a concise comment explaining this race prevention for timeout/federated cleanup scenarios.

2. Make token acquisition in `validateAndRedirect` slightly more resilient
- Replace the single hard-fail token budget behavior with a bounded second chance strategy:
  - Attempt `fetchSessionWithBudget(currentBudget)` as today.
  - If no ID token, perform one additional short retry window (small fixed delay + one more budgeted fetch), then fail.
  - Keep this retry conservative (about 500-800ms added max), not a long fallback window.
- Keep total wait bounded to avoid regressing perceived login speed.
- Preserve current user-facing messaging, but ensure this path only errors after bounded retry is exhausted.

3. Prevent duplicate validation trigger collisions on login page
- In `useAuthValidation`, ensure auto-validation effects and Hub `signedIn` handling do not launch overlapping validation attempts during timeout recovery transitions.
- Reconfirm all entry points funnel through shared guard state (`isValidationInProgress`) and do not race with stale `authStatus` updates.

4. Add lightweight diagnostics for timeout re-login path
- Add dev-safe logs (existing `devLog` patterns) around:
  - retry sign-in success timestamp
  - first token-available timestamp
  - validation start/end status
- Keep logs free of sensitive data and removable behind existing dev logging controls.
- After QA confirms stability, remove temporary instrumentation before final merge unless explicitly requested to retain it.

## Validation Plan
1. Manual QA scenarios
- Idle timeout -> redirected to splash -> immediate password login attempt succeeds first try.
- Idle timeout -> immediate Google login attempt succeeds first try.
- Wrong password still fails immediately with expected error.
- Reporting-as users still route to `/auth/reporting-as` correctly.
- Normal fresh login (no prior timeout) remains unchanged.

2. Regression checks
- Ensure logout spinner/gate flags behavior still prevents auto-redirect loops.
- Confirm no extra duplicate requests to `/api/auth/login` in a single successful attempt.
- Verify rate-limit behavior remains unchanged functionally.

3. Code quality checks
- Run lint/TypeScript checks for edited files.
- If test harness exists for auth hooks/services, add or update focused tests for the retry boundary condition.

## Risks and Mitigations
- Risk: Increased login latency if retries are too long.
  - Mitigation: Strict bounded retry window and retain existing fast path.
- Risk: Reintroducing redirect loops on login page.
  - Mitigation: Keep central guard (`isValidationInProgress`) authoritative and verify effect dependencies.
- Risk: Timeout-specific fix impacting non-timeout login.
  - Mitigation: Limit behavior changes to retry/fallback branches; preserve existing happy path.

## Definition of Done
- First login attempt after idle timeout succeeds consistently in QA.
- No regressions in normal login, OAuth callback, or reporting-as flow.
- Lint/type checks pass for touched files.
- Logs confirm token/validation sequence is stable during timeout recovery.