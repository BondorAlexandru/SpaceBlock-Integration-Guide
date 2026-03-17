# Global Elements Guide

Global elements (navbar, footer, CTA, etc.) are configured once per project and available on every page. They live in `src/components/layout/`.

## Registering Global Elements from Your App

External apps register their global elements via the **public upsert API**. No seed scripts or CMS-side setup required — your app declares which global elements it needs and the CMS creates them automatically.

### PUT `/api/public/global-elements/:projectId`

Upserts (creates or updates) a global element. Authenticate with your project API key.

**Request:**

```bash
curl -X PUT \
  "https://your-cms.com/api/public/global-elements/PROJECT_ID?apiKey=YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "elementId": "cta",
    "type": "cta",
    "content": {
      "variant": "dark",
      "text": "Ready to get started?",
      "buttonText": "Learn More",
      "buttonUrl": "/apply",
      "buttonIcon": ""
    }
  }'
```

**Body fields:**

| Field | Required | Description |
|-------|----------|-------------|
| `elementId` | Yes | Unique ID within the project (e.g. `"navbar"`, `"cta"`) |
| `type` | Yes | Element type — drives which CMS editor is shown |
| `content` | No | JSON content object (defaults to `{}`) |
| `customization` | No | JSON customization object (defaults to `{}`) |
| `order` | No | Display order (auto-assigned if omitted) |
| `isActive` | No | Whether element is active (defaults to `true`) |

**Response (200):**

```json
{
  "element": {
    "id": "ge_abc123",
    "elementId": "cta",
    "type": "cta",
    "content": { ... },
    "customization": {},
    "order": 2,
    "isActive": true
  }
}
```

If the element already exists (matched by `elementId`), it updates the existing record. If not, it creates a new one.

### Registering on App Startup

Call the upsert endpoint from your app's initialization to ensure global elements exist:

```typescript
// src/lib/register-global-elements.ts

const API_BASE = import.meta.env.VITE_CONTENTLITE_API_BASE;
const API_KEY = import.meta.env.VITE_CONTENTLITE_API_KEY;
const PROJECT_ID = import.meta.env.VITE_CONTENTLITE_PROJECT_ID;

const GLOBAL_ELEMENTS = [
  {
    elementId: 'navbar',
    type: 'navbar',
    content: {
      logo: '/media/logo.svg',
      logoAlt: 'My App',
      items: [],
    },
  },
  {
    elementId: 'footer',
    type: 'footer',
    content: {
      columns: [],
      copyright: '© 2026',
    },
  },
  {
    elementId: 'cta',
    type: 'cta',
    content: {
      variant: 'dark',
      text: 'Ready to get started?',
      buttonText: 'Learn More',
      buttonUrl: '/apply',
    },
  },
];

export async function registerGlobalElements() {
  for (const element of GLOBAL_ELEMENTS) {
    try {
      await fetch(
        `${API_BASE}/global-elements/${PROJECT_ID}?apiKey=${API_KEY}`,
        {
          method: 'PUT',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify(element),
        }
      );
    } catch {
      // Silently skip — elements may already exist
    }
  }
}
```

Call it once on startup (e.g. in `main.tsx` or `App.tsx`):

```typescript
import { registerGlobalElements } from '@/lib/register-global-elements';

registerGlobalElements(); // fire-and-forget
```

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

1. **Register elements from your app** — use the PUT upsert API on startup so the CMS always has the elements your app expects
2. **Always provide defaults** — your UI must work before the CMS is configured
3. **Merge with defaults** — `setContent({ ...DEFAULT_CONTENT, ...data })` so missing fields fall back gracefully
4. **Support legacy `links`** — check `items` first, fall back to `links` for backward compatibility
5. **Keep layout thin** — `Layout.tsx` only renders Navbar, children, Footer, GlobalCta, and TemplateRegistry; no business logic
