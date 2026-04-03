---
name: Security Logging Monitoring
overview: Implement PHI-safe, structured security logging and baseline monitoring signals for auth-related API routes, starting with login. Replace ad-hoc console logs with consistent event schema and add operator-friendly outcomes/reason codes without exposing identifiers.
todos:
  - id: define-security-event-schema
    content: Define typed PHI-safe security event schema and helper in shared utils
    status: completed
  - id: refactor-login-route-logs
    content: Replace ad-hoc console logs in auth login route with structured security events
    status: completed
  - id: add-monitoring-fields
    content: Standardize outcome/reasonCode/requestId fields for queryable monitoring
    status: completed
  - id: document-event-taxonomy
    content: Document event types, reason codes, and example alert/query patterns
    status: completed
  - id: validate-no-phi-leakage
    content: Run lint/type checks and verify logs exclude identifiers/tokens
    status: completed
isProject: false
---

# Security Logging and Monitoring Plan

## Scope

- Target auth/security events first, with immediate focus on `[/Users/lukey/Sites/UT/SPOTS/spots-app/app/api/auth/login/route.ts](/Users/lukey/Sites/UT/SPOTS/spots-app/app/api/auth/login/route.ts)`.
- Introduce shared structured logging utilities in `[/Users/lukey/Sites/UT/SPOTS/spots-app/app/_lib/utils/dev-logger.ts](/Users/lukey/Sites/UT/SPOTS/spots-app/app/_lib/utils/dev-logger.ts)` and `[/Users/lukey/Sites/UT/SPOTS/spots-app/app/_lib/utils/server-log.ts](/Users/lukey/Sites/UT/SPOTS/spots-app/app/_lib/utils/server-log.ts)`.
- Keep current behavior/response contracts intact (no auth flow changes).

## Event Model (PHI-safe)

- Define a minimal security event contract:
  - `eventType` (e.g., `auth.login.blocked`, `auth.login.failed`, `auth.login.succeeded`)
  - `outcome` (`allow`, `deny`, `error`)
  - `reasonCode` (e.g., `rate_limit`, `invalid_origin`, `lambda_error`, `session_create_error`)
  - `requestId` (generated per request), `route`, `statusCode`, `timestamp`
  - Optional safe metrics (`latencyMs`, `retryAfterSeconds`, `rateLimitRemaining`)
- Explicitly exclude email, tokens, cookies, family IDs, REDCap IDs, and raw request/response bodies.

## Implementation Steps

1. Add shared helper(s) for structured security events (typed interfaces + one logging function).
2. Refactor `console.warn/error/log` in login route to emit structured events.
3. Preserve existing `Server-Timing`; include timing values in event metadata where useful.
4. Add lightweight monitoring counters via log fields (`eventType`, `reasonCode`, `outcome`) to support CloudWatch/Datadog queries and alerts.
5. Document the event taxonomy and query examples in the existing security plan/doc area.

## Initial Event Coverage (login route)

- Rate-limit denied request (`429`)
- Invalid origin/CSRF block (`403`)
- Upstream Lambda non-OK responses (including `502`)
- Session creation success/failure
- Route-level exception path (`500`)
- Successful login (without user identifiers)

## Validation

- Verify logs still appear in local dev and production environments.
- Confirm no direct identifier leakage (especially current success log with email).
- Ensure each denial/error path maps to one stable `eventType` + `reasonCode`.
- Run lint/typecheck for touched files and a manual auth smoke pass (login success, rate-limit, invalid origin simulation).

