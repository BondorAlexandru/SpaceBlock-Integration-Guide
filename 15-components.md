# Component Builder Guide

The **Component Builder** lets you build reusable components in the SpaceBlock
dashboard — including importing them straight from Figma — and have them render
on your site **alongside your existing `data-cms-*` elements**, with no rebuild.

Unlike hand-coded templates (where *your* site owns the markup and SpaceBlock
only fills in content), a built component carries its **own markup + scoped CSS**.
The SDK injects it into a mount point you place on the page, then binds its
fields exactly like any native element.

## How it works

1. You build a component in the dashboard (or import one from Figma). It is stored
   with: a `templateId`, framework-agnostic **HTML markup**, **scoped CSS** (every
   rule is prefixed with `.sb-cmp-<templateId>`), an editable-field **schema**
   (`data-cms-id` / `data-cms-type`), and default content.
2. On your site, you add a **mount point** where you want it to appear:

   ```html
   <div data-cms-component="cmp-hero-a1b2c3"></div>
   ```

3. The SpaceBlock SDK, on load, fetches your project's components, injects each
   component's markup + CSS into its matching mount point, then binds the
   `data-cms-id` fields from your content — the same binding it does for native
   elements. Your own `data-cms-*` elements are untouched; both render together.

Because the SDK fetches live, **editing a component in the dashboard updates your
site on the next load — no redeploy.**

## Adding a mount point

Put the mount `<div>` anywhere in your page where the component should render. The
`data-cms-component` value is the component's `templateId` (shown on each card in
the Components area):

```html
<header><!-- your native markup --></header>

<!-- CMS-built component renders here -->
<div data-cms-component="cmp-hero-a1b2c3"></div>

<footer><!-- your native markup --></footer>
```

The SDK sets `data-sb-mounted="1"` on a mount point once it's hydrated, so
re-renders (e.g. in React/SPA apps) won't double-inject. In React, render the
mount point like any empty container — the SDK's MutationObserver picks up
late-rendered mount points automatically.

## Variants

A component can have **variants** — alternate versions of the same component
(e.g. a `Dark` or `Compact` take). In the Components area, a component's variants
appear as chips on its card; you create them when importing from Figma ("Save as
→ Variant of …") or in a component's **Edit** dialog.

To render a specific variant at a mount point, add `data-cms-variant` with the
variant's name (case-insensitive) alongside the **parent** component's
`templateId`:

```html
<!-- the default (parent) component -->
<div data-cms-component="cmp-hero-a1b2c3"></div>

<!-- the "Dark" variant of that same component -->
<div data-cms-component="cmp-hero-a1b2c3" data-cms-variant="Dark"></div>
```

If the named variant doesn't exist, the SDK falls back to the parent component.
Each variant is a self-contained definition with its own markup and scoped CSS,
so variants never style-collide with each other or with the parent.

## Counters (repeatable lists)

A component can mark one inner list as **repeatable** so an editor can choose how
many items show — e.g. how many feature rows a pricing card displays. You set
this in the component (import review or **Edit** → "Repeatable list"): pick the
list, optionally set a **group size** (when one item is several sibling elements
— common in flat Figma exports, e.g. icon+label pairs), and a default count.

Under the hood the list container is tagged with the SDK's repeater attributes
(`data-cms-repeater`, `data-cms-repeater-item` on each row — or on a
`display:contents` wrapper per group) and a count field. The
SDK shows the first *N* rows and hides the rest, where *N* comes from the count
value in your content feed. **Nothing extra is required in your page markup** —
the behaviour is baked into the component; just keep your `data-cms-component`
mount point as usual.

There are two ways a list repeats:

- **Existing rows** — when the Figma frame already contains several copies of the
  item, the SDK reveals/hides those rows by the count (bounded by how many were
  built).
- **Single item (clone)** — when the frame contains the item only once, mark it
  repeatable anyway; the editor wraps it in a generated **flex container** (you
  choose direction, gap, wrap, justify, align) and the SDK **clones** it up to the
  count. Pair it with a collection (below) for distinct data per row.

## Collection data

A repeatable list can be **driven by a Collection** instead of fixed content — so
the list renders one row per collection item (e.g. a team grid from your "Team"
collection). In the component editor, open **Data source**, pick a collection, and
map each field in the repeated item to a collection column (`title`, `image`,
`description`, `slug`, or any custom column).

Under the hood the list container gets `data-cms-collection="<slug>"` and each
mapped field gets `data-cms-field="<column>"`. At runtime the SDK fetches the
collection, **clones the first row once per item**, and fills the mapped fields
(text → element text, images → `src`). It also pulls the list out of the
count-based repeater pass, so a collection list shows **all** its items.

**Nothing extra is required in your page markup** — keep your usual
`data-cms-component` mount point; the binding travels with the component. The SDK
reads items from the public collection endpoint
(`/api/public/project/{projectId}/collections/{slug}`), the same feed documented
in the Collections guide.

## Public endpoint

The SDK reads component definitions from:

```
GET /api/public/components/:projectId?apiKey=YOUR_API_KEY
→ { components: [ { templateId, name, category, markup, css, schema, defaultContent, countConfig, parentTemplateId, variantName } ] }
```

Variants are returned as ordinary entries in the same list, each carrying a
`parentTemplateId` (the parent's `templateId`) and a `variantName`. The SDK uses
those to resolve `data-cms-variant`.

It is API-key gated like the content endpoint, returns only active components that
have markup, and never exposes project secrets. You normally don't call this
yourself — `SpaceBlock.init(...)` handles it — but it's available for debugging.

## Importing from Figma

1. In **Settings → Figma**, add a Figma **personal access token** (read-only file
   scope is enough). The token is stored server-side and never sent to the browser.
   This is a one-time, admin-only setup.
2. In the **Components** area, click **Import from Figma** and **paste the Figma
   share link** the client sent (the whole `figma.com/design/…` URL — no need to
   extract anything). SpaceBlock opens the file and shows its frames as
   thumbnails; **click the ones to import** (no file keys or node IDs to type).
3. SpaceBlock walks each selected frame's **full node tree** and reproduces it
   faithfully:
   - **Structure & layout** → the complete nesting is preserved. Auto-layout
     frames become flex; free-form (absolutely-positioned) designs are
     reproduced with exact positions and sizes from each layer's bounds.
   - **Text layers** → editable text / textarea fields.
   - **Image fills** (leaf images and container backgrounds) → the original
     asset is re-hosted on your project's Media CDN (Figma URLs expire); leaf
     images become editable image fields.
   - **Vectors/icons** → inlined as re-hosted SVG.
   - **Styling** → solid & gradient fills, strokes, per-corner radius, shadows,
     and opacity are translated to scoped CSS.
4. Review the preview, then **Save**. The component appears in the list and is
   immediately served to your site.

> Image import requires the project's **Media CDN (Bunny.net)** to be configured
> (Settings → Media CDN), since Figma asset URLs are short-lived and must be
> re-hosted.

## Styling & scoping notes

- A component's CSS is **class-prefixed** (`.sb-cmp-<templateId>`), not Shadow DOM.
  That isolates most styles, but very aggressive global rules on the host page
  (e.g. `img { width: 100% }`) can still reach injected markup. If you hit a
  collision, scope your global CSS or adjust the component's CSS in the editor.
- Each component's CSS is injected once into `<head>` as
  `<style data-sb-cmp="<templateId>">`.

## Editing

Open a component to edit its **name**, **category**, **markup**, and **scoped CSS**
with a live preview. Its editable fields (from `data-cms-id` in the markup) are
listed for reference. Saving invalidates the cache; your site reflects the change
on the next load.
