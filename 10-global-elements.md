# Global Elements Guide

Global elements (navbar, footer, CTA, etc.) are configured once per project and available on every page. They live in `src/components/layout/`.

> ## ⚠️ Read this before making the navbar/footer a global element
>
> **The default, production-proven pattern is to keep the navbar and footer
> HARDCODED in code — not CMS-managed at all.** Real SpaceBlock sites do this:
> e.g. accessmemory.co has ~99 `data-cms-id` fields across its page content but
> **zero** inside its `<nav>` or `<footer>` — the CMS owns the page, the site
> owns its chrome. This sidesteps every problem below.
>
> Why you'll hit problems if you make navbar/footer CMS elements:
>
> 1. **No field schema.** A global element's editor is chosen **solely by its
>    `type`**. `navbar`/`footer`/`cta` *may* have dedicated form editors, but
>    **any dashboard build without them falls back to a raw JSON editor**, and
>    there is **no** `schema`/`template` you can define to add fields (unlike
>    [templates](./03-templates.md) or [collections](./07-collections.md)).
> 2. **Permanent Element shadowing.** Once a `navbar`/`footer` global-element
>    **record exists**, the editor surfaces it as a **"Permanent Element"** with
>    the JSON editor whenever you click that region — **even if you also render
>    per-field `data-cms-id` content**. The record wins.
> 3. **You can't remove it from code.** The public API can only **deactivate**
>    (`isActive:false`), **not hard-delete**, a global element. A lingering
>    record keeps showing the Permanent Element panel until someone deletes it
>    in the dashboard.
>
> **Guidance:**
>
> - **Rarely-changing chrome (most navbars/footers): hardcode it** in
>   `Navbar.tsx` / `Footer.tsx` as static markup. Done — no records, no JSON.
> - **Need per-field editing AND your dashboard lacks the navbar/footer
>   editors:** use **`data-cms-id` content fields** (see
>   [Navbar & Footer as content fields](#navbar--footer-as-content-fields)) —
>   but **do not also create a navbar/footer global-element record**, or the
>   Permanent Element panel (point 2) will shadow your fields.
> - **Your dashboard has the dedicated `navbar`/`footer`/`cta` editors:** use
>   real global elements as described in the rest of this chapter.

## Creating Global Elements

Global elements are **created in the SpaceBlock dashboard**, not from your app at runtime. Your React app only **reads** global elements (via `fetchGlobalElement`) and listens for `GLOBAL_ELEMENT_UPDATE` postMessage events from the editor.

### Why not auto-register from the browser?

The PUT endpoint at `/api/public/global-elements/:projectId` exists, but **CORS preflight blocks `PUT` from browser origins** — so calling it from your React app at startup fails with `Method PUT is not allowed by Access-Control-Allow-Methods`. GET works, POST may, but PUT is rejected. Don't try to write a browser-side seed helper; it won't work and silently failing PUTs will pollute your console.

### How to create them

1. Open the SpaceBlock dashboard for your project.
2. Find the **Global Elements** section (or wherever your CMS surfaces project-level singletons).
3. Create one record per element your app needs:
   - `navbar` (type: `navbar`)
   - `footer` (type: `footer`)
   - `cta` (type: `cta`) — only if you're using `GlobalCta`
4. Fill in the content shape your component expects (see "Content Types" below).

After they exist in the CMS, your `Navbar` / `Footer` components fetch them on mount and re-render on every editor change.

### Server-side seeding via `npm run seed:globals`

The recommended way to bootstrap navbar/footer (and any future global elements) is a Node script invoked via npm. CORS doesn't apply to Node, so PUT works.

`scripts/seed-global-elements.mjs` reads `.env.local`, fetches the existing elements, and only PUTs ones whose `elementId` is missing — so re-running it is safe and won't clobber editor changes.

```bash
npm run seed:globals
```

Sample output on a fresh project:

```
→ Fetching existing global elements at https://www.spaceblock.app/api/public/global-elements/<projectId>
  Found 0 existing element(s): []
→ Creating navbar (type=navbar)... OK
→ Creating footer (type=footer)... OK

Done. Created 2, skipped 0.
```

To customize the seeded defaults, edit `DEFAULT_GLOBAL_ELEMENTS` in `scripts/seed-global-elements.mjs`. To seed during deploy, add `npm run seed:globals` to your CI step (idempotent, so safe to run every deploy).

### Manual seeding via `curl`

If you'd rather not check in the script, the same effect with a one-liner:

```bash
curl -X PUT \
  "https://www.spaceblock.app/api/public/global-elements/$PROJECT_ID?apiKey=$API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "elementId": "navbar",
    "type": "navbar",
    "content": { "logoAlt": "My App", "items": [] }
  }'
```

Repeat for `footer`, etc. PUT is an upsert — it creates if missing, updates if present.

### PUT API reference

| Field | Required | Description |
|-------|----------|-------------|
| `elementId` | Yes | Unique ID within the project (e.g. `"navbar"`) |
| `type` | Yes | Element type — drives which CMS editor is shown |
| `content` | No | JSON content object (defaults to `{}`) |
| `customization` | No | JSON customization object (defaults to `{}`) |
| `order` | No | Display order (auto-assigned if omitted) |
| `isActive` | No | Whether element is active (defaults to `true`) |

## File Structure

```
src/components/layout/
  Layout.tsx     ← wraps every page; owns TemplateRegistry
  Navbar.tsx     ← fetches CMS navbar; handles live preview
  Footer.tsx     ← fetches CMS footer; handles live preview
  GlobalCta.tsx  ← fetches CMS CTA; handles live preview
  index.ts       ← re-exports Layout, Navbar, Footer
```

`App.tsx` uses `<Layout>` as the single shell:

```tsx
// src/App.tsx
import { Layout } from '@/components/layout'

export default function App() {
  return (
    <Layout>
      <Routes>
        <Route path="/" element={<HomePage />} />
        {/* ... */}
        <Route path="/*" element={<DynamicPage />} />
      </Routes>
    </Layout>
  )
}
```

## Fetching Global Elements

```typescript
// src/lib/api.ts

export async function fetchNavbarContent(): Promise<NavbarContent | null> {
  const element = await fetchGlobalElement('navbar')
  return element ? (element.content as NavbarContent) : null
}

export async function fetchFooterContent(): Promise<FooterContent | null> {
  const element = await fetchGlobalElement('footer')
  return element ? (element.content as FooterContent) : null
}

export async function fetchCtaContent(): Promise<CtaContent | null> {
  const element = await fetchGlobalElement('cta')
  return element ? (element.content as CtaContent) : null
}
```

## Navbar

```tsx
// src/components/layout/Navbar.tsx

export function Navbar() {
  const [content, setContent] = useState<NavbarContent | null>(null)
  const [hoveredItem, setHoveredItem] = useState<string | null>(null)

  // Initial fetch
  useEffect(() => {
    fetchNavbarContent().then(setContent).catch(() => {})
  }, [])

  // Live preview: update instantly when edited in the CMS
  useEffect(() => {
    function handleMessage(event: MessageEvent) {
      if (event.data.type === 'GLOBAL_ELEMENT_UPDATE' && event.data.elementType === 'navbar') {
        setContent(event.data.content)
      }
    }
    window.addEventListener('message', handleMessage)
    return () => window.removeEventListener('message', handleMessage)
  }, [])

  // Support both new 'items' format and legacy 'links' format
  const items = content?.items?.length > 0
    ? content.items
    : content?.links?.map(l => ({ label: l.label, path: l.path })) ?? DEFAULT_ITEMS

  return (
    <nav className="fixed top-0 inset-x-0 z-50 bg-white/90 backdrop-blur-md border-b border-gray-100"
         onMouseLeave={() => setHoveredItem(null)}>
      {/* logo + nav items */}
      {/* dropdown area */}
    </nav>
  )
}
```

## Footer

```tsx
// src/components/layout/Footer.tsx

const DEFAULT_CONTENT: FooterContent = { /* ... */ }

export function Footer() {
  const [content, setContent] = useState<FooterContent>(DEFAULT_CONTENT)

  useEffect(() => {
    fetchFooterContent()
      .then(data => { if (data) setContent({ ...DEFAULT_CONTENT, ...data }) })
      .catch(() => {})
  }, [])

  // Live preview
  useEffect(() => {
    function handleMessage(event: MessageEvent) {
      if (event.data.type === 'GLOBAL_ELEMENT_UPDATE' && event.data.elementType === 'footer') {
        setContent({ ...DEFAULT_CONTENT, ...event.data.content })
      }
    }
    window.addEventListener('message', handleMessage)
    return () => window.removeEventListener('message', handleMessage)
  }, [])

  return <footer>...</footer>
}
```

## CTA (Thin Call-to-Action Bar)

A thin CTA bar displayed above the footer on all pages.

```tsx
// src/components/layout/GlobalCta.tsx

export function GlobalCta() {
  const [content, setContent] = useState<CtaContent | null>(null)

  useEffect(() => {
    fetchCtaContent().then(setContent).catch(() => {})
  }, [])

  useEffect(() => {
    function handleMessage(event: MessageEvent) {
      if (event.data.type === 'GLOBAL_ELEMENT_UPDATE' && event.data.elementType === 'cta') {
        setContent(event.data.content)
      }
    }
    window.addEventListener('message', handleMessage)
    return () => window.removeEventListener('message', handleMessage)
  }, [])

  if (!content || (!content.text && !content.buttonText)) return null

  return (
    <ThinCta
      variant={content.variant || 'dark'}
      text={content.text}
      buttonText={content.buttonText}
      buttonUrl={content.buttonUrl}
      buttonIcon={content.buttonIcon}
    />
  )
}
```

Add it to `Layout.tsx` above the footer:

```tsx
export function Layout({ children }: LayoutProps) {
  return (
    <div className="min-h-screen flex flex-col">
      <Navbar />
      <main className="flex-1">{children}</main>
      <GlobalCta />
      <Footer />
      <TemplateRegistry />
      <TemplateOverlay />
    </div>
  )
}
```

## Content Types

### Navbar

```typescript
interface NavbarContent {
  logo?: string
  logoAlt?: string
  items?: NavItem[]   // preferred
  links?: NavLink[]   // legacy fallback
}

interface NavItem {
  label: string
  path?: string
  children?: NavLink[]  // dropdown items
}
```

### Footer

```typescript
interface FooterContent {
  columns?: Array<{ title: string; links: NavLink[] }>
  contactTitle?: string
  contactEmail?: string
  copyright?: string
  legalLinks?: NavLink[]
  socialLinks?: Array<{ label: string; url: string }>
  designCredit?: string
}
```

### CTA

```typescript
interface CtaContent {
  variant?: 'dark' | 'light'
  text?: string
  buttonText?: string
  buttonUrl?: string
  buttonIcon?: string
}
```

## Supported Element Types

The CMS ships with dedicated editors for these types:

| Type | Editor | Description |
|------|--------|-------------|
| `navbar` | NavbarEditor | Logo, navigation items with dropdown support |
| `footer` | FooterEditor | Columns, links, contact, social, legal |
| `cta` | CtaEditor | Dark/light variant, text, button with icon |

Any other `type` value falls back to a raw JSON editor in the CMS.

## Live Preview Messages

The CMS Visual Editor sends these `postMessage` events:

| Type | Payload | Action |
|------|---------|--------|
| `GLOBAL_ELEMENT_UPDATE` | `{ elementType, content }` | Update Navbar, Footer, or CTA state directly |
| `CMS_INSERT_ELEMENT` | `{ elementId }` | Remove SDK-inserted DOM node, re-fetch page |
| `CMS_REFRESH_AFTER_DELETE` | — | Re-fetch page |

Each global element component listens for its own `GLOBAL_ELEMENT_UPDATE` event (filtered by `elementType`). `DynamicPage` handles the page-level events.

## Navbar & Footer as content fields

> Only reach for this if you genuinely need per-field editing of the navbar/
> footer **and** your dashboard lacks the dedicated editors. For most sites,
> **hardcoding is simpler and is the production norm** (see the warning at the
> top). **Critical:** this approach only works if there is **no** `navbar`/
> `footer` global-element record — an existing record surfaces the "Permanent
> Element" JSON panel and shadows these fields. Delete any such record in the
> dashboard first (the public API can't hard-delete it).

When your dashboard shows raw JSON for the navbar/footer, skip the
global-element type entirely and build them from **`data-cms-id` content
fields**. Each part becomes a normal inline-editable field, content is
project-wide (so it's global), and it works in every dashboard build.

### A tiny content hook

Content blocks are edited by clicking the visible element; keep them live with a
small hook that fetches all content and patches on `CMS_UPDATE`:

```tsx
// src/lib/useCmsContent.ts — fetch content blocks + live-update on edit.
export function useCmsContent() {
  const [content, setContent] = useState<Record<string, CmsFieldValue>>({})
  useEffect(() => { fetchContent().then(setContent).catch(() => {}) }, [])
  useEffect(() => {
    const on = (e: MessageEvent) => {
      const m = e.data
      if (m?.type === 'CMS_UPDATE' && typeof m.cmsId === 'string')
        setContent(prev => ({ ...prev, [m.cmsId]: m.content }))
    }
    window.addEventListener('message', on)
    return () => window.removeEventListener('message', on)
  }, [])

  const text = (id: string, fb = '') => {
    const v = content[id]
    return (typeof v === 'object' && v ? (v.text ?? v.url ?? '') : v ? String(v) : '') || fb
  }
  // A `url` field carries BOTH a destination and a label ({ url, text }).
  const link = (id: string, fbLabel = '', fbUrl = '#') => {
    const v = content[id]
    if (v && typeof v === 'object') return { label: v.text || fbLabel, url: v.url || fbUrl }
    if (typeof v === 'string' && v) return { label: fbLabel, url: v }
    return { label: fbLabel, url: fbUrl }
  }
  return { text, link }
}
```

`fetchContent()` must return the **raw** value (don't flatten `{ url, text }`),
so the hook can recover a link's label and destination.

### Fixed fields + numbered link slots

Render single values as `text`/`url` fields, and repeating lists (nav items,
footer link columns) as numbered slots (`navbar-link-N`, `footer-explore-N`).
Keep empty slots editable in the editor so items can be added, but hide them on
the public site:

```tsx
const items = Array.from({ length: 6 }, (_, i) => {
  const n = i + 1
  const d = DEFAULT_NAV[i]
  const { label, url } = link(`navbar-link-${n}`, d?.label ?? '', d?.url ?? '/')
  return { n, label, url }
}).filter(it => it.label || isInCmsEditor())  // empty slots: editor-only

// <a data-cms-id={`navbar-link-${n}`} data-cms-type="url" href={url}>{label}</a>
// <span data-cms-id="footer-copyright" data-cms-type="text">{text('footer-copyright','…')}</span>
```

### Responsive caveat — don't duplicate `data-cms-id`

A field's `data-cms-id` must appear **once** in the DOM. If your navbar renders
links in both a desktop bar and a mobile dropdown, put the editable
`data-cms-id` links in **one** of them (typically the desktop bar) and render
the other as plain, non-editable copies — otherwise the SDK sees duplicate IDs.
Editors then edit the navbar at the width where the editable copy is visible.

### No seeding needed

Content fields are created the first time you edit them, so there's nothing to
seed — `scripts/seed-global-elements.mjs` only matters for real global elements.

## Best Practices

1. **Don't write a browser-side auto-register helper** — CORS blocks PUTs from browser origins. The Cosmos pattern is to create global elements in the SpaceBlock dashboard and consume them via GET only.
2. **Always provide defaults in the component** — `Navbar.tsx` and `Footer.tsx` should render reasonable content before any fetch resolves AND when the CMS doesn't have a record yet. The component is the safety net.
3. **Merge with defaults** — `setContent({ ...DEFAULT_CONTENT, ...data })` so missing fields fall back gracefully when the CMS returns a partial record.
4. **Support legacy `links`** — check `items` first, fall back to `links` for backward compatibility.
5. **Keep layout thin** — `Layout.tsx` only renders Navbar, children, Footer, GlobalCta, and TemplateRegistry; no business logic.
6. **For deploy-time seeding, use a server-side `curl`** — never browser-side. PUT is unblocked from a server, blocked from a browser.
