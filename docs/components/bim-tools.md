---
title: BIM Viewer Tools
description: The toolbar tools available in the BIM viewer — clipping, measurement, inspection, model loading, and more.
category: components
status: draft
last_updated: 2026-08-31
---

# BIM Viewer Tools

The BIM viewer toolbar is built from a list of `Tool` objects defined in `useBimToolbarTools()`. Each tool is a React component rendered in a `ToolbarSubmenu` and activated/deactivated via `ToolsContext`.

Source: `@collabdt/core/components/viewers/bim/src/tools/`

## Available Tools

| Tool ID | Component | Description |
|---------|-----------|-------------|
| `bim-clipping` | `ClippingTool` | Add and remove section planes, and a section box, to cut through the model |
| `bim-camera-fit` | `FitCameraTool` | Fit camera to the loaded model |
| `bim-add` | `AddToBim` | Add files, comments, sensors, IFC/BCF/CAD/IDS content |
| `bim-dimensions` | `MeasureBimTool` | Measure length, area, volume and angle in the 3D model |
| `bim-inspect` | `InspectBimTool` | Inspect element properties by clicking |
| `bim-share` | `ShareBimTool` | Share a link to the current camera position |

---

## `ClippingTool`

Adds section planes to the Three.js scene using `@thatopen/components-front`. Each plane is rendered with a `LineMaterial` outline.

`ClippingTool` is the toolbar UI only. The planes themselves are owned by the `ClippingPlanes` component (`tools/ClippingTool/ClippingPlanes.ts`), which wires the `Clipper` and the cut style once per world and keeps an undo history of every plane change.

### Behaviour

- Activates via `ToolsContext` dispatch `SET-TOOL` with `tool.id`.
- Sets the viewer cursor to a crosshair while active, and shows the instructions in a persistent toast.
- On double-click, a clipping plane is added on the front-most face under the cursor. The pick is restricted to model geometry so helper meshes and already-sectioned geometry cannot be picked through.
- **Enter** finishes and keeps the planes. **Escape** clears every plane and exits. **Backspace** or **Delete** removes the plane under the cursor.
- Leaving the mode hides the translucent squares but keeps the arrow gizmos, so an existing section can still be dragged.
- `CTRL+Z` / `CTRL+Y` step through adding, deleting and moving planes. The shortcut is bound in `BimViewer`, which routes it to whichever of the appearance and clipping histories changed last.
- Deactivates when another tool is selected — planes persist until explicitly removed.

### Section box

The same menu offers a **section box**: an axis-aligned box that isolates what is inside it. It is owned by the `ClippingBoxes` component (`tools/ClippingTool/ClippingBoxes.ts`), independent of `ClippingPlanes`.

- **Section box** fits a padded box around everything currently in the scene; the same item clears it.
- Each face carries a grab handle. Dragging one moves that face only, and the box never becomes thinner than 10 cm on any axis.
- Cut geometry is capped in black, the same `ClipStyler` style the section planes use. `ClipStyler.create()` returns edges with `visible === false`, and that setter is what adds the cap to the scene — unlike `createFromClipping()`, it does not show them for you.
- The box is six inward clipping planes appended to `world.renderer.clippingPlanes`. Three.js clipping is an intersection, so this gives keep-inside behaviour with no shader work — and any renderer already subscribed to `onClippingPlanesUpdated`, such as the BIM point clouds, is cut by the box too.
- Planes and the box are independent. Both cuts apply at once, and clearing one leaves the other alone.

### Dependencies

`@thatopen/components` (`OBC`), `@thatopen/components-front` (`OBF`), `three.js`, `sonner`

---

## `AddToBim`

Submenu with sub-tools for attaching content to the BIM model:

| Sub-tool | Description |
|----------|-------------|
| `bim-add-comment` | Pin a comment to a 3D position |
| `bim-add-file` | Attach a file at a 3D position |
| `bim-add-sensor` | Place a sensor in the model |
| `bim-add-ifc` | Load an additional IFC model |
| `bim-add-bcf` | Import a BCF topic file |
| `bim-add-cad` | Import a DXF/CAD file via `AddDxf` |
| `bim-add-ids` | Import an IDS validation file |

Position is set by clicking in the 3D view, captured as `x, y, z` coordinates relative to the model.

---

## `MeasureBimTool`

Measures length, area, volume and angle. The submenu offers five modes, all backed by
`@thatopen/components-front` measurement components rather than hand-rolled raycasting:

| Mode | Component | Interaction |
|------|-----------|-------------|
| Free | `OBF.LengthMeasurement` (`mode: 'free'`) | Double-click two points |
| Edge | `OBF.LengthMeasurement` (`mode: 'edge'`) | Double-click an edge; the measurement spans the whole edge |
| Area | `OBF.AreaMeasurement` (`mode: 'free'`) | Double-click each corner, then `Enter` to close the polygon |
| Volume | `OBF.VolumeMeasurement` | Double-click each element to add it, then `Enter` to total the volume |
| Angle | `OBF.AngleMeasurement` | Double-click three points: start, vertex, end |

`Delete` or `Backspace` removes the measurement under the cursor. `Escape` cancels the
in-progress shape and leaves measurement mode. `Clear` removes every measurement of every kind.

Picking a mode shows its instructions in a persistent toast, replaced when the mode changes and
dismissed when measurement mode ends — the same pattern `ClippingTool` uses.

### `BimMeasurementManager`

All four components are owned by a single `OBC.Component`,
`BimMeasurements/BimMeasurementManager.ts`. Activation is exclusive: each measurement
component binds its own pointer listeners when enabled, so leaving two enabled at once would
make one double-click feed both.

The manager also handles three things the raw components do not:

- **World binding is resolved per activation**, not cached in the constructor. `Components.get()`
  caches, and the toolbar calls it on its first render — which can land before `CurrentWorld.world`
  is published. A constructor snapshot would cache `null` for the rest of the session.
- **Ordering.** `Measurement.enabled`'s setter calls `setEvents()`, which throws when `world` is
  null, so the world and all configuration are applied before `enabled = true`.
- **Hoverer coordination.** The Hoverer is disabled while length, area or angle is active.
  It is deliberately left alone for volume, because `VolumeMeasurement` saves, force-enables and
  restores it itself in order to highlight the items being picked.

Measurements are held in each component's own `list` and persist across tool switches until
`Clear` is used or the viewer is disposed.

### Measuring geometry that is not a fragment

The OBF measurers snap against fragment geometry only. Anything else — a point cloud, today —
offers itself through `registerPickSource(source)`, where a source implements `ScenePickSource`
from `bim/src/lib/scenePicker.ts`:

```ts
interface ScenePickSource {
  pick(ray: THREE.Ray, camera: THREE.Camera, thresholdPx: number): { point: THREE.Vector3 } | null
}
```

The manager tracks the cursor itself and intercepts the measurer's `lastPick` **at assignment
time**, replacing the library's snap with the registered source's hit whenever that hit is nearer.
Ties go to the fragment snap. Call `unregisterPickSource(source)` when the source's scene goes away.

:::note
Subscribing to `onPointerMove` instead does not work, and fails in a way that looks like the tool
is broken rather than merely imprecise. The library registers its own preview handler in the
`Measurement` constructor, so it reads `lastPick` before any later subscriber can change it — and
it reads `null` whenever the cursor is over geometry the fragment picker cannot see. The preview
line's end never moves, and a length measurement commits with both points on top of each other.
Intercepting the write puts every reader in agreement regardless of subscription order.
:::

One consequence to expect: a hit that came from a pick source gets no snap marker, since the
library draws that from its own `_vertexPicker`.

### Snap tuning

The library defaults every measurement component to all three snap classes
(`POINT`, `LINE`, `FACE`) and to `MeasurementPickMode.MOUSE_MOVE`. All three classes compete on
each pick and the nearest wins, so the snap target flips between three different answers as the
cursor moves. The defaults here narrow that:

| Setting | Library default | CDT default |
|---------|-----------------|-------------|
| `snappings` | `[LINE, POINT, FACE]` | One or two classes per mode — `[LINE]` for edge length, `[FACE]` for face area, `[POINT, LINE]` otherwise, `undefined` for volume |
| `pickMode` | `MOUSE_MOVE` | `MOUSE_STOP` — one GPU pick per intentional cursor stop rather than one per animation frame |
| `delay` | 300 ms | 120 ms |
| `pickerSize` | 6 px | 10 px |
| `SnapResolvers` `maxDistance` | 1 m | 0.5 m |
| `stickyRadiusPx` | 12 px | 16 px |

:::note
`Measurement.snapDistance` looks like the snap-range knob but has no effect in
components-front 3.4.3 — its setter writes `GraphicVertexPicker.maxDistance`, which
`GraphicVertexPicker.get()` never reads. The live knob is
`components.get(OBC.SnapResolvers).get().maxDistance`.
:::

Colour, units, decimal places, snap range and marker size are user-editable under
**Measurements** in the BIM sidebar's Settings tab. Values are held on the manager, so they last
as long as the viewer's `Components` instance.

---

## `InspectBimTool`

Activates element inspection mode. On click, highlights the selected element and reads its IFC properties. Supports `line` and `area` inspect types.

### Behaviour

- Sets cursor to a pointer while active.
- Attaches a `click` listener to the viewer container's canvas element.
- Removes the listener on deactivate.

---

## `FitCameraTool`

Fits the Three.js camera to the bounding box of the loaded model. Single-action tool — no persistent active state.

---

## `ShareBimTool`

Generates a shareable URL encoding the current camera position. Reads from `BimContext`.

---

## Activating a tool

Tools are activated and deactivated through `ToolsContext`:

```tsx
const { dispatch } = useContext(ToolsContext)

// Activate
dispatch({ type: 'SET-TOOL', payload: { currentToolId: 'bim-clipping' } })

// Deactivate
dispatch({ type: 'SET-TOOL', payload: { currentToolId: null } })
```

Only one tool is active at a time. When a new tool is activated, the previous tool's component is responsible for cleaning up (removing event listeners, resetting cursor, etc.).

---

## Placement editor

One editor places everything in the BIM scene: fragment models, loaded 3D objects (GLB, DXF) and
point clouds. It is not a toolbar tool — it has no entry of its own and is started from whatever
is being placed.

Source: `@collabdt/core/components/viewers/bim/src/Placement/`

### The scene registry

`SceneObjectRegistry` is the single index of everything the viewer put in the BIM scene that is not
a fragment model: loaded 3D objects, DXF drawings and the pins standing in for other file types.
Entries are keyed by **file id**, and the scene object is named `file:<id>`. Reach it with
`components.get(BimSceneObjects).registry`; it is `null` only before the world exists.

The sidebar, the viewport menu and the placement editor all resolve a file through
`sceneObjectForFile(file, registry)`, so none of them can disagree about what a file is. Objects
placed before their upload finishes are held under a temporary key with `fileId: null` — they draw
no marker and the viewport menu skips them — until `rekey(tempKey, fileId)` hands them over.

Because the registry is the one index, the list and the scene stay in step in both directions:
deleting a row removes the object, and placing a file marks its row visible.

| Method | Use |
|--------|-----|
| `add({ key, fileId, kind, root, dispose })` | Register a loaded object; replaces and disposes any entry under that key |
| `get` / `has` / `list` | Address an object by file id |
| `rekey(key, fileId)` | Hand a just-uploaded placement to its file record; fires `onAdded` |
| `setVisible(key, visible)` | Back the row's visibility toggle |
| `remove(key)` | Detach and dispose; fires `onRemoved` |
| `onAdded` / `onRemoved` | Subscribe; each returns its own unsubscribe |

### Entry points

- **Sidebar row menu** — the `move` action on a Files, Models or Point Clouds row.
- **Right-click in the viewport** — `useViewportContextMenu` picks whatever is under the cursor,
  nearest hit wins, and opens the same card of actions. Right-clicking empty space opens nothing.

The hook owns the right button, not the `contextmenu` event: `camera-controls` trucks the camera
with the same button, so the menu opens on release only if the press never moved past
`DRAG_SLOP_PX` (5px). Duration is ignored — a pan always moves, and a slow click that opened
nothing would read as a bug.

The native menu is suppressed by a **capture-phase listener on `window`**, gated on
`withinViewport(canvas.getBoundingClientRect(), …)` rather than on the event target. A listener
bound to the canvas is not enough: CSS2D marker overlays sit above it without being its
descendants and default to `pointer-events: auto`, so a right-click on one reaches neither our
handler nor `camera-controls`' own — which is how the native menu used to appear alongside the
card. Gating on position covers every overlay at once instead of patching each marker type.

:::note
Suppression is unconditional inside the viewport, not conditional on hitting geometry. The pick is
async and `contextmenu` is synchronous, so what was hit is not yet known when the decision has to
be made — and a viewport whose right button pans has no use for the native menu anywhere.
:::

Both resolve through the registry, so a DXF drawing and a GLB behave identically. The card's
`delete` action removes the file and its scene object together, after a confirmation.

:::warning
`pickSceneObject` calls `updateMatrixWorld(true)` before raycasting. `Raycaster.intersectObject()`
does not do it, and the BIM renderer draws on demand — so an object added or moved with no render
since is otherwise tested against a stale, usually identity, matrix and the click misses it.
:::

:::note
An object placed before its upload finishes is held under a temporary key. If the save fails there
is no record to key it to, so the placement is discarded rather than left in the scene as something
neither the menu nor the sidebar can address.
:::

Only one placement session is live at a time; starting a second one commits the first. The editor
claims the exclusive viewer slot through `ViewModeCoordinator`, so another tool taking the viewer
ends the session and keeps the edit.

### What each kind can save

`DbFile` carries `x`, `y`, `z` and `bimRotation` (yaw, radians). Point clouds store a full
transform in their `pointCloudTransform` JSON. `capabilitiesForFile()` decides what a given file
may change, in one place, so the card, the viewport menu and the adapters cannot disagree.

| Target | Rotation | Scale | Stored as |
|--------|----------|-------|-----------|
| Point cloud (`laz`, `las`) | Three axes | Yes | `pointCloudTransform` JSON |
| 3D object (`glb`, `gltf`, `fbx`, `obj`, …) | Yaw only | Yes | `x`, `y`, `z`, `bimRotation` |
| DXF | Yaw only | Yes | `x`, `y`, `z`, `bimRotation` |
| BIM model (`frag`, `ifc`) | Yaw only | No | `x`, `y`, `z`, `bimRotation` |

Scaling is always **proportional — one number, never per axis**. The gizmo writes only the axis
being dragged, so `uniformScale()` resolves the three components back to the single value the drag
meant and the object is re-scaled uniformly.

A DXF's scale doubles as its drawing-unit conversion (`0.001` for millimetres → metres), so the
stored value is absolute rather than a multiplier, matching `Position3DCard`.

:::warning
There is no `scale` column yet, so an object's or DXF's scale applies live but does not survive a
reload. The ask is recorded in `knowledge/Plans/2026-08-27-pointclouds-in-bim-viewer.md`; once the
column exists it is one line in `objectTarget.commit`, plus the adapter, the type and the load path.
:::

### Animation

A loaded model that carries animation clips gets a fourth action on the placement card,
**Animation**. Nothing else does: clips are a fact about the loaded file, which no extension can
predict, so `markerActionsFor(capabilities, { animated })` takes it as a second argument rather
than as a placement capability.

The panel offers a clip dropdown (shown only when the file has more than one), play/pause, and a
speed slider from 0.1x to 3x. Clips autoplay at 1x on every load path; one clip runs at a time, so
a file offering alternatives does not blend them.

:::warning
Playback is **session-only** and is never persisted — there is no column for it and none is planned.
A reload restores the first clip at 1x.
:::

:::note
The BIM renderer draws on demand. `ModelManager` sets `renderer.needsUpdate` only while a mixer is
actually playing, so a paused or clipless scene stays idle. A mixer updated without that flag
advances the animation but renders nothing.
:::

### The port

A target is the only code that knows how its kind reads and stores a transform:

```ts
interface PlacementTarget {
  id: string
  name: string
  capabilities: { rotation: 'yaw' | 'full'; scale: boolean }
  object(): THREE.Object3D | null
  read(): PointCloudPlacement
  apply(placement: PointCloudPlacement): void
  bounds(): THREE.Vector3 | null
  commit(placement: PointCloudPlacement): Promise<void>
}
```

`narrowPlacement(placement, capabilities)` strips whatever the target cannot hold. It runs on
`apply` as well as `commit`, so the live preview shows what will actually be saved rather than
snapping back afterwards.

### Pivot and centring

Rotation and scale otherwise turn about the target's own origin, which on a georeferenced scan sits
far outside the points. **Pick centre point** waits for a double-click and turns about that instead;
**Centre on scene origin** moves the target so its own centre sits on the world origin. Both are
available for every target kind and in every mode.

:::note
`GizmoController.attach()` resets the gizmo to translate mode, so the editor always attaches first
and applies the mode afterwards. Setting the mode before attaching silently loses it.
:::
## Key Files

| File | Role |
|------|------|
| `@collabdt/core/components/viewers/bim/src/tools/bimToolbar.ts` | Tool list definition |
| `@collabdt/core/components/viewers/bim/src/tools/ClippingTool/ClippingTool.tsx` | Clipping plane and section box tool |
| `@collabdt/core/components/viewers/bim/src/tools/AddToBim/index.tsx` | Add content sub-menu |
| `@collabdt/core/components/viewers/bim/src/tools/InspectBimTool.tsx` | Element inspection |
| `@collabdt/core/components/viewers/bim/src/tools/measureBimTool.tsx` | Measurement submenu and hint card |
| `@collabdt/core/components/viewers/bim/src/BimMeasurements/BimMeasurementManager.ts` | Owns the four measurement components; exclusive activation, world binding, event wiring |
| `@collabdt/core/components/viewers/bim/src/BimMeasurements/measurementSettings.ts` | Snap tuning, units and the per-mode snap-class table |
| `@collabdt/core/components/viewers/bim/src/BimSidebar/src/SettingsTab/src/MeasurementSettings.tsx` | Colour, units, precision, snap range and marker size controls |
| `@collabdt/core/components/viewers/bim/src/tools/FitCameraTool.tsx` | Fit camera |
| `@collabdt/core/components/viewers/bim/src/Placement/PlacementEditor.ts` | One placement session: gizmo, pivot, exclusivity, commit |
| `@collabdt/core/components/viewers/bim/src/Placement/placementTarget.ts` | The `PlacementTarget` port and capability narrowing |
| `@collabdt/core/components/viewers/bim/src/Placement/PlacementPanel.tsx` | The numeric card, gated by capabilities |
| `@collabdt/core/components/viewers/bim/src/Placement/useViewportContextMenu.ts` | Sole owner of the canvas `contextmenu` event |
| `@collabdt/core/components/viewers/bim/src/SceneObjects/sceneObjectRegistry.ts` | The one index of scene objects, keyed by file id |
| `@collabdt/core/components/viewers/bim/src/Placement/pickSceneObject.ts` | Nearest registered object under the ray, models and drawings alike |
| `@collabdt/core/components/viewers/bim/src/Placement/contextMenuGesture.ts` | Click-versus-pan decision for the right button |
| `@collabdt/core/components/viewers/bim/src/SceneObjects/index.ts` | `BimSceneObjects`, the component wrapper around the registry |
| `@collabdt/core/components/viewers/bim/src/lib/sceneContent.ts` | `sceneObjectForFile` / `isFileInScene` |
| `@collabdt/core/components/viewers/bim/src/ModelManager/modelAnimation.ts` | Clip, speed and play state; decides when frames are needed |
| `@collabdt/core/components/viewers/bim/src/Placement/AnimationPanel.tsx` | Clip, play/pause and speed controls |
| `@collabdt/core/components/viewers/bim/src/Placement/AnimationSession.ts` | Which model's playback panel is open |
| `@collabdt/core/components/viewers/bim/src/tools/shareBimTool.tsx` | Share camera position |

## Permissions

<!-- TODO: Confirm which tools are gated by CASL permissions (e.g., adding content likely requires create permissions on File/Comment/Sensor). -->

## Related

- [State Management](../architecture/state-management.mdx) — `ToolsContext`, `BimContext`
- [Components — Toolbar](./toolbar.md)
- [Concepts — BIM and IFC](../concepts/bim-and-ifc.mdx)
- [Guides — BIM Viewer](../guides/bim-viewer.md)
