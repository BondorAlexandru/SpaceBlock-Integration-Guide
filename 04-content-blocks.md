# Content Blocks Guide

Content Blocks are individual editable pieces of content identified by a unique ID. They're the simplest way to make content editable.

## Basic Usage

### HTML

```html
<!-- Text content -->
<h1 data-cms-id="site-title">My Website</h1>

<!-- Rich text content -->
<div data-cms-id="about-text" data-cms-type="richtext" class="rich-content">
  <p>This is editable rich text content.</p>
</div>

<!-- Image -->
<img
  data-cms-id="hero-image"
  data-cms-type="image"
  src="/default-image.jpg"
  alt="Hero image"
/>
```

### React/JSX

```jsx
function HomePage() {
  return (
    <main>
      <h1 data-cms-id="home-title">Welcome</h1>

      <div
        data-cms-id="home-intro"
        data-cms-type="richtext"
        className="rich-content"
        dangerouslySetInnerHTML={{ __html: introContent }}
      />

      <img
        data-cms-id="home-hero-image"
        data-cms-type="image"
        src={heroImage}
        alt="Hero"
      />
    </main>
  );
}
```

## Content Block Types

### Plain Text

```html
<span data-cms-id="copyright">2024 My Company</span>
```

### Rich Text (HTML)

```html
<div data-cms-id="content" data-cms-type="richtext" class="rich-content">
  <p>Formatted content with <strong>bold</strong> and <em>italic</em>.</p>
</div>
```

### Images

```html
<img
  data-cms-id="profile-photo"
  data-cms-type="image"
  src="/placeholder.jpg"
  alt="Profile photo"
/>
```

### Links/URLs

```html
<a data-cms-id="contact-link" data-cms-type="url" href="/contact">
  Contact Us
</a>
```

## Fetching Content via API

### API Helper

Use `fetchContent()` from `src/lib/api.ts` — it calls the public content endpoint and returns a `Record<string, string>` keyed by `cmsId`:

```typescript
import { fetchContent } from '@/lib/api'

const content = await fetchContent()
console.log(content['hero-title'])       // "Welcome to Our Site"
console.log(content['hero-description']) // "<p>This is rich text content.</p>"
```

### Response Shape

The underlying API returns:

```json
{
  "content": [
    {
      "cmsId": "hero-title",
      "type": "text",
      "content": { "text": "Welcome to Our Site" },
      "updatedAt": "2024-01-15T10:30:00Z"
    },
    {
      "cmsId": "hero-description",
      "type": "richtext",
      "content": { "text": "<p>This is rich text content.</p>" },
      "updatedAt": "2024-01-15T10:30:00Z"
    },
    {
      "cmsId": "hero-image",
      "type": "image",
      "content": { "url": "https://cdn.example.com/hero.jpg" },
      "updatedAt": "2024-01-15T10:30:00Z"
    }
  ]
}
```

> ⚠️ **Two things routinely trip people up here.**
>
> 1. The key is **`cmsId`**, not `contentId` — mapping the wrong one produces a
>    map full of `undefined` keys and every field silently falls back.
> 2. **`content` is a wrapper object, not a bare string.** Text and rich text
>    arrive as `{ text }`, images as `{ url }`, and numeric fields as
>    `{ count }` / `{ rating }`. Assigning it straight into a `Record<string,
>    string>` renders `[object Object]` on the live site.
>
> Flatten with a helper rather than at each call site, and keep the raw value
> around for `url` fields — those carry BOTH a destination and a label
> (`{ url, text }`), and flattening early throws the label away.

### Plain-text blocks come back as HTML

The dashboard edits **every** content block with a rich-text editor, including
ones your site renders as a bare string. Saving the plain text
`Stilul începe cu privirea` stores:

```json
{ "text": "<p>Stilul începe cu privirea</p>" }
```

Rendered as a React text node, that shows the visitor `<p>…</p>`. It only
surfaces after the first edit, so it passes every check until an editor touches
the field. Expose two readers and pick per field:

```ts
// Strips markup — for anything rendered as a text node (almost everything).
cms.text('home-hero-title', 'Stilul începe cu privirea')

// Raw — ONLY where the value goes through dangerouslySetInnerHTML.
cms.html('boutique-address', '<p>Adina Buzatu</p><p>…</p>')
```

A workable stripper turns block-level tags into spaces (so `<p>a</p><p>b</p>`
reads "a b"), removes the remaining tags, and decodes entities **after** tag
removal — decoding first would turn an encoded `&lt;script&gt;` into a live one.
Always sanitise whatever you pass to `dangerouslySetInnerHTML`.

## When to Use Content Blocks vs Page Elements

**Prefer page elements** ([Pages & Elements](./05-pages.md)) for any composable page where editors should be able to add, remove, reorder, or swap sections. That includes the homepage and every marketing page in this project — they all flow through `DynamicPage`.

**Use content blocks** for individual values bound to a fixed UI component that lives in code — e.g. a tagline inside a static `Banner`, a copyright string in the footer, or per-section copy on a landing page that's intentionally not a CMS page.

## React Integration (Vite + React)

Use `fetchContent()` from `src/lib/api.ts` which returns a `Record<string, string>` keyed by `cmsId`:

```tsx
// src/components/AnnouncementBanner.tsx — illustrative; this banner lives in code
import { useEffect, useState } from 'react'
import { fetchContent } from '@/lib/api'

export function AnnouncementBanner() {
  const [content, setContent] = useState<Record<string, string>>({})

  useEffect(() => {
    fetchContent().then(setContent).catch(() => {})
  }, [])

  return (
    <aside className="bg-pannu-section py-3 px-6">
      <span data-cms-id="announcement-text">
        {content['announcement-text'] ?? 'Default announcement.'}
      </span>
      <a data-cms-id="announcement-link" data-cms-type="url" href={content['announcement-link'] ?? '#'}>
        Learn more
      </a>
    </aside>
  )
}
```

`fetchContent()` in `src/lib/api.ts`:

```typescript
/** Unwrap the `{ text }` / `{ url }` wrapper the API returns. */
function flatten(raw: unknown): string {
  if (raw == null) return ''
  if (typeof raw === 'string') return raw
  if (typeof raw === 'object') {
    const obj = raw as { text?: unknown; url?: unknown }
    if (obj.text != null) return String(obj.text)
    if (obj.url != null) return String(obj.url)
  }
  return String(raw)
}

export async function fetchContent(): Promise<Record<string, string>> {
  // No projectId parameter — this endpoint is keyed off the API key alone.
  const data = await get<{ content: ContentBlock[] }>(publicUrl('/api/public/content'))
  const map: Record<string, string> = {}
  for (const block of data.content) map[block.cmsId] = flatten(block.content)
  return map
}
```

If any of your fields are `url` type, return the **raw** value instead and
flatten at the point of use — see the `useCmsContent` hook in
[Global Elements](./10-global-elements.md), which needs both halves of a link.

## A field is only editable if its node can be CLICKED

The SDK binds click-to-edit to elements carrying `data-cms-id`. A marker hidden
with `className="hidden"` satisfies "has the attribute" and fails "can be
clicked" — the field exists, shows up in no panel, and can never be changed.

This bites hardest on **buttons**, where the label is visible but the
destination is not. Don't split them across two blocks with the URL in a hidden
marker. Use ONE block of type `custom` holding both halves:

```jsonc
// content of the block `home-hero-cta`, type: "custom"
{ "text": "DESCOPERĂ COLECȚIA", "url": "/categorie-produs/ochelari-de-soare" }
```

```tsx
const cta = cms.link('home-hero-cta', 'DESCOPERĂ', '/magazin')  // { label, url }

<a href={cta.url}>
  <span data-cms-id="home-hero-cta" data-cms-type="url">{cta.label}</span>
</a>
```

A `custom` block renders **one labelled input per key it holds**, so clicking
the button opens a panel with a Text field and a Url field together. (Other
types read fixed keys off an image-shaped scaffold, so they show `alt` / `fit` /
`linkTarget` rows whether or not the block uses them.)

When a destination genuinely has no visible element of its own — a whole card
that is itself the link — give it one, rendered only inside the editor:

```tsx
{isInCmsEditor() && (
  <span className="mt-2 inline-flex items-center gap-1 rounded bg-black/80 px-2 py-1 text-[11px] text-white">
    🔗 <span data-cms-id={`card-${n}-link`} data-cms-type="url">{link}</span>
  </span>
)}
```

Keep that chip **outside** the flex/grid child that sizes the card, or it
becomes a sibling column and breaks the row.

## Never bind `data-cms-id` to a shortened or derived value

Whatever is inside the bound element is what the editor shows you, and what it
writes back on save. So the node must contain the **whole, exact** value.

The classic way to get this wrong is a "read more" excerpt:

```tsx
{/* WRONG — the element holds a 300-character slice plus a button label. */}
<p data-cms-id="collection-camasi-description">
  {isExpanded ? description : description.slice(0, 300)}
  <button>Citește mai mult</button>
</p>
```

Saving that stores the excerpt as the real description and appends the button's
text to it. In one project this silently truncated 28 category descriptions —
849 characters down to 317 — before anyone noticed.

```tsx
{/* RIGHT — full text in the DOM, collapsed visually, control outside. */}
<p data-cms-id="collection-camasi-description" className={expanded ? '' : 'line-clamp-4'}>
  {description}
</p>
{isLong && <button onClick={toggle}>Citește mai mult</button>}
```

Clamping in CSS also keeps the whole text in the DOM for search engines. The
same rule covers any transform: don't bind to uppercased, sentence-cased,
formatted or otherwise-derived text — bind to the raw value and style it.

> **Recovery note.** Once a value is in the CMS it overrides the code fallback,
> so a bad value cannot be repaired by re-reading the rendered page — you'd read
> your own bad data back. Recover from source.

## Naming Conventions

Use descriptive, hierarchical names:

```
page-section-element
```

Examples:
- `home-hero-title`
- `home-hero-description`
- `about-team-heading`
- `footer-copyright`
- `nav-logo`

## Best Practices

1. **Unique IDs**: Every `data-cms-id` must be unique across your site
2. **Meaningful Names**: Use descriptive names that indicate location and purpose
3. **Default Content**: Always provide fallback content in your HTML
4. **Type Hints**: Use `data-cms-type` for non-text content
5. **Organize by Page**: Prefix IDs with page or section names

## Migrating from Legacy Content Blocks

If you're using the older ContentBlock system:

1. Content blocks still work with the Visual Editor
2. The SDK automatically maps `data-cms-id` to existing content
3. New content is created as PageElement content when using Pages

## Next Steps

- [Pages & Elements](./05-pages.md) - Modern page-based content management
- [API Reference](./09-api-reference.md) - Full API documentation
