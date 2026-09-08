---
sidebar_position: 2
title: BIM Viewer
description: Open IFC models, navigate them in 3D, inspect properties, validate against IDS, and coordinate with BCF topics.
---

import BrowserOnly from '@docusaurus/BrowserOnly';

# BIM Viewer

The BIM viewer loads, navigates, and interrogates IFC models independently of their map context. It is built on [That Open Engine](https://thatopen.com/) — an IFC engine on top of Three.js that gives full access to BIM geometry, metadata, and property sets while honouring openBIM standards.

## Goal

Load a BIM model, navigate it in 3D, read its property sets, and use the openBIM tools (IDS validation, BCF topics) for coordination work.

## Prerequisites

- A CDT account with **User** or **Admin** role (to upload).
- An IFC file. Public test files are at the [buildingSMART sample repository](https://github.com/buildingSMART/Sample-Test-Files).

## Load a model

**Goal:** get an IFC into the viewer.

1. Open the BIM Viewer from the left sidebar.
2. Drag the IFC file onto the viewer, or use the **File** tab → **Upload** button.
3. The platform converts the file to **Fragments 2.0** (`.frag`) on the server. The progress bar tracks parse → conversion → load.
4. Once finished, the model appears in the scene and is added to the **File** tab list.

The conversion happens once. Subsequent loads stream the cached `.frag` and are much faster.

**Supported formats** beyond IFC: glTF / GLB, FBX, OBJ, Collada, DXF, LAZ / LAS / COPC.

**Result:** the model is visible in the viewport and selectable.

## Navigate in 3D

| Control | What it does |
|---------|--------------|
| **Left-click + drag** | Orbit around the model |
| **Right-click + drag** | Pan |
| **Scroll wheel** | Zoom |
| **Double-click** | Set the orbit pivot point |
| **Fit Extents** (toolbar) | Frame the full model in the viewport |

**Result:** smooth navigation in any direction, with predictable pivot behaviour.

## Inspect element properties

**Goal:** read the IFC schema data for a specific element.

1. Click any element in the 3D scene.
2. The right panel populates with:
   - Entity attributes and geometry
   - Property sets (Psets)
   - Quantity sets (Qsets)
   - Material assignments
   - Associated `IfcTask` entries
   - Spatial container relationships
3. Use the search field to filter large property trees.
4. Hold **Shift** and click multiple elements to compare attributes side-by-side.
5. Click **Export** to save the current selection's properties as JSON or CSV.

**Result:** the property panel shows the full IFC data for the selected element(s).

## Cut a section

**Goal:** see inside the model with a clipping plane.

1. Click **Clipping plane** in the toolbar, then **Add clipping plane**.
2. Double-click any face on the model — a section plane appears aligned with that face.
3. Drag the arrow handle to adjust depth.
4. Press **Enter** (or **Cancel add clipping plane**) to finish. The section stays and its arrow is still draggable.
5. **Backspace** removes the plane under the cursor, **Esc** removes every plane, and **Ctrl+Z** undoes the last change.

**Result:** the model is sliced at your chosen plane and the interior is visible.

## Browse and filter the model

**Goal:** navigate to elements via a list rather than hunting in 3D, and control what is visible.

Open the **Layers** tab in the left panel. It holds two collapsible groups, each with a switcher between two views:

| Group | Views | What it lists |
|-------|-------|---------------|
| **Drawings** | Floorplans / Elevations | Generated 2D views of the model |
| **Classifier** | Spatial / IFC Classes | The IFC spatial tree, or every IFC class in the model |

Collapsing one group gives its height to the other, and the divider between them can be dragged.

### Drawings

Selecting a storey generates its floorplan and frames it in the view. Below the drawing, a **Layers** list gives every IFC class in the plan its own visibility switch and colour picker, so you can mute furniture, recolour the structure, and so on.

#### Rooms

Rooms (`IFCSPACE`) get their own **Rooms** layer, drawn the way most authoring software does: a translucent light-blue fill over the room's footprint, an X across it, and the room's name at its centre.

The layer starts **off**, because room fills sit over the linework underneath. Switch it on from the Layers list.

Picking a colour for the layer replaces the light blue and **hides the X** — the cross reads as "unstyled room", so it stops making sense once a room carries a deliberate colour. The name tags stay either way, since they are information rather than styling.

The fill follows the room's real footprint, so an L-shaped room is drawn as an L rather than as its bounding rectangle.

### Spatial

The spatial view is the IFC containment hierarchy, rooted at each model's building:

<BrowserOnly>
  {() => {
    const HierarchyTree = require('@site/src/components/HierarchyTree').default;
    return (
      <HierarchyTree
        data={{
          label: 'IfcBuilding',
          children: [{
            label: 'IfcBuildingStorey',
            children: [
              { label: 'IfcSpace' },
              { label: 'IfcWall / IfcSlab / …' },
            ],
          }],
        }}
      />
    );
  }}
</BrowserOnly>

Rows show each element's **name** (`Basic Wall:Generic 200mm`) with its IFC class beside it. Every loaded model contributes its own building, so a federated set appears as sibling branches in one tree.

### IFC Classes

The classes view groups the same elements by what they *are* rather than where they sit — one row per IFC class (`IFCWALL`, `IFCSLAB`, `IFCDOOR`, …) with the number of elements in it. Class names are shown exactly as the IFC defines them, in every language. Only classes that have geometry are listed.

:::note Classes hidden by default

Three classes start hidden, because their geometry envelops the building and would obscure everything behind it:

| Class | Why |
|-------|-----|
| `IFCGEOGRAPHICELEMENT` | Topography in IFC4 |
| `IFCSITE` | Topography in IFC2x3, where the terrain hangs off the site |
| `IFCSPACE` | Room volumes |

Which class topography lands in depends on the schema version and the authoring software's IFC export mapping — some export topography geometry as `IfcSite`, others as `IfcGeographicElement` — so both cases are covered. They are still listed — switch one on to show it. The choice sticks for the rest of the session; a model loaded afterwards comes in with its own topography and spaces hidden.

`IFCBUILDINGELEMENTPROXY` is **not** hidden by default. Some IFC2x3 exports put terrain there too, but it is also the catch-all for any element the authoring software has no specific IFC class for, so hiding it would take real building geometry with it. Switch it off manually if your export needs it.

:::

### Select, hide, isolate

Every row in both views supports the same three actions:

| Action | How | Effect |
|--------|-----|--------|
| **Select** | Click the row | Highlights the element(s) in 3D and opens the properties panel |
| **Hide / show** | The row's switch | Toggles visibility of the row and everything under it |
| **Isolate** | The crosshair button (appears on hover) | Hides everything else |
| **Recolour / fade** | The small circle at the left of the row | Sets a colour and an opacity for the row and everything under it |

These act across **all loaded models**, so isolating `IFCWALL` leaves only walls in a federated set. Use the eye button in the view's toolbar — or **Selection → Show all** in the viewer toolbar — to bring everything back.

The search box at the top of the tab filters both groups at once and opens the branches leading to each match.

### Colour and opacity

Every row carries a small circle at its left. It stays an empty outline until you use it, then fills with whatever colour and opacity you gave that row.

Click it to open the picker: a colour well and an opacity slider. Both are optional and independent, so you can tint a class without fading it, or fade a storey while it keeps its own colours. **Reset** in the picker clears just that row.

Colouring cascades. Tint a storey in the **Spatial** view and everything inside it takes the colour; a wall you had already coloured separately keeps its own, whichever order you set the two in. Setting only an opacity on that wall leaves it the storey's colour and fades it.

The two views can name the same element, since `IFCWALL` and a storey overlap. Whichever view you touched most recently wins the overlap, and undoing that change hands the elements back to the other view.

| Control | Where | Effect |
|---------|-------|--------|
| **Reset** | Inside a row's picker | Clears that row |
| **Reset colours** | The brush button in the view's toolbar | Clears the whole view, leaving the other one alone |
| **Undo** | `Ctrl+Z` (`Cmd+Z` on macOS) | Steps back one colour or opacity change |
| **Redo** | `Ctrl+Y`, or `Ctrl+Shift+Z` | Puts the change you just undid back |

Undo and redo cover colour and opacity only, not visibility or selection, and they work anywhere in the viewer rather than only while the Layers tab is open. Inside the search box `Ctrl+Z` edits the text as usual.

A whole slider drag counts as one step, so undoing a fade takes one keystroke rather than one per tick, and redoing it lands where you let go. Making a new change after undoing drops the redo trail, as everywhere else.

Colours last for the session. They are not saved with the model, and they are not shared with anyone else looking at it.

**Result:** you can find any element by location or by class, reduce the scene to just what you are working on, and colour-code what is left.

## Validate against an IDS file

**Goal:** check whether the model satisfies an Information Delivery Specification.

1. Open the **File** tab.
2. Click **Import IDS** and pick your `.ids` file.
3. The viewer evaluates each requirement against the model and lists pass/fail counts.
4. Click any failed requirement to see the offending elements highlighted in 3D.

**Result:** every requirement in the IDS shows pass/fail with element-level traceability.

## Track issues with BCF topics

**Goal:** open a coordination issue against a specific element and a specific viewpoint.

1. Click the element(s) related to the issue.
2. Open the **Topics** tab → **New topic**.
3. Fill in title, description, responsible party, status, and priority. The current viewpoint is captured automatically.
4. Save the topic.
5. Export to `.bcf` from the topic list to share with teams using Revit, Archicad, or any compliant authoring tool.

**Result:** the topic is saved against the element's `GlobalId` and viewpoint, and exports as a vendor-neutral `.bcf`.

## Generate a floor plan

**Goal:** view a 2D plan of any storey.

1. Open the **File** tab.
2. Floor plans are auto-generated for every `IfcStorey` — pick one from the list.
3. The viewer switches to a top-down 2D plan with the same measurement and annotation tools as the 3D view.

**Result:** a precise 2D plan of the chosen storey, navigable like the 3D view.

## Toolbar quick reference

| Tool | Description |
|------|-------------|
| **Clipping plane** | Section cut from any face. |
| **Fit extents** | Reset the camera to frame the model. |
| **Add feature** | Import IFC, IDS, or BCF; upload DXF; add media. |
| **Measurements** | Distance, angle, area, and element volume. |
| **Share** | URL + QR code encoding camera position and asset ID. |

## Settings reference

The **Settings** tab in the left panel controls the Three.js scene:

- **Theme** — system, dark, light.
- **Camera** — perspective vs orthographic; FOV, speed, frustum.
- **Grid** — toggle, resize, recolour.
- **Lighting** — position, intensity, colour.
- **Renderer** — gamma correction, ambient occlusion, gloss, outline effects.

## Placing and editing scene content

Right-click anything you have added to the scene — a 3D object, a DXF drawing, a point cloud — to
open its card: **Move**, **Rotate**, **Scale** and **Delete**. Holding the right button to pan the
camera does not open it; only a right-click that stays put does. The same card is on the pin that
floats above small objects, and the `move` action on a Files row opens it too.

The file list and the scene stay in step. Delete a row and the object leaves the scene; place a
file and its row is marked visible. Deleting from the viewport card asks for confirmation first.

## Animated 3D models

A GLB or glTF containing animation clips gets a fourth action on that card, **Animation**: pick a
clip, play or pause, and set speed from 0.1x to 3x. Clips play automatically when the model loads.
Models without clips do not show the action.

Playback settings are not saved — reopening the model starts its first clip at normal speed.

## DXF / CAD overlay

Upload a `.dxf` to overlay a 2D drawing inside the 3D scene. The platform parses it through [DXF-Viewer](https://github.com/vagran/dxf-viewer) into Three.js lines. You can position, scale, and rotate it relative to the IFC coordinate system, and toggle CAD layers individually.

## Supported openBIM formats

| Format | Purpose |
|--------|---------|
| **IFC** | Building model geometry + metadata |
| **IDS** | Information delivery requirements + validation |
| **BCF** | Issue tracking and coordination |
| **bSDD** | buildingSMART Data Dictionary — element classification |

## Related

- [Concepts → BIM & IFC](../concepts/bim-and-ifc.mdx)
- [File Management](./file-management.md)
- [Collaboration → BCF Topics](./collaboration.md)
- [Components → BIM Tools](../components/bim-tools.md)
