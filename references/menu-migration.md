# Menu Migration Guide

## Why this guide exists

Older template-era examples used `ztoolkit.Menu.register()`. Current toolkit releases such as `zotero-plugin-toolkit@5.1.2` no longer export a `Menu` helper, so menu integrations should use Zotero's built-in menu API instead:

- `Zotero.MenuManager.registerMenu()`
- `Zotero.MenuManager.unregisterMenu()`

This guide describes the Zotero 8 migration shape for moving from older helper-based menus to `Zotero.MenuManager`.

## Compatibility

After applying the migration described in this guide, treat the resulting menu implementation as Zotero 8-only unless you separately verify backward compatibility. In particular, field-menu usage such as `itemPane/info/row` should be treated as a Zotero 8+ pattern.

## Core registration shape

Use this top-level option structure:

```ts
Zotero.MenuManager.registerMenu({
  pluginID: addon.data.config.addonID,
  menuID: "item-menu",
  target: "main/library/item",
  menus: [
    {
      menuType: "menuitem",
      l10nID: getLocaleID("menuitem-label"),
      onCommand: (_event, context) => {
        addon.hooks.onLintInBatch("standard", context.items!);
      },
    },
  ],
});
```

The required ideas are:

- `pluginID`: your addon ID
- `menuID`: unique identifier for this registered menu group
- `target`: the Zotero menu surface to extend
- `menus`: one or more menu descriptors

## Menu descriptor shape

Use `menuType` values such as:

- `menuitem`
- `separator`
- `submenu`

Typical descriptor fields include:

- `l10nID`
- `icon`
- `darkIcon`
- `enableForTabTypes`
- `onShowing`
- `onShown`
- `onHiding`
- `onHidden`
- `onCommand`
- `menus` for submenu children

Example submenu shape:

```ts
{
  menuType: "submenu",
  l10nID: getLocaleID("menuitem-label"),
  icon,
  menus: [
    {
      menuType: "menuitem",
      l10nID: getLocaleID("menuitem-stdFormatFlow"),
      onCommand: (_event, { items }) => {
        if (items) addon.hooks.onLintInBatch("standard", items);
      },
    },
    {
      menuType: "separator",
    },
  ],
}
```

## Target strings for the Zotero 8 migration path

Common targets include:

- `main/library/item`
- `main/library/collection`
- `main/menubar/file`
- `main/menubar/edit`
- `main/menubar/view`
- `main/tab`
- `reader/menubar/file`
- `itemPane/info/row`

Choose the target based on the UI surface you are extending, not based on where the callback code lives.

## Context-driven menu behavior

This migration style relies heavily on context helpers in lifecycle hooks.

In `onShowing` and related hooks, expect a context object with helpers such as:

- `menuElem`
- `setVisible(...)`
- `setEnabled(...)`
- `setL10nArgs(...)`
- `setIcon(...)`

This enables patterns like:

- hiding a field-row menu unless the current `fieldName` matches
- disabling commands when multiple items are selected
- updating menu labels or args right before display

Representative pattern:

```ts
onShowing: (_event, context) => {
  const visible = rule.targetItemField === context.fieldName;
  context.setVisible(visible);
  if (!visible) return;

  const disabled = !!rule.fieldMenu?.setDisabled?.(context);
  context.setEnabled(!disabled);
};
```

## Localization pattern

For menus in shared Zotero windows, prefer Fluent-backed IDs:

```ts
l10nID: getLocaleID("menuitem-label");
```

Prefer this over raw `label` strings so the menu participates in the same localization flow as the rest of the plugin UI.

For menu-facing Fluent messages, prefer `.label` attributes instead of bare value-only entries. This is the safer cross-platform shape for shared-window menu labels.

Example migration:

```ftl
rule-require-language-menu-item =
  .label = Auto detect item language
```

If the menu appears in a shared Zotero window, make sure the relevant Fluent files are inserted into that window before relying on `l10nID`. A representative pattern is:

```ts
getLocaleFileFullNames(localeFilesForMainWindow).forEach((file) =>
  (win as any).MozXULElement.insertFTLIfNeeded(file),
);
```

That pattern matters for migration because replacing old helper-based menu labels with `l10nID` is not sufficient on its own. The window also needs the matching FTL files registered through `insertFTLIfNeeded(...)`, and those links should be removed on window unload when the plugin cleans up.

Keep menu-facing strings in a window-oriented FTL file such as `main-window.ftl`, loaded for shared-window usage, instead of assuming a generic addon-level FTL file alone is enough for menu localization.

If a menu's localized string may not be ready immediately on first display, use a defensive `onShown` pattern that checks `menuElem.textContent` and repairs or warns if localization is missing.

## TypeScript pattern

Use a narrow typed style for menu descriptors when `_ZoteroTypes.MenuManager.*` definitions are available:

```ts
type FieldMenu =
  _ZoteroTypes.MenuManager.MenuData<_ZoteroTypes.MenuManager.ItemPaneMenuContext>;
type ItemMenu =
  _ZoteroTypes.MenuManager.MenuData<_ZoteroTypes.MenuManager.LibraryMenuContext>;
```

Use narrow menu/context types when the codebase already has access to `_ZoteroTypes.MenuManager.*` definitions. This makes `context.items`, `context.fieldName`, and other target-specific properties safer to use.

## Migration rules from older template code

When migrating older `ztoolkit.Menu.register()` examples:

1. Replace helper-specific registration calls with `Zotero.MenuManager.registerMenu({...})`.
2. Convert old tag-based menu nodes into `menuType` descriptors.
3. Replace direct label strings with `l10nID` where the menu appears in shared Zotero UI.
4. Ensure the target window has the required Fluent files inserted with `MozXULElement.insertFTLIfNeeded(...)` before relying on `l10nID`.
5. Convert menu-facing Fluent messages to `.label` attributes when migrating older bare-value entries.
6. Replace inline DOM-placement assumptions with the correct `target` string.
7. Use nested `menus` arrays for submenus.
8. Move runtime enable/visible logic into `onShowing` and related hooks.
9. Unregister manually with `Zotero.MenuManager.unregisterMenu(menuID)` only when needed; plugin unload handling in Zotero will also remove plugin-owned menu registrations.

## Mapping from old helper style to current Zotero API

Use this conceptual mapping:

- old helper target like item context menu → `target: "main/library/item"`
- old helper target like collection context menu → `target: "main/library/collection"`
- old helper target like File menu → `target: "main/menubar/file"`
- old popup children arrays → nested `menus` on a `submenu`
- old `commandListener` / inline command strings → `onCommand` callback functions

## Recommended practice for this template

In this template family:

- register menus from lifecycle/setup code such as `src/hooks.ts`-triggered module functions
- keep menu construction inside feature modules, not in bootstrap wiring
- derive `pluginID` from config
- derive `l10nID` with `getLocaleID(...)`
- ensure shared-window FTL files are inserted with `insertFTLIfNeeded(...)` before menu display
- keep menu icons under packaged addon assets and reference them via `rootURI` or `chrome://...`

## References

- Local skill summary: `../SKILL.md`
- Local API map: `./api-style.md`
- Menu example reference: `https://github.com/northword/zotero-format-metadata/blob/main/src/modules/menu.ts`
- Locale registration reference: `https://github.com/northword/zotero-format-metadata/blob/main/src/utils/locale.ts`
- Zotero core menu API implementation: `https://github.com/zotero/zotero/blob/main/chrome/content/zotero/xpcom/pluginAPI/menuManager.js`
