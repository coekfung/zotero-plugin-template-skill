# Repository Map

## Runtime layers

### 1. Zotero bootstrap layer

- `addon/bootstrap.js`
  - Registers chrome content.
  - Loads the compiled script bundle into the plugin sandbox.
  - Calls `Zotero.__addonInstance__.hooks.onStartup()`.
  - Delegates window load/unload and shutdown to exported hooks.

### 2. Sandbox entry layer

- `src/index.ts`
  - Guards against duplicate initialization by checking `Zotero[config.addonInstance]` first.
  - Creates `new Addon()` when needed.
  - Sets `_globalThis.addon` and defines the `ztoolkit` getter.
  - Stores the addon instance at `Zotero[config.addonInstance]`.

### 3. Shared state layer

- `src/addon.ts`
  - Owns `addon.data`.
  - Stores `alive`, `env`, `initialized`, `ztoolkit`, locale state, prefs-window state, and dialog state.

### 4. Lifecycle dispatcher layer

- `src/hooks.ts`
  - Waits for Zotero readiness on startup.
  - Initializes locale.
  - Registers prefs, notifiers, shortcuts, item-pane integrations, and per-window UI.
  - Acts as the coordination layer, though the template still keeps some direct startup UI work here.

### 5. Feature layer

- `src/modules/examples.ts`
  - Demonstrates notifier registration, menus, shortcuts, item tree columns, item pane sections, prompt commands, dialogs, clipboard, file picker, and progress windows.
- `src/modules/preferenceScript.ts`
  - Owns preference-window behavior and virtualized-table setup.
  - Initializes pref-window state, localized columns, scripted event listeners, and row/table interactions.

### 6. Utility layer

- `src/utils/locale.ts`
  - Initializes Fluent localization and wraps string lookup.
- `src/utils/prefs.ts`
  - Wraps `Zotero.Prefs` with config-prefixed accessors.
- `src/utils/ztoolkit.ts`
  - Creates and configures `ZoteroToolkit`.
- `src/utils/window.ts`
  - Provides `isWindowAlive()`.

### 7. Packaged static layer

- `addon/manifest.json`
  - Declares addon metadata and Zotero version compatibility.
- `addon/prefs.js`
  - Declares pref defaults before scaffold prefixing.
- `addon/content/preferences.xhtml`
  - Wires preference pane load to `hooks.onPrefsEvent('load', { window })`.
  - Declares the preference-pane localization link, direct `preference="..."` bindings, the scripted table container, and Fluent help text placeholders.
- `addon/content/zoteroPane.css`
  - Shows packaged stylesheet usage.
- `addon/locale/**`
  - Holds Fluent strings per locale.
  - Mirrors file families such as `addon.ftl`, `mainWindow.ftl`, and `preferences.ftl` across locales.

## Build layer

- `package.json`
  - Defines plugin identity and npm commands.
- `zotero-plugin.config.ts`
  - Defines scaffold source roots, dist root, build defines, pref prefixing, and esbuild output path.
- `tsconfig.json`
  - Extends `zotero-types/entries/sandbox/` and includes `src` plus `typings`.
- `eslint.config.mjs`
  - Uses `@zotero-plugin/eslint-config` with a template-specific override for unused variables in example code.

## Development commands

- `npm start` → `zotero-plugin serve` (interactive/blocking; launches Zotero and watches files)
- `npm run build` → `zotero-plugin build && tsc --noEmit`
- `npm run test -- --exit-on-finish` → safe non-blocking test invocation for agents
- `npm run lint:check` → `prettier --check . && eslint .`

## Compatibility signals

- `addon/manifest.json` sets `strict_min_version` to `6.999` and `strict_max_version` to `8.*`.
- Treat the template as explicitly configured for Zotero 7 and 8.
- Treat Zotero 9 as future-looking only until tested and the compatibility range is updated.

## Testing

- `test/startup.test.ts`
  - Verifies the addon instance is mounted at `Zotero[config.addonInstance]`.
