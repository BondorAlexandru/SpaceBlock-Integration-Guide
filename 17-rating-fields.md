# Rating Fields

SpaceBlock has a **first-class rating field** — a 1–5 star picker in the Visual
Editor. It is detected by a single attribute and is easy to miss because it is
**not** driven by `data-cms-type`.

## The contract

An editable element becomes a rating field when it carries a **`data-cms-rating`**
attribute. The attribute value is the current/default rating (1–5).

```html
<!-- This is a rating field. No data-cms-type needed. -->
<span data-cms-id="quote-1-rating" data-cms-rating="5"></span>
```

The SDK's schema scanner keys off the attribute directly (paraphrased from
`spaceblock-sdk.js`):

```js
var ratingValue = editEl.getAttribute('data-cms-rating');
if (ratingValue) {
  fieldType = 'rating';
  defaultContent[cmsId] = parseInt(ratingValue) || 5;
  schema.push({ key: cmsId, label: fieldLabel, type: 'rating', ... });
  return; // checked BEFORE data-cms-type
}
```

Because the check runs **before** `data-cms-type`, adding
`data-cms-type="rating"` does nothing — there is no such type. Use the
`data-cms-rating` **attribute**.

## Where rating fields appear in the editor

Rating fields are collected into a dedicated **"Star Ratings"** panel in the
editor sidebar, grouped separately from everything else — including
[slot-template](./11-slot-templates.md) accordions. This is by design and is
**not** configurable:

- A field named `…-quote-1-rating` inside a `quote-N-*` slot family will **not**
  nest inside that slot's accordion row. It is hoisted into "Star Ratings".
- This is true regardless of the field's position in the DOM. The dashboard
  special-cases rating fields (and the SDK's `_handleRating` matches any
  `cmsId` containing `-rating`).

> **If you need the control inline in a slot accordion instead** (e.g. edited
> right next to the quote and author), you cannot use a rating field. Use a
> `select` with numeric options instead — the accordion inlines `select`:
>
> ```html
> <span data-cms-id="quote-1-score" data-cms-type="select" data-cms-options="5,4,3,2,1"></span>
> ```
>
> Trade-off: a dropdown, not a star picker. **Avoid the substring `rating`** in
> the field name for this workaround — the dashboard hoists anything matching
> `-rating` into the Star Ratings panel even when typed as `select`.

## Stored content shape

Rating content comes back as a number, a numeric string, or `{ rating: number }`.
Unwrap all three:

```ts
function unwrapRating(raw: unknown): number | undefined {
  if (raw == null) return undefined
  if (typeof raw === 'number') return raw
  if (typeof raw === 'string') return parseInt(raw, 10) || undefined
  if (typeof raw === 'object' && typeof (raw as any).rating === 'number') {
    return (raw as any).rating
  }
  return undefined
}
```

If your generic field `unwrap()` already handles `{ url, text }`, add a
`rating` branch to it so suffix lookups (`find('rating')`) resolve correctly:

```ts
if (typeof raw === 'object') {
  if (raw.url != null) return raw.url
  if (raw.text != null) return raw.text
  if (typeof raw.rating === 'number') return String(raw.rating)
  return ''
}
```

## Rendering the stars

You have two options.

### Option A (recommended) — render your own stars, keep the field editor-only

Render brand-styled stars in your component from the numeric value, and expose
the rating to the editor via a **hidden** `data-cms-rating` marker (gated to the
editor iframe like any other editor-only DOM — see
[Editor-Only DOM](./13-editor-only-dom.md)). The editor edits the field in the
"Star Ratings" panel; your rendered stars update from state on `CMS_UPDATE`.

```tsx
// Editor-only marker (inside your <EditorOnly> gate)
<span data-cms-id={fid('rating')} data-cms-rating={value} className="hidden" />

// Visible, brand-styled stars (plain render, no CMS attributes)
<Stars count={value} />   // your own component, any colour you like
```

This keeps full control of the visual (colour, size, half-stars) and avoids the
SDK's built-in star styling.

### Option B — let the SDK paint the stars

Give the rating element `data-cms-star` children. The SDK's `_handleRating`
colours each star inline based on the value:

```html
<div data-cms-id="quote-1-rating" data-cms-rating="5">
  <span data-cms-star="1">★</span>
  <span data-cms-star="2">★</span>
  <span data-cms-star="3">★</span>
  <span data-cms-star="4">★</span>
  <span data-cms-star="5">★</span>
</div>
```

The SDK sets **inline** colours — filled `#facc15` (yellow), empty `#d1d5db`
(gray) — and toggles `.text-yellow-400` / `.text-gray-300`. To keep the stars
on-brand, override with `!important` (a stylesheet `!important` rule beats the
SDK's inline style):

```css
[data-cms-star].text-yellow-400 { color: #2e5941 !important; } /* filled */
[data-cms-star].text-gray-300  { color: rgba(46,89,65,.28) !important; } /* empty */
```

Option B is WYSIWYG (the star picker edits the visible element) but you inherit
the SDK's star markup and must fight its inline colours.

## Live preview

Rating edits arrive over the standard `CMS_UPDATE` channel
(see [Pages & Elements](./05-pages.md)). Because the content may be
`{ rating: n }`, keep the raw value in state (don't flatten to a string) so your
`unwrap` can recover the number:

```ts
// In the CMS_UPDATE handler, store the raw value; unwrap at read time.
const value =
  typeof content === 'object' && content !== null ? content : String(content)
applyFieldUpdate(cmsId, value)
```

## Checklist

1. Declare the field with the **`data-cms-rating`** attribute (not
   `data-cms-type="rating"`).
2. Expect it in the **"Star Ratings"** editor panel, not in slot accordions.
3. Handle the `{ rating }` / number / numeric-string content shapes in `unwrap`.
4. Render brand stars yourself (Option A) or override the SDK's inline star
   colours with `!important` (Option B).
5. Keep `elementId`-prefixed keys (`${elementId}-quote-1-rating`) like every
   other field so live preview and page fetch resolve them.

## Next Steps

- [Slot-Based Templates](./11-slot-templates.md) — numbered slot families
- [Editor-Only DOM](./13-editor-only-dom.md) — gating hidden field markers
- [Pages & Elements](./05-pages.md) — `CMS_UPDATE` live-preview channel
