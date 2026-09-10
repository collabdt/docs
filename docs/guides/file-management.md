---
sidebar_position: 5
title: File Management
description: Upload, organize, and manage files attached to buildings and sites.
---

# File Management

CDT stores all binary assets in [MinIO](https://min.io/), a high-performance open-source object store with the S3 API. Metadata and relational attributes live in PostgreSQL.

## Goal

Upload files to a building or site and understand where they go.

## Prerequisites

- A CDT account with **User** or **Admin** role on the target Building or Site.
- Files in one of the supported formats listed below.

## Upload a file

**Goal:** add a file to a Building.

1. Open the building → **Files** tab → **Upload**.
2. Pick one or more files.

The platform detects the file type, routes it to the right pipeline, stores the binary in MinIO, and creates a metadata record.

**Result:** the files appear in the file list and are downloadable, previewable, and (if applicable) loadable in the BIM viewer.

## Supported file types

| Category | Formats |
|----------|---------|
| **BIM models** | IFC |
| **3D geometry** | glTF, GLB, FBX, OBJ, Collada |
| **Point clouds** | LAS, LAZ, COPC (`.copc.laz`), E57 |
| **CAD drawings** | DXF |
| **GIS data** | GeoJSON |
| **Documents** | PDF |
| **Media** | JPG, PNG, MP4, MP3, and other common video/audio formats |

Point clouds are converted to a Potree octree on upload — three files (`metadata.json`,
`octree.bin`, `hierarchy.bin`) that the viewer streams. Those are conversion *output*, not
something you upload.

## What happens to an IFC on upload

When you upload an IFC, the server runs an optimization pipeline before storage:

1. Parses the raw IFC STEP file.
2. Converts geometry and metadata to **Fragments 2.0** (`.frag`) using FlatBuffers encoding.
3. Stores both the original IFC and the `.frag` version in MinIO.
4. Streams the `.frag` to the client at load time.

This dramatically reduces RAM and load time compared with parsing IFC in the browser — important for multi-gigabyte federated models.

## Inspect file metadata

Every file record stores:

| Field | Description |
|-------|-------------|
| Name | Display name. |
| Format | File type. |
| Author | User who uploaded it. |
| Created / Updated | Timestamps. |
| GlobalId | Linked IFC GlobalId (for BIM files). |
| Location | Longitude/latitude or XYZ coordinates. |
| CRS | Coordinate reference system. |
| Building | Parent Building record. |
| Organization | Access-control scope. |

GlobalId linkage means sensor data, BCF topics, IDS results, and media all pin to the same physical asset across different file types.

## Permissions

File visibility and editability follow the Organization's role assignments. A file uploaded by one Organization member is accessible to all members of the same Organization based on their role.

For the full matrix, see [Authorization → Permission reference](../authorization/permission-reference.mdx).

## Storage architecture

The MinIO instance for the hosted CDT runs on Canadian infrastructure (Fullhost VPS in Vancouver and Toronto) for data sovereignty. All storage and processing remain within Canadian boundaries. Network policy restricts database and storage access to the application server's IP only.

Self-hosted deployments inherit the same architecture — see [Self-Hosting](../deployment/self-hosting.md)

## Related

- [Buildings & Sites](./buildings-and-sites.md)
- [BIM Viewer](./bim-viewer.md)
- [Components → File Details](../components/file-details.md)
- [Architecture → Data Layer](../architecture/data-layer.mdx)
