# Sticky Tabs for Brave (Reference Implementation)

See `../sticky-tabs.md` for the design.

This directory contains **reference patches** showing the exact Chromium-tree
changes for the feature. They are generated as `git diff` against Brave's
post-patch Chromium 157 tree, so they do **not** apply cleanly via Brave's
`apply_patches` step — Brave already has its own patches touching some of the
same files (e.g. `pinned_tab_codec.cc` adds `BraveShouldSkipPinnedTab`).

## Integration into Brave's patch system

To ship this feature as native Brave patches, these diffs must be **merged
into Brave's existing `brave/patches/*.patch` files** for each affected
Chromium source file. New files (`sticky_tab_controller.{h,cc}`) can either:

1. Live in `brave/browser/ui/tabs/` (Brave tree) with hooks from Chromium
   files via `brave/chromium_src/`, or
2. Be added as new-file patches under `brave/patches/`.

## Files in this directory

### New Brave/Chromium source files

| File | Purpose |
|---|---|
| `chrome-browser-ui-tabs-sticky_tab_controller.h.patch` | Per-tab controller: sticky state, home URL, title. |
| `chrome-browser-ui-tabs-sticky_tab_controller.cc.patch` | Implementation: make-sticky, reset, placement, persistence, close hook, rename dialog. |

### Chromium file edits

Every hook is marked with `// STICKY_TABS:` for easy grep.

| File | What |
|---|---|
| `components-tabs-public-tab_context_menu_command.h.patch` | 4 new context-menu commands. |
| `chrome-browser-ui-tabs-BUILD.gn.patch` | Registers the new files. |
| `chrome-browser-ui-tabs-public-tab_features.h.patch` | Owner pointer. |
| `chrome-browser-ui-tabs-tab_features.cc.patch` | Instantiation. |
| `chrome-browser-ui-tabs-tab_data.h.patch` | `sticky_state` field in display data. |
| `chrome-browser-ui-tabs-tab_data.cc.patch` | Fixed title + sticky-state assignment. |
| `chrome-browser-ui-tabs-tab_strip_model.cc.patch` | Command dispatch, insertion constraint, close hook. |
| `chrome-browser-ui-tabs-tab_menu_model.cc.patch` | Menu items. |
| `chrome-browser-ui-tabs-pinned_tab_codec.cc.patch` | Sticky title in pinned_tabs pref. |
| `chrome-browser-ui-startup-startup_tab.h.patch` | `sticky_title` field on `StartupTab`. |
| `chrome-browser-ui-startup-startup_browser_creator_impl.cc.patch` | Restore title; session-fail fallback to pref. |
| `chrome-browser-ui-browser_commands.cc.patch` | Home command / Cmd+Shift+H hook. |
| `chrome-browser-sessions-session_service_base.cc.patch` | Persist extra_data on session rebuild. |
| `chrome-browser-ui-browser_tabrestore.cc.patch` | Restore extra_data on tab restore. |
| `chrome-browser-ui-browser_live_tab_context.cc.patch` | Capture extra_data for Cmd+Shift+T. |
| `chrome-browser-ui-views-tabs-tab-tab_icon.h.patch` | Dot color member. |
| `chrome-browser-ui-views-tabs-tab-tab_icon.cc.patch` | Green/yellow dot painting. |
| `chrome-browser-ui-views-tabs-tab.cc.patch` | Double-click rename (horizontal strip). |
| `chrome-browser-ui-views-tabs-common-tab_view.cc.patch` | Double-click rename (vertical strip). |
