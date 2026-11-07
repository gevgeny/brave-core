# Sticky Tabs

A sticky tab remembers a home URL and a custom title. Navigating away marks the tab as changed; one action returns to the home URL.

Reference designs: Zen (`src/zen/tabs/ZenPinnedTabManager.mjs`), Arc, Dia.

Goal: smallest possible diff against Chromium; reuse existing UI.

## Model

Sticky state is **derived from tab-group membership and pinning**:

| Trigger | Result |
|---|---|
| Pin a tab | Auto-sticky (pinned tabs cannot be in groups in Chromium) |
| Unpin a tab | Auto-unstick (unless in a group) |
| Drag tab into group / "Add to new group" / "Add to existing group" | Auto-sticky |
| Drag tab out of group / "Remove from group" | Auto-unstick |

No "Make tab sticky" menu item. Groups + pins are the only ways to sticky a tab. Ungrouped non-pinned tabs cannot be sticky.

## UI

| Element | Implementation |
|---|---|
| Title | Sticky title overrides the page title via `TabData` (`tab_data.cc`); shows in horizontal/vertical strip, tooltip, hover card. |
| State dot | Favicon dot via `TabData::sticky_state` → `TabIcon`: green (`kColorAlertLowSeverity`) at home URL, yellow (`kColorAlertMediumSeverityIcon`) after leaving. Same for pinned and grouped sticky tabs. |
| Back home | Chrome Home command (toolbar ⌂, Cmd+Shift+H, History → Home) resets the active sticky tab instead of opening the homepage; hook in `chrome::Home()`. Also menu "Back to sticky URL". |
| Rename | Double-click a sticky tab's title, or menu "Rename…" → centered modal (`chrome::ShowBrowserModal`). Enter saves, Esc cancels. |
| Context menu (sticky tab only) | "Back to sticky URL" (only when changed), "Set current page as sticky URL", "Rename…". |
| Close hook | X button / Cmd+W / context menu close — yellow sticky → redirect to home; green sticky → confirm dialog. Guarded by `CLOSE_USER_GESTURE` so quit, session cleanup, extension API bypass. |

Skipped: inline title editing, in-tab reset button / favicon click (layout churn → upstream conflicts), edit-URL dialog, tab-anchored rename popup.

## Decisions

**Built into Chromium, not an extension.** `chrome.tabs` IDs change on every restart, so an extension can only match stored tabs by guessing (URL, position).

**Storage: session `extra_data` + existing `pinned_tabs` pref.** Only pinned sticky tabs survive when the session isn't restored. Bookmarks rejected (visible, user-editable, async load).

| Store | Contents | Role |
|---|---|---|
| Session `extra_data` (`sticky.home_url`, `sticky.title`) | home URL, title | Travels with the tab: normal restore and Cmd+Shift+T. Written on change (`AddTabExtraData`) and on session rebuild (`BuildCommandsForTab`, which otherwise drops tab extra data). |
| Pref `pinned_tabs` (`PinnedTabCodec`) | `url` = home URL, `sticky_title` | Recovery of pinned sticky tabs when the session isn't restored. Lifecycle handled by existing `PinnedTabService`. |

Non-pinned (grouped) sticky tabs are session-only. Groups are not persisted in `pinned_tabs`.

**Startup:**
- Session restored → `browser_tabrestore.cc` parks state on the WebContents; the tab's `StickyTabController` picks it up.
- Session not restored ("Open the New Tab page") → pinned sticky tabs open from `pinned_tabs` at their home URL with their title.
- Session restore attempted but returned nothing (missing/corrupt session with "Continue" on) → `RestoreOrCreateBrowser` falls back to the `pinned_tabs` backup so pinned sticky tabs are not lost.
- Pinned tab without saved state (legacy, pre-feature pin) → becomes sticky on first load, title follows page until renamed.

**Placement: constrain inserts only.** New tabs that would land before or inside the sticky run open right after it: `TabStripModel::ConstrainInsertionIndex` lower bound becomes `IndexOfFirstNonStickyTab()` (first non-pinned, non-sticky index; groups with a sticky tab are skipped as a unit). `ConstrainMoveIndex` unchanged — dragging is free.

**Redraw:** sticky state changes trigger `WebContents::NotifyNavigationStateChanged(INVALIDATE_TYPE_TITLE)` — same path as page title changes; refreshes horizontal/vertical tabs, tooltip, hover card.

**Leave detection:** committed URL differs from home in any part, `#fragment` included (`GURL` equality in `DidFinishNavigation`, which also fires for same-document / `pushState` changes). Sites that rewrite URL on load: user picks "Set current page as sticky URL".

No backups: Preferences writes are atomic; losing the file means losing the whole profile.
