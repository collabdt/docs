---
title: NavigationBar
description: Top navigation bar that provides search and sidebar toggle controls for different viewer modes.
category: components
status: draft
last_updated: 2025-01-14
---

# NavigationBar

Top-level navigation bar that adapts its contents based on the current viewer mode. Displays a sidebar toggle button and viewer-specific search tools (Geocoder for map, BIMSearchTool for BIM). Hides itself when the info sidebar is open in viewers that support it.

## Usage

```tsx
import NavigationBar from '@collabdt/core/components/NavigationBar';

// Typically rendered at the layout level, not directly instantiated
<NavigationBar />
```

## Props

This component accepts no props. It reads viewer state from `MenusContext` and sidebar state from `useSidebar()`.

## Behaviour

- **Viewer-aware rendering**: The component checks `currentViewer` from `MenusContext` to determine which search tool to display:
  - `ViewerNames.map` → renders `Geocoder`
  - `ViewerNames.bim` → renders `BIMSearchTool`
- **Sidebar toggle**: A menu button appears when the current viewer is BIM or map. Clicking it calls `toggleInfoSidebar()`, which opens the [ViewerSidebar](./viewer-sidebar.md).
- **Auto-hide**: When `openInfo` is true and the viewer needs a sidebar, the entire NavigationBar returns `null` to avoid visual overlap.
- **Hover state**: The sidebar toggle button transitions from 70% to 100% opacity on hover.

## Design Decisions

<!-- TODO: Why was this component built this way? Note any tradeoffs, constraints, or alternatives considered. -->

## Permissions

This component is not gated by CASL permissions.

## Related

- [Geocoder](/docs/components/top-navigation-bar) — map search tool
- [BIMSearchTool](/docs/components/top-navigation-bar) — BIM search tool
- [Sidebar](/docs/components/app-sidebar) — sidebar context provider and hooks
- [MenusContext](/docs/architecture/state-management) — viewer state management