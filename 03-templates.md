# Templates Guide

Templates are reusable content blocks that editors can insert into pages. They define the structure and editable fields of a component.

## How Templates Work

1. Each section component is **dual-mode**: without an `elementId` it outputs `data-cms-insertable` on its root (template mode); with an `elementId` it renders live content
2. `TemplateRegistry` inside `Layout.tsx` renders every section component without an `elementId`, making the CMS SDK discover all templates on any page
3. The SDK scans those nodes and populates the "Add Element" list in the Visual Editor
4. When an editor inserts a template, `DynamicPage` renders the component with the assigned `elementId`

## Recommended File Structure

```
src/
  components/
    layout/
      Layout.tsx        ← Navbar + main + Footer + TemplateRegistry (private)
      Navbar.tsx
      Footer.tsx
      index.ts
    sections/
      HeroBanner.tsx    ← dual-mode section (template + rendered)
      FeatureSection.tsx
      SimpleText.tsx
      RichText.tsx
  pages/
    DynamicPage.tsx     ← renderElement() + find() helper (no separate file)
```

## Dual-Mode Section Components

Every section component accepts an optional `elementId`. The template behaviour is derived from its absence.

```tsx
// src/components/sections/SimpleText.tsx

interface Props {
  elementId?: string
  text?: string
}

export function SimpleText({ elementId, text = 'Text goes here' }: Props) {
  const isTemplateMode = !elementId

  // Field IDs: no prefix in template mode, prefixed in render mode
  const textId = elementId ? `${elementId}-text` : 'text'

  return (
    <section
      // Template registration attributes — only present in template mode
      data-cms-insertable={isTemplateMode ? 'simple-text' : undefined}
      data-cms-name={isTemplateMode ? 'Simple Text' : undefined}
      data-cms-category={isTemplateMode ? 'content' : undefined}
      // Element ID for live editing — only present in render mode
      data-cms-element-id={!isTemplateMode ? elementId : undefined}
      className="max-w-3xl mx-auto px-8 py-12"
    >
      <p data-cms-id={textId} className="text-gray-700">
        {text}
      </p>
    </section>
  )
}
```

### Field ID pattern

| Mode | `data-cms-id` value | Example |
|------|---------------------|---------|
| Template (no `elementId`) | bare suffix | `text` |
| Render (with `elementId`) | `${elementId}-${suffix}` | `simple-text-abc123-text` |

## The TemplateRegistry

`TemplateRegistry` is a **private function component** inside `Layout.tsx` — not a separate file. It renders every section in template mode inside a visually-hidden container.

```tsx
// src/components/layout/Layout.tsx

function TemplateRegistry() {
  const isInCmsIframe = typeof window !== 'undefined' && window.self !== window.top
  if (!isInCmsIframe) return null

  return (
    <div
      id="cms-templates"
      aria-hidden="true"
      className="absolute w-px h-px p-0 -m-px overflow-hidden [clip:rect(0,0,0,0)] whitespace-nowrap border-0"
    >
      <SimpleText />
      <HeroBanner />
      <FeatureSection />
      <RichText />
      {/* Add every new section here */}
    </div>
  )
}

export function Layout({ children }: { children: React.ReactNode }) {
  return (
    <div className="min-h-screen flex flex-col">
      <Navbar />
      <main className="flex-1 pt-16">{children}</main>
      <Footer />
      <TemplateRegistry />
    </div>
  )
}
```

The Tailwind class `[clip:rect(0,0,0,0)]` is the visually-hidden pattern — equivalent to `clip: rect(0,0,0,0)` without inline styles.

## Template Attributes Reference

| Attribute | Where | Description |
|-----------|-------|-------------|
| `data-cms-insertable` | section root, template mode only | Template ID used by the SDK |
| `data-cms-name` | section root, template mode only | Display name in the "Add Element" UI |
| `data-cms-category` | section root, template mode only | Groups templates in the picker |
| `data-cms-element-id` | section root, render mode only | Identifies the live element instance |
| `data-cms-id` | field elements | Bare suffix (template) or `${elementId}-suffix` (render) |
| `data-cms-type` | field elements | `text` · `richtext` · `image` · `url` · `select` · `toggle` |

## Rendering in DynamicPage

`renderElement()` is a **plain function inside `DynamicPage.tsx`** — not a separate component or file. It uses triple-detection so matching works even when `templateName` is unpopulated.

```tsx
// src/pages/DynamicPage.tsx

function renderElement(element: PageElement) {
  const { elementId, templateName, content } = element
  const contentKeys = Object.keys(content)

  const find = (suffix: string) => {
    const key = contentKeys.find(k => k.endsWith(`-${suffix}`))
    return key ? String(content[key] ?? '') : undefined
  }

  if (
    templateName === 'Simple Text' ||
    elementId.includes('simple-text') ||
    contentKeys.some(k => k.endsWith('-text'))
  ) {
    return <SimpleText elementId={elementId} text={find('text')} />
  }

  if (
    templateName === 'Hero Banner' ||
    elementId.includes('hero-banner') ||
    contentKeys.some(k => k.endsWith('-hero-title'))
  ) {
    return (
      <HeroBanner
        elementId={elementId}
        title={find('hero-title')}
        description={find('hero-description')}
        ctaLabel={find('hero-cta-label')}
        ctaUrl={find('hero-cta-url')}
        image={find('hero-image')}
      />
    )
  }

  if (import.meta.env.DEV) {
    return (
      <div className="border-2 border-dashed border-orange-300 bg-orange-50 p-4 rounded text-orange-700 text-sm">
        Unknown template: <strong>{templateName || '(no name)'}</strong> — <code>{elementId}</code>
      </div>
    )
  }
  return null
}
```

## Adding a New Template — Checklist

1. **Create** `src/components/sections/MySection.tsx` with the dual-mode pattern
2. **Import and add** `<MySection />` inside `TemplateRegistry` in `Layout.tsx`
3. **Add a detection branch** in `renderElement()` in `DynamicPage.tsx`

## Conditional Field Visibility

Use `data-cms-show-when` to show fields only for certain variant values:

```tsx
<span
  data-cms-id={variantId}
  data-cms-type="select"
  data-cms-options="simple,with-cta"
  className="hidden"
>
  {variant}
</span>

{/* Only visible in the editor when variant = "with-cta" */}
<span data-cms-id={ctaLabelId} data-cms-show-when="variant:with-cta" className="hidden">
  {ctaLabel}
</span>
```

## Next Steps

- [Pages & Elements](./05-pages.md) — DynamicPage pattern and `find()` helper
- [Content Blocks](./04-content-blocks.md) — Simple `data-cms-id` editing without templates
- [Slot-Based Templates](./11-slot-templates.md) — Numbered slot components with drag-and-drop ordering (project showcases, team grids, galleries)
