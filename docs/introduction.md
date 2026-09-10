---
sidebar_position: 1
title: Introduction
description: Welcome to CDT — a browser-based digital twin platform that bridges BIM, GIS, and live data on a single interactive map.
---

# Introduction

**Collab Digital Twins (CDT)** is an open-source, browser-based platform for working with buildings, sites, point clouds, datasets, and live sensor data on a single interactive map. It bridges BIM and GIS using open standards, so you can move from a national map view down to a single IFC element without leaving the browser or switching tools.

CDT is used to view federated BIM models, monitor sensor feeds, browse open data, and coordinate work across teams.

## Start here

Three audiences read these docs. Pick the path that matches what you want to do — each section is self-contained and links to the others where relevant.

<div className="audienceGrid">

**[End user / operator →](./getting-started/quickstart.mdx)**
You signed in to CDT and want to view buildings, browse open data, monitor sensors, or collaborate with a team.
*Start with the [Quickstart](./getting-started/quickstart.mdx) — a 30-minute walkthrough from sign-in to your first uploaded model.*

**[Developer / integrator →](./developer-introduction.md)**
You want to extend CDT with a plugin, integrate it with another system, or contribute to the core.
*Start with the [Developer Introduction](./developer-introduction.md), then the [Architecture Overview](./architecture/overview.mdx) and [Plugins](./plugins/overview.md).*

**[Self-hoster / deployer →](./deployment/overview.md)**
You want to run CDT on your own infrastructure or evaluate a hosted deployment.
*Start with the [Deployment Overview](./deployment/overview.md) and the [Environment variables reference](./getting-started/environment-variables.mdx).*

</div>

## What CDT is

CDT is infrastructure for managing the relationship between a physical environment and its digital representation over time. The platform combines:

- **Two integrated viewers** — a MapLibre-powered web map and an IFC BIM viewer (That Open Engine), which also streams point clouds — sharing a single coordinate system so a building uploaded in 3D appears in the right place on the map.
- **Open data integration** — a federated catalogue of national, provincial, and municipal data portals, fetched live and overlaid on the map.
- **Live data feedback loops** — IoT and sensor feeds linked to BIM elements or anywhere in the map, so design models evolve into operational digital twins.
- **Multi-tenant collaboration** — organizations, role-based permissions, threaded comments, and BCF issue tracking.

For background, the [Concepts](./concepts/digital-twins.mdx) section explains digital twins, BIM, GIS, point clouds, and the open standards CDT builds on.

## What CDT is not

CDT is not a BIM authoring tool — it does not replace Revit, Archicad, or Tekla. It is also not a desktop GIS — it does not replace QGIS or ArcGIS Pro. CDT is a federation, visualization, and collaboration layer that sits above those authoring tools and consumes their outputs (IFC, GeoJSON, LAS).

If you are evaluating CDT against alternatives, see [CDT vs alternatives](./cdt-vs-alternatives.md).

## Principles

CDT is built around four principles that shape the platform:

- **Open standards** — interoperability over proprietary lock-in (IFC, GeoJSON, LAS, BCF, IDS, OGC services).
- **Open source** — transparent, community-driven development.
- **Browser-native** — no specialized software to install.
- **Multidisciplinary** — designed for stakeholders across BIM, GIS, planning, and operations.

## Get help

- [Contact form](https://collabdt.org/En/contact/)
- [Core repository on GitHub](https://github.com/CollabDigitalTwins/core)
- [Troubleshooting](./getting-started/troubleshooting.mdx)
