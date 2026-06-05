# Editor-Only DOM

> **TL;DR** — Any element that exists *purely* so the Visual Editor can attach to it (typically a hidden `<img data-cms-id="…" src="…" className="hidden">` marker) must be gated to the editor iframe — otherwise public visitors download every marker's `src` on every page load. CSS `display: none` / `sr-only` / `opacity-0` do **not** prevent the browser from fetching `src`.

This applies to SPAs that render their own content (React, Vue, etc. with SDK `autoLoad: false`). Sites that use `autoLoad: true` — where the SDK populates content via the markers — should skip this page.

---

## The trap

Without thinking about it, the natural way to expose an extra image field to the editor looks like this:

```tsx
<img data-cms-id="hero-desktop-image" data-cms-type="image" src={url} className="hidden" />
```

It's hidden — feels free. It isn't. The browser's **preload scanner** runs before any CSS is applied: it walks the raw HTML, sees `src="…"`, and starts the download. By the time `display: none` could matter, the request is in flight. Every public visitor pays for every marker, on every page.

Multiplies fast in carousels:

```tsx
{/* 12 phantom fetches per visit */}
{Array.from({ length: 12 }, (_, i) => (
  <img key={i} data-cms-id={slotId(i)} data-cms-type="image" src={images[i]} className="hidden" />
))}
```

To find every instance in a project:

```bash
grep -rn 'data-cms-type="image"' src/
```

Any line that isn't the **visible** image is almost certainly the bug.

---

## The fix

There are two clean answers. Use whichever fits the case.

### Option 1 (preferred) — put `data-cms-id` on the visible element

If the field already drives something the user sees (a hero image, a card thumbnail, a logo), put `data-cms-id` directly on that `<img>`. One element serves both roles; no hidden duplicate to gate.

```tsx
{/* Single visible <img> — editor binds to the same element the user sees. */}
<img
  data-cms-id="hero-image"
  data-cms-type="image"
  src={url}
  alt={alt}
  loading="lazy"
/>
```

This is the structural fix and it's the simplest possible solution: the bug can't exist because the wasteful pattern is never written.

### Option 2 — gate editor-only DOM by iframe presence

When a marker has no visible counterpart (extra carousel slots, alternate viewport variants, optional fields), wrap its render in a two-line iframe check:

```tsx
{typeof window !== 'undefined' && window.self !== window.top && (
  <img data-cms-id="hero-desktop-image" data-cms-type="image" src={url} className="hidden" />
)}
```

That's the whole pattern. No URL params, no SDK API call, no helper required.

- **Iframe presence == editor presence.** The Visual Editor loads your site inside an iframe; public visitors never do. The SDK's own `isInEditor()` does exactly this check (`window.self !== window.top`).
- **Race-free.** The result is decided by the browser before any JS runs, so it's stable from the very first React render.
- **SSR-safe.** Returns `false` on the server (no `window`), which is what you want — markers shouldn't appear in server-rendered HTML either.

For projects with many gated blocks, factor it once into a helper to taste:

```ts
// lib/cms.ts
export const isInCmsEditor = (): boolean => {
  if (typeof window === 'undefined') return false;
  try { return window.self !== window.top; }
  catch { return true; }  // cross-origin frame access throws → treat as framed
};
```

Then call `{isInCmsEditor() && <img … />}` at each gate site. The helper is purely a DRY-ness convenience; the inline two-liner is equally correct.

### Loops

Gate the **whole iteration**, not each element — the array isn't even constructed in public mode:

```tsx
{/* Good — one branch, no allocation when not in editor. */}
{isInCmsEditor() && Array.from({ length: 12 }, (_, i) => (
  <img key={i} data-cms-id={slotId(i)} data-cms-type="image" src={images[i]} className="hidden" />
))}
```

The same applies to TemplateRegistry-style blocks that list every insertable schema — wrap the whole `<div>` once, not each child.

---

## What about text/url field markers?

Hidden `<span data-cms-id="…">{title}</span>` markers don't trigger network requests — text has no `src`. The fetch problem is specific to image (and any other `src`/`href`-bearing) markers, so gating text spans is a style choice rather than a perf fix. Most projects leave them ungated for simplicity.

---

## Compatibility

The gate is safe whenever **your site renders its own content** — React / Vue / SPA consumers that call the public API directly and pass values into components as props. That's the recommended setup and what every guide page in this series assumes.

The gate is **not** safe with `SDK autoLoad: true`, where the SDK fetches content from the API and writes it into the DOM via marker selectors. In that mode the markers are not editor-only — they're the canonical sites where content actually lives, regardless of editor presence. Don't gate them.

---

## Quick reference

```tsx
// Preferred: data-cms-id on the visible element.
<img data-cms-id="hero" data-cms-type="image" src={url} alt="" loading="lazy" />

// When there's no visible counterpart — gate by iframe presence.
{typeof window !== 'undefined' && window.self !== window.top && (
  <img data-cms-id="hero-desktop" data-cms-type="image" src={url} className="hidden" />
)}

// Audit:
//   grep -rn 'data-cms-type="image"' src/
// Every result that isn't the visible image or gated is a phantom fetch.
```
