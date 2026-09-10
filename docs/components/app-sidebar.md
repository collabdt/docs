---
title: AppSidebarContent
description: Main sidebar navigation component that renders viewer, dataset, and plugin menu items based on organization configuration and user role.
category: components
status: draft
last_updated: 2025-01-14
---

# AppSidebarContent

Renders the primary sidebar navigation for the CDT platform. Displays grouped menu items for 3D viewers (Map, BIM), datasets (Sites, Buildings, Files, Infrastructure), and plugins. Adapts to organization-specific content restrictions, user roles, and collapsed/expanded sidebar states.

## Usage

```tsx
import { AppSidebarContent } from '@collabdt/core/components/AppSidebarContent';

<AppSidebarContent
  organization={organization}
  countrySubdivisionsData={countrySubdivisions}
/>
```

## Props

| Prop | Type | Required | Default | Description |
|------|------|----------|---------|-------------|
| `organization` | `Organization` | Yes | — | The current organization object containing name, logo, allowed viewers, and language settings. |
| `countrySubdivisionsData` | `Record<string, string>` | No | `undefined` | Mapping of country subdivision codes to names, used to populate map context. |

## Behaviour

- On mount, dispatches `SET_ORGANIZATION` to `AppConfigContext` and optionally `SET_COUNTRY_SUBDIVISIONS` to `MapContext`.
- Clicking a menu item calls `handleChangeViewer`, which resets selected item/site/file state, sets view to `'table'`, and updates `currentViewer` in menus state.
- Menu items are filtered by `organization.appContent` (if defined) and by user role via `accessibleTo`.
- Collapsed state hides text labels and centers icons; on mobile, the sidebar sheet open state determines layout.
- Footer displays language switcher (if organization supports multiple languages) and service links (e.g., support email).
- Active viewer is visually highlighted with primary color styling.

## Design Decisions

AppSidebarContent is responsible for navigation and organization context — it controls which viewers are available, who can see them, and dispatches the viewer change when a user clicks a menu item.

Menu items are defined as three static arrays (`viewerItems`, `datasetItems`, `managementItems`) rather than one flat list. This grouping maps directly to the three sidebar sections rendered in the UI, and makes it easy to add, remove, or reorder items within a section without affecting the others. Commented-out items (Land, Users, Feedback) are intentionally left in place as placeholders for features that are partially implemented or pending — removing them entirely would lose the context of where they belong.

Viewer availability is controlled by `appContent` on the Organization model. If `appContent` is empty, all viewers are shown; otherwise the list is filtered to only what the organization has enabled. This filtering happens at the item level via `.filter(item => appContent.includes(item.id))` so the sidebar automatically reflects each organization's configuration without any additional logic.

The list itself comes from `resolveAppContent(organization)` in `src/core/utils/appContent.ts`, which the sidebar shares with any other entry point into a viewer, notably the map building popover's tool row. Keeping one implementation matters because a tool that navigates to a viewer the sidebar hides is a dead end: the user arrives somewhere they cannot navigate back to. `resolveAppContent` always includes `map` and treats an unconfigured `appContent` as "everything", so an organization that never set the field keeps the full app.

Role-based visibility is handled via the `accessibleTo` field on `MenuItem` and the `canRenderItem` callback, which checks the current user's role against the allowed roles for each item. This is intentionally separate from CASL — it controls whether a nav item is visible at all, while CASL controls what actions are available once inside a viewer.

The `handleChangeViewer` function is exported so it can be called from other parts of the app (e.g. map interactions, HeaderButtons) that need to trigger a viewer change without going through the sidebar directly. It always resets selected item, site, file, and view state to prevent stale detail views carrying over between viewers.

Collapsed/expanded state drives label visibility and layout adjustments throughout. On mobile, the sheet open state is treated as expanded so labels render correctly when the drawer is visible — this is handled by the `isCollapsed` derived value rather than reading `sidebarState` directly.

## Permissions

Menu items can be restricted by role using the `accessibleTo` property. Items are rendered only if the user's role matches one of the allowed roles.

```tsx
{
  title: t('users'),
  id: ViewerNames.users,
  accessibleTo: [RoleNames.admin],
  // ...
}
```

## Related

- [Sidebar UI Components](/docs/components/app-sidebar)
- [LanguageSwitch](/docs/components/overview)
- [Logo](/docs/components/overview)
- [useMenusContext](/docs/hooks/ui)
- [useUserRole](/docs/hooks/users)
- [Organization Data Model](/docs/architecture/data-model#organization)
- [ViewerNames](/docs/components/overview)
