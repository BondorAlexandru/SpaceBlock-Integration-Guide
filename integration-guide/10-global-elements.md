# Global Elements Guide

Global elements (navbar, footer) are configured once per project and available on every page. They live in `src/components/layout/`.

## File Structure

```
src/components/layout/
  Layout.tsx   ← wraps every page; owns TemplateRegistry
  Navbar.tsx   ← fetches CMS navbar; handles live preview
  Footer.tsx   ← fetches CMS footer; handles live preview
  index.ts     ← re-exports Layout, Navbar, Footer
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
  const elements = await fetchGlobalElements()
  const navbar = elements.find(el => el.type === 'navbar')
  return (navbar?.content as NavbarContent) ?? null
}

export async function fetchFooterContent(): Promise<FooterContent | null> {
  const elements = await fetchGlobalElements()
  const footer = elements.find(el => el.type === 'footer')
  return (footer?.content as FooterContent) ?? null
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

## Live Preview Messages

The CMS Visual Editor sends these `postMessage` events:

| Type | Payload | Action |
|------|---------|--------|
| `GLOBAL_ELEMENT_UPDATE` | `{ elementType, content }` | Update Navbar or Footer state directly |
| `CMS_INSERT_ELEMENT` | `{ elementId }` | Remove SDK-inserted DOM node, re-fetch page |
| `CMS_REFRESH_AFTER_DELETE` | — | Re-fetch page |

Navbar and Footer each listen for their own `GLOBAL_ELEMENT_UPDATE` event. `DynamicPage` handles the page-level events.

## Best Practices

1. **Always provide defaults** — your UI must work before the CMS is configured
2. **Merge with defaults** — `setContent({ ...DEFAULT_CONTENT, ...data })` so missing fields fall back gracefully
3. **Support legacy `links`** — check `items` first, fall back to `links` for backward compatibility
4. **Keep layout thin** — `Layout.tsx` only renders Navbar, children, Footer, and TemplateRegistry; no business logic
