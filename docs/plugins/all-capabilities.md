---
title: Capabilities
description: Everything a CDT plugin can add, the fields each registration needs, and the props each contribution receives.
sidebar_position: 3
category: plugins
status: draft
last_updated: 2026-08-17
---

# Capabilities

A capability is a place in CDT where a plugin can add something. A plugin declares the capabilities it uses in its manifest, then calls `ctx.register()` once per contribution.

These seven are the whole list. Registering anything else is a compile error, and registering a capability that is not declared in the manifest throws.

| Capability | Where it appears | Fields |
|---|---|---|
| `map.tools` | Map toolbar | `id`, `label`, `icon`, `component` |
| `bim.tools` | BIM toolbar | `id`, `label`, `icon`, `component` |
| `map.layers` | Drawn on the map | `id`, `component` |
| `viewer.legends` | Legend card, map and BIM viewer | `id`, `title`, `useLegend`, optional `viewers` |
| `viewer.tabs` | Viewer sidebar, as a tab | `id`, `labelKey`, `icon`, `component`, optional `viewers` |
| `data.pages` | Datasets nav, as a full page | `id`, `titleKey`, `icon`, `useRows`, `columns` |
| `ui.dialogs` | Anywhere, opened by id | `id`, `titleKey`, `component`, optional `size` |

Each `id` must be unique within the plugin. Reusing one silently drops the second registration.

## Toolbar tools

The three toolbars share one registration shape. The plugin supplies the panel content; CDT supplies the button and the dropdown around it.

```ts
ctx.register('bim.tools', {
  id: 'spaces',
  label: 'Spaces',
  icon: 'Boxes',            // a lucide icon name, or a component
  component: SpacesPanel,
  stayActive: true,         // optional: keep the panel open
})
```

Naming the icon as a string is what survives a JSON manifest, and it means no icon package is ever imported. An unknown name falls back to a placeholder rather than breaking the toolbar.

Each toolbar passes its own viewer to the component as props:

```ts
// map.tools
interface MapToolProps {
  map: import('maplibre-gl').Map | null
}

// bim.tools
interface BimToolProps {
  components: OBC.Components | null
  world: OBC.World | null
  fragments: OBC.FragmentsManager | null
  modelIds: string[]

  selection: ModelIdMap              // live; updates as the user clicks
  select: (items: ModelIdMap) => Promise<void>
  clearSelection: () => void
  fitToSelection: () => Promise<void>

  isolate: (items: ModelIdMap) => Promise<void>
  setItemsVisible: (items: ModelIdMap, visible: boolean) => Promise<void>
  showAll: () => Promise<void>

  getItemsOfCategory: (category: string) => Promise<ModelIdMap>
  getProperties: (items: ModelIdMap, attributes?: string[]) => Promise<BimItemProperties[]>
}
```

`ModelIdMap` is `{ [modelId]: Set<localId> }` — keyed by model, because more than one can be loaded at once.

Every handle is nullable, since a component can render before the viewer has finished initialising. Guard rather than assert.

:::tip IFC spaces start hidden
`getItemsOfCategory('IFCSPACE')` finds the spaces whether or not they are visible, and they are hidden by default, being volumetric solids that would obscure everything inside them. Call `setItemsVisible(spaces, true)` as well.
:::

## Colouring elements

Colour and opacity come from a hook rather than from `BimToolProps`, because they are scoped to the calling plugin. That also makes painting available from a sidebar tab, not only from a `bim.tools` panel.

:::note
`usePluginBimAppearance` is currently resolvable only from a plugin compiled into core. It is not among the entries CDT publishes to a mounted plugin — see [What a plugin can import](./mounting-a-plugin.md#what-a-plugin-can-import).
:::

```tsx
const { setAppearance, clearAppearance } = usePluginBimAppearance()

// Colour is 0xRRGGBB, opacity is 0-1, and either may be omitted.
setAppearance(spaces.map(space => ({
  items: space.items,
  appearance: { color: space.colour },
})))

clearAppearance()   // back to the model's own colours
```

**Pass every group in one call, not one call per group.** A plugin holds a single paint entry, so calling `setAppearance` in a loop replaces the previous group each time and leaves only the last one painted. Twenty spaces in twenty colours is one call with twenty groups. An empty list clears the paint.

**Paint from an effect rather than a click handler** where the colours can change. A re-render with new colours then repaints, and the model follows the list instead of waiting for another button press.

Painting a hidden element colours something that is not visible, so call `setItemsVisible(items, true)` first.

Paint is scoped per plugin, so two plugins painting the same model do not clobber one another and either can be cleared alone.

## Legends

`viewer.legends` adds a section to the one shared legend card, which appears in both the map and the BIM viewer. `viewers` narrows a legend to where it makes sense. The registration takes a hook, so the legend can re-read live counts.

```ts
ctx.register('viewer.legends', {
  id: 'my-legend',
  title: 'Sensors',
  viewers: [ViewerNames.bim],   // omit to appear in every viewer
  useLegend: () => ({
    active: true,               // false: the section is left out entirely
    rows: [{ label: 'Warm', color: '#ef9161', count: 12 }],
  }),
})
```

## Drawing on the map

`map.layers` registers a component CDT mounts for as long as the map exists. It renders `null` — everything it does goes through the map handle.

```ts
ctx.register('map.layers', { id: 'markers', component: MarkersLayer })
```

Use it rather than a `map.tools` panel whenever the drawing has to outlive the toolbar. **A tool's panel is a dropdown: it unmounts when it closes and takes its sources and layers with it.** Anything drawn from a sidebar tab or a data page has no panel at all.

```tsx
export function MarkersLayer({ map }: MapToolProps) {
  useEffect(() => {
    if (!map) return

    const ensureLayer = () => {
      if (!map.getStyle() || map.getSource(SOURCE)) return
      map.addSource(SOURCE, { type: 'geojson', data: featureCollection })
      map.addLayer({
        id: LAYER, type: 'circle', source: SOURCE,
        paint: { 'circle-color': ['get', 'colour'], 'circle-radius': 7 },
      })
    }

    if (map.isStyleLoaded()) ensureLayer()
    map.on('styledata', ensureLayer)
    map.on('click', LAYER, onFeatureClick)

    return () => {
      map.off('styledata', ensureLayer)
      map.off('click', LAYER, onFeatureClick)
      if (map.getLayer(LAYER)) map.removeLayer(LAYER)
      if (map.getSource(SOURCE)) map.removeSource(SOURCE)
    }
  }, [map])

  return null
}
```

Three common mistakes:

- **Re-add on `styledata`, not only once.** Switching the basemap replaces the style and silently drops every source and layer added before it.
- **Guard both cleanup calls.** The style can be torn down before cleanup runs, and removing a layer that is already gone throws. A source left behind makes the next mount fail on a duplicate id.
- **`maplibre-gl` cannot be imported,** so `new maplibregl.Marker()` and `new maplibregl.Popup()` are unavailable. A GeoJSON source with a circle or symbol layer does the same job and pans and zooms on the GPU for free. A popup can be built by portalling an element into `map.getContainer()` and positioning it with `map.project()`.

Update features with `setData` rather than removing and re-adding the layer, or they flicker on every change.

For colours, `stringToColour(key)` and `MAP_COLOUR_PALETTE` are exported from `@collabdt/core/plugins-sdk`. They are the platform's colourblind-accessible palette, and `stringToColour` is deterministic, so the same key always produces the same colour.

## Data pages

`data.pages` puts a full page in the Datasets group of the sidebar, beside Buildings and Sites. CDT renders the frame, breadcrumb, title, search box and table; the plugin supplies the rows and columns.

```ts
ctx.register('data.pages', {
  id: 'rooms',
  titleKey: 'rooms.title',
  icon: 'Table',
  useRows: useRoomRows,
  columns: [
    { key: 'name',  labelKey: 'rooms.name' },
    { key: 'floor', labelKey: 'rooms.floor' },
  ],
  emptyKey: 'rooms.empty',     // optional
  searchKeys: ['name'],        // optional; omit to search every column
})
```

`useRows` is a **hook**, not a value. CDT calls it while rendering the page, so the rows can come from anywhere a hook can reach. It returns the rows and, optionally, what clicking one does:

```ts
function useRoomRows(): DataPageRows<Room> {
  const { items } = usePluginStore<Room>('rooms')
  const { open } = usePluginDialogs()

  return {
    rows: items.map(item => ({ key: item.key, ...item.data })),
    onRowClick: row => open('room-detail', { roomKey: row.key }),
  }
}
```

`onRowClick` is returned from the hook rather than set on the registration because it usually needs other hooks, and `activate()` runs outside React. Omit it and the rows stay non-interactive, which is also what a screen reader is told.

A column renders its raw value unless given a `render`. Values that are not strings, numbers or booleans render empty rather than `[object Object]`.

The data behind a plugin page belongs to the plugin, so the permission subject to check is `PluginRecord`. Use `usePluginPermissions()` from `@collabdt/core/plugins-sdk/data` to hide controls the person may not use. Either way the server re-checks every write, so a rejected one should be handled visibly rather than prevented only by hiding a control.

## Viewer sidebar tabs

`viewer.tabs` adds a tab to the viewer sidebar, beside Files, Layers and Sensors. CDT owns the tab strip and the panel frame.

```ts
ctx.register('viewer.tabs', {
  id: 'rooms',
  labelKey: 'rooms.tab',
  icon: 'ListChecks',
  viewers: [ViewerNames.map, ViewerNames.bim],   // omit to appear in all of them
  component: RoomsTab,
})
```

Import `ViewerNames` from `@collabdt/core/plugins-sdk`; it is exported as a value so that a tab or a legend can name its viewers. A mounted plugin built against `@collabdt/plugin-kit` spells them as plain strings instead — `viewers: ['map', 'bim']`, typed as `PluginViewerTarget`.

:::tip Say where it goes
Omitting `viewers` means every viewer, which is rarely a location anyone chose. `create-cdt-plugin` now writes the list explicitly, taken from the viewer surfaces you scaffolded with — pick `bim.tools` and a tab, and you get `viewers: ['bim']`.

Only `'map'` and `'bim'` host tabs and legends. Any other name — a typo like `'BIM'`, or a `ViewerNames` member such as `settings` that is a route rather than a viewer — renders nowhere; the platform logs a warning naming the plugin and the value.
:::

The component receives no props. It renders inside the panel, so it should fill the width and let the panel scroll.

## Dialogs

`ui.dialogs` registers a modal that CDT owns, opened by id from any other surface of the same plugin.

```ts
ctx.register('ui.dialogs', {
  id: 'room-detail',
  titleKey: 'rooms.detail',
  size: 'lg',            // 'sm' | 'md' | 'lg' | 'xl', default 'md'
  component: RoomDetail,
})
```

```ts
const { open, close } = usePluginDialogs()

open('room-detail', { roomKey: 'r-12' })   // props reach the component
close('room-detail')                        // or the component calls its own `close`
```

The component receives whatever `open` passed, plus a `close` function. CDT supplies the overlay, the title bar, the focus trap and Escape; the plugin renders the body only.

Two consequences of CDT owning the dialog stack:

- **A dialog outlives whatever opened it.** One opened from a map tool's panel stays on screen after that panel closes.
- **A plugin can only open its own dialogs.** The plugin id comes from the scope CDT established, so naming another plugin's dialog id addresses nothing.

## Reading platform data

Buildings, sites, sensors, comments and files come from `@collabdt/core/plugins-sdk/data`. A plugin goes through the same request path as the rest of CDT, so it inherits the signed-in user's session, the organization scoping and the shared cache — there is no second data path and no way past the tenant boundary.

```tsx
import { useBuildings, useSensorsByBuilding } from '@collabdt/core/plugins-sdk/data'

function Panel() {
  const { buildings, isLoading } = useBuildings()
  const { sensors } = useSensorsByBuilding(buildings[0]?.id ?? null)

  if (isLoading) return null
  return <p>{sensors.length} sensors</p>
}
```

Every read hook returns its payload plus `isLoading` and `isError`. A list hook returns `[]` before it resolves rather than `undefined`, so it can be mapped straight away; a single-record hook returns `null`.

| Read | Hook |
|---|---|
| Buildings | `useBuildings`, `useBuilding`, `useBuildingsByOsm`, `useBuildingOsmIds` |
| Sites | `useSites`, `useSite` |
| Infrastructure | `useInfrastructures`, `useInfrastructure` |
| Organization | `useOrganization`, `useOrganizationByName` |
| Files | `useFiles`, `useFile`, `useFilesByBuildingId`, `useFilesBySiteId`, `useDownloadFile` |
| Sensors | `useSensors`, `useSensor`, `useSensorsByBuilding`, `useSensorsByAuthor`, `useSensorTypes`, `useSensorType` |
| Comments | `useComments`, `useComment`, `useCommentsByBuilding`, `useCommentsByAuthor` |

Buildings and sites are read-only to a plugin: they are canonical asset records, so changing one is a change to CDT rather than to a plugin. Sensors and comments are the domains plugins are expected to author, and they have `useCreateSensor`, `useCreateComment` and `useDeleteComments`. A plugin's own records go in `usePluginStore`.

Types come from `@collabdt/plugin-kit/types/data`, which declares the fields the SDK commits to rather than every column CDT's schema carries.

## Where to keep state

Surfaces share state through hooks rather than props. The choice depends on whether the value belongs in a database:

| Kind of value | Hook | Survives a reload |
|---|---|---|
| A selection, a filter, a draft | `usePluginState` | No |
| Records the plugin owns | `usePluginStore` | Yes |
| Per-user preferences | `usePluginConfig`, via user settings | Yes |
| Org-wide settings | `usePluginConfig` | Yes |

`usePluginState` is in-memory and scoped per plugin, so two plugins using the key `selected` never see each other's value. `usePluginStore` will hold a selection, but makes every click a database write.

[Example: one plugin, several surfaces](./hello-map-example.md) shows both in one plugin.

## Registering more than one contribution

`activate()` can call `ctx.register()` as many times as needed, under one capability or several:

```ts
// manifest.json: "capabilities": ["map.tools", "viewer.legends"]
export function activate(ctx: PluginContext): void {
  ctx.register('map.tools', { id: 'rooms-inspect', label: 'Inspect', icon: 'Search', component: InspectTool })
  ctx.register('map.tools', { id: 'rooms-measure', label: 'Measure', icon: 'Ruler', component: MeasureTool })
  ctx.register('viewer.legends', { id: 'rooms-legend', title: 'Rooms', useLegend })
}
```

If any `register` call names a capability missing from the manifest, CDT removes every contribution that plugin made and marks it errored. Activation is all-or-nothing, so a half-registered plugin never occurs.
