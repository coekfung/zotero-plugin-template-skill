# API and Style Map

## Zotero APIs exercised by the template

This template is a Zotero bootstrap plugin pattern that is explicitly configured through Zotero 8 in its checked-in manifest range. Use the API list below as the baseline for Zotero 7/8-era development, and re-test before claiming Zotero 9 compatibility.

### Lifecycle and readiness

- `Zotero.initializationPromise`
- `Zotero.unlockPromise`
- `Zotero.uiReadyPromise`
- `Zotero.getMainWindows()`
- `Zotero.Promise.delay()`
- `Zotero.Promise.defer()`

Use these to gate startup and UI rendering instead of assuming Zotero is ready.

### Preferences

- `Zotero.PreferencePanes.register()`
- `Zotero.Prefs.get()`
- `Zotero.Prefs.set()`
- `Zotero.Prefs.clear()`

Use `src/utils/prefs.ts` when code needs programmatic pref access. The provided preference pane itself is primarily wired through `addon/content/preferences.xhtml` bindings plus `src/modules/preferenceScript.ts`.

Preference-pane pattern in this template:

- `preferences.xhtml` carries the pane markup, control bindings, localization link, and load hook.
- `preferenceScript.ts` builds richer scripted UI such as `ztoolkit.VirtualizedTable` and attaches event listeners after the pane loads.
- `preferences.ftl` carries pref-pane-only strings, while shared/plugin strings remain in `addon.ftl` and main-window strings remain in `mainWindow.ftl`.

### Observation and plugin cleanup

- `Zotero.Notifier.registerObserver()`
- `Zotero.Notifier.unregisterObserver()`
- `Zotero.Plugins.addObserver()`

Use these when the feature needs Zotero event streams or plugin-shutdown cleanup hooks.

### Item list and item pane extension points

- `Zotero.ItemTreeManager.registerColumns()`
- `Zotero.ItemPaneManager.registerInfoRow()`
- `Zotero.ItemPaneManager.registerSection()`
- `Zotero.ItemPaneManager.unregisterSection()`

Mirror the patterns in `src/modules/examples.ts` when extending list columns or pane sections.

### Search and item lookup

- `new Zotero.Search()`
- `Zotero.Items.get()`
- `Zotero.Items.getAsync()`

Use these for item-based commands and prompt-driven searches.

## Toolkit surfaces exercised by the template

- `new ZoteroToolkit()`
- `ztoolkit.log()`
- `ztoolkit.UI.createElement()`
- `ztoolkit.Menu.register()`
- `ztoolkit.Keyboard.register()`
- `new ztoolkit.ProgressWindow()`
- `new ztoolkit.Dialog()`
- `new ztoolkit.VirtualizedTable()`
- `ztoolkit.Prompt.register()`
- `new ztoolkit.Clipboard()`
- `new ztoolkit.FilePicker()`
- `ztoolkit.unregisterAll()`

Prefer these wrappers when working on packaged UI because the template already relies on them for registration and much of its cleanup.

## Coding style recommended for new work, based on the repo

### Naming

- Classes and factories use `PascalCase`.
- Functions, hooks, helpers, and locals use `camelCase`.
- IDs and pref names derive from `config.addonRef`.

### Typing

- Prefer TypeScript signatures and helper generics.
- Keep ambient runtime names in `typings/global.d.ts`.
- Accept narrow runtime escape hatches only where the sandbox/global model requires them.

### Error handling

- Use straightforward `async/await`.
- Guard plugin state with checks like `if (!addon?.data.alive) return` where appropriate.
- Log through `ztoolkit.log()`.

### UI and localization

- Use toolkit UI creation or declarative XHTML.
- Prefer user-facing strings in Fluent files for new work.
- Resolve Fluent keys with `getString()` and `getLocaleID()` rather than hardcoding prefixed IDs.

### Cleanup discipline

- Unregister toolkit-managed UI on `onMainWindowUnload()` and `onShutdown()`, and review non-toolkit Zotero registrations separately.
- Close plugin-owned dialogs/windows during cleanup.
