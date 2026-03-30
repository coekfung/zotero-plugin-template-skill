# Menu Migration Guide

## Why this guide exists

Older template-era examples used `ztoolkit.Menu.register()`. Current toolkit releases such as `zotero-plugin-toolkit@5.1.2` no longer export a `Menu` helper, so menu integrations should use Zotero's built-in menu API instead:

- `Zotero.MenuManager.registerMenu()`
- `Zotero.MenuManager.unregisterMenu()`

This guide describes the migration shape and is informed by both the local template migration and a Zotero 8-style upstream example from `northword/zotero-format-metadata/src/modules/menu.ts`.

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

## Target strings seen in Zotero 8-style usage

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

The upstream Zotero 8-style example uses context helpers heavily in lifecycle hooks.

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

If a menu's localized string may not be ready immediately on first display, the upstream example shows a defensive `onShown` pattern that checks `menuElem.textContent` and repairs or warns if localization is missing.

## TypeScript pattern from upstream example

The upstream example also shows a useful typed style for menu descriptors:

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
4. Replace inline DOM-placement assumptions with the correct `target` string.
5. Use nested `menus` arrays for submenus.
6. Move runtime enable/visible logic into `onShowing` and related hooks.
7. Unregister manually with `Zotero.MenuManager.unregisterMenu(menuID)` only when needed; plugin unload handling in Zotero will also remove plugin-owned menu registrations.

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
- keep menu icons under packaged addon assets and reference them via `rootURI` or `chrome://...`

## References

- Local skill summary: `../SKILL.md`
- Local API map: `./api-style.md`
- Upstream Zotero 8-style example: `https://github.com/northword/zotero-format-metadata/blob/main/src/modules/menu.ts`
- Zotero core menu API implementation: `https://github.com/zotero/zotero/blob/main/chrome/content/zotero/xpcom/pluginAPI/menuManager.js`
