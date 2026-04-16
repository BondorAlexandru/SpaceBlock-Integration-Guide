# Visual Editor Integration

The Visual Editor allows content editors to click directly on elements in your website and edit them inline.

## How It Works

1. Your website loads inside an iframe in the SpaceBlock Visual Editor
2. The SDK detects it's running in the editor and enables edit mode
3. Content editors can click on elements with `data-cms-id` attributes to edit them
4. Changes are saved to SpaceBlock and synced to your site

## Making Elements Editable

Add `data-cms-id` attributes to elements you want to be editable:

```html
<!-- Simple text -->
<h1 data-cms-id="hero-title">Welcome to Our Site</h1>

<!-- Paragraph with rich text -->
<p data-cms-id="hero-description" data-cms-type="richtext">
  This is editable rich text content.
</p>

<!-- Image -->
<img
  data-cms-id="hero-image"
  data-cms-type="image"
  src="/placeholder.jpg"
  alt="Hero image"
/>

<!-- Link/URL -->
<a data-cms-id="cta-button" data-cms-type="url" href="/contact">
  Contact Us
</a>
```

## Data Attributes

### `data-cms-id` (Required)
Unique identifier for the content block. Use descriptive names like:
- `hero-title`
- `about-description`
- `footer-copyright`

### `data-cms-type` (Optional)
Specifies the field type in the editor:

| Type | Description | Editor |
|------|-------------|--------|
| `text` | Plain text (default) | Text input |
| `richtext` | HTML content | Rich text editor (TipTap) |
| `image` | Image URL | Image picker |
| `url` | Link URL | URL input |
| `select` | Dropdown options | Select dropdown |
| `toggle` | Boolean on/off | Toggle switch |
| `rating` | Star rating (1-5) | Rating input |
| `collection-select` | Collection picker | Collection dropdown |

### `data-cms-options` (for select type)
Comma-separated options for select fields:

```html
<select
  data-cms-id="image-size"
  data-cms-type="select"
  data-cms-options="small,medium,large,full"
>
  <option value="medium">Medium</option>
</select>
```

## Click-to-Edit Behavior

When a user clicks an element in the Visual Editor:

1. The element is highlighted with a blue outline
2. The sidebar shows the editing form
3. For text: Direct inline editing is enabled
4. For rich text: TipTap editor opens in sidebar
5. For images: Image picker/uploader opens
6. Changes save automatically

## Setting Up Your Domain

1. Go to **Project Settings** in SpaceBlock
2. Set your **Production Domain** (e.g., `https://mysite.com`)
3. Set your **Development Domain** (e.g., `http://localhost:5173`)
4. Open the Visual Editor to see your site

## Nested Elements

For complex layouts, use parent elements with `data-cms-id`:

```html
<section data-cms-id="feature-section">
  <h2 data-cms-id="feature-section-title">Features</h2>
  <p data-cms-id="feature-section-description" data-cms-type="richtext">
    Description text here.
  </p>
</section>
```

## Handling Dynamic Content

For React/SPA apps, ensure elements exist before the editor tries to find them:

```jsx
function Hero({ title, description }) {
  return (
    <section>
      <h1 data-cms-id="hero-title">{title}</h1>
      <p data-cms-id="hero-description" data-cms-type="richtext">
        {description}
      </p>
    </section>
  );
}
```

## Rich Text Styling

Add this CSS class for rich text content:

```css
.rich-content {
  /* Headings */
  h1, h2, h3, h4, h5, h6 { font-weight: bold; margin: 1em 0 0.5em; }
  h1 { font-size: 2em; }
  h2 { font-size: 1.5em; }
  h3 { font-size: 1.25em; }

  /* Lists */
  ul, ol { padding-left: 1.5em; margin: 0.5em 0; }
  li { margin: 0.25em 0; }

  /* Links */
  a { color: #0066cc; text-decoration: underline; }

  /* Paragraphs */
  p { margin: 0.5em 0; }
}
```

Then apply it to rich text elements:

```html
<div
  data-cms-id="content"
  data-cms-type="richtext"
  class="rich-content"
>
  <!-- Rich text content renders here -->
</div>
```

## Live Preview Messages

The Visual Editor communicates with your site via `postMessage` to provide real-time preview updates. This is especially important for global elements (navbar/footer) that need to update across all pages.

### Listening for CMS Messages

Add this listener in your main page component to handle live preview updates:

```javascript
useEffect(() => {
  function handleCmsMessage(event) {
    // Global element updates (navbar/footer)
    if (event.data.type === 'GLOBAL_ELEMENT_UPDATE') {
      // Clear cache and refresh to show updated global elements
      const slug = location.pathname.replace(/^\//, '') || 'home';
      pageCache.delete(slug);
      setRefreshTrigger(prev => prev + 1);
    }

    // New element inserted
    if (event.data.type === 'CMS_INSERT_ELEMENT') {
      const elementId = event.data.elementId;
      setTimeout(() => {
        const inserted = document.querySelector(`[data-cms-element-id="${elementId}"]`);
        if (inserted) inserted.remove();
        const slug = location.pathname.replace(/^\//, '') || 'home';
        pageCache.delete(slug);
        setRefreshTrigger(prev => prev + 1);
      }, 100);
    }

    // Element deleted
    if (event.data.type === 'CMS_REFRESH_AFTER_DELETE') {
      const slug = location.pathname.replace(/^\//, '') || 'home';
      pageCache.delete(slug);
      setRefreshTrigger(prev => prev + 1);
    }
  }

  window.addEventListener('message', handleCmsMessage);
  return () => window.removeEventListener('message', handleCmsMessage);
}, [location.pathname]);
```

### Message Types

| Message Type | Purpose | Payload |
|-------------|---------|---------|
| `GLOBAL_ELEMENT_UPDATE` | Global element (navbar/footer) content changed | `{ elementType, content }` |
| `CMS_INSERT_ELEMENT` | New page element inserted | `{ elementId }` |
| `CMS_REFRESH_AFTER_DELETE` | Element deleted from page | - |
| `CMS_UPDATE` | Content block updated | `{ cmsId, content, contentType }` |

### Why This Matters

Without the message listener:
- Navbar/footer changes only show after saving and manually refreshing
- New elements appear twice (SDK-inserted + React-rendered)
- Deleted elements leave ghost DOM nodes

With the message listener:
- ✅ Global elements update in real-time as you type
- ✅ New elements render cleanly as React components
- ✅ Deleted elements disappear immediately
- ✅ Better editor experience with instant feedback

### Implementation Notes

1. **Cache Clearing**: Always clear your page cache when receiving update messages
2. **Refresh Trigger**: Use a state variable to trigger re-renders (e.g., `refreshTrigger`)
3. **Cleanup**: Remove the event listener when component unmounts
4. **Error Handling**: Wrap in try-catch to prevent crashes from malformed messages

See [Global Elements Guide](./10-global-elements.md) for complete integration examples.

## Troubleshooting

### Content not appearing
- Check that `data-cms-id` matches exactly in your HTML and CMS
- Verify your domain settings in Project Settings
- Check browser console for SDK errors

### Visual Editor not loading
- Ensure your site allows being embedded in iframes
- Check for CSP (Content Security Policy) issues
- Verify the SDK is loading correctly

### Changes not saving
- Check network tab for API errors
- Verify your API key is correct
- Ensure you have permission to edit the project

## Next Steps

- [Templates](./03-templates.md) - Create insertable content blocks
- [Content Blocks](./04-content-blocks.md) - Legacy content management
