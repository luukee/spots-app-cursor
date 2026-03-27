---
name: Login Page Skeleton Only
overview: "Add a Next.js loading UI for the root (login) route only: create app/loading.tsx that shows a LoginSkeleton matching the login form and layout. No changes to auth flow, page structure, PPR, or Server Actions."
todos: []
isProject: false
---

# Login Page Skeleton Loading UI

## Goal

Add a skeleton loading state for the login page at `/` so users see a layout-matched placeholder instead of a blank or generic spinner while the page loads. No changes to the existing client-side login flow or page structure.

## Current state

- Root page: [app/page.tsx](spots-app/app/page.tsx) is a client component that renders [LoginPageLayout](spots-app/app/_components/auth/LoginPageLayout.tsx) and [LoginForm](spots-app/app/_components/auth/LoginForm.tsx).
- No [app/loading.tsx](spots-app/app/loading.tsx) exists; the home route has [app/(pages)/home/loading.tsx](spots-app/app/(pages)/home/loading.tsx) as a reference for skeleton patterns (`animate-pulse`, `rounded-lg bg-gray-200`).

## What to add

### 1. Create `app/loading.tsx`

- **Default export**: A component that renders the login skeleton (so Next.js shows it automatically when the root segment is loading).
- **Named export**: `LoginSkeleton` – the same skeleton component, so it can be reused (e.g. in a future Suspense fallback) and so the loading file clearly exposes the skeleton.

The file can be a single module: define `LoginSkeleton` and default-export a wrapper that returns `<LoginSkeleton />`, or default-export the skeleton directly and also export it as a named export (re-export pattern).

### 2. LoginSkeleton layout (match LoginPageLayout + LoginForm)

Mirror the structure and key classes so the transition from skeleton to real content has minimal layout shift:


| Area       | Real UI reference                                                                            | Skeleton                                                                                                                                                                                                                                                                                                                 |
| ---------- | -------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Outer      | `LoginPageLayout`: `relative min-h-screen overflow-hidden`, section `max-w-6xl`, `px-4 py-8` | Same wrapper; optional top-left decorative placeholder (or omit for simplicity).                                                                                                                                                                                                                                         |
| Header     | `WelcomeLogo` in section                                                                     | Logo area: e.g. one or two `animate-pulse` bars (text + image placeholders), `my-6`.                                                                                                                                                                                                                                     |
| Main card  | `max-w-3xl rounded-2xl border-2 border-[#5AA1D9] bg-white dark:bg-gray-800 p-6 shadow-sm`    | Same card; inside:                                                                                                                                                                                                                                                                                                       |
| Form block | `LoginForm`: h2 `mb-4 text-2xl`, form `space-y-4 mb-4`                                       | Heading: `h-6` or `h-8` bar, `mb-4`. Two input-height bars (`h-12` or `py-3` height), `rounded-lg`. Primary button bar. Divider: two flex lines + short “or” bar. Secondary button bar. Two small link bars (e.g. `h-4`). Use `animate-pulse` and `bg-gray-200` / `dark:bg-gray-700` (or existing home loading pattern). |
| Info box   | `mt-10 max-w-3xl rounded-2xl border-2 border-[#5AA1D9] ... p-6`                              | Same card; paragraph placeholder (2–3 lines); small pill placeholder for language toggle.                                                                                                                                                                                                                                |


Use the same Tailwind classes for the cards and section as in [LoginPageLayout](spots-app/app/_components/auth/LoginPageLayout.tsx) (lines 48–65) so spacing and width match. For skeleton bars, follow the pattern in [app/(pages)/home/loading.tsx](spots-app/app/(pages)/home/loading.tsx) (`animate-pulse`, `rounded-lg bg-gray-200`); add `dark:bg-gray-700` (or similar) where the real UI is dark-mode aware so the skeleton looks good in both themes.

### 3. No other changes

- Do **not** change [app/page.tsx](spots-app/app/page.tsx).
- Do **not** add PPR, metadata, Server Actions, or useActionState.
- Do **not** add or change `next.config.js`.

## File summary


| File                                                  | Action                                                                                          |
| ----------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| **New: [app/loading.tsx](spots-app/app/loading.tsx)** | Create; default export = loading UI for `/`, named export `LoginSkeleton` matching form/layout. |


## Implementation order

1. Implement `LoginSkeleton` with the structure above (layout + card + form placeholders + info box).
2. In `app/loading.tsx`, default-export the loading view (e.g. the skeleton or a thin wrapper) and named-export `LoginSkeleton`.

