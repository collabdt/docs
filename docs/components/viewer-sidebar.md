---
title: ViewerSidebar
description: The shared sidebar shell for the map and BIM viewers, including the tab strip and tab panels.
category: components
status: draft
last_updated: 2026-07-31
---

# ViewerSidebar

The left-hand sidebar that overlays a spatial viewer. `ViewerSidebar` picks the sidebar for whichever viewer is active; each viewer's sidebar then declares its tabs and hands them to `ViewerSidebarShell`, which renders the building header, the tab strip and the active tab's panel.

:::info Renamed in 0.4.5
This component was called `InfoSidebar` before `@collabdt/core@0.4.5`, and `ViewerSidebarShell` replaced `InfoSidebarContainer`. There is no deprecated alias — see the [changelog](../changelog.md) for the migration. The `useSidebar()` API is unchanged: `toggleInfoSidebar`, `openInfo` and `setOpenInfo` keep their names.
:::

## Usage

You rarely render `ViewerSidebar` yourself — `SidebarProvider` mounts it inside the resizable overlay. You reach for `ViewerSidebarShell` when building or changing a viewer's sidebar:

```tsx
import { ViewerSidebarShell } from '@collabdt/core/components/ui/ViewerSidebar/Shell';

import type { ViewerSidebarTab } from '@collabdt/core/components/ui/ViewerSidebar/sidebarTabs';

export function BimSidebar({ organization }) {
  const tabs: ViewerSidebarTab[] = [
    { id: 'file', content: <FileTab /> },
    { id: 'settings', content: <SettingsTab /> },
  ];

  return <ViewerSidebarShell tabs={tabs} organization={organization} />;
}
```

## Props

### `ViewerSidebarShell`

| Prop | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `tabs` | `ViewerSidebarTab[]` | Yes | — | The tabs this viewer contributes, in display order. |
| `organization` | `Organization` | No | — | Used by the header to label the location when no building is selected. |

### `ViewerSidebarTab`

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `id` | `SidebarTabType` | Yes | — | One of `file`, `layers`, `communication`, `sensors`, `settings`. Determines the tab's icon and label. |
| `content` | `React.ReactNode` | Yes | — | The panel body, rendered only while this tab is active. |
| `enabled` | `boolean` | No | `true` | `false` hides the tab from the strip entirely. Use this for permission gating. |

## Behaviour

- **Tab state lives in the store.** The shell reads `selectedTab` from `MenusContext` and dispatches `SET_SIDEBAR_SELECTED_TAB`. A viewer never wires this up itself.
- **Only the active tab's `content` is mounted.** Switching tabs unmounts the previous panel, so a viewer-coupled panel does not keep subscriptions alive in the background.
- **`selectedTab` survives a viewer switch, so it may name a tab the new viewer does not have.** When that happens the shell falls back to the first available tab and dispatches the correction, rather than rendering an empty body. Example: Sensors is active in the map, the user switches to the BIM viewer, which does not offer it, and lands on Files.
- **Disabled tabs are absent, not empty.** `enabled: false` removes the tab button. Gate on the declaration, not inside the panel — a tab the user can select but that renders nothing is a worse outcome than no tab.

## The tab strip

`SIDEBAR_TAB_META` maps each `SidebarTabType` to a lucide icon and a key in the `TabSelector` i18n namespace. It is the single place a tab's presentation is declared:

| `id` | Icon | i18n key |
|------|------|----------|
| `file` | `FolderClosed` | `fileLabel` |
| `layers` | `Layers` | `layersTitle` |
| `communication` | `MessageCircle` | `communicationTitle` |
| `sensors` | `Radio` | `sensorsTitle` |
| `settings` | `Settings` | `settingsTitle` |

Each tab renders its icon with the label underneath. When the strip is too narrow for readable labels, the labels drop and the icons carry it alone; the label stays available as the tooltip and the accessible name.

That threshold is measured, not a breakpoint. `useCompactTabStrip` observes the strip element with a `ResizeObserver` and compacts when the width available per tab falls under `MIN_TAB_LABEL_WIDTH` (64px). The sidebar's desktop width is user-resizable and persisted, so a viewport media query cannot tell that the user dragged the sidebar narrow on a wide screen. The strip starts compact and expands once measured, so the first paint is never a row of truncated stubs.

## Accessibility

The strip is a `role="tablist"` of `<button role="tab">` elements: `aria-selected` tracks the active tab, `aria-controls` points at the panel, and the panel carries `role="tabpanel"` with `aria-labelledby` back to its button. Focus is roving — the strip is one tab stop, and Left/Right move between tabs with Home/End jumping to the ends. Every tab keeps an `aria-label` even when its visible text is hidden, so icon-only mode is not a screen-reader regression.

## Tab panels

`ViewerSidebarPanel` is the body wrapper for a tab. It replaces the wrapper `div` and search-field markup each tab used to carry:

| Prop | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `children` | `React.ReactNode` | Yes | — | Panel content. |
| `variant` | `'sections' \| 'scroll'` | No | `'sections'` | `sections` does not scroll — its children manage their own overflow. `scroll` is a single padded, scrolling stack, used by the settings tabs. |
| `search` | `{ value, onChange, placeholder? }` | No | — | Renders a `SearchInput` above the content. |
| `className` | `string` | No | — | Merged with `cn`, so a utility here overrides the variant's default (e.g. `space-y-4`). |

Two tabs are fully shared and need no per-viewer copy:

- **`SensorsTab`** — identical for every viewer; `SensorsSection` resolves the active viewer itself.
- **`CommunicationTab`** — takes an optional `topics` node rendered above the comments. The BIM viewer passes its BCF topics list; the map viewer passes nothing.

## Adding a tab to a viewer

1. If the tab is a new kind, add it to `SidebarTabType` in `store/Menus/reducer.ts` and give it an entry in `SIDEBAR_TAB_META`.
2. Build the panel body wrapped in `ViewerSidebarPanel`.
3. Add `{ id, content }` to that viewer's `tabs` array, with `enabled` if it is permission-gated.

Nothing else changes — the strip, the icon, the keyboard handling and the fallback behaviour all come from the shell.

## Design Decisions

Each viewer sidebar had its own `TabSelector`, and the BIM and map copies were byte-identical. Consolidating them into `ViewerSidebarShell` plus a declarative tab list means the tab strip, its accessibility, and the store wiring have one implementation, and adding a tab to a viewer is a one-line change.

Tab *bodies* stay viewer-owned on purpose. `FileTab`, `LayersTab` and `SettingsTab` are each wired to a different engine (`@thatopen/components`, maplibre) and have genuinely diverged; forcing them behind a shared abstraction would trade real duplication for a leaky interface. Only `SensorsTab` and `CommunicationTab`, which were already the same code, moved into the shared module.

The strip is a hand-written tablist rather than the `Tabs` primitive in `components/ui/Tabs.tsx`. That primitive is styled with hardcoded `gray-*` values instead of theme tokens, and its Radix `TabsContent` would change the mount and unmount semantics of these viewer-coupled panels.

## Related

- [NavigationBar](./top-navigation-bar.md) — owns the button that opens this sidebar.
- [Viewer](./viewer.md)
- [State Management](../architecture/state-management.mdx) — the `Menus` store that holds `selectedTab`.
