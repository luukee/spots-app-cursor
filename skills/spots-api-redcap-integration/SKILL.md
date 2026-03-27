---
name: spots-api-redcap-integration
description: >-
  Guides Next.js API routes and client services that call the SPOTS Lambda/API
  Gateway layer (REDCap-backed). Use when adding or changing app/api proxies,
  symptom or family flows, or data fetching that must hit real backend data.
  Emphasizes no silent mock fallbacks, correct auth forwarding, and
  parent/child REDCap context. Triggers: REDCap, Lambda, API Gateway, BFF,
  app/api, proxy, spots backend, family_id, repeat_id, symptom_deleted.
---

# SPOTS API and REDCap integration (frontend BFF)

The browser **does not** talk to REDCap directly. The Next.js app calls **AWS API Gateway → Lambda** (Developer 1), which owns REDCap access. Treat the backend contract as the source of truth.

## Configuration

- Use the shared gateway base URL from `@/app/_lib/config/api-gateway` (`API_GATEWAY_BASE_URL`), derived from `NEXT_PUBLIC_AWS_API_GATEWAY`. If it is missing, return a **clear 500** (or equivalent) — do not invent a URL.
- Keep query paths consistent with existing routes (e.g. trailing slash behavior on `API_GATEWAY_BASE_URL` — it is normalized to end with `/` in config).

## No silent fallbacks (development discipline)

- **Do not** return canned JSON, local `LOCAL/redcap` files, or stub data when the backend is down or misconfigured just to “make the UI work.” That hides integration bugs.
- **Do** surface failures explicitly: appropriate HTTP status, structured error payload, and safe user-facing message (see `spots-security-privacy` for PHI-safe errors).
- Unit tests may mock `fetch`; runtime paths should assume real Gateway responses unless the feature is explicitly test-only (e.g. routes under `app/api/test/`).

## Authentication

- Require `Authorization` (Bearer) when the Lambda expects it. Pass the header **through unchanged** if the authorizer is sensitive to format (match patterns in existing routes such as `app/api/symptoms/delete/route.ts`).
- Return **401** when the client omits auth for protected operations.

## Parent vs child / REDCap account context

- Some operations use the **parent’s** REDCap account when the logged-in user is a child. Follow established patterns: e.g. optional JSON body with `family_id` where the backend and `.cursor/rules` / feature docs specify it.
- Do not widen access: only send fields the backend documents; do not add bypass query parameters.

## Request shaping

- Mirror backend method and query params exactly (`repeat_id`, filters, etc.). Use `encodeURIComponent` for query values.
- Use `cache: 'no-store'` (or project standard) for user-specific health data unless a route is explicitly static and approved.
- Preserve `Content-Type: application/json` when sending bodies.

## Responses and errors

- On non-OK Gateway responses, parse body when JSON; avoid logging full raw bodies in production (redact; see security skill).
- Prefer shared helpers where they exist (e.g. `createRedcapErrorResponse` from `@/app/_lib/utils/redcap-utils`) for consistent JSON error shape **when appropriate** — match neighboring routes in the same folder.
- Map unknown failures to a controlled 5xx / 4xx with a message suitable for the client.

## Types and services

- Define TypeScript interfaces for request/response shapes in `lib/types` or co-located modules; avoid `any`; use `unknown` + narrowing for untyped Gateway payloads until unknown is narrowed.
- Centralize repeated Gateway calls in small service modules so API routes stay thin.

## Checklist for new proxies

- [ ] Uses `API_GATEWAY_BASE_URL` (or existing project equivalent for that resource)
- [ ] Auth and headers match backend expectations
- [ ] Child/parent context matches documented body/query contract
- [ ] No production mock fallback; errors are explicit
- [ ] Responses typed or validated at boundaries
- [ ] Logging avoids PHI (see `spots-security-privacy`)

## Related code

- Gateway config: `app/_lib/config/api-gateway.ts`
- Example PATCH proxy: `app/api/symptoms/delete/route.ts`
- REDCap-oriented helpers: `app/_lib/utils/redcap-utils.ts`
