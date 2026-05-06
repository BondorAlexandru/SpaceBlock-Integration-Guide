# Claude Workflow: Set Up a SpaceBlock CMS React Project

Use this workflow when bootstrapping a new React + Vite + TypeScript + Tailwind project integrated with SpaceBlock CMS.

Paste the prompt below into Claude Code (in the target project directory) to create the full skeleton in one shot.

---

## Prompt

```
Set up a React + Vite + TypeScript + Tailwind CSS project integrated with SpaceBlock CMS.

Follow the structure and patterns from the SpaceBlock integration guide located at:
<path to integration-guide>

### Project details
- Project name: <name>
- Dev server port: <port, e.g. 5173>
- Templates to create: <comma-separated list, e.g. "Hero Banner, Feature Section, Simple Text, Rich Text">

### Steps to complete

1. **Config files** — create these files:
   - `package.json` with dependencies: react, react-dom, react-router-dom; devDependencies: @types/node, @types/react, @types/react-dom, @vitejs/plugin-react, autoprefixer, postcss, tailwindcss, typescript, vite
   - `vite.config.ts` — include `@/` path alias pointing to `./src`, dev server on the specified port
   - `tsconfig.json`, `tsconfig.app.json`, `tsconfig.node.json` — standard Vite+React+TS setup; `tsconfig.app.json` includes `@/*` path alias; `tsconfig.node.json` includes `"types": ["node"]`
   - `tailwind.config.ts` — content: `['./index.html', './src/**/*.{ts,tsx}']`
   - `postcss.config.js`
   - `.gitignore` — ignore node_modules, dist, .env, .env.local
   - `.env.example` — with VITE_SPACEBLOCK_API_BASE, VITE_SPACEBLOCK_API_KEY, VITE_SPACEBLOCK_PROJECT_ID; note that API_BASE is the root URL only (no /api/public suffix)

2. **index.html** — dynamic SDK loader script (chooses localhost:3000 vs spaceblock.app based on hostname); `<div id="root">`; module script for `src/main.tsx`

3. **src/lib/types.ts** — TypeScript interfaces for: SpaceBlockInitOptions, ContentBlock, BlogPost (with BlogBlock types), Page, PageElement, Collection + CollectionItem + CollectionItemLink, NavbarContent + NavItem + NavLink, FooterContent, GlobalElement; `CmsMessage` union type covering `GLOBAL_ELEMENT_UPDATE`, `CMS_INSERT_ELEMENT`, `CMS_REFRESH_AFTER_DELETE`, and `CMS_UPDATE` (where `content` is `unknown` because the CMS may send strings, numbers, booleans, or `{ url, text }` objects); `declare global { interface Window { SpaceBlock?: ... } }`

4. **src/lib/api.ts** — fetch helpers using `import.meta.env.VITE_SPACEBLOCK_*`. Every fetch must use `cache: 'no-store'` AND a `_t=Date.now()` query parameter — without both, browsers/proxies serve stale page data while the CMS editor is open. Implement: fetchContent(), fetchPages(), fetchPage(slug), fetchBlogPosts(params?), fetchBlogPost(slug), fetchBlogPreview(token), fetchCollection(name), fetchGlobalElements(), fetchNavbarContent(), fetchFooterContent(); internal `get<T>(url)` helper and `publicUrl(path, params?)` builder that injects the cache-buster.

5. **src/components/layout/Navbar.tsx** — fetches CMS navbar via fetchNavbarContent(); listens for GLOBAL_ELEMENT_UPDATE postMessage to update content in real time; supports items (with children dropdowns) and legacy links; falls back to DEFAULT_ITEMS

6. **src/components/layout/Footer.tsx** — fetches CMS footer via fetchFooterContent(); listens for GLOBAL_ELEMENT_UPDATE; merges with DEFAULT_CONTENT; renders columns, contact, copyright, legal links, social links

7. **src/components/layout/Layout.tsx** — exports `Layout` wrapping `<Navbar /> <main className="flex-1 pt-16">{children}</main> <Footer /> <TemplateRegistry />`. `TemplateRegistry` is a **private function component in the same file** (not exported, not separate). It only renders inside the CMS iframe (`window.self !== window.top`) and lives inside a visually-hidden div: `className="absolute w-px h-px p-0 -m-px overflow-hidden [clip:rect(0,0,0,0)] whitespace-nowrap border-0"`.

   Inside that div, declare each template as a **hardcoded `<div data-cms-insertable="ID" data-cms-name="..." data-cms-category="...">` schema block** with `<span data-cms-id="suffix" data-cms-type="...">` / `<img data-cms-id="..." data-cms-type="image">` children for each field. Do **not** call the section components in template mode — the registry is the schema, the section components are the renderers; keep them decoupled. Field IDs in the registry are bare suffixes (e.g. `title`, `step-1-description`, `1-url`).

8. **src/components/layout/index.ts** — `export { Layout } from './Layout'`

9. **src/components/sections/<TemplateName>.tsx** — one file per requested template, render-mode only:
   - Accept `elementId?: string` plus typed props for every field
   - Build a `fid()` helper: `const fid = (s: string) => elementId ? \`${elementId}-${s}\` : s`
   - Put `data-cms-element-id={elementId}` on the section root
   - Put `data-cms-id={fid('suffix')}` on each editable element, with the matching `data-cms-type` attribute
   - Provide sensible default content via prop defaults so the component renders standalone for testing
   - Do NOT emit `data-cms-insertable` / `data-cms-name` / `data-cms-category` — those belong only in the registry

10. **src/pages/DynamicPage.tsx** — single CMS page renderer. Includes:
    - Module-level `pageCache: Map<string, { page: Page; elements: PageElement[]; timestamp: number }>`
    - `findValue(content, suffix)` and `findImage(content, ...suffixes)` helpers — both unwrap `{ url, text }` objects
    - `renderElement(element)` plain function with **triple-detection** for each template (`templateName === 'X'` || `elementId.includes('x')` || a unique content-key suffix)
    - **Live preview wiring** with three channels:
      a. `message` listener for `CMS_UPDATE` — patches the matching element's content in state via an `applyFieldUpdate` helper (compare current value, ignore no-ops, honor intentional clears via DOM fallback)
      b. `MutationObserver` on `document.body` watching `attributes/src` on `[data-cms-type="image"]` — mirrors src changes into state when image pickers update the DOM directly
      c. Structural events: `CMS_INSERT_ELEMENT` (record timestamp in `useRef<Map>`, remove placeholder DOM node, invalidate cache, refresh), `CMS_REFRESH_AFTER_DELETE` (invalidate cache, refresh)
    - Render `<main data-cms-zone="main">` containing zone-filtered, order-sorted elements

11. **src/pages/HomePage.tsx** — fixed composition of section components fed by `fetchContent()`. Pass each section a stable `elementId` (e.g. `home-hero`, `home-methodology`, `home-cta`) so field keys are namespaced (`home-hero-title`, `home-hero-description`, …). Mirror DynamicPage's `CMS_UPDATE` handler and `<img src>` MutationObserver, but patch the flat content map instead of a per-element store.

12. **src/pages/BlogListPage.tsx** — fetches posts via fetchBlogPosts(); tag filter via URL search params; loading skeleton; post grid with title, excerpt, author, date, tags

13. **src/pages/BlogPostPage.tsx** — fetches post via fetchBlogPost(slug); renders BlogContentBlock for each block; supports both block-based and HTML string content; back link to /blog

14. **src/pages/BlogPreviewPage.tsx** — fetches via fetchBlogPreview(token); amber preview banner when isPreview=true; same content rendering as BlogPostPage

15. **src/components/sections/BlogContentBlock.tsx** — renders all SpaceBlock blog block types: richtext, heading, image, video, audio, quote, table, download, cta, divider, two-column-text, three-column-text, intro-summary, executive-summary, author-details, acknowledgements, footnotes

16. **src/App.tsx** — module-level `let sdkInitialised = false`; useEffect that guards with sdkInitialised flag, polls for window.SpaceBlock, calls init() then load() via rAF+setTimeout(200); cleanup clears timeout; renders `<Layout><Routes>...</Routes></Layout>`; routes: / HomePage, /blog BlogListPage, /blog/preview/:token BlogPreviewPage, /blog/:slug BlogPostPage, /* DynamicPage

17. **src/main.tsx** — StrictMode + BrowserRouter wrapping App

18. **src/index.css** — @tailwind base/components/utilities; .rich-content prose helper class

19. **src/vite-env.d.ts** — ImportMetaEnv interface with VITE_SPACEBLOCK_* variables

### After creating all files
Run `npm install` to install dependencies.
```

---

## What the workflow produces

```
<project-root>/
├── index.html
├── package.json
├── vite.config.ts
├── tsconfig.json
├── tsconfig.app.json
├── tsconfig.node.json
├── tailwind.config.ts
├── postcss.config.js
├── .gitignore
├── .env.example
└── src/
    ├── App.tsx
    ├── main.tsx
    ├── index.css
    ├── vite-env.d.ts
    ├── lib/
    │   ├── api.ts
    │   └── types.ts
    ├── components/
    │   ├── layout/
    │   │   ├── Layout.tsx       (+ TemplateRegistry inside, hardcoded schemas)
    │   │   ├── Navbar.tsx
    │   │   ├── Footer.tsx
    │   │   └── index.ts
    │   └── sections/
    │       ├── HeroBanner.tsx
    │       ├── FeatureSection.tsx
    │       ├── SimpleText.tsx
    │       ├── RichText.tsx
    │       ├── BlogContentBlock.tsx
    │       └── index.ts
    └── pages/
        ├── DynamicPage.tsx
        ├── HomePage.tsx
        ├── BlogListPage.tsx
        ├── BlogPostPage.tsx
        └── BlogPreviewPage.tsx
```

## Post-setup checklist

- [ ] Copy `.env.example` → `.env.local` and fill in credentials
  - `VITE_SPACEBLOCK_API_BASE` = root URL only, **no** `/api/public` suffix
  - `VITE_SPACEBLOCK_API_KEY` = from SpaceBlock Project Settings
  - `VITE_SPACEBLOCK_PROJECT_ID` = from SpaceBlock Project Settings
- [ ] In SpaceBlock → Project Settings → Domains, add your dev URL (e.g. `http://localhost:5173`)
- [ ] Run `npm install && npm run dev`
- [ ] Open the SpaceBlock Visual Editor
  - Verify templates appear in the **Add Element** list (driven by `TemplateRegistry`)
  - Drop a template onto a page; confirm it renders via `DynamicPage`'s `renderElement()`
  - Edit a text field; confirm the change appears live without a page reload (CMS_UPDATE channel)
  - Replace an image; confirm the new image appears live (MutationObserver channel)
- [ ] Add a new template by following the **Adding a new template** flow below

## Adding a new template later

Ask Claude:

```
Add a new SpaceBlock template called "<Template Name>" with these fields:
- <field name> (<type: text|richtext|image|url|select|toggle>)
- ...

Follow the integration-guide pattern:
1. Add a hardcoded <div data-cms-insertable="<id>" data-cms-name="<Name>" data-cms-category="<category>"> schema block to TemplateRegistry in src/components/layout/Layout.tsx, with one <span> or <img> child per field carrying data-cms-id (bare suffix) and data-cms-type.
2. Create src/components/sections/<TemplateName>.tsx as a render-mode-only component: accept elementId?, build fid() helper, put data-cms-element-id on root, data-cms-id={fid('suffix')} on every editable element. No data-cms-insertable.
3. Re-export from src/components/sections/index.ts.
4. Add a triple-detection branch to renderElement() in src/pages/DynamicPage.tsx using a unique content-key suffix as the fallback detector. Use findImage() for any image fields.
5. (If used on the homepage) wire the props in HomePage.tsx with elementId="home-<id>" and the matching home-prefixed content keys.
```
