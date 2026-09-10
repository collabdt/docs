---
sidebar_position: 3
title: Point Clouds in the BIM Viewer
description: Upload a LAS, LAZ or E57 scan, convert it to a streaming octree, and place it alongside your IFC models.
---

# Point Clouds in the BIM Viewer

Point clouds load in the **BIM viewer**, in the same scene as your IFC models. There is no
separate point cloud viewer — the standalone one was retired once the BIM viewer could stream
clouds itself, so a scan and a model now share one camera, one clipping set and one
measurement tool.

## Goal

Upload a scan, watch it convert, place it against a model, and tune how it draws.

## Prerequisites

- A CDT account with access to a building.
- A scan in **LAS**, **LAZ** or **E57** format.
- A reachable conversion service (`POINTCLOUD_API_URL`). Self-hosters: see
  [Deployment → Services](../deployment/services.md).

## Upload a scan

**Goal:** get a cloud into the building.

1. Open the **BIM viewer** and select the building.
2. In the sidebar's **File** tab, find the **Point Clouds** section and use the **+** button.
3. Pick your file. Two progress bars follow in turn:
   - **Uploading** — the file is going to object storage.
   - **Converting** — the service is building the Potree octree the viewer streams.
4. When conversion finishes the row becomes viewable. No page reload is needed.

**Result:** the scan is listed and can be switched on.

A scan is only renderable once conversion succeeds; until then the row is listed but offers no
**view** action. Two recovery actions appear when something goes wrong:

| Icon | When it appears | What it does |
|------|-----------------|--------------|
| Refresh | The file uploaded but conversion did not finish | Re-runs the conversion |
| Upload | The upload itself died, leaving a row with no object behind it | Removes the row and lets you re-pick the file |

## Place a cloud against a model

**Goal:** line the scan up with the design model.

1. On the cloud's row, choose **Move**.
2. Drag the pivot, or type exact offsets and rotations in the placement panel.
3. The placement is saved to the file record, so it survives a reload.

**Result:** the cloud sits where the model does.

## Tune how points draw

Sidebar → **Settings** → the point cloud block. It applies to every cloud in the scene except
opacity, which is per cloud.

| Control | What it does |
|---------|--------------|
| **Point budget** | How many points may be on screen at once. The main performance lever |
| **Point size** | Base size in world units |
| **Max size** | Upper bound in pixels, so near points do not become blobs |
| **Size type** | `fixed`, `attenuated` (shrinks with distance) or `adaptive` |
| **Shape** | `square`, `circle` or `paraboloid` |
| **Opacity** | Per cloud. **Ghost** on the file row is the same value |
| **Show octree boxes** | Draws the streaming node boxes — a diagnostic for streaming, not a display mode |

Colour comes from the scan itself. Elevation, intensity and classification colour modes are
not currently exposed.

## Navigate

Sidebar → **Settings** → **Camera Settings**.

- **Orbit** — left-drag to orbit, right-drag to pan, scroll to zoom.
- **Walk** — first person. Move with **WASD** or the arrow keys; **Q** and **E** move down and
  up. Set a **move speed**, and **lock elevation at** a height in metres to hold a constant eye
  level while walking. Walk mode needs a perspective projection; it is unavailable under
  orthographic.

## Measure and section

Measurement and clipping operate on cloud geometry as well as model geometry, so you can
measure from a scanned surface to a modelled element, and section through both at once.

## Supported formats

| Format | Notes |
|--------|-------|
| **LAS** | Standard uncompressed format. Read directly |
| **LAZ** | Compressed LAS. Read directly, and the smaller upload |
| **E57** | Common scanner exchange format. Transcoded to LAZ server-side before conversion |

Every format is converted server-side into a Potree octree; the viewer streams that octree
rather than the original file. Conversion time scales with point count, so a large scan takes a
while — the progress bar reports the converter's own percentage.

## Troubleshooting

**The scan uploaded but never appears in the Point Clouds section.** The file record needs a
`point-cloud-file` type or a recognised extension. If neither is set it lands in the generic
**Files** list instead.

**Conversion finishes but nothing renders.** Check that the conversion service can reach object
storage, and that the octree's `metadata.json` was written.

For more, see [Troubleshooting → Viewers](../getting-started/troubleshooting.mdx#viewers).

## Related

- [Concepts → Point Clouds](../concepts/point-clouds.mdx)
- [BIM Viewer](./bim-viewer.md)
- [File Management](./file-management.md)
