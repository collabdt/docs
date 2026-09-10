---
title: Mounted plugins in practice
description: What a runtime-loaded plugin can and cannot reach, and the patterns that work around each limit.
sidebar_position: 6
category: plugins
status: draft
last_updated: 2026-09-10
---

# Mounted plugins in practice

[The previous example](./hello-map-example.md) is compiled into CDT. A **mounted** plugin — built to a `dist/index.js`, dropped in a folder and loaded at runtime, as described in [Run your plugin](./mounting-a-plugin.md) — is how a plugin written outside the CDT repo arrives, and it is the only route available to a self-hosted deployment.

The difference matters more than it sounds. A mounted plugin resolves a fixed list of imports, receives viewer handles only where the platform passes them as props, and never joins the message catalog. Everything on this page follows from those three facts, and none of it applies to a plugin compiled into core.

The examples come from a plugin that classifies IFC spaces across seven surfaces, but the constraints are the same whatever a mounted plugin does.

## Only one surface receives the viewer

Contributions are not equal. A `bim.tools` component is handed [`BimToolProps`](./all-capabilities.md#toolbar-tools); a `viewer.tabs` component is handed **nothing at all**, and `data.pages`, `ui.dialogs` and `viewer.legends` contribute hooks rather than components. A compiled-in plugin can call `useBimViewer()` from any of them; a mounted plugin cannot, because that module is not among the entries CDT publishes.

So do the work in the one place that holds the handles, and publish the result to the rest:

```tsx
// bim.tools — the only surface holding the handles
export function ScanTool(props: ToolbarToolProps & BimToolProps) {
  const { items } = useModelScan(props)   // scans once, writes to usePluginState
  ...
}

// viewer.tabs — no props at all, reads what the tool published
export function ResultsTab() {
  const [items] = usePluginState<Item[]>('items', [])
  ...
}
```

Requests travel the same way. A surface without handles cannot select, frame or hide anything itself, so it leaves a request for the one that can:

```ts
export interface ViewerCommand {
  action: 'select' | 'isolate' | 'showAll'
  key: string | null
  nonce: number        // asking twice for the same target has to happen twice
}
```

and the toolbar component carries it out in an effect keyed on the nonce.

:::caution The toolbar panel is not always mounted
CDT renders a toolbar contribution inside a dropdown, which unmounts its children when it closes. A request left by another surface is therefore only acted on while that panel is open. Say so in the interface rather than appearing to do nothing: have the tool claim a flag in plugin state while it is mounted, and have the other surfaces disable the affected controls and explain why when the flag is false.
:::

## Reaching past the SDK with the handles you are given

The props a viewer surface receives carry more than the SDK wraps. `getProperties` forwards attributes and nothing else, which leaves out three things a model-reading plugin usually needs: a durable IFC `GlobalId` — the `Guid` in an attributes read is a numeric index local to the model, not the 22-character identifier — quantities such as floor area, and spatial containment. All of them are reachable through the raw `fragments` handle on the same props:

```tsx
const model = fragments.list.get(modelId)

// A localId is only meaningful while the model is open; store records against the GUID.
const guids = await model.getGuidsByLocalIds(localIds)

const data = await model.getItemsData(localIds, {
  attributesDefault: true,
  relations: {
    IsDefinedBy: { attributes: true, relations: true },   // quantities and property sets
    Decomposes: { attributes: true, relations: false },   // the storey
  },
})
```

`@thatopen/components` may be imported for its **types**: type imports erase, so the built bundle still imports nothing outside the published list. Importing it as a runtime value is a build error, and rightly so.

The shapes that come back are not stable across IFC versions — a quantity sits at a different depth in IFC2X3 than in IFC4 — so walk them defensively, with a depth cap, and treat a missing value as a normal model rather than a failure. Plenty of real exports carry no quantities at all.

:::tip Some categories start hidden
`getItemsOfCategory('IFCSPACE')` finds spaces whether or not they are visible, and they are hidden by default. Anything acting on one calls `setItemsVisible(items, true)` first.
:::

## Keeping a record in step with itself

`usePluginStore.put` replaces a record whole, and `items` lags the write that caused it. Two edits to the same record in quick succession therefore lose the first, because the second merges onto a snapshot taken before the first landed.

Merge against your own pending copy as well as the store:

```ts
const written = React.useRef(new Map<string, ItemRecord>())

const edit = async (key: string, changes: Partial<ItemRecord>) => {
  const current = written.current.get(key) ?? records.get(key)
  const next = { ...current, ...changes }

  written.current.set(key, next)   // before the await, so the next edit sees it
  await store.put(key, next)
}
```

Any plugin storing more than one field per record wants this.

## Reading platform data

The data hooks work exactly as they do for a compiled-in plugin — this is what `@collabdt/core/plugins-sdk/data` is for:

```tsx
const { buildings } = useBuildings()
const building = buildings.find(candidate => candidate.id === record.buildingId)
```

That id has to be *put* there by someone. **Nothing tells a plugin which building the open model belongs to**: the SDK offers `useBuildings()` and `useBuilding(id)`, and no way to ask which building the viewer is showing. A plugin that needs the association has the user name it once and stamps the answer onto every record, so the surfaces that run with no model open still work.

## Message keys always fall back

`titleKey`, `labelKey` and `emptyKey` resolve against the plugin's own message namespace and fall back to the key itself. For a **mounted** plugin they always fall back today: the catalog is assembled from the manifests compiled into the build, and a mounted manifest arrives too late to join it.

Write those three as English prose rather than as keys, so an unresolved lookup still reads correctly:

```ts
ctx.register('data.pages', {
  id: 'items',
  titleKey: 'Inventory',                 // renders as written
  columns: [{ key: 'name', labelKey: 'Name' }],
  emptyKey: 'Nothing recorded yet.',
})
```

Strings inside components are unaffected — `usePluginTranslations()` takes an inline English fallback at each call, which is why every example passes one.

## A data page row is a bag of unknowns

`CapabilityRegistry` pins a `data.pages` row to `Record<string, unknown>`, so a typed row cannot cross the registration under `strict: true` — which is what the scaffolder emits. Build the rows with a type of your own and narrow inside each column:

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

`usePluginBimAppearance` is not among the entries CDT publishes to a mounted plugin, so the SDK's colour API is out of reach. Colour itself is not: the `fragments` handle carries it.

```tsx
// MaterialDefinition types `color` as a THREE.Color, but fragments only reads r, g and b.
const material = { color: { r, g, b }, opacity: 1, transparent: false,
                   preserveOriginalMaterial: false } as unknown as FRAGS.MaterialDefinition

await model.highlight(localIds, material)   // one call per colour, never per element
await model.resetHighlight(localIds)        // back to the model's own colours
```

Four things to get right:

- **`preserveOriginalMaterial` must stay `false`.** At `true` fragments skips deduplication and spends one of the model's ~65 500 material slots per element per call. At `false` a colour costs one slot however many elements wear it — so bucket elements by colour and make one call per bucket.
- **Convert sRGB to linear.** `new THREE.Color(hex)`, which is what core passes on its own path, converts into the renderer's working space. Passing raw sRGB values gives visibly different colours from the rest of the app.
- **Show before painting.** A hidden category stays hidden, so paint it without `setItemsVisible(items, true)` and you have coloured something invisible.
- **Paint is a change to the model, not to your component.** It outlives the toolbar panel that applied it — which is the only reason a colour survives the dropdown closing — so do not clear it in a cleanup function, and give the user a way to turn it off.

This bypasses core's own `ElementAppearance`, so plugin paint and the Layers tab's colouring overwrite each other, and CTRL+Z does not undo the plugin's. For a plugin painting a category the sidebar trees rarely touch — spaces, say — that is an acceptable trade; for anything else, prefer being compiled in and using `usePluginBimAppearance`.

## When mounting is the wrong answer

Most of the limits above have a workaround. Two do not, and they are the signal to submit the plugin to core as a pull request instead:

- **The SDK appearance API, and anything else outside the published import list.** The workarounds reach around core rather than through it, and they can conflict with it.
- **Translated interface text.** A mounted plugin's message keys cannot resolve, so it ships in one language.

Everything else on this page a mounted plugin can do for itself today.
