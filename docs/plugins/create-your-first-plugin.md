---
title: Create your first plugin
description: Scaffold a working CDT plugin, understand the manifest and the entry point, and get it into the app.
sidebar_position: 2
category: plugins
status: draft
last_updated: 2026-08-17
---

# Create your first plugin

This walkthrough produces a button in the map toolbar that shows where the map is centred, translated into three languages.

## 1. Scaffold it

```bash
npx create-cdt-plugin
```

The command asks for a name and which surfaces to contribute to, then writes a folder that builds and runs. The surface question is a multi-select, so a plugin that spans several is one answer rather than a later rewrite — pick just *Map toolbar* to follow along.

Scripting it instead of answering prompts? `--surface` is repeatable and takes a comma-separated list: `--surface bim.tools,viewer.tabs`.

```
map-centre/
  manifest.json
  package.json
  tsup.config.ts
  src/
    index.ts
    components/
      ExampleMap.tsx
```

The files can be written by hand, but the build configuration is best left to the scaffold. A misconfigured build is the most common reason a plugin fails to load, and the preset used here fails the build rather than producing something broken.

## 2. The manifest

The manifest holds everything CDT needs before running any plugin code, translations included, so a small plugin is one file to write and one file to hand a translator.

```json
{
  "slug": "map-centre",
  "name": "Map Centre",
  "version": "0.1.0",
  "hostApi": 1,
  "description": "Shows where the map is currently centred.",
  "author": "Your name",
  "icon": "MapPin",
  "capabilities": ["map.tools"],
  "requiredPermissions": [],
  "configSchema": {
    "type": "object",
    "properties": {
      "decimals": { "type": "number", "default": 5 }
    }
  },
  "messages": {
    "en": { "title": "Map Centre", "latitude": "Latitude", "longitude": "Longitude" },
    "fr": { "title": "Centre de la carte", "latitude": "Latitude", "longitude": "Longitude" },
    "es": { "title": "Centro del mapa", "latitude": "Latitud", "longitude": "Longitud" }
  }
}
```

| Field | Purpose |
|---|---|
| `slug` | The plugin's namespace. It must match the folder name. Settings, translations and stored data are all keyed by it. |
| `hostApi` | The version of the plugin API the plugin was built against. CDT refuses to load a mismatch rather than failing later in a less obvious place. Use `1`. |
| `capabilities` | What the plugin may register. Registering an undeclared capability throws. |
| `requiredPermissions` | Shown to an administrator before the plugin is added. List these accurately. |
| `configSchema` | Settings an administrator can change without touching code. |
| `icon` | Optional. A [lucide](https://lucide.dev/icons/) icon name, shown beside the plugin on the Plugins page. A name that is not an icon, or none at all, shows a puzzle piece. |
| `messages` | Optional. Merged under `plugins.<slug>`, so keys cannot collide with those of core or another plugin. |

## 3. The entry point

CDT calls `activate()` once, with a context bound to the plugin. Each contribution is added with `ctx.register()`.

```ts
import { MapCentreTool } from './components/MapCentreTool'

import type { MapPluginContext } from '@collabdt/plugin-kit/types/map'

export function activate(ctx: MapPluginContext): void {
  ctx.register('map.tools', {
    id: 'map-centre',
    label: 'Map Centre',
    icon: 'MapPin',
    component: MapCentreTool,
    stayActive: true,
  })
}
```

The context has three members:

| Member | Description |
|---|---|
| `ctx.pluginId` | The slug declared in the manifest. |
| `ctx.config` | The settings an administrator has saved, shaped by `configSchema`. Empty if none are declared. |
| `ctx.register(key, item)` | The only way to add a contribution. `key` must be a declared capability; the shape of `item` follows from the key, and TypeScript checks it. |

The context type is named after the surface — `MapPluginContext` here, `BimPluginContext`, `UiPluginContext` and so on. That binds `register` to the right shapes, so passing a component that expects the BIM viewer to `map.tools` is a compile error rather than a plugin that loads and displays nothing.

Each of those aliases binds **one** viewer. A plugin that touches two names the slots itself, which is what the scaffolder writes when you pick surfaces in more than one viewer:

```ts
import type { PluginContext } from '@collabdt/plugin-kit/types/ui'
import type { MapToolProps } from '@collabdt/plugin-kit/types/map'
import type { BimToolProps } from '@collabdt/plugin-kit/types/bim'

type Ctx = PluginContext<MapToolProps, BimToolProps>
```

The order is map, BIM, legend, and trailing slots you do not use can be left off. Reaching for `MapPluginContext & BimPluginContext` instead does not work: the second viewer's component ends up checked against a registration bound to `unknown`.

A plugin that sets up timers or listeners outside React should also export `deactivate(ctx)`. CDT calls it when the plugin is switched off, then removes the contributions automatically.

```ts
export function deactivate(ctx: MapPluginContext): void {
  // clear intervals, remove listeners
}
```

## 4. The component

The component supplies the **panel content**. CDT wraps it in the standard toolbar button and dropdown, using the `label` and `icon` from the registration.

```tsx
'use client'

import { Separator } from '@collabdt/core/plugins-sdk/components'
import { usePluginConfig } from '@collabdt/core/plugins-sdk/config'
import { usePluginTranslations } from '@collabdt/core/plugins-sdk/messages'
import { useEffect, useState } from 'react'

import type { MapToolProps, ToolbarToolProps } from '@collabdt/plugin-kit/types/map'

export function MapCentreTool({ map }: ToolbarToolProps & MapToolProps) {
  const t = usePluginTranslations()
  const { decimals = 5 } = usePluginConfig<{ decimals?: number }>()
  const [centre, setCentre] = useState<{ lat: number; lng: number } | null>(null)

  useEffect(() => {
    if (!map) return

    const read = () => {
      const { lat, lng } = map.getCenter()
      setCentre({ lat, lng })
    }

    read()
    map.on('move', read)
    return () => { map.off('move', read) }
  }, [map])

  return (
    <div className="w-60 p-1">
      <p className="px-2 py-1 text-sm font-medium">{t('title', 'Map Centre')}</p>
      <Separator className="my-1" />
      {centre ? (
        <p className="px-2 py-1 text-sm tabular-nums">
          {centre.lat.toFixed(decimals)}, {centre.lng.toFixed(decimals)}
        </p>
      ) : (
        <p className="px-2 py-1 text-sm text-muted-foreground">
          {t('waiting', 'Waiting for the map…')}
        </p>
      )}
    </div>
  )
}
```

Three practices to follow:

- **Guard the viewer handle.** `map` is null until the viewer has initialised, so check it rather than asserting it.
- **Remove every listener on cleanup.** A leaked `move` handler keeps firing after the viewer changes.
- **Pass a fallback to `t()`.** The second argument is used when a key has no translation, so the plugin still reads correctly in an untranslated language.

## 5. Run it

```bash
npm install
npm run build
```

Then load the plugin into CDT and enable it — see [Run your plugin](./mounting-a-plugin.md).

## The rules

- **Import only from `@collabdt/core/plugins-sdk/*`, `@collabdt/plugin-kit/types/*`, `react` and the plugin's own files.** The build fails on anything else, and names what it rejected.
- **`ctx.register()` is the only way to add a contribution,** and only during `activate`.
- **Use the components from `@collabdt/core/plugins-sdk/components`.** They are the approved set, and they match the rest of the app.

:::note Contributing a plugin to core instead
A plugin can also live inside `@collabdt/core`, at `src/core/plugins/<slug>/`, which places it in every CDT installation, including the hosted platform. That route means a pull request against core and waiting for a release, so it is worth building and testing the plugin as a mounted one first.

The source is nearly identical. A plugin inside core imports from `../../sdk/*` rather than from the two packages above, is listed in `manifests.ts`, and is paired with a dynamic import in `installed.ts`:

```ts
export const INSTALLED_PLUGINS: PluginSource[] = [
  { manifest: manifestFor('map-centre'), entry: () => import('./map-centre') },
]
```
:::

## Next

- [Capabilities](./all-capabilities.md) — everything a plugin can add, and what each contribution receives
- [Example: one plugin, several surfaces](./hello-map-example.md) — a plugin with six of them
