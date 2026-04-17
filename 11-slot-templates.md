# Slot-Based Templates

Slot-based templates are components that manage a numbered list of repeating content items — think project showcases, team grids, or portfolio galleries — where editors need to add/remove items **and** drag them into a custom display order.

## How It Works

1. Each item occupies a numbered **slot** (`project-1-*`, `project-2-*`, … up to your chosen max)
2. A `project-N-order` field per slot stores that slot's display position
3. The Visual Editor auto-detects the grouped pattern, collapses all slots into a draggable accordion, and writes updated `order` values on drop
4. Your component sorts populated slots by `order` before rendering

This keeps the data model simple (each slot's fields are independent key-value pairs) while giving editors a clean drag-and-drop UI without any custom CMS configuration.

## Naming Convention

All fields for slot `N` share a common prefix: `project-N-`:

| Field | `data-cms-type` | Purpose |
|-------|----------------|---------|
| `project-N-image` | `image` | Cover image |
| `project-N-client` | `text` | Display name (shown in drag list) |
| `project-N-tags` | `text` | Comma-separated tags |
| `project-N-url` | `url` | Link destination |
| `project-N-order` | `text` | Display position (integer); managed by drag-and-drop |
| `project-N-variant` | `select` | Layout hint — `square`, `tall`, or `wide` |

The `client` field doubles as the row label in the Visual Editor drag list. The `order` field is hidden from the flat field list — it is written automatically when the editor reorders items.

## Showcase Component

```tsx
// src/components/sections/Showcase.tsx
import { Fragment } from 'react'

const SLOT_COUNT = 60

interface ShowcaseProject {
  client?: string
  image?: string
  url?: string
  tags?: string
  order?: number
  variant?: 'tall' | 'square' | 'wide'
}

interface ShowcaseProps {
  projects?: ShowcaseProject[]
  marginTop?: number
  marginBottom?: number
  elementId?: string
}

function aspectClass(variant?: string) {
  if (variant === 'tall') return 'aspect-[3/4]'
  if (variant === 'wide') return 'aspect-video'
  return 'aspect-square'
}

export function Showcase({
  projects = [],
  marginTop = 0,
  marginBottom = 0,
  elementId,
}: ShowcaseProps) {
  const isTemplateMode = !elementId
  const id = (name: string) => (elementId ? `${elementId}-${name}` : name)

  // Sort populated slots by their per-slot order value, then by slot index
  const populated = projects
    .map((p, i) => ({ ...p, slot: i + 1 }))
    .filter(p => p.image || p.client)

  const displayProjects = [...populated].sort((a, b) => {
    const aOrd = a.order ?? a.slot
    const bOrd = b.order ?? b.slot
    return aOrd - bOrd
  })

  const templateProjects = isTemplateMode
    ? [
        { slot: 1, client: 'Client A', variant: 'square' as const },
        { slot: 2, client: 'Client B', variant: 'tall' as const },
        { slot: 3, client: 'Client C', variant: 'wide' as const },
        { slot: 4, client: 'Client D', variant: 'square' as const },
      ]
    : displayProjects

  return (
    <div
      data-cms-insertable={isTemplateMode ? 'showcase' : undefined}
      data-cms-name={isTemplateMode ? 'Showcase' : undefined}
      data-cms-category={isTemplateMode ? 'portfolio' : undefined}
      data-cms-element-id={!isTemplateMode ? elementId : undefined}
      style={{ marginTop: `${marginTop}px`, marginBottom: `${marginBottom}px` }}
    >
      {/* Rendered grid */}
      <div className="grid grid-cols-2 md:grid-cols-3 gap-4 px-8 py-12">
        {templateProjects.map(project => {
          const card = (
            <div className={`relative overflow-hidden bg-gray-100 group ${aspectClass(project.variant)}`}>
              {project.image && (
                <img
                  src={project.image}
                  alt={project.client || ''}
                  className="absolute inset-0 w-full h-full object-cover transition-transform duration-500 group-hover:scale-105"
                />
              )}
              {(project.client || isTemplateMode) && (
                <div className="absolute bottom-0 left-0 right-0 p-4 bg-gradient-to-t from-black/60 to-transparent">
                  <span className="text-white text-sm font-medium">
                    {project.client || 'Client Name'}
                  </span>
                </div>
              )}
            </div>
          )

          return project.url ? (
            <a key={project.slot} href={project.url} className="block" target="_blank" rel="noopener noreferrer">
              {card}
            </a>
          ) : (
            <div key={project.slot} className="block">{card}</div>
          )
        })}
      </div>

      {/* Hidden CMS fields — 60 numbered slots */}
      {Array.from({ length: SLOT_COUNT }, (_, i) => {
        const n = i + 1
        const p = projects[i]
        return (
          <Fragment key={n}>
            <img
              data-cms-id={id(`project-${n}-image`)}
              data-cms-type="image"
              src={p?.image || undefined}
              alt=""
              className="hidden"
            />
            <span data-cms-id={id(`project-${n}-client`)} data-cms-type="text" className="hidden">
              {p?.client}
            </span>
            <span data-cms-id={id(`project-${n}-tags`)} data-cms-type="text" className="hidden">
              {p?.tags}
            </span>
            <span data-cms-id={id(`project-${n}-url`)} data-cms-type="url" className="hidden">
              {p?.url}
            </span>
            {/* order field — written by drag-and-drop, not edited manually */}
            <span data-cms-id={id(`project-${n}-order`)} data-cms-type="text" className="hidden">
              {p?.order ?? n}
            </span>
            <span
              data-cms-id={id(`project-${n}-variant`)}
              data-cms-type="select"
              data-cms-options="square,tall,wide"
              className="hidden"
            >
              {p?.variant || 'square'}
            </span>
          </Fragment>
        )
      })}

      <span data-cms-id={id('margin-top')} data-cms-type="text" className="hidden">{marginTop}</span>
      <span data-cms-id={id('margin-bottom')} data-cms-type="text" className="hidden">{marginBottom}</span>
    </div>
  )
}
```

### Key points

- `SLOT_COUNT = 60` — all 60 hidden field groups are always rendered so the CMS can discover them. Only slots with content are displayed.
- The `id()` helper produces bare suffixes in template mode and `${elementId}-suffix` in render mode.
- `project-N-client` is the field used as the row label in the Visual Editor drag list — choose a human-readable identifier field for this role.
- `project-N-order` starts at `n` (natural slot order) and is overwritten by the Visual Editor on each drag.

## Registering the Template

Add `<Showcase />` to your `TemplateRegistry` inside `Layout.tsx`:

```tsx
// src/components/layout/Layout.tsx
import { Showcase } from '../sections/Showcase'

function TemplateRegistry() {
  const isInCmsIframe = typeof window !== 'undefined' && window.self !== window.top
  if (!isInCmsIframe) return null

  return (
    <div id="cms-templates" aria-hidden="true" className="sr-only">
      {/* existing templates … */}
      <Showcase />
    </div>
  )
}
```

## Rendering in DynamicPage

```tsx
// src/pages/DynamicPage.tsx
import { Showcase } from '../components/sections/Showcase'

function renderElement(element: PageElement) {
  const { elementId, content, templateName } = element
  const contentKeys = Object.keys(content)

  // … other templates …

  if (elementId.includes('showcase') || templateName === 'Showcase') {
    const SLOT_COUNT = 60

    const projects = Array.from({ length: SLOT_COUNT }, (_, i) => {
      const n = i + 1
      const get = (suffix: string) => {
        const key = contentKeys.find(k => k.endsWith(`-project-${n}-${suffix}`))
        return key ? String(content[key] ?? '') : undefined
      }
      const getImage = (suffix: string) => {
        const key = contentKeys.find(k => k.endsWith(`-project-${n}-${suffix}`))
        if (!key) return undefined
        const raw = content[key]
        return typeof raw === 'object' && raw !== null
          ? (raw as { url: string }).url
          : raw ? String(raw) : undefined
      }

      const rawVariant = get('variant') ?? 'square'
      const variant = rawVariant === 'tall' ? 'tall'
        : rawVariant === 'wide' ? 'wide'
        : 'square' as const

      const rawOrder = get('order')
      const order = rawOrder ? parseInt(rawOrder, 10) : undefined

      return { client: get('client'), image: getImage('image'), url: get('url'), tags: get('tags'), order, variant }
    })

    const marginTop = parseInt(String(content[contentKeys.find(k => k.endsWith('-margin-top')) ?? ''] ?? '0'), 10) || 0
    const marginBottom = parseInt(String(content[contentKeys.find(k => k.endsWith('-margin-bottom')) ?? ''] ?? '0'), 10) || 0

    return <Showcase projects={projects} marginTop={marginTop} marginBottom={marginBottom} elementId={elementId} />
  }

  return null
}
```

## Visual Editor Behaviour

When the editor opens a Showcase element the Visual Editor automatically detects the `project-N-*` grouping pattern and switches to a special UI:

### Projects accordion

All populated slots are collapsed into a labelled, draggable list. Each row shows:
- A **drag handle** (six-dot grip icon)
- The slot's **client name** as the row label
- A **chevron** to expand the slot's individual fields

Expanding a slot shows its image (with Library picker), client, tags, URL, and variant fields — editable inline without leaving the panel.

### Adding projects

Below the accordion sits a dashed **Add project** button. Clicking it:
1. Finds the next unpopulated slot in schema order (slot 1 first, then 2, …)
2. Seeds `project-N-client` with `"Project N"` and `project-N-order` with the next sequential order value
3. Sends `CMS_UPDATE` for both fields so the new row appears in the accordion and the iframe preview updates instantly

The button also renders when zero slots are populated — this is how you create the first item on a freshly inserted block. It disappears once every slot declared in the schema is filled. Rename the seeded `Project N` label inline via the accordion once you've added the row.

### Drag-and-drop ordering

Dragging a row to a new position immediately:
1. Reorders the list visually (no snap-back)
2. Writes new integer order values to each affected `project-N-order` field
3. Sends a `CMS_UPDATE` postMessage for every changed order field so the live preview re-sorts instantly

Clicking **Save** persists the new order values to the database. On next page load the Showcase reads `project-N-order` and sorts accordingly.

### Detection threshold

The grouped UI activates when the element's schema contains **`project-N-*` fields across two or more distinct slot numbers**. Any template that follows this naming convention will automatically get the drag-and-drop accordion — you do not need to configure anything extra.

## Adapting the Pattern

The same approach works for any numbered list of items. Change the prefix to match your domain:

| Use case | Prefix | Order field |
|----------|--------|-------------|
| Project showcase | `project-N-` | `project-N-order` |
| Team members | `member-N-` | `member-N-order` |
| Testimonials | `testimonial-N-` | `testimonial-N-order` |
| FAQ items | `question-N-` | `question-N-order` |

The Visual Editor groups any schema that has `project-N-*`-style fields — as long as the prefix before the number is `project`. For other prefixes use the same structure; the CMS uses the `client`-equivalent field (the first text field in the slot) as the row label in the drag list. The **Add project** button also requires a `project-N-client` field to seed — schemas without one will not get the add button.

## Checklist

1. Name slot fields `project-N-{field}` where `N` is 1-based
2. Include a `project-N-client` (or equivalent text) field — used as the drag-list row label
3. Include a `project-N-order` text field — initialise it to `n`, the slot index
4. Sort `projects` by `order ?? slot` before rendering
5. Register `<Showcase />` in `TemplateRegistry`
6. Add a detection branch in `renderElement()` matching `elementId.includes('showcase') || templateName === 'Showcase'`
7. Click **Save** in the Visual Editor after reordering to persist the new order

## Next Steps

- [Templates](./03-templates.md) — Dual-mode components and TemplateRegistry basics
- [Pages & Elements](./05-pages.md) — DynamicPage, `find()` helper, live preview messages
