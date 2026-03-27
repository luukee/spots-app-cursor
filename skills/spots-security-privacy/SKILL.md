---
name: spots-security-privacy
description: >-
  Applies security and privacy practices for the SPOTS healthcare web app
  (Next.js, Amplify/Cognito, REDCap-backed APIs). Use when implementing or
  reviewing authentication, API routes, logging, error handling, env config,
  data handling, debugging, or any feature that could expose PHI or weaken
  least-privilege. Triggers: security, privacy, HIPAA, PHI, COPPA, secrets,
  tokens, cookies, session, audit, PII, vulnerability, XSS, CSRF, CSP.
---

# SPOTS security and privacy

SPOTS serves pediatric oncology families; treat data as sensitive health-adjacent information. Prefer **fail closed**, **least privilege**, and **minimal disclosure**.

## Secrets and configuration

- Never commit API keys, REDCap tokens, Lambda secrets, or `.env.local` contents. Use env vars and platform secrets (e.g. Amplify). Confirm `.gitignore` covers local env files.
- Do not log environment variable names **and** values together in a way that could leak in production logs.
- **Client bundles:** only `NEXT_PUBLIC_*` (or equivalent) may be exposed to the browser. Everything else stays server-only. Review that no server secrets are imported into client components.

## PHI, identifiers, and logs

Assume session payloads, REDCap fields, symptom reports, and family identifiers may constitute **PHI** or sensitive data.

- **Do not** `console.log`, debug-print, or ship analytics events that include names, emails, medical details, `spots_user_id` / family IDs, raw API bodies from symptom endpoints, or Cognito subject tokens.
- **Do not** paste production data, HAR captures, or full API responses into issues, docs, or chat unless redacted for a specific approved workflow.
- Prefer structured server-side logging with **redaction** and message templates; avoid dumping full `request` / `response` objects.
- When showing users an error, use **generic** copy (“Something went wrong”) and log details **only** on the server with care.

## Authentication and session

- Enforce auth for protected routes and API handlers per project patterns (e.g. Amplify server helpers). Do not trust client-only checks for authorization.
- Session expiry and logout flows should clear client state that could show another user’s context; avoid leaving cached PHI in `localStorage` unless explicitly required and reviewed.
- Treat ID tokens and refresh material as **opaque**; never echo them in UI or expose them in URLs.

## API routes and backend calls

- Validate and **normalize inputs** on the server (types, allowed ranges, required fields). Do not rely on the browser for security boundaries.
- Forward identity/session to the backend **only** through established patterns (headers, cookies, server-side token retrieval). Do not add “debug” query params that bypass auth.
- When proxying to Developer 1 / REDCap Lambdas, preserve **intended** user context (e.g. parent vs child / `family_id` rules) and do not widen access accidentally.

## Browser and UI

- Avoid `dangerouslySetInnerHTML` with untrusted or partially trusted content. If unavoidable, sanitize per project standards.
- Prefer parameterized data binding over string-built HTML. For user‑generated text, escape appropriately for the context.
- Follow existing **CSP** / security headers rules for the repo (see project rules on CSP if present).

## Third parties and retention

- Before adding analytics, chat widgets, or external scripts: confirm **BAA** / institutional policy compliance and data flow; default to **no** new trackers for health flows.
- Do not store more data than needed; align with study protocol and retention policies.

## Review checklist (short)

Use when touching auth, API, or health data:

- [ ] No secrets or PHI in client code, logs, or commits
- [ ] Errors: safe for users, detailed only server-side and redacted
- [ ] AuthZ matches role (parent/child/care team as designed)
- [ ] Inputs validated server-side
- [ ] Dependencies/links for new integrations vetted for privacy

## When uncertain

Flag privacy or compliance risk in review notes; suggest stakeholder (PI, security, IRB) review rather than guessing.
