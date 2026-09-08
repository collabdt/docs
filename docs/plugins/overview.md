---
title: Overview
description: What a CDT plugin is, where one can appear in the app, and the two ways to get one running.
sidebar_position: 1
category: plugins
status: draft
last_updated: 2026-08-17
---

import BrowserOnly from '@docusaurus/BrowserOnly';

# Plugins

A plugin adds functionalities to the CDT platform without changing CDT core. A plugin is a folder holding a manifest and some React components; the app picks it up and renders its contributions alongside the core.

A plugin declares what it adds, and CDT calls its `activate()` function once at start-up. Everything else — the toolbar button, the panel frame, the dialog overlay, the page layout — is handled by the app. The plugin supplies the contents.

<BrowserOnly>
  {() => {
    const PluginSurfaces = require('@site/src/components/PluginSurfaces').default;
    return <PluginSurfaces />;
  }}
</BrowserOnly>

A single plugin can use as many of these as it needs — `hello-map` ships with CDT and uses six. See [Capabilities](./all-capabilities.md) for what each one receives.

## Getting a plugin running

Two conditions must be met, and they are separate on purpose.

**1. The code has to be present.** Either a built plugin folder sits next to a CDT deployment, or the plugin is compiled into `@collabdt/core` and ships with every installation.

|  | Mounted next to a deployment | Compiled into core |
|---|---|---|
| Intended for | Anyone self-hosting CDT | Plugins useful to every installation |
| How it gets in | Build it, add the folder, restart | A pull request to core, then a release |
| Available on the CDT-hosted platform | No | Yes |

Mounting is the place to start. A plugin that turns out to be broadly useful can then be submitted to core as a pull request.

**2. The plugin has to be enabled.** An administrator adds it to their organization on the **Plugins** page, and each person then chooses whether it runs for them. Nothing runs merely because the code is present.

## Security

A plugin runs with the same access as CDT itself. There is no sandbox: it can read anything the signed-in person can read and call any endpoint they can call.

Treat a plugin as any other dependency being granted full access, and read it before mounting it. This is why plugin loading is off unless explicitly enabled, and why an administrator must still add a plugin before it runs.

## In this section

1. [Create your first plugin](./create-your-first-plugin.md) — a working plugin, start to finish
2. [Capabilities](./all-capabilities.md) — everything a plugin can add, and what each contribution receives
3. [Run your plugin](./mounting-a-plugin.md) — building, loading and enabling one
4. [Example: one plugin, several surfaces](./hello-map-example.md) — how the surfaces work together
5. [Example: a mounted plugin against a real model](./mounted-plugin-example.md) — what a runtime-loaded plugin can and cannot reach
6. [Building a plugin with AI](./building-a-plugin-with-ai.md) — a prompt, and the mistakes to expect
