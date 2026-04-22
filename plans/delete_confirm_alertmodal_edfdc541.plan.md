---
name: Delete confirm AlertModal
overview: Extend the existing `AlertModal` with an optional two-button confirmation mode (destructive primary + cancel) while preserving current single-button behavior for `SymptomQuestionnaire`, then gate `SymptomsSummary` deletes behind that dialog with copy aligned to your mockup and REDCap-backed strings where possible.
todos:
  - id: extend-alert-modal
    content: Add optional onConfirm + labels + confirmVariant to AlertModal; keep single-button default; match mockup button order and colors
    status: completed
  - id: symptoms-summary-flow
    content: Add pending-delete state; open AlertModal before PATCH; localized title/message with EN fallbacks
    status: completed
  - id: manual-qa
    content: Verify keep/overlay vs delete, z-index over questionnaire, child delete body unchanged
    status: completed
isProject: false
---

# Delete confirmation via shared `AlertModal`

## Current behavior

- [`app/_components/common/AlertModal.tsx`](/Users/lukey/Sites/UT/SPOTS/spots-app/app/_components/common/AlertModal.tsx): portal + `ModalOverlay`, centered card, single **`Close`** (localized via `getLocSetting('Close')`), optional `title` / `message`, `zIndex` for stacking.
- [`app/_components/symptoms/SymptomQuestionnaire.tsx`](/Users/lukey/Sites/UT/SPOTS/spots-app/app/_components/symptoms/SymptomQuestionnaire.tsx): one `showAlert` / `alertError` state; `AlertModal` shows either default strings (extreme-option path) or `Unable to save` + error message. **No change required** to those flows if we keep default props backward-compatible.
- [`app/_components/symptoms/SymptomsSummary.tsx`](/Users/lukey/Sites/UT/SPOTS/spots-app/app/_components/symptoms/SymptomsSummary.tsx): trash icon calls `handleDeleteSymptom(symptom)` immediately ([`onClick` ~657–661](/Users/lukey/Sites/UT/SPOTS/spots-app/app/_components/symptoms/SymptomsSummary.tsx)); no confirmation.

## Design: extend `AlertModal` (DRY)

Add **optional** confirmation props. When **not** passed, behavior stays identical to today (one centered primary button).

Suggested API (exact names can match your codebase style):

| Prop | Role |
|------|------|
| `onConfirm?: () => void` | If set, switches to **two-button** layout |
| `confirmLabel?: string` | Default: e.g. `getLocSetting('ProblemStation_Delete')` at call site, or a new key |
| `cancelLabel?: string` | e.g. `getLocSetting('...Keep')` with fallback `"Keep"` |
| `confirmVariant?: 'danger' \| 'primary'` | Maps to red vs blue styling; delete uses `'danger'` |

**Layout (match mockup):** one row, two buttons side by side — **Delete (red) left**, **Keep (blue) right** — reuse Tailwind classes already used on the existing blue button and add a red variant for the confirm button.

**Behavior:**

- Overlay click: same as **cancel** → call `onClose()` only (do not call `onConfirm`).
- **Keep**: `onClose()`.
- **Delete**: call `onConfirm()`, then `onClose()` (or only `onConfirm()` if it encapsulates close — pick one convention and use it in both call sites). Recommended: **parent** owns `open` state; `onConfirm` runs the action; `onClose` clears `open` — implement buttons so cancel/overlay always invoke `onClose`, and confirm invokes `onConfirm()` then `onClose()` to avoid leaving the dialog open.

**Accessibility:** keep `role="dialog"` and `aria-modal="true"`. Optionally add `aria-labelledby` / `aria-describedby` if you add ids to title and body (nice-to-have, not blocking).

**Z-index:** `SymptomsSummary` should pass a `zIndex` at least as high as stacked modals when the edit questionnaire is open (questionnaire alert uses `10002`). Using **`10003`** (or reuse `10002` if deletes are disabled while editing — today they are not) avoids the confirm dialog hiding under the questionnaire overlay.

## Wire `SymptomsSummary`

1. Add state, e.g. `symptomPendingDelete: SymptomItem | null` (or store `key` + resolve from `items`).
2. Trash button: **set pending** instead of calling `handleDeleteSymptom` directly (still `stopPropagation`).
3. Render `<AlertModal open={!!symptomPendingDelete} onClose={() => setSymptomPendingDelete(null)} ... confirm props />` with:
   - **Title:** new loc key with fallback `"Delete"` (e.g. `ProblemStation_DeleteConfirmTitle`).
   - **Message:** new loc key with fallback matching your SS: *"Are you sure you want to delete this problem from your list of Current Problems?"* (e.g. `ProblemStation_DeleteConfirmMessage`).
   - **Confirm:** `onConfirm={() => { const s = symptomPendingDelete; if (s) void handleDeleteSymptom(s); }}` then close via the shared `onClose` pattern above.

4. **REDCap / loc:** Strings ultimately come from [`useLocalization` / `/api/loc`](/Users/lukey/Sites/UT/SPOTS/spots-app/app/_lib/hooks/loc/useLocalizedStrings.ts). Until new `settings_id` rows exist, **`getLocSetting(key, fallback)`** English fallbacks preserve the mockup behavior.

## What stays unchanged

- `SymptomQuestionnaire` continues using `AlertModal` **without** `onConfirm` — no UX regression for extreme-option or save-error alerts.

## Files to touch

1. [`app/_components/common/AlertModal.tsx`](/Users/lukey/Sites/UT/SPOTS/spots-app/app/_components/common/AlertModal.tsx) — props + conditional two-button UI + doc comment (layman + technical per your doc rule when you implement).
2. [`app/_components/symptoms/SymptomsSummary.tsx`](/Users/lukey/Sites/UT/SPOTS/spots-app/app/_components/symptoms/SymptomsSummary.tsx) — pending-delete state, modal, redirect trash click.

Optional follow-up (out of scope unless you want it in the same PR): add the new `settings_id` values in REDCap so Spanish and copy are fully CMS-driven.

## Testing (manual)

- Problem Station: click delete → dialog matches copy and colors → **Keep** / overlay → no API call, symptom still listed.
- **Delete** → dialog closes, row shows existing spinner, then item removed / events fire as today.
- Open edit questionnaire from Problem Station, trigger an error/save alert → still single-button `AlertModal`.
- Child user with `family_id`: delete still sends body as today after confirmation.
