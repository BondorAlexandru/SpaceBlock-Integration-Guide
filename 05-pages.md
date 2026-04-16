# Pages & Elements Guide

Pages provide structured content management with reorderable elements. This is the recommended approach for dynamic, page-based websites.

## Concepts

- **Page**: A named URL in your project (e.g. Home, About, Contact)
- **PageElement**: A content block on a page, created from a template
- **Zone**: A region on a page where elements live (e.g. `"main"`, `"sidebar"`)

## Fetching Pages

### List All Pages

```javascript
const response = await fetch(
  'https://www.spaceblock.app/api/public/pages/{projectId}/list?apiKey=YOUR_API_KEY'
)
const { pages } = await response.json()
```

### Fetch Single Page with Elements

```javascript
const response = await fetch(
  'https://www.spaceblock.app/api/public/pages/{projectId}/{pageSlug}?apiKey=YOUR_API_KEY'
)
const { page, elements } = await response.json()
```

Response shape:

```json
{
  "page": { "id": "page-123", "name": "Home", "slug": "home", "path": "/" },
  "elements": [
    {
      "id": "elem-1",
      "elementId": "hero-banner-abc123",
      "templateName": "Hero Banner",
      "zone": "main",
      "order": 0,
      "content": {
        "hero-banner-abc123-hero-title": "Welcome",
        "hero-banner-abc123-hero-description": "<p>Description</p>"
      }
    }
  ]
}
```

## DynamicPage Pattern

`DynamicPage.tsx` is the single component that handles every CMS-driven page. It:

1. Reads the URL slug
2. Fetches the page + elements from the API (with in-memory cache)
3. Sorts elements by `order` within each zone
4. Calls `renderElement()` for each element
5. Listens for CMS postMessage events to invalidate cache in real time

```tsx
// src/pages/DynamicPage.tsx

const pageCache = new Map<string, { page: Page; elements: PageElement[] }>()

export function DynamicPage() {
  const { '*': splat } = useParams()
  const location = useLocation()
  const slug = (splat ?? location.pathname.replace(/^\//, '')) || 'home'
  const [data, setData] = useState<{ page: Page; elements: PageElement[] } | null>(null)
  const [refreshTrigger, setRefreshTrigger] = useState(0)

  // Fetch (cached) — no loading or error state; elements appear when fetch resolves
  useEffect(() => {
    let cancelled = false

    const cached = pageCache.get(slug)
    if (cached) { setData(cached); return }

    fetchPage(slug)
      .then(result => {
        if (cancelled) return
        pageCache.set(slug, result)
        setData(result)
      })
      .catch(() => {})

    return () => { cancelled = true }
  }, [slug, refreshTrigger])

  // CMS live-preview messages
  useEffect(() => {
    function handleMessage(event: MessageEvent<CmsMessage>) {
      const currentSlug = location.pathname.replace(/^\//, '') || 'home'

      if (event.data.type === 'GLOBAL_ELEMENT_UPDATE') {
        pageCache.delete(currentSlug)
        setRefreshTrigger(n => n + 1)
      }

      if (event.data.type === 'CMS_INSERT_ELEMENT') {
        const { elementId } = event.data
        setTimeout(() => {
          document.querySelector(`[data-cms-element-id="${elementId}"]`)?.remove()
          pageCache.delete(currentSlug)
          setRefreshTrigger(n => n + 1)
        }, 100)
      }

      if (event.data.type === 'CMS_REFRESH_AFTER_DELETE') {
        pageCache.delete(currentSlug)
        setRefreshTrigger(n => n + 1)
      }
    }
    window.addEventListener('message', handleMessage)
    return () => window.removeEventListener('message', handleMessage)
  }, [location.pathname])

  const mainElements = (data?.elements ?? [])
    .filter(el => el.zone === 'main')
    .sort((a, b) => a.order - b.order)

  return (
    <main data-cms-zone="main">
      {mainElements.map(element => (
        <div key={element.id}>{renderElement(element)}</div>
      ))}
    </main>
  )
}
```

## The `find()` Helper

Inside `renderElement()`, use a suffix-based accessor so you never have to construct the full `${elementId}-${field}` key manually:

```tsx
function renderElement(element: PageElement) {
  const { elementId, templateName, content } = element
  const contentKeys = Object.keys(content)

  const find = (suffix: string) => {
    const key = contentKeys.find(k => k.endsWith(`-${suffix}`))
    return key ? String(content[key] ?? '') : undefined
  }

  // ... detection + rendering
}
```

## Triple-Detection Strategy

Always use all three detection layers to handle edge cases (e.g. `templateName` not populated by the CMS):

```tsx
if (
  templateName === 'Hero Banner'          // 1. templateName from API
  || elementId.includes('hero-banner')    // 2. elementId prefix (most reliable)
  || contentKeys.some(k => k.endsWith('-hero-title'))  // 3. content key pattern
) {
  return <HeroBanner elementId={elementId} title={find('hero-title')} ... />
}
```

## Content Key Pattern

Content keys follow `{elementId}-{fieldSuffix}`:

```
hero-banner-abc123-hero-title       → suffix: hero-title
hero-banner-abc123-hero-description → suffix: hero-description
```

The `find(suffix)` helper handles this automatically.

## Zones

Mark the render target with `data-cms-zone` so the Visual Editor knows where to insert:

```tsx
<main data-cms-zone="main">
  {/* elements with zone="main" render here */}
</main>
```

## Creating Pages via API

```javascript
await fetch('/api/projects/{projectId}/pages', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json', 'Authorization': 'Bearer TOKEN' },
  body: JSON.stringify({ name: 'About Us', slug: 'about', path: '/about' })
})
```

## Best Practices

1. **One DynamicPage for all routes** — use `/*` catch-all and derive slug from `useLocation`
2. **Cache page data** — use a module-level Map; invalidate on CMS messages
3. **Sort by `order`** — always sort elements before rendering
4. **Triple-detect templates** — never rely on `templateName` alone
5. **`data-cms-zone="main"`** — always present on the render container

## Next Steps

- [Blog Integration](./06-blog.md) - Blog posts and block rendering
- [Templates](./03-templates.md) - Registering templates and section components
