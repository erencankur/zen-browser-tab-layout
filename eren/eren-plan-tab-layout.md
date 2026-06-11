# Native Horizontal Top Tabs Plan for Zen Browser

## 1. Project Context

This fork is on branch `feature/native-horizontal-tabs` and tracks Zen Browser desktop, with `origin` pointing to the fork and `upstream` pointing to `https://github.com/zen-browser/desktop`.

Zen Browser is currently vertical-tab and workspace oriented. The normal Firefox tabbrowser surface is heavily adapted by Zen so tabs are organized inside workspace-owned vertical containers rather than only inside the standard top tab strip.

CSS-only or `userChrome.css` approaches are not stable enough for this goal because the layout is not just visual. Zen repositions the toolbar, bookmarks toolbar, tab strip, workspace containers, pinned tabs, essentials, overflow behavior, drag-and-drop, and window controls at runtime.

A source-level native solution is required so horizontal tabs can participate in real browser behavior: tab selection, close buttons, favicons, loading state, overflow, pinned tabs, workspace assignment, session restore, customization, bookmarks toolbar visibility, and platform titlebar constraints.

## 2. Target Behavior

Horizontal mode should produce a Brave, Waterfox, or classic Firefox-like browser chrome:

- Browser tabs appear at the top of the window.
- Tabs are arranged side by side horizontally.
- Each tab uses native tabbrowser data and states where possible:
  - favicon
  - page title
  - selected/active styling
  - hover styling
  - close button
  - loading/throbber/busy state
- The new tab button appears immediately after the visible tab list.
- The navigation toolbar appears below the horizontal tab strip.
- The bookmarks toolbar is visible by default below the navigation toolbar.
- The current Zen vertical sidebar tab layout is hidden, disabled, or bypassed when horizontal mode is active.
- Workspace labels such as a large floating `Default` or `Space` indicator must not appear inside the top tab strip.
- No part of the solution should rely on `userChrome.css`.
- Existing Zen functionality should be preserved where it is compatible with horizontal tabs.

## 3. Repository Areas to Inspect

| Area | Why It Matters |
| --- | --- |
| `src/` | Root for Zen's source patch system and browser chrome modifications. |
| `src/zen/` | Zen-owned modules, styles, tab behavior, workspace behavior, startup code, and UI managers live here. |
| `src/zen/tabs/` | Contains `zen-tabs.css`, vertical tab CSS, pinned tab manager code, essentials UI, and tab-specific packaging. This is the main style and behavior area for Zen tab UI. |
| `src/zen/workspaces/` | Requested inspection path. This directory does not exist in the current tree. Workspace code appears to live under `src/zen/spaces/`. |
| `src/zen/spaces/` | Contains workspace implementation: `ZenSpace.mjs`, `ZenSpaceManager.mjs`, workspace icons, workspace CSS, session/workspace tab organization, and workspace-specific bookmarks. |
| `src/zen/common/` | Contains startup and UI coordination modules such as `ZenStartup.mjs`, `ZenUIManager.mjs`, `ZenCustomizableUI.sys.mjs`, shared styles, globals, and browser UI helpers. |
| `src/browser/` | Contains Firefox browser chrome patch files. Important areas include `browser/base/content`, `browser/components/tabbrowser`, `browser/components/customizableui`, and `browser/themes/shared/tabbrowser`. |
| `prefs/` | Default preference YAML files. Relevant files include `prefs/zen/zen.yaml`, `prefs/zen/view.yaml`, `prefs/zen/workspaces.yaml`, and `prefs/firefox/browser.yaml`. |
| `configs/` | Branding/configuration area. No obvious horizontal-tab logic was found here, but it should be kept in the inspection list for build or product config impacts. |
| `package.json` | Defines build and run commands. Relevant scripts include `build`, `build:ui`, `init`, `start`, `lint`, and `test`. |

## 4. Existing Architecture Notes

### Tab Rendering

Firefox's native tabbrowser remains the base. Zen patches:

- `src/browser/base/content/navigator-toolbox-inc-xhtml.patch`
- `src/browser/components/tabbrowser/content/tab-js.patch`
- `src/browser/components/tabbrowser/content/tabs-js.patch`
- `src/browser/components/tabbrowser/content/tabbrowser-js.patch`
- `src/browser/themes/shared/tabbrowser/tabs-css.patch`

The tab markup still includes the native structures needed for horizontal tabs: `.tab-icon-stack`, `.tab-label-container`, `.tab-label`, `.tab-close-button`, throbber/busy attributes, selected/visuallyselected attributes, and overflow support.

Assumption: the safest path is to reuse Firefox's existing horizontal tabbrowser rendering and Zen's patched tab elements, instead of creating a separate custom tab component.

### Vertical Sidebar Layout

Zen's layout is introduced through Firefox chrome patches and runtime DOM movement:

- `browser-xhtml.patch` wraps the browser in `#zen-main-app-wrapper`.
- `browser-box-inc-xhtml.patch` moves `navigator-toolbox.inc.xhtml` into the main browser hbox, creates `#zen-appcontent-wrapper`, `#zen-appcontent-navbar-wrapper`, `#zen-appcontent-navbar-container`, and `#zen-tabbox-wrapper`.
- `navigator-toolbox-inc-xhtml.patch` adds `#titlebar`, `#zen-toolbar-background`, `#zen-essentials`, and `#zen-tabs-wrapper` around the tabbrowser tabs.
- `ZenStartup.mjs` moves `#nav-bar` and `#PersonalToolbar` into `#zen-appcontent-navbar-container`.
- `ZenUIManager.mjs` defines `gZenVerticalTabsManager`, toggles `tabbrowser-tabs` between `vertical` and `horizontal` orientation, moves top buttons, nav bar, window controls, and rebuilds toolbar areas.
- `ZenCustomizableUI.sys.mjs` creates `#zen-sidebar-top-buttons`, `#zen-sidebar-foot-buttons`, and the sidebar splitter.
- `zen-tabs.css` currently imports `zen-tabs/vertical-tabs.css` unconditionally.
- `vertical-tabs.css` forces column layout on `#tabbrowser-tabs`, `#TabsToolbar`, `#titlebar`, and `#TabsToolbar-customization-target`.

### Workspace UI

Workspace UI appears to be under `src/zen/spaces/`, not `src/zen/workspaces/`.

Important files:

- `src/zen/spaces/ZenSpace.mjs`
- `src/zen/spaces/ZenSpaceManager.mjs`
- `src/zen/spaces/zen-workspaces.css`
- `src/zen/spaces/ZenSpaceIcons.mjs`
- `src/zen/spaces/ZenSpaceBookmarksStorage.js`

`ZenSpace.mjs` defines a `zen-workspace` custom element. Its markup includes:

- `.zen-current-workspace-indicator`
- `.zen-current-workspace-indicator-name`
- a vertical `arrowscrollbox`
- `.zen-workspace-pinned-tabs-section`
- `.zen-workspace-normal-tabs-section`
- `#tabs-newtab-button`
- `.zen-workspace-empty-space`

This is likely where large workspace labels such as `Default` or `Space` are introduced.

`ZenSpaceManager.mjs` rehomes tabs from the native tabbrowser strip into per-workspace sections via `#initializeTabsStripSections()` and `#createWorkspaceTabsSection()`. It also exposes fallback getters for the original Firefox containers when the workspace strip is not initialized.

Assumption: horizontal mode should either avoid initializing the workspace tab strip or make `activeWorkspaceStrip`, `pinnedTabsContainer`, and related getters deliberately return the native Firefox containers.

### Browser Toolbar and Titlebar Layout

The toolbar/titlebar layout is controlled by:

- `src/browser/base/content/navigator-toolbox-inc-xhtml.patch`
- `src/browser/base/content/browser-box-inc-xhtml.patch`
- `src/browser/base/content/navigator-toolbox-js.patch`
- `src/zen/common/modules/ZenStartup.mjs`
- `src/zen/common/modules/ZenUIManager.mjs`
- `src/zen/common/sys/ZenCustomizableUI.sys.mjs`
- `src/zen/common/styles/zen-toolbar.css`
- `src/zen/common/styles/zen-single-components.css`
- `src/zen/common/styles/zen-browser-ui.css`

`gZenVerticalTabsManager._updateEvent()` is the main runtime layout switch. It already computes `isVerticalTabs` from `zen.tabs.vertical` and sets `orient="vertical"` or `orient="horizontal"` on `gBrowser.tabContainer` and its scrollbox. However, other Zen code and CSS still assume the vertical/workspace model.

### Preferences and Default Settings

Relevant preferences found:

- `prefs/zen/zen.yaml`
  - `zen.tabs.vertical` is currently `true`.
  - `zen.tabs.vertical.right-side` is currently `false`.
  - `zen.tabs.show-newtab-vertical` is currently `true`.
- `prefs/zen/view.yaml`
  - `zen.view.use-single-toolbar` is currently `true`.
  - `zen.view.sidebar-expanded` is currently `true`.
  - `zen.view.show-newtab-button-top` is currently `true`.
- `prefs/zen/workspaces.yaml`
  - `zen.workspaces.hide-default-container-indicator` is currently `true`.
  - workspace and essentials behavior is configured here.
- `prefs/firefox/browser.yaml`
  - `browser.toolbars.bookmarks.visibility` is currently `"never"`.
  - `sidebar.revamp` and `sidebar.verticalTabs` are false and locked, which suggests Zen's vertical tab UI is separate from Firefox's newer sidebar vertical tabs.

### Bookmarks Toolbar Visibility

`ZenStartup.mjs` moves `#PersonalToolbar` into `#zen-appcontent-navbar-container` together with `#nav-bar`.

`ZenUIManager.mjs` watches `#PersonalToolbar` visibility and sets/removes `zen-has-bookmarks` on the document root. The actual default visibility is most likely controlled by `browser.toolbars.bookmarks.visibility` in `prefs/firefox/browser.yaml`.

For this fork, the default should become `"always"`.

## 5. Proposed Implementation Strategy

Use a preference-gated source implementation.

Recommended primary preference:

```text
zen.tabs.layout
```

Allowed values:

```text
vertical
horizontal
```

Default for this fork:

```text
horizontal
```

Keep the existing vertical implementation intact and activate the new horizontal behavior only when `zen.tabs.layout == "horizontal"`.

Recommended compatibility approach:

- Add `zen.tabs.layout` as the clear primary pref.
- Keep `zen.tabs.vertical` temporarily as a compatibility pref for existing Zen code.
- Refactor layout checks so new code uses a single helper or getter, for example:

```js
const layout = Services.prefs.getStringPref("zen.tabs.layout", "vertical");
const isHorizontalTabs = layout === "horizontal";
const isVerticalTabs = layout !== "horizontal";
```

- In the first implementation phase, derive vertical behavior from `zen.tabs.layout`.
- Avoid adding `zen.tabs.horizontal.enabled` unless there is a short-term migration need. It duplicates the string pref and creates more states to test.
- For this fork, consider setting `zen.tabs.vertical` to `false` only after all remaining code paths no longer assume it as the source of truth.

Implementation principle:

Reuse Firefox/Zen's native `tabbrowser-tabs` horizontal mode. Do not hand-build a separate tab model.

## 6. UI/Layout Plan

### Top Tab Container

Horizontal mode should use the existing `#TabsToolbar` and `#tabbrowser-tabs` surface as the top tab strip. The DOM already contains:

- `#TabsToolbar`
- `#tabbrowser-tabs`
- `#tabbrowser-arrowscrollbox`
- a native `tab is="tabbrowser-tab"`
- `#tabs-newtab-button`
- `#new-tab-button`

The layout should place the tab strip above the navigation toolbar:

1. Window titlebar/top tab row
2. Navigation toolbar
3. Bookmarks toolbar
4. Browser content

### Individual Tab Item Structure

Use existing native tab markup from Firefox with Zen's patches:

- `.tab-icon-stack` for favicon, throbber, pending icon, audio overlay, busy/loading state
- `.tab-label-container` and `.tab-label` for title
- `.tab-close-button` for close behavior
- `.tab-background` and selected attributes for active styling
- existing `busy`, `progress`, `pending`, `visuallyselected`, `selected`, `attention`, and hover states

Only add horizontal-specific styling where vertical CSS currently overrides or hides native horizontal behavior.

### Favicon

Use the existing `.tab-icon-image` inherited from Firefox tabbrowser. Do not introduce a custom favicon loader.

### Title Text

Use `.tab-label`. Ensure text ellipsizes in the tab width and remains visible for normal unpinned tabs.

### Close Button

Use `.tab-close-button`. Horizontal mode should restore classic Firefox behavior:

- visible on selected tabs and hover where Firefox normally shows it
- hidden or compact for pinned/essential tabs if those modes remain supported
- click behavior remains handled by `tab.js`

### Active State

Use `selected` and `visuallyselected` attributes. Add a horizontal stylesheet for Zen visual polish rather than changing tab selection logic.

### Hover State

Use CSS on `.tabbrowser-tab:hover .tab-background` and existing Firefox variables. Keep hover low-noise and consistent with Zen's theme tokens.

### Loading State

Use native `.tab-throbber` and `busy/progress` attributes. Verify with pages that load slowly.

### Tab Overflow Behavior

Prefer the native `arrowscrollbox` overflow model:

- `#tabbrowser-tabs[overflow]`
- `#tabbrowser-arrowscrollbox[overflowing]`
- scroll buttons or scrollbox behavior from Firefox
- `ensureElementIsVisible` on tab select

Zen currently overrides overflow through `gZenWorkspaces.updateOverflowingTabs()` and workspace sections. Horizontal mode should bypass or guard those workspace-specific overflow calls.

### New Tab Button

The new tab button should appear after the tab list.

Preferred approach:

- Use the standard horizontal `#tabs-newtab-button` inside the tab strip if it behaves correctly after workspace bypass.
- Otherwise use `#new-tab-button` as Firefox normally does for horizontal tabstrips.
- Avoid the vertical-only `#vertical-tabs-newtab-button` in horizontal mode.

### Relationship Between Tab Bar, Navigation Bar, and Bookmarks Toolbar

Horizontal mode should not use Zen's sidebar/top-button geometry as the primary chrome layout.

Target order:

```text
#TabsToolbar
#nav-bar
#PersonalToolbar
#tabbrowser-tabbox / content
```

`#PersonalToolbar` should remain visible by default and should not be collapsed by horizontal layout CSS.

## 7. Vertical Sidebar Handling

When horizontal mode is active:

- Do not show the Zen vertical tab sidebar as the primary tab UI.
- Do not show workspace tab sections inside the top tab strip.
- Hide or bypass:
  - `#zen-tabs-wrapper` vertical/workspace scroll area
  - `#zen-essentials` if it visually occupies the tab strip
  - `.zen-current-workspace-indicator`
  - `#zen-sidebar-top-buttons` and `#zen-sidebar-foot-buttons` unless specific controls are intentionally relocated
  - `#zen-sidebar-splitter`
- Set `#tabbrowser-tabs` and `#tabbrowser-arrowscrollbox` to `orient="horizontal"`.
- Keep the actual browser content and session/tab model unchanged.

Avoid workspace labels in the tab strip by preventing `zen-workspace` indicator markup from being used as the horizontal tab container. Specifically, the `.zen-current-workspace-indicator-name` from `ZenSpace.mjs` should never be inserted into, or displayed within, the horizontal tab row.

Workspace preservation options:

- Safest V1: keep workspace data/session attributes, but disable workspace strip UI when horizontal mode is active.
- More advanced later: provide a compact workspace selector outside the tab strip, such as a toolbar button or menu, without affecting horizontal tabs.

## 8. Bookmarks Toolbar Plan

The bookmarks toolbar should be visible by default in this fork.

Likely source of default visibility:

```text
prefs/firefox/browser.yaml
```

Current value:

```yaml
- name: browser.toolbars.bookmarks.visibility
  value: "never"
```

Recommended fork default:

```yaml
- name: browser.toolbars.bookmarks.visibility
  value: "always"
```

Additional implementation checks:

- Verify `#PersonalToolbar` is not left `collapsed`.
- Verify `ZenStartup.mjs` still moves `#PersonalToolbar` into the intended container for horizontal mode or stops moving it if the standard Firefox toolbox order is restored.
- Keep `ZenUIManager._initBookmarkCollapseListener()` because it sets `zen-has-bookmarks`, which Zen styles use.
- For existing profiles, consider a one-time migration in `ZenUIMigration.sys.mjs` only if the fork should force bookmarks visible for users who already have `"never"` stored.

## 9. Preference and Configuration Plan

Cleanest preference strategy:

1. Add `zen.tabs.layout` as a string pref in `prefs/zen/zen.yaml`.
2. Use values `"vertical"` and `"horizontal"`.
3. Default it to `"horizontal"` in this fork.
4. Keep `zen.tabs.vertical` during transition for compatibility, but stop adding new logic that depends on it directly.
5. Do not add `zen.tabs.horizontal.enabled` unless a temporary bridge is needed.
6. Set `browser.toolbars.bookmarks.visibility` to `"always"` for fresh profiles.

Recommended future pref state:

```yaml
- name: zen.tabs.layout
  value: "horizontal"

- name: zen.tabs.vertical
  value: false
```

The string pref is preferable because it can grow later without boolean conflicts, for example:

- `vertical`
- `horizontal`
- possible future `compact-horizontal`

## 10. Files Likely to Change

| Area | File or Directory | Expected Change | Risk Level |
| ---- | ----------------- | --------------- | ---------- |
| Preferences | `prefs/zen/zen.yaml` | Add `zen.tabs.layout`; eventually align `zen.tabs.vertical` default for this fork. | Medium |
| Preferences | `prefs/firefox/browser.yaml` | Change `browser.toolbars.bookmarks.visibility` from `"never"` to `"always"`. | Low |
| Startup layout | `src/zen/common/modules/ZenStartup.mjs` | Guard nav/bookmarks reparenting and vertical startup orientation when horizontal mode is active. | High |
| UI manager | `src/zen/common/modules/ZenUIManager.mjs` | Replace direct `zen.tabs.vertical` assumptions with layout getter; add horizontal mode branch in `gZenVerticalTabsManager._updateEvent()`. | High |
| Customizable UI | `src/zen/common/sys/ZenCustomizableUI.sys.mjs` | Avoid creating or showing vertical sidebar top/foot buttons as primary chrome in horizontal mode. | Medium |
| Browser XUL patch | `src/browser/base/content/navigator-toolbox-inc-xhtml.patch` | Ensure `#TabsToolbar`, titlebar items, and new tab button are ordered correctly for horizontal mode. | High |
| Browser XUL patch | `src/browser/base/content/browser-box-inc-xhtml.patch` | Potentially condition/harden Zen wrapper layout so the toolbox can behave as top chrome in horizontal mode. | High |
| Browser JS patch | `src/browser/base/content/navigator-toolbox-js.patch` | Review middle-click tab opening on `#zen-tabs-wrapper`; horizontal mode may not use that wrapper. | Low |
| Tab container behavior | `src/browser/components/tabbrowser/content/tabs-js.patch` | Guard workspace-specific `newTabButton`, `allTabs`, overflow, focusable items, and insert behavior for horizontal mode. | High |
| Tab behavior | `src/browser/components/tabbrowser/content/tab-js.patch` | Confirm close, favicon, title, loading, pinned, essential, and split view interactions still work horizontally. | Medium |
| Tabbrowser behavior | `src/browser/components/tabbrowser/content/tabbrowser-js.patch` | Guard vertical/workspace-specific tab movement, close animations, pinned tabs, and empty tab behavior. | High |
| Drag and drop | `src/browser/components/tabbrowser/content/drag-and-drop-js.patch` and `src/zen/drag-and-drop/ZenDragAndDrop.js` | Ensure horizontal tab drag/drop uses horizontal geometry and does not assume vertical workspace containers. | High |
| Tab CSS | `src/zen/tabs/zen-tabs.css` | Stop importing vertical CSS unconditionally; import or scope horizontal/vertical styles by pref/root attribute. | High |
| New tab CSS | `src/zen/tabs/zen-tabs/horizontal-tabs.css` | Add native horizontal tab strip styling for this fork. | Medium |
| Vertical CSS | `src/zen/tabs/zen-tabs/vertical-tabs.css` | Scope column/sidebar rules to vertical mode only. | High |
| Packaging | `src/zen/tabs/jar.inc.mn` | Register new horizontal stylesheet. | Low |
| Workspace element | `src/zen/spaces/ZenSpace.mjs` | Keep workspace indicator hidden or unused in horizontal mode; avoid vertical-only event assumptions. | Medium |
| Workspace manager | `src/zen/spaces/ZenSpaceManager.mjs` | Bypass workspace strip initialization or use native container fallbacks in horizontal mode. | High |
| Workspace CSS | `src/zen/spaces/zen-workspaces.css` | Scope workspace strip/indicator styles to vertical mode. | Medium |
| Shared CSS | `src/zen/common/styles/zen-single-components.css`, `zen-browser-ui.css`, `zen-toolbar.css`, `zen-browser-container.css` | Fix top toolbar, titlebar, bookmarks, and content spacing in horizontal mode. | Medium |
| Tests | `src/zen/tests/tabs/`, `src/zen/tests/spaces/`, `src/zen/tests/pinned/` | Add or update coverage for horizontal layout and ensure vertical behavior remains covered. | Medium |

## 11. Step-by-Step Implementation Plan

1. Inspect current tab/sidebar architecture.
   - Reconfirm the relationships among `ZenStartup.mjs`, `ZenUIManager.mjs`, `ZenSpaceManager.mjs`, `ZenSpace.mjs`, `tabs-js.patch`, and `zen-tabs.css`.

2. Identify tab rendering entry points.
   - Confirm how `#TabsToolbar`, `#tabbrowser-tabs`, `#tabbrowser-arrowscrollbox`, `#tabs-newtab-button`, and `#new-tab-button` are built after patch application.

3. Add preference.
   - Add `zen.tabs.layout` with `"vertical"` and `"horizontal"` values.
   - Default to `"horizontal"` for this fork.
   - Add a shared getter/helper so layout checks do not spread across the codebase.

4. Create or adapt horizontal tab container.
   - Prefer adapting the existing native `#TabsToolbar` and `#tabbrowser-tabs`.
   - Ensure the tab strip remains a horizontal `arrowscrollbox`.
   - Use native tab markup for favicon, title, close button, selected state, hover state, and loading state.

5. Conditionally disable vertical tab UI.
   - In horizontal mode, bypass workspace tab strip initialization or keep workspace DOM hidden.
   - Hide `#zen-tabs-wrapper`, workspace indicators, essentials grid, sidebar top/foot buttons, and sidebar splitter where appropriate.

6. Connect tab actions.
   - Verify tab selection, close, new tab, middle-click, keyboard navigation, overflow scrolling, pinned tabs, and drag/drop.
   - Guard Zen workspace-specific code in `tabs-js.patch`, `tabbrowser-js.patch`, and `ZenDragAndDrop.js`.

7. Ensure bookmarks toolbar visibility.
   - Set `browser.toolbars.bookmarks.visibility` to `"always"`.
   - Confirm `#PersonalToolbar` is in the desired visual order and remains uncollapsed in fresh profiles.

8. Adjust styles/layout.
   - Add `horizontal-tabs.css`.
   - Scope vertical rules so `vertical-tabs.css` does not force column layout in horizontal mode.
   - Fix nav bar, window controls, toolbar height, bookmarks toolbar, and content top spacing.

9. Test with multiple tabs.
   - Open enough tabs to trigger overflow.
   - Confirm title truncation, selected styling, close buttons, and scroll behavior.

10. Test pinned tabs if supported.
    - Pin, unpin, close, and restore pinned tabs.
    - Decide how Zen essentials interact with pinned tabs in horizontal mode.

11. Test workspace behavior.
    - Confirm workspace data is not corrupted.
    - Confirm no large workspace label appears in the tab strip.
    - Confirm workspace switching either remains available through an intentional control or is safely bypassed in horizontal V1.

12. Test build.
    - Run UI rebuild first, then full build if needed.
    - Run targeted browser tests where available.

13. Document changes.
    - Update a short fork note or implementation doc describing the pref, default behavior, known limitations, and vertical-mode preservation.

## 12. Testing Plan

Functional testing:

- One tab
  - Fresh window opens with one visible top horizontal tab.
  - Favicon/title area renders without workspace label pollution.
- Many tabs
  - Open 20 to 50 tabs.
  - Confirm horizontal layout, title truncation, overflow behavior, and no layout collapse.
- Active tab switching
  - Click tabs.
  - Use keyboard shortcuts.
  - Confirm selected and visually selected state follows the active browser.
- Closing tabs
  - Close selected tab.
  - Close background tab.
  - Close last normal tab.
  - Confirm close animation does not use vertical-only geometry incorrectly.
- Opening a new tab
  - Click the new tab button after the tab list.
  - Middle-click empty tab strip area if supported.
  - Use keyboard shortcut.
- Tab overflow
  - Confirm overflow indicators or scroll behavior work.
  - Confirm selecting a hidden/overflowed tab scrolls it into view.
- Bookmarks toolbar visibility
  - Test fresh profile.
  - Confirm `#PersonalToolbar` is visible by default.
  - Confirm bookmarks toolbar remains below nav bar.
- Workspace behavior
  - Test existing workspace session restore.
  - Confirm workspace IDs remain stable if still used.
  - Confirm no large `Default` or `Space` indicator appears in the top tab strip.
- Fresh profile behavior
  - Start with a clean profile.
  - Confirm horizontal layout is default.
  - Confirm bookmarks toolbar default is visible.
- Build/run verification
  - Run UI build and browser start.
  - Run full build before considering the feature complete.

Regression testing:

- Vertical mode still works when `zen.tabs.layout` is `"vertical"`.
- Private windows do not crash when workspace sync is disabled.
- macOS window controls remain usable.
- Compact mode either works or is intentionally disabled/guarded in horizontal mode.
- Split view, glance, essentials, folders, and pinned tab behavior are not silently broken.

## 13. Build and Run Commands

Initial setup:

```bash
npm i
npm run init
python3 ./scripts/update_en_US_packs.py
```

Build:

```bash
npm run build
```

Run:

```bash
npm start
```

Faster UI rebuild during browser chrome iteration:

```bash
npm run build:ui
```

Useful verification commands:

```bash
npm run lint
npm test
```

## 14. Risks and Unknowns

- Firefox upstream browser UI is complex, especially around `browser.xhtml`, `navigator-toolbox`, CustomizableUI, titlebar items, tabbrowser custom elements, and platform-specific window controls.
- Zen's workspace model assumes tabs live in workspace sections, so bypassing or reusing those sections horizontally may affect session restore, workspace switching, essentials, and pinned tabs.
- `zen-tabs.css` imports vertical tab CSS unconditionally, so horizontal mode needs careful CSS scoping rather than a small visual tweak.
- `ZenStartup.mjs` currently forces `#tabbrowser-arrowscrollbox` back to vertical after delayed startup. This must be guarded.
- `gZenVerticalTabsManager` already supports an `isVerticalTabs` computation, but many dependent systems still appear vertical-first.
- Pinned tabs and Zen essentials may have special containers and reset/unload behavior that do not map cleanly to classic horizontal tabs.
- Tab drag-and-drop uses Zen-specific logic and may need horizontal geometry fixes.
- The workspace indicator markup is part of `zen-workspace`; it must not leak into horizontal tab UI.
- macOS titlebar/window control placement may need separate handling.
- Existing profiles may retain old toolbar visibility prefs even after changing defaults.
- Build times can be high, so `npm run build:ui` should be used during iterative UI work, followed by full build verification.

## 15. Final Recommendation

The safest implementation path is to make `zen.tabs.layout` the primary source of truth, default it to `"horizontal"` in this fork, and reuse Firefox's native horizontal `#TabsToolbar`/`#tabbrowser-tabs` behavior wherever possible.

Do not build a custom tab strip. Instead, preserve Zen's patched tab elements and browser actions, then guard or bypass the workspace/vertical-sidebar systems when horizontal mode is active. Start by scoping CSS and startup/layout reparenting behind the new preference, then fix tab container APIs and workspace fallbacks until the native horizontal tab strip works end to end.
