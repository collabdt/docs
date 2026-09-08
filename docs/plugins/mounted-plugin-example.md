---
title: 'Example: a mounted plugin against a real model'
description: What a runtime-loaded plugin can and cannot reach, built out as a room inventory across seven surfaces.
sidebar_position: 6
category: plugins
status: draft
last_updated: 2026-08-21
---

# Example: a mounted plugin against a real model

[The previous example](./hello-map-example.md) is compiled into CDT. This one is **mounted** —
built to a `dist/index.js`, dropped in a folder and loaded at runtime — which is how a plugin
written outside the CDT repo arrives. The difference matters more than it sounds: a mounted
plugin resolves a fixed list of imports and receives viewer handles only where the platform
passes them as props. Everything below follows from that.

The worked example is a room inventory. It reads every `IfcSpace` out of a building model,
classifies each one by programme, department and seat capacity, and shows the result in the
viewer, across the portfolio and on the map — seven surfaces from one manifest and one
`activate()`:

| Surface | What it does |
|---|---|
| `bim.tools` | Reads the rooms; show, isolate, reset |
| `viewer.tabs` | Classifies them and saves the classification |
| `map.tools` | Portfolio totals and a programme filter |
| `map.layers` | One figure per building, with a breakdown popup |
| `data.pages` | Every classified room, across every building |
| `ui.dialogs` | One room, opened from three of the others |
| `viewer.legends` | A row per programme in use, with live counts |

## Only one surface receives the viewer

A `bim.tools` component is handed [`BimToolProps`](./all-capabilities.md#toolbar-tools). A
`viewer.tabs` component is handed **nothing at all**, and `data.pages`, `ui.dialogs` and
`viewer.legends` contribute hooks rather than components. A compiled-in plugin can call
`useBimViewer()` from any of them; a mounted plugin cannot, because that module is not among
the entries CDT publishes.

So the reading happens in one place and the result is published to the rest:

```tsx
// bim.tools — the only surface holding the handles
export function InventoryTool(props: ToolbarToolProps & BimToolProps) {
  const { rooms } = useRoomScan(props)   // scans once, writes to usePluginState
  ...
}

// viewer.tabs — no props at all, reads what the tool published
export function InventoryTab() {
  const [rooms] = usePluginState<Room[]>('rooms', [])
  ...
}
```

Requests travel the same way. A tab cannot select or frame a room itself, so it leaves one:

```ts
export interface RoomCommand {
  action: 'select' | 'isolate' | 'showAll'
  key: string | null
  nonce: number        // asking twice for the same room has to happen twice
}
```

and the toolbar component carries it out in an effect keyed on the nonce.

:::caution The toolbar panel is not always mounted
CDT renders a toolbar contribution inside a dropdown, which unmounts its children when it
closes. A request left by another surface is therefore only acted on while that panel is open.
Say so in the interface rather than appearing to do nothing: the tool here claims a flag in
plugin state while it is mounted, and the tab disables **Isolate in 3D** and explains why when
the flag is false.
:::

## What `getProperties` reaches, and what it does not

`getProperties` forwards attributes and nothing else. Three things a space-planning plugin
wants are outside it:

- **The IFC `GlobalId`.** `Guid` in an attributes read is a numeric index local to the model,
  not the 22-character IFC identifier.
- **Quantities** — `IfcElementQuantity`, where the floor area lives.
- **Spatial containment** — which storey a space belongs to.

All three are reachable through the raw `fragments` handle, which the same props carry:

```tsx
const model = fragments.list.get(modelId)

// The durable identity. This is the key to store records against; a localId is
// only meaningful while the model is open.
const guids = await model.getGuidsByLocalIds(localIds)

const data = await model.getItemsData(localIds, {
  attributesDefault: true,
  relations: {
    IsDefinedBy: { attributes: true, relations: true },   // quantities and property sets
    Decomposes: { attributes: true, relations: false },   // the storey
  },
})
```

`@thatopen/components` may be imported for its **types**: type imports erase, so the built
bundle still imports nothing outside the published list. Importing it as a runtime value is a
build error, and rightly so.

The shapes that come back are not stable across IFC versions — a quantity sits at a different
depth in IFC2X3 than in IFC4 — so walk them defensively, with a depth cap, and treat a missing
value as a normal model rather than a failure. Plenty of real exports carry no quantities at
all.

:::tip IFC spaces start hidden
`getItemsOfCategory('IFCSPACE')` finds the spaces whether or not they are visible, and they are
hidden by default. Anything acting on one calls `setItemsVisible(items, true)` first.
:::

## Keeping a record in step with itself

`usePluginStore.put` replaces a record whole, and `items` lags the write that caused it. Two
edits to the same record in quick succession therefore lose the first, because the second
merges onto a snapshot taken before the first landed.

Merge against your own pending copy as well as the store:

```ts
const written = React.useRef(new Map<string, RoomRecord>())

const classify = async (key: string, changes: Partial<RoomRecord>) => {
  const current = written.current.get(key) ?? records.get(key)
  const next = { ...current, ...changes }

  written.current.set(key, next)   // before the await, so the next edit sees it
  await store.put(key, next)
}
```

Any plugin storing more than one field per record wants this.

## Reading platform data

The data hooks work exactly as they do for a compiled-in plugin — this is what
`@collabdt/core/plugins-sdk/data` is for. The dialog resolves a building name from an id the
record carries:

```tsx
const { buildings } = useBuildings()
const building = buildings.find(candidate => candidate.id === room.buildingId)
```

That id has to be *put* there by someone. **Nothing tells a plugin which building the open
model belongs to**: the SDK offers `useBuildings()` and `useBuilding(id)`, and no way to ask
which building the viewer is showing. This plugin has the user name it once in the tab and
stamps the answer onto every record, so the portfolio surfaces still work with no model open.

## Message keys and a mounted plugin

`titleKey`, `labelKey` and `emptyKey` resolve against the plugin's own message namespace and
fall back to the key itself. For a **mounted** plugin they always fall back today: the catalog
is assembled from the manifests compiled into the build, and a mounted manifest arrives too
late to join it.

Write those three as English prose rather than as keys, so an unresolved lookup still reads
correctly:

```ts
ctx.register('data.pages', {
  id: 'rooms',
  titleKey: 'Room inventory',            // renders as written
  columns: [{ key: 'name', labelKey: 'Room' }],
  emptyKey: 'No rooms classified yet.',
})
```

Strings inside components are unaffected — `usePluginTranslations()` takes an inline English
fallback at each call, which is why every example passes one.

## A data page row is a bag of unknowns

`CapabilityRegistry` pins a `data.pages` row to `Record<string, unknown>`, so a typed row
cannot cross the registration under `strict: true` — which is what the scaffolder emits.
Build the rows with a type of your own and narrow inside each column:

```tsx
export function useRows(): DataPageRows<Record<string, unknown>> {
  const rows = React.useMemo<Row[]>(() => /* … */, [])
  return { rows, onRowClick: row => typeof row.key === 'string' && open(row.key) }
}

export const columns: DataPageColumn<Record<string, unknown>>[] = [
  { key: 'area', labelKey: 'Area', render: row => format(typeof row.area === 'number' ? row.area : null) },
]
```

## Colouring from a mounted plugin

`usePluginBimAppearance` is not among the entries CDT publishes to a mounted plugin, so the SDK's
colour API is out of reach. Colour itself is not: the `fragments` handle carries it.

```tsx
// MaterialDefinition types `color` as a THREE.Color, but fragments only reads r, g and b.
// A plugin that cannot import three passes the three numbers and casts.
const material = { color: { r, g, b }, opacity: 1, transparent: false,
                   preserveOriginalMaterial: false } as unknown as FRAGS.MaterialDefinition

await model.highlight(localIds, material)   // one call per colour, never per element
await model.resetHighlight(localIds)        // back to the model's own colours
```

Four things to get right:

- **`preserveOriginalMaterial` must stay `false`.** At `true` fragments skips deduplication and
  spends one of the model's ~65 500 material slots per element per call. At `false` a colour
  costs one slot however many elements wear it — so bucket elements by colour and make one call
  per bucket.
- **Convert sRGB to linear.** `new THREE.Color(hex)`, which is what core passes on its own path,
  converts into the renderer's working space. Passing raw sRGB values gives visibly different
  colours from the rest of the app.
- **Show before painting.** `IFCSPACE` is hidden by default, so paint it without
  `setItemsVisible(items, true)` and you have coloured something invisible.
- **Paint is a change to the model, not to your component.** It outlives the toolbar panel that
  applied it — which is the only reason a colour survives the dropdown closing — so do not clear
  it in a cleanup function, and give the user a way to turn it off.

This bypasses core's own `ElementAppearance`, so plugin paint and the Layers tab's colouring
overwrite each other, and CTRL+Z does not undo the plugin's. For a plugin painting a category
the sidebar trees rarely touch — spaces, say — that is an acceptable trade; for anything else,
prefer being compiled in and using `usePluginBimAppearance`.

## What still needs core

**A `data.pages` contribution had no nav entry.** `AppSidebarContent` filtered the Datasets group
through `appContent.includes(item.id)`, and a `plugin:<pluginId>:<pageId>` key can never be a
member of a Prisma enum array — so every plugin page was reachable by URL and invisible in the
interface. Fixed in core by routing the nav filter through the same `isViewerAllowed` helper the
router already used. Nothing a plugin could do about it from outside.

Everything else on this page a plugin can do for itself today.
