# Pages & Elements Guide

Pages are CMS-managed URLs whose content is built from a list of reorderable **PageElements** — each one created from a template. `DynamicPage` is the single React component that renders any CMS page.

## Concepts

- **Page** — a named URL in your project (e.g. Home, About, Contact).
- **PageElement** — a content block on a page, instanced from a template. Each has an `elementId` like `hero-banner-abc123`, a `zone`, an `order`, and a `content` map keyed by `${elementId}-${suffix}`.
- **Zone** — a region on a page where elements live (e.g. `"main"`).

## Fetching Pages

```ts
const { page, elements } = await fetchPage(slug)
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
        "hero-banner-abc123-title": "Welcome",
        "hero-banner-abc123-description": "<p>Description</p>",
        "hero-banner-abc123-background": { "url": "https://..." }
      }
    }
  ]
}
```

Note that image fields can come back as either a plain string or `{ url, text }` — the `findImage` helper handles both.

## Cache-busting and `cache: 'no-store'`

`api.ts` adds a `_t=Date.now()` query parameter and uses `fetch(url, { cache: 'no-store' })` on every request. This is essential while the CMS editor is open — without it, browsers and proxies will return stale page data and edits won't appear.

```ts
function publicUrl(path: string, params?: Record<string, string | undefined>): string {
  const url = new URL(`${API_BASE}${path}`)
  url.searchParams.set('apiKey', API_KEY)
  url.searchParams.set('_t', Date.now().toString())
  // ...
}

async function get<T>(url: string): Promise<T> {
  const res = await fetch(url, {
    cache: 'no-store',
    headers: { 'Content-Type': 'application/json' },
  })
  // ...
}
```

## DynamicPage Pattern

`DynamicPage.tsx` is the single component that handles every CMS-driven page. It:

1. Reads the URL slug.
2. Fetches the page + elements (with in-memory cache).
3. Sorts elements by `order` within each zone.
4. Calls `renderElement()` for each element.
5. **Listens for live-preview updates** on three channels: postMessage, MutationObserver on `<img src>`, and structural events.

```tsx
const pageCache = new Map<string, { page: Page; elements: PageElement[]; timestamp: number }>()

export function DynamicPage() {
  const params = useParams()
  const location = useLocation()
  const slug = (params['*'] ?? location.pathname.replace(/^\//, '')) || 'home'

  const [data, setData] = useState<{ page: Page; elements: PageElement[] } | null>(null)
  const [refreshTrigger, setRefreshTrigger] = useState(0)
  const insertTimestamps = useRef<Map<string, number>>(new Map())

  useEffect(() => {
    const cached = pageCache.get(slug)
    if (cached) { setData({ page: cached.page, elements: cached.elements }); return }

    fetchPage(slug)
      .then(result => {
        pageCache.set(slug, { ...result, timestamp: Date.now() })
        setData(result)
      })
      .catch(() => {})
  }, [slug, refreshTrigger])

  // ...live-preview wiring...

  const main = (data?.elements ?? [])
    .filter(el => el.zone === 'main')
    .sort((a, b) => a.order - b.order)

  return (
    <main data-cms-zone="main">
      {main.map(el => <div key={el.id}>{renderElement(el)}</div>)}
    </main>
  )
}
```

## Live Preview — Three Update Channels

The CMS editor pushes changes into the rendered page via three different mechanisms. `DynamicPage` handles all three.

### 1. `CMS_UPDATE` postMessage — single field edits

Fires whenever the editor changes one field (text, richtext, url, select, toggle). The handler patches just the affected element's content in state — no refetch — so React re-renders only the changed component.

```tsx
function applyFieldUpdate(cmsId: string, newValue: string, domFallbackValue?: string) {
  setData(prev => {
    if (!prev) return prev
    let changed = false
    const nextElements = prev.elements.map(el => {
      const matches = cmsId in el.content || cmsId.startsWith(`${el.elementId}-`)
      if (!matches) return el
      // ...compare against current value; ignore no-ops; honor intentional clears...
      changed = true
      return { ...el, content: { ...el.content, [cmsId]: newValue } }
    })
    return changed ? { ...prev, elements: nextElements } : prev
  })
}

if (msg.type === 'CMS_UPDATE') {
  const { cmsId, content } = msg
  let newValue =
    typeof content === 'string' || typeof content === 'number' || typeof content === 'boolean'
      ? String(content)
      : content != null && typeof content === 'object'
        ? ((content as { url?: string; text?: string }).url ??
           (content as { url?: string; text?: string }).text ?? '')
        : ''

  // DOM fallback: some CMS update flows mutate textContent directly and send
  // an empty postMessage. Read the DOM to recover the value.
  let domFallbackValue: string | undefined
  if (!newValue) {
    const el = document.querySelector(`[data-cms-id="${cmsId}"]`)
    const text = el?.textContent?.trim() ?? ''
    if (text) { newValue = text; domFallbackValue = text }
  }
  applyFieldUpdate(cmsId, newValue, domFallbackValue)
}
```

Why the DOM fallback: when an editor "clears" a field, the postMessage content is empty. We want to honor that clear. But some other update flows (e.g. the SDK's internal updateCount) also send empty content while having already written the value to `textContent`. We compare the DOM value against current state — if they match, the empty content was a real clear, not a missed update.

### 2. MutationObserver — `<img src>` changes

Image pickers in some CMS flows update the `<img src>` attribute directly and skip the postMessage. A `MutationObserver` on the document body catches that and mirrors the new src into state:

```tsx
const observer = new MutationObserver(mutations => {
  for (const m of mutations) {
    if (m.type !== 'attributes' || m.attributeName !== 'src') continue
    const target = m.target as HTMLElement
    if (target.getAttribute('data-cms-type') !== 'image') continue
    const cmsId = target.getAttribute('data-cms-id')
    if (!cmsId) continue
    applyFieldUpdate(cmsId, target.getAttribute('src') || '')
  }
})
observer.observe(document.body, { attributes: true, attributeFilter: ['src'], subtree: true })
```

### 3. Structural events — insert / delete / global update

These events change the **set** of elements on the page. They invalidate the cache and trigger a refetch.

```tsx
if (msg.type === 'CMS_INSERT_ELEMENT') {
  const { elementId } = msg
  insertTimestamps.current.set(elementId, Date.now())
  setTimeout(() => {
    document.querySelector(`[data-cms-element-id="${elementId}"]`)?.remove()
    pageCache.delete(slug)
    setRefreshTrigger(n => n + 1)
  }, 100)
}

if (msg.type === 'CMS_REFRESH_AFTER_DELETE') {
  pageCache.delete(slug)
  setRefreshTrigger(n => n + 1)
}
```

`insertTimestamps` is a `useRef<Map<string, number>>` that records when each element was inserted. Use it inside detection branches to ignore "initialization" updates from the CMS that fire in the first few hundred milliseconds after insertion (some templates ship default state via postMessage that races with React's first render).

Global element updates (Navbar/Footer/CTA) are handled by those components themselves — `DynamicPage` ignores `GLOBAL_ELEMENT_UPDATE`.

## The `find()` and `findImage()` Helpers

Inside `renderElement()`, use suffix-based lookups instead of constructing full `${elementId}-${field}` keys:

```tsx
function findValue(content: Record<string, unknown>, suffix: string): string | undefined {
  const key = Object.keys(content).find(k => k.endsWith(`-${suffix}`) || k === suffix)
  if (!key) return undefined
  const raw = content[key]
  if (raw == null) return undefined
  if (typeof raw === 'object') {
    const obj = raw as { url?: string; text?: string }
    return obj.url ?? obj.text ?? ''
  }
  return String(raw)
}

function findImage(content: Record<string, unknown>, ...suffixes: string[]): string | undefined {
  const keys = Object.keys(content)
  for (const suffix of suffixes) {
    const key = keys.find(k => k.endsWith(`-${suffix}`) || k === suffix)
    if (!key) continue
    const raw = content[key]
    if (raw == null) continue
    if (typeof raw === 'object') {
      const url = (raw as { url?: string }).url
      if (url) return url
      continue
    }
    const str = String(raw)
    if (str) return str
  }
  return undefined
}
```

Both helpers handle the `{ url }` / `{ text }` object form that the CMS sometimes returns. `findImage` accepts multiple suffixes for fields whose name might shift across template versions.

## Triple-Detection Strategy

Always use all three detection layers — `templateName` is sometimes unpopulated by the CMS:

```tsx
if (
  templateName === 'Hero Banner'                              // 1. exact templateName match
  || elementId.includes('hero-banner')                        // 2. elementId prefix (most reliable)
  || contentKeys.some(k => k.endsWith('-partner-logo'))       // 3. unique field key
) {
  return <HeroBanner elementId={elementId} ... />
}
```

Pick a content-key suffix that's **unique to that template** as the third detector. For example, `partner-logo` only appears in HeroBanner, so its presence is unambiguous; `title` is too generic.

## Zones

Mark the render container with `data-cms-zone` so the Visual Editor knows where to drop new elements:

```tsx
<main data-cms-zone="main">
  {/* elements with zone="main" render here */}
</main>
```

If you have multiple zones (e.g. `main` + `sidebar`), filter and render each zone separately.

## HomePage — Static Composition Variant

`HomePage.tsx` is a sibling pattern: it doesn't fetch a `Page`, it renders a fixed composition of section components and feeds them content fetched from the simple **content blocks** API (`fetchContent()`). Each section gets a stable `elementId` like `home-hero`, so its field keys end up as `home-hero-title`, `home-hero-description`, etc. — readable and unique across the site.

It listens for the same `CMS_UPDATE` postMessage and the same `<img src>` `MutationObserver`, but patches its flat content map instead of a per-element store.

```tsx
useEffect(() => {
  function patch(key: string, value: string) {
    setContent(prev => (prev[key] === value ? prev : { ...prev, [key]: value }))
  }

  function handleMessage(event: MessageEvent<CmsMessage>) {
    if (event.data?.type !== 'CMS_UPDATE') return
    // ...unwrap value...
    patch(event.data.cmsId, value)
  }
  // ...same MutationObserver as DynamicPage...
}, [])
```

Use this pattern for any page where the layout is fixed in code but the copy is editable.

## Best Practices

1. **One DynamicPage for all CMS routes** — use `/*` catch-all and derive the slug from `useLocation`.
2. **Cache page data** — module-level `Map`; invalidate on insert/delete/global-update events.
3. **Sort by `order`** — always sort elements before rendering.
4. **Triple-detect templates** — never rely on `templateName` alone.
5. **Use `findImage` for every image field** — never call `find()` for image data.
6. **`data-cms-zone="main"`** — always present on the render container.
7. **Cache-bust every fetch** — `_t=Date.now()` query param + `cache: 'no-store'` on the request.

## Next Steps

- [Templates](./03-templates.md) — TemplateRegistry, schema declaration, section components
- [Blog Integration](./06-blog.md) — Blog posts and block rendering
