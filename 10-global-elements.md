# Global Elements Guide

Global elements (navbar, footer, CTA, etc.) are configured once per project and available on every page. They live in `src/components/layout/`.

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

## Best Practices

1. **Don't write a browser-side auto-register helper** — CORS blocks PUTs from browser origins. The Cosmos pattern is to create global elements in the SpaceBlock dashboard and consume them via GET only.
2. **Always provide defaults in the component** — `Navbar.tsx` and `Footer.tsx` should render reasonable content before any fetch resolves AND when the CMS doesn't have a record yet. The component is the safety net.
3. **Merge with defaults** — `setContent({ ...DEFAULT_CONTENT, ...data })` so missing fields fall back gracefully when the CMS returns a partial record.
4. **Support legacy `links`** — check `items` first, fall back to `links` for backward compatibility.
5. **Keep layout thin** — `Layout.tsx` only renders Navbar, children, Footer, GlobalCta, and TemplateRegistry; no business logic.
6. **For deploy-time seeding, use a server-side `curl`** — never browser-side. PUT is unblocked from a server, blocked from a browser.
