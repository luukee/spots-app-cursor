---
name: Avatar name below icon
overview: Remove the desktop avatar Tooltip in [Header.tsx](spots-app/app/_components/layout/Header.tsx) and show the reporting/active display name as persistent text under the circular avatar, with minor layout tweaks so the header does not fight a fixed 96px row height.
todos:
  - id: remove-avatar-tooltip
    content: In Header.tsx, remove Tooltip wrapper around desktop avatar button; add flex-col stack with label using reportingForDisplayName (or tooltip-equivalent string).
    status: completed
  - id: layout-a11y
    content: Adjust lg header row height (min-h / h-auto / padding) so avatar+label fit; strengthen button aria-label for screen readers.
    status: completed
isProject: false
---

# Show reporting name under desktop header avatar

## Current behavior

In `[app/_components/layout/Header.tsx](spots-app/app/_components/layout/Header.tsx)` (approx. lines 474–496), the desktop avatar (`hidden lg:block`) is wrapped in `[Tooltip](spots-app/app/_components/ui/Tooltip.tsx)` with:

```ts
content={getCurrentDisplayName() || getLocSetting("User")}
```

`Tooltip` renders a `relative inline-block` wrapper around the trigger (matching the DOM path you saw).

## Target behavior

- **Desktop only** (`header_avatar_icon`): drop the `Tooltip` wrapper; keep the same `<button>` (avatar, menu toggle).
- **Below the button**: render a single line of text, centered under the 96px circle, using the same information users care about as “reporting for.”

**Label string (recommended):** use existing `[reportingForDisplayName](spots-app/app/_components/layout/Header.tsx)` (`getCurrentDisplayName() || loggedInAsDisplayName`), already used in the dropdown (“Reporting for: …”). That stays consistent with the menu and is rarely empty for signed-in users.

**Alternative:** if you must preserve the old tooltip’s empty-state exactly (`getCurrentDisplayName() || getLocSetting("User")`), use that expression instead — it differs from `reportingForDisplayName` only when a parent has no selected child (`getCurrentDisplayName()` is `''`).

## Layout / styling

- Wrap avatar + label in a **column**: e.g. `flex flex-col items-center gap-1` on the existing `header_avatar_icon` container (keep `ref={userMenuRef}` on that outer div so the dropdown still anchors correctly).
- **Typography:** small, centered, e.g. `text-xs` or `text-sm`, `text-center`, `max-w-24` (match avatar width), `truncate` or `line-clamp-2` if long names are common (pick one; `truncate` is simpler).
- **Header height:** the inner bar uses `[lg:h-24](spots-app/app/_components/layout/Header.tsx)` while the avatar alone is already 96px tall. Adding a label will exceed one row unless you relax height:
  - Prefer `**lg:min-h-24 lg:h-auto`** (or similar) on that flex row **or** modest vertical padding so avatar + label fit without clipping.
  - Header already has `overflow-visible` on the outer `<header>`; the goal is to avoid awkward vertical centering or overflow clipping on the inner flex container.

## Accessibility

- Improve `aria-label` on the avatar button: today it is only `getLocSetting("User")`. After removing the hover tooltip, expose context explicitly, e.g. combine menu action + who is active: open user menu / reporting for {name} (wording can follow your localization pattern).

## Files to touch


| File                                                                               | Change                                                                                           |
| ---------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| `[app/_components/layout/Header.tsx](spots-app/app/_components/layout/Header.tsx)` | Remove `Tooltip` around avatar; add label `div`/`span`; optional `aria-label` + row height tweak |


No change required to `[Tooltip.tsx](spots-app/app/_components/ui/Tooltip.tsx)` unless you want to remove an unused import after the edit (nav links and settings still use `Tooltip`).

## Out of scope

- **Mobile:** avatar block is `hidden lg:block`; mobile menu already shows reporting text in the sheet. No change unless you want parity in the compact header (not requested).

## Quick verification

- Parent with multiple children: name updates when switching child; dropdown still opens; `top-full` menu still positions under the avatar column.
- Child user: name shows under avatar.
- Light/dark header text classes: label inherits header `text-white` / `text-black` like other header copy.

