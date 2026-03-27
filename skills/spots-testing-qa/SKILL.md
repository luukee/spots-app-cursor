---
name: spots-testing-qa
description: >-
  Guides automated testing and manual QA for SPOTS: Jest unit/integration tests,
  Playwright E2E, the tsx test-runner, and role/data scenarios (parent/child,
  family_id). Use when adding tests, debugging CI failures, or planning coverage
  for auth, symptoms, cart, or API routes. Triggers: Jest, Playwright, E2E,
  test-runner, coverage, regression, QA, mock fetch, jsdom.
---

# SPOTS testing and QA

## Test stacks (what runs where)

| Command | Role |
|---------|------|
| `npm test` | `test-runner.ts` — lightweight checks; **client-only cases skip in Node** (see runner header comment). |
| `npm run test:jest` | **Primary** unit/integration suite (Jest + **jsdom**). Use for hooks, services, utils, and mocked `fetch`. |
| `npm run test:jest:coverage` | Coverage over `app/`, `lib/`, `components/` per `jest.config.js`. |
| `npm run test:all` | `npm test` then Jest with coverage. |
| `npm run test:e2e` | **Playwright** — `e2e/`; needs app reachable (see `playwright.config.ts`, `E2E_BASE_URL`). |

Before changing Jest behavior, read **`jest.config.js`**: `e2e/` is ignored; some files are intentionally excluded (e.g. route tests that need polyfills — coverage may live in service-level tests instead).

## Where to put new tests

- **Business logic / services:** co-locate `__tests__/*.test.ts` under the service folder (matches existing `app/_lib/services/**/__tests__`).
- **Hooks / React:** `*.test.tsx` next to hooks or under `__tests__` with Testing Library patterns used elsewhere (see `useSymptomReporting.test.tsx`, `useLocalizedStrings.test.tsx`).
- **API route contracts:** Prefer testing handlers via **extracted logic** or **service mocks** when full `NextRequest` setup is fragile; follow the delete-route note in Jest config if adding similar cases.
- **E2E:** `e2e/*.spec.ts` — keep flows stable and minimal; one clear assertion path per scenario where possible.

## Design principles

- **Unit tests:** Mock external I/O (`fetch`, Gateway, Cognito) with explicit fixtures; assert shapes and status handling. No `any` in new tests — type mocks or use `unknown` + guards.
- **Integration (Jest):** Still no real REDCap; use **controlled** responses. Production code paths must not substitute mock JSON for live data (see `spots-api-redcap-integration`).
- **E2E:** Run against **dev/staging** or local with real backend when validating true integration; document env needs in the spec or `docs/testing/e2e.md`. Avoid committing secrets or real PHI — use study-approved test accounts and **synthetic** data (`spots-security-privacy`).
- **Happy path + failures:** Session expiry, 401/403, network errors, and empty states are high value for symptom and auth flows.

## SPOTS-specific QA scenarios

- **Roles:** Parent vs child behavior, `family_id` / reporting context — see `docs/testing/family-id-testing.md`, `docs/testing/family-id-quick-test.md`, and login/reporting plans under `docs/testing/`.
- **Symptoms:** Cart, submit, delete (soft delete / list refresh), Problem Station — align with `.cursor/rules` feature notes.
- **Accessibility:** For critical flows, note keyboard/focus and labels in manual QA or add checks where the project uses tooling.

## Debugging

- **Client instrumentation:** Strict **CSP** blocks debug `fetch` to localhost from the browser — use `console` patterns from `.cursor/rules/debug-mode-csp.mdc` for client-side investigation.
- **CI:** Playwright `forbidOnly` and retries depend on `CI`; keep **no `.only`** in committed specs.

## Doc pointers

- E2E setup: `docs/testing/e2e.md`
- Family / reporting: `docs/testing/family-id-testing.md`, `docs/testing/login-and-reporting-test-plan.md`

## Related skills

- **`spots-security-privacy`** — redact logs and fixtures; safe test users only.
- **`spots-api-redcap-integration`** — contract expectations when mocking Gateway responses.
- **`spots-pro-ctcae-business`** — what to assert when testing question progression or cart validation.
