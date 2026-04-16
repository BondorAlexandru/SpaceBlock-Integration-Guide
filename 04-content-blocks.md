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

Use `fetchContent()` from `src/lib/api.ts` — it calls the public content endpoint and returns a `Record<string, string>` keyed by `contentId`:

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
      "id": "abc123",
      "contentId": "hero-title",
      "content": "Welcome to Our Site",
      "type": "text",
      "projectId": "project-123"
    },
    {
      "id": "def456",
      "contentId": "hero-description",
      "content": "<p>This is rich text content.</p>",
      "type": "richtext",
      "projectId": "project-123"
    }
  ]
}
```

## React Integration (Vite + React)

Use `fetchContent()` from `src/lib/api.ts` which returns a `Record<string, string>` keyed by `contentId`:

```tsx
// src/pages/HomePage.tsx
import { useEffect, useState } from 'react'
import { fetchContent } from '@/lib/api'

export function HomePage() {
  const [content, setContent] = useState<Record<string, string>>({})

  useEffect(() => {
    fetchContent().then(setContent).catch(() => {})
  }, [])

  return (
    <main>
      <h1 data-cms-id="home-title">
        {content['home-title'] ?? 'Default Title'}
      </h1>
      <div
        data-cms-id="home-intro"
        data-cms-type="richtext"
        className="rich-content"
        dangerouslySetInnerHTML={{
          __html: content['home-intro'] ?? '<p>Default intro text.</p>'
        }}
      />
      <img
        data-cms-id="home-hero-image"
        data-cms-type="image"
        src={content['home-hero-image'] ?? '/placeholder.jpg'}
        alt="Hero"
      />
    </main>
  )
}
```

`fetchContent()` in `src/lib/api.ts`:

```typescript
export async function fetchContent(): Promise<Record<string, string>> {
  const data = await get<{ content: ContentBlock[] }>(
    publicUrl('/api/public/content', { projectId: PROJECT_ID }),
  )
  const map: Record<string, string> = {}
  for (const block of data.content) map[block.contentId] = block.content
  return map
}
```

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
