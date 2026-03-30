---
name: zotero-plugin-template
description: Use this skill when implementing, extending, or reviewing a Zotero plugin built from this zotero-plugin-template repository. It captures the template's architecture, Zotero integration points, and coding conventions.
---

# Zotero Plugin Template

## When to Use

Load this skill when:

- Adding or modifying plugin behavior in a repo based on this template.
- Mapping a feature request to the correct bootstrap hook, module, or static addon file.
- Reviewing whether a change follows this template's Zotero API usage and coding style.
- Working in an environment where the agent cannot directly inspect the original template repo and needs the architecture, API usage, and command rules in one place.

## Version Compatibility

Use this compatibility guidance:

- The checked-in template manifest uses `applications.zotero.strict_min_version = "6.999"` and `strict_max_version = "8.*"`.
- Treat Zotero 7 and Zotero 8 compatibility as supported by this template pattern.
- Treat Zotero 9 compatibility as unconfirmed. The bootstrap-plugin architecture is likely to remain viable, but support should not be claimed until the plugin is tested against Zotero 9 and the manifest max version is updated accordingly.
- When shipping a plugin, set `strict_max_version` to the latest Zotero major/minor range actually tested.

Do not tell agents to assume Zotero 9 compatibility by default.

## Agent Command Safety

Follow these command rules strictly:

- Do not run `npm start` from agents. It maps to `zotero-plugin serve`, launches Zotero, watches files, and blocks.
- Do not run `npm run release` from agents unless a human explicitly requests a release flow and is prepared for interactive prompts.
- Use `npm run build` for non-interactive validation builds.
- Use `npm run lint:check` for non-interactive formatting and lint validation.
- Run tests as `npm run test -- --exit-on-finish`, not plain `npm run test`, so the test runner exits cleanly instead of staying in watch-style behavior.

Prefer these safe commands:

```bash
npm run build
npm run lint:check
npm run test -- --exit-on-finish
```

## Core Rule

Preserve the template's split between bootstrap wiring, lifecycle dispatch, feature modules, and packaged addon assets. Add feature logic in `src/modules/**` or `src/utils/**`, keep `src/hooks.ts` focused on coordination, and keep `addon/**` limited to required bootstrap, manifest, prefs, locale, CSS, and markup wiring.

## Architecture

Follow this runtime flow:

1. Zotero loads `addon/bootstrap.js`.
2. `startup()` registers chrome content and loads `content/scripts/__addonRef__.js` into the plugin sandbox.
3. The bundle comes from `src/index.ts`, which guards against duplicate initialization, creates `new Addon()` when needed, stores it at `Zotero[config.addonInstance]`, sets `_globalThis.addon`, and defines the `ztoolkit` getter.
4. `addon/bootstrap.js` then calls `Zotero.__addonInstance__.hooks.onStartup()`.
5. `src/hooks.ts` waits for Zotero readiness, initializes locale, registers prefs/notifiers/shortcuts/UI, and dispatches per-window setup through `onMainWindowLoad()`.
6. `onShutdown()` and `onMainWindowUnload()` clean up registered UI and dialog state.

Use these files as the architectural map:

- `addon/bootstrap.js`: Zotero bootstrap entry and sandbox loader.
- `src/index.ts`: sandbox-global registration and addon instance injection.
- `src/addon.ts`: long-lived plugin state container.
- `src/hooks.ts`: lifecycle dispatcher.
- `src/modules/*.ts`: actual feature logic.
- `src/utils/*.ts`: reusable helpers for locale, prefs, toolkit, and window liveness.
- `addon/content/*`, `addon/locale/*`, `addon/prefs.js`, `addon/manifest.json`: packaged addon resources and metadata.

For a deeper file map, see `references/repo-map.md`.

## Standalone Template Snapshot

Use this section when the template repo is not visible.

### Essential config keys

Expect `package.json` to define:

```json
{
  "config": {
    "addonName": "Plugin Name",
    "addonID": "plugin@example.com",
    "addonRef": "pluginref",
    "addonInstance": "PluginInstance",
    "prefsPrefix": "extensions.zotero.pluginref"
  }
}
```

Interpret them this way:

- `addonName`: display name.
- `addonID`: plugin ID used by manifest, pref panes, and release metadata.
- `addonRef`: chrome namespace, Fluent prefix base, DOM/pref naming base.
- `addonInstance`: runtime mount point at `Zotero[addonInstance]`.
- `prefsPrefix`: full pref namespace used for programmatic pref access.

### Expected lifecycle entrypoints

Expect these bootstrap and hook functions:

- `addon/bootstrap.js`: `install`, `startup`, `onMainWindowLoad`, `onMainWindowUnload`, `shutdown`, `uninstall`
- `src/hooks.ts`: `onStartup`, `onShutdown`, `onMainWindowLoad`, `onMainWindowUnload`, `onNotify`, `onPrefsEvent`, `onShortcuts`, `onDialogEvents`

### Required startup gating

Before registering most Zotero UI or plugin integrations, wait for:

- `Zotero.initializationPromise`
- `Zotero.unlockPromise`
- `Zotero.uiReadyPromise`

### Global/runtime model

The plugin runs in a bootstrap sandbox. The template pattern mounts the addon instance at `Zotero[config.addonInstance]`, stores `_globalThis.addon`, exposes `ztoolkit` through a getter, and relies on sandbox-visible globals such as `rootURI`.

### Build output model

Expect scaffold to compile `src/index.ts` into:

```text
.scaffold/build/addon/content/scripts/${addonRef}.js
```

Expect production packaging under `.scaffold/build/addon/**` plus generated `.xpi` and update manifest artifacts.

## Directory Responsibilities

Place code by responsibility:

- Put executable plugin code in `src/**`.
- Put feature implementations in `src/modules/**`.
- Put small reusable wrappers in `src/utils/**`.
- Put generated or ambient typings in `typings/**`.
- Put bootstrap-time and packaged resources in `addon/**`.
- Put preference markup in `addon/content/preferences.xhtml`.
- Put Fluent strings in `addon/locale/<locale>/*.ftl`.
- Put default pref values in `addon/prefs.js`.

Do not move feature logic into `bootstrap.js`, `manifest.json`, or `preferences.xhtml` beyond the minimal wiring already used by the template.

## Build and Packaging Model

Treat `package.json` and `zotero-plugin.config.ts` as the source of truth for plugin identity and build behavior.

- Read plugin identity from `package.json > config`:
  - `addonName`
  - `addonID`
  - `addonRef`
  - `addonInstance`
  - `prefsPrefix`
- Read build flow from `zotero-plugin.config.ts`:
  - source roots: `src`, `addon`
  - dist root: `.scaffold/build`
  - esbuild entry: `src/index.ts`
  - bundled output: `.scaffold/build/addon/content/scripts/${addonRef}.js`

Keep new code consistent with scaffold placeholder usage:

- `__addonRef__`
- `__addonInstance__`
- `__addonName__`
- `__buildVersion__`
- `__buildTime__`

Do not hardcode values that already come from `package.json` config.

## Zotero and Toolkit APIs Used by This Template

Prefer the same API surfaces already exercised by the template:

- `Zotero.Notifier.registerObserver()` for plugin event observation.
- `Zotero.MenuManager.registerMenu()` and `unregisterMenu()` for main-window and context-menu integrations.
- `Zotero.PreferencePanes.register()` for preferences UI.
- `Zotero.ItemTreeManager.registerColumns()` for item list columns.
- `Zotero.ItemPaneManager.registerInfoRow()` and `registerSection()` for item pane integrations.
- `Zotero.Search`, `Zotero.Items`, `Zotero.getMainWindows()`, and `Zotero.Promise.*` for Zotero-native operations.
- `zotero-plugin-toolkit` for higher-level utilities:
  - `ZoteroToolkit`
  - `ztoolkit.UI.createElement()`
  - `ztoolkit.Keyboard.register()`
  - `ztoolkit.ProgressWindow`
  - `ztoolkit.Dialog`
  - `ztoolkit.VirtualizedTable`
  - `ztoolkit.Prompt`
  - `ztoolkit.Clipboard`
  - `ztoolkit.FilePicker`

Prefer Zotero's built-in menu API for menus and toolkit wrappers for the remaining helper surfaces instead of ad hoc DOM manipulation when an existing API already exists.

### Menu API migration note

Do not use `ztoolkit.Menu.register()` in new work. In current toolkit releases such as `zotero-plugin-toolkit@5.1.2`, the exported toolkit helpers no longer include a `Menu` helper. Use `Zotero.MenuManager.registerMenu()` with:

- `menuID`
- `pluginID`
- `target`
- `menus`

Use `menuType` values like:

- `menuitem`
- `separator`
- `submenu`

Use Zotero menu targets such as:

- `main/library/item`
- `main/library/collection`
- `main/menubar/file`
- `main/tab`
- `reader/menubar/file`

For labels in shared Zotero windows, prefer `l10nID: getLocaleID(...)` over raw label strings so the menu can localize through Fluent.

For a repo-grounded API map, see `references/api-style.md`.

## Coding Style

Follow these repository conventions.

### 1. Keep hooks as coordination-first entry points

Treat `src/hooks.ts` as the lifecycle coordination layer. The current template still performs some direct startup UI work there, but new feature logic should usually live in modules and be called from hooks rather than expanding hooks into large business-logic files.

### 2. Centralize shared state in `Addon.data`

Store long-lived state on `addon.data`, as in `src/addon.ts`. Add fields there when the state is truly plugin-wide.

### 3. Use TypeScript strongly, with pragmatic Zotero interop at the edges

Prefer typed wrappers like `getPref<K extends keyof PluginPrefsMap>()`, typed hook signatures, and generated typings from `typings/**`. Limit unsafe runtime escapes to the same places the template already uses them, such as injected globals in `src/index.ts`.

Do not introduce broad untyped plumbing when a typed wrapper is practical.

### 4. Reuse config-driven naming

Build IDs, pref keys, chrome URLs, and localization IDs from `config.addonRef`, `config.addonID`, and `config.prefsPrefix`.

Examples already in the repo:

- `chrome://${addon.data.config.addonRef}/content/...`
- `#zotero-prefpane-${config.addonRef}-...`
- `${config.prefsPrefix}.${key}`
- `${config.addonRef}-${localeKey}`

### 5. Prefer async/await and explicit startup gating

Gate startup work on:

- `Zotero.initializationPromise`
- `Zotero.unlockPromise`
- `Zotero.uiReadyPromise`

Use `Promise.all()` when startup work is independent. Use `Zotero.Promise.delay()` or `defer()` only for Zotero-specific timing and rendering workflows already used by the template.

### 6. Use `ztoolkit.log`

Prefer `ztoolkit.log(...)` for plugin logs. The template configures console logging by environment in `src/utils/ztoolkit.ts`.

### 7. Clean up toolkit-managed UI on unload, and review non-toolkit registrations explicitly

Call `ztoolkit.unregisterAll()` and close transient windows/dialogs during shutdown or window unload. For Zotero registrations that are not obviously toolkit-managed, such as item-tree or item-pane integrations, verify separately whether explicit unregister logic is needed.

## Localization and Preferences

Use the template's existing localization helpers and its actual preference wiring pattern.

### Localization

- Put strings in `addon/locale/<locale>/*.ftl`.
- Load addon locale through `initLocale()` from `src/utils/locale.ts`.
- Resolve strings with `getString()`.
- Resolve Fluent IDs with `getLocaleID()`.
- Insert main-window Fluent files with `win.MozXULElement.insertFTLIfNeeded(...)` in window setup.

Keep locale keys unprefixed in source files; the helper adds the `${config.addonRef}-` prefix.

### Preferences

- Declare default values in `addon/prefs.js`.
- Use `preference="..."` bindings in `addon/content/preferences.xhtml` for pref-pane controls, matching the template's current implementation.
- Use `src/modules/preferenceScript.ts` for preference-window behavior and table/UI logic.
- Use `src/utils/prefs.ts` when feature code outside the pref pane needs pref access through the configured prefix.
- Update `typings/prefs.d.ts` expectations by keeping prefs compatible with scaffold-generated typing.
- Keep behavioral logic out of XHTML beyond the existing load hook.

### Preference pane structure

Model the preference pane as a four-part unit:

- `addon/content/preferences.xhtml`: static pane markup, Fluent hookup, and direct pref bindings.
- `src/modules/preferenceScript.ts`: runtime behavior after the pane loads.
- `addon/prefs.js`: default values for pref keys such as `enable` and `input`.
- `addon/locale/<locale>/preferences.ftl`: labels and help text used only by the preference pane.

Expect `addon/content/preferences.xhtml` to contain:

- a `<linkset>` with `<html:link rel="localization" href="__addonRef__-preferences.ftl" />`
- a root `groupbox` with `onload="Zotero.__addonInstance__.hooks.onPrefsEvent('load', { window })"`
- direct pref-bound controls using `preference="..."`
- concrete IDs derived from `__addonRef__`, such as `zotero-prefpane-__addonRef__-enable` and `zotero-prefpane-__addonRef__-input`
- a scripted container for richer UI, in this template: `<html:div id="__addonRef__-table-container" />`
- help text rendered with Fluent args using `__buildTime__`, `__addonName__`, and `__buildVersion__`

Expect `src/modules/preferenceScript.ts` to do the dynamic work:

- initialize or refresh `addon.data.prefs.window`
- define localized table columns with `getString("prefs-table-title")` and `getString("prefs-table-detail")`
- build a `ztoolkit.VirtualizedTable` into `${config.addonRef}-table-container`
- bind DOM events to the pref controls by querying IDs like `#zotero-prefpane-${config.addonRef}-enable`
- handle richer interactions such as selection callbacks, deletion keys, rerendering, and progress-window notifications

Use this split consistently:

- keep static layout, localization links, and simple pref bindings in `preferences.xhtml`
- keep event listeners, table logic, and imperative UI behavior in `preferenceScript.ts`

### Multi-language directory structure

Mirror the same file families under every locale directory:

```text
addon/locale/
  en-US/
    addon.ftl
    mainWindow.ftl
    preferences.ftl
  zh-CN/
    addon.ftl
    mainWindow.ftl
    preferences.ftl
```

Use the files by scope:

- `addon.ftl`: plugin-global strings such as startup text, menu labels, tab labels, and shared labels like table headers.
- `mainWindow.ftl`: strings injected into Zotero main windows, such as item-pane section headers, tooltips, and row labels.
- `preferences.ftl`: preference-pane-only strings such as pref titles, field labels, and help text with Fluent arguments.

When adding a new locale:

1. Create a sibling directory under `addon/locale/`.
2. Add the same file set: `addon.ftl`, `mainWindow.ftl`, `preferences.ftl`.
3. Keep message keys aligned across locales.
4. Keep source keys unprefixed; scaffold will prefix packaged IDs with `${config.addonRef}-`.

Do not mix preference-pane strings into `mainWindow.ftl`, and do not place main-window section strings into `preferences.ftl`.

## Implementation Playbook

When adding a feature, follow this order:

1. Decide whether the feature is bootstrap-wide, per-window, prefs-only, or a utility.
2. Add or update a module in `src/modules/**` or `src/utils/**`.
3. Register it from the relevant hook in `src/hooks.ts`.
4. Add static assets in `addon/**` only if the feature needs packaged UI, locale strings, CSS, or pref defaults.
5. Reuse config-based IDs and localization helpers.
6. Add cleanup logic to `onMainWindowUnload()` or `onShutdown()` when the feature registers UI or observers, and review whether any non-toolkit Zotero registrations need explicit teardown.
7. Run repo validation with non-blocking commands:
   - `npm run build`
   - `npm run test -- --exit-on-finish`
   - `npm run lint:check`

## Safe Development Flow for Agents

Use this flow when operating autonomously:

1. Read or infer the plugin identity from `package.json > config`.
2. Change TypeScript or addon asset files in the correct layer.
3. Avoid any command that launches Zotero or waits for interactive release input.
4. Validate with `npm run build`.
5. Validate with `npm run test -- --exit-on-finish`.
6. Validate with `npm run lint:check`.
7. If manual Zotero UI verification is required, stop and ask a human unless the environment explicitly supports launching Zotero safely.

## Common Feature Routing

Route work to the same layer the template uses:

- Add menu items, prompts, item pane sections, or columns in a feature module modeled after `src/modules/examples.ts`.
- Add preference-window behavior in `src/modules/preferenceScript.ts` plus matching `addon/content/preferences.xhtml` and `addon/prefs.js` changes.
- Add plugin-wide state in `src/addon.ts`.
- Add lifecycle registration in `src/hooks.ts`.
- Add toolkit initialization changes in `src/utils/ztoolkit.ts`.
- Add locale or pref helpers in `src/utils/locale.ts` or `src/utils/prefs.ts`.

## Constraints Specific to This Template

- Do not bypass `package.json` config for addon identity.
- Do not grow `src/hooks.ts` into a large business-logic file when the behavior can live in modules.
- Do not use raw pref keys without the configured prefix.
- Prefer FTL-backed user-facing strings for new work, even though some demo/example strings in the template are hardcoded.
- Do not forget unload/shutdown cleanup for registered UI.
- Do not treat this as a React app or a browser-extension popup app; this template is a Zotero bootstrap plugin with XUL/XHTML-era surfaces.
- Do not run `npm start` from autonomous agents.
- Do not run plain `npm run test` when a non-blocking run is needed; append `-- --exit-on-finish`.
- Do not claim Zotero 9 support unless the plugin has actually been tested and the manifest compatibility range has been updated.

## Verification Checklist

Before considering work complete:

- Confirm the feature is registered from the correct lifecycle hook.
- Confirm IDs and chrome paths derive from `config` values.
- Confirm locale strings live in FTL files and code uses `getString()` or `getLocaleID()`.
- Confirm pref access uses either the existing XHTML preference bindings or `src/utils/prefs.ts`, depending on where the feature runs.
- Confirm cleanup happens on unload or shutdown, and double-check any non-toolkit Zotero registrations.
- Confirm `npm run build`, `npm run test -- --exit-on-finish`, and `npm run lint:check` pass.

## References

- `references/repo-map.md`
- `references/api-style.md`
- `../README.md`
- `../src/hooks.ts`
- `../src/modules/examples.ts`
- `../src/modules/preferenceScript.ts`
- `../src/utils/locale.ts`
- `../src/utils/prefs.ts`
- `../src/utils/ztoolkit.ts`
- `../addon/bootstrap.js`
- `../zotero-plugin.config.ts`
