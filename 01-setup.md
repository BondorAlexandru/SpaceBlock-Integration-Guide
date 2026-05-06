# Setup Guide

This guide walks you through setting up SpaceBlock in a React + Vite + TypeScript + Tailwind application.

## Prerequisites

- A SpaceBlock account
- A project created in the SpaceBlock dashboard
- Your API Key and Project ID (found in Project Settings)

## 1. SDK Loader — `index.html`

Use a dynamic loader so the SDK fetches from localhost when developing and from the production CDN otherwise. Both `localhost` and `127.0.0.1` are treated as local.

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>My App</title>

    <!--
      SpaceBlock CMS SDK loader.
      The SDK must come from the same origin as the CMS so the editor's iframe
      messaging works. Vite substitutes %VITE_SPACEBLOCK_API_BASE% from .env at
      both dev and build time.
    -->
    <script>
      (function () {
        var base = '%VITE_SPACEBLOCK_API_BASE%';
        if (!base || base.indexOf('%VITE_') === 0) {
          console.warn('[SpaceBlock] VITE_SPACEBLOCK_API_BASE is not set; SDK will not load.');
          return;
        }
        var script = document.createElement('script');
        script.src = base.replace(/\/$/, '') + '/spaceblock-sdk.js';
        script.defer = true;
        document.head.appendChild(script);
      })();
    </script>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.tsx"></script>
  </body>
</html>
```

> **Don't branch the SDK URL on `window.location.hostname`.** Your dev server runs on `localhost` but the CMS lives wherever `VITE_SPACEBLOCK_API_BASE` points. If you hardcode `localhost:3000` for the SDK in dev, the script tag silently fails to load (unless you're also running SpaceBlock locally), `window.SpaceBlock` is never defined, and the editor sees zero templates. Always derive the SDK origin from the env var.

## 2. Environment Variables

Create `.env.local` (copy from `.env.example`):

```bash
# Root URL only — do NOT include /api/public
VITE_SPACEBLOCK_API_BASE=https://www.spaceblock.app
VITE_SPACEBLOCK_API_KEY=your-api-key
VITE_SPACEBLOCK_PROJECT_ID=your-project-id
```

For local development pointing at a local CMS instance:

```bash
VITE_SPACEBLOCK_API_BASE=http://localhost:3000
```

Add these to `src/vite-env.d.ts`:

```ts
interface ImportMetaEnv {
  readonly VITE_SPACEBLOCK_API_BASE: string
  readonly VITE_SPACEBLOCK_API_KEY: string
  readonly VITE_SPACEBLOCK_PROJECT_ID: string
}

interface ImportMeta {
  readonly env: ImportMetaEnv
}
```

## 3. SDK Initialisation — `src/App.tsx`

Initialise the SDK **once**, after React has mounted. The module-level `sdkInitialised` flag prevents React StrictMode's double-invoke of `useEffect` from calling `init()` twice.

```tsx
// src/App.tsx
import { useEffect } from 'react'
import { Route, Routes } from 'react-router-dom'
import { Layout } from '@/components/layout'

// Survives React StrictMode double-invoke
let sdkInitialised = false

export default function App() {
  useEffect(() => {
    if (sdkInitialised) return
    sdkInitialised = true

    let timeoutId: ReturnType<typeof setTimeout>

    function init() {
      if (window.SpaceBlock) {
        window.SpaceBlock.init({
          apiKey: import.meta.env.VITE_SPACEBLOCK_API_KEY as string,
          projectId: import.meta.env.VITE_SPACEBLOCK_PROJECT_ID as string,
          autoLoad: false,       // We call load() manually after React renders
          cacheEnabled: true,
          hideUntilLoaded: false, // React provides content via props/state
          debug: import.meta.env.DEV,
        })

        // Wait two animation frames for React to finish rendering, then load
        requestAnimationFrame(() => {
          requestAnimationFrame(() => {
            timeoutId = setTimeout(() => {
              window.SpaceBlock?.load()
            }, 200)
          })
        })
      } else {
        // SDK not yet loaded — poll every 50 ms
        timeoutId = setTimeout(init, 50)
      }
    }

    init()
    return () => clearTimeout(timeoutId)
  }, [])

  return (
    <Layout>
      <Routes>
        {/* your routes */}
      </Routes>
    </Layout>
  )
}
```

### Why `autoLoad: false`?

React renders components before the SDK has seen the DOM. Setting `autoLoad: false` and calling `load()` manually (after two rAF ticks + 200 ms) ensures CMS attributes are in the DOM before the SDK scans them.

## 4. Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `apiKey` | string | — | Your SpaceBlock API key (required) |
| `projectId` | string | — | Your project ID (required) |
| `autoLoad` | boolean | `false` | Auto-load content on init. Use `false` in SPAs |
| `cacheEnabled` | boolean | `true` | Enable local content caching |
| `hideUntilLoaded` | boolean | `true` | Hide content until loaded. Use `false` when React provides fallback values |
| `debug` | boolean | `false` | Enable SDK debug logging |

## 5. SDK Methods

### `window.SpaceBlock.init(options)`
Initialise the SDK. Call once, before `load()`.

### `window.SpaceBlock.load()`
Scan the DOM for `data-cms-id` / `data-cms-insertable` attributes and activate the editor. Returns a Promise.

### `window.SpaceBlock.refresh()`
Re-fetch content from the server, bypassing cache.

## 6. Setting Up Domains

In **SpaceBlock → Project Settings → Domains**, add:

- **Production domain**: your live site URL (e.g. `https://mysite.com`)
- **Development domain**: your local dev server (e.g. `http://localhost:5173`)

The Visual Editor loads your site in an iframe using the domain that matches where you're accessing SpaceBlock from.

## Next Steps

- [Visual Editor Integration](./02-visual-editor.md) — Enable inline editing
- [Templates](./03-templates.md) — Create insertable content blocks
- [Content Blocks](./04-content-blocks.md) — Make elements editable
