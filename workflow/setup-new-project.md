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

3. **src/lib/types.ts** — TypeScript interfaces for: SpaceBlockInitOptions, ContentBlock, BlogPost (with BlogBlock types), Page, PageElement, Collection + CollectionItem + CollectionItemLink, NavbarContent + NavItem + NavLink, FooterContent, GlobalElement, CmsMessage union type; `declare global { interface Window { SpaceBlock?: ... } }`

4. **src/lib/api.ts** — fetch helpers using `import.meta.env.VITE_SPACEBLOCK_*`; implement: fetchContent(), fetchPages(), fetchPage(slug), fetchBlogPosts(params?), fetchBlogPost(slug), fetchBlogPreview(token), fetchCollection(name), fetchGlobalElements(), fetchNavbarContent(), fetchFooterContent(); internal `get<T>(url)` helper and `publicUrl(path, params?)` builder

5. **src/components/layout/Navbar.tsx** — fetches CMS navbar via fetchNavbarContent(); listens for GLOBAL_ELEMENT_UPDATE postMessage to update content in real time; supports items (with children dropdowns) and legacy links; falls back to DEFAULT_ITEMS

6. **src/components/layout/Footer.tsx** — fetches CMS footer via fetchFooterContent(); listens for GLOBAL_ELEMENT_UPDATE; merges with DEFAULT_CONTENT; renders columns, contact, copyright, legal links, social links

7. **src/components/layout/Layout.tsx** — exports `Layout` wrapping `<Navbar /> <main className="flex-1 pt-16">{children}</main> <Footer /> <TemplateRegistry />`; `TemplateRegistry` is a **private function component in the same file** (not exported, not a separate file); it only renders when `window.self !== window.top`; it renders every section component **without an elementId** (template mode) inside a visually-hidden div using Tailwind: `className="absolute w-px h-px p-0 -m-px overflow-hidden [clip:rect(0,0,0,0)] whitespace-nowrap border-0"`; because sections are dual-mode, rendering them without an elementId automatically produces the `data-cms-insertable` attributes the SDK needs

8. **src/components/layout/index.ts** — `export { Layout } from './Layout'`

9. **src/components/sections/** — one file per requested template using the **dual-mode pattern**:
   - Accept `elementId?: string`; derive `const isTemplateMode = !elementId`
   - Put `data-cms-insertable`, `data-cms-name`, `data-cms-category` on the root element **only when `isTemplateMode`**
   - Put `data-cms-element-id={elementId}` on the root element **only when not `isTemplateMode`**
   - For each field: `const fieldId = elementId ? \`${elementId}-suffix\` : 'suffix'` then `data-cms-id={fieldId}`
   - This way the component handles both template registration and live rendering

10. **src/pages/DynamicPage.tsx** — module-level pageCache Map; fetchPage() with cache; CMS postMessage handler (GLOBAL_ELEMENT_UPDATE, CMS_INSERT_ELEMENT, CMS_REFRESH_AFTER_DELETE); `renderElement(element)` plain function (not a component, not a separate file) with `find(suffix)` helper and triple-detection (templateName + elementId.includes() + content key pattern) for each template; renders `<main data-cms-zone="main">`

11. **src/pages/HomePage.tsx** — fetches content via fetchContent(); renders with data-cms-id attributes for inline editing; provide default fallback strings

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
    │   │   ├── Layout.tsx       (+ TemplateRegistry inside)
    │   │   ├── Navbar.tsx
    │   │   ├── Footer.tsx
    │   │   └── index.ts
    │   └── sections/
    │       ├── HeroBanner.tsx
    │       ├── FeatureSection.tsx
    │       ├── SimpleText.tsx
    │       ├── RichText.tsx
    │       └── BlogContentBlock.tsx
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
- [ ] Open the SpaceBlock Visual Editor and verify templates appear in the "Add Element" list
- [ ] Add a new template: register in `TemplateRegistry`, create section component, add detection branch in `renderElement()`

## Adding a new template later

Ask Claude:

```
Add a new SpaceBlock template called "<Template Name>" with these fields:
- <field name> (<type: text|richtext|image|url|select|toggle>)
- ...

Follow the pattern in the integration guide:
1. Register it in src/components/layout/Layout.tsx TemplateRegistry
2. Create src/components/sections/<TemplateName>.tsx
3. Add detection + rendering in src/pages/DynamicPage.tsx renderElement()
```
