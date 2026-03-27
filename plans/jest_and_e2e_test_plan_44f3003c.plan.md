---
name: Jest and E2E Test Plan
overview: Add Jest tests for login types, reporting-as, symptom submit/delete contracts, and session/Problem Station logic; then add a Playwright E2E suite for one critical path (login, submit symptom, Problem Station, refresh, delete) against a real or test environment.
todos: []
isProject: false
---

# Jest + E2E test plan for login and symptom flows

## Goal

Reduce manual QA by:

1. **Jest**: Lock down request/response contracts, role/reporting-as logic, and session/Problem Station state so regressions are caught in seconds.
2. **E2E (Playwright)**: Automate one critical path through the real app (login, submit symptom, see in Problem Station, refresh, delete) for pre-release or scheduled runs.

Manual testing then focuses on "did REDCap get the data?" and spot-checks; the heavy logic is covered by Jest and one full flow by E2E.

---

## Part 1: Jest tests

### 1.1 Auth and login flow

**Existing:** [app/_lib/services/auth/**tests**/role-service.test.ts](app/_lib/services/auth/__tests__/role-service.test.ts) — keep as-is.

**New: login-service.performValidation**

- **File:** `app/_lib/services/auth/__tests__/login-service.test.ts`
- **Approach:** Mock `fetch` (global or via jest.mock of `node-fetch`/global fetch), `router.push`, `reportingAsPendingStore.set`, and `userCookies.setUser`. Mock `trackAuthLoginApi` and `devLog` to avoid side effects.
- **Cases:**
  - **Single child:** `fetch` returns `{ success: true, user: { role: 'C', login_id: 'L1', ... } }` → assert `router.push('/home')` called, `userCookies.setUser(result.user)` called, `reportingAsPendingStore.set` not called.
  - **Single parent:** same with `role: 'P'` → redirect `/home`, user cookie set.
  - **Child + parent (reporting-as):** `fetch` returns `{ success: true, needsReportingAs: true, candidates: [{ role: 'C', ... }, { role: 'P', ... }] }` → assert `reportingAsPendingStore.set({ email, candidates })`, `router.push('/auth/reporting-as')`, no `/home` redirect.
  - **Validation failure:** `{ success: false, error: '...' }` → `setError` called with message, no redirect.
  - **429 rate limit:** status 429 → `setError` with rate-limit message, return `false`.

**New: reporting-as helpers (extract then test)**

- **Extract:** From [app/auth/reporting-as/page.tsx](app/auth/reporting-as/page.tsx), move pure helpers into a module, e.g. `app/_lib/utils/reporting-as-utils.ts`:
  - `mapCandidateRoleToChoice(candidate)` — P → "parent", else "child"
  - `isParentOrAdminRole(role)` — true for "P" or "A"
  - `getCandidateButtonLabel(candidate, choice)` — name fallbacks
  - `buildButtonOptions(pending, childNameBySpotsUserId, parentDisplayName)` — one option per child candidate, one for parent
- **Update:** reporting-as page to import from that module.
- **File:** `app/_lib/utils/__tests__/reporting-as-utils.test.ts`
- **Cases:** P-only candidate → one parent option; C-only → one child option; P+C candidates → parent + child options; `isParentOrAdminRole('P'|'C'|'A'|'')`; label from first_name, child_name, spots_user_id, fallback "Parent"/"Child".

### 1.2 Symptom submit contract

- **File:** `app/_lib/services/symptoms/__tests__/symptom-service-submit.test.ts`
- **Mocks:** `fetch` (global or injectable), `TokenService.getIdToken()` returning a dummy token.
- **Subject:** [SymptomService.submitSymptomReport](app/_lib/services/symptoms/symptom-service.ts) (use `new SymptomService()` or the exported singleton; if singleton, avoid mutating shared caches in ways that break other tests).
- **Cases:**
  - **Success:** POST to `/api/symptoms` with body containing `symptomId`, `loginId`, `individualId`, `responses`; response 200 with `reportId`, `repeatId` (or `record.redcap_repeat_instance`) → returns object with that `reportId` / `redcapRepeatInstance`.
  - **Parent reporting for child:** report includes `for_child: true` and `child_individual_id` when provided → request body includes them.
  - **Failure:** response 4xx/5xx with `{ error: '...' }` → throws Error with that message.

### 1.3 Session symptom service (Problem Station list and merge)

- **File:** `app/_lib/services/symptoms/__tests__/session-symptom-service.test.ts`
- **Subject:** [sessionSymptomService](app/_lib/services/symptoms/session-symptom-service.ts) singleton; use `clearAll()` in `beforeEach` so tests are isolated.
- **Cases:**
  - **Add and get:** `addSymptom(loginId, record)` then `getSessionSymptoms(loginId)` returns that record; second add same loginId/individual_id/symptom_id updates; different `individual_id` → both present.
  - **Filter deleted:** `addSymptom(loginId, { ...record, symptom_deleted: '1' })` then `getSessionSymptoms(loginId)` → record excluded.
  - **Merge:** `addSymptom(loginId, sessionOnlyRecord)` then `loadSessionSymptoms(loginId, apiSymptoms)` → merged list; session record keeps or receives `redcap_repeat_instance` / `symptom_report_id` from API when composite key matches.
  - **Clear:** `clearSession(loginId)` then `getSessionSymptoms(loginId)` → empty.

### 1.4 Delete symptom contract (optional, small)

- **Option A (recommended):** In the same symptom-service test file or a dedicated `symptom-service-delete.test.ts`, mock `fetch` for PATCH to `/api/symptoms/delete?repeat_id=...`; test the **client** that calls it (e.g. a thin wrapper or the code path in SymptomsSummary that builds the request). Assert URL and method.
- **Option B:** Add a Next.js route handler test for [app/api/symptoms/delete/route.ts](app/api/symptoms/delete/route.ts) that mocks the Lambda `fetch`; assert 200 → `{ success: true }`, 401/500 → JSON error. Requires running in Node with Next request/response mocks.

### 1.5 useSymptomReporting hook (optional)

- **File:** `app/_lib/hooks/symptoms/__tests__/useSymptomReporting.test.ts`
- Mock `symptomService.submitSymptomReport`. Render hook, call `submitReport(report)` → success: `submitting` toggles, no `error`, returns value; failure: `error` set, throws.

---

## Part 2: E2E with Playwright

**Why Playwright:** No E2E stack exists yet (no Playwright/Cypress in repo). Playwright is a good fit for Next.js, has built-in auth storage, and runs well in CI.

### 2.1 Setup

- Add devDependencies: `@playwright/test`.
- Add `playwright.config.ts` at repo root: baseURL (e.g. `http://localhost:3000`), one browser (e.g. chromium), timeout and retries; optional `projects` for smoke vs full later.
- Add npm script: `"test:e2e": "playwright test"` (and optionally `test:e2e:ui`).
- **Environment:** E2E will hit a running app (local or deployed test env). Use env vars for base URL and test credentials (e.g. `E2E_BASE_URL`, `E2E_CHILD_EMAIL`, `E2E_CHILD_PASSWORD` or OAuth test account). Document in README or `docs/testing/e2e.md` that a test backend/REDCap (or dedicated test project) is required for full flow.

### 2.2 Critical path spec

- **File:** `e2e/symptom-flow.spec.ts` (or `tests/e2e/symptom-flow.spec.ts`).
- **Flow:**
  1. Go to base URL (e.g. `/` or `/login`).
  2. Log in (email/password or Google test account — depending on how your login UI works; if Cognito Hosted UI, use `storageState` or cookie injection after a manual login once, or use a test user that can be automated).
  3. Wait for redirect to `/home` (or dashboard).
  4. Navigate to a page where a symptom can be submitted (e.g. body parts or search).
  5. Select a symptom and complete minimal steps to submit (e.g. pick severity/frequency if required, submit).
  6. Assert: Problem Station shows the new symptom (e.g. open sidebar/list, check for the symptom term or count).
  7. Reload page; assert Problem Station still shows the symptom (confirms merge or refetch).
  8. Delete the symptom from Problem Station (click delete, confirm if needed).
  9. Assert: Symptom no longer in the visible list (soft-delete: record still in REDCap but UI hides it).
- **Auth note:** If login is Google OAuth only, options are (a) use Playwright’s `storageState` after one manual login to reuse session, or (b) add a test-only login path (e.g. env-based test user/password) for E2E. Document the chosen approach.

### 2.3 CI / local usage

- **Local:** `npm run build && npm run start` (or point to a test deployment), then `npm run test:e2e` with `E2E_BASE_URL` set.
- **CI:** Add a job that starts the app (or uses a test environment URL), runs `test:e2e`, and fails on flaky or environment issues. Keep E2E optional or on a schedule if CI time is a concern.

---

## Implementation order


| Phase | Task                                                                              | Dependencies               |
| ----- | --------------------------------------------------------------------------------- | -------------------------- |
| 1     | Reporting-as utils: extract to `reporting-as-utils.ts`, add unit tests            | None                       |
| 2     | Login-service tests: `login-service.test.ts` (performValidation)                  | Mock fetch, router, stores |
| 3     | Session-symptom-service tests                                                     | clearAll in beforeEach     |
| 4     | Symptom-service submit tests (submit contract)                                    | Mock fetch, TokenService   |
| 5     | Optional: delete contract test; useSymptomReporting test                          | Same mocks as submit       |
| 6     | Playwright: install, config, env docs                                             | None                       |
| 7     | E2E: one critical path spec (login → submit → Problem Station → refresh → delete) | Test env + auth strategy   |


---

## Out of scope (by design)

- **Jest:** No real Cognito, REDCap, or browser; no testing that "REDCap has the record" (that stays manual or backend tests).
- **E2E:** No full matrix of all pages (body parts, activities, past problems, feelings, search) or all login types in E2E initially; one critical path covers the main flow. Expand later if needed.
- **Child + parent reporting-as in E2E:** Can be added as a second spec (login with shared-account test user, choose Child then submit, then choose Parent then submit) once the first spec is stable.

---

## Files to add or change (summary)

**New files**

- `app/_lib/utils/reporting-as-utils.ts` (extracted from reporting-as page)
- `app/_lib/utils/__tests__/reporting-as-utils.test.ts`
- `app/_lib/services/auth/__tests__/login-service.test.ts`
- `app/_lib/services/symptoms/__tests__/session-symptom-service.test.ts`
- `app/_lib/services/symptoms/__tests__/symptom-service-submit.test.ts`
- Optional: `app/_lib/services/symptoms/__tests__/symptom-service-delete.test.ts`, `app/_lib/hooks/symptoms/__tests__/useSymptomReporting.test.ts`
- `playwright.config.ts`
- `e2e/symptom-flow.spec.ts` (or `tests/e2e/...`)
- `docs/testing/e2e.md` (or section in existing testing doc): env vars, auth, how to run

**Modified files**

- `app/auth/reporting-as/page.tsx` — import helpers from `reporting-as-utils.ts`
- `package.json` — add `@playwright/test`, scripts `test:e2e` (and optionally `test:e2e:ui`)

