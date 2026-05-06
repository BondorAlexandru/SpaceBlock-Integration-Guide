# Templates Guide

Templates are reusable content blocks that editors can insert into pages. Each template declares a **schema** (the editable fields it offers) and is paired with a **section component** that renders live instances of that template on a page.

## How Templates Work

1. **`TemplateRegistry`** inside `Layout.tsx` declares every template as a hardcoded `<div data-cms-insertable>` schema block, with `<span data-cms-id>` / `<img data-cms-id>` children listing each field.
2. The CMS SDK scans those nodes (in the editor iframe) and populates the **Add Element** list in the Visual Editor.
3. When an editor inserts a template, the CMS creates a `PageElement` with an `elementId` derived from the template ID (e.g. `hero-banner-abc123`).
4. **`DynamicPage`** fetches the page's elements, runs each through `renderElement()`, and renders the matching section component with the live `elementId` and content.

The template schema (in `TemplateRegistry`) and the rendering component (`src/components/sections/*`) are **decoupled**: the registry is the schema, the component is the renderer. They agree on field-suffix names — that's the only contract.

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
      HeroBanner.tsx    ← live-render component for the hero-banner template
      MethodologySection.tsx
      PillarsGrid.tsx
      RecentWork.tsx
      SimpleText.tsx
      RichText.tsx
      CtaSection.tsx
      index.ts          ← re-exports
  pages/
    DynamicPage.tsx     ← renderElement() with triple-detection + find()/findImage()
```

## TemplateRegistry — Hardcoded Schema Pattern

`TemplateRegistry` is a **private function component** inside `Layout.tsx` — not a separate file. Each template is a hardcoded JSX block that declares its fields. The block lives inside a visually-hidden container so it only exists for the SDK to discover, not for the user to see.

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
      {/* Hero Banner */}
      <div data-cms-insertable="hero-banner" data-cms-name="Hero Banner" data-cms-category="hero">
        <span data-cms-id="title" data-cms-type="text" />
        <span data-cms-id="description" data-cms-type="richtext" />
        <img data-cms-id="partner-logo" data-cms-type="image" alt="" />
        <img data-cms-id="background" data-cms-type="image" alt="" />
      </div>

      {/* Pillars Grid */}
      <div data-cms-insertable="pillars-grid" data-cms-name="Pillars Grid" data-cms-category="content">
        <span data-cms-id="title" data-cms-type="text" />
        <span data-cms-id="1-title" data-cms-type="text" />
        <span data-cms-id="1-description" data-cms-type="richtext" />
        <span data-cms-id="1-url" data-cms-type="url" />
        {/* ...repeat for slots 2 and 3... */}
        <span data-cms-id="cta-label" data-cms-type="text" />
      </div>

      {/* ...one block per template... */}
    </div>
  )
}

export function Layout({ children }: { children: React.ReactNode }) {
  return (
    <div className="min-h-screen flex flex-col">
      <Navbar />
      <main className="flex-1 pt-24">{children}</main>
      <Footer />
      <TemplateRegistry />
    </div>
  )
}
```

### Why hardcoded JSX, not `<HeroBanner />` calls?

Earlier patterns rendered the actual section component in template mode (without an `elementId`) so it would emit `data-cms-insertable` itself. That worked but had downsides:

- The hidden registry rendered the **full visual** of every template (gradients, blurs, decorative elements) just to surface schema attributes.
- Schema and rendering were welded together — changing the schema meant editing the component.
- Every section component had to carry conditional `isTemplateMode` logic.

Hardcoded schema blocks are leaner: the registry only declares fields, while the section component focuses on rendering live content.

## Template Attributes Reference

Used inside `TemplateRegistry`:

| Attribute | Where | Description |
|-----------|-------|-------------|
| `data-cms-insertable` | template root | Stable template ID — the SDK uses this in the elementId of new instances (e.g. `hero-banner-abc123`) |
| `data-cms-name` | template root | Display name in the Add Element picker |
| `data-cms-category` | template root | Groups templates in the picker (e.g. `hero`, `content`, `cta`) |
| `data-cms-id` | field child | Bare field suffix (e.g. `title`, `step-1-description`, `1-url`) |
| `data-cms-type` | field child | `text` · `richtext` · `image` · `url` · `select` · `toggle` |

Used by section components when rendering live:

| Attribute | Where | Description |
|-----------|-------|-------------|
| `data-cms-element-id` | section root | The element instance ID (`hero-banner-abc123` from CMS, `home-hero` from HomePage) |
| `data-cms-id` | field elements | Full key — `${elementId}-${suffix}` — so the SDK can sync each field |
| `data-cms-type` | field elements | Same value as in the registry |

## Section Components — Render-Mode Only

Section components only render live content. They take an optional `elementId` and use a small `fid()` helper to produce the full field key:

```tsx
// src/components/sections/HeroBanner.tsx

interface HeroBannerProps {
  elementId?: string
  title?: string
  description?: string
  partnerLogo?: string
  background?: string
}

export function HeroBanner({
  elementId,
  title = 'Default title',
  description = '',
  partnerLogo,
  background,
}: HeroBannerProps) {
  // Bare suffix when no elementId (used in static contexts) — otherwise prefixed
  const fid = (suffix: string) => (elementId ? `${elementId}-${suffix}` : suffix)

  return (
    <section
      data-cms-element-id={elementId}
      className="relative w-full overflow-hidden bg-pannu-darker"
    >
      <h1 data-cms-id={fid('title')} className="text-white">
        {title}
      </h1>
      <p data-cms-id={fid('description')} data-cms-type="richtext">
        {description}
      </p>
      {/* ... */}
    </section>
  )
}
```

The component does **not** emit `data-cms-insertable` — that's the registry's job. It just renders content with stable, syncable field IDs.

### Field ID pattern

| Context | `data-cms-id` value | Example |
|---------|---------------------|---------|
| Registry schema (template mode) | bare suffix | `title` |
| Section component with `elementId` | `${elementId}-${suffix}` | `hero-banner-abc123-title` |
| HomePage with `elementId="home-hero"` | `${elementId}-${suffix}` | `home-hero-title` |

`DynamicPage`'s `find(suffix)` helper is suffix-based, so it locates the value regardless of which prefix the CMS used.

## Rendering in DynamicPage

`renderElement()` is a plain function inside `DynamicPage.tsx`. It uses **triple-detection** so matching works even when the CMS doesn't populate `templateName`:

```tsx
function renderElement(element: PageElement) {
  const { elementId, templateName, content } = element
  const contentKeys = Object.keys(content)
  const find = (suffix: string) => findValue(content, suffix)
  const image = (...suffixes: string[]) => findImage(content, ...suffixes)

  if (
    templateName === 'Hero Banner' ||
    elementId.includes('hero-banner') ||
    contentKeys.some(k => k.endsWith('-partner-logo'))
  ) {
    return (
      <HeroBanner
        elementId={elementId}
        title={find('title')}
        description={find('description')}
        partnerLogo={image('partner-logo')}
        background={image('background')}
      />
    )
  }

  // ...one branch per template...
}
```

`findValue` and `findImage` are both defined in `DynamicPage.tsx`:

- **`findValue(content, suffix)`** — locates a field by suffix and unwraps `{ url, text }` objects.
- **`findImage(content, ...suffixes)`** — tries multiple suffixes in order and unwraps `{ url }` objects. Use it for any image field; the CMS sometimes returns plain strings, sometimes `{ url }`.

## Adding a New Template — Checklist

1. **Register the schema** in `TemplateRegistry` inside `Layout.tsx`:
   ```tsx
   <div data-cms-insertable="my-section" data-cms-name="My Section" data-cms-category="content">
     <span data-cms-id="title" data-cms-type="text" />
     <span data-cms-id="body" data-cms-type="richtext" />
   </div>
   ```
2. **Create the renderer** at `src/components/sections/MySection.tsx` using the section-component pattern (optional `elementId`, `fid()` helper, `data-cms-element-id` on root, `data-cms-id` on fields).
3. **Re-export** from `src/components/sections/index.ts`.
4. **Add a detection branch** in `renderElement()` in `DynamicPage.tsx` so dropped instances render.
5. **(If used on the homepage)** Add the props to `HomePage.tsx` with `elementId="home-my-section"`.

## Conditional Field Visibility

Use `data-cms-show-when` on a field in the registry to show it only when a select/toggle has a specific value:

```tsx
<div data-cms-insertable="cta-section" data-cms-name="CTA Section" data-cms-category="cta">
  <span data-cms-id="variant" data-cms-type="select" data-cms-options="dark,light" />
  <span data-cms-id="title" data-cms-type="text" />
  <span data-cms-id="cta-icon" data-cms-type="image" data-cms-show-when="variant:dark" />
</div>
```

## Next Steps

- [Pages & Elements](./05-pages.md) — DynamicPage, live-preview, helpers
- [Content Blocks](./04-content-blocks.md) — simple `data-cms-id` editing without templates
- [Slot-Based Templates](./11-slot-templates.md) — variable slot counts (1-12 columns, etc.)
