---
name: Report page chart localization
overview: Add a DRY helper in symptom-term-localization for hydrate + label mapping; use it from useSummaryChart and report page summary. Localize report loadSymptoms labels the same way; keep one catalog fetch for display order where needed.
todos:
  - id: add-localize-helper
    content: Add localizeSymptomCountRows (async) in symptom-term-localization.ts; refactor useSummaryChart to use it
    status: completed
  - id: report-summary-localize
    content: Report loadSummary — use helper + validatedLanguage + effect deps (remove getAllSymptoms term map)
    status: completed
  - id: report-symptoms-dropdown
    content: Report loadSymptoms — use helper for terms; getAllSymptoms(catalogLanguage) only for displayOrder map + language in deps
    status: completed
  - id: verify-report-es
    content: "Manual: /report in ES — chart + filter list labels match catalog"
    status: completed
isProject: false
---

# Localize report page summary chart (DRY)

## Cause

[`app/(pages)/report/page.tsx`](spots-app/app/(pages)/report/page.tsx) does **not** use the shared localization pipeline. The `loadSummary` effect (~67–141) calls **`getAllSymptoms('en')`** and maps English terms only. Axis titles in [`SummaryChart.tsx`](spots-app/app/_components/reports/SummaryChart.tsx) use `getLocSetting`, hence mixed ES/EN.

Effect deps omit **`language`**, so toggling EN/ES does not refresh summary data.

[`useSummaryChart`](spots-app/app/_lib/hooks/reports/useSummaryChart.ts) already does the right steps inline (unique IDs → `hydrateSymptomTerms` → `getLocalizedSymptomDisplayName` per row) — same logic should not be duplicated on the report page.

## DRY approach

### 1. Shared helper (single place for “counts rows → localized labels”)

In [`symptom-term-localization.ts`](spots-app/app/_lib/services/symptoms/symptom-term-localization.ts), add an **async** function, e.g. **`localizeSymptomCountRows`**:

- **Input:** `rows: readonly T[]` where `T extends { symptom_id: string; symptom_term: string }`, plus `language: SymptomCatalogLanguage`.
- **Behavior:** collect unique `symptom_id`s → `await hydrateSymptomTerms(ids, language)` → return **`rows.map(row => ({ ...row, symptom_term: getLocalizedSymptomDisplayName(...) }))`** (new objects so React state updates reliably).
- **Docstring:** note it is for `SymptomCount`-shaped rows and any minimal `{ symptom_id, symptom_term }` list (e.g. report filter options).

### 2. Refactor [`useSummaryChart.ts`](spots-app/app/_lib/hooks/reports/useSummaryChart.ts)

Replace the inline block (uniqueIds / `hydrateSymptomTerms` / `forEach` on `counts`) with **`await localizeSymptomCountRows(counts, language)`** and return that result. No behavior change; removes duplication before the report page uses the same API.

### 3. Report page — `loadSummary`

- Destructure **`language`** from `useLocalization()`; **`validatedLanguage`** same pattern as [`home/page.tsx`](spots-app/app/(pages)/home/page.tsx).
- After a successful `/api/reports/summary` response, build the raw row array, then **`setSummaryData(await localizeSymptomCountRows(rawRows, validatedLanguage))`**.
- Remove **`symptomService.getAllSymptoms('en')`** and the English **`symptomTermMap`** for this path.
- Add **`validatedLanguage`** to the effect dependency array.

### 4. Report page — `loadSymptoms` (dropdown)

- After fetching counts, run **`localizeSymptomCountRows`** on the mapped list so **`symptom_term`** matches UI language.
- **Display order:** keep a **single** **`symptomService.getAllSymptoms(validatedLanguage)`** pass to build **`symptomDisplayOrderMap` only** (drop the old parallel English term map; terms come from the helper).
- Add **`validatedLanguage`** to effect deps.

**Note:** First visit may call the catalog twice (helper’s hydrate + `getAllSymptoms` for order) if the service layer does not dedupe; acceptable for this change. Optional later: extend shared hydration to cache `displayOrder` and read it in the report sort to avoid a second fetch.

### 5. Optional

Call **`clearLanguageCache(validatedLanguage)`** when language changes before reloading summary, matching [`usePastProblems`](spots-app/app/_lib/hooks/reports/usePastProblems.ts) / `useSummaryChart` — only if you want strict parity (not required for correct labels when deps include language).

## Files to touch

| File | Change |
|------|--------|
| [`symptom-term-localization.ts`](spots-app/app/_lib/services/symptoms/symptom-term-localization.ts) | Add `localizeSymptomCountRows` |
| [`useSummaryChart.ts`](spots-app/app/_lib/hooks/reports/useSummaryChart.ts) | Use helper instead of inline hydrate + loop |
| [`report/page.tsx`](spots-app/app/(pages)/report/page.tsx) | `validatedLanguage`, `loadSummary` + `loadSymptoms` as above |

## Verification

- `/report` in Spanish: chart Y-axis + legend + filter symptom names match catalog (consistent with home).
- Toggle EN/ES: summary (and dropdown effect) refresh with correct labels.
- Home summary chart unchanged in behavior (still uses hook; hook now calls helper).
