---
name: DRY Problem Station rows
overview: Extract the three pulse “symptom row” placeholders into one shared component (or constant + small helper) in the Problem Station skeleton module, then use it from both `ProblemStationSkeleton` and `SymptomsSummary` so row styling cannot drift.
todos:
  - id: extract-rows-component
    content: Add PROBLEM_STATION_SKELETON_ROW_CLASS + ProblemStationSymptomRowsSkeleton in ProblemStationSkeleton.tsx
    status: completed
  - id: wire-full-skeleton
    content: Use ProblemStationSymptomRowsSkeleton inside ProblemStationSkeleton p-5 body
    status: completed
  - id: wire-symptoms-summary
    content: Replace SymptomsSummary loading IIFE with shared component + ariaLabel
    status: completed
  - id: align-row-styles
    content: Unify h-14 vs h-15 and problem-station-skeleton-item; verify Tailwind + visuals
    status: completed
  - id: smoke-check
    content: "Quick manual check: loading state, home loading, ProtectedRoute shell, dark mode"
    status: completed
isProject: false
---

# DRY: Problem Station list row skeletons

## Current state

- `[app/_components/symptoms/ProblemStationSkeleton.tsx](spots-app/app/_components/symptoms/ProblemStationSkeleton.tsx)`: full-card skeleton with header pulses, optional `h-4 w-3/4` line, **three row pulses** (`h-15`, `problem-station-skeleton-item`, …), and a **footer** pulse block. The footer and header pulses are **not** present in the live sidebar loading state.
- `[app/_components/symptoms/SymptomsSummary.tsx](spots-app/app/_components/symptoms/SymptomsSummary.tsx)`: real header + `p-5` body; when `loading`, an IIFE renders **three row pulses** with `**h-14`** and no `problem-station-skeleton-item`. `aria-label="Loading problem station"` is important for accessibility.

There is intentional **layout** difference (full skeleton vs in-place loading); only the **three rows** should be shared.

## Drift to eliminate


| Aspect       | Skeleton file                   | SymptomsSummary             |
| ------------ | ------------------------------- | --------------------------- |
| Row height   | `h-15`                          | `h-14`                      |
| Extra class  | `problem-station-skeleton-item` | none                        |
| Wrapper a11y | none on row list                | `aria-label` on `space-y-3` |


After refactor, both call sites use **one** row class string (and optionally one row count).

## Recommended approach

1. **Add a small exported building block** in the same file as the full skeleton (keeps imports simple for consumers that already use `ProblemStationSkeleton`):
  - Name idea: `ProblemStationSymptomRowsSkeleton` (or `ProblemStationListRowsSkeleton`).
  - Props (minimal):
    - `ariaLabel?: string` — pass `"Loading problem station"` from `SymptomsSummary`; omit for full-card skeleton (decorative).
    - Optional `rowCount?: number` default `3` if you ever need flexibility; otherwise hard-code `3`.
  - Renders a single outer `div` with `className="space-y-3"` and the mapped row divs inside.
  - Use a **shared constant** for the row `className` (e.g. `PROBLEM_STATION_SKELETON_ROW_CLASS`) so Tailwind classes live in one place.
2. **Update `ProblemStationSkeleton`** to replace its inline `[1,2,3].map` block with `<ProblemStationSymptomRowsSkeleton />` (still below the `h-4 w-3/4` line inside `p-5`). Footer block stays as-is.
3. **Update `SymptomsSummary`** to replace the `loading && (() => { ... })()` block with:
  - `{loading && <ProblemStationSymptomRowsSkeleton ariaLabel="Loading problem station" />}`
  - Drops the unnecessary IIFE.
4. **Pick one row height** after a quick visual check against real symptom rows in the same file (~`h-14 `cards). Prefer **`h-14`** unless design explicitly needs` h-15`; if` h-15 `is not defined in your Tailwind theme, align to **`h-14`** or an arbitrary value like` h-[3.75rem] `documented in a one-line comment. Remove **`problem-station-skeleton-item`** unless you add matching styles in` [app/globals.css](spots-app/app/globals.css)` or confirm it is required for tests/CSS hooks.
5. **Verify** `home/loading.tsx` and `ProtectedRoute` unchanged behavior (they only import `ProblemStationSkeleton`; the full card should look the same aside from unified row classes).
6. **Manual QA**: toggle `PREVIEW_PROBLEM_STATION_SKELETON_ONLY` / home loading preview if you still use them; confirm dark mode and mobile Problem Station panel.

## Out of scope (optional later)

- DRYing the **header** pulses with the real `h2`/`p` header (different semantics: fake vs real text).
- DRYing **footer** skeleton (only exists on full `ProblemStationSkeleton`).

## Files touched

- `[app/_components/symptoms/ProblemStationSkeleton.tsx](spots-app/app/_components/symptoms/ProblemStationSkeleton.tsx)` — new export + constant; wire into existing skeleton.
- `[app/_components/symptoms/SymptomsSummary.tsx](spots-app/app/_components/symptoms/SymptomsSummary.tsx)` — import shared rows component; simplify loading branch.

